---
title: "Embedded Python Interview Questions and Answers (179 Questions)"
description: "179 Embedded Python interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Embedded Python Mastery Path."
---

# Embedded Python Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Embedded Python Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · MicroPython vs. Python](level-1/01-micropython-vs-python.md)

**Q1.** What is MicroPython?

**Answer:** A lean Python 3 implementation that runs directly on microcontrollers

No OS needed.

**Q2.** What's a key limitation versus CPython?

**Answer:** Small heap (tens to hundreds of KB) and a reduced standard library

Design for memory.

**Q3.** What does `machine` module provide?

**Answer:** Hardware access: Pin, PWM, ADC, I2C, SPI, Timer and more

Port-specific.

**Q4.** Why is MicroPython productive for hardware?

**Answer:** Interactive REPL and quick edit-run cycles without a compile/flash step

Iterate quickly.

### [02 · REPL &amp; Workflow](level-1/02-repl-workflow.md)

**Q5.** What is the REPL on the board?

**Answer:** A live Python prompt running on the chip

Interactive testing.

**Q6.** What does Ctrl-D do at the REPL?

**Answer:** Soft reset (restarts the Python VM)

Re-runs boot.py/main.py.

**Q7.** What do boot.py and main.py do?

**Answer:** boot.py runs first for setup; main.py runs your application

Keep boot.py minimal.

**Q8.** What does `mpremote` do?

**Answer:** Connects to the board to run code, copy files and open the REPL

Common tooling.

### [03 · GPIO Basics](level-1/03-gpio-basics.md)

**Q9.** How do you create an output pin in MicroPython?

**Answer:** machine.Pin(2, machine.Pin.OUT)

Then .value(1).

**Q10.** What does Pin.PULL_UP do?

**Answer:** Enables an internal pull-up so a floating input reads high by default

Buttons to ground.

**Q11.** Why debounce a button?

**Answer:** Contacts bounce, creating multiple edges per press

Time-based filter.

**Q12.** What does pin.value() return for an input?

**Answer:** 0 or 1 for the current level

Read a state.

### [04 · PWM &amp; ADC](level-1/04-pwm-adc.md)

**Q13.** What does PWM duty control?

**Answer:** The fraction of time the pin is high, setting average power (LED brightness)

duty_u16 or duty.

**Q14.** An ESP32 12-bit ADC with 3.3 V reads 1024. What voltage?

```python
print(round(1024/4095*3.3,3))
```

**Answer:** 0.825

raw/4095 x 3.3. Verified by running it.

**Q15.** Why is the ESP32's ADC noisy and non-linear?

**Answer:** It's a SAR ADC with attenuation settings and WiFi noise; average multiple samples and calibrate

read_uv helps.

**Q16.** What does ADC attenuation (ADC.ATTN_11DB) change?

**Answer:** The measurable input voltage range

Larger range at lower accuracy.

### [05 · Timers &amp; Interrupts](level-1/05-timers-interrupts.md)

**Q17.** What does a Timer provide?

**Answer:** Periodic or one-shot callbacks driven by hardware

Timer(period=..., callback).

**Q18.** What must an interrupt callback avoid in MicroPython?

**Answer:** Heap allocation (no list/str creation, no print/exceptions)

ISR constraints.

**Q19.** How do you pass data from an ISR safely?

**Answer:** Set a flag or use micropython.schedule to run code outside the ISR

Defer work.

**Q20.** What's `micropython.alloc_emergency_exception_buf` for?

**Answer:** Allowing exception reports from within ISRs

Handy for debugging.

### [06 · I2C &amp; SPI Sensors](level-1/06-i2c-spi-sensors.md)

**Q21.** What does this print (little-endian 16-bit sensor reading)?

```python
import struct
raw=bytes([0x34,0x12])
print(hex(struct.unpack('<H', raw)[0]), struct.unpack('>H', raw)[0])
```

**Answer:** 0x1234 13330

Byte order matters when reading registers. Verified by running it.

**Q22.** How does I2C address a device?

**Answer:** A 7-bit address followed by a read/write bit

i2c.scan() finds devices.

**Q23.** What's the difference between I2C and SPI?

**Answer:** I2C: 2 wires and addresses; SPI: faster with a select line per device

Pick by sensor.

**Q24.** What do you do first with a new I2C sensor?

**Answer:** Scan for its address and read the WHO_AM_I/ID register

Verify the wiring.

