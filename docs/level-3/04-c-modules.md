# Writing C Modules for MicroPython

Viper gets you 10x; sometimes you need the full generality and speed of
C — a proper DSP routine, a vendor SDK you must link against, or a data
structure the Python object model isn't a good fit for. MicroPython's
user C module system lets you write C, compile it into the firmware,
and call it from Python as if it were a normal module. This is a
build-system-and-C-API topic with no runtime here to execute it against
— everything below is reviewed for accuracy against MicroPython's
documented C module API (`py/obj.h`, `py/runtime.h`, the
`micropython.mk`/`micropython.cmake` build hooks) rather than compiled
and run.

## The user C module layout

A user C module is a directory with a `micropython.mk` (for the
Make-based ports) or `micropython.cmake` (for CMake-based ports like
the modern ESP32 and RP2040 builds), plus your C source:

```
my_module/
├── micropython.cmake
├── micropython.mk
└── modmymodule.c
```

`micropython.cmake` registers the module with the build:

```cmake
add_library(usermod_mymodule INTERFACE)

target_sources(usermod_mymodule INTERFACE
    ${CMAKE_CURRENT_LIST_DIR}/modmymodule.c
)

target_include_directories(usermod_mymodule INTERFACE
    ${CMAKE_CURRENT_LIST_DIR}
)

target_link_libraries(usermod INTERFACE usermod_mymodule)
```

The firmware build is then pointed at it via
`USER_C_MODULES=/path/to/my_module/micropython.cmake` (or the `.mk`
equivalent) when invoking the port's `make`/`cmake` build — this is a
firmware **build-time** decision; a module can't be added to a running
device the way a `.py` file can be copied over.

## Defining a function callable from Python

```c
#include "py/runtime.h"

// C-level implementation
STATIC mp_obj_t mymodule_double(mp_obj_t x_obj) {
    mp_int_t x = mp_obj_get_int(x_obj);
    return mp_obj_new_int(x * 2);
}
// MP_DEFINE_CONST_FUN_OBJ_1 wraps a 1-argument C function as a
// callable MicroPython object
STATIC MP_DEFINE_CONST_FUN_OBJ_1(mymodule_double_obj, mymodule_double);

STATIC const mp_rom_map_elem_t mymodule_globals_table[] = {
    { MP_ROM_QSTR(MP_QSTR___name__), MP_ROM_QSTR(MP_QSTR_mymodule) },
    { MP_ROM_QSTR(MP_QSTR_double), MP_ROM_PTR(&mymodule_double_obj) },
};
STATIC MP_DEFINE_CONST_DICT(mymodule_globals, mymodule_globals_table);

const mp_obj_module_t mymodule_user_cmodule = {
    .base = { &mp_type_module },
    .globals = (mp_obj_dict_t*)&mymodule_globals,
};

MP_REGISTER_MODULE(MP_QSTR_mymodule, mymodule_user_cmodule);
```

Called from Python exactly like any built-in module:

```python
import mymodule
print(mymodule.double(21))   # 42
```

`mp_obj_get_int()` and `mp_obj_new_int()` are the boundary functions —
every value crossing from Python into C or back goes through an
`mp_obj_t`-to-native (or native-to-`mp_obj_t`) conversion. Skipping
this and treating an `mp_obj_t` as if it were already a C integer is a
common beginner mistake that compiles (it's just a pointer-sized
value) and crashes or corrupts memory at runtime.

## Argument count variants and error handling

Functions take a fixed number of `MP_DEFINE_CONST_FUN_OBJ_N` variants
(`_0` through `_3` for fixed arg counts, `_VAR` and `_KW` for
variable/keyword arguments). Raising a Python exception from C uses the
`mp_raise_*` helpers rather than C's own error mechanisms:

```c
STATIC mp_obj_t mymodule_safe_divide(mp_obj_t a_obj, mp_obj_t b_obj) {
    mp_int_t a = mp_obj_get_int(a_obj);
    mp_int_t b = mp_obj_get_int(b_obj);
    if (b == 0) {
        mp_raise_ValueError(MP_ERROR_TEXT("division by zero"));
    }
    return mp_obj_new_int(a / b);
}
STATIC MP_DEFINE_CONST_FUN_OBJ_2(mymodule_safe_divide_obj, mymodule_safe_divide);
```

