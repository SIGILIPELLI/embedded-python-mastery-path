---
description: "REPL & Workflow — The thing that makes MicroPython development fast is the REPL: a live Python prompt running on the chip itself. You can toggle a pin…"
---

# 02 · REPL & Workflow

The thing that makes MicroPython development *fast* is the REPL: a live
Python prompt running on the chip itself. You can toggle a pin, read a
sensor, or test a function interactively — no compile, no flash, no
30-second upload cycle like the C/C++ world. This module covers the REPL,
its keyboard shortcuts, the board's `boot.py`/`main.py` startup sequence,
and the three main ways to get code onto a board: Thonny, `mpremote`, and
VS Code. In Wokwi, the REPL appears automatically in the serial console when
you press play — everything below works there too.

## The serial REPL

A MicroPython board presents a serial port over USB (115200 baud). Connect
any serial terminal — or just use the tools below — and you get:

```
MicroPython v1.23.0 on 2024-06-02; Generic ESP32 module with ESP32
Type "help()" for more information.
>>> 2 + 2
4
>>> from machine import Pin
>>> led = Pin(2, Pin.OUT)
>>> led.value(1)          # the board's LED turns on. Right now. Interactively.
```

This is real hardware responding to each line as you type it. Four control
keys matter — memorize them:

| Key | Action |
|-----|--------|
| `Ctrl-C` | Interrupt the running program, back to `>>>` |
| `Ctrl-D` | **Soft reset** — restart the interpreter, re-run `boot.py`/`main.py` |
| `Ctrl-E` | **Paste mode** — paste multi-line code without auto-indent mangling it, finish with `Ctrl-D` |
| `Ctrl-A` / `Ctrl-B` | Raw REPL (used by tools) / back to friendly REPL |

A *soft* reset (`Ctrl-D`) restarts Python but keeps the board powered — the
filesystem survives, and it's how you re-run `main.py` after editing it. A
*hard* reset (the physical EN/RST button, or `machine.reset()`) reboots the
whole chip.

## boot.py and main.py

The board has a small flash filesystem (module 7 explores it). On startup,
MicroPython automatically runs two files from it, in order:

1. **`boot.py`** — early setup, runs first. Keep it minimal (or empty);
   a crash here can make the board hard to talk to.
2. **`main.py`** — your application. This is the file that makes a board
   run your program on power-up, with no computer attached.

While developing you'll usually `Ctrl-C` out of `main.py` to get a prompt,
edit, then `Ctrl-D` to run it again. In Wokwi, the editor's `main.py` *is*
the board's `main.py` — press play to run it, and click into the serial
console and press `Ctrl-C` to drop to the REPL.

```python
# main.py — a minimal auto-starting program
from machine import Pin
import time

led = Pin(2, Pin.OUT)
while True:
    led.value(not led.value())
    time.sleep(0.5)
```

## Workflow 1: Thonny (easiest start)