### [07 · Files &amp; Persistence](level-1/07-files-persistence.md)

**Q25.** Where does persistent data live?

**Answer:** The flash filesystem (survives resets)

Flash has limited writes.

**Q26.** Why avoid writing flash every second?

**Answer:** Flash wears out after limited erase cycles

Batch writes.

**Q27.** What does this print?

```python
import json
d={'t':21.5,'ok':True}
s=json.dumps(d)
print(s, json.loads(s)['t'])
```

**Answer:** {"t": 21.5, "ok": true} 21.5

JSON for config and logs. Verified by running it.

**Q28.** How do you bound a log file?

**Answer:** Rotate or truncate when it exceeds a size

Flash is small.

### [08 · WiFi &amp; HTTP](level-1/08-wifi-http.md)

**Q29.** What does `network.WLAN(STA_IF)` do?

**Answer:** Creates a station interface to join a WiFi network

wlan.connect(ssid, pw).

**Q30.** Why handle failed connections?

**Answer:** Networks drop; retry with timeouts instead of hanging

Robustness.

**Q31.** What should you never commit to source control?

**Answer:** WiFi credentials and API keys

Use secrets.json.

**Q32.** What's urequests/requests for?

**Answer:** HTTP requests from the device

Close responses to free memory.

### [09 · uasyncio Basics](level-1/09-uasyncio-basics.md)

**Q33.** What does uasyncio provide?

**Answer:** Cooperative multitasking with async/await on a single thread

Lightweight.

**Q34.** What blocks uasyncio?

**Answer:** time.sleep() or long synchronous work; use await asyncio.sleep()

Yield to the loop.

**Q35.** What does this print (CPython asyncio model of the same idea)?

```python
import asyncio
async def a(n):
    await asyncio.sleep(0.01*n); print('t',n)
async def main():
    await asyncio.gather(a(3),a(1),a(2))
asyncio.run(main())
```

**Answer:** t 1
t 2
t 3

Tasks interleave at awaits. Verified by running it.

**Q36.** How do you run a task concurrently?

**Answer:** asyncio.create_task(coro())

Then await the loop.

### [10 · Capstone — WiFi Sensor Logger](level-1/10-capstone-wifi-sensor-logger.md)

**Q37.** What does the WiFi sensor logger combine?

**Answer:** Sensor reads, bounded flash log, display and a tiny web server

Level 1 capstone.

**Q38.** How do you keep the log bounded?

**Answer:** Rotate or cap entries and size

Flash and RAM limits.

**Q39.** Why serve JSON as well as HTML?

**Answer:** Machines can consume the data and the HTML page can use it

API-first.

**Q40.** How do you avoid blocking the server?

**Answer:** Use uasyncio so sensing and serving run concurrently

Cooperative.

## Level 2 · Intermediate

### [MQTT &amp; IoT Messaging](level-2/01-mqtt-iot-messaging.md)

**Q41.** What is MQTT?

**Answer:** A lightweight publish/subscribe protocol via a broker

Ideal for IoT.

**Q42.** What's a topic?

**Answer:** A hierarchical string like home/kitchen/temp that routes messages

Wildcards + and #.

**Q43.** Why call check_msg() or wait_msg() regularly?

**Answer:** The client must process incoming packets and keepalives

Else the broker disconnects.

**Q44.** What does QoS 1 guarantee?

**Answer:** At-least-once delivery (duplicates possible)

Handle duplicates.

### [BLE with aioble](level-2/02-ble-aioble.md)

**Q45.** What does aioble provide?

**Answer:** Async BLE APIs for MicroPython (advertising, GATT, scanning)

Built on bluetooth.

**Q46.** Why choose BLE for a wearable?

**Answer:** Low power and phone compatibility

Coin cell friendly.

**Q47.** What's a GATT service?

**Answer:** A group of characteristics exposing device data

Standard UUIDs exist.

**Q48.** What does notify do?

**Answer:** Pushes characteristic updates to subscribed clients

Avoid polling.

### [Deep Sleep &amp; Battery Power](level-2/03-deep-sleep-battery.md)

**Q49.** What does machine.deepsleep(ms) do?

**Answer:** Powers down most of the chip and restarts (like a reset) after the time or a wake source

Microamps.

**Q50.** What survives deep sleep on ESP32?

**Answer:** RTC memory and NVS/flash (RAM contents are lost)

Store state wisely.

