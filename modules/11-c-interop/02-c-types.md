# 11.2 — C Types

An `extern fn` declaration is a hand-written ABI contract. Nothing checks it
against the real C header, so every type in it has to be right. This lesson is
the mapping table, measured on this machine.

## The `c_*` types

Zig's `i32`/`u64` have fixed widths. C's `int`/`long` do not — they depend on
the target. The `c_*` types are aliases that track whatever the target's C ABI
says, so use them in any signature that faces C.

```zig
const std = @import("std");

pub fn main() void {
    inline for (.{
        .{ "c_char", c_char },      .{ "c_short", c_short },
        .{ "c_int", c_int },        .{ "c_uint", c_uint },
        .{ "c_long", c_long },      .{ "c_ulong", c_ulong },
        .{ "c_longlong", c_longlong }, .{ "c_longdouble", c_longdouble },
        .{ "usize", usize },        .{ "isize", isize },
    }) |e| {
        std.debug.print("{s:<13} bits={d:<3} size={d} align={d} signed={}\n", .{
            e[0], @bitSizeOf(e[1]), @sizeOf(e[1]), @alignOf(e[1]),
            @typeInfo(e[1]) == .int and @typeInfo(e[1]).int.signedness == .signed,
        });
    }
}
```

```sh
zig run types.zig
```

```
c_char        bits=8   size=1 align=1 signed=true
c_short       bits=16  size=2 align=2 signed=true
c_int         bits=32  size=4 align=4 signed=true
c_uint        bits=32  size=4 align=4 signed=false
c_long        bits=64  size=8 align=8 signed=true
c_ulong       bits=64  size=8 align=8 signed=false
c_longlong    bits=64  size=8 align=8 signed=true
c_longdouble  bits=64  size=8 align=8 signed=false
usize         bits=64  size=8 align=8 signed=false
isize         bits=64  size=8 align=8 signed=true
```

`@bitSizeOf(c_int)` is **32** on this aarch64-macos target. Three readings
of that table matter:

- **`c_long` is 64 bits here, 32 bits on Windows.** This is the single most
  common cross-platform interop bug. `i64` would be wrong on win64; `c_long` is
  right everywhere.
- **`c_char` is signed** on this target, unsigned on ARM Linux. If you need a
  byte, write `u8`; use `c_char` only when the C signature literally says `char`
  and the signedness matters.
- **`c_longdouble` is 64 bits** here — the same as `double`. On x86_64 Linux it
  is 80-bit extended. Never assume.

| C | Zig | Not |
|---|---|---|
| `char` | `c_char` (or `u8` for bytes) | `i8` |
| `signed char` / `unsigned char` | `i8` / `u8` | |
| `short` / `unsigned short` | `c_short` / `c_ushort` | `i16` |
| `int` / `unsigned int` | `c_int` / `c_uint` | `i32` |
| `long` / `unsigned long` | `c_long` / `c_ulong` | **`i64`** — wrong on Windows |
| `long long` | `c_longlong` / `c_ulonglong` | |
| `float` / `double` | `f32` / `f64` | |
| `long double` | `c_longdouble` | `f80`/`f128` |
| `size_t` | `usize` | |
| `ssize_t` / `ptrdiff_t` | `isize` | |
| `int32_t`, `uint64_t`, … | `i32`, `u64`, … | fixed-width C types map directly |
| `bool` (C99 `_Bool`) | `bool` | `u8` |
| `void` return | `void` | |
| `void *` | `?*anyopaque` | |
| `const void *` | `?*const anyopaque` | |

There is no `c_size_t` — `size_t` is `usize` by definition in Zig. Likewise
`c_bool` does not exist; use `bool`, which is ABI-compatible with `_Bool`.

## `extern struct` — the layout that matters

Zig **reorders struct fields**. This is the behavior that makes `extern` not
optional at an ABI boundary:

```zig
const std = @import("std");

const ZigS = struct { a: u8, b: u32, c: u8 };
const CS = extern struct { a: u8, b: u32, c: u8 };
const PS = packed struct { a: u8, b: u32, c: u8 };

fn dump(comptime T: type, comptime name: []const u8) void {
    std.debug.print("{s:<14} size={d} align={d}  a@{d} b@{d} c@{d}\n", .{
        name, @sizeOf(T), @alignOf(T),
        @offsetOf(T, "a"), @offsetOf(T, "b"), @offsetOf(T, "c"),
    });
}

pub fn main() void {
    dump(ZigS, "struct");
    dump(CS, "extern struct");
    std.debug.print("{s:<14} size={d} align={d}  bits={d}\n", .{
        "packed struct", @sizeOf(PS), @alignOf(PS), @bitSizeOf(PS),
    });
}
```

```sh
zig run layout.zig
```

```
struct         size=8 align=4  a@4 b@0 c@5
extern struct  size=12 align=4  a@0 b@4 c@8
packed struct  size=8 align=8  bits=48
```

Read the offsets:

- **Plain `struct`: `b` is at offset 0, `a` at 4.** Zig put the `u32` first to
  avoid padding, fitting all three fields into 8 bytes. The declaration order is
  not the memory order.
- **`extern struct`: `a@0 b@4 c@8`, 12 bytes.** C's order, C's padding — three
  bytes wasted after `c`. Exactly what a C compiler produces.
- **`packed struct`: 48 bits, no padding at all.** This is for bit-level layout
  (wire formats, hardware registers), *not* for the C ABI. C's `struct` is never
  packed unless the header says `__attribute__((packed))`.

Pass a non-`extern` struct to C and Zig refuses to compile it:

```zig
const std = @import("std");
const P = struct { x: c_int, y: c_int };
extern fn takes(p: P) c_int;
pub fn main() void {
    std.debug.print("{d}\n", .{takes(.{ .x = 1, .y = 2 })});
}
```

```sh
zig run -lc bad.zig
```

```
bad.zig:3:17: error: parameter of type 'bad.P' not allowed in function with calling convention 'aarch64_aapcs_darwin'
extern fn takes(p: P) c_int;
                ^~~~
bad.zig:3:17: note: struct with automatic layout has no guaranteed in-memory representation
bad.zig:2:11: note: struct declared here
const P = struct { x: c_int, y: c_int };
          ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

"no guaranteed in-memory representation" — a compile error, which is the good
case. The same struct behind a `*P` pointer would compile and silently read the
wrong fields.

| Layout | Field order | Padding | Use for |
|---|---|---|---|
| `struct` | **compiler's choice** | minimized | everything internal to Zig |
| `extern struct` | declaration order | C rules | **every C boundary** |
| `packed struct` | declaration order | **none**, bit-packed | wire formats, registers, bitfields |

## Proving the layout matches, from both sides

Don't trust the table — measure it. This program asks C for `sizeof` and
`offsetof` and compares with Zig's `@sizeOf`/`@offsetOf`.

`cside.h`:

```c
#ifndef CSIDE_H
#define CSIDE_H
#include <stddef.h>

typedef struct { int x; int y; char tag; } CPoint;

enum Mode { MODE_OFF = 0, MODE_READ = 1, MODE_WRITE = 2 };

typedef struct { unsigned int read:1; unsigned int write:1; unsigned int exec:1; } CFlags;

size_t c_point_size(void);
size_t c_point_offset_y(void);
int c_point_sum(CPoint p);
void c_point_bump(CPoint *p, int by);
int c_mode_value(enum Mode m);
char *c_make_greeting(const char *name);
void c_free_greeting(char *s);
int c_flags_bits(CFlags f);

#endif
```

`cside.c`:

```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include "cside.h"

size_t c_point_size(void)     { return sizeof(CPoint); }
size_t c_point_offset_y(void) { return offsetof(CPoint, y); }
int c_point_sum(CPoint p)     { return p.x + p.y; }
void c_point_bump(CPoint *p, int by) { p->x += by; p->y += by; }
int c_mode_value(enum Mode m) { return (int)m * 100; }

