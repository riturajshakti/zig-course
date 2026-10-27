# 2.2 — Numbers

There is no `int`. Every integer type states its signedness and its exact width,
and the width can be any number of bits from 0 to 65535.

```zig
const std = @import("std");

pub fn main() void {
    const a: u8 = 255;
    const b: i8 = -128;
    const c: u3 = 7;
    const d: i7 = -64;
    const e: u64 = 18446744073709551615;
    const f: usize = 42;
    const g: isize = -42;

    std.debug.print("u8     {d}\n", .{a});
    std.debug.print("i8     {d}\n", .{b});
    std.debug.print("u3     {d}\n", .{c});
    std.debug.print("i7     {d}\n", .{d});
    std.debug.print("u64    {d}\n", .{e});
    std.debug.print("usize  {d}\n", .{f});
    std.debug.print("isize  {d}\n", .{g});
}
```

```
u8     255
i8     -128
u3     7
i7     -64
u64    18446744073709551615
usize  42
isize  -42
```

`u3` is not a rounded-up `u8`. It is a 3-bit integer with a range of 0–7, and in
a `packed struct` it occupies exactly three bits. That is why `u1`, `u7`, `u24`
and `u48` exist: hardware registers and binary wire formats are not built out of
bytes.

## Ranges, without arithmetic

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("u8    {d} .. {d}\n", .{ std.math.minInt(u8), std.math.maxInt(u8) });
    std.debug.print("i8    {d} .. {d}\n", .{ std.math.minInt(i8), std.math.maxInt(i8) });
    std.debug.print("u3    {d} .. {d}\n", .{ std.math.minInt(u3), std.math.maxInt(u3) });
    std.debug.print("i7    {d} .. {d}\n", .{ std.math.minInt(i7), std.math.maxInt(i7) });
    std.debug.print("u32   {d} .. {d}\n", .{ std.math.minInt(u32), std.math.maxInt(u32) });
    std.debug.print("i64   {d} .. {d}\n", .{ std.math.minInt(i64), std.math.maxInt(i64) });
    std.debug.print("usize {d} .. {d}\n", .{ std.math.minInt(usize), std.math.maxInt(usize) });
}
```

```
u8    0 .. 255
i8    -128 .. 127
u3    0 .. 7
i7    -64 .. 63
u32   0 .. 4294967295
i64   -9223372036854775808 .. 9223372036854775807
usize 0 .. 18446744073709551615
```

`minInt` and `maxInt` take the *type* as an argument and run at compile time —
`std.math.maxInt(u8)` is the constant `255` in the binary, not a function call.

| Type | Use it for |
|---|---|
| `u8` | a byte; a UTF-8 code unit; Zig's "char" |
| `i32` | ordinary signed integers |
| `u32` / `u64` | ids, counters, hashes, timestamps |
| `usize` | **lengths, indices, sizes** — pointer-width, 64-bit here |
| `isize` | a signed offset or difference between pointers |
| `u1` … `u7`, `u24`, `u48` | bit fields, packed structs, wire formats |

**`usize` is not interchangeable with `u64`** even though both are 64 bits here.
`.len` is a `usize`, array indices are `usize`, and `@sizeOf` returns `usize`.
Use `usize` whenever the number counts memory, and a fixed width whenever the
number is data.

## Literals

```zig
const std = @import("std");

pub fn main() void {
    const dec = 1_000_000;
    const hex = 0xFF;
    const bin = 0b1010_0001;
    const oct = 0o755;
    const ch = 'A';
    const newline = '\n';
    const unicode = 'λ';

    std.debug.print("{d}\n", .{dec});
    std.debug.print("{d} {x}\n", .{ hex, hex });
    std.debug.print("{d} {b}\n", .{ bin, bin });
    std.debug.print("{d} {o}\n", .{ oct, oct });
    std.debug.print("{d} {c}\n", .{ ch, ch });
    std.debug.print("{d}\n", .{newline});
    std.debug.print("{d} {u}\n", .{ unicode, unicode });
}
```

```
1000000
255 ff
161 10100001
493 755
65 A
10
955 λ
```

| Literal | Base | Note |
|---|---|---|
| `1_000_000` | 10 | `_` is a readability separator anywhere between digits |
| `0xFF` | 16 | `0xDEAD_BEEF` works too |
| `0b1010_0001` | 2 | group nibbles with `_` |
| `0o755` | 8 | `0o` prefix — a bare leading `0` is **not** octal |
| `'A'` | — | a character literal is an **integer**, value 65 |
| `'λ'` | — | the Unicode codepoint 955; print it with `{u}` |

A character literal really is just a number — `'A' + 1 == 66`. There is no
separate `char` type, which is why `u8` doubles as Zig's byte-and-character type.

## Floats

```zig
const std = @import("std");

