---
title: "Embedded Python Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free Embedded Python cheat sheet: 40 quick-reference tables from every level of the Embedded Python Mastery Path, each linked to its full lesson."
---

# Embedded Python Cheat Sheet

Quick-reference tables from **40 lessons** of the free Embedded Python Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [01 · MicroPython vs. Python](level-1/01-micropython-vs-python.md)

| Concept | The short version |
|---|---|
| MicroPython | Python 3 reimplemented to run on microcontrollers in ~256 KB |
| stdlib | Subset: `json`, `os`, `time`, `struct`, `socket`... — no big desktop libs |
| `u`-modules | Old names (`ujson`, `utime`) — now aliases for the plain names |
| `machine` module | The hardware API: `Pin`, `PWM`, `ADC`, `I2C`, `SPI`, `Timer` |
| `gc.mem_free()` | Bytes of heap remaining — memory is a real budget here |
| Boards | ESP32 (primary), Raspberry Pi Pico (RP2040), ESP8266, Pyboard |
| CircuitPython | Adafruit fork: `board`/`digitalio` API, USB-drive workflow, no `machine` |
| Firmware flashing | Once per board: `esptool.py write_flash 0x1000 firmware.bin` (ESP32) |
| Zero hardware | Wokwi → New Project → **MicroPython on ESP32** — free, in-browser |
| Package install | `mip` (on-device) from micropython-lib, not `pip` |

### [02 · REPL &amp; Workflow](level-1/02-repl-workflow.md)

| Command / key | Purpose |
|---|---|
| `Ctrl-C` | Interrupt running code → REPL prompt |
| `Ctrl-D` (at REPL) | Soft reset — reruns `boot.py` then `main.py` |
| `Ctrl-E` … `Ctrl-D` | Paste mode for multi-line code |
| `boot.py` | Runs first at startup — keep minimal |
| `main.py` | Your app — auto-runs on every boot |
| `mpremote` | Open REPL on auto-detected board |
| `mpremote run file.py` | Execute local file on board without saving |
| `mpremote fs cp f.py :f.py` | Upload file to board flash (`:` = board) |
| `mpremote fs ls` / `rm` | List / delete board files |
| `machine.reset()` | Hard reset from code |
| Thonny | GUI IDE: REPL + file manager + firmware flasher |

### [03 · GPIO Basics](level-1/03-gpio-basics.md)

| Function / idiom | Purpose |
|---|---|
| `Pin(n, Pin.OUT)` | Configure GPIO *n* as output |
| `Pin(n, Pin.IN, Pin.PULL_UP)` | Input with internal pull-up — idle `1`, pressed `0` |
| `p.value(1)` / `p.value(0)` | Drive high / low (also `p.on()` / `p.off()`) |
| `p.value()` | Read the pin (returns `0` or `1`) |
| `p.value(not p.value())` | Toggle |
| LED wiring | pin → 220 Ω resistor → LED anode; cathode → GND |
| Button wiring (pull-up) | pin → button → GND; pressed reads `0` |
| Floating input | Unconnected input reads noise — always define idle state |
| `time.ticks_ms()` / `ticks_diff()` | Overflow-safe elapsed-time math |
| Bounce | Contacts chatter ~1–10 ms; require ~30 ms of stability |
| ESP32 pin notes | 34–39 input-only; avoid 0/2/12/15 for buttons |

### [04 · PWM &amp; ADC](level-1/04-pwm-adc.md)

| Function / idiom | Purpose |
|---|---|
| `PWM(Pin(n), freq=1000)` | Create PWM output on pin *n* |
| `pwm.duty_u16(0..65535)` | Set duty cycle (portable ESP32/Pico API) |
| `pwm.freq(hz)` | Change frequency (tones: the note; LEDs: keep ≥ ~200 Hz) |
| `pwm.deinit()` | Release the pin back to plain GPIO |
| Servo | `freq=50`, pulse 0.5–2.4 ms mapped into `duty_u16` |
| `ADC(Pin(34))` | Create ADC reader (ESP32: use GPIO 32–39) |
| `adc.atten(ADC.ATTN_11DB)` | ESP32 only: extend range to ~3.3 V |
| `adc.read()` | 0–4095 (12-bit, ESP32) |
| `adc.read_u16()` | 0–65535 — portable across ports |
| Noise | Average 8–16 readings |
| `scale()` helper | Map a reading from one range to another |

