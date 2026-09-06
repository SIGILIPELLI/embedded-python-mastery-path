# mpremote & mip — Tooling Deep Dive

Level 1 introduced `mpremote` and `mip` at a survival level. This module is
the deep dive: the full `mpremote` command set (mount, run, filesystem,
scripted REPL), installing packages from GitHub as well as
micropython-lib, and automating a flash-and-test loop so you're not
manually copy-pasting code into a REPL all day. `mpremote` commands are
demonstrated as shell/CLI usage — install it locally with
`pip install mpremote` to run these against real hardware; there's no
Wokwi equivalent for host-side tooling.

## `mpremote` command set

```bash
# List connected boards
mpremote connect list

# Open an interactive REPL (Ctrl-] to exit)
mpremote

# Run a local script on the board without copying it to flash
mpremote run main.py

# Copy files to/from the board's filesystem
mpremote cp main.py :main.py
mpremote cp :data.json ./data.json

# List files on the board
mpremote fs ls
mpremote fs ls :/lib

# Remove a file
mpremote fs rm :old_config.json

# Chain commands with +
mpremote connect /dev/ttyUSB0 + run test_sensor.py + repl
```

The `:` prefix means "on the board's filesystem" — `mpremote cp main.py
:main.py` copies local `main.py` to the board's root as `main.py`;
`mpremote fs ls :/lib` lists the board's `/lib` directory. Chaining with
`+` runs several steps in one invocation, ending in `repl` to drop into an
interactive session right after — handy for "flash it, then watch what it
prints."

## Mounting a local directory

`mpremote mount` makes a directory on your computer show up as the
board's filesystem *live*, no copying:

```bash
mpremote mount ./src repl
```

Edit a `.py` file in `./src` on your laptop, `import` it (or reset the
board) inside the mounted REPL, and it reads the current version straight
off your disk — the fastest edit-test loop available, at the cost of
running noticeably slower than code actually flashed to the board (every
file access crosses the serial link). Good for development; copy files to
flash (`mpremote cp`) before doing anything performance-sensitive or
before disconnecting the board from your computer.

## Automating a flash-and-test loop

A small shell script that copies a whole project directory to a board and
then runs its test entry point:

```bash
#!/bin/sh
# deploy.sh — copy project files, then run tests, then drop to REPL
set -e

mpremote cp main.py :main.py
mpremote cp config.py :config.py
mpremote cp -r lib/ :lib/

mpremote run test_hardware.py
mpremote reset
mpremote repl
```

`mpremote cp -r` copies a directory recursively. `mpremote reset` performs
a soft reset (re-runs `boot.py`/`main.py` from flash) — useful right after
deploying so the board boots into the newly-copied code before you attach
the REPL to watch it.

## Installing packages with `mip`

From micropython-lib, by package name:

```python
import mip
mip.install("umqtt.simple")
mip.install("aioble")
```

From an arbitrary GitHub path — a single file or a whole package
directory:

```python
mip.install("github:org/repo/path/to/module.py")
mip.install("github:org/repo/path/to/package", version="main")
```

`mip` needs the board online (it fetches over HTTP), and it installs into
`/lib` by default. Running `mip.install(...)` from your computer's Python
(not the board) targets a *local* directory instead — useful for
vendoring dependencies into a project you'll `mpremote cp` wholesale:

```bash
mpremote mip install --target ./lib umqtt.simple
```

That's the CLI form of `mip`, run entirely from your machine, writing
files to `./lib` on disk rather than talking to a connected board at all
— pair it with `mpremote cp -r lib/ :lib/` in the deploy script above.

## Tooling gotchas

!!! warning "`mpremote run` doesn't persist the script"
    `mpremote run file.py` executes the file on the board's RAM and
    doesn't save it — reset the board and it's gone. For code that should
    survive a power cycle, `mpremote cp` it to `main.py` (or import it
    from `main.py`).

