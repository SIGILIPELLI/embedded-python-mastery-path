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
