---
description: "WiFi & HTTP — The ESP32's superpower is the radio: a $5 board that joins your WiFi and speaks HTTP turns every project into an IoT project. In MicroPython…"
---

# 08 · WiFi & HTTP

The ESP32's superpower is the radio: a $5 board that joins your WiFi and
speaks HTTP turns every project into an *IoT* project. In MicroPython the
`network` module manages the connection, `requests` fetches web APIs, and
the `socket` module lets the board *be* a web server. Wokwi simulates all
of it — the virtual ESP32 connects to a simulated access point named
`Wokwi-GUEST` (no password) and has real internet access from your browser
sandbox, so every example below runs with zero hardware.

## Connecting: network.WLAN

The chip has two WiFi interfaces: **station** (`STA_IF` — join an existing
network; what we want) and **access point** (`AP_IF` — broadcast its own).
The connect dance is: activate, connect, wait:

```python
import network
import time

def wifi_connect(ssid, password, timeout_s=15):
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    if wlan.isconnected():
        return wlan
    print("Connecting to", ssid, "...")
    wlan.connect(ssid, password)
    start = time.ticks_ms()
    while not wlan.isconnected():
        if time.ticks_diff(time.ticks_ms(), start) > timeout_s * 1000:
            raise OSError("WiFi connect timed out")
        time.sleep_ms(200)
    print("Connected, IP:", wlan.ifconfig()[0])
    return wlan

wlan = wifi_connect("Wokwi-GUEST", "")     # Wokwi's simulated open AP
```

`ifconfig()` returns `(ip, netmask, gateway, dns)`. Other handy calls:
`wlan.scan()` lists visible networks, `wlan.status()` reports progress/
failure codes, `wlan.disconnect()` drops the link. On real hardware,
replace the SSID/password with your own — ideally loaded from the
`config.json` pattern of module 7 rather than hard-coded.

!!! note "2.4 GHz only"
    The ESP32 does not see 5 GHz networks. If a real board can't find your
    WiFi, that's the first thing to check.

## HTTP client: requests

MicroPython ships a small version of the beloved `requests` library (named
`urequests` in older firmware — import whichever exists):

```python
try:
    import requests
except ImportError:
    import urequests as requests
```

**GET a public JSON API** — here, the current price of Bitcoin, then
current weather from Open-Meteo (both free, no API key):

```python
r = requests.get("https://api.coindesk.com/v1/bpi/currentprice.json")
data = r.json()                # parses straight into dicts/lists
r.close()                      # ALWAYS close — sockets are scarce on-chip
print("BTC:", data["bpi"]["USD"]["rate"])

url = ("https://api.open-meteo.com/v1/forecast"
       "?latitude=52.52&longitude=13.41&current_weather=true")
r = requests.get(url)
print("Berlin now:", r.json()["current_weather"]["temperature"], "C")
r.close()
```

**POST JSON** — the shape you'll use to push sensor readings to any
backend:

```python
import json

payload = {"device": "esp32-lab", "temp_c": 23.4}
r = requests.post("https://httpbin.org/post",
                  data=json.dumps(payload),
                  headers={"Content-Type": "application/json"})
print(r.status_code)           # 200
print(r.json()["json"])        # httpbin echoes what you sent
r.close()
```

Two habits matter more here than on the desktop: **always `close()`** the
response (the chip has a handful of sockets, not thousands), and wrap
network calls in `try/except OSError` — radios drop, DNS fails, servers
time out, and an unattended device must shrug and retry.

## HTTP server: your board as a website

Serving is just sockets: bind port 80, accept connections, read the
request, write a response. A minimal but complete status-page server:

```python
import socket
from machine import Pin

led = Pin(2, Pin.OUT)
counter = 0

PAGE = """<!DOCTYPE html>
<html><head><title>ESP32 Status</title></head>
<body><h1>ESP32 MicroPython server</h1>
<p>Visits: {visits}</p><p>LED is {led}</p>
<p><a href="/on">LED on</a> | <a href="/off">LED off</a></p>
</body></html>
"""

addr = socket.getaddrinfo("0.0.0.0", 80)[0][-1]
s = socket.socket()
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.bind(addr)
s.listen(2)
print("Listening on http://%s/" % wlan.ifconfig()[0])

while True:
    conn, client = s.accept()
    try:
        req = conn.recv(1024).decode()
        path = req.split(" ")[1] if " " in req else "/"   # "GET /on HTTP/1.1"
        if path == "/on":
            led.value(1)
        elif path == "/off":
            led.value(0)
        counter += 1
        body = PAGE.format(visits=counter,
                           led="ON" if led.value() else "OFF")
        conn.send("HTTP/1.0 200 OK\r\nContent-Type: text/html\r\n\r\n")
        conn.send(body)
    except OSError:
        pass                       # a dropped client must not kill the server
    finally:
        conn.close()
```

