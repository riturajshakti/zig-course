# 6.5 — Reflection

`@typeInfo(T)` hands you a type's complete structure as a comptime value. Walk
it with `inline for` and you get serializers, comparators and validators with no
macros, no `derive`, and no runtime cost.

> ## 0.17 redesigned this API — read this first
>
> Every tutorial, every Stack Overflow answer and almost certainly your own
> memory uses `.fields` with `f.name` and `f.type`. **That is gone.** 0.17 uses
> **parallel arrays**.
>
> ```zig
> const std = @import("std");
>
> const User = struct { id: u32, name: []const u8, active: bool };
>
> pub fn main() void {
>     inline for (@typeInfo(User).@"struct".fields) |f| {
>         std.debug.print("{s}: {s}\n", .{ f.name, @typeName(f.type) });
>     }
> }
> ```
>
> **Deliberate compile error:**
>
> ```
> r_old.zig:6:43: error: no field named 'fields' in struct 'lang.Type.Struct'
>     inline for (@typeInfo(User).@"struct".fields) |f| {
>                                           ^~~~~~
> /opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/lang.zig:750:24: note: struct declared here
>     pub const Struct = struct {
>                        ^~~~~~
> ```
>
> `std.meta.fields(T)` is gone too, and says so explicitly:
>
> ```
> /opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/meta.zig:248:20: error: deprecated in favor of @typeInfo
> pub const fields = @compileError("deprecated in favor of @typeInfo");
>                    ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
> ```

## The correct 0.17 shape

`field_names` and `field_types` are two arrays of equal length. Iterate both at
once with a multi-object `inline for`:

```zig
const std = @import("std");

const User = struct {
    id: u32,
    name: []const u8,
    active: bool,
};

pub fn main() void {
    const st = @typeInfo(User).@"struct";
    std.debug.print("{s}: {d} fields, {d} decls, tuple={}, layout={t}\n", .{
        @typeName(User), st.field_names.len, st.decl_names.len, st.is_tuple, st.layout,
    });
    inline for (st.field_names, st.field_types) |name, FT| {
        std.debug.print("  {s}: {s} (size {d}, offset {d})\n", .{
            name, @typeName(FT), @sizeOf(FT), @offsetOf(User, name),
        });
    }
}
```

```
r1.User: 3 fields, 0 decls, tuple=false, layout=auto
  id: u32 (size 4, offset 16)
  name: []const u8 (size 16, offset 0)
  active: bool (size 1, offset 20)
```

Two payoffs hiding in that output: the `.@"struct"` tag needs quoting because
`struct` is a keyword, and **the offsets aren't in declaration order** — `name`
is at 0, `id` at 16. Zig reorders fields of an `auto`-layout struct to minimize
padding. Reflection reports the declared order; `@offsetOf` reports the truth.

### What `Struct` actually contains

| Field | Type | Notes |
|---|---|---|
| `field_names` | `[]const [:0]const u8` | declaration order, null-terminated |
| `field_types` | `[]const type` | same length as `field_names` |
| `field_attrs` | `[]const FieldAttributes` | same length; alignment, `comptime`, default |
| `decl_names` | `[]const [:0]const u8` | **`pub` declarations only** |
| `is_tuple` | `bool` | `true` for `.{ a, b }` types |
| `layout` | `enum { auto, @"extern", @"packed" }` | |
| `backing_integer` | `?type` | non-null only for `packed` |

There is **no** `.fields`, no `f.name`, no `f.type`, and no `f.default_value`.

| Old API | 0.17 |
|---|---|
| `info.fields` | `info.field_names` + `info.field_types` |
| `f.name` | the `field_names` element |
| `f.type` | the `field_types` element |
| `f.alignment` | `field_attrs[i].@"align"` |
| `f.default_value_ptr` | `field_attrs[i].defaultValue(FT)` |
| `std.meta.fields(T)` | `@typeInfo(T)` directly |
| `@Type(.{...})` | `@Int(.unsigned, bits)` for ints |

Enums and unions follow the same pattern — `@typeInfo(E).@"enum"` has
`field_names` + `field_values`, and error sets have `error_names`.

## `decl_names` and `@hasDecl` see only `pub`

```zig
const std = @import("std");

const User = struct {
    id: u32,
    name: []const u8,

    pub const kind = "user";
    pub fn greet(_: User) void {}
    fn private(_: User) void {}
};

pub fn main() void {
    const st = @typeInfo(User).@"struct";
    std.debug.print("decls: {d}\n", .{st.decl_names.len});
    inline for (st.decl_names) |d| std.debug.print("  {s}\n", .{d});
    std.debug.print("hasDecl greet={} private={} missing={}\n", .{
        @hasDecl(User, "greet"), @hasDecl(User, "private"), @hasDecl(User, "nope"),
    });
    std.debug.print("hasField id={} nope={}\n", .{ @hasField(User, "id"), @hasField(User, "nope") });
}
```

