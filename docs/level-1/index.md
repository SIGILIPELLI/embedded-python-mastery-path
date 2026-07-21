# Level 1 · Entry <span class="level-badge">Foundations</span>

Goal: get comfortable programming microcontrollers in Python using
MicroPython on the ESP32 — the REPL-driven workflow, digital and analog I/O,
timers and interrupts, I2C sensors and displays, the onboard filesystem,
WiFi and HTTP, and cooperative multitasking with `uasyncio` — finishing with
a complete simulated WiFi sensor logger with its own web dashboard.

**You do not need to own any hardware for this level.** Every example runs
in the free [Wokwi online simulator](https://wokwi.com/): create a new
project and choose the **MicroPython on ESP32** template, and you get a
browser-based ESP32 with virtual LEDs, buttons, potentiometers, DHT22
sensors, and OLED displays — plus a working REPL and simulated WiFi.
Everything also works unchanged on real ESP32 boards (and, with pin-number
changes, on the Raspberry Pi Pico).

**Prerequisite:** basic Python — variables, functions, loops, classes,
exceptions. If you're new to Python, do
[Python Mastery Path — Level 1](https://sigilipelli.github.io/python-mastery-path/)
first; this level assumes it.

## Modules

1. [MicroPython vs. Python](01-micropython-vs-python.md)
2. [REPL & Workflow](02-repl-workflow.md)
3. [GPIO Basics](03-gpio-basics.md)
4. [PWM & ADC](04-pwm-adc.md)
5. [Timers & Interrupts](05-timers-interrupts.md)
6. [I2C & SPI Sensors](06-i2c-spi-sensors.md)
7. [Files & Persistence](07-files-persistence.md)
8. [WiFi & HTTP](08-wifi-http.md)
9. [uasyncio Basics](09-uasyncio-basics.md)
10. [Capstone — WiFi Sensor Logger](10-capstone-wifi-sensor-logger.md)

By the end of this level you'll be able to read buttons and sensors, drive
LEDs and displays, log data to flash, connect an ESP32 to WiFi and serve a
web page from it, run several activities concurrently with `uasyncio`, and
combine all of it into a working networked device — entirely in the browser
if you want.
