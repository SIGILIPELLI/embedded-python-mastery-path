# Capstone — Production IoT Product

This capstone combines the entire Level 4 toolkit into one coherent
product design: a device that's provisioned safely, ships firmware
updates over the air to a fleet, protects itself against hangs and
corruption, encrypts its network traffic, reports its health to a
central registry, is built from source with a reproducible pipeline,
passes automated tests before release, and rolls off a factory line
with each unit individually verified. There's no hardware or backend
here to actually run this against — this is a full design document,
reviewed for internal technical consistency against every module it
draws from, with the pieces of logic that are pure computation verified
via `python3`.

## The product

A battery-and-mains-capable environmental sensor node: reads
temperature/humidity, reports to a cloud backend over TLS, receives
staged OTA updates, survives unattended for years, and is manufactured
in batches of hundreds.

## End-to-end lifecycle

```
Factory                Provisioning              Field operation
┌─────────────┐       ┌──────────────────┐      ┌───────────────────────┐
│ Flash        │──────▶│ AP-mode setup    │─────▶│ Boot: state machine    │
│ generic      │       │ portal: WiFi +   │      │ BOOT→CONNECTING→RUNNING│
│ firmware     │       │ backend endpoint │      │                        │
│              │       │                  │      │ TLS telemetry, staged  │
│ Self-test    │       │ Writes per-device│      │ OTA polling, watchdog  │
│ + serial     │       │ config + secrets │      │ fed each healthy loop  │
│              │       │ to filesystem    │      │                        │
│ Record yield │       │                  │      │ Reports to fleet       │
└─────────────┘       └──────────────────┘      │ registry each check-in │
                                                  └───────────────────────┘
```

## Layered architecture (module 01)

```
main.py                 # boot.py crash-count check, then hand off
app/
  config.py              # load+validate; defaults survive a missing file
  state.py               # DeviceState machine + transition table
  drivers/
    dht_sensor.py          # hardware-facing only
  services/
    telemetry.py            # TelemetryBatcher + TLS send, injected deps
    ota.py                   # version check, staged rollout, confirm/rollback
    diag.py                   # fixed diagnostic command set (module 07)
```

`telemetry.py` and `ota.py` both take their dependencies (the sensor,
the network socket, the config) as constructor arguments, exactly as
in module 01 — this is what makes the state-machine transitions and the
rollout-percentage logic below testable on the Unix port (module 08)
without a single real peripheral involved.

## Boot sequence combining the state machine, watchdog, and safe mode

```python
# boot.py — runs first, before main.py, per module 04's safe-mode pattern
def crash_count():
    try:
        with open("crash_count.txt") as f:
            return int(f.read())
    except (OSError, ValueError):
        return 0

_SAFE_MODE = crash_count() >= 3
if not _SAFE_MODE:
    with open("crash_count.txt", "w") as f:
        f.write(str(crash_count() + 1))
```

```python
# main.py — only runs if boot.py didn't force safe mode
import gc
from machine import WDT
from app.config import load as load_config
from app.state import StateMachine, DeviceState

def run():
    wdt = WDT(timeout=8000)   # generous enough for a TLS handshake + OTA check
    cfg = load_config()
    sm = StateMachine()

    sm.transition(DeviceState.CONNECTING)
    connect_wifi(cfg)
    sm.transition(DeviceState.RUNNING)

    consecutive_errors = 0
    while True:
        try:
            do_sample_and_report(cfg)
            consecutive_errors = 0
            wdt.feed()   # only fed after real, successful work — module 04
        except Exception as e:
            consecutive_errors += 1
            print("cycle failed:", e)
            if consecutive_errors >= 5:
                sm.transition(DeviceState.ERROR)
                raise   # let it propagate — a full reset is warranted
        gc.collect()   # idle boundary, not mid-cycle — module 01/10 pattern

    # First fully healthy pass through the loop above is the natural
    # place to call clear_crash_count() and, if this boot followed an
    # OTA update, mark the new firmware confirmed (module 02).
```

Note the direct reuse of the high-speed-DAQ project's rule from Level
3: `gc.collect()` runs at the bottom of the loop, an idle boundary
between complete cycles — never inside `do_sample_and_report`, where a
multi-millisecond pause could stack with a TLS handshake's own latency
and trip the watchdog for the wrong reason.

