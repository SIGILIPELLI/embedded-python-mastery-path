---
description: "I2C & SPI Sensors — Real projects read real sensors, and most sensors speak one of two buses: I2C (two wires, shared by many devices, each with an…"
---

# 06 · I2C & SPI Sensors

Real projects read real sensors, and most sensors speak one of two buses:
**I2C** (two wires, shared by many devices, each with an address) or **SPI**
(faster, more wires, one select line per device). MicroPython drives both
through `machine.I2C` and `machine.SPI`. This module scans an I2C bus, reads
a DHT22 temperature/humidity sensor, puts text on an SSD1306 OLED display,
and shows how to install drivers with `mip` — all in Wokwi's *MicroPython on
ESP32* project type.

## I2C in 60 seconds

Two wires: **SCL** (clock) and **SDA** (data), shared by every device on
the bus, plus power and ground. Each device has a 7-bit address (0x3C for a
typical OLED, 0x68 for many RTCs/IMUs). The controller (our ESP32) initiates
everything.

```python
from machine import Pin, I2C

# ESP32 default-ish wiring: SCL=22, SDA=21. Hardware bus 0.
i2c = I2C(0, scl=Pin(22), sda=Pin(21), freq=400000)

print([hex(a) for a in i2c.scan()])    # e.g. ['0x3c'] with an OLED attached
```