**Q51.** A device wakes for 2 s at 80 mA then sleeps 58 s at 0.01 mA. What's the average current in mA?

```python
print(round((2*80+58*0.01)/60,3))
```

**Answer:** 2.676

Duty-cycled average. Verified by running it.

**Q52.** Why shut down WiFi before sleeping?

**Answer:** Radios consume power and delay sleep; disconnect cleanly

wlan.active(False).

### [Writing Custom Sensor Drivers](level-2/04-custom-sensor-drivers.md)

**Q53.** What's the first step writing a sensor driver from a datasheet?

**Answer:** Identify the I2C/SPI address and read the ID register to verify communication

Then configure registers.

**Q54.** Why wrap register access in a class?

**Answer:** Encapsulate addresses and conversions and reuse across projects

Clean API.

**Q55.** How do you convert raw sensor values?

**Answer:** Apply the datasheet's scaling formula and handle two's complement

Check units.

**Q56.** How do you test a driver without hardware?

**Answer:** Mock the I2C bus with canned responses on the host

Unit tests.

### [mpremote &amp; mip — Tooling Deep Dive](level-2/05-mpremote-mip-tooling.md)

**Q57.** What does `mip` do?

**Answer:** Installs MicroPython packages onto the board

mip.install('...').

**Q58.** What does `mpremote mount .` do?

**Answer:** Mounts a local folder on the board so you can run code without copying

Fast iteration.

**Q59.** How do you copy files with mpremote?

**Answer:** mpremote cp file.py :

:path is the device.

**Q60.** Why script mpremote commands?

**Answer:** Repeatable deploys and CI

Automation.

### [NTP, RTC &amp; Timekeeping](level-2/06-ntp-rtc-timekeeping.md)

**Q61.** Why must the device sync its clock?

**Answer:** The board boots thinking it's 2000 and timestamps would be meaningless

NTP over WiFi.

**Q62.** What does ntptime.settime() set?

**Answer:** The RTC to UTC from an NTP server

Apply your offset.

**Q63.** Why store timestamps in UTC?

**Answer:** Avoid ambiguity and DST issues

Convert on display.

**Q64.** What does the RTC do through deep sleep?

**Answer:** Keeps time (but may drift without sync)

Resync periodically.

### [NeoPixels, Displays &amp; Device UI](level-2/07-neopixels-displays-ui.md)

**Q65.** What do NeoPixels (WS2812) need?

**Answer:** A precisely timed single data line, handled by the neopixel module

Add power and a level shifter.

**Q66.** Why add a capacitor and series resistor to LED strips?

**Answer:** Protect against inrush and signal spikes

Hardware hygiene.

**Q67.** What do OLED displays like SSD1306 use?

**Answer:** I2C or SPI with a framebuffer

framebuf.

**Q68.** How do you update displays without flicker?

**Answer:** Draw to the framebuffer then show() once

Double buffering idea.

### [ESP-NOW &amp; Peer-to-Peer WiFi](level-2/08-espnow-peer-to-peer.md)

**Q69.** What is ESP-NOW?

**Answer:** Connectionless peer-to-peer messaging between ESP devices without a router

Low latency.

**Q70.** What must you configure for ESP-NOW?

**Answer:** WiFi in STA mode and peer MAC addresses

add_peer.

**Q71.** What's a limitation?

**Answer:** Small payloads (~250 bytes) and no built-in delivery guarantee

Design acks.

**Q72.** When prefer ESP-NOW over MQTT?

**Answer:** Nearby devices needing low power and no infrastructure

Local control.

### [Error Handling &amp; On-Device Logging](level-2/09-error-handling-logging.md)

**Q73.** What should a headless device do on an unhandled exception?

**Answer:** Log it and recover (reset or retry) rather than stop

Resilience.

**Q74.** Why persist logs to flash?

**Answer:** To diagnose crashes after a reset

Bounded files.

**Q75.** What does try/except around the main loop enable?

**Answer:** Catch errors, record a trace and restart safely

machine.reset().

**Q76.** How do you distinguish reset causes?

**Answer:** machine.reset_cause() (power-on, watchdog, deepsleep)

Diagnose.

### [Project — MQTT Weather Station](level-2/10-project-mqtt-weather-station.md)

**Q77.** What does the MQTT weather station combine?

**Answer:** Sensor reading, timestamp, MQTT publish, display and deep sleep

Level 2 capstone.

**Q78.** What's the power-friendly loop?

