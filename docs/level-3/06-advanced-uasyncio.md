# Advanced uasyncio Patterns

Basic `async def`/`await` gets you concurrent I/O without threads. Real
device firmware needs more: coordinating multiple coroutines safely,
cancelling work that's taking too long, and structuring a program that
does several unrelated things at once without turning into a tangle of
callbacks. `uasyncio` (MicroPython's `asyncio`) supports most of
CPython's `asyncio` primitives with a few device-specific gaps worth
knowing. Coordination logic below runs under plain `python3`'s
`asyncio` — the same primitives, close enough semantics — with
MicroPython-specific gaps called out explicitly.

## Events — one-shot signaling between coroutines

```python
import asyncio

async def waiter(event, name):
    print(name, "waiting")
    await event.wait()
    print(name, "proceeding")

async def setter(event):
    await asyncio.sleep(1)
    print("setting event")
    event.set()

async def main():
    event = asyncio.Event()
    await asyncio.gather(
        waiter(event, "A"),
        waiter(event, "B"),
        setter(event),
    )

asyncio.run(main())
```

`Event.set()` wakes **every** coroutine waiting on it, not just one —
that's what distinguishes it from a queue. Once set, `event.wait()`
returns immediately for any coroutine that awaits it afterward too,
until `event.clear()` resets it. A common bug is expecting `set()` to
behave like a single-consumer signal; if only one waiter should proceed
per signal, a `Lock` or a bounded `Queue` is the right primitive, not
`Event`.

## Locks — protecting shared state across coroutines

```python
import asyncio

counter = 0
lock = asyncio.Lock()

async def increment():
    global counter
    async with lock:
        current = counter
        await asyncio.sleep(0)   # yield point — without the lock, a race here
        counter = current + 1

async def main():
    await asyncio.gather(*(increment() for _ in range(10)))
    print(counter)   # 10, reliably, because the lock serializes the critical section

asyncio.run(main())
```

Because `uasyncio` is cooperative single-threaded scheduling, a race
only becomes possible at an actual `await` point inside a "critical
section" — code with no `await` in it can't be interrupted by another
coroutine. This is different from the `_thread`/dual-core races covered
next module, where **any** line can be preempted. It's tempting to
conclude locks are unnecessary in `uasyncio` for that reason; they're
still needed whenever a critical section spans an `await` (as above),
which is common — any critical section that itself awaits I/O.

## Queues — producer/consumer without polling

```python
import asyncio

async def producer(queue):
    for i in range(5):
        await queue.put(i)
        await asyncio.sleep(0.1)
    await queue.put(None)   # sentinel signaling "done"

async def consumer(queue):
    while True:
        item = await queue.get()
        if item is None:
            break
        print("consumed:", item)

async def main():
    queue = asyncio.Queue(maxsize=3)
    await asyncio.gather(producer(queue), consumer(queue))

asyncio.run(main())
```

`Queue(maxsize=3)` makes `put()` block once the queue is full, which is
what provides backpressure — a producer that's faster than its
consumer is throttled automatically instead of growing the queue
unbounded (a real memory concern given how small the heap is; see the
memory module). A sentinel value (`None` here) is the idiomatic way to
signal "no more items" since `uasyncio` queues have no built-in
"closed" state.

## Stream readers/writers

`uasyncio` supports `asyncio.StreamReader`/`StreamWriter` for
socket-based protocols (this maps closely onto MicroPython's
`asyncio.open_connection` / `asyncio.start_server`):

```python
async def handle_client(reader, writer):
    data = await reader.readline()
    writer.write(b"echo: " + data)
    await writer.drain()
    await writer.wait_closed()

async def main():
    server = await asyncio.start_server(handle_client, "0.0.0.0", 8080)
    async with server:
        await server.serve_forever()
```

`await writer.drain()` after a write is easy to skip and easy to
regret — without it, writes queue up in an internal buffer with no
backpressure, and a slow or stalled client can grow that buffer
indefinitely on a device with a few hundred KB of heap. Always pair
`write()` with `drain()` in a loop.

## Graceful cancellation and timeouts

```python
import asyncio

async def slow_sensor_read():
    await asyncio.sleep(5)
    return 42

async def main():
    try:
        result = await asyncio.wait_for(slow_sensor_read(), timeout=1)
        print("got:", result)
    except asyncio.TimeoutError:
        print("sensor read timed out — falling back to cached value")

asyncio.run(main())
```

`asyncio.wait_for` cancels the inner coroutine by raising
`CancelledError` inside it at its next `await` point — cancellation is
**cooperative**, not immediate. Code holding a resource (a lock, an
open file, a peripheral in a half-configured state) needs a `finally`
block to release it, because cancellation is just another exception
being raised at an await point:

```python
async def careful_read(lock):
    async with lock:
        try:
            await asyncio.sleep(10)   # cancellable here
        finally:
            print("cleanup runs even if cancelled")
```

Catching `CancelledError` and *not* re-raising it is a documented
anti-pattern — it silently defeats the cancellation the caller
requested. If a `try/except` around await needs cleanup logic, use
`finally`, and let `CancelledError` propagate unless there's a specific
reason to suppress it (rare, and worth a comment explaining why).

## Structuring larger async applications

For anything beyond a couple of coroutines, structure the app as a set
of independent long-running tasks started once and supervised, rather
than nesting `gather()` calls:

```python
async def sensor_task(state):
    while True:
        state["temp"] = read_temp()
        await asyncio.sleep(1)

async def network_task(state):
    while True:
        await publish(state.get("temp"))
        await asyncio.sleep(5)

async def main():
    state = {}
    tasks = [
        asyncio.create_task(sensor_task(state)),
        asyncio.create_task(network_task(state)),
    ]
    await asyncio.gather(*tasks)
```

A shared plain `dict` (`state` above) is a common, simple way to pass
data between long-running tasks without a queue when the "freshest
value wins" semantics are fine — but it's a design choice, not a
default: use a `Queue` instead when every value must be seen exactly
once (an event, not a reading).

## MicroPython-specific gaps versus CPython's `asyncio`

- No `asyncio.TaskGroup` (a more recent CPython addition) on most
  MicroPython builds — `gather()` plus a supervising loop is the
  portable pattern.
- Exception handling inside `gather()`: an unhandled exception in one
  task doesn't necessarily cancel siblings the way `TaskGroup` does in
  recent CPython — check the specific `uasyncio` version's behavior
  before relying on it.
- Testing async device code with no event loop of its own available
  (i.e., off-device) generally means running the coroutine logic under
  desktop `asyncio` with hardware calls mocked out, exactly as done in
  the code above.

## Cheat sheet

| Primitive | Use for |
|---|---|
| `Event` | One-to-many, "something happened" signaling |
| `Lock` | Protecting a critical section that spans an `await` |
| `Queue(maxsize=n)` | Producer/consumer with automatic backpressure |
| `StreamReader`/`StreamWriter` | Socket protocols; always pair `write()` with `drain()` |
| `wait_for(coro, timeout=...)` | Bounding how long a coroutine can run |
| `finally` around cancellable code | Guaranteed cleanup regardless of cancellation |
| `asyncio.create_task()` + `gather()` | Structuring several independent long-running loops |

## Exercise

Using plain `python3`'s `asyncio`, write a small pipeline: a
`producer()` coroutine that pushes 20 integers onto an
`asyncio.Queue(maxsize=5)` with a small `sleep` between each, and two
`consumer()` coroutines running concurrently that each pull from the
queue and print what they consumed along with their own name, until a
sentinel `None` tells them to stop (the producer should put one
sentinel per consumer). Wrap the whole run in `asyncio.wait_for` with a
generous timeout, and add a `try/finally` in each consumer that prints
"consumer done" on exit — including if cancelled. Run it and confirm
all 20 items are consumed across the two consumers with no item lost
or duplicated.