pub fn main() void {
    const a: f16 = 1.0 / 3.0;
    const b: f32 = 1.0 / 3.0;
    const c: f64 = 1.0 / 3.0;
    const d: f128 = 1.0 / 3.0;

    std.debug.print("f16  {d:.10}\n", .{a});
    std.debug.print("f32  {d:.10}\n", .{b});
    std.debug.print("f64  {d:.10}\n", .{c});
    std.debug.print("f128 {d:.10}\n", .{d});

    std.debug.print("sizes {d} {d} {d} {d}\n", .{ @sizeOf(f16), @sizeOf(f32), @sizeOf(f64), @sizeOf(f128) });
}
```

```
f16  0.3333000000
f32  0.3333333400
f64  0.3333333333
f128 0.3333333333
sizes 2 4 8 16
```

Watch the precision die off as the type shrinks: `f16` manages three digits.
`f64` is the sane default; `f16` is for GPUs and ML weights; `f128` is rare.

Floats behave the way IEEE-754 floats always do, and `std.math` gives you the
vocabulary for it:

```zig
const std = @import("std");

pub fn main() void {
    const inf = std.math.inf(f64);
    const nan = std.math.nan(f64);
    var a: f64 = 0.1;
    a += 0.2;

    std.debug.print("{d} {d}\n", .{ inf, -inf });
    std.debug.print("{d}\n", .{nan});
    std.debug.print("nan == nan   -> {}\n", .{nan == nan});
    std.debug.print("isNan(nan)   -> {}\n", .{std.math.isNan(nan)});
    std.debug.print("a == 0.3     -> {}\n", .{a == 0.3});
    std.debug.print("a            =  {d:.20}\n", .{a});
}
```

```
inf -inf
nan
nan == nan   -> false
isNan(nan)   -> true
a == 0.3     -> false
a            =  0.30000000000000004000
```

`a` had to be a `var` mutated at runtime to show that last line. Pin it to `f64`
at compile time instead and the comparison flips:

```zig
const std = @import("std");

pub fn main() void {
    const a: f64 = 0.1 + 0.2;
    std.debug.print("const f64    -> {}\n", .{a == 0.3});
    std.debug.print("comptime_float -> {}\n", .{0.1 + 0.2 == 0.3});
}
```

```
const f64    -> true
comptime_float -> false
```

`const a: f64 = 0.1 + 0.2;` rounds the sum to the nearest `f64`, and `0.3`
rounds to that same `f64`, so they match. The bare literal expression stays
`comptime_float` — 128-bit precision — where the two values are still
distinguishable. The lesson is not "floats are broken" but **the width you
compare at decides the answer**.

## Mixing widths

Zig coerces *widening* conversions silently, because they cannot lose
information:

```zig
const std = @import("std");

pub fn main() void {
    var a: u8 = 7;
    a += 0;
    const b: u16 = a;
    const c: i32 = a;
    const d: f64 = a;
    std.debug.print("{d} {d} {d}\n", .{ b, c, d });
}
```

```
7 7 7
```

Narrowing is a compile error — no silent truncation, ever:

```zig
const std = @import("std");

pub fn main() void {
    var a: u16 = 300;
    a += 0;
    const b: u8 = a;
    std.debug.print("{d}\n", .{b});
}
```

```
error: expected type 'u8', found 'u16'
    const b: u8 = a;
                  ^
note: unsigned 8-bit int cannot represent all possible unsigned 16-bit values
```

Note the compiler's reasoning: not "different types" but "cannot represent all
possible values." That is the actual rule, and it is why `u8 → i32` is fine
while `u16 → u8` is not. Arithmetic follows the same logic — operands are
coerced to whichever type can hold both:

```zig
const std = @import("std");

pub fn main() void {
    const x: u8 = 10;
    const y: u16 = 20;
    const z = x + y;
    std.debug.print("{d} {s}\n", .{ z, @typeName(@TypeOf(z)) });
}
```

```
30 u16
```

| Conversion | Allowed? |
|---|---|
| `u8` → `u16`, `u8` → `i32`, `i8` → `i64` | implicit — no information lost |
| `u8` → `f64`, `i32` → `f64` | implicit |
| `u16` → `u8`, `i32` → `u32`, `i64` → `i32` | **compile error** — needs `@intCast` or `@truncate` |
| `f64` → `f32`, `f64` → `i32` | **compile error** — needs an explicit builtin |
| `comptime_int` → any int that fits | implicit, checked at compile time |

Lesson 2.3 covers the explicit casts and what happens when a value does not fit.

---

Next: [2.3 — Overflow & Casting](./03-overflow-casting.md)