!!! warning "Serial port contention"
    Only one process can hold the board's serial port at a time. A REPL
    left open in one terminal blocks `mpremote cp` from a second terminal
    with a cryptic "could not enter raw repl" or "port busy" error. Close
    other connections (including the IDE's built-in serial monitor)
    before scripting `mpremote`.

!!! warning "`mount` is slow and fragile across a bad USB cable/hub"
    Because every file read crosses the wire live, a marginal USB
    connection that would tolerate simple `cp` operations can produce
    intermittent `OSError`s under `mount`. If a mounted session gets
    flaky, fall back to `cp`-and-reset for anything beyond quick
    iteration.

!!! warning "`mip.install` overwrites without asking"
    Re-running `mip.install("package")` silently replaces whatever's in
    `/lib/package` — including any local edits you made to a previously
    installed driver. Keep hand-edited vendor code outside `/lib`, or
    rename it, so a future `mip.install` doesn't clobber your changes.

## How It Actually Works

Every `mpremote` command ultimately reduces to two things: putting the board
into "raw REPL" mode over the serial port, and either shipping Python source
as text or streaming a file's bytes through small `os.open`/`write`/`close`
calls executed remotely.

- **`mpremote run` and `mpremote cp` use fundamentally different mechanisms
  even though both "get code onto the board."** `run` drops the board into
  raw REPL, sends your file's *entire source* as one paste-mode block for
  the interpreter to compile and execute immediately, and never touches the
  filesystem — that's why a reset erases all trace of it. `cp` instead sends
  a short remote-execution snippet that opens a file in write-binary mode on
  the board and repeatedly calls `write()` with chunks of your file's bytes
  read from your computer, base64- or raw-encoded over the raw-REPL
  protocol — a genuine file transfer, ending with the board's own
  filesystem holding the bytes on flash, independent of the running
  program's state.
- **`mount` avoids all copying by intercepting `import`/`open()` at the
  filesystem level and satisfying reads over the serial link on demand.**
  `mpremote` implements a small custom "remote filesystem" protocol: when
  the board's VM tries to `import mymodule`, it's actually asking a
  `VfsRaw`-style driver mpremote installed, which round-trips a request
  over serial back to your computer, reads the real file from disk there,
  and returns its bytes. That per-access round trip over a ~1 Mbit serial
  link is exactly why mount is slow compared to code already sitting on
  flash — every `import`, every `open().read()` pays a full serial
  round-trip that a locally-copied file never has to.
- **Serial port contention is a hardware fact, not an mpremote limitation.**
  A USB-to-serial adapter (or the ESP32's built-in USB-UART bridge) exposes
  exactly one open file descriptor to the OS at a time; the tty driver
  itself refuses a second process's `open()` on the same device node while
  one is already held. `mpremote`'s "raw repl" handshake specifically needs
  exclusive control of that byte stream to reliably frame commands and
  responses without another process's bytes interleaving — which is why
  even a passive terminal window (not actively typing) still blocks a
  second `mpremote` process.
- **`mip.install`'s silent overwrite follows from it fetching and writing
  files exactly the way `mpremote cp` does — it has no concept of "existing
  file, ask first."** Under the hood, `mip` fetches package metadata (a
  small JSON manifest) over HTTP from micropython-lib or GitHub's raw
  content API, then does the same open-write-close file transfer as any
  other filesystem write to `/lib`, one file at a time — there's no diffing
  or merge logic anywhere in that path, only "write these bytes to this
  path," which is why hand-edited vendor code sitting at the same path
  disappears without warning on a re-install.

## Cheat sheet

| Command | Purpose |
|---|---|
| `mpremote connect list` | List connected boards/ports |
| `mpremote` | Open interactive REPL |
| `mpremote run file.py` | Execute a local file on-board, in RAM only |
| `mpremote cp file.py :file.py` | Copy a file to the board's filesystem |
| `mpremote cp -r dir/ :dir/` | Copy a directory recursively |
| `mpremote fs ls [:path]` | List files on the board |
| `mpremote fs rm :file` | Delete a file on the board |
| `mpremote mount ./dir` | Live-mount a local dir as the board's filesystem |
| `mpremote reset` | Soft reset — re-runs flash code |
| `a + b + c` (chained) | Run several `mpremote` steps in one call |
| `mip.install("name")` (on-board) | Install from micropython-lib/GitHub to `/lib` |
| `mpremote mip install --target ./lib name` | Install to a local dir from your computer |

## Exercise

Write a `deploy.sh` shell script (as shown above, adapted) that: copies a
project's `main.py`, `config.py`, and a `lib/` directory to a connected
board; runs a `test_hardware.py` script via `mpremote run` (writing a
minimal stub that just prints `"self-test OK"` is fine); performs a soft
reset; then drops into the REPL so you can watch boot output. Add a second
script, `vendor.sh`, that uses `mpremote mip install --target ./lib` to
fetch `umqtt.simple` and `umqtt.robust` into a local `lib/` directory
without touching a connected board, so `deploy.sh` can then copy them over
in one shot. Note in a comment which of the two scripts needs a board
attached and which doesn't.
