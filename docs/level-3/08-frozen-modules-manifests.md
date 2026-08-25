# Frozen Modules & Custom Manifests

Every `.py` file copied to a device's filesystem has to be parsed and
compiled to bytecode at import time, and it lives in the same flash
filesystem competing for space with everything else you store there.
Freezing bakes Python modules into the firmware image itself, as
precompiled bytecode embedded alongside the interpreter, controlled by
a `manifest.py` at build time. This is a firmware build-system topic —
reviewed against MicroPython's documented manifest/freeze mechanism,
with no firmware build performed here.

## What freezing actually changes

A normal `.py` file on the filesystem: read from flash as text at
import time, tokenized, parsed, and compiled to bytecode, every single
boot. A frozen module: **already compiled to bytecode** and linked
directly into the firmware binary — importing it is just pointing at
that already-compiled bytecode in flash, skipping tokenize/parse/compile
entirely.

The practical effects:

- **Faster imports** — no parse/compile step at import time, which
  matters most for larger modules imported on every boot
- **RAM savings** — a frozen module's bytecode lives in flash and is
  executed from there (MicroPython supports execute-in-place for frozen
  bytecode on most ports); a filesystem `.py` module's bytecode, once
  compiled, occupies **heap** RAM for the life of the import
- **No longer independently editable** — changing a frozen module
  means editing the source and reflashing the entire firmware, not
  just copying a new file over — this trade-off is central to how
  frozen modules get used in practice (see below)

## The manifest file

`manifest.py` is a small Python DSL, executed by the build system (not
by MicroPython itself at runtime), that declares what gets frozen:

```python
# manifest.py
freeze("$(PORT_DIR)/modules")          # freeze every .py in this directory
freeze("my_app", ("main_logic.py",))   # freeze a specific file from "my_app"
include("$(MPY_DIR)/extmod/asyncio/manifest.py")  # pull in another manifest
```

`freeze(path, files=None)` walks `path` (or just the given `files`
within it) and compiles each into the firmware image. `include(...)`
composes manifests — this is how a board's default manifest pulls in
the standard library pieces (`asyncio`, `urequests`, etc.) that ship
frozen by default on most ports, without every board's manifest
duplicating that list.

The build is pointed at a custom manifest with a build-time variable,
e.g. `make FROZEN_MANIFEST=/path/to/manifest.py` (exact variable name
varies slightly by port) — like C modules, this is a decision made at
firmware-build time, not something a running device can change.

## Freezing as bytecode, not source

An important distinction: `freeze()` compiles to `.mpy` bytecode and
embeds that — the original `.py` source text is not what ships in the
firmware. This has two consequences worth knowing:

1. Frozen modules are **not directly readable** by inspecting the
   firmware binary as text — they're compiled bytecode, same as a
   `.mpy` cross-compiled with `mpy-cross` for filesystem deployment.
2. Bytecode compatibility is tied to the MicroPython version/build —
   frozen modules are compiled against the exact interpreter they ship
   with, so this isn't a portable artifact you move between firmware
   versions independently (unlike a `.py` file, which is source and
   recompiles fresh on any compatible MicroPython version).

## Booting straight into your application

Freezing `main.py` (or whatever your entry point is) into the firmware,
combined with structuring the board so the frozen module runs at boot,
is how products ship a MicroPython application that behaves like
dedicated firmware rather than "MicroPython plus some files someone
could delete or corrupt":

```python
# manifest.py
freeze("app", ("main.py", "config.py", "sensors.py"))
```

Freezing the application logic means a factory-reset or filesystem
corruption event can't remove the app itself — the interpreter's
default boot sequence looks for `main.py` and, when nothing on the
filesystem overrides it, MicroPython on most ports will still run a
frozen module by that name if the filesystem copy is absent. This
combination — frozen application logic plus a filesystem reserved for
config and data — is the standard shape of production MicroPython
firmware (the production architecture module goes further into this
layering).

## Trade-offs, honestly

| | Filesystem `.py` | Frozen module |
|---|---|---|
| Update mechanism | Copy a new file (mpremote, OTA over the filesystem, etc.) | Reflash the whole firmware image |
| Import speed | Parse + compile every boot | No parse/compile — already bytecode |
| RAM for bytecode | Heap, for the life of the import | Flash, execute-in-place on most ports |
| Editability post-deploy | Trivial | Requires a full firmware rebuild |
| Good fit for | Application logic that changes often, user config, anything updated independently | Stable core logic, libraries, anything that should survive a factory reset |

Most real products use both: a frozen, stable core (drivers, protocol
handling, safety-critical logic) plus a filesystem-based, more mutable
layer (configuration, feature flags, whatever's expected to change
between deployments without a full firmware rebuild).

## A subtle trap — frozen modules shadow filesystem modules

Import resolution order on most ports checks frozen modules and the
filesystem in a specific order (frozen modules are typically resolved
via `sys.path`-like precedence that varies slightly by port version).
The practical trap: if a module is frozen **and** a same-named `.py` is
also copied to the filesystem expecting it to "override" the frozen
one for a quick fix, whether that actually happens depends on the
specific port's import order — this isn't something to assume works
without checking the target port's documented precedence, and it is a
frequent source of "I edited the file but nothing changed" confusion
during development.

## Cheat sheet

| Concept | Detail |
|---|---|
| `manifest.py` | Build-time DSL controlling what gets frozen |
| `freeze(path, files=None)` | Compile and embed `.py` files as bytecode |
| `include(other_manifest)` | Compose manifests (how stdlib pieces get pulled in) |
| Frozen bytecode location | Flash, executed in place on most ports — not heap |
| Filesystem bytecode location | Heap, for the life of the import |
| Update path | Reflash firmware (frozen) vs. copy a file (filesystem) |
| Import precedence | Port-specific — verify, don't assume, when both exist |

## Exercise

This is a build-system topic with no code to execute locally. Write out
a `manifest.py`, as a fenced code block, that: freezes an entire
directory called `drivers/` containing multiple sensor driver modules,
freezes a single specific file `app/main.py` by name, and includes one
hypothetical other manifest at `$(PORT_DIR)/variants/manifest_base.py`.
Beneath it, write a short paragraph describing the concrete boot-time
and RAM difference a user would observe between a board built with
this manifest and the same application shipped entirely as loose `.py`
files copied via `mpremote`, referencing both the import-time
parse/compile cost and where the resulting bytecode lives in each case.