## TLS telemetry with staged OTA polling

```python
# app/services/ota.py
def check_for_update(device_id, current_version, remote_manifest, rollout_bucket_fn):
    if not is_newer(remote_manifest["version"], current_version):
        return None
    if rollout_bucket_fn(device_id) >= remote_manifest["rollout_percent"]:
        return None   # not this device's turn yet — module 02's staged rollout
    return remote_manifest
```

This reuses module 02's `rollout_bucket`/`is_newer` functions verbatim
— the capstone's contribution isn't new logic here, it's wiring
existing, already-tested pieces together at the right point in the
main loop: an OTA check happens periodically inside the RUNNING state,
never mid-critical-section, and any update that's fetched goes through
the temp-file-then-rename pattern (module 04) and a confirm-or-rollback
step (module 02) before it's trusted.

## Factory test mode as a boot-time branch

```python
# A factory unit boots with a jumper/GPIO strap set, selecting the
# self-test path instead of the normal application (module 09)
from machine import Pin

def is_factory_test_mode():
    return Pin(0, Pin.IN, Pin.PULL_UP).value() == 0   # strap held low on the line

if is_factory_test_mode():
    from app.factory_test import run_self_test
    run_self_test()   # reports PASS/FAIL per peripheral, never reaches app logic
else:
    run()   # normal application boot sequence above
```

Gating factory test mode behind a physical strap (rather than a
software command) means it can never be triggered accidentally on a
unit already in the field — the self-test path and the production
application path are mutually exclusive by construction, not by
convention.

## Release pipeline gating a rollout

Putting together modules 05, 08, and 02 into one release flow:

1. Unix-port test suite runs on every commit (module 08) — the state
   machine, config validation, `is_newer`, `rollout_bucket`, and the
   `_decode_raw`-style pure sensor logic all covered here.
2. Hardware-in-the-loop tests run on release branches against a bench
   rig (module 08) — real TLS handshake, real watchdog behavior, real
   sensor reads.
3. On success, a reproducible firmware build (module 05) is produced
   from a pinned MicroPython commit and pinned toolchain.
4. The image is hashed and signed; the hash is what devices verify
   before switching partitions (module 02).
5. The release is staged into the OTA rollout at 5%, monitored via the
   fleet registry's health view (module 07) for stale-device and
   version-fragmentation signals, then widened in stages.

## Cheat sheet — which module each capstone piece draws from

| Capstone piece | Module |
|---|---|
| Layered `app/` structure, dependency injection | 01 |
| `is_newer`, `rollout_bucket`, confirm-or-rollback | 02 |
| AP-mode provisioning, per-device identity | 03 |
| `boot.py` crash counter, safe mode, watchdog feed placement | 04 |
| Pinned, reproducible firmware build | 05 |
| TLS with pinned cert, no `CERT_NONE` in production | 06 |
| Device registry, staged-rollout monitoring | 07 |
| Unix-port + hardware-in-the-loop test gating | 08 |
| Factory self-test strap, serialization, yield tracking | 09 |

## Stretch goals

- Implement the full `DeviceState` transition table plus `run()`'s
  error-counting loop in plain Python with every dependency
  (`connect_wifi`, `do_sample_and_report`) faked, and write a Unix-port-
  style test (module 08's `@test`/`run_all()` pattern) that drives it
  through a forced failure streak and confirms it transitions to
  `ERROR` at exactly 5 consecutive failures, not before or after.
- Extend `check_for_update` into a full simulation: generate 500 fake
  device IDs, run them all through a rollout at 10%, then 25%, then
  100%, and confirm the 10% group is a strict subset of the 25% group
  which is a strict subset of the 100% group — a property the
  hash-based bucketing (module 02) guarantees but is worth confirming
  in code rather than assuming.
- Design (in a code comment / short doc, not necessarily runnable) the
  exact sequence of writes and renames a real OTA update must perform
  on the filesystem side to be safe against power loss at every single
  step, referencing both module 02's atomic-rename pattern and module
  04's brownout material explicitly.
- Write the `YieldTracker`-based factory report (module 09) that a
  release manager would review before approving a batch for shipment,
  combining it with the CI gate from module 08 into one end-to-end
  checklist: what must be true, in order, before a batch of freshly
  manufactured units is approved for wide distribution.
