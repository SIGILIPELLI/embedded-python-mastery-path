# Writing Custom Sensor Drivers

Every sensor you've used so far (`dht`, `ssd1306`) came with a driver
someone already wrote. Sooner or later you'll buy a part with only a PDF
datasheet and no MicroPython support — this module is about turning that
datasheet into a clean driver class: register maps, `struct` unpacking,
calibration math, and packaging it so `mip` could install it. The register
math and class logic run as pure Python via `python3`; the `machine.I2C`
calls are reviewed against MicroPython docs since Wokwi doesn't expose an
arbitrary configurable I2C sensor to test register reads against.

## Reading a datasheet like a driver author

A typical I2C sensor datasheet gives you:

- **7-bit I2C address** (sometimes selectable via a pin, e.g. 0x68 or 0x69)
- **Register map** — a table of addresses (0x00, 0x01, ...) and what each
  byte means
- **Data format** — signed/unsigned, byte order (endianness), scaling
  factor
- **Init sequence** — registers that must be written before readings are
  valid (power-on config, sample rate, range)

Take a made-up-but-typical accelerometer: address `0x1D`, register `0x20`
is "power control" (write `0x07` to enable all axes), registers `0x28`-`0x2D`
are X/Y/Z as three little-endian signed 16-bit values, scaled by a
sensitivity factor from the datasheet.

## Building the driver class

```python
from machine import I2C
import struct

_ADDR = 0x1D
_REG_POWER_CTL = 0x20
_REG_DATA = 0x28
_SENSITIVITY = 0.004   # g per LSB, from the (fictional) datasheet

class Accelerometer:
    def __init__(self, i2c, addr=_ADDR):
        self.i2c = i2c
        self.addr = addr
        self._init_device()

    def _init_device(self):
        self.i2c.writeto_mem(self.addr, _REG_POWER_CTL, b"\x07")

    def read_raw(self):
        data = self.i2c.readfrom_mem(self.addr, _REG_DATA, 6)
        # 3 signed 16-bit little-endian values
        x, y, z = struct.unpack("<hhh", data)
        return x, y, z

    def read_g(self):
        x, y, z = self.read_raw()
        return (x * _SENSITIVITY, y * _SENSITIVITY, z * _SENSITIVITY)
```

`struct.unpack("<hhh", data)` is the workhorse: `<` means little-endian,
`h` means signed 16-bit ("half word"), three of them for X/Y/Z. Get the
endianness or signedness wrong and you get numbers that are *almost*
right — off by a scale factor, or occasionally wildly negative — which is
a much nastier bug than an outright crash.

```python
i2c = I2C(0, scl=machine.Pin(22), sda=machine.Pin(21))
accel = Accelerometer(i2c)
print(accel.read_g())      # e.g. (0.02, -0.01, 0.98) — resting, Z ~ gravity
```

## `struct` format cheat reference

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

## Calibration

Raw sensor output usually needs an offset and/or scale correction found by
measuring known reference points. A simple two-point linear calibration:

```python
def calibrate_linear(raw_low, raw_high, true_low, true_high):
    """Return (scale, offset) such that true = raw * scale + offset."""
    scale = (true_high - true_low) / (raw_high - raw_low)
    offset = true_low - raw_low * scale
    return scale, offset

# e.g. a temperature sensor read 100 raw at 0 C and 500 raw at 40 C
scale, offset = calibrate_linear(100, 500, 0, 40)
def raw_to_celsius(raw):
    return raw * scale + offset

print(raw_to_celsius(300))   # ~20.0
```

Store `scale`/`offset` (or more calibration points) in a config file on
flash (Level 1, module 7) so each physical unit can be calibrated once at
build time, not hardcoded in the driver.

## Packaging for `mip`

A driver becomes installable once it's a plain `.py` module hosted
somewhere `mip` can fetch — a GitHub repo, or micropython-lib itself.
Minimum shape:

```
my_accel_driver/
  accel.py          # the class above, no board-specific code inside
```

Keep board wiring (which `I2C` instance, which pins) *out* of the driver —
the caller constructs `I2C(...)` and passes it in, exactly like the
`Accelerometer.__init__(self, i2c, addr=...)` signature above. A driver
that calls `machine.I2C(0, scl=Pin(22), ...)` internally only works on
boards wired exactly that way; passing `i2c` in makes it reusable.
Install with:

```python
import mip
mip.install("github:yourname/my_accel_driver/accel.py")
```

## Driver-writing traps

!!! warning "Endianness bugs look like calibration bugs"
    A byte-order mistake often produces values that are merely wrong, not
    obviously broken (e.g. `0x0100` read as 1 instead of 256). If numbers
    are "close but scaled weird," check `struct` format before suspecting
    your math.

!!! warning "I2C exceptions during init should not be silent"
    If `_init_device()` fails (device not wired, wrong address), let the
    `OSError` propagate from `__init__` rather than swallowing it — a
    driver that "succeeds" to construct but never actually configured the
    device produces readings that are garbage, not obviously erroring.

!!! warning "Don't do slow work in `__init__`"
    Some devices need a settling delay (`time.sleep_ms(50)`) after power-up
    before the first read is valid. Do it once in `__init__`, not on every
    `read_*()` call — the latter silently triples your loop's cycle time.

## Cheat sheet

| Function / idiom | Purpose |
|---|---|
| `i2c.readfrom_mem(addr, reg, n)` | Read raw bytes from a register |
| `i2c.writeto_mem(addr, reg, bytes)` | Write bytes to configure a register |
| `struct.unpack(fmt, data)` | Decode raw bytes into numbers |
| `"<hhh"` | Little-endian, three signed 16-bit values |
| pass `i2c` into `__init__`, don't create it inside | Keeps the driver board-agnostic |
| two-point linear calibration | `scale, offset` from two known references |
| propagate `OSError` from init | Fail loud, not silently-wrong |

## Exercise

Write (and test with `python3`, feeding it fabricated byte strings — no
hardware needed for this part) a `Thermistor` driver class with
`__init__(self, i2c, addr=0x40)`, a `read_raw()` that unpacks one unsigned
16-bit big-endian value from register `0x00` via
`i2c.readfrom_mem(addr, 0x00, 2)`, and a `read_celsius()` that applies a
`calibrate_linear`-derived `scale`/`offset` computed from two reference
points you choose (e.g. raw 0 at -10 C, raw 65535 at 85 C — the sensor's
full range). Prove the unpacking works by constructing raw bytes with
`struct.pack(">H", value)` for a few test values and checking
`read_celsius()` output against hand-calculated expected temperatures.
Add a docstring noting which register/format assumptions would need to be
confirmed against a real datasheet before running on hardware.
