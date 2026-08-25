# The Inline Assembler

Below viper's typed Python and above a full C module sits one more
tool: `@micropython.asm_thumb`, MicroPython's inline assembler for ARM
Thumb (the instruction set on Cortex-M cores like those inside the
ESP32-S3's coprocessors and, more commonly, STM32/SAMD/nRF boards that
run MicroPython — the plain ESP32's Xtensa core and the RP2040 use
different assemblers, `asm_xtensa` and none respectively, since RP2040
work of this kind is done in PIO instead). This is the lowest-level
tool in MicroPython short of a full C module, and the least portable —
reviewed here against the documented `asm_thumb` API and ARM Thumb
instruction reference, with nothing to execute locally.

## Why reach for it at all

Viper compiles Python-like source to machine code automatically; the
inline assembler skips that translation and lets you write the machine
code yourself. The honest reasons to do this are narrow:

- An instruction sequence viper's compiler won't emit (a specific
  hardware register access, a saturating arithmetic instruction, a
  precise cycle-counted delay loop)
- Wrapping a tiny, extremely hot primitive where even native/viper
  overhead is too much
- Interfacing with a calling convention or register layout a specific
  peripheral or bootloader expects exactly

For nearly everything else, viper reaches 90% of the benefit with a
small fraction of the risk and effort — the inline assembler is a
last-resort tool, not a default upgrade path from viper.

## Anatomy of an `asm_thumb` function

```python
import micropython

@micropython.asm_thumb
def add_one(r0):
    add(r0, r0, 1)
```

The function body is not Python being compiled — it's ARM Thumb
assembly, one mnemonic per line, written using MicroPython's Python-ish
syntax for it (`add(dst, src, imm)` rather than the raw `ADD R0, R0, #1`
textual assembly a standalone assembler would take). Called from Python
exactly like any other function:

```python
print(add_one(41))   # 42
```

## Registers, arguments, and return values

`asm_thumb` functions can take up to **three** arguments (some ports
support up to four), always passed and returned via specific
registers, per the ARM procedure call standard MicroPython follows:

- Arguments arrive in `r0`, `r1`, `r2` (in that order)
- The return value is whatever is in `r0` when the function returns
- `r0`-`r3` are freely usable as scratch registers within the function
- Registers beyond that (`r4` and up) are typically reserved by the
  calling convention and must be saved/restored (pushed/popped) if
  used, or corruption of caller state results

```python
@micropython.asm_thumb
def multiply_add(r0, r1, r2):
    mul(r0, r1)        # r0 = r0 * r1
    add(r0, r0, r2)    # r0 = r0 + r2
    # r0 now holds the return value implicitly
```

```python
print(multiply_add(3, 4, 5))   # 3*4 + 5 = 17
```

Unlike viper's `int`/`ptr8` abstractions, there is no type system here
at all — every value is just whatever bit pattern is in a register, and
it's entirely the programmer's job to know whether that pattern means a
signed integer, an unsigned integer, or a raw pointer at any given
instruction.

## Reading the generated code

MicroPython can disassemble what an `asm_thumb` function actually
compiled to, which is essential for verifying an instruction sequence
did what was intended — a wrong mnemonic or operand order might still
assemble to *something*, just not the intended something:

```python
import micropython

@micropython.asm_thumb
def add_one(r0):
    add(r0, r0, 1)

print(add_one)   # <generator/function repr varies by port>
```

On ports/builds with disassembly support enabled, `micropython.opt_level()`
and build-time flags control how much is emitted for inspection; the
practical workflow on real hardware is: write the small routine, call
it with known inputs, and check outputs against hand-computed expected
values, since the inline assembler offers none of Python's usual
safety net (no exceptions on bad memory access, no type errors) to
catch a mistake for you.

## When hand-written assembly is (and isn't) worth it

Worth it:
- A single, very small, extremely hot primitive (a handful of
  instructions) called from a much larger, otherwise-fine viper or
  native routine
- Directly manipulating a status/control register in a way the higher
  layers don't expose a clean way to do

Not worth it, in nearly every real case:
- Anything longer than a few instructions — bugs at this level are
  invisible to Python's normal debugging tools (no traceback points
  into hand-written assembly meaningfully) and cost disproportionate
  debugging time
- Anything viper already reaches acceptable performance for — always
  benchmark native → viper → asm_thumb in that order and stop as soon
  as one meets the requirement, per the profiling module's
  methodology
- Anything that needs to be ported across architectures later — Thumb
  assembly is specific to ARM Cortex-M; it does not run on the
  ESP32's Xtensa core or the RP2040's PIO at all, so this is the least
  portable tool in the whole optimization ladder

## Cheat sheet

| Aspect | Detail |
|---|---|
| Decorator | `@micropython.asm_thumb` (ARM Cortex-M targets only) |
| Arguments | Up to 3 (some ports 4), passed in `r0`, `r1`, `r2` |
| Return value | Whatever is in `r0` at function exit |
| Scratch registers | `r0`-`r3` freely usable; `r4`+ must be saved/restored if touched |
| Type system | None — every value is a raw register bit pattern |
| Portability | ARM Cortex-M only; not Xtensa (ESP32), not RP2040 (use PIO there) |
| When to use | Last resort, after native and viper both fall short |

## Exercise

This is architecture-specific assembly with nothing to execute here, so
treat it as a design/documentation exercise. Write, as a fenced code
block, an `@micropython.asm_thumb` function `clamp_byte(r0)` that takes
one integer argument and returns it clamped to the 0-255 range (if
negative, return 0; if greater than 255, return 255; otherwise return
it unchanged) using Thumb compare-and-branch style pseudo-instructions
in MicroPython's `asm_thumb` syntax (`cmp`, conditional branches like
`bge`/`ble`, `mov`, and labels). Beneath the code, write a short
explanation of why this exact function is a good candidate for
`@micropython.native` or `@micropython.viper` instead, and under what
specific circumstance dropping all the way to hand-written assembly
would actually be justified for something this simple.
