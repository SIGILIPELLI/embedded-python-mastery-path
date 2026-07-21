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