char *c_make_greeting(const char *name) {
    size_t n = strlen(name) + 8;
    char *s = malloc(n);
    snprintf(s, n, "hi, %s!", name);
    return s;
}

void c_free_greeting(char *s) { free(s); }

int c_flags_bits(CFlags f) { return f.read | (f.write << 1) | (f.exec << 2); }
```

`main.zig`:

```zig
const std = @import("std");

const CPoint = extern struct { x: c_int, y: c_int, tag: u8 };

const Mode = enum(c_uint) { off = 0, read = 1, write = 2 };

const CFlags = packed struct(c_uint) {
    read: bool,
    write: bool,
    exec: bool,
    _pad: u29 = 0,
};

extern fn c_point_size() usize;
extern fn c_point_offset_y() usize;
extern fn c_point_sum(p: CPoint) c_int;
extern fn c_point_bump(p: *CPoint, by: c_int) void;
extern fn c_mode_value(m: Mode) c_int;
extern fn c_make_greeting(name: [*:0]const u8) [*:0]u8;
extern fn c_free_greeting(s: [*:0]u8) void;
extern fn c_flags_bits(f: CFlags) c_int;

pub fn main() void {
    std.debug.print("zig  @sizeOf  = {d}   C sizeof  = {d}\n", .{ @sizeOf(CPoint), c_point_size() });
    std.debug.print("zig  @offsetOf= {d}   C offsetof= {d}\n", .{ @offsetOf(CPoint, "y"), c_point_offset_y() });

    var p: CPoint = .{ .x = 3, .y = 4, .tag = 'p' };
    std.debug.print("c_point_sum   = {d}\n", .{c_point_sum(p)});
    c_point_bump(&p, 10);
    std.debug.print("after bump    = ({d}, {d}) tag={c}\n", .{ p.x, p.y, p.tag });

    std.debug.print("c_mode_value(.write) = {d}\n", .{c_mode_value(.write)});

    const g = c_make_greeting("raj");
    defer c_free_greeting(g);
    std.debug.print("C-owned string = {s} (len {d})\n", .{ std.mem.span(g), std.mem.span(g).len });

    const f: CFlags = .{ .read = true, .write = false, .exec = true };
    std.debug.print("c_flags_bits  = {d} (0b{b})\n", .{ c_flags_bits(f), c_flags_bits(f) });
}
```

```sh
zig build-exe main.zig cside.c -lc -I.
./main
```

```
zig  @sizeOf  = 12   C sizeof  = 12
zig  @offsetOf= 4   C offsetof= 4
c_point_sum   = 7
after bump    = (13, 14) tag=p
c_mode_value(.write) = 200
C-owned string = hi, raj! (len 8)
c_flags_bits  = 5 (0b101)
```

Both sides agree on 12 and 4. **Put a check like this in a test** when you wrap
a real library — it catches a mistyped field before a segfault does.

Note `c_point_bump(&p, 10)` mutating a Zig stack struct through a C pointer.
`extern struct` means the bytes are laid out the way C expects, so C can write
to them directly. That is the whole point.

## Enums

A C `enum` is an `int`-sized integer with named constants. It is **not**
exhaustive — `(enum Mode)99` is legal C. Three ways to model it:

```zig
const std = @import("std");

const Exhaustive = enum(c_uint) { off = 0, read = 1, write = 2 };
const Open = enum(c_uint) { off = 0, read = 1, write = 2, _ };

const MODE_OFF: c_uint = 0;
const MODE_READ: c_uint = 1;
const MODE_WRITE: c_uint = 2;

