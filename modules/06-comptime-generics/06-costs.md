# 6.6 — comptime Costs

Comptime is not free. It moves work from runtime to **compile time** and from
one function to **one per type**. Both are measurable, and this lesson measures
them on this machine.

## One instantiation per type

```zig
const std = @import("std");

fn Box(comptime T: type) type {
    return struct {
        v: T,
        fn get(b: @This()) T {
            return b.v;
        }
    };
}

pub fn main() void {
    const a: Box(u8) = .{ .v = 1 };
    const b: Box(u16) = .{ .v = 2 };
    const c: Box(u8) = .{ .v = 3 };
    std.debug.print("Box(u8) and Box(u16) are the same type: {}\n", .{Box(u8) == Box(u16)});
    std.debug.print("Box(u8) twice is cached:                {}\n", .{Box(u8) == Box(u8)});
    std.debug.print("{s} {s} {s}\n", .{ @typeName(@TypeOf(a)), @typeName(@TypeOf(b)), @typeName(@TypeOf(c)) });
    std.debug.print("values {d} {d} {d}\n", .{ a.get(), b.get(), c.get() });
}
```

```
Box(u8) and Box(u16) are the same type: false
Box(u8) twice is cached:                true
inst.Box(u8) inst.Box(u16) inst.Box(u8)
values 1 2 3
```

Caching is per **argument set**, so the cost scales with the number of distinct
instantiations, not the number of call sites. Three uses, two compiled copies of
`Box` and `get`.

You can watch the compiler do it. From 6.1, `@compileLog` fires once per
instantiation:

```zig
const std = @import("std");

fn sizeTag(comptime T: type) []const u8 {
    @compileLog("inspecting ", @typeName(T), @sizeOf(T));
    return @typeName(T);
}

pub fn main() void {
    std.debug.print("{s} {s}\n", .{ sizeTag(u8), sizeTag(u64) });
}
```

**Deliberate compile error** (`@compileLog` always fails the build):

```
cl.zig:4:5: error: found compile log statement
    @compileLog("inspecting ", @typeName(T), @sizeOf(T));
    ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
cl.zig:4:5: note: also here

Compile Log Output:
@as(*const [11:0]u8, "inspecting "), @as(*const [3:0]u8, "u64"), @as(comptime_int, 8)
@as(*const [11:0]u8, "inspecting "), @as(*const [2:0]u8, "u8"), @as(comptime_int, 1)
```

Two lines = two compiled bodies. This is the cheapest way to audit how many
instantiations a generic function really has.

## Measuring code bloat

Two programs doing the same amount of work. One calls a generic function at 56
different integer widths; the other calls a single `u64` function 56 times.

```zig
const std = @import("std");

fn sumGeneric(comptime T: type, vals: []const T) T {
    var acc: T = 0;
    for (vals) |v| acc +%= v;
    return acc;
}

pub fn main() void {
    var total: u64 = 0;
    inline for (8..64) |bits| {
        const T = @Int(.unsigned, bits);
        var vals = [_]T{ 1, 2, 3 };
        vals[0] +%= @intCast(bits % 3);
        total += sumGeneric(T, &vals);
    }
    std.debug.print("{d}\n", .{total});
}
```

```zig
const std = @import("std");

fn sumU64(vals: []const u64) u64 {
    var acc: u64 = 0;
    for (vals) |v| acc +%= v;
    return acc;
}

pub fn main() void {
    var total: u64 = 0;
    inline for (8..64) |bits| {
        var vals = [_]u64{ 1, 2, 3 };
        vals[0] +%= @intCast(bits % 3);
        total += sumU64(&vals);
    }
    std.debug.print("{d}\n", .{total});
}
```

Both print `392`. Built with `zig build-exe -O Debug`, three runs each with the
cache cleared:

| | compile | binary |
|---|---|---|
| `sumGeneric` — 56 instantiations | 1103 / 1110 / 1089 ms | **2,026,648 bytes** |
| `sumU64` — 1 instantiation | 1083 / 1088 / 1081 ms | **2,001,608 bytes** |

**+25,040 bytes** for 55 extra copies — about 455 bytes each, and perfectly
reproducible across runs. Compile time barely moved; this is a *size* cost.

> `@Int(.unsigned, bits)` builds an integer type from a width. In 0.17 this
> replaced `@Type(.{ .int = ... })` — `@Type` no longer exists and reports
> `invalid builtin function: '@Type'`.

