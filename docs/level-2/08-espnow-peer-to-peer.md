# ESP-NOW & Peer-to-Peer WiFi

MQTT (module 1) needs a broker and a network — great for cloud-connected
devices, overkill for two ESP32s in the same room that just need to talk
to each other. **ESP-NOW** is Espressif's connectionless protocol:
low-latency, no router required, ESP32-to-ESP32 (or ESP32-to-ESP8266)
messaging over raw WiFi radio. This module covers pairing peers,
broadcast vs. unicast, range/reliability tradeoffs, and building a
router-free sensor mesh. The `espnow` module is reviewed against
MicroPython docs — Wokwi's WiFi simulation doesn't model raw
ESP-NOW radio frames between two simulated boards, so pairing/range
behavior described here should be validated on real hardware.

## No router, no broker — just MAC addresses

ESP-NOW devices don't join a WiFi network; they talk directly over the
WiFi radio using each other's **MAC address** as the destination. Setup
needs the radio active but not connected to an access point:

```python
import network
import espnow

sta = network.WLAN(network.STA_IF)
sta.active(True)
sta.disconnect()          # ensure not associated with any AP

e = espnow.ESPNow()
e.active(True)
```

Find your own MAC address (you'll need to give it to peers):

```python
print(sta.config('mac'))          # bytes, e.g. b'\xa4\xcf\x12\x34\x56\x78'
```

## Pairing a peer

Every device you want to send to must be added as a peer first — ESP-NOW
won't send to an unregistered MAC:

```python
peer_mac = b'\xa4\xcf\x12\x34\x56\x78'   # the receiving board's MAC
e.add_peer(peer_mac)

e.send(peer_mac, b"hello from station A")
```

## Receiving messages

```python
import espnow

e = espnow.ESPNow()
e.active(True)

while True:
    host, msg = e.recv()          # blocks until a message arrives (or timeout)
    if msg:
        print("from", host, ":", msg)
```

`e.recv(timeout_ms)` accepts an optional timeout; without one it blocks
indefinitely, which is fine in a dedicated receive loop but needs
combining with `asyncio` or a timeout if the board also has other work to
do (sensor reads, a display refresh) between messages:

```python
while True:
    host, msg = e.recv(100)      # 100 ms timeout
    if msg:
        print("from", host, ":", msg)
    do_other_periodic_work()
```

## Broadcast vs. unicast

Unicast (above) targets one known peer. **Broadcast** reaches every
ESP-NOW-listening device in range without knowing individual MACs —
useful for "who's out there?" discovery or a one-to-many sensor network:

```python
_BROADCAST_MAC = b'\xff\xff\xff\xff\xff\xff'

e.add_peer(_BROADCAST_MAC)
e.send(_BROADCAST_MAC, b"discovery ping")
```

A discovery pattern: broadcast a ping, collect MACs of everyone who
replies, then `add_peer()` each one and switch to unicast for the actual
data exchange — broadcast is good for finding peers, unicast is more
reliable for sustained traffic.

## Building a router-free sensor mesh

A simple hub-and-spoke: several sensor nodes unicast readings to one
"base station" node, which is the only one that also connects to WiFi/MQTT
(bridging ESP-NOW to the internet):

```python
# --- sensor node ---
import espnow
import network
import ujson
import time

sta = network.WLAN(network.STA_IF)
sta.active(True)

e = espnow.ESPNow()
e.active(True)
base_mac = b'\xaa\xbb\xcc\xdd\xee\xff'   # base station's MAC
e.add_peer(base_mac)

while True:
    payload = ujson.dumps({"node": "A", "temp": 21.4}).encode()
    try:
        e.send(base_mac, payload)
    except OSError as ex:
        print("send failed:", ex)         # peer out of range / not listening
    time.sleep(10)
```

```python
# --- base station: receives ESP-NOW, forwards to MQTT ---
import espnow
import network
from umqtt.simple import MQTTClient

sta = network.WLAN(network.STA_IF)
sta.active(True)
sta.connect("Wokwi-GUEST", "")            # base station DOES join WiFi

e = espnow.ESPNow()
e.active(True)

client = MQTTClient("base-station", "test.mosquitto.org")
client.connect()

while True:
    host, msg = e.recv(1000)
    if msg:
        client.publish(b"embedded-mastery/mesh/relay", msg)
```

## ESP-NOW gotchas

!!! warning "`send()` can raise even to a registered peer"
    `e.send()` raises `OSError` if the peer is out of range or powered
    off — ESP-NOW has no persistent connection to detect that in advance.
    Wrap every send in `try/except OSError` in unattended code; a single
    unreachable peer shouldn't crash a sensor loop that has other peers or
    other work to do.

!!! warning "Range is shorter and less predictable than a WiFi AP link"
    ESP-NOW typically has similar or slightly better range than
    ordinary WiFi association line-of-sight, but performance through
    walls/obstacles varies a lot by antenna and channel congestion.
    Don't assume "WiFi reaches the garage" implies "ESP-NOW reaches the
    garage" — test the actual link.

!!! warning "Peers must share a WiFi channel"
    If the base station's `sta.connect()` to a real AP happens *after*
    ESP-NOW peers were added on a different channel, sends can silently
    stop working. Set up ESP-NOW peers only after any WiFi channel is
    finalized, or explicitly pin the channel with `sta.config(channel=...)`
    on all devices.

!!! warning "No built-in retry or ack beyond `send()`'s own return"
    ESP-NOW's `send()` return value/exception tells you if the *radio*
    delivery succeeded, not whether the receiving application actually
    processed it. For a "did the base station really get this reading"
    guarantee, add your own application-level acknowledgment (base station
    ESP-NOW-sends a tiny ack back) if that matters for your use case.

## Cheat sheet

| Function / idiom | Purpose |
|---|---|
| `sta.active(True); sta.disconnect()` | Radio on, not associated with an AP |
| `espnow.ESPNow()` then `.active(True)` | Create and enable the ESP-NOW interface |
| `sta.config('mac')` | This device's MAC address, as bytes |
| `e.add_peer(mac)` | Register a peer before sending to it |
| `e.send(mac, bytes)` | Send to one peer (unicast) — can raise `OSError` |
| `b'\xff\xff\xff\xff\xff\xff'` | Broadcast MAC — reaches all listeners |
| `e.recv(timeout_ms)` | Receive; `None` on timeout instead of blocking forever |
| hub-and-spoke mesh | Only the base station needs real WiFi/MQTT |

## Exercise

Design (and document with clear comments, since this needs two physical
boards to fully verify) a **two-node ESP-NOW link**: node A broadcasts a
discovery ping every 5 seconds until it receives any reply; node B listens
for broadcasts and, on receiving one, `add_peer()`s the sender's MAC and
unicasts back an acknowledgment containing its own MAC. Once A gets the
ack, it switches to unicasting a `{"seq": n, "temp": ...}` JSON reading to
B every 2 seconds, incrementing `seq`, wrapped in `try/except OSError` so
a temporarily-out-of-range B doesn't crash A's loop. B prints every
received reading along with the gap between `seq` numbers, so dropped
packets are visible in the log. State plainly in comments which parts you
verified by reading MicroPython's `espnow` documentation versus which
require two real ESP32 boards to confirm.