### [05 · Timers &amp; Interrupts](level-1/05-timers-interrupts.md)

| Function / idiom | Purpose |
|---|---|
| `Timer(0).init(period=500, mode=Timer.PERIODIC, callback=f)` | Call `f` every 500 ms |
| `Timer.ONE_SHOT` | Fire once after `period` ms |
| `tim.deinit()` | Stop a timer |
| `pin.irq(trigger=Pin.IRQ_FALLING, handler=f)` | Call `f` on 1→0 edge |
| `Pin.IRQ_RISING`, `FALLING`, or both OR'd | Edge selection |
| ISR ban | No allocation: no prints, floats, list growth, str building |
| Flag pattern | ISR sets a global int/bool; main loop does the real work |
| `micropython.schedule(f, arg)` | Queue `f` to run soon, outside ISR context |
| `micropython.alloc_emergency_exception_buf(100)` | Readable ISR tracebacks |
| ISR debounce | Ignore edges < ~200 ms since last accepted |

### [06 · I2C &amp; SPI Sensors](level-1/06-i2c-spi-sensors.md)

| Function / idiom | Purpose |
|---|---|
| `I2C(0, scl=Pin(22), sda=Pin(21), freq=400000)` | Hardware I2C bus |
| `SoftI2C(scl=..., sda=...)` | Bit-banged I2C on any pins |
| `i2c.scan()` | List responding addresses — debugging tool #1 |
| `i2c.readfrom_mem(addr, reg, n)` | Read *n* bytes from a device register |
| `i2c.writeto_mem(addr, reg, bytes)` | Write to a device register |
| `dht.DHT22(Pin(15))` → `measure()` | Built-in temp/humidity driver (2 s max rate) |
| `mip.install("ssd1306")` | Install a driver from micropython-lib (needs WiFi) |
| `ssd1306.SSD1306_I2C(128, 64, i2c)` | OLED at 0x3C |
| `oled.fill(0)` / `text()` / `show()` | Clear, draw, then *push* to the panel |
| SPI | SCK/MOSI/MISO + one CS per device; CS is active-low |

### [07 · Files &amp; Persistence](level-1/07-files-persistence.md)

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

### [08 · WiFi &amp; HTTP](level-1/08-wifi-http.md)

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

### [09 · uasyncio Basics](level-1/09-uasyncio-basics.md)

| Function / idiom | Purpose |
|---|---|
| `import uasyncio as asyncio` | Works on all firmware (plain `asyncio` on recent) |
| `async def f(): ...` | Define a coroutine |
| `await asyncio.sleep_ms(n)` / `sleep(s)` | Pause this task, run the others |
| `asyncio.create_task(coro())` | Start a concurrent task |
| `asyncio.run(main())` | Start the event loop |
| `asyncio.gather(a(), b())` | Run several, wait for all |
| `asyncio.wait_for(coro(), s)` | Timeout → raises `asyncio.TimeoutError` |
| `time.sleep()` in a task | **Never** — blocks the whole loop |
| Long computation | `await asyncio.sleep_ms(0)` periodically to yield |
| Shared state | Plain dicts/objects — single-threaded, no locks needed |
| vs `millis()` pattern | Same scheduling idea, but per-task code stays linear |

### [10 · Capstone — WiFi Sensor Logger](level-1/10-capstone-wifi-sensor-logger.md)

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

## Level 2 · Intermediate

### [MQTT &amp; IoT Messaging](level-2/01-mqtt-iot-messaging.md)

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

### [BLE with aioble](level-2/02-ble-aioble.md)

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

### [Deep Sleep &amp; Battery Power](level-2/03-deep-sleep-battery.md)