`mp_raise_ValueError` unwinds via `longjmp` back into the interpreter's
exception handling — it does not return, so any C-allocated resources
before that call need to already be safely released or not yet
acquired, since there's no automatic unwind/cleanup path like C++
destructors or Python's `finally`.

## Defining a class callable from Python

C modules can expose full types, not just functions — useful for
wrapping a stateful driver or hardware peripheral:

```c
typedef struct _counter_obj_t {
    mp_obj_base_t base;
    mp_int_t value;
} counter_obj_t;

STATIC mp_obj_t counter_make_new(const mp_obj_type_t *type, size_t n_args,
                                  size_t n_kw, const mp_obj_t *args) {
    counter_obj_t *self = mp_obj_malloc(counter_obj_t, type);
    self->value = 0;
    return MP_OBJ_FROM_PTR(self);
}

STATIC mp_obj_t counter_increment(mp_obj_t self_in) {
    counter_obj_t *self = MP_OBJ_TO_PTR(self_in);
    self->value += 1;
    return mp_obj_new_int(self->value);
}
STATIC MP_DEFINE_CONST_FUN_OBJ_1(counter_increment_obj, counter_increment);

STATIC const mp_rom_map_elem_t counter_locals_dict_table[] = {
    { MP_ROM_QSTR(MP_QSTR_increment), MP_ROM_PTR(&counter_increment_obj) },
};
STATIC MP_DEFINE_CONST_DICT(counter_locals_dict, counter_locals_dict_table);

MP_DEFINE_CONST_OBJ_TYPE(
    counter_type, MP_QSTR_Counter, MP_TYPE_FLAG_NONE,
    make_new, counter_make_new,
    locals_dict, &counter_locals_dict
);
```

`mp_obj_malloc` allocates the object **on MicroPython's own
GC-managed heap** — this is the detail people miss coming from plain C:
a C module's objects still participate in Python's garbage collection,
so they must be allocated through MicroPython's allocator, not `malloc`
directly, or the collector will never see them and either free memory
still in use (if you `malloc` and register a pointer to it as if it
were GC-owned) or leak it (if you `malloc` outside GC entirely and the
Python side loses its last reference).

## When C beats viper

- Viper's `int`/`ptr*` types cover integer arithmetic and raw memory
  well, but floating point, complex control flow, calling into a vendor
  SDK, or needing genuine C structs and multiple translation units are
  all out of viper's scope — that's the C module's territory.
- C modules can wrap any existing C/C++ library (a vendor's DSP or
  crypto library, for instance) with a thin binding layer, which viper
  cannot do at all — viper only compiles the Python-level source you
  write.
- The cost is a full firmware rebuild-and-reflash per change, versus
  editing a `.py` file and re-running it — a genuinely slower
  development loop, which is a real reason to reach for viper first and
  only drop to C when viper's ceiling is actually hit.

## Cheat sheet

| Need | Reach for |
|---|---|
| Fast integer/pointer loop, still Python source | `@micropython.viper` |
| Call an existing C/C++ library | User C module |
| A stateful type with methods, backed by a C struct | User C module with `make_new`/`locals_dict` |
| Raising errors from native code | `mp_raise_ValueError`/`mp_raise_TypeError`/etc., not `abort()` or C `errno` |
| Allocating memory the GC must track | `mp_obj_malloc`, never bare `malloc` for objects handed to Python |

## Exercise

You have no C toolchain targeting MicroPython here, so this is a
design exercise. Write the full C source (as a fenced code block, not
executed) for a user C module function `crc8(data: bytes) -> int` that
computes a simple XOR-based checksum over an `mp_obj_t` bytes argument.
Your code must: extract the underlying buffer safely via
`mp_get_buffer()` (look up and describe its `mp_buffer_info_t` usage in
a comment, since it wasn't shown above), loop over the bytes computing
the XOR, and return the result via `mp_obj_new_int()`. Add a comment
explaining what happens if the caller passes a non-buffer-protocol
object like an `int` — which `mp_raise_*` call should fire and why.