pub fn main() void {
    const a: Exhaustive = .write;
    std.debug.print("enum(c_uint)  tag={t} int={d} size={d}\n", .{ a, @intFromEnum(a), @sizeOf(Exhaustive) });

    const b: Open = @enumFromInt(99);
    std.debug.print("non-exhaustive int={d} known={?s}\n", .{ @intFromEnum(b), std.enums.tagName(Open, b) });

    const c: Open = .read;
    std.debug.print("non-exhaustive known={?s}\n", .{std.enums.tagName(Open, c)});

    std.debug.print("plain consts  = {d} {d} {d}\n", .{ MODE_OFF, MODE_READ, MODE_WRITE });
}
```

```sh
zig run enums.zig
```

```
enum(c_uint)  tag=write int=2 size=4
non-exhaustive int=99 known=null
non-exhaustive known=read
plain consts  = 0 1 2
```

| Form | When | Risk |
|---|---|---|
| `enum(c_uint) { … }` | you control all callers; values are closed | C returning an unlisted value is **UB**; `{t}` on it **panics** |
| `enum(c_uint) { …, _ }` | a C library might return something new | none — `std.enums.tagName` returns `?[]const u8` |
| plain `const`s + `c_uint` | matches `translate-c`; version-tolerant flags | no type safety |

**Prefer the non-exhaustive `_` form for anything a C library returns.** The
`tagName` on an unlisted value returning `null` instead of panicking is the
whole reason. `@tagName` would panic; `std.enums.tagName` does not.

Also note `@intFromEnum` and `@enumFromInt` both compile, but **`zig fmt`
rewrites them** to `@backingInt` and `@intCast`. Write the new spellings if you
run `zig fmt --check` in CI.

## Bitflags

C bitfields and OR-ed flag constants both map to `packed struct(T)`, which
guarantees bit positions:

```zig
const std = @import("std");

const Flags = packed struct(c_uint) {
    read: bool = false,
    write: bool = false,
    exec: bool = false,
    _pad: u29 = 0,
};

pub fn main() void {
    const f: Flags = .{ .read = true, .exec = true };
    const bits: c_uint = @bitCast(f);
    std.debug.print("struct -> bits = {d} (0b{b:0>3})\n", .{ bits, bits });
    std.debug.print("size = {d}, bitSize = {d}\n", .{ @sizeOf(Flags), @bitSizeOf(Flags) });

    const from_c: Flags = @bitCast(@as(c_uint, 0b110));
    std.debug.print("bits -> struct = read={} write={} exec={}\n", .{ from_c.read, from_c.write, from_c.exec });

    var acc: Flags = .{};
    acc.write = true;
    acc.read = true;
    std.debug.print("accumulated    = 0b{b:0>3}\n", .{@as(c_uint, @bitCast(acc))});
}
```

```sh
zig run flags.zig
```

```
struct -> bits = 5 (0b101)
size = 4, bitSize = 32
bits -> struct = read=false write=true exec=true
accumulated    = 0b011
```

`packed struct(c_uint)` makes the backing integer explicit, so `@bitCast` to and
from `c_uint` is free and total. Field 0 is the **low** bit. The `_pad: u29`
is required — a `packed struct(c_uint)` must account for all 32 bits.

This beats `flags: c_uint` plus `const O_RDONLY = 1`: `acc.write = true` cannot
be confused with `acc.exec = true`, and you never write `|` with the wrong
constant.

## Pointers: `*T` vs `[*]T` vs `[]T` vs `[*c]T` vs `?*T`

Zig has one C pointer type and four better ones. The distinctions C cannot
express are exactly where Zig adds value.

```zig
const std = @import("std");

