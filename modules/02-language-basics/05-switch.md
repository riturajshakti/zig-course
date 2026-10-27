# 2.5 — Switch

## Exhaustiveness is enforced

```zig
const std = @import("std");

const Color = enum { red, green, blue };

pub fn main() void {
    const c = Color.green;
    switch (c) {
        .red => std.debug.print("stop\n", .{}),
        .green => std.debug.print("go\n", .{}),
    }
}
```

```
error: switch must handle all possibilities
    switch (c) {
    ^~~~~~
note: unhandled enumeration value: 'blue'
const Color = enum { red, green, blue };
                                 ^~~~
```

The compiler names the missing case. Add `blue` and it builds:

```zig
const std = @import("std");

const Color = enum { red, green, blue };

pub fn main() void {
    for ([_]Color{ .red, .green, .blue }) |c| {
        switch (c) {
            .red => std.debug.print("stop\n", .{}),
            .green => std.debug.print("go\n", .{}),
            .blue => std.debug.print("chill\n", .{}),
        }
    }
}
```

```
stop
go
chill
```

This is the single best reason to use an enum in Zig. **Add a variant to the
enum and every `switch` on it that lacks an `else` stops compiling**, pointing
at each place that needs updating. Go's `switch` with a `default` would have
silently taken the default path at runtime.

Note the shorthand: `.red` with no type prefix, because Zig infers the enum from
the switch operand. `Color.red` also works and is required where there is no
context to infer from.

## No fallthrough, no `break`

```zig
const std = @import("std");

const Color = enum(u8) { red = 1, green = 2, blue = 4 };

pub fn main() void {
    for ([_]Color{ .red, .green, .blue }) |c| {
        std.debug.print("{t} tag={d}\n", .{ c, @backingInt(c) });
    }

    const c = Color.green;
    switch (c) {
        .red, .blue => std.debug.print("primary-ish\n", .{}),
        .green => {
            std.debug.print("green: no fallthrough\n", .{});
            std.debug.print("green: still green\n", .{});
        },
    }
}
```

```
red tag=1
green tag=2
blue tag=4
green: no fallthrough
green: still green
```

Exactly one arm runs, and a multi-statement arm is just a block — no `break`
needed, and no accidental fallthrough possible. To match several values, list
them with commas: `.red, .blue =>`.

(`enum(u8) { red = 1 }` sets the backing integer type and the explicit tag
values; `{t}` prints the tag name. Module 03 covers enums properly.)

> **Watch the builtin name.** The tag number comes from **`@backingInt`**. Older
> code and every tutorial you will find say `@intFromEnum` — that still compiles
> on 0.17.0, but `zig fmt` silently rewrites it to `@backingInt`, which is the
> name the standard library itself now uses.

## Ranges and `else`

```zig
const std = @import("std");

pub fn main() void {
    for ([_]u8{ 0, 3, 7, 42, 200 }) |n| {
        const bucket = switch (n) {
            0 => "zero",
            1...5 => "small",
            6...10 => "medium",
            else => "big",
        };
        std.debug.print("{d:>3} -> {s}\n", .{ n, bucket });
    }
}
```

```
  0 -> zero
  3 -> small
  7 -> medium
 42 -> big
200 -> big
```

`1...5` is **inclusive on both ends** — three dots, not two. (Zig's `for (0..5)`
range is exclusive and uses two dots; the switch range is inclusive and uses
three. Do not mix them up.)

On integers `else` is effectively mandatory unless you enumerate all 256 values
of a `u8`, which is why exhaustiveness is a feature you feel on enums and a
formality on integers.

## Switch as an expression

That last example was already one. Here it is as a function body:

```zig
const std = @import("std");

const Op = enum { add, sub, mul, div };

fn apply(op: Op, a: i32, b: i32) i32 {
    return switch (op) {
        .add => a + b,
        .sub => a - b,
        .mul => a * b,
        .div => @divTrunc(a, b),
    };
}

pub fn main() void {
    inline for (.{ Op.add, Op.sub, Op.mul, Op.div }) |op| {
        std.debug.print("{t}(12, 5) = {d}\n", .{ op, apply(op, 12, 5) });
    }
}
```

```
add(12, 5) = 17
sub(12, 5) = 7
mul(12, 5) = 60
div(12, 5) = 2
```

Every arm must produce the same type. `return switch (...)` with no `else` is
the idiomatic Zig dispatch function — exhaustive, total, and one expression.

## Capturing the matched value