Point a browser at the printed IP and you're controlling a pin from a web
page. On real hardware that's any phone on your LAN; in Wokwi, the
simulated device can be reached from your machine through Wokwi's IoT
gateway (press **F1** in the editor → *Request a new IoT Gateway* — see
[Wokwi's ESP32 WiFi docs](https://docs.wokwi.com/guides/esp32-wifi)), and
the request/response traffic also shows in the serial log.

Note the server is **blocking**: while waiting in `accept()`, nothing else
runs. That's the exact problem `uasyncio` solves in module 9, and the
capstone's dashboard is this server rebuilt async.

## How It Actually Works

`network` and `socket` are thin Python wrappers over the ESP32's WiFi
firmware blob and lwIP TCP/IP stack — most of the real work happens in C
code baked into MicroPython's build, not in the interpreter.

- **The WiFi radio runs its own firmware, separate from your MicroPython
  program.** The ESP32's WiFi/Bluetooth subsystem is driven by a closed-
  source blob running on internal state machines that handle association,
  authentication, and the 802.11 MAC layer — `wlan.connect()` just posts a
  request to that subsystem and polls a status flag; the actual handshake
  (probe, auth, association, DHCP) happens asynchronously in radio firmware
  while your Python `while not wlan.isconnected()` loop spins. This is why
  connecting has a real, variable wait: `isconnected()` is checking on
  genuine RF-layer negotiation, not a Python-side state change.
- **`requests`/`urequests` builds raw HTTP text and manages a `socket`
  directly — there's no connection pooling, no keep-alive management, no
  urllib3 underneath.** Every `requests.get()` call opens a fresh TCP
  socket, sends a hand-assembled HTTP/1.0 or 1.1 request line and headers as
  bytes, and parses the response by scanning for `\r\n` — hundreds of times
  simpler than CPython's `requests`, which is exactly why `.close()`
  matters so much: the ESP32's lwIP stack has a small, fixed pool of TCP
  Control Blocks (often single digits of concurrent sockets), and a
  response object you forget to close holds one open until the underlying
  socket's own timeout eventually reclaims it — potentially locking you out
  of new connections well before you'd expect resource exhaustion on a
  desktop.
- **DNS resolution and the TCP/IP stack itself are lwIP, a C library
  compiled into the firmware, not Python.** `socket.getaddrinfo()` calls
  straight into lwIP's resolver; the three-way handshake, retransmission
  timers, and checksum computation for every packet you send with
  `conn.send()` are all lwIP state-machine code running outside the VM.
  Python only sees the boundary — bytes in, bytes out — which is also why
  `try/except OSError` is the correct universal guard: lwIP surfaces
  virtually every network failure (timeout, reset, unreachable, DNS
  failure) as the single generic `OSError` with different `errno` values,
  rather than the rich exception hierarchy `requests` gives you on desktop.
- **The blocking `accept()` server blocks because there's no event loop
  underneath it — the VM is simply parked waiting on lwIP to signal a new
  connection.** `s.accept()` calls into lwIP and yields the CPU to the
  scheduler/idle loop until a SYN packet arrives; nothing else in your
  program runs meanwhile because there is no cooperative scheduler managing
  that wait. `uasyncio` (module 9) doesn't change what lwIP does — it wraps
  the same non-blocking socket primitives lwIP exposes in a `select()`-like
  poll loop so Python code can do other things while waiting, which is the
  precise mechanical difference between this server and its async rewrite.

## Cheat sheet

| Function / idiom | Purpose |
|---|---|
| `network.WLAN(network.STA_IF)` | Station interface (join a network) |
| `wlan.active(True)` → `connect(ssid, pw)` | Bring up + join |
| `wlan.isconnected()` / `ifconfig()` | Link state / (ip, mask, gw, dns) |
| `wlan.scan()` | List visible networks |
| `"Wokwi-GUEST"`, password `""` | Wokwi's simulated access point |
| `requests.get(url)` → `.json()` → `.close()` | Fetch + parse + free the socket |
| `requests.post(url, data=..., headers=...)` | Send JSON to a backend |
| `import urequests as requests` | Fallback name on older firmware |
| `socket.socket()` … `bind` … `listen` … `accept` | Be the server |
| `SO_REUSEADDR` | Re-run without "address in use" after soft reset |
| `try/except OSError` | Around every network call — radios fail |

## Exercise

Build a **weather-mirror lamp** in Wokwi: connect to `Wokwi-GUEST`, fetch
the current temperature for your city from Open-Meteo (change the
lat/longitude in the URL), and set the pin-5 LED accordingly — PWM-dim
(module 4) proportional to temperature between 0 °C (off) and 35 °C (full
brightness). Refresh every 60 s without blocking a running status printout
(use `ticks_ms` scheduling from module 3). Then extend the web server with
a `/status` route that returns the last fetched temperature and the LED
duty as a JSON object (`Content-Type: application/json`) — test it from
your browser. Handle a failed fetch by keeping the previous value and
noting the error on serial.
