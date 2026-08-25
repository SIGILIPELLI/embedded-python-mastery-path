# Memory Management & the Garbage Collector

MicroPython runs on a heap measured in tens or a few hundred kilobytes,
not gigabytes. Every object — `int`, `str`, `list`, closures, even
bound methods — is a heap allocation, and the collector that reclaims
them runs on the same core doing your work. Understanding the heap and
the GC is what separates code that runs for five minutes from code that
runs for a year. Everything below is reviewed against MicroPython's
documented GC behavior; the timing figures are the ballpark values
MicroPython's own docs and forum measurements give for split-heap
boards, not numbers measured here.

## The heap, at a glance

`gc.mem_free()` and `gc.mem_alloc()` report the state of the heap in
bytes:

```python
import gc

gc.collect()
print("free:", gc.mem_free(), "used:", gc.mem_alloc())
```

Call `gc.collect()` before reading these — otherwise you're seeing free
space that includes garbage not yet reclaimed, which overstates how
much headroom you actually have.

## What allocates (and what doesn't)

Allocation-free code is a real design goal in tight loops. These
allocate:

- Any new `int` outside the small-int range (roughly ±2^30 on a 32-bit
  build — small ints are packed into the pointer itself and cost nothing)
- String concatenation (`a + b` builds a new string every time)
- List/dict/tuple literals, comprehensions, `.append()` past current capacity
- Closures and bound methods created fresh each call
- f-strings and `.format()` calls
- Slicing (`buf[2:10]` copies)

These don't:

- Reusing an existing `bytearray` via slice assignment (`buf[0:4] = data`)
- Indexing, arithmetic on existing small ints
- Calling a function that takes no default-argument objects to build

```python
# Allocates every iteration — builds a new bytearray each call
def read_frame_bad():
    return bytearray(64)

# Allocates once — reused forever
_frame = bytearray(64)
def read_frame_good():
    # fill _frame in place, e.g. via readinto()
    return _frame
```

`uart.readinto(buf)` / `i2c.readfrom_into()` exist specifically so you
can avoid allocating a fresh buffer per read — always prefer them over
`uart.read()` in a hot loop.

## `const()` — folding away lookups, not allocations

`micropython.const()` doesn't reduce heap pressure directly; it tells
the compiler to inline the value at compile time so there's no module
attribute lookup and, for module-level constants, no dict entry at all:

```python
from micropython import const

_ADC_MAX = const(4095)   # inlined as a literal everywhere it's used
_RETRY_COUNT = const(3)

def scale(raw):
    return raw / _ADC_MAX
```

Without `const()`, `_ADC_MAX` is a normal module global: every
reference is a dict lookup on the module's namespace, and the value
itself still occupies a heap slot. `const()` only helps with
module-level (or class-level, inside a class body) names holding
integers — it has no effect on strings, floats, or local variables, and
using it on a mutable value is meaningless since the point is a
compile-time integer substitution.

## Fragmentation

MicroPython's heap allocator is a simple block allocator with no
compaction. Long-running programs that allocate and free objects of
varying sizes can end up with the heap fragmented — plenty of
*total* free bytes, but no single free block big enough for the next
request:

```python
import gc

gc.collect()
print(gc.mem_free())   # e.g. 40000 bytes free...
buf = bytearray(35000)  # ...but this still raises MemoryError
```

This is the classic trap: `gc.mem_free()` tells you the sum of free
space, not the largest contiguous block. There's no MicroPython API to
query fragmentation directly. The practical fix is architectural:
allocate your large, long-lived buffers (network buffers, framebuffers,
big bytearrays) once at startup, before smaller short-lived objects
have had a chance to fragment the heap, and reuse them for the life of
the program.

```python
# Do this near the top of main.py, before any dynamic allocation
_NET_BUF = bytearray(4096)
_SENSOR_LOG = bytearray(2048)
# ... now run the rest of the app, which mostly allocates small,
# short-lived objects that the collector reclaims cleanly
```

## GC pause timing — the trap that bites at the worst moment

