# 2.3 — Overflow & Casting

This is the lesson where Zig stops resembling C.

## Overflow is a crash, not a wrap

```zig
const std = @import("std");

pub fn main() void {
    var x: u8 = 250;
    x += 10;
    std.debug.print("x = {d}\n", .{x});
}
```

```sh
zig run overflow.zig
```

```
thread 16406623 panic: integer overflow
/tmp/overflow.zig:5:7: 0x10064d1c3 in main (overflow)
    x += 10;
      ^
```

No output, no `4`, no wraparound. The program **dies** pointing at the exact
operator. In C, Go, Java and Rust-in-release this silently produces `4`.

Now the same binary optimized:

```sh
zig run -O ReleaseFast overflow.zig
```

```
x = 4
```

**Same source. Different behavior.** Run all four modes and you get:

| Build mode | `250 + 10` on a `u8` |
|---|---|
| `Debug` (default) | **panic: integer overflow** |
| `ReleaseSafe` | **panic: integer overflow** |
| `ReleaseFast` | `4` — wraps silently |
| `ReleaseSmall` | `4` — wraps silently |

This is not a bug; it is the deal Zig offers. Overflow is **undefined behavior**
in Zig, and the safe build modes insert a check that catches the UB instead of
letting the optimizer exploit it. So:

- Overflow is always a **bug in your code**, never a feature of the type.
- Debug and ReleaseSafe find that bug for you, loudly, at the line that caused it.
- ReleaseFast assumes you already found it and compiles the check away.

If you *want* wrapping, you must say so — and then it wraps in every mode.

## Say what you mean: `+%` and `+|`

```zig
const std = @import("std");

pub fn main() void {
    var a: u8 = 250;
    var b: u8 = 250;
    var c: u8 = 5;
    var d: u8 = 100;

    a +%= 10;
    b +|= 10;
    c -%= 10;
    d *%= 3;

    std.debug.print("250 +%  10 = {d}\n", .{a});
    std.debug.print("250 +|  10 = {d}\n", .{b});
    std.debug.print("  5 -%  10 = {d}\n", .{c});
    std.debug.print("100 *%   3 = {d}\n", .{d});

    var e: i8 = 120;
    var f: i8 = -120;
    e +|= 100;
    f -|= 100;
    std.debug.print("i8 120 +| 100 = {d}\n", .{e});
    std.debug.print("i8 -120 -| 100 = {d}\n", .{f});
}
```

```
250 +%  10 = 4
250 +|  10 = 255
  5 -%  10 = 251
100 *%   3 = 44
i8 120 +| 100 = 127
i8 -120 -| 100 = -128
```

| Operator | Name | On overflow |
|---|---|---|
| `+` `-` `*` | checked | **panics** in Debug/ReleaseSafe, UB in ReleaseFast |
| `+%` `-%` `*%` | wrapping | wraps modulo 2ⁿ — hashes, checksums, PRNGs |
| `+|` `-|` `*|` | saturating | clamps to the type's min/max — audio, pixels, gauges |

Saturating arithmetic is the one most languages make you hand-write. `+|` on a
`u8` gives you exactly the "pin at 255" behavior image and audio code wants, and
it is a single machine instruction on most targets.

## Checked arithmetic: the error-returning version

When overflow is a *possible input condition* rather than a bug — parsing
user data, summing untrusted numbers — ask for an error instead of a panic:

```zig
const std = @import("std");

pub fn main() void {
    const r1 = std.math.add(u8, 250, 10);
    const r2 = std.math.add(u8, 200, 10);
    const r3 = std.math.sub(u8, 5, 10);
    const r4 = std.math.mul(u8, 100, 3);

    std.debug.print("add(250,10) -> {!d}\n", .{r1});
    std.debug.print("add(200,10) -> {!d}\n", .{r2});
    std.debug.print("sub(5,10)   -> {!d}\n", .{r3});
    std.debug.print("mul(100,3)  -> {!d}\n", .{r4});
    std.debug.print("catch 0     -> {d}\n", .{std.math.add(u8, 250, 10) catch 0});

    var x: u8 = 250;
    x += 0;
    const ov = @addWithOverflow(x, @as(u8, 10));
    std.debug.print("@addWithOverflow -> value={d} overflowed={d}\n", .{ ov[0], ov[1] });
}
```

```
add(250,10) -> error.Overflow
add(200,10) -> 210
sub(5,10)   -> error.Overflow
mul(100,3)  -> error.Overflow
catch 0     -> 0
@addWithOverflow -> value=4 overflowed=1
```

Two tools, two shapes:

- `std.math.add/sub/mul(T, a, b)` returns `error{Overflow}!T`, so it composes
  with `try` and `catch` like any other failure.
- `@addWithOverflow(a, b)` returns a **tuple** `.{ wrapped_value, overflow_bit }`
  with no error machinery at all — it is the raw hardware flag, for hot loops.

| Need | Use |
|---|---|
| overflow means my code is broken | plain `+` — let it panic |
| overflow means bad input | `std.math.add` + `try`/`catch` |
| I want modular arithmetic | `+%` |
| I want clamping | `+|` |
| I want the value *and* the flag | `@addWithOverflow` |

## Casting: four builtins, four different meanings

```zig
const std = @import("std");

pub fn main() void {
    var big: u32 = 300;
    big += 0;

    const narrow: u8 = @intCast(big % 256);
    const trunc: u8 = @truncate(big);
    const widened = @as(u32, 7);

    std.debug.print("@intCast(300 % 256) = {d}\n", .{narrow});
    std.debug.print("@truncate(300)      = {d}\n", .{trunc});
    std.debug.print("@as(u32, 7)         = {d}\n", .{widened});
    std.debug.print("300 in binary       = {b}\n", .{big});
    std.debug.print("low 8 bits          = {b}\n", .{trunc});
}
```

```
@intCast(300 % 256) = 44
@truncate(300)      = 44
@as(u32, 7)         = 7
300 in binary       = 100101100
low 8 bits          = 101100
```

Same answer, two completely different contracts. `@truncate` *discards* the high
bits — `100101100` becomes `101100` — and is always safe and always silent.
`@intCast` asserts the value already fits, and if it does not:

```zig
const std = @import("std");

pub fn main() void {
    var big: u32 = 300;
    big += 0;
    const narrow: u8 = @intCast(big);
    std.debug.print("{d}\n", .{narrow});
}
```

```
thread 16408257 panic: integer does not fit in destination type
/tmp/cast.zig:6:24: 0x102d191e3 in main (cast)
    const narrow: u8 = @intCast(big);
                       ^
```

That panic is the whole point of `@intCast`: it is a documented assumption that
gets checked in Debug. **Reach for `@intCast` by default** and only use
`@truncate` when dropping bits is the actual intent (hashing, masking, reading a
protocol field).

Notice also that none of these take a target type as an argument — the result
type comes from the destination. `const narrow: u8 = @intCast(big);` works
because the annotation says `u8`; written bare it cannot infer anything.

## Integers and floats never mix implicitly

```zig
const std = @import("std");

pub fn main() void {
    var n: i32 = 300;
    n += 0;
    var x: f64 = 3.99;
    x += 0.0;
    var neg: f64 = -3.99;
    neg += 0.0;

    const as_float: f64 = @floatFromInt(n);
    const as_int: i32 = @intFromFloat(x);
    const neg_int: i32 = @intFromFloat(neg);

    std.debug.print("@floatFromInt(300)    = {d}\n", .{as_float});
    std.debug.print("@intFromFloat(3.99)   = {d}\n", .{as_int});
    std.debug.print("@intFromFloat(-3.99)  = {d}\n", .{neg_int});
    std.debug.print("rounded               = {d}\n", .{@as(i32, @intFromFloat(@round(x)))});
}
```

```
@floatFromInt(300)    = 300
@intFromFloat(3.99)   = 3
@intFromFloat(-3.99)  = -3
rounded               = 4
```

`@intFromFloat` truncates **toward zero** — `3.99 → 3` and `-3.99 → -3`, not
`-4`. If you want rounding, round first with `@round`, `@floor` or `@ceil`.

You will write `@floatFromInt` constantly, because computing an average means
converting both operands:

```zig
const std = @import("std");

pub fn main() void {
    const nums = [_]u32{ 3, 7, 8, 10 };
    var total: u32 = 0;
    for (nums) |n| total += n;

    const avg = @as(f64, @floatFromInt(total)) / @as(f64, @floatFromInt(nums.len));
    std.debug.print("total {d}, count {d}, avg {d:.2}\n", .{ total, nums.len, avg });
}
```

```
total 28, count 4, avg 7.00
```

Verbose? Yes. It is also the only version where you can see exactly where the
precision loss happens — which is the whole argument for explicit casts.

## `@bitCast` — reinterpret the bits, change nothing

```zig
const std = @import("std");

pub fn main() void {
    var minus_one: i8 = -1;
    minus_one += 0;

    const as_u8: u8 = @bitCast(minus_one);
    const back: i8 = @bitCast(as_u8);

    std.debug.print("i8 -1  @bitCast-> u8 {d}  ({b})\n", .{ as_u8, as_u8 });
    std.debug.print("u8 255 @bitCast-> i8 {d}\n", .{back});

    var f: f32 = 1.0;
    f += 0.0;
    const bits: u32 = @bitCast(f);
    std.debug.print("f32 1.0 bit pattern = 0x{x}\n", .{bits});
}
```

