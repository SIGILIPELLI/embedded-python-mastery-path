# NTP, RTC & Timekeeping

A board that boots up thinking it's January 1, 2000 can't timestamp
sensor readings meaningfully. This module covers syncing the clock over
the network with `ntptime`, reading/writing `machine.RTC`, handling
timezones and DST on-device (MicroPython gives you no timezone database —
you do the math), timestamping data, and keeping time across deep sleep.
Time-math helpers run via `python3`; `ntptime`/`machine.RTC` calls are
reviewed against MicroPython docs, exercised in Wokwi where its simulated
network clock allows it.

## The RTC starts wrong

On power-up, `machine.RTC()` holds whatever the last set time was (or the
epoch, on a cold boot with no battery-backed RTC) — it does **not** know
the real date/time until something sets it.

```python
import machine

rtc = machine.RTC()
print(rtc.datetime())
# (2000, 1, 1, 5, 0, 0, 0, 0) on a fresh board — obviously wrong
```

`rtc.datetime()` returns `(year, month, day, weekday, hours, minutes,
seconds, subseconds)` — note **weekday before hours**, an easy field to
transpose.

## Syncing with NTP

```python
import network
import ntptime
import time

wlan = network.WLAN(network.STA_IF)
wlan.active(True)
wlan.connect("Wokwi-GUEST", "")
while not wlan.isconnected():
    time.sleep(0.5)

ntptime.settime()          # blocks briefly; sets RTC to UTC from an NTP server
print("RTC synced:", machine.RTC().datetime())
```

`ntptime.settime()` sets the system clock to **UTC**, always — there is no
timezone parameter. It can raise `OSError` if the network request times
out or the NTP server is unreachable; wrap it and retry rather than
assuming it always succeeds:

```python
def sync_time(retries=3):
    for attempt in range(retries):
        try:
            ntptime.settime()
            return True
        except OSError as e:
            print("NTP sync failed:", e)
            time.sleep(1)
    return False
```

`ntptime.host` can be changed from the default (`pool.ntp.org`) if a
particular server is preferred:

```python
import ntptime
ntptime.host = "time.google.com"
```

## Timezones and DST — you do the math

MicroPython has no `zoneinfo`/`pytz` equivalent. After `ntptime.settime()`
sets UTC, apply an offset yourself:

```python
import time

def local_time(utc_offset_hours):
    utc_secs = time.time()
    local_secs = utc_secs + int(utc_offset_hours * 3600)
    return time.localtime(local_secs)

# IST is UTC+5:30
lt = local_time(5.5)
print("{:04d}-{:02d}-{:02d} {:02d}:{:02d}:{:02d}".format(
    lt[0], lt[1], lt[2], lt[3], lt[4], lt[5]
))
```

DST (where it applies) is just a seasonally-changing offset — track it
with a small date-range check, since there's no library to do it for you:

```python
def is_us_dst(year, month, day):
    # Simplified: DST roughly mid-March to early November.
    # Real code should compute the actual 2nd-Sunday/1st-Sunday rule.
    if month < 3 or month > 11:
        return False
    if 3 < month < 11:
        return True
    return True  # placeholder for edge months; refine per-locale
```

Treat DST logic like this as a stand-in to refine per-locale — the
placeholder above intentionally oversimplifies; a real deployment needs
the exact transition rule for its timezone.

## Timestamping sensor data

```python
import time

def timestamped_reading(temp_c, hum_pct):
    t = time.localtime()
    return {
        "ts": "{:04d}-{:02d}-{:02d}T{:02d}:{:02d}:{:02d}".format(
            t[0], t[1], t[2], t[3], t[4], t[5]
        ),
        "temp": temp_c,
        "hum": hum_pct,
    }

print(timestamped_reading(21.4, 55.0))
# {'ts': '2026-08-25T14:03:11', 'temp': 21.4, 'hum': 55.0}
```

This is plain string formatting and testable entirely with `python3` —
only the `time.localtime()` source of the numbers is board-specific.

## Keeping time across deep sleep

Deep sleep (module 3) resets the chip, but on ESP32 the RTC domain
specifically survives — `machine.RTC()` keeps ticking, so you don't need
to re-sync with NTP after every wake, only periodically:

```python
import machine
import time

rtc = machine.RTC()

def maybe_resync(max_drift_hours=24):
    # Store last-sync timestamp in RTC memory (module 3 pattern)
    last_sync_raw = rtc.memory()
    last_sync = int(last_sync_raw) if last_sync_raw else 0
    now = time.time()
    if now - last_sync > max_drift_hours * 3600:
        if sync_time():
            rtc.memory(str(time.time()).encode())

maybe_resync()
print("current time:", time.localtime())
machine.deepsleep(60_000)
```

This resyncs at most once a day, saving the WiFi-connect cost on every
short wake cycle while still catching drift before it becomes meaningful.

## Timekeeping traps

!!! warning "`ntptime.settime()` sets UTC — don't store it as local"
    Store raw timestamps in UTC (from `time.time()` right after a sync)
    and apply the offset only for display/logging. Storing already-offset
    "local" timestamps makes later math (durations, comparisons) wrong
    the next time you touch the data.

!!! warning "`machine.RTC` field order is `(year, month, day, weekday, ...)`"
    Weekday sits before hours/minutes/seconds — a very easy field to
    misindex when hand-building a `datetime()` tuple. Double check against
    the docs every time you write one.

!!! warning "Cold boot with no RTC battery loses everything"
    A board with no coin-cell RTC backup wakes from a full power loss
    (not deep sleep — actual power removal) back at the epoch. Deep sleep
    preserves RTC time; unplugging power does not. Don't assume
    "survives sleep" implies "survives being unplugged."

!!! warning "NTP over an unreliable network hangs, doesn't fail fast"
    A flaky WiFi connection can make `ntptime.settime()` take several
    seconds before raising `OSError` — budget that latency (and the
    retry loop) into any deep-sleep wake cycle that includes a resync, or
    it silently eats your battery savings.

## Cheat sheet

| Function / idiom | Purpose |
|---|---|
| `machine.RTC().datetime()` | Get/set `(year, mon, day, weekday, hr, min, sec, subsec)` |
| `ntptime.settime()` | Sync RTC to UTC via NTP; raises `OSError` on failure |
| `ntptime.host = "..."` | Change the NTP server |
| `time.time()` | Seconds since epoch, UTC |
| `time.localtime(secs)` | Break seconds into a time tuple |
| `local = utc + offset_hours * 3600` | Manual timezone conversion — no library does this |
| `rtc.memory(bytes)` / `.memory()` | Persist last-sync time across deep sleep |
| RTC domain survives deep sleep, not power loss | Resync occasionally, not every wake |

## Exercise

Write a `sync_time(retries=3)` function (as above) plus a
`timestamped_reading(temp_c, hum_pct, utc_offset_hours=5.5)` function that
returns a dict with an ISO-style local timestamp string. Test both the
timestamp-formatting logic and a `local_time(utc_offset_hours)` helper
using `python3` with a fixed, hardcoded `time.time()`-like value (pass the
seconds in directly rather than calling the real clock) so the test is
deterministic. Then write the on-device integration: connect WiFi, call
`sync_time()`, and if it succeeds, print one `timestamped_reading()` per
second for 10 seconds. Add the `maybe_resync()` RTC-memory pattern so a
version of this script survives being wrapped in a `machine.deepsleep()`
loop without resyncing more than once per (simulated) day.