[Thonny](https://thonny.org/) is a free Python IDE with built-in MicroPython
support. Install it, then: **Tools → Options → Interpreter → MicroPython
(ESP32)**, pick the serial port. Now the bottom pane is the board's REPL,
the **Run** button executes the open file on the board, and **File → Save
as… → MicroPython device** writes files to the board's flash. It can even
flash the firmware for you. If you're on real hardware, start with Thonny.

## Workflow 2: mpremote (the official CLI)

`mpremote` is MicroPython's official command-line tool — the one to learn
for scripting and serious work:

```bash
pip install mpremote

mpremote                          # open a REPL (auto-detects the port)
mpremote run blink.py             # run a local file on the board (not saved to it)
mpremote fs ls                    # list the board's filesystem
mpremote fs cp main.py :main.py   # copy local file TO the board (: = board side)
mpremote fs cp :data.csv data.csv # copy FROM the board
mpremote reset                    # reset the board
```

Commands chain with `+`, which enables the classic edit-upload-run loop as
one shell command:

```bash
mpremote fs cp main.py :main.py + reset
```

The difference between `run` and `cp` matters: `run` executes a file from
your computer *without* storing it (great for experiments); `cp` writes it
to flash so it survives resets and runs on power-up.

## Workflow 3: VS Code

Prefer VS Code? The **MicroPico** extension (for the Pico) and the
**Pymakr** extension (ESP32) give you upload/run/REPL inside the editor. Or
keep it simple: edit in VS Code, and use `mpremote` in the integrated
terminal — many professionals do exactly that.

## Uploading a multi-file project

Real projects are several files — say a driver module and a main script:

```bash
mpremote fs cp sensors.py :sensors.py
mpremote fs cp main.py :main.py
mpremote reset
```

On the board, `import sensors` now works exactly like desktop Python —
the flash filesystem is on `sys.path`. In Wokwi, click the arrow next to
the file tab to add new files to the project; they appear on the simulated
board's filesystem automatically.

!!! tip "Escaping a runaway main.py"
    If `main.py` has an infinite loop (most embedded programs do), the board
    seems "busy" every time it boots. That's normal: connect, hit `Ctrl-C`
    to interrupt it, and you have a REPL. In the worst case (a crash-loop so
    fast you can't break in), re-flash the firmware or use
    `mpremote fs rm :main.py` the instant after a reset.

## How It Actually Works

The REPL feels instant because there is no build step standing between it and
the chip — but there's real machinery underneath the friendly `>>>` prompt:

- **The REPL is just another script running in the VM.** After boot, the
  firmware calls into a small C function that reads a line, feeds it to the
  same compiler `import` uses, compiles it to bytecode on the fly, and runs
  it on the same interpreter loop as `main.py`. There's no separate
  "interactive mode" binary — `led.value(1)` you typed by hand and
  `led.value(1)` sitting in a file take an identical path through the VM.
  That's why the REPL can `import` your own modules and call functions
  defined in `main.py` if you `Ctrl-C` into it: the module objects are still
  live on the heap.
- **`Ctrl-C` works because MicroPython polls for it between bytecode
  instructions.** The USB/UART driver sets a flag on interrupt when it sees
  byte `0x03`; the VM's bytecode dispatch loop checks that flag periodically
  and raises `KeyboardInterrupt` at the next safe point — it is *not* an
  OS-level signal (there's no OS to deliver one). This is also why a tight
  loop with no I/O and no `time.sleep()` can occasionally feel slow to
  interrupt: the check only happens between opcodes, and a single line like
  a giant list comprehension can run many opcodes before yielding.
- **A soft reset (`Ctrl-D`) reruns boot without power-cycling the chip.** It
  discards the current heap, re-initializes the VM's global state, then
  re-executes `boot.py` and `main.py` from the filesystem — but it does
  *not* reset peripherals the hardware itself remembers (a GPIO left driven
  high by `machine.Pin` can stay high across a soft reset since the pin
  controller's register state isn't touched). A hard reset via `EN`/`RST` or
  `machine.reset()` does a full chip reset, which does clear peripheral
  state — a distinction that matters when you're debugging "why is the LED
  still on after Ctrl-D."
- **`mpremote` talks two different protocols to the same UART.** The
  friendly REPL you type into interactively is a plain terminal echo loop.
  `mpremote fs cp` and `run` instead drop the board into "raw REPL" mode
  (`Ctrl-A`) — a machine-oriented protocol with explicit start/end markers
  and no echo, so the host tool can send exact bytes (including a whole
  file's contents) and reliably tell when execution finished. That's the
  actual meaning of the `Ctrl-A`/`Ctrl-B` row in the cheat sheet below:
  raw REPL is what automation needs, friendly REPL is what humans need, and
  they're the same interpreter behind two different framing protocols.

## Cheat sheet

| Command / key | Purpose |
|---|---|
| `Ctrl-C` | Interrupt running code → REPL prompt |
| `Ctrl-D` (at REPL) | Soft reset — reruns `boot.py` then `main.py` |
| `Ctrl-E` … `Ctrl-D` | Paste mode for multi-line code |
| `boot.py` | Runs first at startup — keep minimal |
| `main.py` | Your app — auto-runs on every boot |
| `mpremote` | Open REPL on auto-detected board |
| `mpremote run file.py` | Execute local file on board without saving |
| `mpremote fs cp f.py :f.py` | Upload file to board flash (`:` = board) |
| `mpremote fs ls` / `rm` | List / delete board files |
| `machine.reset()` | Hard reset from code |
| Thonny | GUI IDE: REPL + file manager + firmware flasher |

## Exercise

In Wokwi (or on a real board): (1) start the blink `main.py` from above,
then click the serial console and press `Ctrl-C` — confirm you get a `>>>`
prompt and the blinking stops. (2) At the REPL, turn the LED on manually
with `led.value(1)`. (3) Press `Ctrl-D` and watch the soft reset banner and
the blinking resume. (4) Add a second file `greet.py` containing
`def hello(name): return "Hello, " + name`, and from the REPL run
`import greet; greet.hello("ESP32")`. (5) On real hardware, do the upload
with `mpremote fs cp greet.py :greet.py` and verify with `mpremote fs ls`.
