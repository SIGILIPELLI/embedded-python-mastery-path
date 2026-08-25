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
