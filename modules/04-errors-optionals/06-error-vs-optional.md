# 4.6 — Error vs Optional

One question decides it: **is the absence of a value a failure?**

| Situation | Type | Example |
|---|---|---|
| Something went wrong | `!T` | disk full, bad JSON, connection refused |
| Nothing is there, and that's normal | `?T` | key not in map, no match found, end of iterator |
| Could fail, *or* legitimately find nothing | `!?T` | DB query: connection dies vs row absent |
| Cannot fail | `T` | `list.items.len` |

A hash map miss is not an error — it is the expected answer to "is this key
here?". A dead database connection *is* an error. Conflating them means callers
`catch` things they should branch on, or branch on things they should propagate.

## All three in one type

```zig
const std = @import("std");

const Store = struct {
    keys: []const []const u8,
    vals: []const u32,

    fn get(s: Store, key: []const u8) ?u32 {
        for (s.keys, s.vals) |k, v| if (std.mem.eql(u8, k, key)) return v;
        return null;
    }

    fn parse(_: Store, text: []const u8) !u32 {
        return std.fmt.parseInt(u32, text, 10);
    }

    fn getParsed(s: Store, key: []const u8) !?u32 {
        const raw = s.get(key) orelse return null;
        if (raw > 1000) return error.OutOfRange;
        return raw;
    }
};

pub fn main() void {
    const s: Store = .{
        .keys = &.{ "a", "b", "big" },
        .vals = &.{ 1, 2, 5000 },
    };

    std.debug.print("get(\"a\")   = {?d}\n", .{s.get("a")});
    std.debug.print("get(\"zz\")  = {?d}\n", .{s.get("zz")});
    std.debug.print("parse(\"9\")  = {!d}\n", .{s.parse("9")});
    std.debug.print("parse(\"x\")  = {!d}\n", .{s.parse("x")});
    std.debug.print("getParsed(\"a\")   = {!?d}\n", .{s.getParsed("a")});
    std.debug.print("getParsed(\"zz\")  = {!?d}\n", .{s.getParsed("zz")});
    std.debug.print("getParsed(\"big\") = {!?d}\n", .{s.getParsed("big")});
}
```

```
get("a")   = 1
get("zz")  = null
parse("9")  = 9
parse("x")  = error.InvalidCharacter
getParsed("a")   = 1
getParsed("zz")  = null
getParsed("big") = error.OutOfRange
```

`{!?d}` prints all three states of a `!?u32`: the value, `null`, or the error.
`get` cannot fail, `parse` cannot be empty, `getParsed` can be either — and the
signature says which, so a caller cannot get it wrong.

## Unwrapping `!?T`

```zig
const std = @import("std");

fn lookup(k: u8) !?[]const u8 {
    if (k == 0) return error.BadKey;
    if (k == 1) return "one";
    return null;
}

pub fn main() !void {
    for (0..3) |i| {
        const k: u8 = @intCast(i);
        if (lookup(k)) |maybe| {
            if (maybe) |v| {
                std.debug.print("{d}: found {s}\n", .{ k, v });
            } else {
                std.debug.print("{d}: absent\n", .{k});
            }
        } else |e| {
            std.debug.print("{d}: failed {t}\n", .{ k, e });
        }
    }

    const flat = (lookup(2) catch null) orelse "<default>";
    std.debug.print("flattened: {s}\n", .{flat});

    const one = (try lookup(1)).?;
    std.debug.print("try-then-unwrap: {s}\n", .{one});
}
```

```
0: failed BadKey
1: found one
2: absent
flattened: <default>
try-then-unwrap: one
```

Three idioms for `!?T`:

| Code | Reads as |
|---|---|
| `if (x) \|m\| { if (m) \|v\| ... } else \|e\|` | handle all three explicitly |
| `(x catch null) orelse d` | collapse error *and* null to one default |
| `(try x).?` | propagate the error, assert non-null |

`catch null` works because `null` is a valid `?T`, so the error branch and the
absent branch fold into one. Use it when the caller genuinely treats "broken" and
"missing" the same way — in a cache, often true; in a request handler, usually not.

> A `!?T` is **not** `?!T`. `?!T` ("maybe a thing that failed") is legal but
> almost never what you mean. Keep errors outermost.

## `unreachable` per release mode