`i2c.scan()` is your first debugging tool forever after: it pings all
addresses and lists who answers. Empty list = check wiring; wrong address =
check the datasheet (or the part's jumper). There's also `SoftI2C`, a
bit-banged version that works on *any* two pins — same API, handy when the
hardware buses are taken:

```python
from machine import SoftI2C
i2c = SoftI2C(scl=Pin(22), sda=Pin(21))
```

Raw reads look like this (module 2 of Level 2 turns datasheets into
drivers; here it's just to demystify):

```python
data = i2c.readfrom_mem(0x68, 0x00, 2)   # read 2 bytes from register 0x00
i2c.writeto_mem(0x68, 0x6B, b"\x00")     # write one byte to register 0x6B
```

## Reading a DHT22 (temperature + humidity)

The DHT22 uses its own one-wire-ish protocol, and MicroPython ships a
built-in `dht` driver — a taste of how nice sensor code is when a driver
exists:

```python
from machine import Pin
import dht
import time

sensor = dht.DHT22(Pin(15))

while True:
    sensor.measure()                       # triggers a reading (takes ~250 ms)
    t = sensor.temperature()               # °C, float
    h = sensor.humidity()                  # %RH, float
    print("{:.1f} C  {:.1f} %".format(t, h))
    time.sleep(2)                          # DHT22 max rate: one read / 2 s
```

**Wiring (Wokwi):** add a **DHT22** part — VCC → 3V3, GND → GND, SDA → pin
15. Click the sensor while the sim runs and drag the temperature/humidity
sliders; your printout follows.

!!! warning "Sensors fail — handle it"
    On real hardware a loose wire makes `measure()` raise `OSError`. Wrap
    reads in `try/except OSError` and keep the last good value; a crashed
    logger records nothing. This becomes a core pattern in the capstone.

## SSD1306 OLED over I2C

The 128×64 SSD1306 OLED is *the* hobby display. Its driver isn't built into
the firmware — it's a Python file, `ssd1306.py`, that you add to the board's
filesystem. Two ways to get it:

**With `mip`** (MicroPython's package installer — needs the board online,
module 8):

```python
import mip
mip.install("ssd1306")        # fetches from micropython-lib to /lib
```

**By hand / in Wokwi:** add a new file named `ssd1306.py` to the project
and paste the driver from
[micropython-lib](https://github.com/micropython/micropython-lib/blob/master/micropython/drivers/display/ssd1306/ssd1306.py).
Anything on the filesystem is importable — drivers are just modules.

Then:

```python
from machine import Pin, I2C
import ssd1306

i2c = I2C(0, scl=Pin(22), sda=Pin(21))
oled = ssd1306.SSD1306_I2C(128, 64, i2c)   # address 0x3C by default

oled.fill(0)                        # clear the framebuffer
oled.text("MicroPython!", 0, 0)     # (text, x, y) — 8x8 font
oled.text("Line two", 0, 12)
oled.rect(0, 30, 128, 20, 1)        # x, y, w, h, color
oled.hline(0, 26, 128, 1)
oled.show()                         # nothing appears until show()!
```

**Wiring (Wokwi):** add an **SSD1306** part — SCL → 22, SDA → 21, VCC →
3V3, GND → GND. The two-step model matters: draw calls modify a RAM
framebuffer; `show()` ships it to the display. Draw everything, then show
once.

Putting both together — a live thermometer display:

```python
while True:
    sensor.measure()
    oled.fill(0)
    oled.text("Temp: {:.1f} C".format(sensor.temperature()), 0, 8)
    oled.text("Hum:  {:.1f} %".format(sensor.humidity()), 0, 24)
    oled.show()
    time.sleep(2)
```

## SPI in brief

SPI uses SCK (clock), MOSI (data out), MISO (data in), and one **CS**
(chip-select) pin per device — more pins, much faster, no addresses. You'll
meet it with SD cards, color TFT displays, and fast ADCs:

```python
from machine import Pin, SPI
spi = SPI(2, baudrate=10_000_000, sck=Pin(18), mosi=Pin(23), miso=Pin(19))
cs = Pin(5, Pin.OUT, value=1)     # CS is active-low: 1 = not selected

cs.value(0)                       # select the device
spi.write(b"\x9f")                # send a command
resp = spi.read(3)                # read 3 bytes
cs.value(1)                       # deselect
```

Rule of thumb at this level: prefer I2C parts when shopping — fewer wires,
easier debugging with `scan()`.

## How It Actually Works

I2C, SPI, and the framebuffer model behind `oled.show()` all trade Python's
usual "just call a method" comfort for tight control over what actually
moves across the wire, one bit at a time.

- **Hardware I2C is a peripheral state machine, `SoftI2C` is Python
  bit-banging the pins directly.** `I2C(0, ...)` configures the ESP32's
  dedicated I2C controller, which generates the start condition, clocks out
  each bit on SDA synchronized to SCL, and watches for the slave's ACK bit —
  all in silicon, at up to 400 kHz, without the CPU touching each bit.
  `SoftI2C` instead has the *interpreter* toggle GPIO pins high and low in a
  timed loop to fake the same protocol — it works on any two pins because
  there's no dedicated hardware behind it, but every bit costs VM bytecode
  dispatch time, which is why SoftI2C tops out far below 400 kHz and why the
  cheat sheet's advice to prefer hardware I2C when available is really about
  timing headroom, not convenience.
- **`i2c.scan()` works by attempting the ACK handshake at all 112 possible
  7-bit addresses.** I2C's addressing scheme reserves one bit of each
  address byte for read/write direction; a device "answers" by pulling SDA
  low during the 9th clock pulse (the ACK slot) if it recognizes its own
  address on the bus. `scan()` is literally 112 tiny transactions, each
  checking for that one ACK bit — which is also why a wiring fault (SDA and
  SCL swapped, missing pull-ups) makes *every* address come back empty: the
  ACK slot never gets pulled low by anything.
- **`readfrom_mem`/`writeto_mem` encode the sensor's own register-map
  protocol, not something I2C defines.** I2C itself only moves raw bytes
  between a controller and a device; "register 0x6B means power management"
  is a convention each chip's datasheet defines on top of that, and
  `readfrom_mem(addr, reg, n)` is MicroPython's convenience wrapper for the
  extremely common pattern of "write the register address, then read back
  N bytes" — two I2C transactions stitched into one Python call.
- **The framebuffer is why `show()` is a separate step.** `oled.fill()` and
  `oled.text()` only flip bits in a `bytearray` sitting in the ESP32's own
  RAM — a 1-bit-per-pixel bitmap, 1024 bytes for a 128×64 panel. The SSD1306
  itself has its *own* separate display RAM on the other side of the I2C
  bus; `show()` is the function that streams your local bytearray across
  I2C into the display controller's memory, which is the only thing that
  actually changes what's lit on the glass. Calling `text()` ten times
  before one `show()` is cheap (RAM writes); calling `show()` ten times
  would be ten slow I2C transfers of the whole framebuffer — the two-step
  API exists specifically so you control that cost.
- **SPI has no addressing because chip-select does that job electrically.**
  Where I2C multiplexes many devices over shared wires using addresses sent
  *in-band*, SPI dedicates a separate physical CS wire per device and
  multiplexes *out-of-band* — pulling CS low is what tells a given chip
  "the next clock pulses are for you," which is why SPI needs one more pin
  per device but can run its shift registers much faster: there's no
  per-bit ACK protocol to negotiate, just raw synchronous shifting.

## Cheat sheet

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

## 🔀 Related lessons on other tracks

- [Embedded — 06 · Sensors, I2C & SPI](https://sigilipelli.github.io/embedded-mastery-path/level-1/06-sensors-i2c-spi/)

## Exercise

Build a **min/max weather display** in Wokwi: DHT22 on pin 15, SSD1306 on
I2C(22/21), button on pin 4. Every 2 s, read the sensor and update the OLED
with current temperature and humidity plus the minimum and maximum
temperature seen so far. The button (debounced) resets min/max to the
current reading. Start the program by printing the I2C scan result and
refuse to run (with a clear message on serial) if `0x3c` isn't on the bus.
Wrap sensor reads in `try/except OSError` — on failure, show `sensor err`
on the display's last line and keep the previous values. Vary the Wokwi
sensor sliders and verify min/max track correctly.