pub fn main() void {
    var arr = [_]c_int{ 10, 20, 30 };

    const single: *c_int = &arr[0];
    const many: [*]c_int = &arr;
    const slice: []c_int = &arr;
    const cptr: [*c]c_int = &arr;
    const opt: ?*c_int = &arr[1];
    const nil: ?*c_int = null;

    std.debug.print("*T    deref        = {d}\n", .{single.*});
    std.debug.print("[*]T  index        = {d}\n", .{many[2]});
    std.debug.print("[]T   len+index    = {d} {d}\n", .{ slice.len, slice[1] });
    std.debug.print("[*c]T index        = {d}\n", .{cptr[0]});
    std.debug.print("[*c]T allows null  = {}\n", .{cptr == null});
    std.debug.print("?*T   unwrap       = {?d}\n", .{if (opt) |p| p.* else null});
    std.debug.print("?*T   null         = {}\n", .{nil == null});

    std.debug.print("sizeOf(*c_int)     = {d}\n", .{@sizeOf(*c_int)});
    std.debug.print("sizeOf(?*c_int)    = {d}\n", .{@sizeOf(?*c_int)});
    std.debug.print("sizeOf([]c_int)    = {d}\n", .{@sizeOf([]c_int)});
    std.debug.print("sizeOf([*c]c_int)  = {d}\n", .{@sizeOf([*c]c_int)});

    const back: []c_int = cptr[0..3];
    std.debug.print("[*c]T -> slice     = {any}\n", .{back});
}
```

```sh
zig run ptrs.zig
```

```
*T    deref        = 10
[*]T  index        = 30
[]T   len+index    = 3 20
[*c]T index        = 10
[*c]T allows null  = false
?*T   unwrap       = 20
?*T   null         = true
sizeOf(*c_int)     = 8
sizeOf(?*c_int)    = 8
sizeOf([]c_int)    = 16
sizeOf([*c]c_int)  = 8
[*c]T -> slice     = { 10, 20, 30 }
```

**`?*T` is 8 bytes, same as `*T`.** The null-pointer optimization means
optionality is free at the ABI level — `null` *is* the zero address. So a C
`T*` that may be NULL should always be `?*T` in your declaration. You pay
nothing and the compiler forces you to check.

**`[]T` is 16 bytes** — pointer plus length. It has no C equivalent and can
never cross the boundary. A C function taking `(int *xs, size_t n)` becomes
`([*]c_int, usize)` in Zig, and you pass `slice.ptr, slice.len`.

| Type | Size | Null? | Arithmetic? | Length? | Use for a C `T*` when |
|---|---|---|---|---|---|
| `*T` | 8 | **no** | no | — | exactly one item, never NULL |
| `?*T` | 8 | yes | no | — | exactly one item, may be NULL |
| `[*]T` | 8 | **no** | yes | no | many items, count passed separately |
| `?[*]T` | 8 | yes | yes | no | many items, may be NULL |
| `[*:0]T` | 8 | no | yes | sentinel | a NUL-terminated string (11.3) |
| `[]T` | **16** | no | yes | **yes** | **never** — Zig-internal only |
| `[*c]T` | 8 | yes | yes | no | **last resort** — see below |

### `[*c]T` is what `translate-c` emits, not what you should write

`[*c]T` is the "C pointer": it allows null, allows arithmetic, and
**auto-coerces to and from every other pointer type**. `translate-c` has to use
it, because a C header does not say whether NULL is legal or whether the pointer
addresses one item or many.

You do know. So after reading `translate-c`'s output, replace it:

| `translate-c` gave you | Write instead | Because the header's docs say |
|---|---|---|
| `[*c]const u8` | `[*:0]const u8` | it's a NUL-terminated string |
| `[*c]const u8` | `?[*:0]const u8` | …and may return NULL |
| `[*c]c_int` | `*c_int` | it's an out-parameter for one int |
| `[*c]c_int` | `[*]c_int` | it's an array with a separate count |
| `[*c]anyopaque` | `?*anyopaque` | it's an opaque handle |

Every replacement buys a compile-time check that `[*c]` silently skips. Keeping
`[*c]` everywhere gives you C's safety with Zig's syntax — the worst trade
available.

## Opaque handles

For a C type you only ever hold a pointer to — `FILE`, `sqlite3`, any
`typedef struct foo foo;` with no visible fields — use `opaque {}`:

```zig
const std = @import("std");

const Handle = opaque {};

extern fn fopen(path: [*:0]const u8, mode: [*:0]const u8) ?*Handle;
extern fn fclose(h: *Handle) c_int;
extern fn fputs(s: [*:0]const u8, h: *Handle) c_int;

