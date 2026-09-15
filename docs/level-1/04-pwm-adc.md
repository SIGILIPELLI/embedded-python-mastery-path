---
description: "PWM & ADC — Digital pins know only 0 and 1, but the real world is analog — brightness, position, temperature, volume. Two peripherals bridge the gap: PWM…"
---

# 04 · PWM & ADC

Digital pins know only 0 and 1, but the real world is analog — brightness,
position, temperature, volume. Two peripherals bridge the gap: **PWM**
(pulse-width modulation) fakes an analog *output* by switching a pin fast
and varying the on-time, and the **ADC** (analog-to-digital converter) reads
a real analog *input* as a number. In MicroPython they're `machine.PWM` and
`machine.ADC`. This module dims LEDs, positions a servo, beeps a buzzer, and
reads a potentiometer — all runnable in a Wokwi *MicroPython on ESP32*
project.

## PWM: analog-ish output

PWM switches a pin between 0 and 3.3 V at a fixed frequency and varies the
**duty cycle** — the fraction of each period spent high. An LED driven at
1 kHz with 25% duty looks like it's at quarter brightness, because your eye
averages the flicker.

```python
from machine import Pin, PWM
import time

led = PWM(Pin(5), freq=1000)      # 1 kHz PWM on GPIO5

while True:
    for duty in range(0, 65536, 1024):      # fade up
        led.duty_u16(duty)                  # 0 = always off, 65535 = always on
        time.sleep_ms(10)
    for duty in range(65535, -1, -1024):    # fade down
        led.duty_u16(duty)
        time.sleep_ms(10)
```

**Wiring:** same as the blink circuit — pin 5 → 220 Ω → LED → GND.

`duty_u16()` takes 0–65535 and is the modern, portable API (identical on the
Pico). You'll also see `duty()` with 0–1023 in older ESP32 code — same
knob, coarser scale.

!!! tip "Human eyes are logarithmic"
    A linear duty ramp looks like it jumps to bright quickly and then barely
    changes. For a perceptually smooth fade, square the fraction:
    `led.duty_u16(int((i / steps) ** 2 * 65535))`.

### Servos: PWM at 50 Hz

A hobby servo reads a pulse every 20 ms (50 Hz) and turns its arm to an
angle set by the pulse width: ~0.5 ms → 0°, ~2.4 ms → 180°:

```python
from machine import Pin, PWM
import time

servo = PWM(Pin(18), freq=50)

def angle(deg):
    # 0.5 ms..2.4 ms pulse inside a 20 ms period, expressed as duty_u16
    us = 500 + (2400 - 500) * deg // 180
    servo.duty_u16(us * 65535 // 20000)

while True:
    for a in (0, 90, 180, 90):
        angle(a)
        time.sleep(1)
```

In Wokwi, add a **servo** part and connect its PWM (orange) wire to pin 18,
V+ to 3V3 (fine in simulation; real servos want their own 5 V supply), and
GND to GND — the horn visibly sweeps.

### Buzzers: PWM where frequency is the point

For a piezo buzzer, duty stays at ~50% and you vary the *frequency* to make
tones — `buzzer.freq(440)` is concert A. Set `duty_u16(0)` for silence.
Wokwi's buzzer part actually plays the tone through your speakers.

## ADC: reading analog inputs

The ESP32's ADC turns a voltage into a 12-bit number (0–4095). One
ESP32-specific detail: by default the ADC only ranges to ~1.1 V, so you set
**attenuation** to read the full 0–3.3 V span:

```python
from machine import ADC, Pin
import time

pot = ADC(Pin(34))                # GPIO 32-39 are the usual ADC pins
pot.atten(ADC.ATTN_11DB)          # full-range: ~0–3.3 V

while True:
    raw = pot.read()              # 0..4095
    volts = raw * 3.3 / 4095
    percent = raw * 100 // 4095
    print("raw={:4d}  {:.2f} V  {:3d}%".format(raw, volts, percent))
    time.sleep_ms(200)
```

**Wiring:** add a **potentiometer** in Wokwi — outer legs to 3V3 and GND,
middle (wiper) leg to pin 34. Turn the knob and watch the numbers track it.

Portability note: `pot.read_u16()` (0–65535) works on both ESP32 and Pico —
prefer it for code you'll move between boards. The Pico needs no `atten()`
call (its ADC natively spans 0–3.3 V).

