---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Embedded Python Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What's MicroPython's biggest practical difference from desktop Python?",
    "options": [
      "No syntax",
      "More libraries",
      "A tiny heap and reduced library, so you must manage memory and hardware directly",
      "More RAM"
    ],
    "correctIndex": 2,
    "explanation": "Design for constraints."
  },
  {
    "question": "Which REPL shortcut soft-resets the board and re-runs main.py?",
    "options": [
      "Ctrl-Z",
      "Ctrl-X",
      "Ctrl-C",
      "Ctrl-D"
    ],
    "correctIndex": 3,
    "explanation": "Ctrl-C interrupts the running program."
  },
  {
    "question": "A button wired to ground should read a clean value. How do you configure the pin?",
    "options": [
      "No pull",
      "PWM",
      "Pin.OUT",
      "machine.Pin(n, Pin.IN, Pin.PULL_UP) and debounce it"
    ],
    "correctIndex": 3,
    "explanation": "Avoid floating inputs."
  },
  {
    "question": "A 12-bit ADC at 3.3 V reads 3000. What voltage is that?\n```python\nprint(round(3000/4095*3.3,2))\n```\n",
    "options": [
      "1.2",
      "2.42",
      "3.0",
      "3.3"
    ],
    "correctIndex": 1,
    "explanation": "raw/4095 x Vref. Verified by running it."
  },
  {
    "question": "An interrupt callback allocates a list and raises MemoryError. Why?",
    "options": [
      "ISRs can't allocate heap memory in MicroPython; set a flag or use micropython.schedule",
      "Pin error",
      "Syntax",
      "Too slow"
    ],
    "correctIndex": 0,
    "explanation": "Preallocate buffers."
  },
  {
    "question": "A sensor sends bytes 0x34 0x12 for a little-endian 16-bit value. What is it?\n```python\nimport struct\nprint(struct.unpack('<H', bytes([0x34,0x12]))[0])\n```\n",
    "options": [
      "18",
      "4660",
      "13330",
      "52"
    ],
    "correctIndex": 1,
    "explanation": "0x1234 = 4660. Verified by running it."
  },
  {
    "question": "Why batch flash writes in a logger?",
    "options": [
      "Flash wears out after limited erase cycles",
      "Law",
      "Style",
      "Slow"
    ],
    "correctIndex": 0,
    "explanation": "Rotate and bound logs."
  },
  {
    "question": "A device must do networking, sensing and display without threads. What do you use?",
    "options": [
      "Threads",
      "uasyncio tasks with await asyncio.sleep() instead of time.sleep()",
      "Polling loops with sleep",
      "Interrupts"
    ],
    "correctIndex": 1,
    "explanation": "Cooperative multitasking."
  },
  {
    "question": "The MQTT client stops receiving after a while. What did you likely forget?",
    "options": [
      "Baud rate",
      "Pull-ups",
      "Calling check_msg()/wait_msg() and keepalive pings regularly",
      "Wi-Fi power"
    ],
    "correctIndex": 2,
    "explanation": "Else the broker disconnects."
  },
  {
    "question": "Wake 2 s at 80 mA each minute, otherwise deep sleep at 0.02 mA. What's the average current (mA)?\n```python\nprint(round((2*80+58*0.02)/60,3))\n```\n",
    "options": [
      "0.02",
      "2.686",
      "80",
      "1.33"
    ],
    "correctIndex": 1,
    "explanation": "Duty-cycled average. Verified by running it."
  },
  {
    "question": "After deep sleep, your variables are gone. Where can state survive?",
    "options": [
      "RAM",
      "RTC memory or flash/NVS (RAM is cleared)",
      "Stack",
      "Registers"
    ],
    "correctIndex": 1,
    "explanation": "Deep sleep restarts the program."
  },
  {
    "question": "The board thinks it's the year 2000. What do you do?",
    "options": [
      "Sync the RTC with ntptime over WiFi and store UTC",
      "Use local time",
      "Set manually",
      "Ignore"
    ],
    "correctIndex": 0,
    "explanation": "Timestamps need a valid clock."
  },
  {
    "question": "Using wrap-safe tick arithmetic, what's ticks_diff(5, MASK-5) with a 30-bit counter?\n```python\nM=(1<<30)-1\nd=(5-(M-5))&M\nprint(d-(M+1) if d>=(M+1)//2 else d)\n```\n",
    "options": [
      "11",
      "5",
      "1073741819",
      "-11"
    ],
    "correctIndex": 0,
    "explanation": "Never subtract tick values directly. Verified by running it."
  },
  {
    "question": "A hot loop is too slow. What's the correct order?",
    "options": [
      "Rewrite everything",
      "Measure with ticks_us, then try @native/@viper, then consider C modules or PIO",
      "Overclock",
      "Assembly first"
    ],
    "correctIndex": 1,
    "explanation": "Profile before optimizing."
  },
  {
    "question": "Heap allocation fails though gc.mem_free() looks large. What's the cause?",
    "options": [
      "Fragmentation: no contiguous block big enough; preallocate and call gc.collect()",
      "WiFi",
      "Flash full",
      "Bug"
    ],
    "correctIndex": 0,
    "explanation": "Large buffers need contiguous space."
  },
  {
    "question": "A field device hangs and never recovers. What's the standard protection?",
    "options": [
      "Reboot manually",
      "Print more",
      "A hardware watchdog fed only when the system is healthy",
      "Sleep"
    ],
    "correctIndex": 2,
    "explanation": "Blind feeding hides hangs."
  },
  {
    "question": "An OTA update loses power halfway. What design avoids a bricked device?",
    "options": [
      "Download to a temp file, verify the hash/signature, swap atomically and keep a fallback",
      "Trust the link",
      "Overwrite",
      "Skip verification"
    ],
    "correctIndex": 0,
    "explanation": "Safe updates."
  },
  {
    "question": "How should WiFi credentials and API keys be handled?",
    "options": [
      "In main.py",
      "In the repo",
      "Provisioned per device and stored outside source control (e.g. encrypted NVS)",
      "In comments"
    ],
    "correctIndex": 2,
    "explanation": "Never hardcode."
  },
  {
    "question": "How do you unit-test hardware-independent MicroPython logic in CI?",
    "options": [
      "Never",
      "Run it under CPython or the unix port with mocked machine modules",
      "Only on a board",
      "By printing"
    ],
    "correctIndex": 1,
    "explanation": "Fast feedback."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### A wrap-safe cooperative scheduler

Implement ticks_add, ticks_diff (correct across counter wraparound like time.ticks_diff) and Scheduler.every/poll so jobs run when their period elapses without blocking and without drifting. The simulation starts near the counter wrap.

**Starter code**

```python
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

**Expected output**

```text
100:blink 200:blink 250:sensor 300:blink 400:blink 500:blink+sensor 600:blink 700:blink
11 -11
```

<details><summary>Show solution and explanation</summary>

<pre><code>MASK = (1 &lt;&lt; 30) - 1          # tick counter wraps like time.ticks_ms() on a MicroPython port

def ticks_add(t, d):
    return (t + d) &amp; MASK

def ticks_diff(a, b):
    d = (a - b) &amp; MASK
    return d - (MASK + 1) if d &gt;= (MASK + 1) // 2 else d

class Scheduler:
    &quot;&quot;&quot;Cooperative scheduler: run jobs whose period has elapsed (no sleep() blocking).&quot;&quot;&quot;
    def __init__(self):
        self.jobs = []                    # [name, period, next_due]

    def every(self, name, period, now):
        self.jobs.append([name, period, ticks_add(now, period)])

    def poll(self, now):
        ran = []
        for job in self.jobs:
            if ticks_diff(now, job[2]) &gt;= 0:
                ran.append(job[0])
                job[2] = ticks_add(job[2], job[1])
        return ran

s = Scheduler()
start = MASK - 250                        # counter about to wrap
s.every(&quot;blink&quot;, 100, start)
s.every(&quot;sensor&quot;, 250, start)
log = []
for step in range(0, 701, 50):
    now = ticks_add(start, step)
    ran = s.poll(now)
    if ran:
        log.append(f&quot;{step}:{&#x27;+&#x27;.join(ran)}&quot;)
print(&quot; &quot;.join(log))
print(ticks_diff(5, MASK - 5), ticks_diff(MASK - 5, 5))
</code></pre>

<p>Subtracting raw tick values breaks at wraparound, so differences are computed modulo the counter size and interpreted as signed. Rescheduling from the previous due time (not &#x27;now&#x27;) avoids drift. Verified by running the solution (CPython model of the MicroPython API).</p>
</details>

### Bounded log and reconnect backoff

Implement BoundedLog.append (drop the oldest lines until the byte budget fits, keep at least one) and Backoff.failure (min(cap, base*2**fails)), the two building blocks of a resilient, flash-friendly logger.

**Starter code**

```python
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

**Expected output**

```text
4 60 {"i":4,"t":24}
[1, 2, 4, 8, 16, 30, 30]
1
```

<details><summary>Show solution and explanation</summary>

<pre><code>import json

class BoundedLog:
    &quot;&quot;&quot;Keep the newest entries within a byte budget, like a size-limited log file on flash.&quot;&quot;&quot;
    def __init__(self, max_bytes):
        self.max_bytes = max_bytes
        self.lines = []

    def size(self):
        return sum(len(l) + 1 for l in self.lines)

    def append(self, record):
        self.lines.append(json.dumps(record, separators=(&quot;,&quot;, &quot;:&quot;)))
        while self.size() &gt; self.max_bytes and len(self.lines) &gt; 1:
            self.lines.pop(0)

class Backoff:
    &quot;&quot;&quot;Exponential backoff with a cap, reset on success (reconnect logic).&quot;&quot;&quot;
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
    log.append({&quot;i&quot;: i, &quot;t&quot;: 20 + i})
print(len(log.lines), log.size(), log.lines[0])
b = Backoff()
print([b.failure() for _ in range(7)])
b.success()
print(b.failure())
</code></pre>

<p>Bounding the log protects limited flash and RAM, and capped exponential backoff keeps reconnect attempts from hammering a dead network or draining the battery. Verified by running the solution.</p>
</details>
