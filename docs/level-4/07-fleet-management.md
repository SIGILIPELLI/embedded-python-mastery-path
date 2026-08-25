# Fleet Management & Remote Monitoring

One device is easy to reason about. A hundred, or a hundred thousand,
need a device registry, a way to see their health without visiting
each one, and a way to act on that visibility before a small problem
becomes a truck roll. This module covers the operational layer above
individual firmware — reviewed against common IoT fleet-management
architecture patterns (device registries, telemetry pipelines,
alerting) rather than any single vendor's product, since the
concepts generalize across platforms. The registry and alerting logic
below is verified with `python3`.

## The device registry — the source of truth for "what's out there"

```python
# Server-side (not on the device) — a minimal registry record
class DeviceRecord:
    def __init__(self, device_id, firmware_version, last_seen, tags=None):
        self.device_id = device_id
        self.firmware_version = firmware_version
        self.last_seen = last_seen
        self.tags = tags or {}   # e.g. {"site": "warehouse-3", "batch": "2026-q1"}

class Registry:
    def __init__(self):
        self._devices = {}

    def upsert(self, record):
        self._devices[record.device_id] = record

    def by_tag(self, key, value):
        return [d for d in self._devices.values() if d.tags.get(key) == value]

    def stale(self, now, max_age_s):
        return [d for d in self._devices.values() if now - d.last_seen > max_age_s]
```

The registry is what every other fleet operation is built on: an OTA
rollout targets "devices with `tags["batch"] == "2026-q1"`" (previous
module), an alert fires for "devices in `stale()`," a dashboard groups
by `tags["site"]`. None of that is possible without a device consistently
identifying itself (the per-device identity from the provisioning
module) and checking in often enough that `last_seen` means something.

## Telemetry pipelines

A device pushing raw sensor readings straight into whatever backend
receives them, unbatched and unbounded, doesn't scale past a handful of
units. The realistic shape at fleet scale:

```python
# On-device: batch readings, send periodically, not per-reading
class TelemetryBatcher:
    def __init__(self, max_batch=20):
        self._batch = []
        self._max_batch = max_batch

    def add(self, reading):
        self._batch.append(reading)
        return len(self._batch) >= self._max_batch

    def flush(self):
        batch, self._batch = self._batch, []
        return batch
```

```python
batcher = TelemetryBatcher(max_batch=5)
for reading in [21.1, 21.3, 21.0, 21.4, 21.2, 20.9]:
    if batcher.add(reading):
        send(batcher.flush())   # sends a batch of 5, then starts a new one
```

Batching trades latency (a reading waits for its batch to fill, or a
timer to expire, before transmission) for a dramatically lower request
rate at the receiving end — the right trade for the overwhelming
majority of telemetry, which doesn't need per-reading real-time
delivery. Fleet-scale pipelines generally also need per-device rate
limiting on the receiving side, since one misbehaving device sending
at full speed shouldn't be able to degrade service for the rest of the
fleet.

## Remote diagnostics and log collection

When a specific device is behaving unexpectedly, the operator needs a
way to pull its state without physical access — but an always-on
verbose remote-shell capability is itself a security liability (see
the hardening module's "disable the REPL" guidance). The middle ground
most production fleets land on is a bounded, command-driven diagnostic
channel:

```python
# On-device: a small, explicit set of diagnostic commands — not an
# open REPL, a fixed menu
_DIAG_COMMANDS = {
    "uptime": lambda: get_uptime_s(),
    "free_mem": lambda: gc_mem_free(),
    "last_errors": lambda: read_error_log(n=10),
    "firmware_version": lambda: CURRENT_VERSION,
}

def handle_diag_request(command_name):
    handler = _DIAG_COMMANDS.get(command_name)
    if handler is None:
        return {"error": "unknown command"}
    try:
        return {"result": handler()}
    except Exception as e:
        return {"error": str(e)}
```

This is deliberately not "run arbitrary code the operator sends" — a
fixed, reviewed set of diagnostic operations means a compromised or
spoofed control channel can't be used to execute arbitrary code on the
fleet, only to run the handful of read-only operations that were
explicitly exposed.

## Alerting on fleet health

```python
def evaluate_fleet_health(registry, now, stale_threshold_s=3600):
    stale = registry.stale(now, stale_threshold_s)
    alerts = []
    if stale:
        alerts.append(f"{len(stale)} device(s) not seen in over {stale_threshold_s}s")
    version_counts = {}
    for d in registry._devices.values():
        version_counts[d.firmware_version] = version_counts.get(d.firmware_version, 0) + 1
    if len(version_counts) > 3:
        alerts.append(f"fleet fragmented across {len(version_counts)} firmware versions")
    return alerts
```

Two different kinds of signal here matter for different reasons: a
device going stale (not checking in) is an individual-device problem —
something's wrong with that unit or its connectivity. A fleet
fragmenting across many firmware versions is an *operational* problem —
it means rollouts aren't completing, old bugs stay unfixed on some
devices indefinitely, and the OTA module's staged-rollout mechanism has
stalled somewhere without anyone noticing. Both need alerting, but they
point an operator toward different remediation.

## Dashboarding fleet health

The registry plus telemetry plus alerting above are the actual data;
a dashboard is a presentation layer over them, not a separate system —
worth stating because it's tempting to treat dashboarding as the
starting point when it should be the last layer built, on top of a
registry and telemetry pipeline that already work correctly on their
own without a UI in front of them.

```python
def fleet_summary(registry, now):
    devices = list(registry._devices.values())
    return {
        "total": len(devices),
        "stale": len(registry.stale(now, 3600)),
        "by_version": {
            v: sum(1 for d in devices if d.firmware_version == v)
            for v in {d.firmware_version for d in devices}
        },
    }
```

## Cheat sheet

| Layer | Purpose |
|---|---|
| Device registry | Source of truth for identity, version, last-seen, tags |
| Telemetry batching | Bounded request rate at fleet scale vs. per-reading sends |
| Fixed diagnostic command set | Remote visibility without an open remote-execution channel |
| Fleet health alerting | Stale devices (per-unit issue) vs. version fragmentation (rollout issue) |
| Dashboard | Presentation over registry/telemetry data, built last |

## Exercise

In plain Python, implement `DeviceRecord` and `Registry` as shown, then
populate a registry with 10 fake devices: 7 on firmware `"2.3.1"`, 2 on
`"2.3.0"`, 1 on `"2.2.5"`, with `last_seen` timestamps such that 2 of
them are older than a 3600-second threshold relative to a fixed `now`.
Implement and call `evaluate_fleet_health` and `fleet_summary`, and
print both results — confirm the stale-device alert fires for exactly
2 devices and the version-fragmentation alert does *not* fire (since 3
distinct versions is at the threshold, not over it, per the `> 3`
check above) — then add one more device on a 4th distinct version and
confirm the fragmentation alert now fires.
