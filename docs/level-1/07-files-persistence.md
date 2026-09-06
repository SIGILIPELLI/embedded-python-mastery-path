# 07 · Files & Persistence

Everything in RAM vanishes at reset — and embedded devices reset a lot
(power loss, watchdogs, deploys). The ESP32's flash chip holds a small
filesystem that survives, and MicroPython exposes it through the same
`open()`/`read()`/`write()` API you know from desktop Python. This module
covers the filesystem, a JSON config pattern every device should use, CSV
sensor logging, and the space limits you must respect. Wokwi's simulated
ESP32 has a working filesystem too — files persist across soft resets
within a session, so everything here is runnable in the browser.

## The onboard filesystem

MicroPython formats part of the ESP32's flash (typically ~2 MB of a 4 MB
chip) as a LittleFS filesystem mounted at `/`. `boot.py` and `main.py` from
module 2 live there, and `os` works as expected:

```python
import os

print(os.listdir())            # ['boot.py', 'main.py', 'ssd1306.py', ...]
os.mkdir("data")               # directories work
os.rename("log.csv", "old.csv")
os.remove("old.csv")
print(os.getcwd())             # '/'
```

How much room is left? `os.statvfs` returns block counts — multiply out:

```python
def fs_free():
    st = os.statvfs("/")
    block_size, total_blocks, free_blocks = st[0], st[2], st[3]
    return free_blocks * block_size, total_blocks * block_size

free, total = fs_free()
print("{} of {} bytes free".format(free, total))   # e.g. 1978368 of 2097152
```

Reading and writing is standard Python — context managers and all:

```python
with open("note.txt", "w") as f:
    f.write("written at boot\n")

with open("note.txt") as f:
    print(f.read())
```

!!! warning "Flash wears out"
    Flash memory endures a limited number of erase cycles (~10k–100k per
    sector). Writing a file every 50 ms will genuinely destroy a board in
    weeks. Log at sensible intervals (seconds/minutes), batch writes, and
    keep constantly-changing state in RAM. This is a real-world failure
    mode the desktop never taught you.

## Pattern: JSON config file

Hard-coding WiFi names, pin numbers, and thresholds means re-editing code
per device. The standard fix — a `config.json` on the filesystem and the
`json` module:

```python
import json

DEFAULTS = {
    "wifi_ssid": "Wokwi-GUEST",
    "wifi_password": "",
    "read_interval_s": 5,
    "temp_alert_c": 30.0,
}

def load_config(path="config.json"):
    try:
        with open(path) as f:
            cfg = json.load(f)
    except (OSError, ValueError):      # missing file or corrupt JSON
        cfg = {}
    out = dict(DEFAULTS)
    out.update(cfg)                    # file values override defaults
    return out

def save_config(cfg, path="config.json"):
    with open(path, "w") as f:
        json.dump(cfg, f)

config = load_config()
print("reading every", config["read_interval_s"], "s")
```

Note the shape of the error handling: a fresh board with no config file
must *run anyway* with defaults, and a corrupt file must not crash the
boot. `except (OSError, ValueError)` covers both. This exact pattern
returns in the capstone.

## Pattern: CSV sensor log

Append-mode (`"a"`) writing plus one line per reading gives you a log any
spreadsheet can open:

```python
from machine import Pin
import dht, time

sensor = dht.DHT22(Pin(15))
LOG = "temps.csv"

def log_reading():
    sensor.measure()
    line = "{},{:.1f},{:.1f}\n".format(
        time.ticks_ms(), sensor.temperature(), sensor.humidity())
    with open(LOG, "a") as f:          # "a" = append, creates if missing
        f.write(line)

for _ in range(10):
    log_reading()
    time.sleep(2)

with open(LOG) as f:                   # read it back
    for line in f:
        print(line.strip())
```

(Real timestamps need a clock — the capstone uses uptime, Level 2 adds NTP.)

### Bounding the log: a poor man's ring buffer

An append-forever log eventually fills the flash and every write starts
raising `OSError: 28` (ENOSPC). Bound it. Simplest robust scheme — two
files: when the current log gets big, rename it to `.old` (deleting the
previous `.old`) and start fresh. You always retain between one and two
files' worth of recent history, and it's crash-safe:

```python
import os

MAX_BYTES = 4096

def rotate_if_needed(path="temps.csv"):
    try:
        if os.stat(path)[6] > MAX_BYTES:      # index 6 = file size
            old = path + ".old"
            try:
                os.remove(old)
            except OSError:
                pass                          # no previous .old — fine
            os.rename(path, old)
    except OSError:
        pass                                  # log doesn't exist yet — fine

rotate_if_needed()
log_reading()
```

The capstone upgrades this idea to a fixed-size in-RAM ring buffer flushed
to flash — but rotation is the workhorse you'll reuse everywhere.

## How It Actually Works

`open()` looks identical to desktop Python, but underneath it MicroPython is
talking to LittleFS, a filesystem specifically engineered for raw NOR flash
and its two defining constraints: erase-before-write, and a limited number
of erases per cell.

- **Flash can't be overwritten in place the way a hard drive can.** A NOR
  flash sector must be *erased* (every bit set to 1) as a whole block —
  typically 4 KB — before any bit in it can be written back to 0. LittleFS
  is a copy-on-write, log-structured filesystem: instead of editing a file's
  data block directly, it writes the new version into a fresh location and
  updates metadata to point at it, only reclaiming (erasing) the old block
  later. This is fundamentally different from ext4 or NTFS, and it's the
  real reason `os.stat()`/`os.statvfs()` exist as thin wrappers around
  LittleFS's own accounting rather than a generic VFS layer pretending flash
  behaves like a disk.
- **Wear-leveling is why "write every 50 ms will destroy a board" is
  literal, not hyperbole.** Each flash sector tolerates on the order of
  10,000–100,000 erase cycles before it stops reliably holding a charge.
  LittleFS spreads writes across free blocks rather than always erasing the
  same physical sector (that's "wear leveling"), which delays the failure
  but cannot eliminate it if you write constantly — the finite number of
  erase cycles is a property of the silicon's floating-gate transistors, not
  something any filesystem algorithm can work around.
- **`json.load`/`json.dump` allocate a full parse tree on the heap.** Unlike
  streaming parsers, MicroPython's `json` module builds the complete
  dict/list structure in RAM before your code sees any of it — fine for a
  small `config.json`, but a reason to never treat it as a database: parsing
  a multi-KB JSON file on a device with ~100 KB of heap competes directly
  with everything else your program has allocated, and is a common,
  hard-to-diagnose source of `MemoryError` on boot when a log file
  accidentally gets loaded as if it were config.
- **The rotate-to-`.old` pattern is a manual, file-level version of what
  LittleFS's own journal does internally at the block level** — never let
  an in-progress write be the *only* copy of anything you can't afford to
  lose. Because a power loss mid-write can leave a file truncated or (rarely,
  depending on wear and corruption) a directory entry inconsistent, LittleFS
  is designed to always recover to *some* consistent prior state after an
  unclean power-down — but "consistent" doesn't mean "your log wasn't
  truncated," which is exactly the gap the two-file rotation scheme closes
  at the application level.

## Cheat sheet

| Function / idiom | Purpose |
|---|---|
| `os.listdir()` / `mkdir` / `remove` / `rename` | Filesystem housekeeping |
| `os.stat(path)[6]` | File size in bytes |
| `os.statvfs("/")` | Blocks free/total → space arithmetic |
| `open(p, "w"/"a"/"r")` | Write / append / read — standard Python |
| `json.load(f)` / `json.dump(cfg, f)` | Config in, config out |
| `except (OSError, ValueError)` | Missing file / corrupt JSON — boot anyway |
| CSV logging | One `f.write(line)` per reading, append mode |
| Rotation | `os.rename` current → `.old` when size exceeds a cap |
| Flash wear | Limited erase cycles — write seconds apart, not milliseconds |
| Typical size | ~2 MB filesystem on a 4 MB-flash ESP32 |

## Exercise

Build a **boot counter with settings** in Wokwi: on startup, load
`config.json` (defaults: `{"blink_ms": 250, "boots": 0}`), increment
`boots`, save it back, and print "Boot #N". Then blink the pin-5 LED at
`blink_ms`. Press `Ctrl-D` in the console a few times and watch the count
climb — persistence across resets, live. Add a button (pin 4) that cycles
`blink_ms` through 100/250/1000 and saves the config on each change. Then
log every button press to `events.csv` (uptime ms + new rate) with rotation
at 1 KB, and print the log plus `fs_free()` at each boot. Finally,
hand-edit `config.json` in the Wokwi file tab to something invalid and
confirm the program still boots with defaults.