| Function / idiom | Purpose |
|---|---|
| `machine.deepsleep(ms)` | Sleep and reset; program restarts from the top |
| `machine.lightsleep(ms)` | Sleep, keep RAM, resume after the call |
| `machine.reset_cause()` | `DEEPSLEEP_RESET`, `PWRON_RESET`, etc. |
| `machine.wake_reason()` | `PIN_WAKE`, `TIMER_WAKE`, `TOUCHPAD_WAKE` |
| `esp32.wake_on_ext0(pin, level)` | Wake on one pin |
| `esp32.wake_on_ext1(pins, level)` | Wake on any of several pins |
| `esp32.wake_on_touch(True)` | Wake on a touch pin |
| `machine.RTC().memory(bytes)` / `.memory()` | Small state store that survives deep sleep |
| duty cycle = active_s / (active_s + sleep_s) | Drives battery life more than raw sleep current |

### [Writing Custom Sensor Drivers](level-2/04-custom-sensor-drivers.md)

| Code | Meaning | Bytes |
|---|---|---|
| `<` / `>` | little-endian / big-endian | — |
| `b` / `B` | signed / unsigned byte | 1 |
| `h` / `H` | signed / unsigned short | 2 |
| `i` / `I` | signed / unsigned int | 4 |
| `f` | float | 4 |

Test the unpacking logic on made-up bytes before ever touching hardware —
this part is pure Python and runs anywhere:

```python
import struct
raw = bytes([0x10, 0x00, 0xF0, 0xFF, 0x00, 0x00])   # 16, -16, 0
print(struct.unpack("<hhh", raw))                    # (16, -16, 0)
```

### [mpremote &amp; mip — Tooling Deep Dive](level-2/05-mpremote-mip-tooling.md)

| Command | Purpose |
|---|---|
| `mpremote connect list` | List connected boards/ports |
| `mpremote` | Open interactive REPL |
| `mpremote run file.py` | Execute a local file on-board, in RAM only |
| `mpremote cp file.py :file.py` | Copy a file to the board's filesystem |
| `mpremote cp -r dir/ :dir/` | Copy a directory recursively |
| `mpremote fs ls [:path]` | List files on the board |
| `mpremote fs rm :file` | Delete a file on the board |
| `mpremote mount ./dir` | Live-mount a local dir as the board's filesystem |
| `mpremote reset` | Soft reset — re-runs flash code |
| `a + b + c` (chained) | Run several `mpremote` steps in one call |
| `mip.install("name")` (on-board) | Install from micropython-lib/GitHub to `/lib` |
| `mpremote mip install --target ./lib name` | Install to a local dir from your computer |

### [NTP, RTC &amp; Timekeeping](level-2/06-ntp-rtc-timekeeping.md)

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

### [NeoPixels, Displays &amp; Device UI](level-2/07-neopixels-displays-ui.md)

| Function / idiom | Purpose |
|---|---|
| `neopixel.NeoPixel(pin, n)` | Create a strip of *n* pixels on `pin` |
| `np[i] = (r, g, b)` | Set pixel *i* in the buffer |
| `np.write()` | Push the buffer to the strip — nothing shows without it |
| `np.fill((r, g, b))` | Set all pixels at once |
| `oled.pixel/line/rect/fill_rect/text` | `framebuf` drawing primitives |
| `oled.show()` | Push the framebuffer to the display |
| falling-edge check + `sleep_ms()` | Crude but usable button debounce |
| redraw on state change, not every loop | Avoids flicker and wasted I2C traffic |

### [ESP-NOW &amp; Peer-to-Peer WiFi](level-2/08-espnow-peer-to-peer.md)

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

### [Error Handling &amp; On-Device Logging](level-2/09-error-handling-logging.md)

| Function / idiom | Purpose |
|---|---|
| `sys.print_exception(e)` | Print a full traceback without crashing |
| `sys.print_exception(e, io.StringIO())` | Capture the traceback as a string |
| rotating `app.log` / `app.log.old` | Cap flash usage from logging |
| `machine.reset_cause()` | `PWRON_RESET`, `HARD_RESET`, `WDT_RESET`, `DEEPSLEEP_RESET` |
| `machine.reset()` | Force a clean reboot after logging a crash |
| `machine.WDT(timeout=ms)` | Reset the board if `feed()` isn't called in time |
| `wdt.feed()` | Prove the main loop is alive; call every iteration |
| catch specific exceptions (`OSError`), not bare `except:` | Don't hide real bugs |