```
decls: 2
  kind
  greet
hasDecl greet=true private=false missing=false
hasField id=true nope=false
```

`private` is invisible to reflection. For the interface checks in 6.3 that is
the right behavior: a non-`pub` method is not part of the contract.

## `@field` — dynamic access with static names

`@field(value, "name")` is field access where the name is a comptime string. In
an `inline for` over `field_names`, that's everything you need:

```zig
const std = @import("std");

const Config = struct {
    host: []const u8 = "localhost",
    port: u16 = 8080,
    debug: bool = false,
};

pub fn main() void {
    var c: Config = .{};
    c.port = 9090;

    inline for (@typeInfo(Config).@"struct".field_names) |n| {
        std.debug.print("{s} = {any}\n", .{ n, @field(c, n) });
    }

    @field(c, "debug") = true;
    std.debug.print("after @field write: debug={}\n", .{c.debug});

    const st = @typeInfo(Config).@"struct";
    inline for (st.field_names, st.field_types, st.field_attrs) |n, FT, attrs| {
        std.debug.print("{s}: default={?any}\n", .{ n, attrs.defaultValue(FT) });
    }
}
```

```
host = { 108, 111, 99, 97, 108, 104, 111, 115, 116 }
port = 9090
debug = false
after @field write: debug=true
host: default={ 108, 111, 99, 97, 108, 104, 111, 115, 116 }
port: default=8080
debug: default=false
```

`@field` reads **and writes**. `attrs.defaultValue(FT)` is how you recover a
field's default — it takes the field type because the stored pointer is
type-erased.

## Auto-serialize any struct

The payoff. One recursive function that handles structs, slices, optionals,
enums and scalars — and refuses anything else at compile time:

```zig
const std = @import("std");

fn writeValue(out: *std.Io.Writer, value: anytype) !void {
    const T = @TypeOf(value);
    switch (@typeInfo(T)) {
        .@"struct" => |st| {
            try out.writeAll("{");
            inline for (st.field_names, 0..) |name, i| {
                if (i > 0) try out.writeAll(", ");
                try out.print("{s}: ", .{name});
                try writeValue(out, @field(value, name));
            }
            try out.writeAll("}");
        },
        .int, .comptime_int => try out.print("{d}", .{value}),
        .float, .comptime_float => try out.print("{d}", .{value}),
        .bool => try out.print("{}", .{value}),
        .pointer => |p| {
            if (p.size == .slice and p.child == u8) {
                try out.print("\"{s}\"", .{value});
            } else {
                try out.writeAll("[");
                for (value, 0..) |v, i| {
                    if (i > 0) try out.writeAll(", ");
                    try writeValue(out, v);
                }
                try out.writeAll("]");
            }
        },
        .optional => {
            if (value) |v| try writeValue(out, v) else try out.writeAll("null");
        },
        .@"enum" => try out.print("\"{t}\"", .{value}),
        else => @compileError("cannot serialize " ++ @typeName(T)),
    }
}

const Role = enum { admin, guest };
const Address = struct { city: []const u8, zip: u32 };
const User = struct {
    id: u32,
    name: []const u8,
    active: bool,
    score: f32,
    role: Role,
    nickname: ?[]const u8,
    tags: []const []const u8,
    addr: Address,
};

pub fn main() !void {
    var buf: [512]u8 = undefined;
    var w = std.Io.Writer.fixed(&buf);

    const u: User = .{
        .id = 7,
        .name = "ada",
        .active = true,
        .score = 9.5,
        .role = .admin,
        .nickname = null,
        .tags = &.{ "x", "y" },
        .addr = .{ .city = "London", .zip = 12345 },
    };

    try writeValue(&w, u);
    std.debug.print("{s}\n", .{w.buffered()});
}
```

```
{id: 7, name: "ada", active: true, score: 9.5, role: "admin", nickname: null, tags: ["x", "y"], addr: {city: "London", zip: 12345}}
```

Add a field to `User` and the output updates with no other change. Add a field
of an unsupported type and you get a compile error naming it.

Mechanisms worth naming:

