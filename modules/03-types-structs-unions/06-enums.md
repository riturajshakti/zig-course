# 3.6 — Enums

## Values and backing types

```zig
const std = @import("std");

const Color = enum { red, green, blue };
const Status = enum(u16) { ok = 200, not_found = 404, teapot = 418 };
const Level = enum(u8) { low, mid = 5, high };

pub fn main() void {
    const c = Color.green;
    const s: Status = .not_found;

    std.debug.print("c = {t}  int = {d}\n", .{ c, @intFromEnum(c) });
    std.debug.print("s = {t}  int = {d}\n", .{ s, @intFromEnum(s) });
    std.debug.print("@tagName(c) = {s}\n", .{@tagName(c)});
    std.debug.print("{{any}} on c  = {any}\n", .{c});

    std.debug.print("\nColor backing   = {s} size={d}\n", .{ @typeName(@typeInfo(Color).@"enum".tag_type), @sizeOf(Color) });
    std.debug.print("Status backing  = {s} size={d}\n", .{ @typeName(@typeInfo(Status).@"enum".tag_type), @sizeOf(Status) });

    std.debug.print("\nLevel values: ", .{});
    inline for (std.meta.tags(Level)) |t| std.debug.print("{t}={d} ", .{ t, @intFromEnum(t) });
    std.debug.print("\n", .{});
}
```

```
c = green  int = 1
s = not_found  int = 404
@tagName(c) = green
{any} on c  = .green

Color backing   = u2 size=1
Status backing  = u16 size=2

Level values: low=0 mid=5 high=6 
```

| Form | Backing | Values |
|---|---|---|
| `enum { red, green, blue }` | **`u2`** — smallest that fits | 0, 1, 2 |
| `enum(u16) { ok = 200, ... }` | `u16` as you asked | exactly what you wrote |
| `enum(u8) { low, mid = 5, high }` | `u8` | 0, 5, **6** — counts on from the last |

**`Color`'s backing type is `u2`, not `u8`.** With no explicit integer type Zig
picks the narrowest one, so you cannot assume the numeric values or the size.
Say `enum(u8)` when the number matters.

`.not_found` with no type prefix works because the target type is known from the
left — the same inference as `.{ ... }` for structs. `{t}` prints the tag name
and is the specifier you want 95% of the time.

## Methods

Enums take methods like structs do.

```zig
const std = @import("std");

const Status = enum(u16) {
    ok = 200,
    created = 201,
    not_found = 404,
    server_error = 500,

    pub fn code(self: Status) u16 {
        return @intFromEnum(self);
    }

    pub fn isError(self: Status) bool {
        return self.code() >= 400;
    }

    pub fn text(self: Status) []const u8 {
        return switch (self) {
            .ok => "OK",
            .created => "Created",
            .not_found => "Not Found",
            .server_error => "Internal Server Error",
        };
    }

    pub fn format(self: Status, w: *std.Io.Writer) !void {
        try w.print("{d} {s}", .{ self.code(), self.text() });
    }
};

pub fn main() void {
    inline for (std.meta.tags(Status)) |s| {
        std.debug.print("{f} -> isError={}\n", .{ s, s.isError() });
    }
}
```

```
200 OK -> isError=false
201 Created -> isError=false
404 Not Found -> isError=true
500 Internal Server Error -> isError=true
```

An enum is a type, so it gets a namespace — methods, constants, nested types,
and a `format` for `{f}`. `std.meta.tags(T)` gives every tag as a comptime array,
which `inline for` walks.

## Switch exhaustiveness

Miss a case and it will not compile. Deliberately broken:

```zig
const std = @import("std");

const Color = enum { red, green, blue };

fn hex(c: Color) []const u8 {
    return switch (c) {
        .red => "#ff0000",
        .green => "#00ff00",
    };
}

pub fn main() void {
    std.debug.print("{s}\n", .{hex(.red)});
}
```

```
e3.zig:6:12: error: switch must handle all possibilities
    return switch (c) {
           ^~~~~~
e3.zig:3:34: note: unhandled enumeration value: 'blue'
const Color = enum { red, green, blue };
```

**This is the payoff.** Add a variant to the enum and every incomplete `switch`
in the codebase becomes a compile error pointing at code you need to update. It
is the same guarantee as Rust's `match` and strictly better than Go's `switch`,
where a missing case silently falls through.

`else` and multi-case:

```zig
const std = @import("std");

const Color = enum { red, green, blue, cyan, magenta };

fn hex(c: Color) []const u8 {
    return switch (c) {
        .red => "#ff0000",
        .green => "#00ff00",
        .blue => "#0000ff",
        else => "#cccccc",
    };
}

fn isWarm(c: Color) bool {
    return switch (c) {
        .red, .magenta => true,
        .green, .blue, .cyan => false,
    };
}

pub fn main() void {
    inline for (std.meta.tags(Color)) |c| {
        std.debug.print("{t:<8} hex={s} warm={}\n", .{ c, hex(c), isWarm(c) });
    }
}
```

```
red      hex=#ff0000 warm=true
green    hex=#00ff00 warm=false
blue     hex=#0000ff warm=false
cyan     hex=#cccccc warm=false
magenta  hex=#cccccc warm=true
```

`else` makes the switch total — and **silently absorbs new variants**, which
is exactly what you give up. Prefer listing cases (`.a, .b => ...`) so adding a
variant still breaks the build.

## Conversions

```zig
const std = @import("std");

const Status = enum(u16) { ok = 200, not_found = 404 };

pub fn main() void {
    const s = Status.not_found;

    std.debug.print("enum -> int    : {d}\n", .{@intFromEnum(s)});
    std.debug.print("int  -> enum   : {t}\n", .{@as(Status, @enumFromInt(200))});
    std.debug.print("enum -> string : {s}\n", .{@tagName(s)});
    std.debug.print("string -> enum : {?t}\n", .{std.meta.stringToEnum(Status, "ok")});
    std.debug.print("string -> enum : {?t}\n", .{std.meta.stringToEnum(Status, "nope")});

    std.debug.print("\ntag count      : {d}\n", .{std.meta.tags(Status).len});
    std.debug.print("fieldNames     : ", .{});
    inline for (comptime std.meta.fieldNames(Status)) |n| std.debug.print("{s} ", .{n});
    std.debug.print("\n", .{});
}
```

```
enum -> int    : 404
int  -> enum   : ok
enum -> string : not_found
string -> enum : ok
string -> enum : null

tag count      : 2
fieldNames     : ok not_found 
```

| Direction | Call | On failure |
|---|---|---|
| enum → int | `@intFromEnum(e)` | can't fail |
| int → enum | `@enumFromInt(n)` | **panics** on an invalid value |
| enum → string | `@tagName(e)` | can't fail (exhaustive enums) |
| string → enum | `std.meta.stringToEnum(T, s)` | returns **`?T`** — `null` |
| all tags | `std.meta.tags(T)` | comptime array |
| all names | `comptime std.meta.fieldNames(T)` | comptime |

`stringToEnum` returning an optional is what you want for parsing user input.
`@enumFromInt` is the dangerous one — it is an unchecked cast:

```zig
const std = @import("std");

const Status = enum(u16) { ok = 200, not_found = 404 };

pub fn main() void {
    var n: u16 = 999;
    _ = &n;
    const s: Status = @enumFromInt(n);
    std.debug.print("{t}\n", .{s});
}
```

```
thread 16477628 panic: invalid enum value
e6.zig:8:23: 0x10086f857 in main (e6)
    const s: Status = @enumFromInt(n);
                      ^
```

Deliberate. In Debug and ReleaseSafe you get this panic; in `ReleaseFast` you get
an enum holding an impossible value and undefined behavior downstream. Validate
untrusted integers before casting, or use a non-exhaustive enum.
(The thread id and the `0x...` addresses differ on every run — only the
message and the source location are stable.)


(`comptime` is required before `std.meta.fieldNames(T)` in an `inline for` — it
returns `[]const [:0]const u8`, a runtime slice type, and the loop needs a
comptime-known length.)

## Non-exhaustive enums: `_`

For values that come from outside your program — a wire protocol, a file format,
a syscall — add `_` as the last "field". The enum then legally holds any value
its backing type can represent.

```zig
const std = @import("std");

const Opcode = enum(u8) {
    nop = 0x00,
    load = 0x01,
    store = 0x02,
    _,
};

fn describe(op: Opcode) []const u8 {
    return switch (op) {
        .nop => "nop",
        .load => "load",
        .store => "store",
        _ => "unknown opcode",
    };
}

pub fn main() void {
    var raw: u8 = 0x99;
    _ = &raw;

    const known: Opcode = .load;
    const unknown: Opcode = @enumFromInt(raw);

    std.debug.print("known   = {s} (0x{x:0>2})\n", .{ describe(known), @intFromEnum(known) });
    std.debug.print("unknown = {s} (0x{x:0>2})\n", .{ describe(unknown), @intFromEnum(unknown) });
    std.debug.print("is it a named tag? {}\n", .{std.enums.tagName(Opcode, unknown) != null});
    std.debug.print("tagName(known)     = {?s}\n", .{std.enums.tagName(Opcode, known)});
}
```

