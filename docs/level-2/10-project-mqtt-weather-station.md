# Project — MQTT Weather Station

This capstone combines every module in Level 2 into one battery-friendly
device: read a sensor, timestamp the reading, publish it over MQTT, show
status on a display, and sleep between cycles to stretch battery life for
weeks instead of hours. Build it in Wokwi's *MicroPython on ESP32* project
type with WiFi enabled; the deep-sleep and reset-cause behavior is
reviewed against MicroPython docs since Wokwi resets its simulation
session across a real `deepsleep()` call the same way hardware does.

## Architecture

```
boot (deep sleep wake or power-on)
  │
  ├─ check machine.reset_cause() / wake_reason() → log which kind of boot
  ├─ init I2C, DHT22, SSD1306
  ├─ connect WiFi (with retry + timeout)
  ├─ maybe_resync() NTP time (module 6 pattern — only if drift is old)
  ├─ read sensor (wrapped in try/except OSError)
  ├─ show reading on OLED
  ├─ connect MQTT, publish timestamped JSON reading (retain=True)
  ├─ log the cycle result (module 9 rotating log)
  └─ machine.deepsleep(CYCLE_MS)
```

Every step after "check reset cause" can fail without crashing the whole
device — a failed WiFi connect should still let the board sleep and try
again next cycle, not hang forever or loop-crash-reboot rapidly and drain
the battery faster than working correctly would.

## Config and constants

```python
# config.py
WIFI_SSID = "Wokwi-GUEST"
WIFI_PASSWORD = ""
MQTT_SERVER = "test.mosquitto.org"
MQTT_CLIENT_ID = "weather-station-1"
MQTT_TOPIC = b"embedded-mastery/weather-station-1/reading"
CYCLE_MS = 60_000          # sleep 60 s between readings (module exercise scale;
                            # a real deployment would use 5-15 minutes)
UTC_OFFSET_HOURS = 5.5
MAX_RESYNC_DRIFT_HOURS = 24
```

Keeping these in `config.py` instead of hardcoded in `main.py` makes the
cycle time and credentials easy to change without touching logic —
same convention as Level 1's persistence module.

## Boot sequence and reset-cause logging

```python
# main.py
import machine
import time
import sys
import io

from config import CYCLE_MS
from logger import log_event

cause = machine.reset_cause()
wake_reason = machine.wake_reason() if cause == machine.DEEPSLEEP_RESET else None
log_event("boot: cause={} wake={}".format(cause, wake_reason))
```

## Connecting WiFi with a timeout

```python
import network
import time

def connect_wifi(ssid, password, timeout_s=15):
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    if not wlan.isconnected():
        wlan.connect(ssid, password)
        start = time.time()
        while not wlan.isconnected():
            if time.time() - start > timeout_s:
                return None
            time.sleep(0.5)
    return wlan
```

Returning `None` on timeout (instead of raising) lets the caller decide
whether to skip this cycle's publish and just sleep — a station with no
WiFi in range shouldn't burn its whole `timeout_s` budget every single
cycle without a plan for what happens next.

## The main cycle

```python
from machine import Pin, I2C
import dht
import ssd1306
import ujson
import time
from umqtt.simple import MQTTClient

from config import (
    WIFI_SSID, WIFI_PASSWORD, MQTT_SERVER, MQTT_CLIENT_ID,
    MQTT_TOPIC, CYCLE_MS, UTC_OFFSET_HOURS,
)
from logger import log_event
from timekeeping import maybe_resync, timestamped_reading

i2c = I2C(0, scl=Pin(22), sda=Pin(21))
oled = ssd1306.SSD1306_I2C(128, 64, i2c)
sensor = dht.DHT22(Pin(15))

def show_status(line1, line2=""):
    oled.fill(0)
    oled.text(line1, 0, 0)
    oled.text(line2, 0, 12)
    oled.show()

def read_sensor():
    try:
        sensor.measure()
        return sensor.temperature(), sensor.humidity()
    except OSError as e:
        log_event("sensor read failed: " + str(e))
        return None

def publish_reading(reading):
    try:
        client = MQTTClient(MQTT_CLIENT_ID, MQTT_SERVER)
        client.connect()
        payload = ujson.dumps(reading).encode()
        client.publish(MQTT_TOPIC, payload, retain=True)
        client.disconnect()
        return True
    except OSError as e:
        log_event("MQTT publish failed: " + str(e))
        return False

def run_cycle():
    show_status("Connecting WiFi...")
    wlan = connect_wifi(WIFI_SSID, WIFI_PASSWORD)
    if wlan is None:
        show_status("WiFi failed", "sleeping...")
        log_event("WiFi connect timed out")
        return

    maybe_resync(MAX_RESYNC_DRIFT_HOURS)

    show_status("Reading sensor...")
    result = read_sensor()
    if result is None:
        show_status("Sensor error", "sleeping...")
        return
    temp, hum = result

    reading = timestamped_reading(temp, hum, UTC_OFFSET_HOURS)
    show_status(
        "T:{:.1f}C H:{:.0f}%".format(temp, hum),
        reading["ts"],
    )

    if publish_reading(reading):
        log_event("published: " + ujson.dumps(reading))
    else:
        show_status("MQTT failed", "sleeping...")

try:
    run_cycle()
except Exception as e:
    buf = io.StringIO()
    sys.print_exception(e, buf)
    log_event("CRASH: " + buf.getvalue())

time.sleep(1)             # let the OLED/log settle before sleeping
machine.deepsleep(CYCLE_MS)
```

