# 6.3 — Generic Functions

Two ways to write a function that works on many types. They are not
interchangeable, and knowing which to reach for is most of this lesson.

```zig
const std = @import("std");

fn maxExplicit(comptime T: type, a: T, b: T) T {
    return if (a > b) a else b;
}

fn maxAny(a: anytype, b: @TypeOf(a)) @TypeOf(a) {
    return if (a > b) a else b;
}

fn swap(comptime T: type, a: *T, b: *T) void {
    const t = a.*;
    a.* = b.*;
    b.* = t;
}

pub fn main() void {
    std.debug.print("{d} {d} {d}\n", .{ maxExplicit(u8, 3, 9), maxExplicit(f64, 1.5, -2.0), maxExplicit(i32, -7, -9) });
    std.debug.print("{d} {d}\n", .{ maxAny(@as(u8, 3), 9), maxAny(@as(f64, 1.5), -2.0) });

    var x: i32 = 1;
    var y: i32 = 2;
    swap(i32, &x, &y);
    std.debug.print("x={d} y={d}\n", .{ x, y });

    var s1: []const u8 = "first";
    var s2: []const u8 = "second";
    swap([]const u8, &s1, &s2);
    std.debug.print("s1={s} s2={s}\n", .{ s1, s2 });
}
```

```
9 1.5 -7
9 1.5
x=2 y=1
s1=second s2=first
```

| | `comptime T: type` | `anytype` |
|---|---|---|
| call site | `max(u8, a, b)` — type written out | `max(a, b)` — inferred |
| `T` usable in the signature | yes, including the return type | only via `@TypeOf` |
| constrains the arguments | yes, both must be `T` | no, each is independent |
| error messages blame | **the call site** | the function body |
| duck typing | no | yes |

## `comptime T: type` — the explicit form

`T` is a parameter like any other; it just happens to be a type, so it can
appear later in the same signature:

```zig
const std = @import("std");

fn sum(comptime T: type, vals: []const T) T {
    var acc: T = 0;
    for (vals) |v| acc += v;
    return acc;
}

fn zip(comptime A: type, comptime B: type, as: []const A, bs: []const B) void {
    for (as, bs) |a, b| std.debug.print("({any},{any}) ", .{ a, b });
    std.debug.print("\n", .{});
}

pub fn main() void {
    std.debug.print("{d} {d}\n", .{ sum(i32, &.{ 1, 2, 3 }), sum(f64, &.{ 1.5, 2.5 }) });
    zip(u8, bool, &.{ 1, 2, 3 }, &.{ true, false, true });
    zip(f32, []const u8, &.{ 1.5, 2.5 }, &.{ "a", "b" });
}
```

```
6 4
(1,true) (2,false) (3,true) 
(1.5,{ 97 }) (2.5,{ 98 }) 
```

Multiple type parameters need no extra ceremony — just more parameters. (`{any}`
on a `[]const u8` prints bytes as numbers; `{s}` would print `a` and `b`.)

## `anytype` — the inferred form

`anytype` means "infer this parameter's type from the argument." Each `anytype`
is inferred **separately**, which is why `maxAny` above declared `b: @TypeOf(a)`
— to force both to agree.

```zig
const std = @import("std");

fn maxAny(a: anytype, b: anytype) @TypeOf(a, b) {
    return if (a > b) a else b;
}

pub fn main() void {
    const r1 = maxAny(@as(u8, 3), @as(u32, 9));
    const r2 = maxAny(@as(i8, -3), @as(i64, 9));
    const r3 = maxAny(@as(f32, 1.5), @as(f64, 0.5));
    std.debug.print("u8,u32  -> {s} = {d}\n", .{ @typeName(@TypeOf(r1)), r1 });
    std.debug.print("i8,i64  -> {s} = {d}\n", .{ @typeName(@TypeOf(r2)), r2 });
    std.debug.print("f32,f64 -> {s} = {d}\n", .{ @typeName(@TypeOf(r3)), r3 });
    std.debug.print("u8,i32  -> {s}\n", .{@typeName(@TypeOf(@as(u8, 1), @as(i32, 1)))});
    std.debug.print("u8,u8,u64 -> {s}\n", .{@typeName(@TypeOf(@as(u8, 1), @as(u8, 2), @as(u64, 3)))});
}
```

```
u8,u32  -> u32 = 9
i8,i64  -> i64 = 9
f32,f64 -> f64 = 1.5
u8,i32  -> i32
u8,u8,u64 -> u64
```

