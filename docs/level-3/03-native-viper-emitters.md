# Native & Viper Code Emitters

MicroPython normally compiles your source to bytecode, interpreted at
runtime — that interpretation overhead is where most of a hot loop's
time goes. Two decorators let you trade portability and Python
semantics for speed: `@micropython.native` compiles a function straight
to machine code instead of bytecode, and `@micropython.viper` goes
further with a stricter, C-like type system. This module is reviewed
against MicroPython's documented emitter behavior — there's no
hardware here to benchmark against, so every timing figure is the
range MicroPython's own docs and community benchmarks report, called
out as such.

## `@micropython.native` — same semantics, compiled

```python
import micropython

@micropython.native
def sum_squares(n):
    total = 0
    for i in range(n):
        total += i * i
    return total
```

`native` keeps full Python semantics — the same object model, the same
exception behavior, `int` still transparently promotes to bignum on
overflow. All it changes is the *emitter*: instead of bytecode
interpreted by the VM, the function body is compiled directly to native
machine code for the target architecture at import/compile time. The
official MicroPython documentation puts the typical speedup at roughly
2x, because you still pay for boxed objects and dynamic dispatch — you
just skip the bytecode dispatch loop itself.

`native` functions can call regular Python functions and be called by
them freely; there's no calling-convention boundary to worry about,
which makes it the easy first thing to try on a hot function before
reaching for `viper`.

## `@micropython.viper` — a different type system

```python
import micropython

@micropython.viper
def sum_squares_viper(n: int) -> int:
    total: int = 0
    i: int = 0
    while i < n:
        total += i * i
        i += 1
    return total
```

Viper is where the real speedups live — commonly cited as 10x or more
over plain bytecode for numeric loops — but it comes with a genuinely
different type system, not just annotations for documentation's sake:

- `int` in viper is a **machine word**, not Python's arbitrary-precision
  `int`. It silently wraps on overflow instead of promoting to bignum.
  A viper `int` that overflows does not raise and does not become
  correct — it wraps like a C `int`, which is a real correctness trap
  if you assume Python's normal big-int behavior carries over.
- `uint` is the unsigned counterpart.
- `ptr8`, `ptr16`, `ptr32` are typed pointers for direct, unchecked
  memory access — think of them as a `bytearray`/buffer viewed as raw
  memory, with **no bounds checking whatsoever**.
- Untyped local variables and any object types other than the above
  fall back to being treated as generic Python objects (boxed,
  interpreted normally) — mixing typed and untyped code in one viper
  function is legal but only the typed parts get the speedup.

## Pointer types — power without a safety net

```python
import micropython

@micropython.viper
def fill_buffer(buf, value: int) -> int:
    p = ptr8(buf)          # view an existing buffer as raw bytes
    n: int = int(len(buf))
    i: int = 0
    while i < n:
        p[i] = value
        i += 1
    return n
```

`ptr8(buf)` doesn't copy `buf` — it's a raw, bounds-unchecked view over
the same memory. `p[i] = value` for `i >= len(buf)` does not raise an
`IndexError` the way `bytearray.__setitem__` would; it writes past the
buffer into whatever memory is next, which on a microcontroller with no
memory protection unit for regular RAM can silently corrupt unrelated
data or crash the interpreter in a way that's hard to trace back to
this line. There is no way to make `ptr8`/`ptr16`/`ptr32` access
bounds-checked — the entire point is skipping that check. Every viper
function using raw pointers needs its bounds reasoned about by the
programmer, by hand, every time.

`ptr16` and `ptr32` index in units of their width, not bytes:

```python
@micropython.viper
def read_word(buf, idx: int) -> int:
    p = ptr32(buf)
    return int(p[idx])   # reads bytes [idx*4 : idx*4+4], not [idx:idx+4]
```

Forgetting that `ptr32` indexing is word-granular, not byte-granular,
is a common source of "reads the wrong data" bugs when porting code
between pointer widths.

## Typed integers and function boundaries

Return types and parameter types matter more in viper than in `native`:
declaring `-> int` means the compiler emits code assuming a machine
word comes back, and calling a viper function that returns `int` from
regular Python code gets a normal boxed Python `int` back at the
boundary — the conversion happens automatically, but only at the
function boundary, not inside the function body.

```python
@micropython.viper
def clamp(x: int, lo: int, hi: int) -> int:
    if x < lo:
        return lo
    if x > hi:
        return hi
    return x

# Called from plain Python — looks completely normal at the call site
print(clamp(500, 0, 255))   # 255, viper's int/return conversion is transparent here
```

