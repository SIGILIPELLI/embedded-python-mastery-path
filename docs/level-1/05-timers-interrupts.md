# 05 · Timers & Interrupts

So far our programs poll: spin in `while True`, checking everything on every
pass. That works until it doesn't — a busy loop misses a 5 ms button tap,
and `time.sleep()` freezes everything else. The hardware offers two better
tools: **timers** that call your function at fixed intervals, and **pin
interrupts** that call your function the instant a signal changes. Both
arrive in MicroPython as callbacks — with one crucial rule about what a
callback is allowed to do. Everything here runs in Wokwi's *MicroPython on
ESP32* project.

## machine.Timer: periodic callbacks

```python
from machine import Pin, Timer

led = Pin(5, Pin.OUT)

def tick(t):                      # every callback receives the Timer object
    led.value(not led.value())

tim = Timer(0)                    # ESP32 has hardware timers 0..3
tim.init(period=500, mode=Timer.PERIODIC, callback=tick)

print("Timer running — the LED blinks 'in the background'.")
# The main program is completely free:
import time
count = 0
while True:
    print("main loop is alive:", count)
    count += 1
    time.sleep(2)
```

The LED blinks at exactly 1 Hz while the main loop does something else
entirely — your first taste of concurrency. `mode=Timer.ONE_SHOT` fires
once instead; `tim.deinit()` stops the timer.

Portability: on the Pico, use virtual timers — `Timer(period=500, ...)` or
`Timer(-1)` — same API otherwise.

## Pin.irq: react the moment a pin changes

Polling a button means you can only notice a press as often as you check.
An **interrupt** flips this: the hardware watches the pin, and your handler
runs the moment the edge happens, whatever the main loop is doing.

```python
from machine import Pin
import time

button = Pin(4, Pin.IN, Pin.PULL_UP)
led = Pin(5, Pin.OUT)

press_count = 0

def on_press(pin):                        # the handler gets the Pin object
    global press_count
    press_count += 1
    led.value(not led.value())

button.irq(trigger=Pin.IRQ_FALLING, handler=on_press)

while True:
    print("presses so far:", press_count)
    time.sleep(2)
```

`IRQ_FALLING` fires on 1→0 (a pull-up button being pressed); `IRQ_RISING`
on 0→1; `trigger=Pin.IRQ_FALLING | Pin.IRQ_RISING` on both. Run it and
you'll also immediately rediscover module 3's lesson: one physical press
often counts 2–5 — **interrupts hear every bounce**. Interrupt-side
debouncing = ignore edges that arrive too soon after the last accepted one:

```python
last_ms = 0

def on_press(pin):
    global press_count, last_ms
    now = time.ticks_ms()
    if time.ticks_diff(now, last_ms) > 200:   # accept at most one press / 200 ms
        last_ms = now
        press_count += 1
```

## The ISR rules: what a handler must not do

Timer and pin callbacks are **interrupt service routines** (ISRs). They
interrupt your main code *anywhere* — possibly mid-way through the memory
allocator. So inside an ISR, MicroPython forbids allocating memory. These
all allocate, and will raise `MemoryError` or corrupt state inside an ISR:

```python
def bad_isr(t):
    print("tick", t)          # building the string allocates
    data.append(reading)      # growing a list allocates
    x = 3.5 * 2               # floats allocate!
    msg = "a" + "b"           # allocates
```

Safe inside an ISR: setting integer globals, reading/writing pins, storing
into *pre-allocated* arrays. The standard patterns:

**Pattern 1 — set a flag, act in the main loop:**

```python
flag = False

def on_press(pin):
    global flag
    flag = True               # just note that it happened

while True:
    if flag:
        flag = False
        handle_press()        # ordinary code — allocate, print, whatever
    do_other_work()
```

**Pattern 2 — `micropython.schedule`: run "soon", outside ISR context.**
The scheduled function runs between bytecodes shortly after, where
allocation is legal again:

```python
import micropython
micropython.alloc_emergency_exception_buf(100)   # readable tracebacks from ISRs

def process(arg):             # runs OUTSIDE the ISR — printing is fine here
    print("edge at", arg)

def on_edge(pin):
    micropython.schedule(process, time.ticks_ms())
```

The `alloc_emergency_exception_buf(100)` line reserves space so that if an
ISR *does* raise, you get a real traceback instead of a truncated one — put
it at the top of any program using interrupts.

!!! warning "Keep ISRs short"
    An ISR blocks everything else while it runs — including other
    interrupts. Do the minimum (set a flag, capture a timestamp, store one
    array element) and get out. Seconds of work in an ISR is a design bug,
    even where it's technically possible.

## Choosing: poll, timer, or interrupt?

| Situation | Tool |
|---|---|
| "Every N ms, do X" | `Timer` (or `uasyncio`, module 9) |
| Rare, timing-critical external event | `Pin.irq` |
| Reading a slow sensor "often enough" | plain polling with `ticks_ms()` |
| Many concurrent activities | `uasyncio` — module 9 |

## Cheat sheet

| Function / idiom | Purpose |
|---|---|
| `Timer(0).init(period=500, mode=Timer.PERIODIC, callback=f)` | Call `f` every 500 ms |
| `Timer.ONE_SHOT` | Fire once after `period` ms |
| `tim.deinit()` | Stop a timer |
| `pin.irq(trigger=Pin.IRQ_FALLING, handler=f)` | Call `f` on 1→0 edge |
| `Pin.IRQ_RISING`, `FALLING`, or both OR'd | Edge selection |
| ISR ban | No allocation: no prints, floats, list growth, str building |
| Flag pattern | ISR sets a global int/bool; main loop does the real work |
| `micropython.schedule(f, arg)` | Queue `f` to run soon, outside ISR context |
| `micropython.alloc_emergency_exception_buf(100)` | Readable ISR tracebacks |
| ISR debounce | Ignore edges < ~200 ms since last accepted |

## Exercise

Build a **reaction timer** in Wokwi: LED on pin 5, button on pin 4. A
one-shot `Timer` turns the LED on after a random 2–5 s delay
(`random.randint` — in the main code, not an ISR!). A `Pin.irq` falling-edge
handler captures `time.ticks_ms()` into a global the instant the button is
pressed. The main loop waits for the press, then prints the reaction time
in ms — computed with `ticks_diff` from when the LED lit. Pressing *before*
the LED lights prints "Too soon!" and restarts. Use the flag pattern —
no printing or float math in the handler. Play five rounds and print the
best time.
