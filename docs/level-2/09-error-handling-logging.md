# Error Handling & On-Device Logging

A laptop-connected script that crashes just prints a traceback and stops.
A weather station bolted to a fence post that crashes needs to recover on
its own, and needs to leave a trail so you can find out *why* it crashed
next time you visit. This module covers exception handling patterns for
unattended devices, `sys.print_exception`, rotating log files on flash,
capturing tracebacks across resets, and `machine.reset_cause()`. Logging
logic and file-rotation code run and are verified via `python3`;
`machine.reset_cause()`/watchdog specifics are reviewed against
MicroPython docs.

## The unattended-device mindset

The core shift from a desktop script: nobody is watching. Every failure
mode needs an automatic answer, not a human pressing Ctrl-C and reading a
traceback. Three tiers, cheapest first:

1. **Catch it, keep going** — the failure is local and recoverable
   (one bad sensor read).
2. **Catch it, log it, retry with backoff** — the failure is
   probably transient (network, I2C bus glitch).
3. **Let it crash, but land in a state that recovers** — reset the board
   and rely on a fresh boot, logging what happened first.

```python
import time

def read_sensor_safe(sensor, default=None):
    try:
        sensor.measure()
        return sensor.temperature(), sensor.humidity()
    except OSError as e:
        print("sensor read failed:", e)
        return default
```

## `sys.print_exception`: readable tracebacks without crashing the REPL

Bare `except Exception:` swallowing the traceback is the single worst
debugging habit in embedded Python — you lose exactly the information
you need. `sys.print_exception` prints a normal-looking traceback from
inside a `try/except` without re-raising:

```python
import sys

try:
    risky_operation()
except Exception as e:
    sys.print_exception(e)      # full traceback to stdout/serial
    # then decide: retry, log, or let the device reset
```

To also capture that traceback as a string (for writing to a log file
instead of just printing it), redirect it into an `io.StringIO`:

```python
import sys
import io

try:
    risky_operation()
except Exception as e:
    buf = io.StringIO()
    sys.print_exception(e, buf)
    log_text = buf.getvalue()
    print(log_text)
    # write log_text to flash — see below
```

## Rotating log files on flash

Flash has finite write cycles and finite space — an unbounded log file
will eventually fill the filesystem. A simple two-file rotation caps
usage:

```python
import os

_LOG_FILE = "app.log"
_LOG_FILE_OLD = "app.log.old"
_MAX_LOG_SIZE = 4096          # bytes

def log_write(message):
    try:
        size = os.stat(_LOG_FILE)[6]
    except OSError:
        size = 0

    if size > _MAX_LOG_SIZE:
        try:
            os.remove(_LOG_FILE_OLD)
        except OSError:
            pass
        os.rename(_LOG_FILE, _LOG_FILE_OLD)

    with open(_LOG_FILE, "a") as f:
        f.write(message + "\n")
```

This rotation logic is pure file/string operations and is fully testable
with `python3` against a temp directory rather than real flash:

```python
import os

def test_log_rotation(tmp_path):
    log_file = os.path.join(tmp_path, "app.log")
    # simulate writes exceeding _MAX_LOG_SIZE, then assert app.log.old exists
    ...
```

Include a timestamp (module 6) in every log line so entries are useful
after the fact:

```python
import time

def log_event(msg):
    t = time.localtime()
    line = "{:04d}-{:02d}-{:02d}T{:02d}:{:02d}:{:02d} {}".format(
        t[0], t[1], t[2], t[3], t[4], t[5], msg
    )
    log_write(line)
```

## Capturing tracebacks across a reset

The most useful crash log is the one written *just before* an unhandled
exception forces a reset. Wrap `main()` in a top-level catch-all that
writes the traceback to flash before resetting, then check for that file
on the next boot:

```python
import machine
import sys
import io

def main():
    ...  # your actual application

if __name__ == "__main__":
    try:
        main()
    except Exception as e:
        buf = io.StringIO()
        sys.print_exception(e, buf)
        log_write("CRASH: " + buf.getvalue())
        time.sleep(1)
        machine.reset()                 # hard reset — start clean
```