## Benchmarking the speedup — methodology

Because these decorators only matter when a function is genuinely hot,
always benchmark with `time.ticks_us()`/`ticks_diff` (see the
profiling module) comparing three versions of the same function —
plain, `native`, `viper` — over enough iterations to average out noise,
on the actual target board. Desktop CPython has no equivalent of these
decorators (they're MicroPython-specific compiler features), so there
is nothing meaningful to run here to reproduce the ratios; the
comparison only exists on-device.

## How It Actually Works

The three emitters (bytecode, native, viper) are three different *code
generators* sharing one compiler front-end — the difference between them is
entirely about what the compiler's back-end targets and what type
information it's given to work with.

- **The bytecode emitter compiles to a stack-based virtual instruction
  set** — each source-level operation like `total += i * i` becomes several
  opcodes (`LOAD_FAST`, `LOAD_FAST`, `BINARY_OP MULTIPLY`, `LOAD_FAST`,
  `BINARY_OP ADD`, `STORE_FAST`) that the VM's dispatch loop decodes and
  executes one at a time, each requiring a jump through a dispatch table
  (or a giant switch statement) plus boxed-object arithmetic that checks
  operand types at runtime — every `+` genuinely asks "what kind of thing
  is this?" before deciding how to add it. That per-opcode dispatch and
  per-operation type check is the interpretation overhead this whole module
  exists to escape.
- **`@micropython.native` compiles the *same* bytecode-level operations
  directly to the target's native instruction set (ARM Thumb, Xtensa,
  RISC-V depending on the port) at import time, but keeps every object
  boxed and every operation's semantics identical.** A native-compiled
  `total += i * i` still calls the same boxed-integer-arithmetic runtime
  functions bytecode would have called — it just reaches those calls
  through direct machine-code branches instead of a dispatch-loop-and-opcode
  fetch, which is why the speedup is real but modest (~2x): you've removed
  the dispatch overhead, not the boxing/dynamic-typing overhead underneath.
- **`@micropython.viper` is a genuinely different compiler mode that emits
  code operating on raw machine words instead of boxed `mp_obj_t` values.**
  A viper `int` is compiled to occupy an actual CPU register or raw memory
  slot holding a native 32-bit value — no boxing, no dynamic type check, no
  runtime dispatch to a generic "add" function; `total += i * i` becomes
  literal ADD/MUL machine instructions on registers, exactly like C would
  generate. This is precisely why it's 10x+ and precisely why it silently
  wraps on overflow: a genuine machine-word `int` has no bignum-promotion
  path at all, because Python's arbitrary-precision integer semantics
  simply were not compiled in for that variable.
- **`ptr8`/`ptr16`/`ptr32` compile to a bare pointer dereference with no
  runtime bounds check inserted, because inserting one is exactly the
  Python-level safety machinery viper exists to strip away.** A regular
  `bytearray[i]` goes through `mp_obj_t` dispatch to a subscript handler
  that explicitly compares `i` against the buffer's stored length before
  reading — that comparison is Python-level object-model code viper's
  typed pointers never generate, which is also why `ptr32` indexing is
  word-granular: the compiler emits `base_address + i * 4`, the same raw
  address arithmetic a C compiler would produce for `int32_t *p; p[i]`,
  with no notion of "byte offset" surviving into the generated code at all.

## Cheat sheet

| Emitter | Typical speedup | Type system | Bounds-checked | Risk |
|---|---|---|---|---|
| (default, bytecode) | 1x baseline | Full Python | Yes | None beyond normal Python bugs |
| `@micropython.native` | ~2x | Full Python | Yes | Low — same semantics, just compiled |
| `@micropython.viper` | ~10x+ | Machine `int`/`uint`, typed pointers | No, for `ptr*` access | High — overflow wraps silently, `ptr*` has no bounds checks |

## Exercise

You cannot run `@micropython.viper` code outside MicroPython, so this
exercise is a design/review task. Write out, in a comment block, a
viper version of a function `checksum(buf) -> int` that XORs every byte
of a `bytearray` together using `ptr8`. Then write, in prose comments
beneath it, three specific bugs a programmer could introduce in this
function that would compile fine but behave wrong or unsafely on
target hardware: one involving the `int` overflow/wraparound behavior,
one involving an off-by-one on the pointer bounds, and one involving
mixing an untyped Python object into the typed loop without realizing
it falls back to boxed/interpreted handling.
