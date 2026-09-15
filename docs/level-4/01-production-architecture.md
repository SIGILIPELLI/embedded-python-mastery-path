---
description: "Production Firmware Architecture — Everything so far has been about making individual pieces of MicroPython code fast and correct. Shipping a product…"
---

# Production Firmware Architecture

Everything so far has been about making individual pieces of
MicroPython code fast and correct. Shipping a product means those
pieces have to fit together in a structure that survives years of
unattended operation, updates, and edge cases nobody tested for. This
module is about that structure — reviewed against real-world
MicroPython production patterns (documented in the official docs'
deployment guidance and widely-used community architectures) rather
than a single canonical framework, since MicroPython itself is
unopinionated about application layering. The layering and config
logic below runs as plain Python via `python3` to verify the design's
mechanics.

## Layered modules, not one big `main.py`

A `main.py` that does networking, sensor reads, business logic, and
error handling all inline is the single biggest predictor of a
firmware project that becomes unmaintainable. A minimal but real
layering:

```
main.py              # boot sequence only: init, then hand off to app.run()
app/
  __init__.py
  config.py           # load/validate configuration, no side effects at import
  state.py            # the state machine (below)
  drivers/
    sensor_x.py        # hardware-facing, no business logic
  services/
    telemetry.py        # business logic, depends on drivers + config
    commands.py
```

The rule that keeps this useful: **drivers know nothing about business
logic, and business logic never touches registers directly.** A driver
module (`drivers/sensor_x.py`) exposes `read()`, `configure()`, not
`get_temperature_and_decide_if_alarm_needed()`. This isn't purity for
its own sake — it's what makes the driver testable on desktop (mock the
bus, not the whole app) and what lets a hardware revision swap one
driver module without touching services.

## Config management — separate from code, validated at boot

```python
# app/config.py
import ujson

_DEFAULTS = {
    "sample_interval_s": 10,
    "mqtt_host": "broker.local",
    "mqtt_port": 1883,
}

def load(path="config.json"):
    cfg = dict(_DEFAULTS)
    try:
        with open(path) as f:
            cfg.update(ujson.load(f))
    except OSError:
        pass   # no config file yet — defaults stand, this is expected on first boot
    except ValueError as e:
        print("config.json is corrupt, using defaults:", e)
    _validate(cfg)
    return cfg

def _validate(cfg):
    if not (1 <= cfg["sample_interval_s"] <= 3600):
        raise ValueError("sample_interval_s out of sane range")
    if not (0 < cfg["mqtt_port"] < 65536):
        raise ValueError("mqtt_port invalid")
```

Two details matter: defaults merged **before** validation so a partial
or missing config file still produces a fully-valid config, and a
corrupt config file (`ValueError` from `ujson.load`) is caught
separately from a merely-absent one (`OSError`) — a device that
bricks itself on first boot because `config.json` doesn't exist yet is
a common, avoidable failure mode. Validating ranges at load time, not
wherever the value is first used, means a bad config fails loudly and
early instead of causing a mysterious failure three subroutines away.

## State machines for lifecycle management

Ad hoc boolean flags (`is_connected`, `is_provisioning`, `had_error`)
scattered through a codebase are the second biggest predictor of
firmware bugs — combinations of flags nobody intended become reachable.
An explicit state machine constrains what's possible:

```python
class DeviceState:
    BOOT = "boot"
    PROVISIONING = "provisioning"
    CONNECTING = "connecting"
    RUNNING = "running"
    ERROR = "error"

_TRANSITIONS = {
    DeviceState.BOOT: {DeviceState.PROVISIONING, DeviceState.CONNECTING},
    DeviceState.PROVISIONING: {DeviceState.CONNECTING, DeviceState.ERROR},
    DeviceState.CONNECTING: {DeviceState.RUNNING, DeviceState.ERROR},
    DeviceState.RUNNING: {DeviceState.ERROR, DeviceState.CONNECTING},
    DeviceState.ERROR: {DeviceState.BOOT},
}

class StateMachine:
    def __init__(self):
        self.state = DeviceState.BOOT

    def transition(self, new_state):
        allowed = _TRANSITIONS.get(self.state, set())
        if new_state not in allowed:
            raise ValueError(f"illegal transition {self.state} -> {new_state}")
        print(f"state: {self.state} -> {new_state}")
        self.state = new_state
```

The `_TRANSITIONS` table is the actual design document — reviewing it
answers "can the device go straight from PROVISIONING to RUNNING
without ever CONNECTING?" without reading a single line of business
logic. Raising on an illegal transition, rather than silently allowing
it, turns a logic bug into an immediate, loud failure during
development instead of a field report of a device "acting strange."

## Dependency isolation — for testability and swappability

```python
# services/telemetry.py
class TelemetryService:
    def __init__(self, sensor, transport, clock):
        self._sensor = sensor
        self._transport = transport
        self._clock = clock

    def sample_and_send(self):
        reading = self._sensor.read()
        payload = {"t": self._clock.now(), "v": reading}
        self._transport.send(payload)
```