The outer `try/except Exception` around `run_cycle()` is the module 9
safety net — any bug inside sensor/display/MQTT code gets logged rather
than leaving the board hung mid-cycle, and the board still reaches
`deepsleep()` afterward to try again next time.

## Battery estimate for this design

Reusing module 3's estimator:

```python
def battery_life_hours(battery_mah, active_ma, active_s, sleep_ua, sleep_s):
    cycle_s = active_s + sleep_s
    active_mah = active_ma * (active_s / 3600)
    sleep_mah = (sleep_ua / 1000) * (sleep_s / 3600)
    cycles = battery_mah / (active_mah + sleep_mah)
    return cycles * cycle_s / 3600

# ~4 s active (WiFi connect + read + publish + display) at ~150 mA,
# 56 s asleep at ~15 uA, 60 s total cycle, 2000 mAh battery
print(battery_life_hours(2000, 150, 4, 15, 56) / 24, "days")
```

This is exactly the kind of number worth computing *before* committing to
a cycle time — a 60-second cycle (chosen here so the module's demo isn't
slow to observe) trades battery life for responsiveness; a real
deployment reading weather every 10-15 minutes would last dramatically
longer on the same battery.

## Cheat sheet

| Piece | Module it came from |
|---|---|
| `dht.DHT22` read wrapped in `try/except OSError` | Level 1 module 6 + Level 2 module 9 |
| `ssd1306` status display | Level 1 module 6 |
| `connect_wifi()` with timeout | This project + Level 1 module 8 patterns |
| `maybe_resync()` / `timestamped_reading()` | Level 2 module 6 |
| `MQTTClient` publish with `retain=True` | Level 2 module 1 |
| `log_event()` rotating log | Level 2 module 9 |
| outer `try/except` + `sys.print_exception` | Level 2 module 9 |
| `machine.deepsleep(CYCLE_MS)` | Level 2 module 3 |
| `battery_life_hours()` estimate | Level 2 module 3 |

## Stretch goals

- **RTC-memory history**: keep the last 5 readings in `machine.RTC().memory()`
  (JSON-encoded) so a boot after a MQTT failure can catch up by publishing
  a small backlog once connectivity returns, instead of silently dropping
  the missed reading.
- **Adaptive cycle time**: shrink `CYCLE_MS` automatically when the
  temperature is changing quickly between readings (e.g. more than 2°C
  since last publish) and lengthen it during stable stretches, trading
  battery life for responsiveness only when something is actually
  happening.
- **ESP-NOW fallback**: if WiFi/MQTT fails twice in a row, broadcast the
  reading over ESP-NOW (module 8) to a nearby always-on relay node instead
  of dropping it, demonstrating the hub-and-spoke pattern from that
  module as a resilience layer.
- **NeoPixel status ring**: add a single NeoPixel showing cycle status at
  a glance — green flash on successful publish, yellow on a sensor
  hiccup, red on a WiFi/MQTT failure — dimmed to a low brightness to
  respect the power budget from module 7.
- **OTA-style config reload**: publish device configuration (new
  `CYCLE_MS`, new MQTT topic) to a `.../config` topic that the station
  subscribes to briefly at the start of each wake cycle before going back
  to sleep, so the field-deployed cycle time can be tuned without a
  physical reflash.