### [Project — MQTT Weather Station](level-2/10-project-mqtt-weather-station.md)

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

## Level 3 · Advanced

### [Memory Management &amp; the Garbage Collector](level-3/01-memory-gc.md)

| Tool / pattern | Effect |
|---|---|
| `gc.collect()` | Force a full mark-sweep now, on your schedule |
| `gc.mem_free()` / `gc.mem_alloc()` | Query heap state (call after `collect()` for accuracy) |
| `const()` | Compile-time inline for module/class-level int constants — no lookup, no separate object |
| `bytearray` + `readinto()` | Reuse one buffer instead of allocating per read |
| Pre-allocate big buffers at startup | Avoids fragmentation from later small allocations |
| `gc.threshold(n)` | Tune how often automatic collection fires |
| `gc.disable()` | Stops automatic collection — you own calling `gc.collect()` |

### [Performance Profiling &amp; Optimization](level-3/02-performance-profiling.md)

| Technique | Saves |
|---|---|
| `time.ticks_diff(new, old)` | Correct timing across tick wraparound |
| Bind method/function to a local before a loop | Repeated attribute/global lookup |
| `array.array` for numeric buffers | Per-element object boxing overhead |
| Preallocate + index-assign vs. repeated `.append()` | Reallocation/copy churn |
| `sum()`, `min()`, `max()` built-ins | C-level loop vs. Python-level loop |
| Average over N calls | Noise from IRQs and occasional GC pauses |

### [Native &amp; Viper Code Emitters](level-3/03-native-viper-emitters.md)

| Emitter | Typical speedup | Type system | Bounds-checked | Risk |
|---|---|---|---|---|
| (default, bytecode) | 1x baseline | Full Python | Yes | None beyond normal Python bugs |
| `@micropython.native` | ~2x | Full Python | Yes | Low — same semantics, just compiled |
| `@micropython.viper` | ~10x+ | Machine `int`/`uint`, typed pointers | No, for `ptr*` access | High — overflow wraps silently, `ptr*` has no bounds checks |

### [Writing C Modules for MicroPython](level-3/04-c-modules.md)

| Need | Reach for |
|---|---|
| Fast integer/pointer loop, still Python source | `@micropython.viper` |
| Call an existing C/C++ library | User C module |
| A stateful type with methods, backed by a C struct | User C module with `make_new`/`locals_dict` |
| Raising errors from native code | `mp_raise_ValueError`/`mp_raise_TypeError`/etc., not `abort()` or C `errno` |
| Allocating memory the GC must track | `mp_obj_malloc`, never bare `malloc` for objects handed to Python |

### [RP2040 PIO — Programmable I/O](level-3/05-rp2040-pio.md)

| Concept | What it means |
|---|---|
| Program memory | 32 instruction slots per state machine — hard limit, no exceptions |
| `[n]` delay | Up to 31 extra cycles after an instruction, avoids spending slots on waiting |
| `.side(n)` | Drive side-set pins from the same instruction as its main operation |
| `wrap_target()`/`wrap()` | Loop without a `jmp` instruction, free-running programs |
| TX/RX FIFO | 4 words deep each — the only data channel between PIO and Python |
| `autopull`/`autopush` + `pull_thresh` | Controls when FIFO transfers happen automatically |
| `sm.irq(handler)` | PIO-to-CPU signaling; handler runs as an interrupt-context callback |

### [Advanced uasyncio Patterns](level-3/06-advanced-uasyncio.md)

| Primitive | Use for |
|---|---|
| `Event` | One-to-many, "something happened" signaling |
| `Lock` | Protecting a critical section that spans an `await` |
| `Queue(maxsize=n)` | Producer/consumer with automatic backpressure |
| `StreamReader`/`StreamWriter` | Socket protocols; always pair `write()` with `drain()` |
| `wait_for(coro, timeout=...)` | Bounding how long a coroutine can run |
| `finally` around cancellable code | Guaranteed cleanup regardless of cancellation |
| `asyncio.create_task()` + `gather()` | Structuring several independent long-running loops |

### [The Inline Assembler](level-3/07-inline-assembler.md)

