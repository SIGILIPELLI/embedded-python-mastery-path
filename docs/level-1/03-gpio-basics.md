---
description: "GPIO Basics — GPIO — general-purpose input/output — is the foundation of everything a microcontroller does: each pin can be driven high or low (output) or…"
---

# 03 · GPIO Basics

GPIO — *general-purpose input/output* — is the foundation of everything a
microcontroller does: each pin can be driven high or low (output) or read
(input). In MicroPython all of it goes through one class, `machine.Pin`.
This module covers driving LEDs, reading buttons with pull-ups, debouncing —
the first genuinely *embedded* problem you'll solve in Python — and a
traffic-light exercise to lock it in.

All circuits run in the [Wokwi](https://wokwi.com/) **MicroPython on ESP32**
project type: click **+** in the diagram pane, add an LED, a resistor, and a
pushbutton, and drag wires between pin holes exactly as described.

## Outputs: driving an LED

```python
from machine import Pin
import time

led = Pin(5, Pin.OUT)      # GPIO5 configured as an output

while True:
    led.value(1)           # 3.3 V on the pin — LED on
    time.sleep(0.25)
    led.value(0)           # 0 V — LED off
    time.sleep(0.25)
```

**Wiring:** ESP32 pin 5 → resistor (220 Ω) → LED anode (the bent leg in
Wokwi); LED cathode → GND.

!!! warning "Always use a current-limiting resistor"
    An LED will draw as much current as the supply allows and burn out —
    possibly damaging the pin, which can only source a few tens of mA. A
    220–330 Ω resistor in series keeps it safe. Wokwi is forgiving about
    this; real hardware is not, so build the habit in the simulator.

`Pin` gives you several equivalent spellings — pick one and be consistent:

```python
led.on()                     # same as led.value(1)
led.off()                    # same as led.value(0)
led.value(not led.value())   # read-modify-write: toggle current state
```

(On the Raspberry Pi Pico the on-board LED is `Pin("LED", Pin.OUT)` and the
port also offers `led.toggle()` — same idea, different sugar.)

## Inputs: reading a button

A pushbutton connects two points when pressed. The subtlety is what the pin
sees when the button is **not** pressed: connected to nothing, the pin
*floats* and reads random electrical noise. Every digital input needs a
defined idle state, provided by a **pull-up** or **pull-down** resistor —
and the chip has them built in, enabled from Python:

```python
from machine import Pin

button = Pin(4, Pin.IN, Pin.PULL_UP)   # internal resistor holds pin high when idle
led = Pin(5, Pin.OUT)

while True:
    # With PULL_UP the logic is inverted: not pressed → 1, pressed → 0
    led.value(0 if button.value() else 1)
```

**Wiring:** one side of the button → pin 4, the other side → GND. That's
all — the internal pull-up replaces an external resistor.

| Configuration | Idle reads | Pressed reads | Wiring |
|---------------|-----------|---------------|--------|
| `Pin.PULL_UP` (most common) | `1` | `0` | button between pin and **GND** |
| `Pin.PULL_DOWN` | `0` | `1` | button between pin and **3V3** |
| No pull, nothing attached | random noise | — | never do this |

The inverted logic of pull-ups (`0` = pressed) trips up every beginner
once. Hide it behind a well-named helper:

```python
def pressed(pin):
    return pin.value() == 0
```

!!! note "ESP32 pin gotchas"
    A few ESP32 pins are special: GPIO 34–39 are **input-only** (fine for
    buttons and ADC, no LEDs), and GPIO 0/2/12/15 are sampled at boot to
    select boot modes — avoid hanging buttons on those. GPIO 4, 5, 16–19,
    21–23, 25–27, 32, 33 are safe everyday choices.

## Debouncing

Mechanical button contacts don't close cleanly — they *bounce*, making and
breaking contact for a few milliseconds. Python on a 240 MHz chip is easily
fast enough to see every bounce, so naive counting code counts one press as
several:

```python
# BUGGY: counts bounces, not presses
if pressed(button):
    count += 1
```

The fix: after a change, ignore further changes until the reading has been
stable for ~20–50 ms. Here's the classic non-blocking debounce with **edge
detection** (act once per press, not continuously while held). Note
`time.ticks_ms()` and `time.ticks_diff()` — MicroPython's overflow-safe
milliseconds clock, the correct tool for *all* elapsed-time math on a board:

```python
from machine import Pin
import time

button = Pin(4, Pin.IN, Pin.PULL_UP)
led = Pin(5, Pin.OUT)

DEBOUNCE_MS = 30
led_on = False
last_reading = 1        # raw reading last time we looked
stable_state = 1        # debounced, "official" button state
last_change = time.ticks_ms()

while True:
    reading = button.value()

    if reading != last_reading:
        last_change = time.ticks_ms()   # signal changed — restart stability timer
        last_reading = reading

    if (time.ticks_diff(time.ticks_ms(), last_change) > DEBOUNCE_MS
            and reading != stable_state):
        stable_state = reading
        if stable_state == 0:           # falling edge = the moment of the press
            led_on = not led_on         # toggle once per press
            led.value(led_on)
            print("toggled:", "on" if led_on else "off")
```

Press the button: the LED toggles on. Press again: off. There is no
`time.sleep()` anywhere — the loop spins freely, which matters as soon as
the program does more than one thing (modules 5 and 9 build on this).

Wokwi's virtual button can bounce realistically: select it and set
`"bounce": "1"` in its properties, then try the buggy counter and watch it
miscount, exactly like real hardware.

## How It Actually Works

`Pin(5, Pin.OUT)` looks like it creates a "pin object," but what it actually
does is configure a handful of bits in the ESP32's **GPIO matrix** and
**IO_MUX** peripheral registers, and every call after that is a direct
memory-mapped register write — there is no OS driver layer in between:

- **`Pin(5, Pin.OUT)` writes to the GPIO_ENABLE register.** Each GPIO pin on
  the chip has a dedicated bit in a 32-bit "enable" register bank that
  decides whether the pin's output driver is connected at all. Constructing
  the `Pin` object in MicroPython sets that bit (and configures the IO_MUX
  to route the pin's *function* to plain GPIO rather than, say, UART or
  I2C — modules 4 and 6 revisit that muxing). This happens once, at
  construction time, which is why re-creating `Pin` objects in a hot loop is
  wasted work: keep the object and just call `.value()`.
- **`led.value(1)` is a single-instruction register write under the hood.**
  On the ESP32, output pins 0–31 share one 32-bit "write 1 to set" register
  (`GPIO_OUT_W1TS`) and a matching "write 1 to clear" register
  (`GPIO_OUT_W1TC`) — writing a `1` to bit 5 of `W1TS` sets GPIO5 high
  without disturbing any other pin's state, and the same bit in `W1TC`
  clears it. `machine.Pin.value()` is a thin Python wrapper around exactly
  that: one 32-bit store to a fixed physical address. The "3.3 V on the pin"
  comment in the code above is not a metaphor — that store happens, and
  microseconds later the transistor-level output driver on the die actually
  changes voltage.
- **A floating input isn't "random" by accident — it's an unbiased,
  extremely high-impedance node picking up whatever capacitive coupling and
  electrical noise is nearby.** The pull-up/pull-down resistors you enable
  in software (`Pin.PULL_UP`) are physically tiny (~45 kΩ on ESP32) resistors
  built into the pin's analog front end, connected or disconnected by
  another register bit — `Pin.PULL_UP` doesn't change any logic in the VM,
  it changes the analog circuit the digital reading is sampled from.
- **Debouncing is a software workaround for a hardware fact the VM can't
  hide.** The interpreter reads `button.value()` by loading the current
  state of the GPIO_IN register — a snapshot of real voltage sampled by a
  Schmitt-trigger input buffer, updated continuously by hardware regardless
  of how often or rarely your Python loop asks. Mechanical bounce is a
  physical property of the switch contact, not of MicroPython, which is why
  no amount of "faster Python" fixes it — only time-based filtering
  (`ticks_ms`/`ticks_diff`) does, because the fix has to live in the time
  domain the bounce actually occurs in (a few milliseconds), independent of
  how fast the interpreter itself can loop.

## Cheat sheet

| Function / idiom | Purpose |
|---|---|
| `Pin(n, Pin.OUT)` | Configure GPIO *n* as output |
| `Pin(n, Pin.IN, Pin.PULL_UP)` | Input with internal pull-up — idle `1`, pressed `0` |
| `p.value(1)` / `p.value(0)` | Drive high / low (also `p.on()` / `p.off()`) |
| `p.value()` | Read the pin (returns `0` or `1`) |
| `p.value(not p.value())` | Toggle |
| LED wiring | pin → 220 Ω resistor → LED anode; cathode → GND |
| Button wiring (pull-up) | pin → button → GND; pressed reads `0` |
| Floating input | Unconnected input reads noise — always define idle state |
| `time.ticks_ms()` / `ticks_diff()` | Overflow-safe elapsed-time math |
| Bounce | Contacts chatter ~1–10 ms; require ~30 ms of stability |
| ESP32 pin notes | 34–39 input-only; avoid 0/2/12/15 for buttons |

## Exercise

Build a **traffic light** in Wokwi: red, yellow, and green LEDs on pins 25,
26, 27 (each with a 220 Ω resistor to GND) and a button on pin 4. Normal
cycle: green 4 s → yellow 1 s → red 4 s → repeat, written *without* long
`sleep()` calls — use `ticks_ms()`/`ticks_diff()` so the button stays
responsive. The button is a pedestrian request: when pressed (debounced,
edge-detected) during green, shorten the remaining green to 1 s before going
yellow. Print each state change over serial. Bonus: blink the red LED during
its last second as a "get ready" hint.
