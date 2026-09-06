# BLE with aioble

WiFi isn't the only way an ESP32 talks to the world — **Bluetooth Low
Energy** is the low-power, short-range option, ideal for a phone app
reading a sensor or a wearable that needs to run for months on a coin
cell. `aioble` is MicroPython's `asyncio`-based BLE library, wrapping the
lower-level `bluetooth` module. This module advertises a device, exposes a
GATT service/characteristic, sends notifications, and builds a
phone-readable sensor beacon — reviewed against MicroPython's `aioble`
docs and examples, since Wokwi's BLE simulation support is limited; verify
final behavior on real hardware with the nRF Connect or LightBlue app.

## GATT concepts in 60 seconds

BLE data model, top to bottom:

- **Service** — a group of related data, identified by a UUID (e.g. "this
  is a temperature service").
- **Characteristic** — one value inside a service (e.g. "the current
  temperature reading"), also UUID-identified, with permissions (read,
  write, notify).
- **Advertising** — short broadcast packets a device sends before anyone
  connects, so scanners can find it and see its name.

Standard UUIDs exist for common types (`0x2A6E` is temperature); for
custom data, generate a random 128-bit UUID so you don't collide with
someone else's service.

## Advertising a device

```python
import aioble
import bluetooth
import asyncio

_ENV_SENSE_UUID = bluetooth.UUID(0x181A)          # standard "Environmental Sensing"
_TEMP_CHAR_UUID = bluetooth.UUID(0x2A6E)

service = aioble.Service(_ENV_SENSE_UUID)
temp_char = aioble.Characteristic(
    service, _TEMP_CHAR_UUID, read=True, notify=True
)
aioble.register_services(service)

async def advertise_task():
    while True:
        connection = await aioble.advertise(
            250_000,                              # advertising interval, microseconds
            name="esp32-sensor",
            services=[_ENV_SENSE_UUID],
        )
        print("connected:", connection.device)
        await connection.disconnected()
        print("disconnected")

asyncio.run(advertise_task())
```

`aioble.advertise()` returns only once a **central** (the phone/scanner)
connects — it's an `await`, not a fire-and-forget broadcast loop. After
that connection drops, loop back and advertise again so the device is
discoverable for the next connection.

## Notifying characteristic updates

```python
async def sensor_task():
    while True:
        temp_c = read_temperature()               # your sensor code
        # Environmental Sensing temperature format: signed int16, x100
        temp_char.write(int(temp_c * 100).to_bytes(2, "little", True))
        temp_char.notify(connection)
        await asyncio.sleep(2)
```

`write()` sets the value a *new* reader gets; `notify()` actively pushes
the current value to anyone subscribed. Do both — a client that connects
mid-stream reads the last value via `write()`; a client already connected
gets pushed updates via `notify()`.

## Running advertise and sensor tasks together

```python
async def main():
    connection = None

    async def advertise_and_wait():
        nonlocal connection
        connection = await aioble.advertise(
            250_000, name="esp32-sensor", services=[_ENV_SENSE_UUID]
        )
        await connection.disconnected()
        connection = None

    async def notify_loop():
        nonlocal connection
        while True:
            if connection:
                temp_char.write(int(read_temperature() * 100).to_bytes(2, "little", True))
                temp_char.notify(connection)
            await asyncio.sleep(2)

    await asyncio.gather(advertise_and_wait(), notify_loop())

asyncio.run(main())
```

## aioble/asyncio gotchas

!!! warning "`notify()` before a connection exists raises `OSError`"
    Guard every `notify()` call with a connection check (as above). A
    common bug: starting the sensor loop immediately instead of waiting
    for `advertise()` to resolve, then crashing the whole `asyncio.run()`
    the moment the first reading is ready.

!!! warning "One `asyncio.run()` per program"
    Like uasyncio generally (Level 1, module 9), don't nest `asyncio.run()`
    calls or mix `await` outside an `async def`. Structure BLE + sensor +
    any other concurrent work as tasks under one `main()`.

!!! warning "Advertising stops while connected"
    A device that's already connected to one central is not advertising
    and won't be found by a second scanner. If you need multiple
    simultaneous clients, that's a different, more advanced BLE role
    (peripheral supporting multiple connections) — plan around
    one-connection-at-a-time unless you've confirmed your board's BLE
    stack supports more.

!!! warning "GATT tables are built once, at startup"
    `aioble.register_services()` must be called before advertising, and
    the service/characteristic layout can't change afterward without a
    restart. Decide your data model before writing the advertise loop.

## How It Actually Works

`aioble` sits on two layers you never see directly: the `bluetooth` module's
thin binding to the chip's Bluetooth controller firmware, and `uasyncio`'s
event loop translating BLE stack events into `await`-able Python objects.

- **The actual radio protocol work happens in a separate Bluetooth
  controller, not in your Python code.** The ESP32 (and most MicroPython
  BLE ports) run a full Bluetooth Low Energy link-layer and host stack as
  firmware, communicating with the main application core over an internal
  HCI (Host Controller Interface) transport. `bluetooth.UUID`,
  `aioble.Service`, and friends are Python objects that get serialized into
  HCI commands — "start advertising," "register this GATT table" — sent
  across that internal interface. This is why GATT tables must be built
  once before advertising starts: the controller firmware allocates its own
  internal attribute-table memory when told about your services, and most
  BLE controller firmware doesn't support redefining that table on the fly.
- **`await aioble.advertise(...)` blocks the *task*, not the interpreter,
  because advertising and connection events arrive as IRQ-driven callbacks
  from the Bluetooth stack that get bridged into `uasyncio`'s scheduler.**
  Internally, the `bluetooth` module registers a C-level IRQ handler with
  the BLE stack; when a central completes a connection, that handler posts
  an event that resolves the awaiting coroutine's future, waking the task
  on the event loop's next pass — the same "interrupt sets a flag, the loop
  picks it up" pattern from Level 1's ISR rules, just wired specifically for
  BLE connection events instead of GPIO edges.
- **`write()` and `notify()` are two genuinely different operations at the
  ATT protocol level, which is why you need both.** `write()` only updates
  the characteristic's value in the controller's local attribute table — a
  future GATT Read Request from any central will see the new value, but
  nothing is pushed anywhere. `notify()` triggers the controller to actively
  send an ATT Handle Value Notification packet over the air to a specific
  already-connected central. A client that reads on connect needs the
  former; a client that's already subscribed and listening needs the
  latter — they are not two ways of doing the same thing, they're the pull
  and push halves of the GATT model.
- **"Advertising stops while connected" is a link-layer state, not an
  aioble limitation.** A classic single-role BLE peripheral's radio can only
  be in one of a small set of link-layer states at a time (advertising,
  initiating, connected, scanning); actually supporting multiple
  simultaneous central connections while still being discoverable requires
  the controller firmware to implement multi-link scheduling, time-slicing
  the radio between roles — a controller capability aioble can only expose
  if the underlying chip's Bluetooth firmware actually supports it, which
  is why the module tells you to plan around one connection unless you've
  specifically verified otherwise.

## Cheat sheet

| Function / idiom | Purpose |
|---|---|
| `aioble.Service(uuid)` | Define a GATT service |
| `aioble.Characteristic(service, uuid, read=, notify=, write=)` | Define one value in a service |
| `aioble.register_services(service, ...)` | Finalize the GATT table before advertising |
| `await aioble.advertise(interval_us, name=, services=)` | Broadcast; resolves when a central connects |
| `connection.disconnected()` | Await until the central drops |
| `char.write(bytes)` | Set the value future readers get |
| `char.notify(connection)` | Push the current value to a subscribed client |
| `bluetooth.UUID(0x181A)` | Standard 16-bit UUID; use random 128-bit for custom data |
| `asyncio.gather(task1(), task2())` | Run advertise + sensor loops concurrently |

## Exercise

Design a **BLE sensor beacon**: advertise as `"esp32-beacon"` with a
custom 128-bit service UUID, exposing one notify-and-read characteristic
carrying a JSON-encoded string (`temp`, `hum`, `uptime_s`). Run an
`advertise_and_wait()` task and a `notify_loop()` task together with
`asyncio.gather()`, updating the characteristic every 3 seconds only while
a connection is live. Add a second, write-only characteristic that accepts
a single byte (`0`/`1`) to toggle an onboard LED, with a task that reads
new writes and applies them. Document (in comments) which real BLE scanner
app you'd use to verify this on hardware, since Wokwi's BLE support won't
run it end-to-end.
