# 6.2 — Types Are Values

A type in Zig is a **value of type `type`**. Not a declaration, not syntax — a
value you can store, compare, pass and return.

```zig
const std = @import("std");

const MyInt = u32;
const Alias = MyInt;

pub fn main() void {
    std.debug.print("@TypeOf(u32) = {s}\n", .{@typeName(@TypeOf(MyInt))});
    std.debug.print("MyInt == Alias: {}\n", .{MyInt == Alias});
    std.debug.print("u32 == i32: {}\n", .{u32 == i32});

    const types = [_]type{ u8, i64, f32, bool, []const u8 };
    inline for (types) |T| {
        std.debug.print("{s:>12} size={d} align={d}\n", .{ @typeName(T), @sizeOf(T), @alignOf(T) });
    }
}
```

```
@TypeOf(u32) = type
MyInt == Alias: true
u32 == i32: false
          u8 size=1 align=1
         i64 size=8 align=8
         f32 size=4 align=4
        bool size=1 align=1
  []const u8 size=16 align=8
```

Three things to absorb from that output:

- `@TypeOf(u32)` is **`type`**. `u32` is a value; `type` is its type.
- `==` works on types. `MyInt == Alias` is `true` because `const MyInt = u32`
  stores a value — it does not create a new type.
- `[_]type{ ... }` is an **array of types**. Ordinary array syntax.

> `inline for` is required there, not plain `for`. The loop body uses `T` as a
> type, so the loop must be unrolled at compile time. A runtime `for` over types
> is meaningless and the compiler says so.

## Generics are just functions

A generic type is a function that takes types and returns a type. There is no
new syntax to learn — it is the function syntax you already know, with `type` as
the return type.

```zig
const std = @import("std");

fn Pair(comptime A: type, comptime B: type) type {
    return struct { a: A, b: B };
}

pub fn main() void {
    std.debug.print("{s}\n", .{@typeName(Pair(u8, u8))});
    std.debug.print("{s}\n", .{@typeName(Pair(u8, []const u8))});

    const p: Pair(u8, bool) = .{ .a = 7, .b = true };
    std.debug.print("p.a={d} p.b={}\n", .{ p.a, p.b });
}
```

```
t1.Pair(u8,u8)
t1.Pair(u8,[]const u8)
p.a=7 p.b=true
```

Compare the three languages the learner already knows:

| Language | Syntax | Mechanism |
|---|---|---|
| TypeScript | `type Pair<A, B> = { a: A; b: B }` | a separate type-level language |
| Rust | `struct Pair<A, B> { a: A, b: B }` | generics + a trait solver |
| Go | `type Pair[A, B any] struct { ... }` | type parameters, constrained |
| **Zig** | `fn Pair(comptime A: type, comptime B: type) type` | **a normal function** |

The angle brackets are gone because nothing needs them: `Pair(u8, bool)` is a
function call whose result is used where a type goes. Everything you can do to a
value — branch on it, loop over it, build it in a block — you can do to a type.

## The naming convention

`Pair` is capitalized because it **returns a type**. That is the whole rule, and
it is the only hint you get that `Pair(u8, bool)` is usable as a type:

| Case | Meaning |
|---|---|
| `TitleCase` | a type, or a function returning a type |
| `camelCase` | a function returning a value |
| `snake_case` | a variable or field |

`std.ArrayList`, `std.heap.DebugAllocator`, `std.meta.FieldEnum` — all
capitalized, all type-returning functions.

## Type identity is by instantiation, and it's cached

This is the single most important property. **Calling a type function twice with
the same arguments gives you the identical type**, not a copy.

```zig
const std = @import("std");

fn Pair(comptime A: type, comptime B: type) type {
    return struct { a: A, b: B };
}

pub fn main() void {
    std.debug.print("Pair(u8,u8) == Pair(u8,u8):  {}\n", .{Pair(u8, u8) == Pair(u8, u8)});
    std.debug.print("Pair(u8,u8) == Pair(u8,u16): {}\n", .{Pair(u8, u8) == Pair(u8, u16)});

    const p: Pair(u8, bool) = .{ .a = 7, .b = true };
    const q: Pair(u8, bool) = .{ .a = 9, .b = false };
    std.debug.print("same type, assignable: {}\n", .{@TypeOf(p) == @TypeOf(q)});
}
```

```
Pair(u8,u8) == Pair(u8,u8):  true
Pair(u8,u8) == Pair(u8,u16): false
same type, assignable: true
```

Without that caching, `Pair(u8, u8)` in two files would be two incompatible
types and generics would be useless. The compiler memoizes every type function
on its arguments.