| Line | What it does |
|---|---|
| `switch (@typeInfo(T))` | dispatch on the *kind* of type; arms that don't apply aren't compiled |
| `|st|` / `|p|` | capture the payload — `Struct`, `Pointer` — to read its fields |
| `p.size == .slice and p.child == u8` | distinguish a string from a generic slice |
| `inline for (..., 0..)` | the index gives you comma separators |
| `writeValue(out, @field(...))` | recursion builds a new instantiation per field type |
| `std.Io.Writer.fixed(&buf)` | a writer with **no `io`** — returns a `Writer` directly, no `.interface` |
| `w.buffered()` | read back what was written |

> `Writer.fixed` is the asymmetry to remember: `File.writer()` and
> `Stream.writer()` give you a wrapper whose `.interface` field is the
> `*std.Io.Writer`, but `Writer.fixed` **is** the `Writer`. It needs no `io` and
> no flush, which makes it ideal for testing writer code.

## A generic `eql`

Zig's `==` deliberately refuses structs:

```zig
const std = @import("std");

const Point = struct { x: i32, y: i32 };

pub fn main() void {
    const a: Point = .{ .x = 1, .y = 2 };
    const b: Point = .{ .x = 1, .y = 2 };
    std.debug.print("{}\n", .{a == b});
}
```

**Deliberate compile error:**

```
r5.zig:6:33: error: operator == not allowed for type 'r5.Point'
    std.debug.print("{}\n", .{a == b});
                              ~~^~~~~
r5.zig:2:15: note: struct declared here
const Point = struct { x: i32, y: i32 };
              ^~~~~~~~~~~~~~~~~~~~~~~~~
```

Zig won't guess whether you meant field-by-field or pointer identity, especially
for a struct holding a slice. Reflection lets you say which:

```zig
const std = @import("std");

fn eql(a: anytype, b: @TypeOf(a)) bool {
    const T = @TypeOf(a);
    return switch (@typeInfo(T)) {
        .@"struct" => |st| {
            inline for (st.field_names) |name| {
                if (!eql(@field(a, name), @field(b, name))) return false;
            }
            return true;
        },
        .pointer => |p| if (p.size == .slice) blk: {
            if (a.len != b.len) break :blk false;
            for (a, b) |x, y| if (!eql(x, y)) break :blk false;
            break :blk true;
        } else a == b,
        .optional => if (a) |x| (if (b) |y| eql(x, y) else false) else b == null,
        else => a == b,
    };
}

const Point = struct { x: i32, y: i32 };
const Shape = struct { name: []const u8, pts: []const Point, label: ?[]const u8 };

pub fn main() void {
    const a: Shape = .{ .name = "tri", .pts = &.{ .{ .x = 0, .y = 0 }, .{ .x = 1, .y = 2 } }, .label = null };
    const b: Shape = .{ .name = "tri", .pts = &.{ .{ .x = 0, .y = 0 }, .{ .x = 1, .y = 2 } }, .label = null };
    const c: Shape = .{ .name = "tri", .pts = &.{.{ .x = 9, .y = 9 }}, .label = "L" };

    std.debug.print("a == b : {}\n", .{eql(a, b)});
    std.debug.print("a == c : {}\n", .{eql(a, c)});
    std.debug.print("std.meta.eql(Point,Point): {}\n", .{std.meta.eql(Point{ .x = 1, .y = 2 }, Point{ .x = 1, .y = 2 })});
}
```

```
a == b : true
a == c : false
std.meta.eql(Point,Point): true
```

Note `b: @TypeOf(a)` in the signature — it forces both arguments to the same
type, so `@field(b, name)` is guaranteed to exist.

`std.meta.eql` ships this, but it compares slices by **pointer**, not contents.
Ours recurses into them. Knowing the difference is the point.

## `std.meta` helpers

```zig
const std = @import("std");

const Color = enum { red, green, blue };
const User = struct { id: u32, name: []const u8 };

pub fn main() void {
    std.debug.print("tags: ", .{});
    inline for (std.meta.tags(Color)) |t| std.debug.print("{t} ", .{t});
    std.debug.print("({d} total)\n", .{std.meta.tags(Color).len});

    inline for (comptime std.meta.fieldNames(User)) |n| std.debug.print("field {s}\n", .{n});

    std.debug.print("stringToEnum(\"green\") = {?t}\n", .{std.meta.stringToEnum(Color, "green")});
    std.debug.print("stringToEnum(\"pink\")  = {?t}\n", .{std.meta.stringToEnum(Color, "pink")});
    std.debug.print("Child(?u8)  = {s}\n", .{@typeName(std.meta.Child(?u8))});
    std.debug.print("Elem([]u8)  = {s}\n", .{@typeName(std.meta.Elem([]u8))});
    std.debug.print("fieldIndex(User,\"name\") = {?d}\n", .{std.meta.fieldIndex(User, "name")});
    std.debug.print("FieldEnum(User) = {s}\n", .{@typeName(std.meta.FieldEnum(User))});

    const ei = @typeInfo(Color).@"enum";
    std.debug.print("tag_type={s} mode={t} values=", .{ @typeName(ei.tag_type), ei.mode });
    inline for (ei.field_names, ei.field_values) |n, v| std.debug.print("{s}={d} ", .{ n, v });
    std.debug.print("\n", .{});
}
```