`TelemetryService` takes its dependencies as constructor arguments
rather than importing `drivers.sensor_x` and a global MQTT client
directly. On desktop, this is tested with fake `sensor`/`transport`/
`clock` objects — no hardware, no network — while on-device it's wired
up with the real drivers once at boot. This single pattern (dependency
injection, without needing a framework to call it that) is what makes
firmware logic actually unit-testable at all, which the CI module
builds on directly.

## Designing for testability and updates from day one

- Business logic should be import-safe on desktop CPython wherever
  possible — avoid `import machine` at the top of a module that
  contains logic worth testing; isolate hardware imports to the
  driver layer so the rest can run, and be tested, off-device.
- Anything that will need to change post-deployment (thresholds,
  server URLs, feature flags) belongs in config, not in code — a
  firmware update should be reserved for actual logic changes.
- The boot sequence in `main.py` should be short and defensive: load
  config, construct the state machine, enter it — with a fallback path
  (see the watchdog module) if any of that fails, rather than an
  unguarded chain of calls that leaves the device stuck if any one of
  them throws.

## How It Actually Works

The layering advice here isn't just software-engineering taste — it maps
directly onto how MicroPython's `import` system and its lack of a
mocking/dependency-injection framework actually behave at runtime.

- **`import machine` at module scope executes immediately, at import time,
  and fails hard if the hardware module doesn't exist in the running
  interpreter** — on desktop CPython, `machine` simply isn't a module that
  exists, so any file with `import machine` at the top raises
  `ModuleNotFoundError` the instant Python tries to load it, before a
  single line of the module's actual logic runs. This is the concrete,
  mechanical reason "business logic should be import-safe on desktop"
  isn't a style preference: a module with hardware imports at the top
  literally cannot be imported at all off-device, regardless of whether
  the specific function being tested ever touches a pin.
- **MicroPython caches every imported module in `sys.modules`, exactly like
  CPython — a module's top-level code runs exactly once per boot, the
  first time anything imports it, and every subsequent `import` just
  returns the cached module object.** This is why `config.py`'s
  `_validate()` running "at load time" genuinely means once, at whatever
  point in the boot sequence `load()` is first called — not every time a
  service reads a config value later. It's also why side effects at
  *module* scope (rather than inside a function) are dangerous in
  practice: a driver module that opens an I2C bus as a module-level
  statement runs that bus-open exactly once, at whatever moment something
  first imports it, which can be an awkward, hard-to-predict point in a
  complex import graph — hence pushing side effects into `__init__`
  methods called explicitly from `main.py`'s boot sequence, where the
  order is visible and deliberate.
- **Dependency injection works here for the same reason it works in any
  language: MicroPython's object model has no special-cased "final" or
  compile-time-bound method dispatch, so passing a `FakeSensor` object
  with a matching `read()` method is genuinely indistinguishable, at the
  bytecode level, from passing a real driver object.** `LOAD_ATTR`
  resolves `self._sensor.read` by walking the actual object's type at
  runtime (module 2's lookup-chain mechanism) — there is no interface or
  type contract enforced anywhere, so any object exposing the right method
  names satisfies `TelemetryService`'s needs, real hardware or not. This
  duck-typing is exactly what makes constructor-injected fakes work without
  any mocking framework, and it's a direct consequence of Python having no
  static type system to route around.
- **Raising `ValueError` on an illegal state transition, rather than
  logging and continuing, matters more here than in a desktop app because
  there's no process supervisor watching stderr** — an unhandled exception
  in MicroPython propagates up the call stack exactly per Level 2 module
  9's mechanics, and without an explicit top-level catch-all (the crash-
  logging wrapper from that module), it eventually reaches `main.py`'s
  outer scope and the interpreter halts or resets, depending on the boot
  configuration — turning a silently-tolerated bad transition into a
  visible, logged failure rather than a device that limps along in an
  undefined combination of states nobody designed for.

## Cheat sheet

| Principle | What it prevents |
|---|---|
| Drivers know nothing of business logic | Untestable, hardware-coupled application code |
| Config merged with defaults, then validated | Bricking on missing/partial config |
| Explicit state machine + transition table | Unreachable-in-theory state combinations happening in practice |
| Dependency injection into services | Business logic that can only be tested on real hardware |
| Hardware imports isolated to drivers | Whole modules unable to run/test on desktop |

## Exercise

In plain Python (no MicroPython-specific imports), implement the
`StateMachine` class above along with its transition table, plus a
`TelemetryService`-style class taking three fake dependencies (a
`FakeSensor` with a `read()` returning a fixed value, a `FakeTransport`
with a `send()` that appends to a list, and a `FakeClock` with a
`now()` returning an incrementing counter). Write a short test
sequence that: constructs the state machine, walks it through
BOOT → CONNECTING → RUNNING, attempts an illegal transition and confirms
it raises `ValueError`, then calls `sample_and_send()` twice and prints
the resulting list of sent payloads to confirm both were captured
correctly.
