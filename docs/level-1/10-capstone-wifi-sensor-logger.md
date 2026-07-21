# 10 · Capstone — WiFi Sensor Logger

Time to combine every module into one real device: a **WiFi sensor logger**
that reads a DHT22, keeps a bounded log on flash, shows live status on an
OLED, and serves a tiny web dashboard — a JSON endpoint plus an HTML page —
from an async web server. It's structured the way real MicroPython projects
are: several small files with clear jobs, a JSON config, and `uasyncio`
tasks tying it together. The whole thing runs in a Wokwi *MicroPython on
ESP32* project with zero hardware.

## What you're building

- **Sense** — DHT22 temperature/humidity every N seconds (module 6)
- **Log** — readings into a RAM ring buffer, flushed to a CSV on flash with rotation (module 7)
- **Display** — OLED shows IP, latest reading, and reading count (module 6)
- **Serve** — `/` returns an HTML dashboard, `/api/data` returns JSON (modules 8–9)
- **Blink** — heartbeat LED so you can see the event loop is alive (module 3)

## Wiring

Create a Wokwi **MicroPython on ESP32** project and add parts with the
diagram **+** button:

| Part | Part pin | ESP32 pin |
|------|----------|-----------|
| DHT22 | VCC / GND / SDA | 3V3 / GND / **15** |
| SSD1306 OLED | VCC / GND / SCL / SDA | 3V3 / GND / **22** / **21** |
| LED (+ 220 Ω to GND) | anode via resistor | **2** |

Add the `ssd1306.py` driver as a project file (module 6 explains where to
get it), then create the four files below with the editor's new-file button.

## config.json

```json
{
  "wifi_ssid": "Wokwi-GUEST",
  "wifi_password": "",
  "read_interval_s": 5,
  "log_file": "log.csv",
  "log_max_bytes": 4096,
  "ring_size": 30
}
```

## sensors.py — reading + ring buffer + flash log

```python
# sensors.py — DHT22 sampling, in-RAM ring buffer, CSV flush with rotation
import dht
import os
import time
from machine import Pin

class SensorLog:
    def __init__(self, cfg):
        self.sensor = dht.DHT22(Pin(15))
        self.ring = [None] * cfg["ring_size"]   # pre-allocated ring buffer
        self.head = 0                           # next write slot
        self.count = 0                          # total successful readings
        self.errors = 0
        self.latest = None                      # (uptime_s, temp, hum)
        self.log_file = cfg["log_file"]
        self.log_max = cfg["log_max_bytes"]

    def read(self):
        """Take one reading. Returns True on success."""
        try:
            self.sensor.measure()
        except OSError:
            self.errors += 1
            return False
        entry = (time.ticks_ms() // 1000,
                 self.sensor.temperature(),
                 self.sensor.humidity())
        self.ring[self.head] = entry
        self.head = (self.head + 1) % len(self.ring)
        self.count += 1
        self.latest = entry
        self._append_csv(entry)
        return True

    def recent(self):
        """Ring contents, oldest first."""
        n = len(self.ring)
        items = [self.ring[(self.head + i) % n] for i in range(n)]
        return [e for e in items if e is not None]

    def _append_csv(self, entry):
        try:
            self._rotate_if_needed()
            with open(self.log_file, "a") as f:
                f.write("{},{:.1f},{:.1f}\n".format(*entry))
        except OSError:
            pass                # full/failing flash must not kill sampling

    def _rotate_if_needed(self):
        try:
            if os.stat(self.log_file)[6] > self.log_max:
                old = self.log_file + ".old"
                try:
                    os.remove(old)
                except OSError:
                    pass
                os.rename(self.log_file, old)
        except OSError:
            pass                # log doesn't exist yet
```

Design notes: the ring is **pre-allocated** (no growth, predictable
memory), a failed read only bumps `errors`, and a failing flash never stops
sampling — priorities a desktop program rarely needs.

## webapp.py — async web server

```python
# webapp.py — async HTTP: HTML dashboard + JSON API
import json
import uasyncio as asyncio

PAGE = """<!DOCTYPE html>
<html><head><title>ESP32 Sensor Logger</title>
<meta http-equiv="refresh" content="5"></head>
<body style="font-family:sans-serif">
<h1>ESP32 Sensor Logger</h1>
<p>Uptime {up} s &middot; readings {count} &middot; errors {errors}</p>
<h2>{temp} &deg;C &nbsp; {hum} %</h2>
<p><a href="/api/data">JSON API</a></p>
</body></html>
"""

class WebApp:
    def __init__(self, log):
        self.log = log

    async def handle(self, reader, writer):
        try:
            request = await reader.readline()          # e.g. b"GET /api/data HTTP/1.1"
            while await reader.readline() != b"\r\n":  # drain headers
                pass
            parts = request.split()
            path = parts[1].decode() if len(parts) > 1 else "/"

            if path == "/api/data":
                body = json.dumps({
                    "latest": self.log.latest,
                    "recent": self.log.recent(),
                    "count": self.log.count,
                    "errors": self.log.errors,
                })
                ctype = "application/json"
            else:
                t = self.log.latest
                body = PAGE.format(
                    up=t[0] if t else 0,
                    count=self.log.count, errors=self.log.errors,
                    temp="%.1f" % t[1] if t else "--",
                    hum="%.1f" % t[2] if t else "--")
                ctype = "text/html"

            writer.write("HTTP/1.0 200 OK\r\nContent-Type: {}\r\n\r\n"
                         .format(ctype).encode())
            writer.write(body.encode())
            await writer.drain()
        except OSError:
            pass                       # dropped client — server lives on
        finally:
            writer.close()
            await writer.wait_closed()
```