The garbage collector is a stop-the-world mark-and-sweep: when it runs,
*everything* pauses, including any interrupt-driven code you thought
was safe (hard IRQs are the exception — they still fire, but Python
callbacks scheduled from them are deferred). Collection time scales
with heap size and how much of it is live, not with how much garbage
there is — a full mark-sweep on a mostly-full 200 KB heap on an ESP32
can take single-digit milliseconds; on boards with more RAM or slower
clocks it can stretch into tens of milliseconds.

This matters enormously for anything with real-time constraints:

```python
import gc
from machine import Pin, Timer

pin = Pin(2, Pin.OUT)

def toggle(t):
    pin.value(not pin.value())

# A collection running mid-period here delays the toggle by however
# long the GC pause takes — if you need microsecond-accurate pulse
# timing, PIO or a hardware timer's own output-compare, not a Python
# callback, is the only reliable answer (see the PIO module).
tim = Timer(0)
tim.init(freq=1000, callback=toggle)
```

Two mitigations, in order of usefulness:

1. **Reduce allocation rate.** Fewer allocations means fewer, further
   apart collections. This is the real fix, not the next one.
2. **Trigger collections at known-safe points.** `gc.collect()` called
   explicitly, right after a chunk of work and before an idle/sleep
   period, means the *automatic* trigger (which fires whenever the
   allocator can't satisfy a request from free space) is less likely to
   land mid-critical-section.

```python
def main_loop():
    while True:
        read_sensors()
        transmit()
        gc.collect()   # pay the pause here, on our terms
        time.sleep_ms(50)
```

`gc.disable()` exists and stops automatic collection entirely — it does
not stop allocation from failing once the heap is full, it just means
you must call `gc.collect()` yourself before that happens. Disabling
the GC in a program that keeps allocating without ever calling
`gc.collect()` just turns a periodic pause into a guaranteed
`MemoryError`.

## Diagnosing `MemoryError`

```python
try:
    big = bytearray(50000)
except MemoryError as e:
    print("alloc failed:", e)
    gc.collect()
    print("free after collect:", gc.mem_free())
```

If `gc.mem_free()` after a fresh `gc.collect()` is still small, you have
a real leak or an undersized heap for the workload — not fragmentation.
Common leak sources on MicroPython specifically:

- Growing a module-level list/dict without bound (a log buffer that's
  never trimmed)
- Exceptions holding references via their traceback (`sys.exc_info()`
  chains, or an exception object stored in a variable that outlives the
  `except` block) — MicroPython's `gc` reclaims cycles, but a live
  reference is not a cycle
- Callbacks registered with closures that capture large objects, kept
  alive by a timer or IRQ handler that's never deregistered

`gc.threshold()` sets the allocation total (in bytes) that triggers an
automatic collection; lowering it collects more often (smaller, more
frequent pauses) at the cost of more total time spent in GC — a knob
worth tuning per application, not a fix for a genuine leak.

## Cheat sheet

| Tool / pattern | Effect |
|---|---|
| `gc.collect()` | Force a full mark-sweep now, on your schedule |
| `gc.mem_free()` / `gc.mem_alloc()` | Query heap state (call after `collect()` for accuracy) |
| `const()` | Compile-time inline for module/class-level int constants — no lookup, no separate object |
| `bytearray` + `readinto()` | Reuse one buffer instead of allocating per read |
| Pre-allocate big buffers at startup | Avoids fragmentation from later small allocations |
| `gc.threshold(n)` | Tune how often automatic collection fires |
| `gc.disable()` | Stops automatic collection — you own calling `gc.collect()` |

## Exercise

Write a MicroPython-style simulation (runnable under plain `python3`,
since the logic is generic) of an allocation tracker: a class
`FakeHeap` with `alloc(name, size)` and `free(name)` methods that track
a dict of live allocations and a running total, and a `mem_free(total)`
method. Then write a `run_scenario()` that allocates a 4000-byte buffer,
a series of ten 200-byte buffers, frees every other one of the ten, and
prints total free space before and after a simulated "collect" (which
in your simulation just recomputes the total from live allocations).
Add a comment explaining, in real MicroPython terms, why `mem_free()`
returning a large number here would still not guarantee a subsequent
4000-byte allocation succeeds.