**Answer:** Wake, connect, read, publish, disconnect, deep sleep

Batch work.

**Q79.** How do you handle failed publishes?

**Answer:** Retry with backoff and cache readings

Store and forward.

**Q80.** What validates battery life?

**Answer:** Measure real current over a full cycle

Power profile.

## Level 3 · Advanced

### [Memory Management &amp; the Garbage Collector](level-3/01-memory-gc.md)

**Q81.** Why does memory fragmentation matter on MicroPython?

**Answer:** Small heaps can fail allocations despite free total memory

gc.mem_free().

**Q82.** What does gc.collect() do?

**Answer:** Frees unreachable objects immediately

Call before big allocations.

**Q83.** How do you reduce allocations?

**Answer:** Preallocate buffers (bytearray) and reuse them

Avoid per-loop objects.

**Q84.** What does micropython.mem_info() show?

**Answer:** Heap usage and fragmentation summary

Debug memory.

### [Performance Profiling &amp; Optimization](level-3/02-performance-profiling.md)

**Q85.** `ticks_diff` handles wraparound. What's the result in this simulation?

```python
M=1<<30
def ticks_diff(a,b):
    d=(a-b)%M
    return d-M if d>=M//2 else d
print(ticks_diff(5, M-5), ticks_diff(M-5,5))
```

**Answer:** 10 -10

Never subtract tick values directly. Verified by running it.

**Q86.** What's the right way to time a code block?

**Answer:** time.ticks_us() before and after with ticks_diff()

No cProfile.

**Q87.** Why measure before optimizing?

**Answer:** Intuition about hotspots is often wrong

Evidence.

**Q88.** What does `@micropython.native` do?

**Answer:** Compiles the function to native machine code for speed

Next lesson.

### [Native &amp; Viper Code Emitters](level-3/03-native-viper-emitters.md)

**Q89.** What does the @micropython.viper decorator provide?

**Answer:** Native code with typed integer/pointer operations for maximum speed

Use for hot loops.

**Q90.** What's a viper trade-off?

**Answer:** Reduced Python semantics (ints wrap, no overflow checks) and portability limits

Be careful.

**Q91.** When use native/viper?

**Answer:** After profiling finds a hot loop that needs speed

Measure.

**Q92.** What do viper type hints like int do?

**Answer:** Let the emitter use raw machine integers instead of Python objects

Speed.

### [Writing C Modules for MicroPython](level-3/04-c-modules.md)

**Q93.** Why write a C module for MicroPython?

**Answer:** Speed or hardware access beyond Python, exposed as a module

USER_C_MODULES.

**Q94.** What do you rebuild to include a C module?

**Answer:** The MicroPython firmware

Custom firmware.

**Q95.** What does MP_REGISTER_MODULE do?

**Answer:** Registers your C module so Python can import it

Boilerplate.

**Q96.** What's a cost of C modules?

**Answer:** Maintaining custom firmware builds

Use when needed.

### [RP2040 PIO — Programmable I/O](level-3/05-rp2040-pio.md)

**Q97.** What is RP2040 PIO?

**Answer:** Programmable I/O state machines that implement custom protocols in hardware

Precise timing.

**Q98.** Why use PIO?

**Answer:** Deterministic bit-level protocols (WS2812, custom serial) without CPU load

rp2.asm_pio.

**Q99.** How many instructions does a PIO program have?

**Answer:** 32 instruction slots per state-machine block

Tiny but powerful.

**Q100.** How does the CPU feed a PIO state machine?

**Answer:** Through FIFOs (put/get) or DMA

Async transfer.

### [Advanced uasyncio Patterns](level-3/06-advanced-uasyncio.md)

**Q101.** What does asyncio.Event do?

**Answer:** Lets tasks wait until another task signals

Coordination.

**Q102.** What's a safe way to stop a task?

**Answer:** Cancel it and handle CancelledError for cleanup

Resource release.

**Q103.** Why use asyncio.Lock?

**Answer:** Protect shared resources across awaits

Cooperative but still interleaves.

**Q104.** What should async code avoid?

**Answer:** Long blocking calls that starve other tasks

Yield often.

### [The Inline Assembler](level-3/07-inline-assembler.md)

**Q105.** What does inline assembler (@micropython.asm_thumb) offer?

**Answer:** Hand-written Thumb instructions for extreme speed or precise timing

Cortex-M only.

**Q106.** When is it justified?

