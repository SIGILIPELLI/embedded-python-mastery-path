# NeoPixels, Displays & Device UI

A blinking LED is fine for module 1; a real device needs a way to show
status and take input without a laptop attached. This module covers
driving WS2812/NeoPixel RGB LED strips with the `neopixel` module,
drawing on displays via the `framebuf` primitives underneath `ssd1306`,
and wiring up simple button-driven menu UIs — all runnable in Wokwi's
*MicroPython on ESP32* project type with NeoPixel and SSD1306 parts.

## NeoPixels: one wire, many colors

WS2812 ("NeoPixel") LEDs chain on a single data pin — each pixel reads its
color, then passes the rest of the signal down the line.

```python
from machine import Pin
import neopixel

NUM_PIXELS = 8
np = neopixel.NeoPixel(Pin(4), NUM_PIXELS)

np[0] = (255, 0, 0)      # red — tuples are (R, G, B), 0-255 each
np[1] = (0, 255, 0)      # green
np[2] = (0, 0, 255)      # blue
np.write()               # nothing changes on the strip until write()!

np.fill((0, 0, 0))       # set every pixel to off...
np.write()               # ...then push it
```

Same two-step model as the OLED (module in Level 1): set values in a
buffer, then `write()` to actually shift the data out. Forgetting
`write()` is the single most common NeoPixel bug.

**Wiring (Wokwi):** add a **NeoPixel Ring** or **NeoPixel Strip** part,
data pin → GPIO 4, VCC → 5V, GND → GND.

### Animations

```python
import time

def rainbow_cycle(np, n, wait_ms=20):
    for j in range(256):
        for i in range(n):
            idx = (i * 256 // n + j) & 255
            np[i] = _wheel(idx)
        np.write()
        time.sleep_ms(wait_ms)

def _wheel(pos):
    if pos < 85:
        return (255 - pos * 3, pos * 3, 0)
    elif pos < 170:
        pos -= 85
        return (0, 255 - pos * 3, pos * 3)
    else:
        pos -= 170
        return (pos * 3, 0, 255 - pos * 3)
```

`_wheel()` is pure math (position → RGB) and testable with `python3`
directly — only `np.write()` needs real hardware.

## Displays: `framebuf` under the hood

`ssd1306.SSD1306_I2C` (Level 1, module 6) is actually a thin wrapper
around `framebuf.FrameBuffer` — the same drawing primitives work on any
display driver built on `framebuf` (SSD1306, ST7789, SH1106):

```python
oled.fill(0)
oled.pixel(10, 10, 1)
oled.line(0, 0, 127, 63, 1)
oled.rect(10, 10, 40, 20, 1)          # outline
oled.fill_rect(60, 10, 40, 20, 1)     # filled
oled.text("Hi", 0, 50)
oled.show()
```

Color values are `1` (on) / `0` (off) on a monochrome OLED; color
displays (ST7789) use `framebuf`'s RGB565 format instead — a different
`FrameBuffer` mode but the same drawing calls.

## Building a button-driven menu

A minimal state machine: current menu index, a button to move down, a
button to select.

```python
from machine import Pin
import ssd1306
import time

MENU_ITEMS = ["Show Temp", "Show Humidity", "Toggle LED", "Sleep"]

btn_next = Pin(4, Pin.IN, Pin.PULL_UP)   # active-low
btn_select = Pin(5, Pin.IN, Pin.PULL_UP)

selected = 0

def draw_menu(oled, items, selected):
    oled.fill(0)
    for i, item in enumerate(items):
        prefix = "> " if i == selected else "  "
        oled.text(prefix + item, 0, i * 10)
    oled.show()

def debounced_press(pin, last_state):
    state = pin.value()
    pressed = (last_state == 1 and state == 0)   # falling edge
    return pressed, state

last_next, last_select = 1, 1

draw_menu(oled, MENU_ITEMS, selected)
while True:
    pressed_next, last_next = debounced_press(btn_next, last_next)
    pressed_select, last_select = debounced_press(btn_select, last_select)

    if pressed_next:
        selected = (selected + 1) % len(MENU_ITEMS)
        draw_menu(oled, MENU_ITEMS, selected)
        time.sleep_ms(150)          # crude debounce

    if pressed_select:
        print("selected:", MENU_ITEMS[selected])
        # dispatch to a handler per item here
        time.sleep_ms(150)

    time.sleep_ms(20)
```

The redraw-only-on-change pattern matters: calling `oled.show()` every
loop iteration (instead of only after a button press) wastes I2C
bandwidth and can introduce visible flicker on some displays — redraw on
state change, not on a timer.

## UI/display gotchas

!!! warning "NeoPixel brightness and power budget"
    A single WS2812 at full white draws ~60 mA. Eight pixels at full
    white is nearly half an amp — enough to brown out a board powered
    from a weak USB port or a small battery. Keep brightness modest
    (scale RGB values down, e.g. multiply by 0.2) unless power is
    confirmed adequate.

!!! warning "NeoPixel timing is sensitive to interrupts"
    `neopixel.write()` bit-bangs precise timing; a poorly-timed interrupt
    (from `machine.Timer` callbacks, or heavy `asyncio` scheduling) mid-write
    can corrupt colors on some pixels. Keep other time-critical work out of
    the immediate vicinity of `np.write()` calls where possible.

!!! warning "Un-debounced buttons fire many menu moves per press"
    A mechanical button "bounces" — several rapid on/off transitions in the
    milliseconds around a press. The falling-edge check plus a short
    `sleep_ms()` above is a *crude* debounce; a heavily-bounced button can
    still double-fire. For anything that must be rock solid, sample the
    pin a few times a millisecond apart and require several consistent
    readings before accepting the state as changed.

!!! warning "Framebuf memory scales with resolution"
    A 128×64 monochrome display's buffer is 1KB; a 240×240 color ST7789
    buffer (RGB565) is well over 100KB — a meaningful chunk of the
    ESP32's RAM. Allocating a big `framebuf.FrameBuffer` on a
    memory-constrained board can trigger the fragmentation issues from
    Level 1 if done repeatedly rather than once at startup.

## Cheat sheet

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

## Exercise

Build a **status beacon + menu** in Wokwi: an 8-pixel NeoPixel ring on pin
4, an SSD1306 on I2C(22/21), and two buttons (pins 4 and 5 conflict with
the NeoPixel pin above — pick 18/19 instead) for next/select. The menu has
three items: "Idle" (NeoPixels off), "Warning" (NeoPixels solid yellow,
dimmed to ~20% brightness), and "Alert" (NeoPixels flash red, alternating
every 300 ms). Draw the current menu with a `>` cursor on the OLED,
redrawing only when the selection or NeoPixel state changes. Debounce both
buttons with the falling-edge-plus-delay pattern, and verify in Wokwi that
holding a button down doesn't rapid-fire multiple menu advances.
