# Embedded Python Mastery Path

A structured, module-wise MicroPython training program that takes you from
your first blinking LED to production-grade, master-level embedded Python —
ESP32-first (with Raspberry Pi Pico notes along the way), with real, runnable
MicroPython code in every module and a hands-on project at the end of each
level.

**No hardware required to start.** Every Level 1 example runs in the free
[Wokwi online simulator](https://wokwi.com/) — create a *MicroPython on
ESP32* project in the browser and you get a virtual ESP32 with LEDs, buttons,
potentiometers, DHT22 sensors, and OLED displays. Everything also works
unchanged on real boards.

## How the program is organized

| Level | Focus | Modules |
|-------|-------|---------|
| [Level 1 · Entry](level-1/index.md) | MicroPython on ESP32: REPL workflow, GPIO, PWM/ADC, timers & interrupts, I2C sensors, files, WiFi, `uasyncio` | 9 topics + 1 capstone |
| [Level 2 · Intermediate](level-2/index.md) | MQTT, BLE with `aioble`, deep sleep & battery power, custom drivers, tooling, ESP-NOW | 9 topics + 1 project |
| [Level 3 · Advanced](level-3/index.md) | Memory & GC internals, native/viper emitters, C modules, RP2040 PIO, performance work | 9 topics + 1 project |
| [Level 4 · Master](level-4/index.md) | Production MicroPython: OTA, provisioning, robustness, custom firmware builds, manufacturing | 9 topics + 1 capstone |

## How to use this site

- Work through each level in order — later modules assume earlier ones.
- Every topic page has real, runnable MicroPython code — paste it into a
  [Wokwi](https://wokwi.com/) *MicroPython on ESP32* project (or onto a real
  board with Thonny/`mpremote`) and run it as you read.
- Each level ends with a project that combines everything learned in that
  level.
- New to Python itself? Do
  [Python Mastery Path — Level 1](https://sigilipelli.github.io/python-mastery-path/)
  first; this site assumes basic Python syntax and jumps straight to the
  embedded parts.
- Prefer C/C++ on microcontrollers? That's the sibling
  [Embedded Systems Mastery Path](https://sigilipelli.github.io/embedded-mastery-path/) —
  the two tracks cover the same hardware from two languages.
- Use the search bar (top of the page) to jump straight to a topic.

Start here → [Level 1 · Entry](level-1/index.md)

🎥 **Prefer video?** Watch the [Mastery Path video series](https://youtube.com/@sigilipelli) on YouTube — Shorts and full walkthroughs of these lessons.

## More from the Mastery Path series

Free, structured, module-wise training across 36 other languages and platforms:

<div class="mastery-grid-wrap">
<p class="mastery-grid-category">Languages</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/python-mastery-path/">🐍 Python</a>
  <a href="https://sigilipelli.github.io/java-mastery-path/">☕ Java</a>
  <a href="https://sigilipelli.github.io/javascript-mastery-path/">🟨 JavaScript</a>
  <a href="https://sigilipelli.github.io/typescript-mastery-path/">🔷 TypeScript</a>
  <a href="https://sigilipelli.github.io/shell-mastery-path/">🐚 Shell/Bash</a>
  <a href="https://sigilipelli.github.io/powershell-mastery-path/">💻 PowerShell</a>
  <a href="https://sigilipelli.github.io/c-mastery-path/">🇨 C</a>
  <a href="https://sigilipelli.github.io/cpp-mastery-path/">➕ C++</a>
  <a href="https://sigilipelli.github.io/go-mastery-path/">🐹 Go</a>
  <a href="https://sigilipelli.github.io/rust-mastery-path/">🦀 Rust</a>
  <a href="https://sigilipelli.github.io/sql-mastery-path/">🗄️ SQL</a>
  <a href="https://sigilipelli.github.io/ruby-mastery-path/">💎 Ruby</a>
  <a href="https://sigilipelli.github.io/php-mastery-path/">🐘 PHP</a>
  <a href="https://sigilipelli.github.io/kotlin-mastery-path/">🟣 Kotlin</a>
  <a href="https://sigilipelli.github.io/swift-mastery-path/">🐦 Swift</a>
  <a href="https://sigilipelli.github.io/dart-mastery-path/">🎯 Dart</a>
  <a href="https://sigilipelli.github.io/scala-mastery-path/">🔴 Scala</a>
  <a href="https://sigilipelli.github.io/r-mastery-path/">📊 R</a>
</div>
<p class="mastery-grid-category">Cloud Platforms</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/aws-mastery-path/">☁️ AWS</a>
  <a href="https://sigilipelli.github.io/azure-mastery-path/">☁️ Azure</a>
  <a href="https://sigilipelli.github.io/gcp-mastery-path/">☁️ GCP</a>
  <a href="https://sigilipelli.github.io/ibm-cloud-mastery-path/">☁️ IBM Cloud</a>
  <a href="https://sigilipelli.github.io/adobe-mastery-path/">🎨 Adobe</a>
</div>
<p class="mastery-grid-category">AI / ML / LLM</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/ai-ml-mastery-path/">🤖 AI/ML</a>
  <a href="https://sigilipelli.github.io/llm-dev-mastery-path/">🧠 LLM Dev</a>
  <a href="https://sigilipelli.github.io/rag-mastery-path/">📚 RAG</a>
  <a href="https://sigilipelli.github.io/edge-ai-mastery-path/">📱 Edge AI</a>
</div>
<p class="mastery-grid-category">Embedded Systems</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/embedded-mastery-path/">🔌 Embedded</a>
  <a href="https://sigilipelli.github.io/embedded-linux-mastery-path/">🐧 Embedded Linux</a>
  <a href="https://sigilipelli.github.io/freertos-mastery-path/">⏱️ FreeRTOS</a>
  <a href="https://sigilipelli.github.io/s32k-mastery-path/">🔧 S32K</a>
</div>
<p class="mastery-grid-category">Leadership & Management</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/product-manager-mastery-path/">📋 Product Manager</a>
  <a href="https://sigilipelli.github.io/product-lead-mastery-path/">🧭 Product Lead</a>
  <a href="https://sigilipelli.github.io/project-manager-mastery-path/">📅 Project Manager</a>
  <a href="https://sigilipelli.github.io/ai-manager-mastery-path/">🤖 AI Manager</a>
  <a href="https://sigilipelli.github.io/servant-leadership-mastery-path/">🤝 Servant Leadership</a>
</div>
</div>