Different arguments mean genuinely different types, and mixing them is an error:

```zig
const std = @import("std");

fn Pair(comptime A: type, comptime B: type) type {
    return struct { a: A, b: B };
}

pub fn main() void {
    const p: Pair(u8, bool) = .{ .a = 7, .b = true };
    const q: Pair(u8, u16) = p;
    std.debug.print("{d}\n", .{q.a});
}
```

**Deliberate compile error:**

```
t3.zig:9:30: error: expected type 't3.Pair(u8,u16)', found 't3.Pair(u8,bool)'
    const q: Pair(u8, u16) = p;
                             ^
t3.zig:4:12: note: struct declared here (2 times)
    return struct { a: A, b: B };
           ^~~~~~~~~~~~~~~~~~~~~
```

Two details in that message are worth pausing on:

- The type is named **`t3.Pair(u8,bool)`** — module, function, arguments. Zig
  builds the name from the call, which is why error messages stay readable.
- **`struct declared here (2 times)`** — one `struct` expression in your source,
  two types instantiated from it. That is the whole generics model in a note.

## `@TypeOf` and `@typeName`

```zig
const std = @import("std");

const Color = enum { red, green, blue };
const Point = struct { x: i32, y: i32 };

pub fn main() void {
    var n: u32 = 5;
    n += 0;
    const arr = [_]i32{ 1, 2, 3, 4, 5 };
    const slice: []const i32 = &arr;

    std.debug.print("{s}\n", .{@typeName(@TypeOf(n))});
    std.debug.print("{s}\n", .{@typeName(@TypeOf(1))});
    std.debug.print("{s}\n", .{@typeName(@TypeOf(1.0))});
    std.debug.print("{s}\n", .{@typeName(@TypeOf(arr))});
    std.debug.print("{s}\n", .{@typeName(@TypeOf(arr[1..4]))});
    std.debug.print("{s}\n", .{@typeName(@TypeOf(slice))});
    std.debug.print("{s}\n", .{@typeName(@TypeOf(Color.red))});
    std.debug.print("{s}\n", .{@typeName(@TypeOf(.{ .x = 1, .y = 2 }))});
    std.debug.print("{s}\n", .{@typeName(Point)});
    std.debug.print("{s}\n", .{@typeName(?*Point)});
    std.debug.print("{s}\n", .{@typeName(anyerror!void)});
}
```

```
u32
comptime_int
comptime_float
[5]i32
*const [3]i32
[]const i32
tp.Color
tp.main__struct_861
tp.Point
?*tp.Point
anyerror!void
```

Four of those surprise people:

| Expression | Type | Why |
|---|---|---|
| `@TypeOf(1)` | `comptime_int` | an untyped literal, arbitrary precision until it's assigned |
| `@TypeOf(arr[1..4])` | **`*const [3]i32`** | comptime-known bounds give a *pointer to array*, not a slice |
| `@TypeOf(.{...})` | `tp.main__struct_861` | an anonymous struct gets a compiler-generated name (the number varies) |
| `@typeName(Point)` | `tp.Point` | qualified by the file it's declared in |

That generated name is a useful tell: a type the compiler had to invent a name
for is a type **you cannot name**, so you cannot write it in a signature.
Anonymous struct literals only coerce to a named struct at the call site — bind
one to a `const` first and it stays `main__struct_861` forever.

`@TypeOf` never evaluates its argument — it's resolved entirely at compile time,
so `@TypeOf(someExpensiveCall())` costs nothing and calls nothing.

## Types returning types that carry methods

The struct a type function returns is a normal struct, so it can hold `const`
declarations and methods. `@This()` names the struct being defined:

```zig
const std = @import("std");

fn Tagged(comptime T: type) type {
    return struct {
        value: T,
        tag: []const u8,

        const Self = @This();

        fn show(s: Self) void {
            std.debug.print("{s}<{s}> = {any}\n", .{ s.tag, @typeName(T), s.value });
        }
    };
}

pub fn main() void {
    const ti: Tagged(i32) = .{ .value = -5, .tag = "count" };
    const tf: Tagged(f64) = .{ .value = 2.5, .tag = "ratio" };
    ti.show();
    tf.show();
}
```

```
count<i32> = -5
ratio<f64> = 2.5
```

`T` is in scope inside the struct body and inside its methods — the returned
struct **closes over** the function's comptime parameters. That is what makes
this work at all, and 6.4 builds real containers on it.

## Dispatching on a type: `switch` on `@typeInfo`

