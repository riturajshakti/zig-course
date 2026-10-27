# 4.2 — `try` & `catch`

Two keywords, and they are not the two you know from other languages.

| Keyword | Means |
|---|---|
| `try expr` | on error, **return it from the current function** |
| `expr catch v` | on error, use `v` instead |
| `expr catch \|e\| { ... }` | on error, run a block with the error value |

There is no `try { }` block, nothing is thrown, and nothing unwinds.

## `try` is pure shorthand

```zig
const std = @import("std");

const MathErr = error{ DivZero, Negative };

fn sqrtInt(n: i32) MathErr!i32 {
    if (n < 0) return error.Negative;
    var r: i32 = 0;
    while ((r + 1) * (r + 1) <= n) r += 1;
    return r;
}

fn manual(a: i32) MathErr!i32 {
    const x = sqrtInt(a) catch |e| return e;
    return x * 10;
}

pub fn main() void {
    std.debug.print("manual(25) = {!d}\n", .{manual(25)});
    std.debug.print("manual(-5) = {!d}\n", .{manual(-5)});
}
```

```
manual(25) = 50
manual(-5) = error.Negative
```

`try sqrtInt(a)` is *exactly* `sqrtInt(a) catch |e| return e`. That's the entire
definition. Go's `if err != nil { return err }` compressed into three letters.

## Propagating

```zig
const std = @import("std");

const MathErr = error{ DivZero, Negative };

fn sqrtInt(n: i32) MathErr!i32 {
    if (n < 0) return error.Negative;
    var r: i32 = 0;
    while ((r + 1) * (r + 1) <= n) r += 1;
    return r;
}

fn hypotLike(a: i32, b: i32) MathErr!i32 {
    const x = try sqrtInt(a);
    const y = try sqrtInt(b);
    return x + y;
}

pub fn main() void {
    std.debug.print("hypotLike(9, 16)  = {!d}\n", .{hypotLike(9, 16)});
    std.debug.print("hypotLike(9, -1)  = {!d}\n", .{hypotLike(9, -1)});
    std.debug.print("hypotLike(-1, 16) = {!d}\n", .{hypotLike(-1, 16)});
}
```

```
hypotLike(9, 16)  = 7
hypotLike(9, -1)  = error.Negative
hypotLike(-1, 16) = error.Negative
```

`try` only compiles if the enclosing function can return that error. `try` in a
`fn f() void` is a compile error — you cannot accidentally swallow or escape a
failure.

## `catch` variations

```zig
const std = @import("std");

fn parse(s: []const u8) !i32 {
    return std.fmt.parseInt(i32, s, 10);
}

pub fn main() void {
    const a = parse("42") catch -1;
    const b = parse("oops") catch -1;
    std.debug.print("a = {d}\n", .{a});
    std.debug.print("b = {d}\n", .{b});

    const c = parse("x") catch blk: {
        std.debug.print("  (recovering)\n", .{});
        break :blk 0;
    };
    std.debug.print("c = {d}\n", .{c});

    const d = parse("7") catch unreachable;
    std.debug.print("d = {d}\n", .{d});
}
```

```
a = 42
b = -1
  (recovering)
c = 0
d = 7
```

| Form | Behaviour |
|---|---|
| `catch -1` | default value, error discarded |
| `catch blk: { ... break :blk 0; }` | labeled block computes the default |
| `catch unreachable` | "this cannot fail" — **panics in Debug if it does** |
| `catch {}` | only valid when the payload is `void` |
| `catch \|e\| return e` | what `try` expands to |
| `catch \|e\| { ...; return; }` | handle and bail out of this function |

## `catch |e|` with a block

```zig
const std = @import("std");

fn parse(s: []const u8) !i32 {
    return std.fmt.parseInt(i32, s, 10);
}

pub fn main() void {
    const a = parse("99999999999") catch |e| {
        std.debug.print("logged: {t}\n", .{e});
        return;
    };
    std.debug.print("unreachable here, a = {d}\n", .{a});
}
```

```
logged: Overflow
```

The block must either produce a value of the payload type or leave the scope
(`return`, `break`, `continue`, `unreachable`). Here it returns, so the second
print never runs.

## `catch |e| switch (e)` — the real workhorse

```zig
const std = @import("std");

fn parse(s: []const u8) !i32 {
    if (s.len == 0) return error.Empty;
    return std.fmt.parseInt(i32, s, 10);
}

fn lenient(s: []const u8) i32 {
    return parse(s) catch |e| switch (e) {
        error.Empty => 0,
        error.InvalidCharacter => -1,
        error.Overflow => std.math.maxInt(i32),
    };
}

pub fn main() void {
    const cases = [_][]const u8{ "12", "", "abc", "99999999999" };
    for (cases) |c| std.debug.print("{s:<12} -> {d}\n", .{ c, lenient(c) });
}
```

