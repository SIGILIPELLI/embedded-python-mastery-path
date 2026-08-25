# Project — High-Speed Data Acquisition

This project combines every tool from Level 3 into one build: sample a
signal as fast as the hardware allows, move the samples off the ADC
without allocating, hand them to a second core for processing while
the first keeps sampling, and stream the results to a host — all while
staying inside a heap that can't tolerate careless allocation. There's
no target hardware here, so this is a full design document reviewed
for technical consistency against each module's documented behavior,
with the sampling-rate math and buffer-sizing logic verified via
`python3`.

## The requirement

Sample an analog signal at a sustained rate high enough that Python's
interpreter loop overhead alone would miss samples, buffer without
allocating during the hot path, and stream results over UART/USB to a
host continuously — with no gaps and no `MemoryError` after the first
few seconds of otherwise-successful running.

## Architecture overview

```
   Core 0 (PIO-fed)                     Core 1
 ┌────────────────────┐    ring       ┌──────────────────────┐
 │ PIO SM: ADC pacing  │──buffer─────▶│ Consume ring buffer   │
 │ + DMA-style FIFO    │  (bytearray, │ Compute rolling stats │
 │ drain into ring buf │   preallocated│ Frame + checksum      │
 └────────────────────┘   at boot)    │ UART/USB write        │
                                       └──────────────────────┘
```

- **PIO** paces ADC reads at a precise, jitter-free interval — the CPU
  never has to spin-wait or trust a software timer for the sample
  clock (module 05).
- **A preallocated ring buffer**, sized once at boot, is the only
  handoff structure between the sampling side and the processing side
  — no allocation on the hot path (module 01).
- **`_thread`** puts sampling-side draining on one core and
  processing/streaming on the other, so a slow UART write never stalls
  the sample clock (module 09), guarded by a lock sized to the
  smallest critical section possible.
- **`@micropython.viper`** implements the rolling-statistics/checksum
  computation, the one CPU-bound piece that must not add up to more
  time per sample than the sample period allows (module 03).
- **`uasyncio`** on Core 1 structures the "stream to host" and
  "watch for host commands" concurrency without a second layer of
  threads (module 06).
- **`gc.collect()`** is called only at a known idle point — the top of
  Core 1's outer loop after a batch is flushed — never inside the
  per-sample path (module 01, 02).

## The ring buffer — allocation-free by construction

```python
class RingBuffer:
    """Fixed-size ring buffer over a pre-allocated bytearray.
    No method here allocates after __init__."""

    def __init__(self, capacity):
        self._buf = bytearray(capacity)
        self._capacity = capacity
        self._head = 0   # next write position
        self._tail = 0   # next read position
        self._count = 0  # bytes currently held

    def write_byte(self, value):
        if self._count >= self._capacity:
            # Buffer full: caller side is falling behind the sample
            # rate — this must be visible, not silently swallowed.
            return False
        self._buf[self._head] = value
        self._head = (self._head + 1) % self._capacity
        self._count += 1
        return True

    def read_byte(self):
        if self._count == 0:
            return None
        value = self._buf[self._tail]
        self._tail = (self._tail + 1) % self._capacity
        self._count -= 1
        return value

    def available(self):
        return self._count
```

Every field is a plain int; `_buf` is allocated exactly once. The
`% self._capacity` wraparound is the same pattern PIO's own FIFO uses
conceptually — a fixed-size structure with independent read/write
cursors, no resizing, ever. `write_byte` returning `False` on overflow
rather than silently dropping data (or worse, raising inside a
time-critical path) is deliberate: the caller (the PIO-draining loop)
increments an overflow counter it can report later, rather than paying
for exception handling in the hot path.

## Feeding the buffer from a PIO-paced ADC read

```python
import rp2
from machine import Pin, ADC

adc = ADC(Pin(26))

@rp2.asm_pio(sideset_init=rp2.PIO.OUT_LOW)
def sample_clock():
    # A minimal free-running pulse used as the sample-rate reference;
    # the CPU polls the IRQ this raises rather than trusting a
    # software timer (see module 05 for wrap/side-set details).
    wrap_target()
    set(pins, 1) .side(1) [9]
    set(pins, 0) .side(0) [9]
    wrap()

def sampling_loop(ring, sm, n_samples):
    overflow_count = 0
    for _ in range(n_samples):
        # In real firmware: block on the PIO IRQ / FIFO here instead
        # of a plain loop, so sampling is paced by hardware, not by
        # how fast this Python loop happens to run.
        raw = adc.read_u16() >> 8   # keep to one byte per sample here
        if not ring.write_byte(raw):
            overflow_count += 1
    return overflow_count
```