!!! note "ADC readings are noisy"
    Real ADCs jitter by a few counts (the ESP32's more than most). The
    standard fix is averaging:
    `sum(pot.read() for _ in range(16)) // 16`. Wokwi's simulated pot is
    clean, but keep the habit.

### Closing the loop: pot dims LED

The classic first "system" — analog in controls analog out:

```python
from machine import Pin, PWM, ADC
import time

pot = ADC(Pin(34))
pot.atten(ADC.ATTN_11DB)
led = PWM(Pin(5), freq=1000)

while True:
    led.duty_u16(pot.read_u16())   # both are 16-bit: map directly
    time.sleep_ms(20)
```

### Scaling readings

Sensor math is constant rescaling; write the helper once:

```python
def scale(x, in_min, in_max, out_min, out_max):
    return (x - in_min) * (out_max - out_min) // (in_max - in_min) + out_min

angle_deg = scale(pot.read(), 0, 4095, 0, 180)   # pot position → servo angle
```

## How It Actually Works

Neither PWM nor the ADC are things Python "does" — they are dedicated
silicon peripherals on the ESP32 die that MicroPython merely configures and
reads; the interpreter is barely in the loop once they're running.

- **PWM runs entirely in hardware, independent of your Python code's
  speed.** `PWM(Pin(5), freq=1000)` loads a divider and a compare value into
  the LEDC (LED Controller) peripheral's registers — a small timer/comparator
  circuit that toggles the pin's output driver on its own, driven by the
  chip's clock, with zero CPU involvement after setup. This is precisely why
  `led.duty_u16(duty)` inside a `time.sleep_ms(10)` loop still produces a
  perfectly clean, glitch-free 1 kHz waveform even though the *Python* loop
  driving the fade is running orders of magnitude slower than 1 kHz — the
  peripheral, not the interpreter, is the thing keeping time at 1 kHz.
  `duty_u16()` just writes a new compare value; the hardware comparator does
  the actual switching between calls.
- **The servo pulse is the same peripheral, reinterpreted.** There's no
  separate "servo mode" in the ESP32 — a servo signal is just PWM at 50 Hz
  with the duty cycle chosen so the *absolute pulse width* (not the
  percentage) falls in the 0.5–2.4 ms window the servo's own internal
  circuitry expects. The math in `angle()` is converting from "percentage of
  a 20 ms period" (what the LEDC hardware wants) to "microseconds of high
  time" (what the servo's decoder actually measures) — two different mental
  models of the exact same electrical signal.
- **The ADC is a successive-approximation converter, and `atten()` changes
  an analog reference voltage, not a software scale factor.** Internally,
  the ESP32's SAR ADC compares the input voltage against an internal
  reference through a binary-search-like process (12 comparisons for a
  12-bit result), completing in microseconds. `ATTN_11DB` doesn't rescale
  the *number* you get back in software — it switches in an analog
  attenuator ahead of the comparator so a 3.3 V input actually lands within
  the converter's native ~1.1 V comparison window. Skip `atten()` and a pin
  at 3.3 V will read as if it were near 1.1 V territory, clipped — a
  hardware ceiling no amount of Python-side math can fix after the fact.
  Read noise (the jitter this module tells you to average away) comes from
  real analog sources — reference voltage ripple, the SAR's own comparator
  noise, capacitive coupling from a switching PWM signal on a nearby pin —
  not from anything the interpreter introduces.

## Cheat sheet

| Function / idiom | Purpose |
|---|---|
| `PWM(Pin(n), freq=1000)` | Create PWM output on pin *n* |
| `pwm.duty_u16(0..65535)` | Set duty cycle (portable ESP32/Pico API) |
| `pwm.freq(hz)` | Change frequency (tones: the note; LEDs: keep ≥ ~200 Hz) |
| `pwm.deinit()` | Release the pin back to plain GPIO |
| Servo | `freq=50`, pulse 0.5–2.4 ms mapped into `duty_u16` |
| `ADC(Pin(34))` | Create ADC reader (ESP32: use GPIO 32–39) |
| `adc.atten(ADC.ATTN_11DB)` | ESP32 only: extend range to ~3.3 V |
| `adc.read()` | 0–4095 (12-bit, ESP32) |
| `adc.read_u16()` | 0–65535 — portable across ports |
| Noise | Average 8–16 readings |
| `scale()` helper | Map a reading from one range to another |

## Exercise

Build a **mood lamp with a manual override** in Wokwi: an LED on pin 5
(PWM), a potentiometer on pin 34, and a button on pin 4 (pull-up). In
*auto* mode the LED "breathes" — fades up and down continuously using the
perceptual (squared) curve. Pressing the button (debounced, from module 3)
switches to *manual* mode, where the pot directly sets brightness; pressing
again returns to auto. Print the mode and the current duty percentage on
every change. Bonus: add a servo on pin 18 whose arm acts as an analog
"brightness gauge", pointing 0–180° proportional to the current duty.
