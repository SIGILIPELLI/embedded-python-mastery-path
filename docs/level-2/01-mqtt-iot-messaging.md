---
description: "MQTT & IoT Messaging — Sensors are only useful once their data leaves the board. MQTT is the lightweight publish/subscribe protocol that IoT devices use…"
---

# MQTT & IoT Messaging

Sensors are only useful once their data leaves the board. **MQTT** is the
lightweight publish/subscribe protocol that IoT devices use to talk to the
cloud (or to each other) over a single TCP connection: a device *publishes*
to a topic, a *broker* fans it out to everyone *subscribed* to that topic.
This module uses `umqtt.simple` (and its sturdier cousin `umqtt.robust`) to
publish sensor readings and subscribe to command topics, in Wokwi's
*MicroPython on ESP32* project type with WiFi enabled.

## Publish/subscribe in 60 seconds

No direct device-to-device connection — everyone talks to the **broker**.
A topic is just a slash-separated string (`home/room1/temp`); publishers
and subscribers don't need to know about each other, only the topic name.
Public test brokers like `test.mosquitto.org` (port 1883, no TLS) are handy
for learning — never for production or private data.

```python
import network
import time

wlan = network.WLAN(network.STA_IF)
wlan.active(True)
wlan.connect("Wokwi-GUEST", "")
while not wlan.isconnected():
    time.sleep(0.5)
print("WiFi:", wlan.ifconfig()[0])
```

`umqtt.simple` isn't built into the firmware — install it once the board
is online:

```python
import mip
mip.install("umqtt.simple")
```

## Publishing sensor data

```python
from umqtt.simple import MQTTClient
import time
import ujson

client = MQTTClient(
    client_id="esp32-station-1",
    server="test.mosquitto.org",
    port=1883,
)
client.connect()
print("MQTT connected")

while True:
    payload = ujson.dumps({"temp": 21.4, "hum": 55.0})
    client.publish(b"embedded-mastery/demo/sensor", payload.encode())
    print("published:", payload)
    time.sleep(5)
```

`client_id` must be unique per device on the broker — two devices sharing
one ID will fight, each disconnecting the other. `publish()` and topic
names take `bytes`, not `str`, on `umqtt.simple`; `.encode()` your payload
and prefix topic literals with `b"..."` or you'll get a `TypeError` deep in
the socket code that doesn't mention encoding at all.

## Subscribing to commands

Real devices don't just report — they take orders. A callback fires for
every message on a subscribed topic:

```python
from umqtt.simple import MQTTClient
import time

def on_message(topic, msg):
    print("got:", topic, msg)
    if msg == b"ON":
        led.value(1)
    elif msg == b"OFF":
        led.value(0)

client = MQTTClient("esp32-station-1", "test.mosquitto.org")
client.set_callback(on_message)
client.connect()
client.subscribe(b"embedded-mastery/demo/cmd")

while True:
    client.check_msg()      # non-blocking: returns immediately if nothing waiting
    time.sleep(0.2)
```

`check_msg()` polls once and returns; `wait_msg()` blocks until a message
arrives. In a loop that also needs to publish periodically, `check_msg()`
is almost always the right choice — `wait_msg()` will stall your publish
schedule waiting for a command that may never come.

## QoS, retain, and last will

```python
client.publish(topic, payload, retain=True, qos=0)
```

- **QoS 0** ("fire and forget") is what `umqtt.simple` supports well —
  fine for frequent sensor readings where one dropped sample doesn't
  matter. QoS 1/2 need more broker/client bookkeeping; `umqtt.robust`
  handles reconnects but not full QoS 2 semantics.
- **retain=True** keeps the last message on that topic at the broker, so a
  client subscribing later immediately gets the last known value instead
  of waiting for the next publish — useful for a "current status" topic.
- **Last will**: register a message the broker publishes *on your behalf*
  if your device drops off ungracefully (power loss, WiFi failure):

```python
client = MQTTClient(
    "esp32-station-1", "test.mosquitto.org",
    keepalive=60,
)
client.set_last_will(b"embedded-mastery/demo/status", b"offline", retain=True)
client.connect()
client.publish(b"embedded-mastery/demo/status", b"online", retain=True)
```

## Reconnect strategy

WiFi and brokers both drop connections. `umqtt.simple` raises `OSError` on
a lost connection rather than retrying — you own the retry loop:

```python
def connect_mqtt():
    while True:
        try:
            c = MQTTClient("esp32-station-1", "test.mosquitto.org")
            c.connect()
            print("MQTT connected")
            return c
        except OSError as e:
            print("MQTT connect failed, retrying:", e)
            time.sleep(2)

client = connect_mqtt()

while True:
    try:
        client.publish(b"embedded-mastery/demo/sensor", b'{"temp":21.4}')
        time.sleep(5)
    except OSError as e:
        print("publish failed, reconnecting:", e)
        client = connect_mqtt()
```

`umqtt.robust` (`mip.install("umqtt.robust")`) wraps this pattern for you
— its `MQTTClient` retries publishes internally — but understanding the
manual version matters: robust's retries can still block for a while on a
truly dead network, and on a battery device that's a cost you need to know
you're paying.

