---
description: "Performance Profiling & Optimization — Before optimizing anything, measure it. MicroPython has no cProfile, no line_profiler, and no sampling profiler …"
---

# Performance Profiling & Optimization

Before optimizing anything, measure it. MicroPython has no `cProfile`,
no `line_profiler`, and no sampling profiler — the toolbox is
`time.ticks_us()`, `time.ticks_diff()`, and careful hand-instrumentation.
That's a smaller toolbox than desktop Python, but it's enough to find
the loops that actually matter. Timing patterns below are exercised
with plain `python3` using `time.perf_counter()` as a stand-in for
`ticks_us()` (the roundtrip/wraparound behavior is called out
separately since desktop `perf_counter` doesn't share it); relative
costs (attribute lookup vs. local variable, etc.) are documented
MicroPython behavior, reviewed against the docs, not measured on real
hardware here.

## `ticks_us()` and why you can't just subtract

```python
import time

start = time.ticks_us()
do_work()
elapsed = time.ticks_diff(time.ticks_us(), start)
print("took:", elapsed, "us")
```

`time.ticks_us()` returns a value that **wraps around** — it is not a
monotonically increasing counter you can subtract with plain `-`. On a
32-bit MicroPython build the tick value wraps at `2**30` (per the
`ticks_us`/`ticks_diff` documentation, values are kept in the range
`[0, 2**30)` specifically so that the subtraction-with-wraparound math
in `ticks_diff` works correctly). Naive subtraction:

```python
# WRONG — breaks silently the moment the counter wraps
elapsed = time.ticks_us() - start   # can go negative near the wrap point
```

`time.ticks_diff(new, old)` handles the wraparound correctly by design
— always use it, never subtract ticks values directly. The same applies
to `ticks_ms()` and `ticks_cpu()`.

## Building a minimal instrumentation helper

```python
import time

def timeit(fn, *args, n=100):
    gc_note = "results include occasional GC pauses — that's realistic"
    start = time.ticks_us()
    for _ in range(n):
        fn(*args)
    total = time.ticks_diff(time.ticks_us(), start)
    return total / n   # average microseconds per call

avg_us = timeit(parse_packet, sample_packet)
print("avg:", avg_us, "us/call —", gc_note)
```

Averaging over many calls is essential: a single call's timing is noisy
(interrupts, an occasional GC pause) and on some boards the first call
into a code path pays a one-time cost (module-level code executing,
bytecode being loaded from flash rather than already cached).

## Where the hot spots actually are

MicroPython's bytecode interpreter has real, measurable costs that
don't exist in compiled C and are easy to trip over without realizing:

**Attribute and global lookups are not free.** Every `self.x` or
`module.func()` walks a lookup chain at runtime — there's no
compile-time binding. In a hot loop:

```python
# Slower: attribute lookup on every iteration
class Sampler:
    def run(self, n):
        for _ in range(n):
            self.adc.read_u16()   # looks up self.adc, then .read_u16, every time

# Faster: bind once outside the loop
class Sampler:
    def run(self, n):
        read = self.adc.read_u16    # bound method, looked up once
        for _ in range(n):
            read()
```

The same applies to module-level functions used inside loops — bind
`len`, `time.ticks_us`, etc. to a local name before a tight loop if it
runs thousands of times.

**Local variables beat globals beat attributes**, in that order, for
lookup speed, because MicroPython's function frames use a fast
array-indexed slot for locals but a dict lookup for globals and
instance attributes.

**Allocation inside a hot loop costs more than the allocation itself**
— see the memory module — because it also raises the odds a GC pause
lands inside your timing window.

## Choosing data structures for speed

- **Tuples over lists** for fixed-size, read-only data — no
  resize logic, slightly smaller.
- **`array.array` over `list`** for large sequences of one numeric
  type — packed C-level storage instead of a list of boxed objects,
  which matters both for allocation count and for cache-friendliness on
  chips with any cache.
- **Preallocate and index-assign, don't `.append()` in a loop** when
  the final size is known — appends can trigger list growth
  (reallocation + copy) at unpredictable points.

```python
import array

# Boxes 1000 separate int objects, plus list overhead
samples_list = [0] * 1000

# One contiguous block of 2-byte unsigned ints — much less memory,
# no per-element object overhead
samples_arr = array.array('H', [0] * 1000)
```

- **Dict lookups are O(1) but not free** — for a small fixed set of
  keys known in advance (e.g. dispatching on a handful of command
  bytes), an `if/elif` chain or even a tuple-indexed list can beat a
  dict for very small N, though the crossover point isn't worth
  chasing without measuring your actual case.

## A worked before/after

```python
import time

def checksum_v1(data):
    total = 0
    for i in range(len(data)):
        total += data[i]
    return total & 0xFF

def checksum_v2(data):
    # avoids repeated len()/indexing overhead by iterating directly,
    # and sum() is implemented in C internally even in MicroPython
    return sum(data) & 0xFF

data = bytes(range(256)) * 4   # 1024 bytes

t1 = timeit(checksum_v1, data, n=200)
t2 = timeit(checksum_v2, data, n=200)
print(f"v1: {t1:.1f}us  v2: {t2:.1f}us  speedup: {t1/t2:.2f}x")
```

On real hardware this kind of change routinely yields 2-4x on the
inner-loop-heavy version — the exact ratio depends on the port and
clock speed, which is exactly why you measure on your target rather
than trusting a rule of thumb.

## When profiling isn't enough

If `checksum_v2`-style Python-level optimization has plateaued and the
loop is still too slow, the next steps — in order of effort — are the
`@micropython.native` and `@micropython.viper` emitters (next module),
then C modules or PIO for the parts that genuinely need
deterministic, cycle-level timing that no amount of Python
restructuring will reach.

## How It Actually Works

Every performance rule of thumb in this module traces back to how
MicroPython's bytecode dispatch loop resolves names and how its timer
peripheral surfaces its counter to Python.

- **`ticks_us()` wraps at `2**30`, not `2**32`, because two bits are
  deliberately reserved so `ticks_diff`'s subtraction can detect
  wraparound unambiguously.** The underlying hardware timer/counter
  peripheral genuinely runs a much wider (often 64-bit) free-running
  counter; MicroPython masks it down to a fixed range specifically so
  `ticks_diff(new, old)` can compute `((new - old + 2**29) % 2**30) - 2**29`
  — a formula that gives the *correct* signed difference even across a
  wrap, as long as the true elapsed time never exceeds half the range.
  Plain subtraction breaks because unmasked arithmetic on wrapped values
  has no way to distinguish "time went backward" from "time wrapped
  forward" — the masking and the special diff function are a matched pair,
  not independent design choices.
- **Attribute lookup costs a real hash-table probe because `self.adc` and
  `module.func` are dictionary lookups at the bytecode level (`LOAD_ATTR`
  / `LOAD_GLOBAL` opcodes), not compile-time-resolved offsets.** Every
  Python object with attributes carries (or points to) a dict-like
  structure MicroPython calls a "map" internally; each `self.x` access
  walks that structure at runtime because Python's attribute model allows
  attributes to be added, removed, or shadowed by a subclass or instance
  at any time — genuine dynamic-language flexibility that has a genuine
  per-access cost. Binding `read = self.adc.read_u16` once turns N runtime
  lookups into 1, because the resulting bound-method object is then just
  invoked directly by reference, skipping the attribute-chain walk on
  every subsequent call.
- **Locals beat globals beat attributes because MicroPython's function
  frames allocate local variables as a fixed-size array indexed by
  bytecode-time slot number (`LOAD_FAST`/`STORE_FAST` opcodes) — computed
  once at compile time, since a function's local variable names are fully
  known before it ever runs.** Globals and attributes have no such
  static slot: they're resolved by name through a dict at every access
  because the set of module-level names or instance attributes isn't
  fixed until runtime. The three-tier speed hierarchy in this module is
  really "how much of the lookup got resolved at compile time" — locals
  fully static, globals looked up by name in one dict, attributes looked
  up by name potentially through more than one map (instance, then class,
  then base classes).
- **`array.array` genuinely stores raw C values contiguously instead of
  boxed Python objects, which is why it saves both memory and per-element
  overhead.** A `list` of ints holds pointers to individually heap-allocated
  int objects (or small-int-encoded pointers, per module 1) — accessing
  element `i` means dereferencing a pointer, then possibly unboxing.
  `array.array('H', ...)` allocates one block sized `2 * len` bytes and
  reads/writes native machine values directly via `struct`-style
  interpretation, with no per-element Python object at all — fewer heap
  allocations (helping the GC pause problem from module 1) and better
  memory locality on any port with an instruction/data cache.

## Cheat sheet

| Technique | Saves |
|---|---|
| `time.ticks_diff(new, old)` | Correct timing across tick wraparound |
| Bind method/function to a local before a loop | Repeated attribute/global lookup |
| `array.array` for numeric buffers | Per-element object boxing overhead |
| Preallocate + index-assign vs. repeated `.append()` | Reallocation/copy churn |
| `sum()`, `min()`, `max()` built-ins | C-level loop vs. Python-level loop |
| Average over N calls | Noise from IRQs and occasional GC pauses |

## Exercise

Write and run (via `python3`) a small benchmark harness with your own
`timeit(fn, *args, n)` helper using `time.perf_counter()`. Implement two
versions of a function that finds the maximum value in a list of 2000
integers: one using a manual `for` loop with indexing, one using the
built-in `max()`. Time both over at least 500 repetitions, print the
average microseconds per call for each, and print the speedup ratio.
Add a short comment on what you'd expect to change (and why) if this
same comparison were run on an actual MicroPython board instead of
desktop CPython.
