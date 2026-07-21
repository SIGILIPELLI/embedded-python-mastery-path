# 09 · uasyncio Basics

Every module so far hit the same wall: `time.sleep()` freezes everything,
and doing several things "at once" meant hand-weaving `ticks_ms()` checks
through one big loop. MicroPython's answer is `uasyncio` — a lean version
of Python's `asyncio` — which lets you write each activity as its own
simple loop and run them all concurrently on one core. This is *the*
architecture for non-trivial MicroPython programs, and the capstone is
built on it. Everything below runs in Wokwi's *MicroPython on ESP32*
project.

## Why async on a microcontroller?

The C/C++ track solves "blink two LEDs at different rates" with the classic
`millis()` pattern — one loop, per-task timestamp variables, and an `if`
per task ([see the sibling lesson](https://sigilipelli.github.io/embedded-mastery-path/level-1/07-timers-interrupts/)).
It works, but every added task multiplies the bookkeeping, and one slow
step stalls the lot. Threads are heavy for a chip with ~100 KB of RAM.
Cooperative coroutines hit the sweet spot: each task *reads* like the
naive blocking version, but every `await` hands control to whoever's ready
next. One core, near-zero overhead, no locks.

```python
# The pattern that doesn't scale:          # The pattern that does:
last1 = last2 = 0                          # async def blink(pin, ms):
while True:                                #     while True:
    now = time.ticks_ms()                  #         pin.value(not pin.value())
    if time.ticks_diff(now, last1) > 250:  #         await asyncio.sleep_ms(ms)
        led1.value(not led1.value())
        last1 = now
    if time.ticks_diff(now, last2) > 700:
        ...
```

## Coroutines, await, and the event loop

`uasyncio` is imported as `asyncio` on current firmware (both names work).
A function declared `async def` is a **coroutine**; calling it creates a
task-to-be; `await` marks the places it's willing to pause:

```python
import uasyncio as asyncio
from machine import Pin

async def blink(pin_no, interval_ms):
    pin = Pin(pin_no, Pin.OUT)
    while True:
        pin.value(not pin.value())
        await asyncio.sleep_ms(interval_ms)   # yields to other tasks

async def main():
    t1 = asyncio.create_task(blink(5, 250))    # fast blinker
    t2 = asyncio.create_task(blink(4, 700))    # slow blinker
    await asyncio.sleep(10)                    # let them run 10 s
    print("done")

asyncio.run(main())
```

**Wiring:** LEDs (each via 220 Ω to GND) on pins 5 and 4. Both blink at
their own rhythm — no timestamp bookkeeping anywhere.

The rules that make it work:

- `await asyncio.sleep_ms(n)` pauses *this* task; others run meanwhile.
- `time.sleep()` inside a task blocks the **whole** event loop — the
  cardinal sin. Inside async code, always `await asyncio.sleep_ms()`.
- A task that computes for a long time without awaiting also starves the
  others — sprinkle `await asyncio.sleep_ms(0)` in long loops.
- `asyncio.create_task(coro())` starts a task; `asyncio.run(main())`
  starts the event loop and runs until `main()` returns.

## Two blinkers + a sensor poller

The canonical structure of a real device — several forever-loops, one
`main()` that launches them:

```python
import uasyncio as asyncio
from machine import Pin
import dht

sensor = dht.DHT22(Pin(15))
latest = {"temp": None, "hum": None}       # tasks share state via plain objects

async def blink(pin_no, interval_ms):
    pin = Pin(pin_no, Pin.OUT)
    while True:
        pin.value(not pin.value())
        await asyncio.sleep_ms(interval_ms)

async def poll_sensor(interval_ms=2000):
    while True:
        try:
            sensor.measure()
            latest["temp"] = sensor.temperature()
            latest["hum"] = sensor.humidity()
        except OSError:
            pass                           # keep last good values
        await asyncio.sleep_ms(interval_ms)

async def report(interval_ms=5000):
    while True:
        print("temp:", latest["temp"], "C  hum:", latest["hum"], "%")
        await asyncio.sleep_ms(interval_ms)

async def main():
    asyncio.create_task(blink(5, 250))
    asyncio.create_task(blink(4, 700))
    asyncio.create_task(poll_sensor())
    asyncio.create_task(report())
    while True:                            # keep main alive forever
        await asyncio.sleep(3600)

asyncio.run(main())
```

Four independent activities, each written as if it owned the chip. Adding a
fifth is one more `async def` and one more `create_task` — compare that to
re-weaving a `millis()` super-loop.

Two more tools you'll use immediately:

```python
# Run things in parallel and wait for all:
results = await asyncio.gather(fetch_a(), fetch_b())

# Give up on something slow:
try:
    await asyncio.wait_for(wifi_connect_async(), 15)
except asyncio.TimeoutError:
    print("no WiFi — running offline")
```

And the async button — no callbacks, no globals, just a loop that waits:

```python
async def watch_button(pin, on_press):
    last = 1
    while True:
        v = pin.value()
        if v == 0 and last == 1:           # falling edge
            on_press()
            await asyncio.sleep_ms(30)     # debounce window
        last = v
        await asyncio.sleep_ms(10)         # poll every 10 ms, yielding
```

10 ms polling from a coroutine is plenty for human buttons, and it composes
cleanly with everything else running.

!!! tip "When interrupts still win"
    `uasyncio` checks things when tasks yield; a `Pin.irq` (module 5)
    catches a 50 µs pulse that polling would miss. Rule of thumb: humans
    and sensors → coroutines; microsecond-precision external signals →
    interrupts feeding a flag that a task consumes.

## Cheat sheet

| Function / idiom | Purpose |
|---|---|
| `import uasyncio as asyncio` | Works on all firmware (plain `asyncio` on recent) |
| `async def f(): ...` | Define a coroutine |
| `await asyncio.sleep_ms(n)` / `sleep(s)` | Pause this task, run the others |
| `asyncio.create_task(coro())` | Start a concurrent task |
| `asyncio.run(main())` | Start the event loop |
| `asyncio.gather(a(), b())` | Run several, wait for all |
| `asyncio.wait_for(coro(), s)` | Timeout → raises `asyncio.TimeoutError` |
| `time.sleep()` in a task | **Never** — blocks the whole loop |
| Long computation | `await asyncio.sleep_ms(0)` periodically to yield |
| Shared state | Plain dicts/objects — single-threaded, no locks needed |
| vs `millis()` pattern | Same scheduling idea, but per-task code stays linear |

## Exercise

Rebuild module 3's **traffic light** with `uasyncio`, then extend it into
an intersection: task 1 runs lights A (pins 25/26/27) through
green-yellow-red; task 2 runs lights B (pins 12/13/14) with the opposite
phase; task 3 watches a pedestrian button (pin 4, async edge-detect); task
4 prints a once-per-second status line of both states. Coordinate A and B
with a shared `state` dict owned by a single `controller()` task — the
light tasks only *display* what the controller decides. A pedestrian press
requests an all-red interval of 3 s at the next phase change. No
`time.sleep()` anywhere. Bonus: wrap the whole thing in
`asyncio.wait_for(run(), 60)` and print a clean "simulation over" message.