```zig
const std = @import("std");

pub fn main() void {
    for ([_]u8{ 2, 9, 50 }) |n| {
        switch (n) {
            0...9 => |digit| std.debug.print("single digit: {d}, doubled {d}\n", .{ digit, digit * 2 }),
            else => |other| std.debug.print("other: {d}\n", .{other}),
        }
    }
}
```

```
single digit: 2, doubled 4
single digit: 9, doubled 18
other: 50
```

`|digit|` binds the value that matched. On a plain integer that is merely
convenient. On a **tagged union** it is the whole point: the capture is the
payload of the matched variant, correctly typed per arm. That is how Zig does
sum types, and module 03 builds on exactly this syntax.

## Switching on strings — you can't

Zig has no string switch, because `[]const u8` is a slice (a pointer and a
length) and comparing slices means comparing *contents*, which is a function
call, not a jump table. So you write the `if` chain:

```zig
const std = @import("std");

pub fn main() void {
    for ([_][]const u8{ "GET", "POST", "BREW" }) |method| {
        const code: u16 = if (std.mem.eql(u8, method, "GET"))
            200
        else if (std.mem.eql(u8, method, "POST"))
            201
        else
            405;
        std.debug.print("{s} -> {d}\n", .{ method, code });
    }
}
```

```
GET -> 200
POST -> 201
BREW -> 405
```

That works, but it throws away exhaustiveness. The idiomatic fix is to **parse
the string into an enum once**, then switch on the enum:

```zig
const std = @import("std");

const Method = enum {
    get,
    post,
    delete,

    fn parse(s: []const u8) ?Method {
        return std.meta.stringToEnum(Method, s);
    }
};

pub fn main() void {
    for ([_][]const u8{ "get", "post", "delete", "brew" }) |raw| {
        const m = Method.parse(raw) orelse {
            std.debug.print("{s} -> unknown method\n", .{raw});
            continue;
        };
        const code: u16 = switch (m) {
            .get => 200,
            .post => 201,
            .delete => 204,
        };
        std.debug.print("{s} -> {d}\n", .{ raw, code });
    }
}
```

```
get -> 200
post -> 201
delete -> 204
brew -> unknown method
```

`std.meta.stringToEnum(T, s)` returns `?T` — it is generated at compile time
from the enum's field names, so adding a variant updates the parser *and* breaks
the switch until you handle it. Validate at the boundary, switch exhaustively
inside. That pattern is the backbone of the HTTP service in module 14.

Note `orelse { ...; continue; }` — the block runs when the optional is null, and
since it `continue`s it never has to produce a value. Module 04 covers optionals.

## `unreachable` in the `else` arm

When you have *proved* the remaining cases cannot happen, say so:

```zig
const std = @import("std");

fn digitName(d: u8) []const u8 {
    return switch (d) {
        0 => "zero",
        1 => "one",
        2 => "two",
        else => unreachable,
    };
}

pub fn main() void {
    std.debug.print("{s} {s} {s}\n", .{ digitName(0), digitName(1), digitName(2) });
    std.debug.print("{s}\n", .{digitName(9)});
}
```

```
zero one two
thread 16413438 panic: reached unreachable code
/tmp/unreach.zig:8:17: 0x1009edf7f in digitName (unreach)
        else => unreachable,
                ^
/tmp/unreach.zig:14:41: 0x1009ed217 in main (unreach)
    std.debug.print("{s}\n", .{digitName(9)});
                                        ^
```

The first line printed; the deliberate fourth call crashed. The trace names both
the `unreachable` and the caller that violated the assumption, which is what you
want when the proof turns out to be wrong.

`unreachable` is not "do nothing." Its meaning is **"I guarantee control never
reaches here"**:

| Build mode | `unreachable` does |
|---|---|
| `Debug`, `ReleaseSafe` | panics with `reached unreachable code` |
| `ReleaseFast`, `ReleaseSmall` | **undefined behavior** — the optimizer assumes the branch is impossible and deletes it |

So `else => unreachable` is a claim about your inputs, not an error handler. If
the value could legitimately come from outside your control, return an error
instead — module 04 draws that line.

| Ending | Meaning |
|---|---|
| all cases listed, no `else` | exhaustive; adding a variant breaks the build (**prefer this**) |
| `else => value` | catch-all; adding a variant silently takes this path |
| `else => unreachable` | "cannot happen"; panics in Debug, UB in ReleaseFast |
| `else => return error.X` | "shouldn't happen, but handle it" |

---

Next: [2.6 — Blocks & Labels](./06-blocks-labels.md)