```
i8 -1  @bitCast-> u8 255  (11111111)
u8 255 @bitCast-> i8 -1
f32 1.0 bit pattern = 0x3f800000
```

`@bitCast` copies the bit pattern into a type of the **same bit width** and
reinterprets it. `-1` as `i8` is `11111111`; those same eight bits read as `u8`
are `255`. Nothing is computed; nothing can fail. It is the tool for binary
formats and the float-bits tricks in module 15.

| Builtin | Changes | Fails when |
|---|---|---|
| `@as(T, v)` | nothing — just states the type of a coercion that is already legal | the coercion isn't legal (compile error) |
| `@intCast(v)` | int width/signedness, **value preserved** | value doesn't fit — panics in Debug |
| `@truncate(v)` | int width, **keeps the low bits** | never |
| `@bitCast(v)` | the type only, **bits preserved** | widths differ (compile error) |
| `@floatFromInt(v)` | int → float | never (may lose precision) |
| `@intFromFloat(v)` | float → int, truncating toward zero | value out of range — panics in Debug |

## Division on signed integers: pick your rounding

Here Zig refuses to guess:

```zig
const std = @import("std");

pub fn main() void {
    var a: i32 = -7;
    a += 0;
    var b: i32 = 2;
    b += 0;
    std.debug.print("{d}\n", .{a / b});
}
```

```
error: division with 'i32' and 'i32': signed integers must use @divTrunc, @divFloor, @divCeil, or @divExact
    std.debug.print("{d}\n", .{a / b});
                               ~~^~~
```

`-7 / 2` is `-3` in C, Go and Rust, and `-4` in Python. Zig will not pick for
you, because the two conventions disagree about exactly the case your code is
most likely to hit:

```zig
const std = @import("std");

pub fn main() void {
    var a: i32 = -7;
    a += 0;
    var b: i32 = 2;
    b += 0;

    std.debug.print("@divTrunc(-7, 2) = {d}\n", .{@divTrunc(a, b)});
    std.debug.print("@divFloor(-7, 2) = {d}\n", .{@divFloor(a, b)});
    std.debug.print("@rem(-7, 2)      = {d}\n", .{@rem(a, b)});
    std.debug.print("@mod(-7, 2)      = {d}\n", .{@mod(a, b)});
    std.debug.print("@divTrunc(7, 2)  = {d}\n", .{@divTrunc(-a, b)});
    std.debug.print("@divFloor(7, 2)  = {d}\n", .{@divFloor(-a, b)});
}
```

```
@divTrunc(-7, 2) = -3
@divFloor(-7, 2) = -4
@rem(-7, 2)      = -1
@mod(-7, 2)      = 1
@divTrunc(7, 2)  = 3
@divFloor(7, 2)  = 3
```

| Builtin | `-7, 2` | Rounds | Pairs with |
|---|---|---|---|
| `@divTrunc` | `-3` | toward zero (C, Go, Rust) | `@rem` |
| `@divFloor` | `-4` | toward −∞ (Python) | `@mod` |
| `@divCeil` | `-3` | toward +∞ | — |
| `@divExact` | **panics** | asserts there is no remainder | — |
| `@rem` | `-1` | sign follows the **dividend** | `@divTrunc` |
| `@mod` | `1` | sign follows the **divisor** | `@divFloor` |

**Use `@mod` for wrapping an index into a range** — it is the one that is never
negative for a positive divisor. `@rem(-1, 8)` is `-1` and will index out of
bounds; `@mod(-1, 8)` is `7` and will not.

The sign rules only diverge for negative operands, which is exactly why this
bug class survives code review. On unsigned integers plain `/` and `%` work
fine, because there is nothing to disagree about.

Dividing by zero is caught too:

```zig
const std = @import("std");

pub fn main() void {
    var a: i32 = 1;
    a += 0;
    var z: i32 = 0;
    z += 0;
    std.debug.print("{d}\n", .{@divTrunc(a, z)});
}
```

```
thread 16408945 panic: division by zero
/tmp/divzero.zig:8:32: 0x10300d28f in main (divzero)
    std.debug.print("{d}\n", .{@divTrunc(a, z)});
                               ^
```

(All three panics in this lesson are deliberate. They are what Debug mode is
*for* — see module 09 for how release modes change which checks survive.)

---

Next: [2.4 — Control Flow](./04-control-flow.md)