| Aspect | Detail |
|---|---|
| Decorator | `@micropython.asm_thumb` (ARM Cortex-M targets only) |
| Arguments | Up to 3 (some ports 4), passed in `r0`, `r1`, `r2` |
| Return value | Whatever is in `r0` at function exit |
| Scratch registers | `r0`-`r3` freely usable; `r4`+ must be saved/restored if touched |
| Type system | None — every value is a raw register bit pattern |
| Portability | ARM Cortex-M only; not Xtensa (ESP32), not RP2040 (use PIO there) |
| When to use | Last resort, after native and viper both fall short |

### [Frozen Modules &amp; Custom Manifests](level-3/08-frozen-modules-manifests.md)

| Concept | Detail |
|---|---|
| `manifest.py` | Build-time DSL controlling what gets frozen |
| `freeze(path, files=None)` | Compile and embed `.py` files as bytecode |
| `include(other_manifest)` | Compose manifests (how stdlib pieces get pulled in) |
| Frozen bytecode location | Flash, executed in place on most ports — not heap |
| Filesystem bytecode location | Heap, for the life of the import |
| Update path | Reflash firmware (frozen) vs. copy a file (filesystem) |
| Import precedence | Port-specific — verify, don't assume, when both exist |

### [Threads &amp; Dual-Core MicroPython](level-3/09-threads-dual-core.md)

| Concept | Detail |
|---|---|
| `_thread.start_new_thread(fn, args)` | Runs `fn` on the second core (RP2040, ESP32) |
| MicroPython's GIL | Protects interpreter internals, not your program's shared data |
| `counter += 1` | Not atomic — read/modify/write across separate bytecode ops |
| `_thread.allocate_lock()` | Mutual exclusion for shared Python state or shared peripherals |
| Bus peripherals (I2C/SPI) shared across threads | Always lock around the full multi-step transaction |
| Default choice | `uasyncio` for I/O-bound concurrency; threads only for sustained CPU-bound work |

### [Project — High-Speed Data Acquisition](level-3/10-project-high-speed-daq.md)

| Component | Module it draws from |
|---|---|
| PIO sample-rate pacing | 05 — RP2040 PIO |
| Preallocated ring buffer | 01 — Memory & GC |
| Cross-core lock around the buffer | 09 — Threads & dual-core |
| Rolling stats/checksum in viper | 03 — Native & viper emitters |
| Host streaming loop structure | 06 — Advanced uasyncio |
| `gc.collect()` at flush boundaries only | 01 & 02 — GC timing, profiling methodology |

## Level 4 · Master

### [Production Firmware Architecture](level-4/01-production-architecture.md)

| Principle | What it prevents |
|---|---|
| Drivers know nothing of business logic | Untestable, hardware-coupled application code |
| Config merged with defaults, then validated | Bricking on missing/partial config |
| Explicit state machine + transition table | Unreachable-in-theory state combinations happening in practice |
| Dependency injection into services | Business logic that can only be tested on real hardware |
| Hardware imports isolated to drivers | Whole modules unable to run/test on desktop |

### [OTA Updates for MicroPython Fleets](level-4/02-ota-updates-fleets.md)

| Concept | Detail |
|---|---|
| File-level update | Replace `.py`/`.mpy` on the filesystem; doesn't touch firmware |
| Firmware-image update | Full OTA flash; ESP32 uses dual A/B partitions, RP2040 typically a full UF2 reflash |
| Tuple version comparison | Correct ordering without string-comparison pitfalls |
| Confirm-or-rollback | Mark new firmware pending until it proves it boots and runs healthily |
| Staged rollout via hashed device ID | Stable per-device bucket, no flip-flopping between checks |
| Verify-then-atomic-rename | Never leave a partially-written file as the live one |

### [Device Provisioning &amp; Secrets Management](level-4/03-provisioning-secrets.md)

| Concern | Approach |
|---|---|
| WiFi/API secrets identical across units | Never — always provisioned, never frozen into firmware |
| Getting real network credentials onto a screenless device | AP-mode captive portal or BLE GATT write, both time-boxed |
| Per-device identity | `machine.unique_id()` — survives reflash and factory reset |
| Secrets on the filesystem | Filesystem storage keeps them out of the shared image; not automatically encrypted at rest |
| Flagging secret files | An explicit allowlist/denylist so backup/log/debug paths don't leak them |