**Answer:** Rare hot paths where native/viper isn't enough

Complexity cost.

**Q107.** What's a risk?

**Answer:** Hard to maintain and tied to one architecture

Document well.

**Q108.** What do arguments get passed in?

**Answer:** Registers r0-r3

Calling convention.

### [Frozen Modules &amp; Custom Manifests](level-3/08-frozen-modules-manifests.md)

**Q109.** What are frozen modules?

**Answer:** Python code compiled into the firmware, stored in flash instead of RAM

Saves RAM.

**Q110.** Why freeze large libraries?

**Answer:** Avoid compiling at import time and using heap

manifest.py.

**Q111.** How do you add frozen modules?

**Answer:** List them in a manifest and rebuild firmware

freeze().

**Q112.** Trade-off of frozen modules?

**Answer:** Updating requires a firmware rebuild/flash

Balance.

### [Threads &amp; Dual-Core MicroPython](level-3/09-threads-dual-core.md)

**Q113.** What does the second RP2040 core allow?

**Answer:** Running code in parallel on another core with _thread

Careful with shared state.

**Q114.** What must you do when sharing data between cores?

**Answer:** Use locks (_thread.allocate_lock) to avoid races

Synchronize.

**Q115.** Why is the GIL absent but caution needed?

**Answer:** MicroPython threads can run truly in parallel on RP2040, so races are real

Locks.

**Q116.** What does ESP32 threading differ in?

**Answer:** Uses FreeRTOS tasks via _thread with limited stack

Small stacks.

### [Project — High-Speed Data Acquisition](level-3/10-project-high-speed-daq.md)

**Q117.** What does the high-speed DAQ project combine?

**Answer:** Timer-driven sampling, preallocated buffers, DMA/PIO and efficient streaming

Level 3 capstone.

**Q118.** Why preallocate buffers?

**Answer:** Avoid heap allocation and GC pauses during capture

Deterministic.

**Q119.** How do you verify sample timing?

**Answer:** Measure intervals with ticks_us and check jitter

Quantify.

**Q120.** What do you do with data fast?

**Answer:** Stream out in chunks or store in ring buffers

Avoid allocation.

## Level 4 · Master

### [Production Firmware Architecture](level-4/01-production-architecture.md)

**Q121.** What does production architecture add to hobby code?

**Answer:** Structure, error handling, config, logging and update paths

Maintainable.

**Q122.** Why separate hardware, logic and app layers?

**Answer:** Testability and portability

Mock hardware.

**Q123.** How should configuration be managed?

**Answer:** In a config file/NVS separate from code

Per-device values.

**Q124.** What's a state machine for?

**Answer:** Explicit event-driven behavior that is easy to test

Avoid spaghetti.

### [OTA Updates for MicroPython Fleets](level-4/02-ota-updates-fleets.md)

**Q125.** What's a safe OTA approach on MicroPython?

**Answer:** Download to a temp file, verify hash/signature, then swap and keep a fallback

Atomic update.

**Q126.** Why stage fleet rollouts?

**Answer:** Limit blast radius if an update misbehaves

Canary.

**Q127.** What should an update include?

**Answer:** Version, integrity check and rollback info

Metadata.

**Q128.** What if the new app crashes at boot?

**Answer:** Detect via counter and roll back to the previous version

Safe mode.

### [Device Provisioning &amp; Secrets Management](level-4/03-provisioning-secrets.md)

**Q129.** How should secrets be provisioned?

**Answer:** Per-device at manufacture, never shared across the fleet or hardcoded

Unique credentials.

**Q130.** Where can secrets be stored more safely?

**Answer:** Encrypted NVS/secure element rather than plain files

Reduce exposure.

**Q131.** Why rotate credentials?

**Answer:** Limit damage if one leaks

Lifecycle.

**Q132.** What do TLS certificates verify?

**Answer:** The server's identity, preventing man-in-the-middle

Validate, don't skip.

### [Watchdogs, Brownout &amp; Robustness Patterns](level-4/04-watchdogs-robustness.md)

**Q133.** What does a watchdog do?

**Answer:** Resets the device if the main loop stops feeding it

WDT(timeout=...).

**Q134.** Why feed the WDT only when healthy?

**Answer:** Blind feeding hides hangs

Check conditions.

**Q135.** What should you do after a WDT reset?

**Answer:** Record the cause and enter a safe state

Diagnose.

**Q136.** What does robust firmware assume?

