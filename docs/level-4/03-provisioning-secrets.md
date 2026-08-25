# Device Provisioning & Secrets Management

A freshly-flashed board knows nothing: no WiFi credentials, no server
address, no identity distinguishing it from every other board that ran
the same firmware. Provisioning is the process of getting that
information onto the device safely, once, per-unit — and secrets
management is making sure none of it ends up baked into the firmware
image every unit shares. Reviewed against documented ESP32 AP-mode and
BLE provisioning patterns and general embedded secrets-handling
practice; the credential-storage and identity logic below is verified
with `python3`.

## Why provisioning can't be "just hardcode it"

```python
# Never do this — every unit built from this firmware shares one
# WiFi password and one API key, both readable by anyone who can
# read the flash (which is most attackers with physical access).
WIFI_PASSWORD = "CompanyWifi2024!"
API_KEY = "sk_live_abc123..."
```

This fails in two independent ways: it doesn't work at all once a unit
ships to a customer's own network (it's not *their* WiFi password), and
it means one leaked firmware image compromises every device that ever
ran it, forever, with no per-device revocation possible. Secrets and
per-network config belong on the device's filesystem, written during
provisioning, not in the firmware image that's identical across the
whole fleet.

## AP-mode provisioning — a captive setup portal

The common pattern for "how does a device that only knows one closed
network get onto the customer's actual WiFi": boot into access-point
mode, serve a small setup page, let the customer submit their real
credentials, then reboot into normal station mode using them.

```python
import network

def start_provisioning_ap(ssid="SetupDevice-A1B2"):
    ap = network.WLAN(network.AP_IF)
    ap.active(True)
    ap.config(essid=ssid, security=0)   # open network for setup only
    return ap

# A minimal handler for a form POST during AP-mode setup
def handle_provision_request(form_data, save_fn):
    ssid = form_data.get("ssid")
    password = form_data.get("password")
    if not ssid:
        return False, "SSID required"
    save_fn({"ssid": ssid, "password": password})
    return True, "saved — rebooting to connect"
```

Two things worth flagging: the AP itself should be open only for the
brief provisioning window it needs to exist (leaving an unauthenticated
AP running permanently is its own vulnerability, covered further in
the security hardening module), and the moment credentials are
received they should be written to the filesystem — not just held in a
variable — since a provisioning session can be interrupted by power
loss before the reboot into station mode happens.

## BLE provisioning

For devices without a screen or a way to join a temporary AP smoothly
(or where opening any AP even briefly is unacceptable), BLE GATT
characteristics carry the same setup data over a link that's off by
default and only advertises during an explicit provisioning window:

```python
# Sketch: aioble GATT service exposing writable characteristics for
# SSID/password, reviewed against aioble's documented API shape
# (module 02 of Level 2 covers aioble/BLE basics in depth)
import aioble
import bluetooth

_PROV_SERVICE = bluetooth.UUID("12345678-0000-0000-0000-000000000000")
_SSID_CHAR = bluetooth.UUID("12345678-0000-0000-0000-000000000001")

async def provisioning_service(save_fn):
    service = aioble.Service(_PROV_SERVICE)
    ssid_char = aioble.Characteristic(service, _SSID_CHAR, write=True, capture=True)
    aioble.register_services(service)

    while True:
        connection, data = await ssid_char.written()
        save_fn(data.decode())
```

BLE provisioning trades a slightly more complex mobile-app-side
implementation for never exposing an open WiFi AP at all — a real
security posture improvement worth the extra complexity for products
that ship at any real scale.

## Per-device identity

Every unit needs something that uniquely identifies it — for telemetry
attribution, for OTA rollout bucketing (previous module), and for
revoking one compromised unit's access without touching the rest of
the fleet:

```python
import machine
import ubinascii

def device_id():
    # machine.unique_id() returns the chip's factory-burned unique ID
    # bytes — stable across reflashes, reboots, and factory resets,
    # because it's read from silicon, not from anything provisioning
    # writes.
    return ubinascii.hexlify(machine.unique_id()).decode()
```

`machine.unique_id()` is the right foundation for identity precisely
because it survives everything a software-level ID would not (a
factory reset that wipes the filesystem, a firmware reflash) — an
identity stored only in a provisioned config file would be lost on
exactly the events (factory reset, corrupted filesystem) where you most
need the device to still be recognizable.

## Keeping secrets out of firmware images

The dividing line: anything identical across every unit of a given
firmware version is safe to freeze into the image (module 08 of Level
3); anything unique per device or per deployment — WiFi credentials,
API tokens, TLS client certificates — belongs in provisioned,
filesystem-level storage, ideally with basic protection against casual
reading:

```python
import ujson

def save_secrets(path, secrets_dict):
    with open(path, "w") as f:
        ujson.dump(secrets_dict, f)
    # A minimal, deliberate step: mark the intent that this file
    # holds secrets, so later code (backups, log dumps, debug
    # commands) can deliberately exclude it rather than accidentally
    # including it.

_SECRET_PATHS = {"wifi_creds.json", "api_token.json"}

def is_secret_file(path):
    return path.rsplit("/", 1)[-1] in _SECRET_PATHS
```

On MicroPython the filesystem is generally not encrypted at rest
unless the port and flash controller specifically support it (the
security hardening module covers encrypted flash/NVS where the target
chip provides it) — treating "on the filesystem" as equivalent to
"safe" is a mistake. The realistic goal at this layer is keeping
secrets *out of the shared firmware image* and *out of paths that get
casually logged, backed up, or transmitted* — genuine at-rest
encryption is a separate, chip-dependent guarantee layered on top.

## Cheat sheet

| Concern | Approach |
|---|---|
| WiFi/API secrets identical across units | Never — always provisioned, never frozen into firmware |
| Getting real network credentials onto a screenless device | AP-mode captive portal or BLE GATT write, both time-boxed |
| Per-device identity | `machine.unique_id()` — survives reflash and factory reset |
| Secrets on the filesystem | Filesystem storage keeps them out of the shared image; not automatically encrypted at rest |
| Flagging secret files | An explicit allowlist/denylist so backup/log/debug paths don't leak them |

## Exercise

In plain Python, implement `save_secrets`/`load_secrets` functions
using `json` (standing in for `ujson`) that write to and read from a
temp file, plus `is_secret_file` with the deny-list pattern shown
above. Then write a small `device_id()` stand-in using Python's
`uuid.getnode()` (a real per-machine identifier, standing in for
`machine.unique_id()`) hex-encoded. Write a short test that: saves a
fake `{"ssid": "home", "password": "hunter2"}` secrets dict, reloads
it and confirms round-trip equality, confirms `is_secret_file` flags
`"wifi_creds.json"` but not `"readings.json"`, and prints the generated
device ID.