pub fn main() !void {
    const h = fopen("/tmp/zc11_opaque.txt", "w") orelse return error.OpenFailed;
    _ = fputs("written through an opaque handle\n", h);
    _ = fclose(h);
    std.debug.print("wrote /tmp/zc11_opaque.txt, sizeOf(*Handle)={d}\n", .{@sizeOf(*Handle)});
}
```

```sh
zig run -lc opaque.zig
cat /tmp/zc11_opaque.txt
```

```
wrote /tmp/zc11_opaque.txt, sizeOf(*Handle)=8
written through an opaque handle
```

An `opaque {}` type has unknown size, so `*Handle` is the only way to hold it
and `h.*` is a compile error. You cannot accidentally copy a struct whose layout
you don't know — which is exactly the bug `void *` lets you write in C.

## `@ptrCast` and `@alignCast`

Crossing from `*anyopaque` back to a real type needs two separate casts: one for
the type, one for the alignment the compiler lost track of.

```zig
const std = @import("std");
const c = std.c;

const Header = extern struct { magic: u32, count: u16, kind: u8 };

pub fn main() !void {
    const mem = c.malloc(@sizeOf(Header)) orelse return error.OutOfMemory;
    defer c.free(mem);

    const h: *Header = @ptrCast(@alignCast(mem));
    h.* = .{ .magic = 0xDEADBEEF, .count = 3, .kind = 7 };
    std.debug.print("via *anyopaque: magic=0x{X} count={d} kind={d}\n", .{ h.magic, h.count, h.kind });

    const raw: [*]const u8 = @ptrCast(h);
    std.debug.print("raw bytes      = {any}\n", .{raw[0..@sizeOf(Header)]});

    const as_bytes = std.mem.asBytes(h);
    std.debug.print("asBytes len    = {d}\n", .{as_bytes.len});

    const back: *const Header = std.mem.bytesAsValue(Header, as_bytes);
    std.debug.print("bytesAsValue   = 0x{X}\n", .{back.magic});

    var ints = [_]c_int{ 5, 1, 4 };
    const opaque_ptr: *anyopaque = @ptrCast(&ints);
    const again: [*]c_int = @ptrCast(@alignCast(opaque_ptr));
    std.debug.print("round tripped  = {any}\n", .{again[0..3]});
}
```

```sh
zig run -lc casts.zig
```

```
via *anyopaque: magic=0xDEADBEEF count=3 kind=7
raw bytes      = { 239, 190, 173, 222, 3, 0, 7, 0 }
asBytes len    = 8
bytesAsValue   = 0xDEADBEEF
round tripped  = { 5, 1, 4 }
```

`{ 239, 190, 173, 222, … }` is `0xDEADBEEF` little-endian: `EF BE AD DE`. Then
`03 00` for `count`, `07` for `kind`, and one padding byte.

| Builtin | Changes | Note |
|---|---|---|
| `@ptrCast` | the pointee type | cannot *increase* alignment |
| `@alignCast` | the claimed alignment | **asserts at runtime** in Debug; UB if wrong in ReleaseFast |
| `@constCast` | removes `const` | writing through it is UB if the data really is const |
| `@intFromPtr` / `@ptrFromInt` | pointer ↔ `usize` | for `uintptr_t` parameters |
| `@bitCast` | reinterprets a **value** | same size required; `packed struct` ↔ integer |
| `std.mem.asBytes(p)` | `*T` → `*[N]u8` | safe, no cast needed |
| `std.mem.bytesAsValue(T, b)` | bytes → `*T` | checks the length |

`@alignCast(mem)` goes **inside** `@ptrCast`. Written the other way round it
still compiles but means something different: you'd be claiming alignment on the
already-cast pointer rather than on the source. Habit: `@ptrCast(@alignCast(p))`.

`malloc` returns suitably-aligned memory for anything, so that `@alignCast` is
always satisfied. Casting into the *middle* of a buffer is where it fires —
and in Debug it panics instead of producing a misaligned load.

---

Next: [11.3 — C Strings](./03-c-strings.md)