**Answer:** Networks fail, sensors glitch and power drops

Design for faults.

### [Custom Firmware Builds &amp; Board Definitions](level-4/05-custom-firmware-builds.md)

**Q137.** Why build custom firmware?

**Answer:** Include frozen modules, C modules or change board config

Reproducible builds.

**Q138.** What does the build need?

**Answer:** The MicroPython source, toolchain (e.g. ESP-IDF) and a board definition

CI helps.

**Q139.** How do you ensure reproducibility?

**Answer:** Pin versions and build in CI

Same inputs, same image.

**Q140.** What do you verify after a build?

**Answer:** Boot, REPL, modules and a smoke test on hardware

Test.

### [Security Hardening — TLS &amp; Secure Storage](level-4/06-security-hardening.md)

**Q141.** Which hardening measure applies to MicroPython devices?

**Answer:** Disable the REPL/USB debug in production and protect credentials

Reduce attack surface.

**Q142.** Why validate all network input?

**Answer:** Untrusted data can crash or exploit the device

Bounds checks.

**Q143.** What does flash encryption protect?

**Answer:** Firmware and data from readout

ESP32 feature.

**Q144.** Why use secure boot?

**Answer:** Only signed firmware runs

Chain of trust.

### [Fleet Management &amp; Remote Monitoring](level-4/07-fleet-management.md)

**Q145.** What does fleet management provide?

**Answer:** Monitoring, configuration and updates across many devices

Scale.

**Q146.** What should each device report?

**Answer:** Version, health, battery and errors

Telemetry.

**Q147.** Why a device identity?

**Answer:** Authenticate and address each device individually

Unique ID/cert.

**Q148.** How to roll out config changes?

**Answer:** Staged with rollback and monitoring

Safe changes.

### [CI, Emulation &amp; Automated Testing](level-4/08-ci-automated-testing.md)

**Q149.** What does CI for MicroPython include?

**Answer:** Host unit tests with mocks, linting and firmware builds

pytest + mpy-cross.

**Q150.** How can you test hardware-independent code?

**Answer:** Run it under CPython/unix port with mocked machine modules

Fast feedback.

**Q151.** What does the unix port enable?

**Answer:** Running MicroPython on a PC for tests

CI friendly.

**Q152.** What do HIL tests add?

**Answer:** Verification on real boards with automated rigs

End to end.

### [Manufacturing &amp; Factory Test](level-4/09-manufacturing-factory-test.md)

**Q153.** What's factory test?

**Answer:** Automated per-unit verification and provisioning on the production line

Quality and identity.

**Q154.** What does a test jig use?

**Answer:** Pogo pins and fixtures to access test points

Repeatable.

**Q155.** What belongs in factory firmware?

**Answer:** Self-tests, serial number/credential provisioning and logging

Then flash the final image.

**Q156.** Why log factory results?

**Answer:** Traceability for support and recalls

Per-serial records.

### [Capstone — Production IoT Product](level-4/10-capstone-production-iot.md)

**Q157.** What does the production IoT capstone integrate?

**Answer:** Secure provisioning, OTA, watchdogs, telemetry and CI

Full product.

**Q158.** How do you prove readiness?

**Answer:** CI, HIL, fault injection, field trials and staged rollouts

Evidence.

**Q159.** What documentation matters?

**Answer:** Architecture, runbooks and release/support procedures

Lifecycle.

**Q160.** What sustains a fleet over years?

**Answer:** Update strategy, security patches and monitoring

Plan support.

## Mixed questions across all levels

**Q161.** What's MicroPython's biggest practical difference from desktop Python?

**Answer:** A tiny heap and reduced library, so you must manage memory and hardware directly

Design for constraints.

**Q162.** Which REPL shortcut soft-resets the board and re-runs main.py?

**Answer:** Ctrl-D

Ctrl-C interrupts the running program.

**Q163.** A button wired to ground should read a clean value. How do you configure the pin?

**Answer:** machine.Pin(n, Pin.IN, Pin.PULL_UP) and debounce it

Avoid floating inputs.

**Q164.** A 12-bit ADC at 3.3 V reads 3000. What voltage is that?

```python
print(round(3000/4095*3.3,2))
```

**Answer:** 2.42

raw/4095 x Vref. Verified by running it.

**Q165.** An interrupt callback allocates a list and raises MemoryError. Why?

**Answer:** ISRs can't allocate heap memory in MicroPython; set a flag or use micropython.schedule