### [Watchdogs, Brownout &amp; Robustness Patterns](level-4/04-watchdogs-robustness.md)

| Mechanism | Catches |
|---|---|
| `machine.WDT` | Hangs/deadlocks — anything that stops forward progress entirely |
| Supervisor with bounded retry | Transient exceptions in application logic, without a full reset |
| Write-to-temp-then-rename | Brownout/power-loss corruption of persisted files |
| `boot.py` crash counter | Fast, repeated crashes a watchdog timeout wouldn't catch |
| Safe mode (skip `main.py`) | Recovering a device whose own application code can't run at all |

### [Custom Firmware Builds &amp; Board Definitions](level-4/05-custom-firmware-builds.md)

| Concern | Where it's controlled |
|---|---|
| Board name, enabled subsystems, heap size | `mpconfigboard.h` |
| Pin-to-peripheral mapping | Board's `pins.csv` or equivalent |
| Flash partitioning (OTA slots vs. filesystem) | `partitions.csv` (ESP-IDF-style ports) |
| Frozen modules for this board specifically | Board-local `manifest.py` |
| Reproducibility | Pinned MicroPython commit + pinned toolchain version + CI build |

### [Security Hardening — TLS &amp; Secure Storage](level-4/06-security-hardening.md)

| Hardening step | Defends against |
|---|---|
| Real cert verification, not `CERT_NONE` | Man-in-the-middle on the network path |
| Pinned CA/cert, ECC over RSA | Handshake cost and memory on constrained hardware |
| Chip-level flash encryption (where available) | Physical flash extraction reading secrets in plaintext |
| Disabled/gated REPL and JTAG in production | Physical-access code execution and flash dumping |
| Per-device credentials, not shared | One compromised unit becoming a fleet-wide compromise |

### [Fleet Management &amp; Remote Monitoring](level-4/07-fleet-management.md)

| Layer | Purpose |
|---|---|
| Device registry | Source of truth for identity, version, last-seen, tags |
| Telemetry batching | Bounded request rate at fleet scale vs. per-reading sends |
| Fixed diagnostic command set | Remote visibility without an open remote-execution channel |
| Fleet health alerting | Stale devices (per-unit issue) vs. version fragmentation (rollout issue) |
| Dashboard | Presentation over registry/telemetry data, built last |

### [CI, Emulation &amp; Automated Testing](level-4/08-ci-automated-testing.md)

| Test type | Runs where | Covers |
|---|---|---|
| Pure-logic unit tests | Unix port, or plain `python3` | Parsing, state machines, config validation, protocol framing |
| Hardware-in-the-loop | Real board via `mpremote` in a bench rig | Actual driver/peripheral behavior, real timing |
| Design rule | Keep `machine`/hardware imports isolated to drivers | Maximizes what's testable without a board |

### [Manufacturing &amp; Factory Test](level-4/09-manufacturing-factory-test.md)

| Stage | Purpose |
|---|---|
| Flashing station | Write generic firmware quickly and verifiably to a blank unit |
| Factory self-test | Prove every peripheral works, per-unit, before boxing |
| Serialization | Human-meaningful, sortable per-unit identity tied to batch/date |
| Yield tracking | Fraction of units passing first-try; earliest signal of a hardware problem |
| Failure breakdown | Turns a yield drop into an actionable, component-specific signal |

### [Capstone — Production IoT Product](level-4/10-capstone-production-iot.md)

| Capstone piece | Module |
|---|---|
| Layered `app/` structure, dependency injection | 01 |
| `is_newer`, `rollout_bucket`, confirm-or-rollback | 02 |
| AP-mode provisioning, per-device identity | 03 |
| `boot.py` crash counter, safe mode, watchdog feed placement | 04 |
| Pinned, reproducible firmware build | 05 |
| TLS with pinned cert, no `CERT_NONE` in production | 06 |
| Device registry, staged-rollout monitoring | 07 |
| Unix-port + hardware-in-the-loop test gating | 08 |
| Factory self-test strap, serialization, yield tracking | 09 |

---

Go deeper with the full [Embedded Python Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