!!! warning "Don't reconnect inside the callback"
    `on_message` runs while `check_msg()` is still inside the MQTT
    library's socket-handling code. Calling `client.connect()` or
    `client.publish()` from inside the callback can corrupt the client's
    internal state. Set a flag in the callback and act on it in the main
    loop instead.

## How It Actually Works

`umqtt.simple` is a small, deliberately minimal Python module (a few hundred
lines, readable in full on the filesystem after `mip.install`) sitting
directly on top of a raw TCP `socket` — there's no separate MQTT daemon or
background thread doing the protocol work for you.

- **MQTT is a byte-level binary protocol, and `client.connect()`/`publish()`
  hand-assemble its packets.** Each MQTT operation (CONNECT, PUBLISH,
  SUBSCRIBE) has a fixed binary header format — a control-packet type
  nibble, flags, and a variable-length remaining-length field encoded as
  1–4 bytes using a 7-bits-per-byte continuation scheme. `umqtt.simple`
  builds these byte strings directly with `struct`-style packing and writes
  them straight to the socket; this is why `publish()` demands `bytes` for
  topic and payload rather than accepting `str` and encoding for you —
  every extra convenience layer costs RAM and code size the library is
  built to avoid.
- **`check_msg()` doing a non-blocking read is really a `socket.settimeout(0)`
  read wrapped in a try/except.** Calling it repeatedly in your main loop is
  literally polling the TCP socket's receive buffer for whether the OS-level
  (well, lwIP-level, per module 8) network stack has assembled a complete
  MQTT packet yet. `wait_msg()` is the same code path but with a blocking
  socket read — the *only* difference between the two functions is whether
  the underlying `recv()` call is allowed to block the whole VM waiting for
  bytes that haven't arrived, which is exactly why mixing `wait_msg()` into
  a loop that also needs to publish on a timer stalls the publishes.
- **The warning against reconnecting inside `on_message` is a reentrancy
  rule, structurally identical to the ISR allocation ban from Level 1
  module 5.** `check_msg()` is mid-way through parsing a packet's bytes off
  the socket (tracking how many more bytes of payload it still expects) when
  it calls your callback; calling `client.connect()` from inside that
  callback tears down and rebuilds the very socket `check_msg()`'s calling
  frame still has a live reference to, corrupting its read-state exactly the
  way allocating inside an ISR corrupts the GC's read-state — a "don't call
  back into code that's currently calling you" hazard, whether the caller is
  a hardware interrupt or a still-unwinding parse loop.
- **QoS 0 is "well supported" because it needs no bookkeeping at all** —
  publish the bytes, forget them, no acknowledgment round-trip, no retry
  queue, no packet-ID tracking. QoS 1/2 require the client to remember
  in-flight packet IDs and retransmit on missing PUBACK/PUBREC — real state
  that has to survive across `check_msg()` calls and, ideally, across
  reconnects. `umqtt.robust`'s reconnect wrapping is a much smaller ask than
  full QoS 2 semantics, which is exactly why the library draws the line
  where it does rather than reimplementing the full MQTT spec on a
  100 KB-heap device.

## Cheat sheet

| Function / idiom | Purpose |
|---|---|
| `mip.install("umqtt.simple")` | Install the MQTT client library |
| `MQTTClient(client_id, server, port=1883)` | Create a client — `client_id` must be unique |
| `client.connect()` | Open the connection to the broker |
| `client.publish(topic, msg, retain=False, qos=0)` | Send a message — topic/msg must be `bytes` |
| `client.subscribe(topic)` | Register interest in a topic |
| `client.set_callback(fn)` | `fn(topic, msg)` fires per incoming message |
| `client.check_msg()` | Poll once, non-blocking |
| `client.wait_msg()` | Block until one message arrives |
| `client.set_last_will(topic, msg, retain=True)` | Broker publishes this if the device vanishes |
| `OSError` on publish/connect | Network/broker dropped — reconnect, don't crash |

## 🔀 Related lessons on other tracks

- [Embedded — MQTT & IoT Messaging](https://sigilipelli.github.io/embedded-mastery-path/level-2/04-mqtt-iot-messaging/)
- [NodeMCU/IoT — 01 · MQTT Basics for IoT Messaging](https://sigilipelli.github.io/nodemcu-mastery-path/level-2/01-mqtt-basics/)

## Exercise

Build a **remote-controlled sensor node** in Wokwi: a DHT22 on pin 15 and
an LED on pin 2. Every 5 seconds, publish `{"temp": ..., "hum": ...}` as
JSON to `embedded-mastery/<yourname>/sensor` with `retain=True`. Subscribe
to `embedded-mastery/<yourname>/led` and toggle the LED based on `b"ON"` /
`b"OFF"` commands, using `check_msg()` so publishing keeps running while
listening. Register a last will of `b"offline"` (retained) on a `.../status`
topic, and publish `b"online"` (retained) right after connecting. Wrap the
publish loop in a reconnect-on-`OSError` handler so a simulated WiFi drop
(toggle WiFi off/on in Wokwi) recovers within a few seconds instead of
crashing the program.
