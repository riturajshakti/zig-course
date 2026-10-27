# 3.7 — Tagged Unions

A `union(enum)` is Zig's sum type: **one of these, and the compiler knows
which.** If you have written `enum Shape { Circle(f64), Rect { w, h } }` in Rust
or a discriminated union in TypeScript, this is the same idea with no runtime
cost you did not ask for.

```zig
const std = @import("std");

const Value = union(enum) {
    none,
    int: i64,
    float: f64,
    text: []const u8,
    pair: struct { a: i32, b: i32 },
};

fn describe(v: Value) void {
    switch (v) {
        .none => std.debug.print("none\n", .{}),
        .int => |n| std.debug.print("int {d}\n", .{n}),
        .float => |f| std.debug.print("float {d}\n", .{f}),
        .text => |s| std.debug.print("text \"{s}\" ({d} bytes)\n", .{ s, s.len }),
        .pair => |p| std.debug.print("pair {d},{d}\n", .{ p.a, p.b }),
    }
}

pub fn main() void {
    const vals = [_]Value{
        .none,
        .{ .int = -7 },
        .{ .float = 2.5 },
        .{ .text = "hello" },
        .{ .pair = .{ .a = 1, .b = 2 } },
    };
    for (vals) |v| describe(v);

    std.debug.print("\n@sizeOf(Value) = {d}\n", .{@sizeOf(Value)});
    std.debug.print("activeTag(vals[1]) = {t}\n", .{std.meta.activeTag(vals[1])});
    std.debug.print("vals[3] == .text ? {}\n", .{vals[3] == .text});
}
```

```
none
int -7
float 2.5
text "hello" (5 bytes)
pair 1,2

@sizeOf(Value) = 24
activeTag(vals[1]) = int
vals[3] == .text ? true
```

| Piece | Meaning |
|---|---|
| `union(enum)` | "generate the tag enum for me from the field names" |
| `none,` with no type | a payload-less variant — `void` |
| `text: []const u8` | this variant carries a slice |
| `pair: struct {...}` | an inline anonymous struct as the payload |
| `.{ .int = -7 }` | construct by naming **one** field |
| `.none` | construct a payload-less variant |
| `\|n\|` in a switch arm | **capture** the payload of that variant |
| `std.meta.activeTag(v)` | which variant is live, as the tag enum |
| `v == .text` | tag comparison — no payload needed |

`@sizeOf(Value)` is 24: the largest payload (`[]const u8`, 16 bytes) plus the tag,
rounded to alignment. **A union is as big as its biggest variant** — there is no
heap indirection, unlike a Rust `Box<dyn>` or a Java subclass.

## Capturing by pointer: `|*v|`

Capture with `|*p|` to mutate the payload in place.

```zig
const std = @import("std");

const Shape = union(enum) {
    const Self = @This();

    circle: f64,
    square: f64,
    rect: struct { w: f64, h: f64 },

    pub fn area(self: Self) f64 {
        return switch (self) {
            .circle => |r| std.math.pi * r * r,
            .square => |s| s * s,
            .rect => |r| r.w * r.h,
        };
    }

    pub fn scale(self: *Self, by: f64) void {
        switch (self.*) {
            .circle => |*r| r.* *= by,
            .square => |*s| s.* *= by,
            .rect => |*r| {
                r.w *= by;
                r.h *= by;
            },
        }
    }

    pub fn format(self: Self, w: *std.Io.Writer) !void {
        switch (self) {
            .circle => |r| try w.print("circle(r={d})", .{r}),
            .square => |s| try w.print("square({d})", .{s}),
            .rect => |r| try w.print("rect({d}x{d})", .{ r.w, r.h }),
        }
    }
};

pub fn main() void {
    var shapes = [_]Shape{
        .{ .circle = 1.0 },
        .{ .square = 2.0 },
        .{ .rect = .{ .w = 2.0, .h = 3.0 } },
    };

    for (&shapes) |*s| {
        std.debug.print("{f} area={d:.3}\n", .{ s.*, s.area() });
        s.scale(2.0);
        std.debug.print("  scaled -> {f} area={d:.3}\n", .{ s.*, s.area() });
    }
}
```