The comment about blocking on the PIO IRQ instead of a plain loop is
the crucial correctness point: a bare Python `for` loop calling
`adc.read_u16()` has no guaranteed timing — GC pauses, interpreter
overhead, and thread scheduling all introduce jitter. The PIO's role is
to be the actual pacing signal; the Python loop's job is only to drain
whatever the hardware has ready when signaled, never to *be* the clock.

## Core 1 — process and stream without starving Core 0

```python
import _thread
import gc

def processing_task(ring, lock, uart):
    batch = bytearray(64)   # preallocated once, reused every batch
    while True:
        idx = 0
        with lock:
            while idx < len(batch) and ring.available() > 0:
                batch[idx] = ring.read_byte()
                idx += 1
        if idx:
            checksum = 0
            for i in range(idx):
                checksum ^= batch[i]
            uart.write(batch[:idx])
            uart.write(bytes([checksum]))
        gc.collect()   # safe point: batch just flushed, nothing in flight

_thread.start_new_thread(processing_task, (ring, lock, uart))
```

`gc.collect()` runs here specifically because this is a natural idle
boundary — the batch has just been written out, and nothing is
mid-transaction. Calling it anywhere inside the `while idx < ...`
drain loop would risk a multi-millisecond pause landing while the lock
is held, stalling Core 0's writer too (module 01's GC-pause timing
discussion applies directly here, compounded by the cross-core lock).

## Sizing the buffer — the math, checked

```python
sample_rate_hz = 10_000       # target sustained sample rate
worst_case_stall_ms = 50      # longest expected UART/processing stall
bytes_per_sample = 1

needed_capacity = int(sample_rate_hz * (worst_case_stall_ms / 1000) * bytes_per_sample)
# needed_capacity = 500 bytes to survive a 50ms stall at 10kHz without overflow

buffer_capacity = needed_capacity * 2   # safety margin
print("ring buffer capacity:", buffer_capacity, "bytes")
```

This is the kind of sizing decision that must be made deliberately and
verified, not guessed: too small and any transient stall (a slow UART
write, a lock contention spike) overflows and silently drops samples;
too large and the ring buffer itself becomes the biggest single
allocation on a heap that also needs room for everything else,
increasing fragmentation risk (module 01) rather than reducing it.

## Where each module's trap shows up in this design

| Trap | Where it would bite here |
|---|---|
| GC pause mid-critical-section | `gc.collect()` placed outside the lock, at a flush boundary — not inside the drain loop |
| Viper `int` overflow | The checksum/stats routine must use `uint`/masking deliberately if ported to viper, not assume Python's bignum promotion |
| PIO 32-instruction limit | The sample-clock program stays under a handful of instructions via `wrap_target()`/`.side()` |
| Dual-core race | Every ring-buffer access from both cores goes through `lock` — no shared field is read or written unlocked |
| Allocation in the hot path | `RingBuffer` and `batch` are both allocated exactly once, at setup |

## Cheat sheet

| Component | Module it draws from |
|---|---|
| PIO sample-rate pacing | 05 — RP2040 PIO |
| Preallocated ring buffer | 01 — Memory & GC |
| Cross-core lock around the buffer | 09 — Threads & dual-core |
| Rolling stats/checksum in viper | 03 — Native & viper emitters |
| Host streaming loop structure | 06 — Advanced uasyncio |
| `gc.collect()` at flush boundaries only | 01 & 02 — GC timing, profiling methodology |

## Stretch goals

- Replace the placeholder polling `sampling_loop` with a real PIO
  IRQ-driven handoff (`sm.irq(handler)`), and reason through, in
  writing, what changes about the overflow-counting logic when samples
  arrive via interrupt rather than a synchronous loop.
- Move the checksum/rolling-average computation into an actual
  `@micropython.viper` function with explicit `ptr8` access into
  `batch`, and write out the exact bounds check that must accompany
  it since viper gives none for free.
- Extend the design to two simultaneous ADC channels feeding two ring
  buffers, and describe where a second lock (versus reusing one) is
  the correct choice, referencing the dual-core race material.
- Add a host-side "backpressure" command (a byte the host can send to
  pause streaming) handled via the `uasyncio` layer on Core 1, and
  explain why this belongs in the async layer rather than in the
  locked ring-buffer drain path.