On the next boot, check `machine.reset_cause()` to know whether this was
a clean start or a recovery:

```python
import machine

cause = machine.reset_cause()
if cause == machine.PWRON_RESET:
    print("normal power-on")
elif cause == machine.HARD_RESET:
    print("recovered from a crash reset — check app.log")
elif cause == machine.WDT_RESET:
    print("watchdog fired — something hung")
```

## The watchdog timer: a safety net for hangs

Some failures don't raise an exception at all — a hung I2C read can block
forever. `machine.WDT` resets the board if it isn't "fed" within a
timeout, catching hangs that plain `try/except` can't:

```python
import machine

wdt = machine.WDT(timeout=10_000)   # must feed within 10 s or reset

while True:
    do_periodic_work()
    wdt.feed()                      # reaching this line proves we're alive
    time.sleep(2)
```

Once created, a `WDT` on most ports **cannot be disabled** — starting one
commits the program to feeding it regularly for the rest of its run, so
only enable it once the main loop structure is stable and you're
confident every code path (including error-handling branches) reaches a
`feed()` call reasonably often.

## Error-handling traps

!!! warning "Bare `except:` hides the real bug"
    `except:` (or `except Exception:` without inspecting `e`) catches
    everything, including bugs you actually want to know about (a typo'd
    variable name raising `NameError`). Catch the specific exception type
    you expect (`OSError` for hardware I/O) and let genuinely unexpected
    exceptions propagate to your crash-logging wrapper.

!!! warning "Logging itself can fail"
    Writing to a full or corrupted filesystem raises `OSError` too — wrap
    log-writing calls in their own `try/except` so a logging failure
    doesn't mask (or crash out of) the original error you were trying to
    record.

!!! warning "The watchdog can't be fed from inside a hang"
    `wdt.feed()` only helps if your main loop keeps running long enough to
    reach it. A genuinely stuck `i2c.readfrom_mem()` call with no timeout
    will still trigger the watchdog reset — which is the *correct*
    outcome, but don't mistake "I added a watchdog" for "I fixed the
    hang."

!!! warning "Reset-cause checks must run early, before anything else can crash"
    Put the `machine.reset_cause()` check at the very top of `main.py`,
    before any sensor/network init that could itself raise — otherwise a
    crash during startup keeps you from ever seeing why the *previous*
    boot failed.

## Cheat sheet

| Function / idiom | Purpose |
|---|---|
| `sys.print_exception(e)` | Print a full traceback without crashing |
| `sys.print_exception(e, io.StringIO())` | Capture the traceback as a string |
| rotating `app.log` / `app.log.old` | Cap flash usage from logging |
| `machine.reset_cause()` | `PWRON_RESET`, `HARD_RESET`, `WDT_RESET`, `DEEPSLEEP_RESET` |
| `machine.reset()` | Force a clean reboot after logging a crash |
| `machine.WDT(timeout=ms)` | Reset the board if `feed()` isn't called in time |
| `wdt.feed()` | Prove the main loop is alive; call every iteration |
| catch specific exceptions (`OSError`), not bare `except:` | Don't hide real bugs |

## Exercise

Build (and unit-test the pure-Python parts with `python3`) a small
`logger.py` module: `log_write(message)` with the size-based rotation
shown above, and `log_event(msg)` that prefixes a timestamp. Write tests
that create a temp file, write past `_MAX_LOG_SIZE`, and assert the
`.old` rotation happened and the newest file only contains post-rotation
entries. Then write `main.py` for a board: check `machine.reset_cause()`
at the top and log which kind of boot this was; wrap the rest of `main()`
in a `try/except Exception` that captures the traceback via
`sys.print_exception(e, io.StringIO())`, logs it as `"CRASH: " + ...`,
and calls `machine.reset()`. Add a `machine.WDT(timeout=10_000)` fed once
per loop iteration, and deliberately introduce a bug (an unhandled
`ZeroDivisionError` on some loop count) to confirm the crash gets logged
and the board comes back up cleanly.