Since types are values, you can compute with them. A type function that picks
different layouts per input:

```zig
const std = @import("std");

fn Storage(comptime T: type) type {
    return switch (@typeInfo(T)) {
        .bool => packed struct { bits: u8 = 0 },
        .int => |i| if (i.bits <= 16)
            struct { small: T, kind: []const u8 = "inline" }
        else
            struct { big: T, kind: []const u8 = "boxed" },
        else => @compileError("no storage strategy for " ++ @typeName(T)),
    };
}

pub fn main() void {
    const b: Storage(bool) = .{};
    const s: Storage(u8) = .{ .small = 3 };
    const l: Storage(u64) = .{ .big = 9 };

    std.debug.print("bool -> {s} ({d} bytes)\n", .{ @typeName(@TypeOf(b)), @sizeOf(@TypeOf(b)) });
    std.debug.print("u8   -> {s} kind={s}\n", .{ @typeName(@TypeOf(s)), s.kind });
    std.debug.print("u64  -> {s} kind={s}\n", .{ @typeName(@TypeOf(l)), l.kind });
}
```

```
bool -> st.Storage__struct_854 (1 bytes)
u8   -> st.Storage__struct_855 kind=inline
u64  -> st.Storage__struct_856 kind=boxed
```

A `switch` returning different `struct` definitions. In Rust this needs
associated types and a trait; in Zig it is a `switch` expression.

Note the names: `Storage__struct_854`, not `Storage(bool)`. Compare with `Pair`
earlier, which printed `t1.Pair(u8,u8)`. The naming rule is narrow and worth
knowing, because it shows up in every error message about your type:

```zig
const std = @import("std");

fn Direct(comptime T: type) type {
    return struct { v: T };
}

fn WithDecls(comptime T: type) type {
    return struct {
        v: T,
        const Self = @This();
        fn get(s: Self) T {
            return s.v;
        }
    };
}

fn ViaIf(comptime T: type) type {
    if (@sizeOf(T) > 100) return struct { big: T };
    return struct { v: T };
}

fn ViaBlock(comptime T: type) type {
    const X = struct { v: T };
    return X;
}

fn ViaSwitch(comptime T: type) type {
    return switch (@typeInfo(T)) {
        .int => struct { v: T },
        else => struct { w: T },
    };
}

pub fn main() void {
    std.debug.print("Direct    {s}\n", .{@typeName(Direct(u8))});
    std.debug.print("WithDecls {s}\n", .{@typeName(WithDecls(u8))});
    std.debug.print("ViaIf     {s}\n", .{@typeName(ViaIf(u8))});
    std.debug.print("ViaBlock  {s}\n", .{@typeName(ViaBlock(u8))});
    std.debug.print("ViaSwitch {s}\n", .{@typeName(ViaSwitch(u8))});
}
```

```
Direct    nm.Direct(u8)
WithDecls nm.WithDecls(u8)
ViaIf     nm.ViaIf(u8)
ViaBlock  nm.ViaBlock.X
ViaSwitch nm.ViaSwitch__struct_857
```

| Form | Name | Rule |
|---|---|---|
| `return struct {...}` directly | `Direct(u8)` ✅ | a `return` of a struct **literal** keeps the pretty name |
| plus methods and `const Self` | `WithDecls(u8)` ✅ | declarations don't matter |
| several `return struct {...}` | `ViaIf(u8)` ✅ | still a direct return, in both branches |
| `const X = struct{...}; return X;` | `ViaBlock.X` ⚠️ | named after the binding, losing the arguments |
| `return switch (...) {...}` | `ViaSwitch__struct_857` ❌ | the struct isn't the `return` operand |

(The `__struct_857` number is an internal counter — expect a different one, since
it shifts with everything else in the file.)

So prefer a direct `return struct { ... }` — several of them if you must branch
with `if`. A `switch` that yields struct literals costs you readable type names
everywhere they appear.

## Why this replaces template syntax

| Thing other languages need | Zig uses |
|---|---|
| `<T>` / `[T any]` parameter lists | function parameters |
| trait bounds / `where` clauses | `if` + `@compileError` (6.3) |
| specialization | `switch` on `@typeInfo` |
| `static_assert` | `std.debug.assert` |
| associated types | a `const` inside the returned struct |
| reflection / `derive` macros | `@typeInfo` + `inline for` (6.5) |
| a build-time codegen script | a `comptime` block |

One language, used twice. That's the differentiator — and the reason the rest of
this module is mostly ordinary-looking Zig.

---

Next: [6.3 — Generic Functions](./03-generic-functions.md)
