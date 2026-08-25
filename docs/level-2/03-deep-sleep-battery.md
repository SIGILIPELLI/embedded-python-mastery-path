# Deep Sleep & Battery Power

An ESP32 running full-tilt draws 80-100+ mA — a fresh 18650 battery lasts
a day, maybe two. The same board in **deep sleep** draws single-digit
microamps, stretching that to months. `machine.deepsleep()` is how
battery-powered MicroPython projects survive: wake on a timer or a pin,
do the minimum work, sleep again. This module covers deep sleep, wake
sources, surviving a full reset with RTC memory, measuring sleep current,
and budgeting battery life — logic and math run via `python3`; the
`machine`/`esp32` sleep APIs are reviewed against MicroPython docs since
Wokwi doesn't model microamp-level current draw.

## Deep sleep is not a pause

This is the single most important fact in this module: **deep sleep resets
the chip.** Your program does not resume where it left off — it restarts
from the top, exactly like a power-cycle. Variables, open sockets, WiFi
connections, everything in RAM: gone.

```python
import machine
import time

print("Awake! Doing work...")
time.sleep(1)                      # stand-in for real sensor work

print("Going to sleep for 30 seconds")
machine.deepsleep(30_000)          # milliseconds; board resets after this
# nothing after this line ever runs — the reset happens inside the call
```

Compare `light sleep` (`machine.lightsleep()`), which *does* preserve RAM
and resumes execution after the call returns — much lower savings, but no
state loss. Deep sleep is the big win; light sleep is the "I still need my
variables" compromise.

## Wake sources

### Timer (most common)

```python
import machine

machine.deepsleep(60_000)          # wake after 60 s, no other source needed
```

### External pin (EXT0 — single pin)

```python
import machine
from machine import Pin

# Wake when pin 27 goes HIGH (a button pulling it up, for example)
wake_pin = Pin(27, Pin.IN, Pin.PULL_DOWN)
esp32.wake_on_ext0(pin=wake_pin, level=esp32.WAKEUP_ANY_HIGH)
machine.deepsleep()
```

(`import esp32` for the `esp32.wake_on_ext0`/`wake_on_ext1`/`WAKEUP_*`
names.) EXT1 wakes on any of several pins at once, useful for multiple
buttons:

```python
import esp32
from machine import Pin

pins = (Pin(27, Pin.IN, Pin.PULL_DOWN), Pin(33, Pin.IN, Pin.PULL_DOWN))
esp32.wake_on_ext1(pins=pins, level=esp32.WAKEUP_ANY_HIGH)
machine.deepsleep()
```

### Touch pins

```python
import esp32
esp32.wake_on_touch(True)          # combine with configured TouchPad(s)
machine.deepsleep()
```

## Finding out why you woke up

Since the program restarts from scratch every time, check the wake reason
at the top of your script to branch behavior:

```python
import machine

reset_cause = machine.reset_cause()
wake_reason = machine.wake_reason()

if reset_cause == machine.DEEPSLEEP_RESET:
    print("woke from deep sleep, reason:", wake_reason)
    if wake_reason == machine.PIN_WAKE:
        print("woken by a pin")
    elif wake_reason == machine.TIMER_WAKE:
        print("woken by the timer")
else:
    print("normal power-on / hard reset")
```

## Surviving the reset: RTC memory

A small chunk of RAM survives deep sleep (though not a full power loss):
`machine.RTC().memory()`. It only stores bytes, so counters/small state
need encoding:

```python
import machine

rtc = machine.RTC()

def load_count():
    data = rtc.memory()
    return int(data) if data else 0

def save_count(n):
    rtc.memory(str(n).encode())

count = load_count() + 1
print("boot count:", count)
save_count(count)

machine.deepsleep(10_000)
```

For structured state, `ujson.dumps()`/`loads()` into that same byte
buffer works, as long as it fits (RTC memory is small — a few hundred
bytes budget, not KB).

## Battery budgeting