```zig
const std = @import("std");
const builtin = @import("builtin");

fn classify(n: u8) []const u8 {
    return switch (n) {
        0...9 => "small",
        10...99 => "medium",
        100...255 => "large",
    };
}

fn half(n: u32) u32 {
    if (n % 2 != 0) unreachable;
    return n / 2;
}

pub fn main() void {
    std.debug.print("mode = {t}\n", .{builtin.mode});
    std.debug.print("classify(5)   = {s}\n", .{classify(5)});
    std.debug.print("classify(200) = {s}\n", .{classify(200)});
    std.debug.print("half(8)       = {d}\n", .{half(8)});
}
```

```sh
zig run -O Debug        modes.zig
zig run -O ReleaseSafe  modes.zig
zig run -O ReleaseFast  modes.zig
zig run -O ReleaseSmall modes.zig
```

```
mode = debug
classify(5)   = small
classify(200) = large
half(8)       = 4
```

```
mode = safe
classify(5)   = small
classify(200) = large
half(8)       = 4
```

```
mode = fast
classify(5)   = small
classify(200) = large
half(8)       = 4
```

```
mode = small
classify(5)   = small
classify(200) = large
half(8)       = 4
```

All four agree, because the claim `half` makes is true for 8. Now break it:

```zig
const std = @import("std");

fn half(n: u32) u32 {
    if (n % 2 != 0) unreachable;
    return n / 2;
}

pub fn main() void {
    std.debug.print("half(8) = {d}\n", .{half(8)});
    std.debug.print("half(7) = {d}\n", .{half(7)});
}
```

**Deliberate failure, Debug** — exit code **134**:

```
half(8) = 4
thread 16449153 panic: reached unreachable code
/tmp/breakit.zig:4:21: 0x1010d8bef in half (breakit)
    if (n % 2 != 0) unreachable;
                    ^
/tmp/breakit.zig:10:46: 0x1010d785b in main (breakit)
    std.debug.print("half(7) = {d}\n", .{half(7)});
```

**Deliberate failure, ReleaseSafe** — also exit code **134**, but a far less
useful trace:

```
half(8) = 4
thread 16449902 panic: reached unreachable code
/opt/.../std/Io/Threaded.zig:2402:26: 0x100179a0f in callMainWithArgs (breakit)
    const thread = Thread.current orelse return .unblocked;
                         ^
```

`half` was inlined, so the trace points into stdlib instead of your line. Still
caught, still exit 134 — but debug the failure in `Debug`, not `ReleaseSafe`.

| Mode | `unreachable` becomes |
|---|---|
| `Debug` | panic `reached unreachable code`, exit 134, accurate trace |
| `ReleaseSafe` | panic, exit 134, trace may be inlined/misleading |
| `ReleaseFast` | **undefined behaviour** — no check; the optimizer assumes it cannot happen |
| `ReleaseSmall` | **undefined behaviour** |

That bottom half is why `unreachable` is not an error-handling tool. In
`ReleaseFast` the compiler *proves* your claim and optimizes around it; reaching
it there does not crash cleanly — it can produce nonsense output, a hang, or a
corrupted write. (The ReleaseFast run of the broken program is left as a
deliberately-unrun exercise: UB has no reproducible output to paste.)

| Tool | Checked in ReleaseFast? | Use for |
|---|---|---|
| `error` + `try` | **yes** — always | anything that can happen at runtime |
| `unreachable` / `.?` | no | claims you can prove |
| `std.debug.assert` | no | internal invariants |
| `if (bad) return error.X` | yes | validating untrusted input |

**Never validate untrusted input with `unreachable` or `assert`.** A malformed
request body must produce `error.Malformed`, not an assert — the assert vanishes
in the build you ship.

## `std.debug.assert`

```zig
const std = @import("std");
const assert = std.debug.assert;

fn copyInto(dst: []u8, src: []const u8) void {
    assert(dst.len >= src.len);
    @memcpy(dst[0..src.len], src);
}

pub fn main() void {
    var buf: [8]u8 = undefined;
    copyInto(&buf, "zig");
    std.debug.print("copied: {s}\n", .{buf[0..3]});
    assert(buf[0] == 'z');
    std.debug.print("assertions held\n", .{});
}
```

```
copied: zig
assertions held
```

Violate it:

```zig
const std = @import("std");
const assert = std.debug.assert;

fn copyInto(dst: []u8, src: []const u8) void {
    assert(dst.len >= src.len);
    @memcpy(dst[0..src.len], src);
}

pub fn main() void {
    var small: [2]u8 = undefined;
    copyInto(&small, "too long");
    std.debug.print("unreachable\n", .{});
}
```

**Deliberate failure** — exit code **134**:

```
thread 16456096 panic: reached unreachable code
/opt/.../std/debug.zig:442:14: 0x100ed1067 in assert (assertfail)
    if (!ok) unreachable; // assertion failure
             ^
/tmp/assertfail.zig:5:11: 0x100fdbfef in copyInto (assertfail)
    assert(dst.len >= src.len);
          ^
/tmp/assertfail.zig:11:13: 0x100fdb85b in main (assertfail)
```

The trace shows the implementation: `std.debug.assert` is *literally*
`if (!ok) unreachable;`. It is not a separate mechanism, it inherits
`unreachable`'s release-mode behaviour exactly, and it takes no message. For a
message, use `if (!ok) std.debug.panic("...", .{});`.

## API design for a backend service

```zig
const std = @import("std");

const UserErr = error{ DbUnavailable, Malformed };

const User = struct { id: u32, name: []const u8 };

const Db = struct {
    up: bool,
    rows: []const User,

    fn findUser(db: Db, id: u32) UserErr!?User {
        if (!db.up) return error.DbUnavailable;
        for (db.rows) |u| {
            if (u.id == id) {
                if (u.name.len == 0) return error.Malformed;
                return u;
            }
        }
        return null;
    }

    fn userCount(db: Db) usize {
        return db.rows.len;
    }
};

fn httpStatus(db: Db, id: u32) u16 {
    const found = db.findUser(id) catch |e| switch (e) {
        error.DbUnavailable => return 503,
        error.Malformed => return 500,
    };
    return if (found == null) 404 else 200;
}

pub fn main() void {
    const up: Db = .{
        .up = true,
        .rows = &.{ .{ .id = 1, .name = "ada" }, .{ .id = 2, .name = "" } },
    };
    const down: Db = .{ .up = false, .rows = &.{} };

    std.debug.print("count       = {d}\n", .{up.userCount()});
    std.debug.print("find(1)     = {!?any}\n", .{up.findUser(1)});
    std.debug.print("find(9)     = {!?any}\n", .{up.findUser(9)});
    std.debug.print("find(2)     = {!?any}\n", .{up.findUser(2)});
    std.debug.print("status(1)   = {d}\n", .{httpStatus(up, 1)});
    std.debug.print("status(9)   = {d}\n", .{httpStatus(up, 9)});
    std.debug.print("status(2)   = {d}\n", .{httpStatus(up, 2)});
    std.debug.print("status down = {d}\n", .{httpStatus(down, 1)});
}
```

```
count       = 2
find(1)     = .{ .id = 1, .name = { 97, 100, 97 } }
find(9)     = null
find(2)     = error.Malformed
status(1)   = 200
status(9)   = 404
status(2)   = 500
status down = 503
```

`UserErr!?User` maps onto HTTP exactly:

| Result | Status |
|---|---|
| a `User` | 200 |
| `null` | **404** — a normal outcome, not an error |
| `error.DbUnavailable` | 503 |
| `error.Malformed` | 500 |

If `findUser` returned `error.NotFound` instead of `null`, the 404 and the 503
would arrive through the same channel and the handler would have to re-separate
them. The type already did that work. And because `UserErr` is a **closed,
explicit** set, that `switch` in `httpStatus` is exhaustive — add
`error.Timeout` to the data layer and the handler fails to compile until someone
assigns it a status code. That is the property you want in a service: new failure
modes cannot silently become 500s.

### Checklist

- **Return `?T`** when "not there" is an answer: lookups, searches, `next()`,
  optional config.
- **Return `!T`** when the caller must know *why*: I/O, parsing, allocation,
  validation.
- **Return `!?T`** when both apply. Don't flatten "missing" into an error just to
  simplify the signature.
- **Explicit error sets on public functions**, inferred `!T` internally — the
  exhaustive `switch` at the boundary is the payoff.
- **Name errors for the cause, not the fix**: `error.ConnectionRefused`, not
  `error.ShouldRetry`.
- **Don't add `error.NotFound`** to a function that already returns `?T`.
- **No `anyerror` in a public signature** — it tells the caller nothing and makes
  exhaustive handling impossible.
- **`unreachable` / `assert` for invariants only**, never for input validation —
  they are compiled out of the build you ship.
- `errdefer` on every acquisition you hand to the caller.
- Log `{t}` plus context where you handle an error; error return traces are
  Debug-only.

---

That's failure handling in Zig: errors are values with types, absence has its own
type, cleanup is a line you write, and the compiler refuses to let you ignore any
of it.

Next: [Module 05 — Memory & Allocators](../05-memory-allocators/)