Preallocate buffers.

**Q166.** A sensor sends bytes 0x34 0x12 for a little-endian 16-bit value. What is it?

```python
import struct
print(struct.unpack('<H', bytes([0x34,0x12]))[0])
```

**Answer:** 4660

0x1234 = 4660. Verified by running it.

**Q167.** Why batch flash writes in a logger?

**Answer:** Flash wears out after limited erase cycles

Rotate and bound logs.

**Q168.** A device must do networking, sensing and display without threads. What do you use?

**Answer:** uasyncio tasks with await asyncio.sleep() instead of time.sleep()

Cooperative multitasking.

**Q169.** The MQTT client stops receiving after a while. What did you likely forget?

**Answer:** Calling check_msg()/wait_msg() and keepalive pings regularly

Else the broker disconnects.

**Q170.** Wake 2 s at 80 mA each minute, otherwise deep sleep at 0.02 mA. What's the average current (mA)?

```python
print(round((2*80+58*0.02)/60,3))
```

**Answer:** 2.686

Duty-cycled average. Verified by running it.

**Q171.** After deep sleep, your variables are gone. Where can state survive?

**Answer:** RTC memory or flash/NVS (RAM is cleared)

Deep sleep restarts the program.

**Q172.** The board thinks it's the year 2000. What do you do?

**Answer:** Sync the RTC with ntptime over WiFi and store UTC

Timestamps need a valid clock.

**Q173.** Using wrap-safe tick arithmetic, what's ticks_diff(5, MASK-5) with a 30-bit counter?

```python
M=(1<<30)-1
d=(5-(M-5))&M
print(d-(M+1) if d>=(M+1)//2 else d)
```

**Answer:** 11

Never subtract tick values directly. Verified by running it.

**Q174.** A hot loop is too slow. What's the correct order?

**Answer:** Measure with ticks_us, then try @native/@viper, then consider C modules or PIO

Profile before optimizing.

**Q175.** Heap allocation fails though gc.mem_free() looks large. What's the cause?

**Answer:** Fragmentation: no contiguous block big enough; preallocate and call gc.collect()

Large buffers need contiguous space.

**Q176.** A field device hangs and never recovers. What's the standard protection?

**Answer:** A hardware watchdog fed only when the system is healthy

Blind feeding hides hangs.

**Q177.** An OTA update loses power halfway. What design avoids a bricked device?

**Answer:** Download to a temp file, verify the hash/signature, swap atomically and keep a fallback

Safe updates.

**Q178.** How should WiFi credentials and API keys be handled?

**Answer:** Provisioned per device and stored outside source control (e.g. encrypted NVS)

Never hardcode.

**Q179.** How do you unit-test hardware-independent MicroPython logic in CI?

**Answer:** Run it under CPython or the unix port with mocked machine modules

Fast feedback.

## Practical coding questions

### A wrap-safe cooperative scheduler

Implement ticks_add, ticks_diff (correct across counter wraparound like time.ticks_diff) and Scheduler.every/poll so jobs run when their period elapses without blocking and without drifting. The simulation starts near the counter wrap.

Starter code:

```
MASK = (1 << 30) - 1          # tick counter wraps like time.ticks_ms() on a MicroPython port

def ticks_add(t, d):
    # wrap-safe addition on a MASK-bit counter
    return 0

def ticks_diff(a, b):
    # signed difference a - b that is correct across wraparound (like time.ticks_diff)
    return 0

class Scheduler:
    """Cooperative scheduler: run jobs whose period has elapsed (no sleep() blocking)."""
    def __init__(self):
        self.jobs = []                    # [name, period, next_due]

    def every(self, name, period, now):
        # register a job first due at now + period
        pass

    def poll(self, now):
        # return the names of jobs that are due (ticks_diff(now, due) >= 0) and reschedule them
        # by adding their period to the previous due time (no drift)
        return []

s = Scheduler()
start = MASK - 250                        # counter about to wrap
s.every("blink", 100, start)
s.every("sensor", 250, start)
log = []
for step in range(0, 701, 50):
    now = ticks_add(start, step)
    ran = s.poll(now)
    if ran:
        log.append(f"{step}:{'+'.join(ran)}")
print(" ".join(log))
print(ticks_diff(5, MASK - 5), ticks_diff(MASK - 5, 5))
```

Expected output:

```text
100:blink 200:blink 250:sensor 300:blink 400:blink 500:blink+sensor 600:blink 700:blink
11 -11
```