```
circle(r=1) area=3.142
  scaled -> circle(r=2) area=12.566
square(2) area=4.000
  scaled -> square(4) area=16.000
rect(2x3) area=6.000
  scaled -> rect(4x6) area=24.000
```

Unions take methods, `@This()`, and a `format` for `{f}` exactly like structs and
enums. `switch (self.*)` with `|*r|` captures a pointer **into** the union's
payload — note `scale` mutates in place and never rebuilds the value.

Switching on a union is exhaustive just like an enum: omit a variant and the
build fails.

## Payload-less variants

A variant with no type is `void` — it costs nothing but the tag, and it is how
you model states that carry no data.

```zig
const std = @import("std");

const Token = union(enum) {
    number: f64,
    ident: []const u8,
    plus,
    minus,
    eof,
};

fn render(t: Token) void {
    switch (t) {
        .number => |n| std.debug.print("NUMBER({d})", .{n}),
        .ident => |s| std.debug.print("IDENT({s})", .{s}),
        .plus => std.debug.print("+", .{}),
        .minus => std.debug.print("-", .{}),
        .eof => std.debug.print("EOF", .{}),
    }
}

pub fn main() void {
    const toks = [_]Token{
        .{ .ident = "x" },
        .plus,
        .{ .number = 3.5 },
        .minus,
        .{ .number = 1 },
        .eof,
    };

    for (toks) |t| {
        render(t);
        std.debug.print(" ", .{});
    }
    std.debug.print("\n", .{});

    std.debug.print("tags: ", .{});
    inline for (std.meta.tags(std.meta.Tag(Token))) |t| std.debug.print("{t} ", .{t});
    std.debug.print("\n", .{});

    var numbers: usize = 0;
    for (toks) |t| if (t == .number) {
        numbers += 1;
    };
    std.debug.print("number tokens = {d}\n", .{numbers});
    std.debug.print("@sizeOf(Token) = {d}\n", .{@sizeOf(Token)});
}
```

```
IDENT(x) + NUMBER(3.5) - NUMBER(1) EOF 
tags: number ident plus minus eof 
number tokens = 2
@sizeOf(Token) = 24
```

`std.meta.Tag(Token)` is the generated tag enum, so everything from lesson 3.6
(`std.meta.tags`, `stringToEnum`, `EnumArray`) works on it. And `t == .number`
checks the tag without unwrapping — handy for filtering.

This is exactly how you build a lexer or an AST node in Zig.

## An explicit tag enum: `union(Kind)`

When the tag's integer values matter — a wire format, a file header, a C enum —
declare the enum yourself and name it in the union.

```zig
const std = @import("std");

const Kind = enum(u8) { text = 1, blob = 2, empty = 3 };

const Payload = union(Kind) {
    text: []const u8,
    blob: []const u8,
    empty,
};

pub fn main() void {
    const items = [_]Payload{
        .{ .text = "hi" },
        .{ .blob = "\x00\x01" },
        .empty,
    };

    for (items) |p| {
        const k = std.meta.activeTag(p);
        std.debug.print("tag={t} wire_byte={d} ", .{ k, @intFromEnum(k) });
        switch (p) {
            .text => |s| std.debug.print("text=\"{s}\"\n", .{s}),
            .blob => |b| std.debug.print("blob={any}\n", .{b}),
            .empty => std.debug.print("(no payload)\n", .{}),
        }
    }

    std.debug.print("Tag(Payload) == Kind ? {}\n", .{std.meta.Tag(Payload) == Kind});
}
```

```
tag=text wire_byte=1 text="hi"
tag=blob wire_byte=2 blob={ 0, 1 }
tag=empty wire_byte=3 (no payload)
Tag(Payload) == Kind ? true
```

`union(Kind)` requires the union's field names to match `Kind`'s exactly. Now
the discriminant byte on the wire is yours to control, and `@intFromEnum` reads
it back.

## ⚠️ Bare `union` vs `union(enum)`

A bare `union` has **no tag**. It is C's union: overlapping storage, and *you*
must remember which field is live.