Rough math, done here in plain Python since it's just arithmetic:

```python
def battery_life_hours(battery_mah, active_ma, active_s, sleep_ua, sleep_s):
    cycle_s = active_s + sleep_s
    active_mah_per_cycle = active_ma * (active_s / 3600)
    sleep_mah_per_cycle = (sleep_ua / 1000) * (sleep_s / 3600)
    mah_per_cycle = active_mah_per_cycle + sleep_mah_per_cycle
    cycles = battery_mah / mah_per_cycle
    return cycles * cycle_s / 3600

# 2000 mAh battery, 120 mA while awake for 2 s, 10 uA asleep for 58 s
hours = battery_life_hours(2000, 120, 2, 10, 58)
print(f"{hours:.0f} hours (~{hours/24:.0f} days)")
```

The lesson the numbers teach: active current dominates unless the duty
cycle (awake time / total cycle time) is tiny. Waking every 60 s for 2 s of
work (3.3% duty cycle) still costs far more battery than waking every 10
minutes for the same 2 s — sleep *longer*, not just *lower-power*, when
the application allows it.

## Deep sleep gotchas

!!! warning "Global setup code runs every wake, not once"
    Anything at module level — WiFi connect, sensor init, driver imports —
    runs again after *every* wake, because the "boot" is a full restart.
    Budget the time and current cost of your setup path; a 3-second WiFi
    reconnect every wake can dwarf the sleep savings you're chasing.

!!! warning "Peripherals lose power state"
    Deep sleep can power down most of the chip, including GPIO drive in
    some configurations. Don't assume a pin you set HIGH before sleeping
    is still HIGH after waking — reconfigure peripherals from scratch each
    boot.

!!! warning "`deepsleep()` with no argument sleeps forever (until a wake source fires)"
    `machine.deepsleep()` with no duration only wakes via a configured
    `esp32.wake_on_*` source — if you forgot to configure one, the board
    sleeps until manually reset. Always pass a timer duration as a
    fallback, or double-check a wake source is armed.

!!! warning "USB serial goes away during sleep"
    You will not see `print()` output appear "live" during a sleep/wake
    cycle over USB in Wokwi or on some real boards — the console
    reconnects after the reset. Log to RTC memory or flash if you need to
    debug a stretch of missed wakeups.

## Cheat sheet

| Function / idiom | Purpose |
|---|---|
| `machine.deepsleep(ms)` | Sleep and reset; program restarts from the top |
| `machine.lightsleep(ms)` | Sleep, keep RAM, resume after the call |
| `machine.reset_cause()` | `DEEPSLEEP_RESET`, `PWRON_RESET`, etc. |
| `machine.wake_reason()` | `PIN_WAKE`, `TIMER_WAKE`, `TOUCHPAD_WAKE` |
| `esp32.wake_on_ext0(pin, level)` | Wake on one pin |
| `esp32.wake_on_ext1(pins, level)` | Wake on any of several pins |
| `esp32.wake_on_touch(True)` | Wake on a touch pin |
| `machine.RTC().memory(bytes)` / `.memory()` | Small state store that survives deep sleep |
| duty cycle = active_s / (active_s + sleep_s) | Drives battery life more than raw sleep current |

## Exercise

Build a **battery-aware logger** (logic testable in plain `python3`, sleep
calls reviewed against docs): on each boot, read `machine.reset_cause()`
and `machine.RTC().memory()` to recover a boot counter (0 if this is a
cold boot). Print `"boot #N, cause=..., reason=..."`. Simulate a sensor
read, save the incremented counter to RTC memory, then call
`machine.deepsleep(15_000)`. Separately, write a plain-Python function
`battery_life_hours(battery_mah, active_ma, active_s, sleep_ua, sleep_s)`
(as above) and use it to compare three duty cycles — wake every 15 s, every
60 s, and every 10 minutes, same 2 s of active work each time — printing
projected battery life in days for a 2000 mAh cell at each interval.