??? success "Show a solution"

    ```
    MASK = (1 << 30) - 1          # tick counter wraps like time.ticks_ms() on a MicroPython port

    def ticks_add(t, d):
        return (t + d) & MASK

    def ticks_diff(a, b):
        d = (a - b) & MASK
        return d - (MASK + 1) if d >= (MASK + 1) // 2 else d

    class Scheduler:
        """Cooperative scheduler: run jobs whose period has elapsed (no sleep() blocking)."""
        def __init__(self):
            self.jobs = []                    # [name, period, next_due]

        def every(self, name, period, now):
            self.jobs.append([name, period, ticks_add(now, period)])

        def poll(self, now):
            ran = []
            for job in self.jobs:
                if ticks_diff(now, job[2]) >= 0:
                    ran.append(job[0])
                    job[2] = ticks_add(job[2], job[1])
            return ran

    s = Scheduler()
    start = MASK - 250                        # counter about to wrap
    s.every("blink", 100, start)
    s.every("sensor", 250, start)
    log = []
    for step in range(0, 701, 50):
        now = ticks_add(start, step)
        ran = s.poll(now)
        if ran:
            log.append(f"{step}:{'+'.join(ran)}")
    print(" ".join(log))
    print(ticks_diff(5, MASK - 5), ticks_diff(MASK - 5, 5))
    ```

    Subtracting raw tick values breaks at wraparound, so differences are computed modulo the counter size and interpreted as signed. Rescheduling from the previous due time (not 'now') avoids drift. Verified by running the solution (CPython model of the MicroPython API).

### Bounded log and reconnect backoff

Implement BoundedLog.append (drop the oldest lines until the byte budget fits, keep at least one) and Backoff.failure (min(cap, base*2**fails)), the two building blocks of a resilient, flash-friendly logger.

Starter code:

```
import json

class BoundedLog:
    """Keep the newest entries within a byte budget, like a size-limited log file on flash."""
    def __init__(self, max_bytes):
        self.max_bytes = max_bytes
        self.lines = []

    def size(self):
        return sum(len(l) + 1 for l in self.lines)

    def append(self, record):
        # add json.dumps(record, separators=(",", ":")) and drop the OLDEST lines until size() <= max_bytes
        # (always keep at least one line)
        pass

class Backoff:
    """Exponential backoff with a cap, reset on success (reconnect logic)."""
    def __init__(self, base=1, cap=30):
        self.base, self.cap, self.fails = base, cap, 0

    def failure(self):
        # return the next delay: min(cap, base * 2**fails), then count the failure
        return 0

    def success(self):
        self.fails = 0

log = BoundedLog(60)
for i in range(1, 8):
    log.append({"i": i, "t": 20 + i})
print(len(log.lines), log.size(), log.lines[0])
b = Backoff()
print([b.failure() for _ in range(7)])
b.success()
print(b.failure())
```

Expected output:

```text
4 60 {"i":4,"t":24}
[1, 2, 4, 8, 16, 30, 30]
1
```

??? success "Show a solution"

    ```
    import json

    class BoundedLog:
        """Keep the newest entries within a byte budget, like a size-limited log file on flash."""
        def __init__(self, max_bytes):
            self.max_bytes = max_bytes
            self.lines = []

        def size(self):
            return sum(len(l) + 1 for l in self.lines)

        def append(self, record):
            self.lines.append(json.dumps(record, separators=(",", ":")))
            while self.size() > self.max_bytes and len(self.lines) > 1:
                self.lines.pop(0)

    class Backoff:
        """Exponential backoff with a cap, reset on success (reconnect logic)."""
        def __init__(self, base=1, cap=30):
            self.base, self.cap, self.fails = base, cap, 0

        def failure(self):
            delay = min(self.cap, self.base * (2 ** self.fails))
            self.fails += 1
            return delay

        def success(self):
            self.fails = 0

    log = BoundedLog(60)
    for i in range(1, 8):
        log.append({"i": i, "t": 20 + i})
    print(len(log.lines), log.size(), log.lines[0])
    b = Backoff()
    print([b.failure() for _ in range(7)])
    b.success()
    print(b.failure())
    ```

    Bounding the log protects limited flash and RAM, and capped exponential backoff keeps reconnect attempts from hammering a dead network or draining the battery. Verified by running the solution.

---

More free practice: the [Embedded Python Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