```zig
const std = @import("std");

const Bare = union { int: u32, float: f32 };
const Tagged = union(enum) { int: u32, float: f32 };

pub fn main() void {
    const b: Bare = .{ .int = 1065353216 };
    const t: Tagged = .{ .int = 1065353216 };

    std.debug.print("mode            = {t}\n", .{@import("builtin").mode});
    std.debug.print("@sizeOf(Bare)   = {d}\n", .{@sizeOf(Bare)});
    std.debug.print("@sizeOf(Tagged) = {d}\n", .{@sizeOf(Tagged)});
    std.debug.print("b.int = {d}\n", .{b.int});
    std.debug.print("activeTag(t) = {t}\n", .{std.meta.activeTag(t)});
}
```

```sh
zig run u3.zig
```

```
mode            = debug
@sizeOf(Bare)   = 8
@sizeOf(Tagged) = 8
b.int = 1065353216
activeTag(t) = int
```

```sh
zig run -OReleaseFast u3.zig
```

```
mode            = fast
@sizeOf(Bare)   = 4
@sizeOf(Tagged) = 8
b.int = 1065353216
activeTag(t) = int
```

**Look at `@sizeOf(Bare)`: 8 in Debug, 4 in ReleaseFast.** Zig adds a *hidden
safety tag* to bare unions in Debug and ReleaseSafe so it can catch misuse, then
drops it in ReleaseFast. That is the only thing standing between you and silent
garbage — and it is a build-mode flag away from disappearing.

Reading the wrong field is caught when the compiler can see it:

```zig
const std = @import("std");

const Bare = union { int: u32, float: f32 };

pub fn main() void {
    const b: Bare = .{ .int = 1065353216 };
    std.debug.print("read as float: {d}\n", .{b.float});
}
```

```
u4.zig:7:48: error: access of union field 'float' while field 'int' is active
    std.debug.print("read as float: {d}\n", .{b.float});
                                              ~^~~~~~
u4.zig:3:14: note: union declared here
const Bare = union { int: u32, float: f32 };
```

Hide the value behind a function and the compile-time check is gone — now it
depends on the build mode:

```zig
const std = @import("std");

const Bare = union { int: u32, float: f32 };

fn makeInt(n: u32) Bare {
    return .{ .int = n };
}

pub fn main() void {
    var n: u32 = 1065353216;
    _ = &n;
    const b = makeInt(n);
    std.debug.print("read as float: {d}\n", .{b.float});
}
```

```sh
zig run u4b.zig
```

```
thread 16493102 panic: access of union field 'float' while field 'int' is active
u4b.zig:13:48: 0x1001d9237 in main (u4b)
    std.debug.print("read as float: {d}\n", .{b.float});
                                               ^
```

```sh
zig run -OReleaseFast u4b.zig
```

```
read as float: 1
```

Deliberate, and this is the whole lesson. **Same program, same input: a panic in
Debug, the number `1` in ReleaseFast.** (The thread id and `0x...` address in the
panic differ on every run; the message and source location do not.) `1065353216` is the IEEE-754 bit pattern
for `1.0`, so the union happily reinterpreted the bytes. No error, no warning —
exactly the C bug Zig's tag exists to prevent.

And a bare union cannot be `switch`ed at all:

```zig
const std = @import("std");

const Bare = union { int: u32, float: f32 };

pub fn main() void {
    const b: Bare = .{ .int = 1 };
    switch (b) {
        .int => |n| std.debug.print("{d}\n", .{n}),
        .float => |f| std.debug.print("{d}\n", .{f}),
    }
}
```

```
u5.zig:7:13: error: switch on union with no attached enum
    switch (b) {
            ^
u5.zig:3:14: note: consider 'union(enum)' here
const Bare = union { int: u32, float: f32 };
```

The compiler even tells you the fix.

| | `union(enum)` | bare `union` |
|---|---|---|
| tag stored | yes, always | only a hidden one in Debug/ReleaseSafe |
| size | payload + tag | payload only (in ReleaseFast) |
| `switch` | **yes, exhaustive** | compile error |
| `std.meta.activeTag` | yes | no |
| `v == .tag` | yes | no |
| wrong-field read | impossible | **UB in ReleaseFast** |
| use it for | **everything** | C ABI, deliberate type punning |