```
known   = load (0x01)
unknown = unknown opcode (0x99)
is it a named tag? false
tagName(known)     = load
```

`@enumFromInt(0x99)` **did not panic** this time — that is the point of `_`.

| | Exhaustive `enum` | Non-exhaustive `enum(u8) { ..., _ }` |
|---|---|---|
| legal values | only the named ones | any `u8` |
| `@enumFromInt` of an unlisted value | **panics** | fine |
| `switch` needs | every named case | every named case **plus `_` or `else`** |
| `@tagName` of an unlisted value | n/a | **panics** `invalid enum value` — use `std.enums.tagName` |

`_` is the `switch` case for "a value with no name", distinct from `else`, which
also swallows named variants you forgot. And note `std.enums.tagName(T, e)`
returns `?[]const u8` — the safe version of `@tagName` for these.

## Enums as keys

```zig
const std = @import("std");

const Color = enum { red, green, blue };

pub fn main() void {
    var counts = std.enums.EnumArray(Color, u32).initFill(0);
    counts.set(.red, 5);
    counts.set(.blue, 2);

    inline for (std.meta.tags(Color)) |c| {
        std.debug.print("counts[{t}] = {d}\n", .{ c, counts.get(c) });
    }

    var set: std.enums.EnumSet(Color) = .empty;
    set.insert(.red);
    set.insert(.green);
    std.debug.print("set has blue? {}  count={d}\n", .{ set.contains(.blue), set.count() });

    const names = [_][]const u8{ "crimson", "emerald", "azure" };
    std.debug.print("names[@intFromEnum(.green)] = {s}\n", .{names[@intFromEnum(Color.green)]});
}
```

```
counts[red] = 5
counts[green] = 0
counts[blue] = 2
set has blue? false  count=2
names[@intFromEnum(.green)] = emerald
```

`EnumArray` is a fixed array indexed by tag — no hashing, no allocation, and
every key is guaranteed present. `EnumSet` is a bitfield. Reach for these before
a `HashMap` when the key is an enum.

> **0.17.0:** `EnumSet(T).initEmpty()` is gone. It is now the constant
> `.empty` (and `.full`), matching how `ArrayList` became `.empty`.

## Operations table

| Operation | Code |
|---|---|
| declare | `const E = enum { a, b };` |
| explicit backing type | `const E = enum(u8) { a = 1, b = 2 };` |
| non-exhaustive | `const E = enum(u8) { a, b, _ };` |
| value | `E.a`, or just `.a` where the type is known |
| to integer | `@intFromEnum(e)` |
| from integer | `@enumFromInt(n)` — **panics** if invalid |
| to string | `@tagName(e)` → `[:0]const u8` |
| to string (non-exhaustive) | `std.enums.tagName(E, e)` → `?[]const u8` |
| from string | `std.meta.stringToEnum(E, s)` → `?E` |
| print the name | `{t}` |
| print optional name | `{?t}` |
| custom printing | `pub fn format(self: E, w: *std.Io.Writer) !void` → `{f}` |
| compare | `a == b` — **allowed**, unlike structs |
| switch | `switch (e) { .a => ..., .b => ... }` — exhaustive |
| multi-case | `.a, .b => ...` |
| catch-all | `else => ...` (absorbs new variants) |
| unnamed values only | `_ => ...` (non-exhaustive enums) |
| all tags | `std.meta.tags(E)` |
| all names | `comptime std.meta.fieldNames(E)` |
| count | `std.meta.tags(E).len` |
| backing type | `@typeInfo(E).@"enum".tag_type` |
| size | `@sizeOf(E)` |
| methods / constants | declare them in the enum body |
| array keyed by tag | `std.enums.EnumArray(E, V)` |
| set of tags | `std.enums.EnumSet(E)` — `.empty` / `.full` |

Unlike structs, **`==` works on enums** — they are just integers underneath.

---

Next: [3.7 — Tagged Unions](./07-unions.md)