`asyncio.start_server` (called from `main.py`) hands each client to
`handle()` as its own task — the server never blocks the samplers, which
is exactly what module 8's blocking version couldn't do.

## main.py — wiring it all together

```python
# main.py — WiFi sensor logger capstone
import json
import network
import time
import uasyncio as asyncio
from machine import Pin, I2C

import ssd1306
from sensors import SensorLog
from webapp import WebApp

# --- config (module 7 pattern) ---
DEFAULTS = {"wifi_ssid": "Wokwi-GUEST", "wifi_password": "",
            "read_interval_s": 5, "log_file": "log.csv",
            "log_max_bytes": 4096, "ring_size": 30}
try:
    with open("config.json") as f:
        cfg = dict(DEFAULTS, **json.load(f))
except (OSError, ValueError):
    cfg = DEFAULTS

# --- hardware ---
led = Pin(2, Pin.OUT)
i2c = I2C(0, scl=Pin(22), sda=Pin(21))
oled = ssd1306.SSD1306_I2C(128, 64, i2c)
log = SensorLog(cfg)

def wifi_connect():
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    wlan.connect(cfg["wifi_ssid"], cfg["wifi_password"])
    for _ in range(100):
        if wlan.isconnected():
            return wlan.ifconfig()[0]
        time.sleep_ms(200)
    return None

# --- tasks ---
async def heartbeat():
    while True:
        led.value(not led.value())
        await asyncio.sleep_ms(500)

async def sampler():
    while True:
        log.read()
        await asyncio.sleep(cfg["read_interval_s"])

async def display(ip):
    while True:
        oled.fill(0)
        oled.text("IP:" + (ip or "offline"), 0, 0)
        if log.latest:
            oled.text("T: {:.1f} C".format(log.latest[1]), 0, 20)
            oled.text("H: {:.1f} %".format(log.latest[2]), 0, 32)
        oled.text("n={} e={}".format(log.count, log.errors), 0, 52)
        oled.show()
        await asyncio.sleep(1)

async def main():
    ip = wifi_connect()
    print("IP:", ip)
    asyncio.create_task(heartbeat())
    asyncio.create_task(sampler())
    asyncio.create_task(display(ip))
    if ip:
        app = WebApp(log)
        await asyncio.start_server(app.handle, "0.0.0.0", 80)
        print("Dashboard: http://%s/" % ip)
    while True:
        await asyncio.sleep(3600)

asyncio.run(main())
```

Boot order: load config (defaults if broken), connect WiFi with a bounded
wait (an offline device still samples and displays), then launch the
tasks. Every failure mode has a decision, not an accident.

## Test plan

Verify each claim before calling it done:

1. **Boot** — serial shows the IP; OLED shows IP + `n=0 e=0`; LED blinks
   at 1 Hz continuously (if it ever stops, a task is blocking the loop).
2. **Sensing** — drag the Wokwi DHT22 sliders; OLED updates within
   `read_interval_s`; `n=` climbs.
3. **Failure** — in `sampler()`, temporarily point the DHT22 at an empty
   pin (or raise `OSError` yourself): `e=` climbs, everything else keeps
   running. Restore it.
4. **Logging** — `Ctrl-C` to the REPL; `open("log.csv").read()` shows one
   CSV line per reading; let it run past 4 KB and confirm `log.csv.old`
   appears (`import os; os.listdir()`).
5. **Web** — via the Wokwi IoT gateway (or the board's IP on real
   hardware): `/` shows the dashboard and auto-refreshes; `/api/data`
   returns valid JSON with at most `ring_size` entries; hammering refresh
   never stops the LED heartbeat.
6. **Config** — set `read_interval_s` to 2 in `config.json`, reset,
   confirm faster sampling; corrupt the JSON, reset, confirm it boots on
   defaults.

## Cheat sheet — the patterns this project locked in

| Pattern | Where |
|---|---|
| Config file with defaults + corrupt-file fallback | `main.py` top |
| Pre-allocated ring buffer, oldest-first read-out | `sensors.py` |
| CSV append + size-based rotation | `sensors.py` |
| Sensor errors counted, never fatal | `SensorLog.read()` |
| Bounded WiFi wait; offline mode still useful | `wifi_connect()` |
| One `async` task per concern; shared state via one object | `main.py` |
| Async server: per-client handler task, always-close | `webapp.py` |
| Heartbeat LED as a liveness probe | `heartbeat()` |

## Exercises

1. **Alerts** — add `temp_alert_c` to the config; when exceeded, blink the
   heartbeat LED at 5 Hz and show `ALERT` on the OLED until it drops back.
2. **Button stats page** — add a button (pin 4) that cycles the OLED
   between the live view and a min/max/average view computed from
   `recent()`.
3. **Chart** — make `/` render the ring buffer as an inline SVG polyline
   of temperature — no JavaScript needed, just string-building server-side.
4. **Push, don't just serve** — every 60 s, POST the latest reading as
   JSON to `https://httpbin.org/post` (module 8) and count successes and
   failures on the OLED.
5. **Port it** — move the project to a Pico W in Wokwi: change the I2C
   pins and SSID, and note everything that *didn't* need changing.

---

**That's Level 1 complete.** You can now build a networked, multi-tasking,
crash-tolerant Python device from scratch. [Level 2](../level-2/index.md)
takes the same hardware into MQTT, BLE, deep sleep, and real tooling.
