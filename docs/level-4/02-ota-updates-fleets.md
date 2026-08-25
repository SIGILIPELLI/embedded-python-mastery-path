# OTA Updates for MicroPython Fleets

Shipping firmware once is easy; shipping the *next* version to devices
already deployed in the field, without a truck roll and without
bricking any of them, is the actual hard problem. MicroPython devices
generally have two distinct update surfaces — updating the frozen
firmware image itself, and updating files on the device's own
filesystem — and production fleets need a considered answer for both,
plus versioning, staged rollout, and integrity verification before
committing to a new version. Reviewed against documented ESP32/RP2040
OTA partition schemes and MicroPython's filesystem update patterns; the
version-comparison and rollout-percentage logic below is verified with
`python3`.

## Two different kinds of "update"

**File-level updates** replace one or more `.py`/`.mpy` files on the
device's existing filesystem, without touching the underlying
MicroPython interpreter/firmware itself. This is the update path for
application logic that lives on the filesystem (see the frozen modules
module for what does and doesn't live there) — conceptually a targeted
file copy, done safely (see below).

**Firmware-image updates** replace the entire MicroPython binary,
including any frozen modules — this is what's needed to pick up a new
MicroPython version, a new frozen module payload, or a change to
compiled C-module code. This requires the chip's own OTA mechanism: the
ESP32's dual-OTA-partition scheme (an "ota_0"/"ota_1" pair, with a
bootloader flag selecting which one boots) or, on RP2040, typically a
full reflash via the bootloader's UF2 mechanism, since RP2040 has no
built-in dual-partition A/B scheme the way ESP-IDF does.

## The A/B partition idea (ESP32)

```
┌────────────┐   ┌────────────┐   ┌────────────┐
│ Bootloader  │──▶│  ota_0      │   │  ota_1      │
│ picks active│   │ (currently  │   │ (write new  │
│ partition   │   │  running)   │   │  image here)│
└────────────┘   └────────────┘   └────────────┘
```

The device boots from whichever OTA partition is currently marked
active. An update writes the new image into the *other, inactive*
partition — the running firmware is untouched and continues serving
the device throughout the download, which can take a while over a slow
or intermittent connection. Only once the new image is fully written
and verified does the bootloader's active-partition pointer flip. This
is what makes power loss mid-download survivable: an interrupted write
into the inactive partition just leaves it incomplete; the device
reboots into the still-intact partition it was already running.

## Versioning and rollback

```python
# app/version.py
CURRENT_VERSION = (2, 3, 1)   # (major, minor, patch)

def is_newer(remote_version, local_version=CURRENT_VERSION):
    return tuple(remote_version) > tuple(local_version)
```

```python
print(is_newer((2, 4, 0)))   # True
print(is_newer((2, 3, 0)))   # False
print(is_newer((2, 3, 1)))   # False — equal is not newer
```

Tuple comparison gives correct semantic-version-style ordering for
free in Python — `(2, 10, 0) > (2, 9, 9)` compares correctly
element-by-element, unlike comparing version strings lexicographically
(where `"2.10.0" < "2.9.9"` as strings, which is wrong).

**Rollback** needs a boot-success confirmation step, not just a
successful flash: the newly-flashed partition should be marked
"pending" rather than immediately and permanently active, and only
promoted to the confirmed/default boot target after the new firmware
has proven it can actually boot and run its own self-check (a watchdog
feed, a successful network connection, whatever the product defines as
"healthy"). Without this, a bad update that flashes successfully but
crashes on boot leaves the device stuck reflashing itself into the same
bad image forever — the rollback module and the watchdog module's
material are meant to be combined here directly.

```python
# Sketch of the confirm-or-rollback logic, run early after a boot
# that followed an update
def confirm_or_rollback(health_check, mark_confirmed, mark_rollback):
    if health_check():
        mark_confirmed()
    else:
        mark_rollback()   # boot into the previous (still-intact) partition
```

## Staged rollouts

Pushing a new version to 100% of a fleet simultaneously is how one bad
release becomes a fleet-wide outage. A staged rollout gates which
devices see the update first, typically by a deterministic per-device
hash so the same device consistently lands in the same rollout wave
across checks:

```python
import uhashlib

def rollout_bucket(device_id, num_buckets=100):
    digest = uhashlib.sha256(device_id.encode()).digest()
    return digest[0] % num_buckets   # 0-99, stable per device

def should_update(device_id, rollout_percent):
    return rollout_bucket(device_id) < rollout_percent
```

```python
# Wave 1: 5% of the fleet
print(should_update("device-001a", 5))
# Wave 2, a day later, if wave 1 looked healthy: 25%
print(should_update("device-001a", 25))
```

Using a hash of the device ID rather than random chance each check
means a given device's bucket is stable — it doesn't randomly flip in
and out of the rollout on every poll, which would make "is this device
part of the canary group" unanswerable.

## Verifying integrity before switching

Never flip the active partition, or overwrite a filesystem file in
place, before confirming what was written matches what was intended.
For firmware images, this is normally the chip's own OTA library
verifying a signature/hash as part of the flash-write API (ESP-IDF's
OTA update APIs do this). For filesystem-level file updates, the same
principle applies at a smaller scale:

```python
import uhashlib
import ubinascii

def verify_download(data, expected_sha256_hex):
    actual = ubinascii.hexlify(uhashlib.sha256(data).digest()).decode()
    return actual == expected_sha256_hex

def apply_update_file(path, data, expected_hash):
    if not verify_download(data, expected_hash):
        raise ValueError("downloaded update failed integrity check")
    tmp_path = path + ".tmp"
    with open(tmp_path, "wb") as f:
        f.write(data)
    import os
    os.rename(tmp_path, path)   # atomic on most filesystems — no partial file visible
```

Writing to a temporary path and renaming into place, rather than
overwriting the target file directly, is the detail that prevents a
power loss mid-write from leaving a half-written, unusable file where
a working one used to be — `os.rename` on a single filesystem is
effectively atomic, while streaming new bytes directly on top of the
existing file is not.

## Cheat sheet

| Concept | Detail |
|---|---|
| File-level update | Replace `.py`/`.mpy` on the filesystem; doesn't touch firmware |
| Firmware-image update | Full OTA flash; ESP32 uses dual A/B partitions, RP2040 typically a full UF2 reflash |
| Tuple version comparison | Correct ordering without string-comparison pitfalls |
| Confirm-or-rollback | Mark new firmware pending until it proves it boots and runs healthily |
| Staged rollout via hashed device ID | Stable per-device bucket, no flip-flopping between checks |
| Verify-then-atomic-rename | Never leave a partially-written file as the live one |

## Exercise

In plain Python, implement `rollout_bucket(device_id, num_buckets=100)`
using `hashlib.sha256` (the desktop equivalent of `uhashlib`), and
`should_update`. Generate 1000 fake device IDs (e.g. `f"device-{i:04d}"`)
and, for a rollout percentage of 10, count how many fall into the
update group — confirm it's reasonably close to 10% (some natural
variance from hashing is expected; note the actual count you get).
Then implement `is_newer` with tuple comparison and write test
assertions confirming `(1, 12, 0)` is correctly treated as newer than
`(1, 9, 5)` — the case a naive string comparison would get wrong.
