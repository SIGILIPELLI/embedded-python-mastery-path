---
description: "CI, Emulation & Automated Testing — Nothing in this course so far has been 'tested' in the sense a web/backend developer means it — most modules…"
---

# CI, Emulation & Automated Testing

Nothing in this course so far has been "tested" in the sense a
web/backend developer means it — most modules explicitly rely on
manual review against documentation because there's no hardware here.
A real product needs an actual automated test suite, and MicroPython
gives you a genuine way to run one: the Unix port, which builds
MicroPython as a native executable that runs on a development machine
or a CI runner, no microcontroller required. This module covers
building that pipeline, plus hardware-in-the-loop testing for the parts
the Unix port can't cover. Reviewed against MicroPython's documented
Unix port and `mpremote` usage; the test-runner logic below is run for
real with `python3`, standing in for the Unix port's behavior where
the code is port-agnostic.

## The Unix port — MicroPython without a microcontroller

MicroPython's `ports/unix` builds a `micropython` executable that runs
on Linux/macOS, implementing the same core language and most
standard-library modules as the embedded ports. This is the single
most valuable testing asset available for MicroPython projects: pure
logic — parsing, protocol framing, the state machine from the
production architecture module, config validation — can run in CI on
every commit in milliseconds, with no hardware in the loop and no
flaky serial connection to a real board.

```bash
# Conceptual CI step — not run in this environment, no Unix port built here
./micropython test_state_machine.py
```

The catch, and the reason "runs on the Unix port" isn't the same claim
as "runs on the target board": the Unix port doesn't implement
`machine`, `network`'s hardware-backed interfaces, or anything
board-specific — code that imports `machine.Pin` at module level fails
immediately on the Unix port. This is precisely why the production
architecture module's driver-isolation principle matters for testing,
not just for code cleanliness: logic that never imports `machine`
directly can run and be tested on the Unix port; logic that does can't,
regardless of how good the test would otherwise be.

## Structuring code so most of it is Unix-port-testable

```python
# drivers/temp_sensor.py — hardware-facing, NOT unit-testable off-device
from machine import I2C

class TempSensor:
    def __init__(self, i2c, addr=0x48):
        self._i2c = i2c
        self._addr = addr

    def read_celsius(self):
        raw = self._i2c.readfrom(self._addr, 2)
        return _decode_raw(raw)   # pure function — testable in isolation

def _decode_raw(raw):
    value = (raw[0] << 8 | raw[1]) >> 4
    if value & 0x800:
        value -= 4096
    return value * 0.0625
```

```python
# test_temp_decode.py — runs fine on the Unix port or plain python3,
# since it never imports machine at all
from drivers.temp_sensor import _decode_raw

def test_positive_reading():
    assert _decode_raw(bytes([0x19, 0x00])) == 25.0

def test_negative_reading():
    # top bit set -> negative temperature per this sensor's encoding
    result = _decode_raw(bytes([0xE7, 0x00]))
    assert result < 0

test_positive_reading()
test_negative_reading()
print("temp decode tests passed")
```

The pattern: pull the pure computation (`_decode_raw`) out of the class
that owns the hardware handle, so the part with actual logic worth
testing has zero dependency on `machine` and the part that touches
hardware is thin enough that it barely needs testing beyond "does it
call the right I2C method with the right arguments," which is better
covered by hardware-in-the-loop testing anyway.

## A minimal test runner

MicroPython (including the Unix port) has no built-in `unittest`-style
framework as rich as CPython's, though a `unittest` subset exists on
recent versions — many MicroPython projects use something close to
this minimal pattern regardless, since it has zero dependencies and
works identically everywhere:

```python
_tests = []

def test(fn):
    _tests.append(fn)
    return fn

def run_all():
    passed, failed = 0, 0
    for fn in _tests:
        try:
            fn()
            passed += 1
        except AssertionError as e:
            failed += 1
            print("FAIL:", fn.__name__, "-", e)
    print(f"{passed} passed, {failed} failed")
    return failed == 0

@test
def test_example():
    assert 1 + 1 == 2

if __name__ == "__main__":
    ok = run_all()
```

This `@test` decorator plus `run_all()` pattern is small enough to copy
into any MicroPython project verbatim and runs identically on the Unix
port, on a real board via `mpremote run`, or under plain `python3` —
the same test file works in all three without modification, which is
exactly the property that makes it worth using over something
CPython-only.

## Hardware-in-the-loop rigs driven by `mpremote`

The Unix port covers pure logic; anything genuinely hardware-dependent
(does this driver actually talk to this real sensor correctly, does
this board actually wake from deep sleep, does the watchdog actually
reset the chip) needs a real board attached to the CI runner, driven
via `mpremote`:

```bash
# Conceptual — not run here, no board attached to this environment
mpremote connect /dev/ttyUSB0 run test_sensor_hardware.py
mpremote connect /dev/ttyUSB0 exec "import test_sensor_hardware; test_sensor_hardware.run_all()"
```

`mpremote run <file>` executes a local script's contents on the
connected board and streams output back — this is the mechanism a CI
pipeline uses to run a hardware-dependent test file against a physical
board wired into the CI runner (a "test bench" or "rig"), the same way
a browser-testing pipeline drives a real or virtual browser. The
practical difference from the Unix-port tests: these are slower, need
physical hardware provisioned in CI (a real bottleneck at scale — most
teams run far fewer of these than pure-logic tests), and are the only
way to catch bugs that are genuinely about the hardware interaction
itself rather than the logic around it.

## CI pipelines that gate firmware releases

Putting it together, a realistic pipeline for a firmware release:

1. **Lint/format check** — fast, catches trivial issues before
   anything else runs.
2. **Unix-port unit tests** — the bulk of the test suite; pure logic,
   runs on every commit, fast enough to block merges.
2. **Hardware-in-the-loop tests** — a smaller, curated set against
   real boards in a bench rig; run on release branches or nightly,
   not necessarily on every single commit, given the cost.
4. **Build the release firmware image** (module 05) with the pinned
   toolchain, only once the above pass.
5. **Stage the release** into the OTA rollout mechanism (module 02) at
   a small percentage, gated on the above having passed — never build
   and roll out from a commit that skipped its own test gates.

## How It Actually Works

The Unix port's usefulness (and its sharp limits) both come from it being
a genuinely different *build target* of the exact same MicroPython source
tree — same language core, deliberately different peripheral layer.

- **`ports/unix` compiles the identical `py/` core (the lexer, compiler,
  bytecode VM, GC, and object model covered throughout Level 3) against
  POSIX system calls instead of a microcontroller's register set** — the
  same `mp_obj_t` machinery, the same mark-sweep collector, the same
  bytecode dispatch loop this course has traced through embedded ports run
  verbatim on the Unix build, which is exactly why pure logic (parsing, a
  state machine, `_decode_raw`'s bit manipulation) behaves identically in
  both places: none of that code path differs between ports at all. What
  differs is the *hardware abstraction layer* — `ports/unix` simply never
  compiles in `machine`, `network`'s WLAN backend, or any peripheral
  driver, because there is no GPIO register or WiFi radio on a Linux
  machine to back them with, the same compile-time module-exclusion
  mechanism module 5's `mpconfigboard.h` uses deliberately, here happening
  by simple absence of any hardware to bind to.
- **`import machine` failing immediately at module scope on the Unix port
  is the identical mechanism Level 4 module 1 leverages for testability —
  Python's `import` statement raises `ModuleNotFoundError` the instant the
  module system can't locate a module with that name, before any of the
  importing file's other code executes.** This single fact is why the
  driver-isolation architecture pattern and the CI testing strategy are
  really the same idea viewed from two angles: code structured so hardware
  imports live only in thin driver modules is, automatically and for free,
  code that can be exercised by the Unix port's identical interpreter core
  without ever touching real silicon.
- **`mpremote run` driving hardware-in-the-loop tests uses the same raw-
  REPL file-transfer-and-execute mechanism from Level 2 module 5** — the
  CI runner's host process pushes the test file's source into the board's
  raw REPL, the board's actual embedded-port interpreter (not the Unix
  port) compiles and runs it against real registers and real peripherals,
  and results stream back over the same serial link. This is the
  structural reason these tests are slower and harder to scale: each one
  requires exclusive access to a physical board's serial port (the same
  contention Level 2 module 5 describes) and genuine wall-clock time for
  real hardware operations (a real I2C transaction, a real deep-sleep
  cycle) that no amount of faster CI compute can shorten, unlike Unix-port
  tests which run at native-process speed with no hardware latency at all.

## Cheat sheet

| Test type | Runs where | Covers |
|---|---|---|
| Pure-logic unit tests | Unix port, or plain `python3` | Parsing, state machines, config validation, protocol framing |
| Hardware-in-the-loop | Real board via `mpremote` in a bench rig | Actual driver/peripheral behavior, real timing |
| Design rule | Keep `machine`/hardware imports isolated to drivers | Maximizes what's testable without a board |

## Exercise

In plain Python, implement `_decode_raw` as shown, plus the `@test`/
`run_all()` minimal test framework. Write at least four test functions
covering: a simple positive-temperature decode, a negative-temperature
decode (top bit set), a boundary value at exactly 0, and one
deliberately-wrong assertion to confirm `run_all()` correctly reports
it as a failure rather than crashing the whole run. Run `run_all()` and
confirm the printed summary shows 3 passed and 1 failed, with the
failing test's name and assertion message printed.