```
tags: red green blue (3 total)
field id
field name
stringToEnum("green") = green
stringToEnum("pink")  = null
Child(?u8)  = u8
Elem([]u8)  = u8
fieldIndex(User,"name") = 1
FieldEnum(User) = meta.FieldEnum(r6.User)
tag_type=u2 mode=exhaustive values=red=0 green=1 blue=2
```

| Helper | Returns |
|---|---|
| `tags(E)` | `*const [N]E` — every enum value, iterable |
| `fieldNames(T)` | `[]const [:0]const u8` |
| `fieldTypes(T)` | `[]const type` |
| `stringToEnum(E, s)` | `?E` — **runtime** string to enum, the one non-comptime helper |
| `Child(T)` | the payload of an optional/pointer/array |
| `Elem(T)` | the element type of a slice/array |
| `fieldIndex(T, "n")` | `?comptime_int` |
| `FieldEnum(T)` | an enum with one tag per field |
| `Tag(U)` | a tagged union's tag enum |
| `eql(a, b)` | shallow structural equality |

`tag_type=u2` is a nice detail: a three-member enum is backed by the smallest
integer that fits.

### The `fieldNames` trap

Unlike `tags`, `fieldNames` returns a *runtime* slice type, so an `inline for`
over it needs an explicit `comptime`:

```zig
const std = @import("std");

const User = struct { id: u32, name: []const u8 };

pub fn main() void {
    inline for (std.meta.fieldNames(User)) |n| {
        std.debug.print("{s}\n", .{n});
    }
}
```

**Deliberate compile error:**

```
r7.zig:6:17: error: unable to resolve comptime value
    inline for (std.meta.fieldNames(User)) |n| {
                ^~~~~~~~~~~~~~~~~~~~~~~~~
r7.zig:6:17: note: inline loop condition must be comptime-known
```

Fix it with `inline for (comptime std.meta.fieldNames(User))`, or skip the
helper and use `@typeInfo(User).@"struct".field_names`, which needs no
`comptime`.

## `inline for` over a tuple

An `anytype` parameter can be a tuple of mixed types — exactly how
`print(fmt, args)` works. `inline for` iterates the *values*:

```zig
const std = @import("std");

fn describeAll(vals: anytype) void {
    const st = @typeInfo(@TypeOf(vals)).@"struct";
    std.debug.print("tuple? {} with {d} members\n", .{ st.is_tuple, st.field_names.len });
    inline for (vals) |v| {
        std.debug.print("  {s:>12} = {any}\n", .{ @typeName(@TypeOf(v)), v });
    }
}

pub fn main() void {
    describeAll(.{ @as(u8, 7), true, @as(f32, 1.5), "hi" });
}
```

```
tuple? true with 4 members
            u8 = 7
          bool = true
           f32 = 1.5
  *const [2:0]u8 = { 104, 105 }
```

Each iteration has a **different** type for `v` — impossible in a runtime loop,
which is the whole reason `inline for` exists. Note `"hi"` arrives as
`*const [2:0]u8`, a pointer to a sentinel-terminated array, not a slice.

## Reflecting on a module

Modules are structs, so `@typeInfo` works on an `@import`:

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("std.math has {d} public decls\n", .{@typeInfo(std.math).@"struct".decl_names.len});
    std.debug.print("std.meta  has {d} public decls\n", .{@typeInfo(std.meta).@"struct".decl_names.len});

    var count: usize = 0;
    inline for (@typeInfo(std.math).@"struct".decl_names) |d| {
        if (d.len >= 3 and std.mem.eql(u8, d[0..3], "log")) {
            std.debug.print("  {s}\n", .{d});
            count += 1;
        }
    }
    std.debug.print("{d} decls starting with \"log\"\n", .{count});
}
```

```
std.math has 142 public decls
std.meta  has 31 public decls
  log2e
  log10e
  log
  log2
  log10
  log10_int
  log_int
  log1p
  log2_int
  log2_int_ceil
10 decls starting with "log"
```

This is how you search the standard library without leaving your editor — and
in 6.1's terms, the loop is gone by the time the program runs.

---

Next: [6.6 — comptime Costs](./06-costs.md)
