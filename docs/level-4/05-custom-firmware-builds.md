# Custom Firmware Builds & Board Definitions

Every module so far has assumed a stock MicroPython build for a
supported board. Production hardware is rarely exactly a stock board —
different flash size, a different pin mapping, specific frozen modules
baked in, a heap size tuned for the actual workload. This module covers
building MicroPython from source and defining a custom board. Reviewed
against MicroPython's documented build system (the `ports/*/boards/`
layout and `mpconfigboard.h`/`mpconfigboard.mk`/`.cmake` mechanism) with
no build performed here — there's no toolchain or target hardware
available in this environment.

## The build system, at a glance

MicroPython's source tree is organized by port (`ports/esp32`,
`ports/rp2`, `ports/stm32`, ...), and each port has a `boards/`
directory with one subdirectory per supported board:

```
ports/esp32/
  boards/
    ESP32_GENERIC/
      mpconfigboard.h
      mpconfigboard.cmake
      sdkconfig.board
    ESP32_GENERIC_S3/
      ...
    MY_CUSTOM_BOARD/        # a new board definition, same shape
      mpconfigboard.h
      mpconfigboard.cmake
      sdkconfig.board
      manifest.py            # optional, this board's frozen modules
```

Building for a specific board is a `make`/`idf.py`/`cmake` invocation
naming the board directory, e.g. (ESP32 port, conceptually):

```bash
cd ports/esp32
make BOARD=MY_CUSTOM_BOARD
```

A custom board definition is not a fork of MicroPython — it's a new
directory of configuration alongside the existing ones, built with the
same shared port source. This is the important framing: most custom
hardware doesn't need custom *interpreter* code, just custom *board
configuration*.

## `mpconfigboard.h` — feature and memory configuration

```c
// mpconfigboard.h (ESP32-style; other ports use analogous headers)
#define MICROPY_HW_BOARD_NAME               "MyCustomBoard"
#define MICROPY_HW_MCU_NAME                 "ESP32-S3"

#define MICROPY_PY_MACHINE_I2C              (1)
#define MICROPY_PY_MACHINE_SPI              (1)
#define MICROPY_PY_BLUETOOTH                (0)   // disabled — not used, saves flash/RAM

// Heap size and flash layout tuned for this specific product, not
// the generic development-board defaults
#define MICROPY_GC_HEAP_SIZE                (128 * 1024)
```

Disabling unused subsystems (`MICROPY_PY_BLUETOOTH` here, as an
example) isn't just tidiness — every compiled-in feature has a flash
and, for some features, RAM footprint cost, whether or not the
application ever calls it. A product that doesn't use Bluetooth
recovers real flash space by turning it off at build time rather than
just not calling the BLE APIs at runtime.

## Custom pin mappings

Board definitions map logical peripheral names to the chip's actual
pins, so application code can refer to a board-specific name rather
than a raw GPIO number that means nothing without the schematic in
hand:

```python
# boards/MY_CUSTOM_BOARD/pins.csv (or equivalent per-port mapping)
# LOGICAL_NAME, GPIO_NUMBER
LED_STATUS, 4
SENSOR_SDA, 21
SENSOR_SCL, 22
BATTERY_ADC, 34
```

Application code then references `Pin.board.LED_STATUS` (naming
conventions vary somewhat by port) instead of `Pin(4)` — the value of
this shows up the moment a board revision moves the status LED to a
different GPIO: application code doesn't change at all, only the pin
mapping does, which is exactly the kind of hardware/software decoupling
the production architecture module argues for at the driver-layer
level, applied here one level lower, at the pin level.

## Tuning heap and flash layout

Flash on a given chip is a fixed total, divided among the bootloader,
the application partition(s) (see the OTA module's A/B partitions),
any NVS/config storage, and the onboard filesystem exposed to
MicroPython. A custom partition table lets a product-specific build
allocate that space deliberately rather than accepting generic
development-board defaults sized for "works reasonably for anything":

```csv
# partitions.csv (ESP-IDF style)
# Name,     Type, SubType, Offset,   Size
nvs,        data, nvs,     0x9000,   0x5000
otadata,    data, ota,     0xe000,   0x2000
ota_0,      app,  ota_0,   0x10000,  0x180000
ota_1,      app,  ota_1,   0x190000, 0x180000
vfs,        data, fat,     0x310000, 0xF0000
```

A product shipping with a small, stable application and no expectation
of frequent large firmware updates might shrink the OTA partitions and
grow the filesystem partition for more local data logging; a product
whose firmware is expected to grow substantially over its lifetime
needs the opposite trade-off made up front, since resizing partitions
after devices are in the field generally means a full re-provisioning,
not a simple update.

## Producing reproducible release images

The property that matters most for a production build: the same
source commit, built the same way, on a different machine or a year
later, produces a bit-identical (or at least behaviorally identical)
firmware image. This mostly comes down to pinning everything the build
depends on:

- Pin the exact MicroPython commit/tag being built, not "whatever's on
  the main branch today" — a `git submodule` at a specific commit, or
  a tagged release, not a floating branch reference.
- Pin the exact toolchain version (the ESP-IDF version for ESP32
  builds, the `arm-none-eabi-gcc` version for RP2040/STM32) — toolchain
  upgrades have, historically, changed generated code size and
  occasionally behavior.
- Record the exact `manifest.py` and board configuration alongside the
  source pin, ideally in the same version-controlled repository, so a
  release image can be reconstructed from a single commit reference
  months later.
- Automate the build in CI (module 08) rather than relying on a
  particular engineer's local toolchain install, which is very hard to
  make bit-for-bit reproducible by hand.

## Cheat sheet

| Concern | Where it's controlled |
|---|---|
| Board name, enabled subsystems, heap size | `mpconfigboard.h` |
| Pin-to-peripheral mapping | Board's `pins.csv` or equivalent |
| Flash partitioning (OTA slots vs. filesystem) | `partitions.csv` (ESP-IDF-style ports) |
| Frozen modules for this board specifically | Board-local `manifest.py` |
| Reproducibility | Pinned MicroPython commit + pinned toolchain version + CI build |

## Exercise

This is a build-configuration topic with no toolchain here to invoke,
so treat it as a design exercise. Write out, as a fenced code block, a
`partitions.csv` for a product that needs: a 24 KB NVS partition, a 2 KB
otadata partition, two OTA app partitions sized generously enough for
a MicroPython+application image that's roughly 1.5 MB today with room
to grow, and the remaining flash (assume a 4 MB total flash chip) given
to the filesystem partition. Show your byte-offset arithmetic in
comments so the partition table is internally consistent (no overlaps,
no gaps left unaccounted for), and note in a final comment what would
have to happen to a fleet of already-deployed devices if this partition
table needed to change later.