## Peer type resolution

`@TypeOf(a, b)` is the trick in that signature: given several values, it returns
the type they all coerce to. This is the same algorithm that types an `if`
expression's two arms or a `switch`'s cases.

| Inputs | Result | Rule |
|---|---|---|
| `u8`, `u32` | `u32` | the wider unsigned wins |
| `i8`, `i64` | `i64` | same, signed |
| `u8`, `i32` | `i32` | needs to represent negatives *and* 0–255 |
| `f32`, `f64` | `f64` | the wider float |
| `u32`, `f64` | `f64` | float absorbs the int |
| `u8`, `u8`, `u64` | `u64` | takes any number of arguments |

Mixed-width *arithmetic* uses the same rule, which is why this is legal:

```zig
const std = @import("std");

pub fn main() void {
    const a: u8 = 200;
    const b: u16 = 400;
    const sum = a + b;
    std.debug.print("u8 + u16 -> {s} = {d}\n", .{ @typeName(@TypeOf(sum)), sum });

    const f = 1.5;
    const mixed = f + 2;
    std.debug.print("1.5 + 2 -> {s} = {d}\n", .{ @typeName(@TypeOf(mixed)), mixed });
}
```

```
u8 + u16 -> u16 = 600
1.5 + 2 -> comptime_float = 3.5
```

Only narrowing *assignment* errors. When no common type exists, peer resolution
says so:

```zig
const std = @import("std");

fn maxAny(a: anytype, b: anytype) @TypeOf(a, b) {
    return if (a > b) a else b;
}

pub fn main() void {
    std.debug.print("{d}\n", .{maxAny(@as(u32, 3), true)});
}
```

**Deliberate compile error:**

```
g9.zig:3:35: error: incompatible types: 'u32' and 'bool'
fn maxAny(a: anytype, b: anytype) @TypeOf(a, b) {
                                  ^~~~~~~~~~~~~
g9.zig:3:43: note: type 'u32' here
fn maxAny(a: anytype, b: anytype) @TypeOf(a, b) {
                                          ^
g9.zig:3:46: note: type 'bool' here
```

## Duck typing

`anytype` accepts a struct and lets you call methods on it. Nothing needs to be
declared — if the method exists, it compiles.

```zig
const std = @import("std");

const Dog = struct {
    fn speak(_: Dog) []const u8 {
        return "woof";
    }
};

const Cat = struct {
    fn speak(_: Cat) []const u8 {
        return "meow";
    }
};

fn announce(animal: anytype) void {
    std.debug.print("{s} says {s}\n", .{ @typeName(@TypeOf(animal)), animal.speak() });
}

pub fn main() void {
    announce(Dog{});
    announce(Cat{});
}
```

```
g2.Dog says woof
g2.Cat says meow
```

No interface, no `trait`, no vtable — and no runtime dispatch either. Two
separate compiled copies of `announce`, each with a direct call.

**This is Zig's interface mechanism**, and its cost is that the contract is
invisible. Pass something without `speak`:

```zig
const std = @import("std");

const Rock = struct { weight: u32 };

fn announce(animal: anytype) void {
    std.debug.print("{s}\n", .{animal.speak()});
}

pub fn main() void {
    announce(Rock{ .weight = 3 });
}
```

**Deliberate compile error:**

```
g3.zig:6:38: error: no field or member function named 'speak' in 'g3.Rock'
    std.debug.print("{s}\n", .{animal.speak()});
                               ~~~~~~^~~~~~
g3.zig:3:14: note: struct declared here
const Rock = struct { weight: u32 };
             ^~~~~~~~~~~~~~~~~~~~~~
referenced by:
    main: g3.zig:10:13
```

It's caught at compile time — but the error points **inside `announce`**, a
function the caller may not own. For a one-line helper that's fine. For a
library it is the C++-template problem: errors in code the user never wrote.

## Constraints: `@typeInfo` + `@compileError`

Zig has no `where` clause. You check the type yourself and fail with a message
you wrote:

```zig
const std = @import("std");

fn announce(animal: anytype) void {
    const T = @TypeOf(animal);
    if (!@hasDecl(T, "speak")) {
        @compileError(@typeName(T) ++ " cannot be announced: it has no `speak` method");
    }
    std.debug.print("{s}\n", .{animal.speak()});
}

const Rock = struct { weight: u32 };

pub fn main() void {
    announce(Rock{ .weight = 3 });
}
```

