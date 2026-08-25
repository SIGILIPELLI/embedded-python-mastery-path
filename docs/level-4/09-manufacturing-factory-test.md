# Manufacturing & Factory Test

Between "firmware works on my dev board" and "firmware works on the
first unit a customer plugs in" sits manufacturing: flashing hundreds
or thousands of boards correctly, verifying each one actually works
before it leaves the factory, giving each unit its own identity, and
tracking which units passed and which didn't. This is the operational
counterpart to provisioning (Level 4, module 03) — provisioning is
what happens when a customer sets a device up; factory test is what
happens before it ever reaches them. Reviewed against common embedded
manufacturing test practice; the yield-tracking and serialization logic
below is verified with `python3`.

## Factory flashing stations

A factory flashing station's job is narrow and needs to be fast and
foolproof at volume: write the firmware image (and only the generic
parts of it — nothing per-device yet) to a blank board as quickly and
reliably as repeated tooling allows.

```python
# Conceptual flashing-station script (would call the actual flash
# tool — esptool.py for ESP32, the RP2040 UF2 bootloader mechanism —
# not something this environment can execute against real hardware)
def flash_station_run(port, firmware_path, expected_size):
    import os
    actual_size = os.path.getsize(firmware_path)
    if actual_size != expected_size:
        raise ValueError("firmware image size mismatch — wrong build artifact?")
    # flash_tool.write(port, firmware_path)  # actual flash call
    return {"port": port, "status": "flashed", "size": actual_size}
```

The size-check before flashing is a cheap, real safeguard against a
common factory-floor mistake: a stale or wrong firmware artifact left
in the flashing tool's staging directory from a previous batch. It
doesn't replace verifying the image hash (module 02's integrity
verification applies here too, before flashing, not just for OTA), but
it catches the most common accident cheaply and instantly.

## Functional test firmware

Immediately after flashing, before a unit is boxed, it needs to prove
its hardware actually works — not just that firmware was written
successfully. This is usually a separate, temporary firmware image
(or a special boot mode of the production firmware) that exercises
every peripheral and reports pass/fail, rather than the customer-facing
application logic:

```python
# Sketch of a factory self-test sequence, run once per unit
def factory_self_test(sensor, radio, led):
    results = {}
    try:
        reading = sensor.read()
        results["sensor"] = "PASS" if reading is not None else "FAIL: no reading"
    except Exception as e:
        results["sensor"] = f"FAIL: {e}"

    try:
        radio.scan_networks()
        results["radio"] = "PASS"
    except Exception as e:
        results["radio"] = f"FAIL: {e}"

    try:
        led.on(); led.off()
        results["led"] = "PASS"
    except Exception as e:
        results["led"] = f"FAIL: {e}"

    return results

def all_passed(results):
    return all(v == "PASS" for v in results.values())
```

Every check catches its own exception rather than letting one failing
peripheral abort the whole test sequence — a unit with a dead LED but
a working sensor and radio should report exactly that (one failure,
two passes), not "test crashed, unknown result," since the factory
operator needs to know precisely what's wrong to decide whether to
rework or scrap the unit.

## Serialization — giving each unit its own identity

Where the chip's factory-burned `machine.unique_id()` (provisioning
module) provides a hardware-level identifier, most products also need
a human-meaningful serial number tying that unit to a batch, a
manufacturing date, and a place in the production run — used on the
physical label, in warranty records, and in the device registry
(fleet management module):

```python
def generate_serial(batch_code, sequence_number):
    # e.g. "2026Q1-B003-000482"
    return f"{batch_code}-{sequence_number:06d}"

def parse_serial(serial):
    batch_code, seq_str = serial.rsplit("-", 1)
    return batch_code, int(seq_str)
```

```python
s = generate_serial("2026Q1-B003", 482)
print(s)                    # 2026Q1-B003-000482
print(parse_serial(s))      # ('2026Q1-B003', 482)
```

The zero-padded sequence number keeps serials sortable as plain
strings in the order they were assigned, which matters for anything
downstream (spreadsheets, simple database queries) that sorts
lexicographically rather than numerically — a real, common gotcha if
the padding is skipped (`"...-482"` sorting before `"...-4820"` as
strings).

## Yield tracking

"Yield" is the fraction of units that pass factory test on the first
attempt — the single most important manufacturing health metric,
because a dropping yield is usually the earliest signal of a hardware
problem (a bad component batch, a marginal design) well before it
would otherwise be noticed.

```python
class YieldTracker:
    def __init__(self):
        self._results = []   # list of (serial, passed: bool, failure_reasons)

    def record(self, serial, test_results):
        passed = all_passed(test_results)
        failures = [k for k, v in test_results.items() if v != "PASS"]
        self._results.append((serial, passed, failures))

    def yield_rate(self):
        if not self._results:
            return 0.0
        passed = sum(1 for _, p, _ in self._results if p)
        return passed / len(self._results)

    def failure_breakdown(self):
        counts = {}
        for _, passed, failures in self._results:
            if not passed:
                for f in failures:
                    counts[f] = counts.get(f, 0) + 1
        return counts
```

```python
tracker = YieldTracker()
tracker.record("B003-000001", {"sensor": "PASS", "radio": "PASS", "led": "PASS"})
tracker.record("B003-000002", {"sensor": "FAIL: timeout", "radio": "PASS", "led": "PASS"})
tracker.record("B003-000003", {"sensor": "PASS", "radio": "FAIL: no networks", "led": "PASS"})

print(tracker.yield_rate())          # 0.333... — 1 of 3 units passed clean
print(tracker.failure_breakdown())   # {'sensor': 1, 'radio': 1}
```

`failure_breakdown()` is what turns a yield number into an actionable
signal: a yield drop concentrated entirely in one component (say,
`"sensor"` failures spiking) points a technician at a specific
component batch or a specific step in assembly, rather than leaving
"yield dropped, cause unknown" as the only available conclusion.

## Cheat sheet

| Stage | Purpose |
|---|---|
| Flashing station | Write generic firmware quickly and verifiably to a blank unit |
| Factory self-test | Prove every peripheral works, per-unit, before boxing |
| Serialization | Human-meaningful, sortable per-unit identity tied to batch/date |
| Yield tracking | Fraction of units passing first-try; earliest signal of a hardware problem |
| Failure breakdown | Turns a yield drop into an actionable, component-specific signal |

## Exercise

In plain Python, implement `factory_self_test`, `all_passed`,
`generate_serial`/`parse_serial`, and `YieldTracker` as shown. Simulate
a batch of 20 units: write a loop that calls `factory_self_test` with
fake sensor/radio/led objects, where roughly 1 in 5 units has a fake
sensor that raises an exception (simulate this with a counter or
`random`, your choice, but make the failure deterministic enough to
reproduce, e.g. every 5th unit) and the rest pass everything. Record
each result in a `YieldTracker` under a generated serial, then print
the final yield rate and failure breakdown, confirming the reported
yield rate matches your known failure pattern (16 of 20 passing = 0.8).
