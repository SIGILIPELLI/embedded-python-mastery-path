# Threads & Dual-Core MicroPython

Both the ESP32 and the RP2040 have more than one CPU core, and
MicroPython's `_thread` module can run Python code on a second core (or
a second OS-level thread on single-core ports that still support
`_thread` cooperatively). This is the one place true, preemptive
concurrency shows up in MicroPython — a fundamentally different hazard
profile than `uasyncio`'s cooperative single-threaded model. Reviewed
here against MicroPython's documented `_thread` behavior per port; the
race-condition examples are demonstrated with Python's own `threading`
module locally, since the GIL-vs-no-GIL distinction below is
specifically about MicroPython, not a claim about what's reproducible
on desktop CPython.

## Starting a second thread

```python
import _thread
import time

def worker(name, delay):
    for i in range(5):
        print(name, "iteration", i)
        time.sleep(delay)

_thread.start_new_thread(worker, ("core1-task", 0.5))

# Main thread keeps running independently
for i in range(5):
    print("main", "iteration", i)
    time.sleep(0.3)
```

On the RP2040, `_thread.start_new_thread` runs the new thread on the
**second physical core** — genuinely parallel execution, not
time-sliced. On the ESP32, MicroPython's `_thread` also targets the
second core (the ESP32 is dual-core; MicroPython by default runs its
main interpreter on one core and can place a `_thread` on the other).
This genuinely parallel execution is the entire point and also the
entire risk: unlike `uasyncio`, where only explicit `await` points can
interleave code, two threads on two cores can interrupt each other at
literally any machine instruction.

## The GIL that MicroPython does — and doesn't — have

MicroPython implements a GIL-like mechanism (a global interpreter lock)
on ports that support threading, which prevents two threads from
executing Python bytecode simultaneously inside the interpreter's core
data structures — this protects the interpreter's own internals (object
headers, the GC) from corruption. It does **not** protect your
program's data: two threads can still race on a shared Python-level
variable, because the lock is released and reacquired between
bytecode instructions, and a single line of Python source can compile
to several bytecode instructions with a lock release possible in
between.

```python
import _thread
import time

counter = 0

def increment(n):
    global counter
    for _ in range(n):
        counter += 1   # NOT atomic: read, add, store — three steps

_thread.start_new_thread(increment, (100000,))
increment(100000)
time.sleep(1)
print(counter)   # frequently less than 200000 — a lost-update race
```

`counter += 1` looks atomic in source but is read-modify-write at the
bytecode level; the GIL can switch threads between those steps, and two
threads both reading the same old value before either writes back loses
one of the increments. This is the single most common dual-core bug:
code that "looks safe" because it's one line of Python.

## Locks for shared state

```python
import _thread

counter = 0
lock = _thread.allocate_lock()

def increment(n):
    global counter
    for _ in range(n):
        with lock:
            counter += 1

_thread.start_new_thread(increment, (100000,))
increment(100000)
```

`_thread.allocate_lock()` returns a lock supporting `acquire()`/
`release()` and the `with` statement (recent MicroPython versions
support context-manager use directly; older ones require explicit
`acquire()`/`release()` — check the target port). Every access to
shared mutable state from more than one thread needs to go through the
same lock, consistently — a lock only protects the code paths that
actually use it; a variable protected by a lock in one thread but read
without it in another is still racy.

## Hardware access races

Locks matter even more once real peripherals are involved, because a
race there can leave hardware in an inconsistent state, not just a
wrong counter value:

```python
import _thread
from machine import I2C

i2c = I2C(0, scl=Pin(22), sda=Pin(21))
i2c_lock = _thread.allocate_lock()

def read_sensor():
    with i2c_lock:
        i2c.writeto(0x48, b'\x00')
        return i2c.readfrom(0x48, 2)

def write_config():
    with i2c_lock:
        i2c.writeto(0x48, b'\x01\xFF')
```

Without `i2c_lock` serializing access, one thread's `writeto`
addressing byte and another thread's interleaved `writeto`/`readfrom`
on the *same bus* can corrupt both transactions — I2C/SPI peripherals
have no concept of "this transaction belongs to thread A," so the
software must enforce mutual exclusion around any multi-step bus
transaction shared between threads.

## When threads beat `uasyncio` — rarely

`uasyncio` handles the overwhelming majority of "do several things at
once" device programming, because most device work is I/O-bound
(waiting on a network socket, a sensor, a timer) rather than
CPU-bound, and cooperative scheduling avoids every hazard described
above by construction (only `await` points can interleave). Threads are
worth reaching for specifically when:

- A task is **genuinely CPU-bound** for a sustained period (a
  computation that would otherwise block the event loop for the
  duration, with no natural `await` point to yield at) and needs to run
  concurrently with I/O-bound `uasyncio` code
- The two cores need to be doing meaningfully independent work with
  minimal coordination — e.g., one core handling a real-time sensor
  sampling loop, the other handling network/UI, with a queue or a few
  locked shared variables as the only contact point

Even in the CPU-bound case, check whether `@micropython.native` or
`@micropython.viper` can shrink the computation enough to fit inside
`uasyncio` cooperatively before reaching for a second thread — a
second core doesn't remove the correctness burden threads add, so it
should be a deliberate trade, not a default.

## Cheat sheet

| Concept | Detail |
|---|---|
| `_thread.start_new_thread(fn, args)` | Runs `fn` on the second core (RP2040, ESP32) |
| MicroPython's GIL | Protects interpreter internals, not your program's shared data |
| `counter += 1` | Not atomic — read/modify/write across separate bytecode ops |
| `_thread.allocate_lock()` | Mutual exclusion for shared Python state or shared peripherals |
| Bus peripherals (I2C/SPI) shared across threads | Always lock around the full multi-step transaction |
| Default choice | `uasyncio` for I/O-bound concurrency; threads only for sustained CPU-bound work |

## Exercise

Using Python's `threading` module (stand-in for `_thread` — the API
shape and race hazard are the same), write a program with a shared
dict `state = {"total": 0}` and two threads, each incrementing
`state["total"]` 50,000 times in a loop with no lock. Run it and print
the final total versus the expected 100,000, demonstrating the race.
Then fix it with a `threading.Lock()` around the increment and confirm
the total is exactly 100,000 every time you run it. In a comment,
explain the one important way this demonstration differs from the real
MicroPython/RP2040 case: `threading` on desktop CPython has a real GIL
serializing bytecode across OS threads on one core, while `_thread` on
RP2040 is genuinely running on two separate physical cores.