**Write `union(enum)`.** A bare `union` is for `extern`/C interop; for
reinterpreting bytes, `@bitCast` says what you mean and is checked.

## How this replaces Rust enums and TS discriminated unions

```rust
// Rust
enum Value { Int(i64), Text(String), None }
match v {
    Value::Int(n) => ...,
    Value::Text(s) => ...,
    Value::None => ...,
}
```

```typescript
// TypeScript
type Value =
  | { kind: "int"; value: number }
  | { kind: "text"; value: string }
  | { kind: "none" };
switch (v.kind) { case "int": v.value; ... }
```

```zig
// Zig
const Value = union(enum) { int: i64, text: []const u8, none };
switch (v) {
    .int => |n| ...,
    .text => |s| ...,
    .none => ...,
}
```

| | Rust | TypeScript | Zig |
|---|---|---|---|
| exhaustiveness | compile error | compile error (with care) | **compile error** |
| payload access | pattern binding | narrowed property | `\|capture\|` |
| discriminant | hidden, niche-optimized | a **real string field** | an integer tag you can name |
| runtime cost | tag + largest payload | an object + a string per value | tag + largest payload |
| heap | `String` allocates | everything allocates | **nothing unless you ask** |

Compared to **Rust**: structurally the same, including exhaustive matching. Zig
has no niche optimization (`?*T` aside), no `impl` blocks — methods go in the
union body — and no destructors, so a variant holding allocated memory needs a
`deinit` you call yourself.

Compared to **TypeScript**: Zig's tag is an integer, not a string you must keep
typing correctly, and the exhaustiveness check cannot be defeated by a stray
`default:`. Nothing is allocated per value.

What Zig lacks is **nested destructuring**: `.pair => |p| p.a` gets you one level
and you go on from there, where Rust could write `Value::Pair{a: 1, b}`.

## Operations table

| Operation | Code |
|---|---|
| declare, inferred tag | `const U = union(enum) { a: i32, b: []const u8, c };` |
| declare, explicit tag | `const U = union(MyEnum) { ... };` |
| declare, untagged | `const U = union { ... };` — **avoid** |
| payload-less variant | `c,` with no type |
| construct | `const u: U = .{ .a = 1 };` |
| construct payload-less | `const u: U = .c;` |
| which variant | `std.meta.activeTag(u)` |
| test the tag | `u == .a` |
| the tag enum type | `std.meta.Tag(U)` |
| read a payload | `switch (u) { .a => \|n\| ..., ... }` |
| mutate a payload | `switch (u.*) { .a => \|*p\| p.* = ..., ... }` |
| replace the whole value | `u = .{ .b = "hi" };` |
| single-variant test | `if (u == .a) ... ;` |
| capture only if it matches | `if (u == .a) u.a` |
| methods | declare in the union body, `const Self = @This();` |
| custom printing | `pub fn format(self: U, w: *std.Io.Writer) !void` → `{f}` |
| size | `@sizeOf(U)` — largest payload + tag |
| all tag names | `inline for (std.meta.tags(std.meta.Tag(U)))` |
| tag's integer | `@intFromEnum(std.meta.activeTag(u))` |

Two things unions do **not** give you: destructors (a variant holding heap memory
needs an explicit `deinit`) and recursion by value. A self-referencing union
needs a pointer:

```zig
const Node = union(enum) {
    leaf: i32,
    branch: struct { left: *Node, right: *Node },
};
```

Without the `*` the type would be infinitely large:

```
error: dependency loop with length 2
    t2.zig:2:46: note: type 't2.Bad' depends on type 't2.Bad__struct_854' for field declared here
    const Bad = union(enum) { leaf: i32, branch: struct { left: Bad, right: Bad } };
                                                 ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

---

That is module 03. Next up: how Zig handles failure and absence — error sets,
`try`, `catch`, `defer`, and optionals.

Next: [Module 04 — Errors & Optionals](../04-errors-optionals/)