```
12           -> 12
             -> 0
abc          -> -1
99999999999  -> 2147483647
```

No `else` branch — and it compiles. The error set is closed and comptime-known,
so the switch is checked for exhaustiveness. Add an error to `parse` and this
function **stops building** until you decide what it means. That is the feature:
"handle every failure mode" is enforced by the compiler, not by a code reviewer.

Mixed handling, where some errors are handled and the rest propagate:

```zig
const std = @import("std");

fn parse(s: []const u8) !i32 {
    if (s.len == 0) return error.Empty;
    return std.fmt.parseInt(i32, s, 10);
}

fn strict(s: []const u8) !i32 {
    return parse(s) catch |e| switch (e) {
        error.Empty => 0,
        else => return e,
    };
}

pub fn main() void {
    std.debug.print("strict(\"\")    = {!d}\n", .{strict("")});
    std.debug.print("strict(\"5\")   = {!d}\n", .{strict("5")});
    std.debug.print("strict(\"abc\") = {!d}\n", .{strict("abc")});
}
```

```
strict("")    = 0
strict("5")   = 5
strict("abc") = error.InvalidCharacter
```

## `if (x) |v| else |e|` — inspect without committing

```zig
const std = @import("std");

const Err = error{ NotFound, Denied };

fn lookup(k: []const u8) Err!i32 {
    if (std.mem.eql(u8, k, "a")) return 1;
    if (std.mem.eql(u8, k, "b")) return error.Denied;
    return error.NotFound;
}

pub fn main() void {
    for ([_][]const u8{ "a", "b", "z" }) |k| {
        if (lookup(k)) |v| {
            std.debug.print("{s}: ok  {d}\n", .{ k, v });
        } else |e| {
            std.debug.print("{s}: err {t}\n", .{ k, e });
        }
    }

    const got = if (lookup("a")) |v| v * 100 else |_| -1;
    std.debug.print("expr form = {d}\n", .{got});
}
```

```
a: ok  1
b: err Denied
z: err NotFound
expr form = 100
```

Note the pipes on **both** branches: `|v|` captures the payload, `|e|` captures
the error. Use `|_|` to ignore one. This is the form to reach for when both
paths are equally normal; `catch` is for when the error path is the exception.

## When an error escapes `main`

```zig
const std = @import("std");

const ConfigErr = error{ MissingPort, BadHost };

fn loadConfig() ConfigErr!u16 {
    return error.MissingPort;
}

pub fn main() !void {
    const port = try loadConfig();
    std.debug.print("port = {d}\n", .{port});
}
```

```sh
zig run badconfig.zig
```

**Deliberate failure.** This is what a real unhandled error looks like:

```
error: MissingPort
/tmp/badconfig.zig:6:5: 0x100ba0c37 in loadConfig (badconfig)
    return error.MissingPort;
    ^
/tmp/badconfig.zig:10:18: 0x100b9f633 in main (badconfig)
    const port = try loadConfig();
                 ^
exit code: 1
```

The runtime prints the error name to stderr, dumps the chain of `return`s that
produced it, and exits **1**. Your paths and hex addresses will differ — those
are absolute paths and ASLR-randomized addresses.

| `main` signature | An error... |
|---|---|
| `pub fn main() !void` | prints `error: Name` + trace, exit code 1 |
| `pub fn main() void` | won't compile if you `try` inside it |
| `pub fn main() u8` | you return your own exit code |

Returning an error from `main` is a perfectly good CLI design — you get a
diagnostic and a nonzero exit for free. Catch it only when you want a friendlier
message:

```zig
const std = @import("std");

const ConfigErr = error{ MissingPort, BadHost };

fn loadConfig() ConfigErr!u16 {
    return error.MissingPort;
}

pub fn main() u8 {
    const port = loadConfig() catch |e| switch (e) {
        error.MissingPort => {
            std.debug.print("config: set PORT and retry\n", .{});
            return 2;
        },
        error.BadHost => {
            std.debug.print("config: HOST is malformed\n", .{});
            return 3;
        },
    };
    std.debug.print("listening on {d}\n", .{port});
    return 0;
}
```

```
config: set PORT and retry
```

```sh
echo $?
```

```
2
```

---

Next: [4.3 — `defer` & `errdefer`](./03-defer-errdefer.md)