Scale that honestly: 455 bytes per instantiation is nothing for 3 types and real
for 300. The usual culprit is a container generic over a type *parameter you
didn't need to vary* — `Map(K, V)` instantiated at 40 value types when the
values could have been `[]const u8`.

## Measuring compile time

Code bloat is the small cost. **Running your program inside the compiler** is
the big one. Same Collatz search, once at comptime and once at runtime:

```zig
const std = @import("std");

fn collatzLen(start: u64) u64 {
    var n = start;
    var steps: u64 = 0;
    while (n != 1) {
        n = if (n % 2 == 0) n / 2 else 3 * n + 1;
        steps += 1;
    }
    return steps;
}

const longest = blk: {
    @setEvalBranchQuota(100_000_000);
    var best: u64 = 0;
    var best_n: u64 = 0;
    var i: u64 = 1;
    while (i < 50_000) : (i += 1) {
        const l = collatzLen(i);
        if (l > best) {
            best = l;
            best_n = i;
        }
    }
    break :blk .{ best_n, best };
};

pub fn main() void {
    std.debug.print("longest collatz under 50000: n={d} steps={d}\n", .{ longest[0], longest[1] });
}
```

```zig
const std = @import("std");

fn collatzLen(start: u64) u64 {
    var n = start;
    var steps: u64 = 0;
    while (n != 1) {
        n = if (n % 2 == 0) n / 2 else 3 * n + 1;
        steps += 1;
    }
    return steps;
}

pub fn main() void {
    var best: u64 = 0;
    var best_n: u64 = 0;
    var i: u64 = 1;
    while (i < 50_000) : (i += 1) {
        const l = collatzLen(i);
        if (l > best) {
            best = l;
            best_n = i;
        }
    }
    std.debug.print("longest collatz under 50000: n={d} steps={d}\n", .{ best_n, best });
}
```

Both print `longest collatz under 50000: n=35655 steps=323`.

| | compile (2 runs) | run |
|---|---|---|
| comptime version | **11,654 / 11,897 ms** | 440 ms |
| runtime version | **1,093 / 1,112 ms** | 445 ms |

**~10.6 seconds of compile time to save 5 milliseconds of runtime.** The
comptime interpreter is roughly two orders of magnitude slower than the compiled
code it replaces, because it evaluates an IR tree instead of executing machine
instructions.

And the cache does not save you — rebuilding without clearing it still took
**11,660 ms**. You pay on every build that touches the file.

| Comptime computation is worth it when | Example |
|---|---|
| the result is needed *as a type* | `Stack(T)`, `RingBuffer(u8, 3)` |
| the result is a small table reused often | a CRC table, a parser's jump table |
| it runs once and the inputs never change | a compile-time config check |
| **not** worth it when | |
| the work is large and the answer is one value | this Collatz search |
| it runs at startup anyway, cheaply | reading a config file |
| it bloats the binary with data | see below |

### Comptime data becomes binary bytes

```zig
const std = @import("std");

const table = blk: {
    @setEvalBranchQuota(10_000_000);
    var t: [20000]u64 = undefined;
    for (&t, 0..) |*slot, i| slot.* = i * i;
    break :blk t;
};

pub fn main() void {
    var acc: u64 = 0;
    for (table[0..1000]) |v| acc += v;
    std.debug.print("{d}\n", .{acc});
}
```

against the same answer computed at runtime, with no table at all:

```zig
const std = @import("std");

pub fn main() void {
    var acc: u64 = 0;
    for (0..1000) |i| acc += i * i;
    std.debug.print("{d}\n", .{acc});
}
```

Both print `332833500`.

| | compile | binary |
|---|---|---|
| with the 20,000-entry table | 1112 ms | **2,149,296 bytes** |
| computed at runtime, no table | 1073 ms | **2,001,488 bytes** |

**+147,808 bytes** ≈ 20,000 × 8 bytes of `u64`, straight into the executable. A
comptime table isn't free memory — it's `.rodata` you ship and page in. Past a
few KB, generating it at startup is often the better trade.

## `anytype` vs `comptime T: type`

Neither is cheaper at runtime — both monomorphize identically. The difference is
**error quality and API clarity**, which 6.3 showed and is worth restating as a
cost:

| | `anytype` | `comptime T: type` |
|---|---|---|
| runtime cost | identical | identical |
| instantiations | one per argument type | one per argument type |
| errors point at | your function body | the caller's call site |
| the contract is | invisible, discovered by compiling | partly in the signature |
| call site reads | `scale(3, 2)` | `scale(u8, 3, 2)` |

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

The first error is in a file the caller may not own. In a library with a deep
call stack, `anytype` reproduces C++'s template-error problem. Add the
`@compileError` guards from 6.3 and you get the diagnostic back.

## When NOT to be generic

The honest list. Each of these is a case where the non-generic version is better
code:

| Situation | Do this instead |
|---|---|
| exactly one type uses it today | write it concretely; generalize on the second caller |
| the types differ but the *logic* differs too | two functions with clear names |
| you need runtime polymorphism (plugins, heterogeneous lists) | a tagged union, or a vtable like `std.mem.Allocator` |
| the type is always a byte buffer | `[]const u8` — don't write `[]const T` |
| only *one* parameter genuinely varies | don't add a second type parameter "for symmetry" |
| it's a one-off script | concrete, every time |

**Runtime polymorphism is the important row.** Generics are resolved at compile
time, so they cannot hold a list of different types. When you need that, Zig's
answer is a tagged union (closed set, known at compile time) or a manual vtable
— which is exactly what `std.mem.Allocator` is, and why it is 16 bytes of
`ptr` + `vtable` instead of `Allocator(T)`.

```zig
const std = @import("std");

const Shape = union(enum) {
    circle: f64,
    square: f64,

    fn area(s: Shape) f64 {
        return switch (s) {
            .circle => |r| 3.14159265 * r * r,
            .square => |side| side * side,
        };
    }
};

pub fn main() void {
    const shapes = [_]Shape{ .{ .circle = 1.0 }, .{ .square = 2.0 }, .{ .circle = 0.5 } };
    var total: f64 = 0;
    for (shapes) |s| total += s.area();
    std.debug.print("{d} shapes in one array, total area {d:.4}\n", .{ shapes.len, total });
    std.debug.print("@sizeOf(Shape) = {d}\n", .{@sizeOf(Shape)});
}
```

```
3 shapes in one array, total area 7.9270
@sizeOf(Shape) = 16
```

One array, two shape kinds. A generic `Shape(T)` could not have done this —
`Shape(Circle)` and `Shape(Square)` would be different types with no common
array element type.

## The readability tradeoff

Comptime-heavy code is harder to read, and the costs are specific:

- **Jump-to-definition degrades.** `s.push(gpa, v)` lands inside an anonymous
  `struct` returned by a function. Your editor can't always tell you which
  instantiation you're looking at.
- **Error messages get a `referenced by:` stack** instead of a single line, and
  the first frame is often inside a library.
- **Type names grow.** `c3.Stack(c3.Stack(u8))` is still readable;
  `HashMap(u64, ArrayList(Pair(u8, []const u8)), AutoContext(u64), 80)` is not.
- **A `switch` on `@typeInfo` loses the pretty type name entirely** (6.2) —
  `Storage__struct_854` in every message about it.
- **Reflection code breaks across compiler versions.** The `.fields` →
  `field_names`/`field_types` change in 0.17 broke essentially every
  reflection-using library in the ecosystem.

That last one deserves weight. Reflection is the least stable part of the
language. Use it for the high-leverage cases — serialization, comparison,
validation, test harnesses — where one generic function replaces one per type.
Don't use it where an explicit function would do.

## Rules of thumb

| Rule | Why |
|---|---|
| write it concrete first, generalize at the second caller | the generic shape is obvious once you've seen two uses |
| `comptime T: type` for public API, `anytype` for duck-typed parameters | error quality lands on the caller |
| guard `anytype` with `@compileError` when others call it | turns a body error into a contract error |
| keep comptime computation small — tables, not searches | 10.6 s of compile time per 5 ms saved |
| measure the binary when instantiations grow past a handful | ~455 bytes each adds up |
| count instantiations with `@compileLog` when a build slows down | one log line per compiled copy |
| reach for a tagged union when the set of types is closed | no monomorphization, and it works at runtime |

---

That's the module. You can now read `std.ArrayList`, `std.HashMap` and
`std.meta` as ordinary Zig — because that is all they are.

Next: [Module 07 — Standard Library Tour](../07-stdlib-tour/)