**Deliberate compile error:**

```
g4.zig:6:9: error: g4.Rock cannot be announced: it has no `speak` method
        @compileError(@typeName(T) ++ " cannot be announced: it has no `speak` method");
        ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
referenced by:
    main: g4.zig:14:13
```

Same bug, a message that names the type and the requirement. Four lines bought
you a usable diagnostic — do this for anything other people call.

> `@hasDecl` only sees **`pub`** declarations. A private `speak` reports
> `false`, which is usually what you want for an interface check.

### A constraint toolkit

```zig
const std = @import("std");

fn requireNumeric(comptime T: type, comptime who: []const u8) void {
    switch (@typeInfo(T)) {
        .int, .float, .comptime_int, .comptime_float => {},
        else => @compileError(who ++ " requires a numeric type, got " ++ @typeName(T)),
    }
}

fn average(comptime T: type, vals: []const T) f64 {
    requireNumeric(T, "average");
    if (vals.len == 0) return 0;
    var acc: f64 = 0;
    for (vals) |v| acc += switch (@typeInfo(T)) {
        .int, .comptime_int => @floatFromInt(v),
        else => @floatCast(v),
    };
    return acc / @as(f64, @floatFromInt(vals.len));
}

pub fn main() void {
    std.debug.print("{d:.2}\n", .{average(u8, &.{ 1, 2, 3, 4 })});
    std.debug.print("{d:.2}\n", .{average(f32, &.{ 1.5, 2.5 })});
    std.debug.print("{d:.2}\n", .{average(i64, &.{ -10, 10, 30 })});
}
```

```
2.50
2.00
10.00
```

Reusable checks for the cases that come up most:

| Requirement | Check |
|---|---|
| any integer | `@typeInfo(T) == .int` |
| unsigned only | `@typeInfo(T).int.signedness == .unsigned` |
| at most N bits | `@typeInfo(T).int.bits <= N` |
| a struct | `@typeInfo(T) == .@"struct"` |
| has a method | `@hasDecl(T, "name")` |
| has a field | `@hasField(T, "name")` |
| a slice | `@typeInfo(T).pointer.size == .slice` |
| an optional | `@typeInfo(T) == .optional` |
| comparable with `==` | not a struct, union, array or slice |

## Choosing between the two forms

Compare the error each gives for the *same* mistake — passing a string where a
number belongs.

```zig
const std = @import("std");

fn scaleAny(v: anytype, factor: anytype) @TypeOf(v) {
    return v * factor;
}

pub fn main() void {
    std.debug.print("{d}\n", .{scaleAny(@as(u8, 3), "oops")});
}
```

**Deliberate compile error:**

```
e1.zig:4:14: error: incompatible types: 'u8' and '*const [4:0]u8'
    return v * factor;
           ~~^~~~~~~~
e1.zig:4:12: note: type 'u8' here
    return v * factor;
           ^
e1.zig:4:16: note: type '*const [4:0]u8' here
```

```zig
const std = @import("std");

fn scale(comptime T: type, v: T, factor: T) T {
    return v * factor;
}

pub fn main() void {
    std.debug.print("{d}\n", .{scale(u8, 3, "oops")});
}
```

**Deliberate compile error:**

```
e2.zig:8:45: error: expected type 'u8', found '*const [4:0]u8'
    std.debug.print("{d}\n", .{scale(u8, 3, "oops")});
                                            ^~~~~~
e2.zig:3:42: note: parameter type declared here
fn scale(comptime T: type, v: T, factor: T) T {
                                         ^
```

The `anytype` version blames `v * factor` — a line in *your* library. The
explicit version underlines `"oops"` in the caller's own source and says what
was expected. That difference compounds across a codebase.

**Reach for `anytype` when:**
- the parameter is a *writer*, *reader*, *allocator-like* or other duck-typed thing
- repeating the type at the call site would be noise (`print(fmt, args)`)
- you accept a tuple or an anonymous struct literal

**Reach for `comptime T: type` when:**
- two or more parameters must be the *same* type
- `T` appears in the return type or a nested type like `[]const T`
- the function is public API and error quality matters
- the caller might reasonably want to *pick* the type (`sum(u64, small_values)`)

`std.ArrayList(T).append(gpa, item)` is the explicit form. `Writer.print(fmt,
args)` is `anytype`. Both are right for what they do.

---

Next: [6.4 — Generic Containers](./04-generic-containers.md)
