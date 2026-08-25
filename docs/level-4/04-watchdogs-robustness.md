# Watchdogs, Brownout & Robustness Patterns

A device deployed in the field has no keyboard, no debugger, and no one
watching. When it locks up, corrupts its own filesystem, or gets stuck
in a boot loop, the only thing standing between "runs for years
unattended" and "someone drives out to power-cycle it" is the
robustness patterns in this module. Reviewed against `machine.WDT`'s
documented API and general embedded resilience practice; the
supervisor/safe-mode/crash-loop logic below is verified with `python3`.

## `machine.WDT` — the hardware watchdog

```python
from machine import WDT

# Must be fed within 5000ms of each feed() call, or the hardware
# resets the whole chip unconditionally — WDT.feed() cannot be
# cancelled or paused once started on most ports.
wdt = WDT(timeout=5000)

def main_loop():
    while True:
        do_work()
        wdt.feed()   # proves this loop is still alive and making progress
```

The watchdog is a piece of hardware, independent of the CPU running
your Python interpreter, that resets the chip if not "fed" within the
configured timeout. Its entire value is that it works **even when the
interpreter itself is wedged** — an infinite loop, a deadlock, code
stuck waiting on a peripheral that never responds — none of these
prevent the watchdog hardware from firing, because it doesn't depend on
Python execution continuing correctly at all.

The trap: feeding the watchdog from a place that doesn't actually prove
the application is healthy defeats its entire purpose.

```python
# WRONG — feeds unconditionally from a timer IRQ, regardless of
# whether the actual application logic is still functioning
from machine import Timer

def feed_regardless(t):
    wdt.feed()

Timer(0).init(period=1000, callback=feed_regardless)
# Now the watchdog can never fire, even if main_loop() above has
# deadlocked completely — the timer keeps feeding it forever
```

`wdt.feed()` should only ever be called from a point in the code that
genuinely proves forward progress on the actual application logic —
typically once per iteration of the main loop, after real work
completed, never from an independent timer that runs regardless of
what the rest of the program is doing.

## Supervisor patterns — process-level self-healing

For applications structured with the state machine from the production
architecture module, a supervisor wraps the whole run loop and reacts
to failures without needing the watchdog's blunt full-chip-reset for
every kind of problem:

```python
import time

def supervised_run(app_step, max_consecutive_errors=5):
    consecutive_errors = 0
    while True:
        try:
            app_step()
            consecutive_errors = 0
            wdt.feed()
        except Exception as e:
            consecutive_errors += 1
            print("app_step failed:", e, "count:", consecutive_errors)
            if consecutive_errors >= max_consecutive_errors:
                raise   # let it propagate — time for a full reset, not a retry
            time.sleep_ms(200 * consecutive_errors)   # backoff
```

Catching `Exception` broadly here is a deliberate, unusual choice —
normally too broad — because the supervisor's whole job is staying
alive through *unexpected* failures in application code, while still
giving up and triggering a real reset once errors are frequent enough
to suggest something more serious than a transient glitch (a stuck
sensor, one dropped network packet) is happening.

## Surviving brownouts

A brownout — the supply voltage dipping below what the chip needs,
often from an under-sized power supply under load spikes (radio
transmit being a common culprit) — can corrupt in-progress flash
writes specifically, since flash writes are the one operation genuinely
vulnerable to power interruption mid-operation.

```python
def safe_write_file(path, data):
    tmp = path + ".tmp"
    with open(tmp, "wb") as f:
        f.write(data)
        f.flush()
    import os
    os.rename(tmp, path)   # atomic — a brownout here leaves the old file intact
```

This is the same write-to-temp-then-rename pattern from the OTA
module, and it applies everywhere a file is written on a device that
might lose power mid-write, not just during updates — any config save,
any log rotation. The failure mode being defended against specifically
is: power dies during the direct-write version, leaving a half-written
file that's neither the old nor the new valid content, which is worse
than losing the update entirely.

## Safe-mode boots

When `main.py` itself is what's crashing (a bad config value, a
corrupted file, logic that throws before ever reaching the point where
it could recover gracefully), the device needs an escape hatch that
doesn't depend on the very code that's broken:

```python
# boot.py runs before main.py on most ports — this is the natural
# place for safe-mode detection, since it executes even if main.py
# itself is what's unable to run
import os

def crash_count():
    try:
        with open("crash_count.txt") as f:
            return int(f.read())
    except (OSError, ValueError):
        return 0

def record_crash():
    with open("crash_count.txt", "w") as f:
        f.write(str(crash_count() + 1))

def clear_crash_count():
    try:
        os.remove("crash_count.txt")
    except OSError:
        pass

if crash_count() >= 3:
    print("repeated crashes detected — entering safe mode, main.py skipped")
    # Safe mode: no main.py import, drop into a minimal REPL-reachable
    # state so the device can still be recovered remotely (module 07)
    # rather than boot-looping forever.
else:
    record_crash()
    # main.py runs next as normal; on success, application code should
    # call clear_crash_count() once it's confirmed healthy (compare to
    # the OTA module's confirm-or-rollback pattern)
```

This mirrors the OTA module's "pending, not confirmed until healthy"
pattern for exactly the same reason: a crash counter that only resets
on confirmed success, checked before the risky code runs, is what
turns an infinite boot loop into a bounded number of attempts followed
by a recoverable safe state.

## Designing for unattended recovery over years

- Every persistent write goes through the safe-write pattern — no
  exceptions, because the cost of getting this wrong (a corrupted
  config or crash-counter file) can itself cause the very crash loop
  the mechanism exists to prevent.
- The watchdog timeout should be generous enough that legitimate slow
  operations (a sensor with a genuinely long conversion time, a
  congested network write) don't trigger spurious resets, but tight
  enough that a real hang doesn't leave the device unresponsive for an
  unacceptable stretch — this number should come from measuring actual
  worst-case legitimate operation time, not a guess.
- Crash-loop detection (the `crash_count.txt` pattern) and the
  watchdog are complementary, not redundant: the watchdog catches
  hangs; the crash counter catches fast, repeated crashes that never
  hang long enough for the watchdog to matter.

## Cheat sheet

| Mechanism | Catches |
|---|---|
| `machine.WDT` | Hangs/deadlocks — anything that stops forward progress entirely |
| Supervisor with bounded retry | Transient exceptions in application logic, without a full reset |
| Write-to-temp-then-rename | Brownout/power-loss corruption of persisted files |
| `boot.py` crash counter | Fast, repeated crashes a watchdog timeout wouldn't catch |
| Safe mode (skip `main.py`) | Recovering a device whose own application code can't run at all |

## Exercise

In plain Python, implement the `crash_count()`/`record_crash()`/
`clear_crash_count()` trio using a temp file in place of the device
filesystem, plus `safe_write_file()` using the temp-then-rename
pattern. Write a short simulation: call `record_crash()` three times
(simulating three failed boots), confirm `crash_count()` reports 3 and
your safe-mode threshold logic would trigger, then call
`clear_crash_count()` and confirm `crash_count()` is back to 0. Add a
`supervised_run`-style loop (from this module) around a fake
`app_step()` function that raises on its first two calls and succeeds
on the third, and confirm it recovers without exceeding a
`max_consecutive_errors` of 5.
