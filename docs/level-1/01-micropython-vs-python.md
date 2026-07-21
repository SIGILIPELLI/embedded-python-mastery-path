# 01 · MicroPython vs. Python

MicroPython is a lean reimplementation of Python 3 that runs *directly on a
microcontroller* — no operating system, no gigabytes of RAM, just your code,
an interpreter, and the chip's registers. You write ordinary Python syntax
(`def`, `class`, `try/except`, list comprehensions, f-strings all work), but
the runtime fits in a few hundred kilobytes and starts in milliseconds. This
module maps out what's the same, what's missing, what's *added*, which boards
run it, and how to start writing code in the next five minutes without buying
any hardware.

## Same language, much smaller world

MicroPython implements the Python 3.4 core language (with many newer
features, like f-strings and `async`/`await`, added since). The differences
are almost all about *scale*:

| | CPython (desktop) | MicroPython (ESP32) |
|---|---|---|
| RAM available to your code | gigabytes | ~100 KB |
| Storage | SSD/HDD, terabytes | ~2 MB flash filesystem |
| Standard library | ~300 modules | a curated subset (~30 built in) |
| Package install | `pip` from PyPI | `mip` from micropython-lib |
| Integers | arbitrary precision | arbitrary precision (same!) |
| Threads/processes | full support | limited; `uasyncio` preferred |
| Startup | interpreter on an OS | *is* the whole system, boots in ms |

The subset stdlib is the biggest day-one surprise. There is no `pandas`, no
`tkinter`, no `subprocess` — but there is `json`, `os`, `sys`, `time`,
`struct`, `math`, `random`, `re` (simplified), and `socket`. Historically the
trimmed modules had `u`-prefixed names (`ujson`, `uos`, `utime`); modern
MicroPython exposes them under the standard names, and you'll still see the
`u` names in older tutorials — both work.

```python
import sys
print(sys.implementation)   # (name='micropython', version=(1, 22, ...), ...)
print(sys.platform)         # 'esp32'

import ujson, json          # same module, two names
print(json is ujson)        # True
```

What MicroPython *adds* is the interesting part — modules that don't exist on
the desktop because desktops don't have pins:

```python
import machine     # Pin, PWM, ADC, I2C, SPI, UART, Timer, RTC, deepsleep...
import network     # WLAN — connect the chip itself to WiFi
import micropython # interpreter controls: const(), mem_info(), schedule()
import gc          # the garbage collector, which you WILL meet on 100 KB
gc.collect()
print(gc.mem_free())   # e.g. 108816 — bytes of heap left. Budget it.
```

That last line captures the mindset shift: on the desktop you never think
about a 100-item list; on a microcontroller, building a big string in a loop
can genuinely exhaust memory. Level 3 goes deep on this; for now just know
`gc.mem_free()` exists.

## Boards that run MicroPython

- **ESP32** (our primary board) — dual-core 240 MHz, WiFi + Bluetooth,
  ~$5. The best all-rounder and what Wokwi simulates.
- **Raspberry Pi Pico / Pico W (RP2040)** — 133 MHz dual-core, famous for
  the PIO programmable-I/O blocks (a Level 3 topic). Same MicroPython API;
  mostly just different pin numbers.
- **ESP8266** — the ESP32's older, smaller sibling; runs MicroPython but
  with tight RAM.
- **Pyboard (STM32)** — the original MicroPython board.

Code in this course is ESP32-flavored (pin numbers, `ADC.atten()`), and we
flag the places where the Pico differs.

## CircuitPython, in brief

**CircuitPython** is Adafruit's friendly fork of MicroPython. Same language,
different hardware API: instead of `machine.Pin(2, Pin.OUT)` you write
`digitalio.DigitalInOut(board.LED)`, and the board mounts as a USB drive —
you copy `code.py` onto it and it just runs. It's a lovely beginner
experience with huge driver support for Adafruit's sensors, but it drops some
MicroPython features (like `machine`, `_thread`, and interrupt handlers).
Everything conceptual in this course transfers; the API names don't. We stick
with upstream MicroPython because it's what Wokwi simulates and what
production ESP32 work typically uses.

## Getting MicroPython onto a real board

A fresh ESP32 ships with no Python on it — you flash the MicroPython
*firmware* (a `.bin` file from
[micropython.org/download](https://micropython.org/download/)) once, and from
then on the board speaks Python over USB. The tool is `esptool`:

```bash
pip install esptool
# 1. wipe the flash (once, recommended for a new board)
esptool.py --port /dev/ttyUSB0 erase_flash
# 2. write the MicroPython firmware
esptool.py --port /dev/ttyUSB0 --baud 460800 write_flash 0x1000 ESP32_GENERIC-20240602-v1.23.0.bin
```

On Windows the port looks like `COM3`; on macOS like `/dev/tty.usbserial-0001`.
The Raspberry Pi Pico is even easier: hold BOOTSEL while plugging in, and
drag a `.uf2` file onto the USB drive that appears. Flashing is a one-time
setup step — afterwards you only ever copy `.py` files around (module 2).

## The zero-hardware path: Wokwi

You don't need a board for any of Level 1. [Wokwi](https://wokwi.com/) is a
free browser-based simulator with first-class MicroPython support:

1. Go to [wokwi.com](https://wokwi.com/), click **New Project**.
2. Choose **MicroPython on ESP32** from the starter templates.
3. You get a code editor (`main.py`), a virtual ESP32, and a play button.

Press play and the on-screen serial console *is* a live Python REPL on the
simulated chip. Try it right now with a first program:

```python
# main.py — runs on the simulated ESP32 in Wokwi
import sys
import gc

print("Hello from MicroPython!")
print("platform:", sys.platform)
print("free heap:", gc.mem_free(), "bytes")
```

You can add virtual parts (LEDs, buttons, DHT22 sensors, OLED displays) from
the **+** button in the diagram pane and wire them with drag-and-drop —
every later module tells you exactly what to add. The same code runs
unchanged on a real ESP32.

## Cheat sheet

| Concept | The short version |
|---|---|
| MicroPython | Python 3 reimplemented to run on microcontrollers in ~256 KB |
| stdlib | Subset: `json`, `os`, `time`, `struct`, `socket`... — no big desktop libs |
| `u`-modules | Old names (`ujson`, `utime`) — now aliases for the plain names |
| `machine` module | The hardware API: `Pin`, `PWM`, `ADC`, `I2C`, `SPI`, `Timer` |
| `gc.mem_free()` | Bytes of heap remaining — memory is a real budget here |
| Boards | ESP32 (primary), Raspberry Pi Pico (RP2040), ESP8266, Pyboard |
| CircuitPython | Adafruit fork: `board`/`digitalio` API, USB-drive workflow, no `machine` |
| Firmware flashing | Once per board: `esptool.py write_flash 0x1000 firmware.bin` (ESP32) |
| Zero hardware | Wokwi → New Project → **MicroPython on ESP32** — free, in-browser |
| Package install | `mip` (on-device) from micropython-lib, not `pip` |

## Exercise

Create a Wokwi *MicroPython on ESP32* project and write a `main.py` that
prints: (1) `sys.platform` and `sys.implementation.version`, (2) the free
heap from `gc.mem_free()`, then (3) builds a list of 1,000 integers, calls
`gc.collect()`, and prints the free heap again. How many bytes did 1,000
small ints cost? Then try `import this` and `import tkinter` — one works,
one raises `ImportError`. Finally, use `dir(machine)` to list everything the
hardware module offers and spot the class names you'll meet in modules 3–5
(`Pin`, `PWM`, `ADC`, `Timer`, `I2C`).
