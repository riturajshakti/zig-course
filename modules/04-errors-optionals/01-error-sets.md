# 4.1 — Error Sets

Zig has no exceptions. An error is a **value** with a type, returned normally,
and the compiler forces you to deal with it.

```zig
const std = @import("std");

const FileErr = error{ NotFound, Denied };

pub fn main() void {
    const e: FileErr = error.NotFound;
    std.debug.print("value   = {t}\n", .{e});
    std.debug.print("name    = {s}\n", .{@errorName(e)});
    std.debug.print("type    = {s}\n", .{@typeName(FileErr)});
    std.debug.print("as int  = {d}\n", .{@intFromError(e)});
    std.debug.print("eq      = {}\n", .{e == error.NotFound});
}
```

```
value   = NotFound
name    = NotFound
type    = error{Denied,NotFound}
as int  = 143
eq      = true
```

`error{ NotFound, Denied }` declares a **type**, like an enum. `error.NotFound`
is one of its values. It compares with `==`, it switches, it has a name — it is
just data. The integer is an implementation detail (the compiler assigns global
ids; don't depend on the number).

## `!T` — the error union

```zig
const std = @import("std");

const FileErr = error{ NotFound, Denied };

fn open(path: []const u8) FileErr!u32 {
    if (path.len == 0) return error.NotFound;
    if (path[0] == '/') return error.Denied;
    return @intCast(path.len);
}

pub fn main() void {
    std.debug.print("open(\"x.txt\") = {!d}\n", .{open("x.txt")});
    std.debug.print("open(\"\")      = {!d}\n", .{open("")});
    std.debug.print("open(\"/etc\")  = {!d}\n", .{open("/etc")});
}
```

```
open("x.txt") = 5
open("")      = error.NotFound
open("/etc")  = error.Denied
```

`FileErr!u32` is a single value that is **either** a `u32` **or** one of two
errors. `{!d}` prints whichever it holds.

| Written | Means |
|---|---|
| `FileErr!u32` | explicit error set, payload `u32` |
| `!u32` | **inferred** error set — compiler works it out |
| `anyerror!u32` | any error in the program |
| `FileErr!void` | fails or succeeds with no value |
| `error{}!u32` | cannot fail (the empty set) |

Note `return error.NotFound;` — no wrapping, no `Err(...)` constructor. An error
coerces into the union automatically.

## Merging with `||`

```zig
const std = @import("std");

const FileErr = error{ NotFound, Denied };
const NetErr = error{ Timeout, Refused };
const AllErr = FileErr || NetErr;

pub fn main() void {
    std.debug.print("{s}\n", .{@typeName(AllErr)});
    const e: AllErr = error.Timeout;
    std.debug.print("{s} fits in AllErr\n", .{@errorName(e)});
    const f: FileErr = error.Denied;
    const g: AllErr = f;
    std.debug.print("FileErr -> AllErr: {s}\n", .{@errorName(g)});
}
```

```
error{Denied,NotFound,Refused,Timeout}
Timeout fits in AllErr
FileErr -> AllErr: Denied
```

`||` on two error *types* is set union, at compile time. (`@typeName` lists
members alphabetically, not in declaration order.)

Widening is implicit: `FileErr` → `AllErr` needs no cast. **Narrowing is a
compile error**, and this is the whole point of the design:

```zig
const std = @import("std");

const FileErr = error{ NotFound, Denied };
const NetErr = error{ Timeout, Refused };
const AllErr = FileErr || NetErr;

pub fn main() void {
    const wide: AllErr = error.Timeout;
    const narrow: FileErr = wide;
    std.debug.print("{s}\n", .{@errorName(narrow)});
}
```

```sh
zig run narrow.zig
```

**Deliberate failure — exit code 1, nothing runs:**

```
narrow.zig:9:29: error: expected type 'error{Denied,NotFound}', found 'error{Denied,NotFound,Refused,Timeout}'
    const narrow: FileErr = wide;
                            ^~~~
narrow.zig:9:29: note: 'error.Timeout' not a member of destination error set
narrow.zig:9:29: note: 'error.Refused' not a member of destination error set
```

It names the two members that don't fit. Compare with Go, where
`errors.Is(err, ErrTimeout)` is a runtime question your caller may forget to
ask; here the set is part of the signature and the compiler does the asking.

## Inferred error sets

Leave the set off and write just `!T`. The compiler unions every error you
actually return — **including those from functions you `try`**:

```zig
const std = @import("std");

fn parseAge(s: []const u8) !u8 {
    if (s.len == 0) return error.Empty;
    const n = try std.fmt.parseInt(u16, s, 10);
    if (n > 150) return error.TooBig;
    return @intCast(n);
}

pub fn main() void {
    std.debug.print("\"42\"   -> {!d}\n", .{parseAge("42")});
    std.debug.print("\"\"     -> {!d}\n", .{parseAge("")});
    std.debug.print("\"999\"  -> {!d}\n", .{parseAge("999")});
    std.debug.print("\"abc\"  -> {!d}\n", .{parseAge("abc")});
}
```

```
"42"   -> 42
""     -> error.Empty
"999"  -> error.TooBig
"abc"  -> error.InvalidCharacter
```

`error.InvalidCharacter` was never written in `parseAge` — it came out of
`std.fmt.parseInt` and the inferred set picked it up. You can read the set back
with reflection:

```zig
const std = @import("std");

fn parseAge(s: []const u8) !u8 {
    if (s.len == 0) return error.Empty;
    const n = try std.fmt.parseInt(u16, s, 10);
    if (n > 150) return error.TooBig;
    return @intCast(n);
}

const Ret = @typeInfo(@TypeOf(parseAge)).@"fn".return_type.?;
const Inferred = @typeInfo(Ret).error_union.error_set;
const names = @typeInfo(Inferred).error_set.error_names.?;

pub fn main() void {
    std.debug.print("{d} errors inferred:\n", .{names.len});
    inline for (names) |name| std.debug.print("  error.{s}\n", .{name});
    std.debug.print("anyerror.error_names = {?any}\n", .{@typeInfo(anyerror).error_set.error_names});
}
```

```
4 errors inferred:
  error.Overflow
  error.InvalidCharacter
  error.Empty
  error.TooBig
anyerror.error_names = null
```

Four errors, two of them inherited. `anyerror`'s `error_names` is `null` because
it is the **open** set — not a list, but "all of them".

| | Inferred `!T` | Explicit `E!T` |
|---|---|---|
| Set grows when a callee adds an error | silently | compile error at the `try` |
| Good for | internal / leaf functions | **public API boundaries** |
| Shows in docs & signature | no | yes |
| Caller can `switch` exhaustively | yes, but set may shift | yes, stable |

Rule of thumb: inferred inside your module, explicit on anything other people
call. An inferred set is a contract you didn't write down.

## `anyerror`

```zig
const std = @import("std");

const FileErr = error{ NotFound, Denied };

fn narrow(n: i32) FileErr!i32 {
    if (n < 0) return error.NotFound;
    return n;
}

fn wide(n: i32) anyerror!i32 {
    if (n < 0) return error.AnythingAtAll;
    return try narrow(n);
}

pub fn main() void {
    std.debug.print("wide(7)  = {!d}\n", .{wide(7)});
    std.debug.print("wide(-1) = {!d}\n", .{wide(-1)});
    const stored: anyerror = error.NotFound;
    std.debug.print("stored   = {t}\n", .{stored});
}
```

```
wide(7)  = 7
wide(-1) = error.AnythingAtAll
stored   = NotFound
```

`anyerror` is the global error set — every error anywhere in the program,
including ones from code you haven't written yet. It accepts anything, which
means it tells a caller nothing. Use it for logging, storing an error in a
struct field, or `catch |e|` handling. Don't put it in a function signature you
want people to rely on.

## Error sets are comptime-known

```zig
const std = @import("std");

const Level = error{ Low, High };

pub fn main() void {
    const count = comptime blk: {
        var n: usize = 0;
        for (@typeInfo(Level).error_set.error_names.?) |_| n += 1;
        break :blk n;
    };
    std.debug.print("Level has {d} members, known at compile time\n", .{count});

    const e: Level = error.High;
    switch (e) {
        error.Low => std.debug.print("low\n", .{}),
        error.High => std.debug.print("high\n", .{}),
    }
}
```

```
Level has 2 members, known at compile time
high
```

That `switch` has no `else` and compiles — a closed error set is **exhaustive**,
exactly like an enum. Add a third member to `Level` and this stops building.

> **`@hasField` does NOT work on error sets.**
>
> ```zig
> const std = @import("std");
>
> const FileErr = error{ NotFound, Denied };
>
> pub fn main() void {
>     std.debug.print("{}\n", .{@hasField(FileErr, "Denied")});
> }
> ```
>
> **Deliberate failure — exit code 1:**
>
> ```
> hasfield.zig:6:41: error: type 'error{Denied,NotFound}' does not support '@hasField'
>     std.debug.print("{}\n", .{@hasField(FileErr, "Denied")});
>                                         ^~~~~~~
> ```
>
> Error sets aren't structs. Reach for `@errorName`, or the
> `@typeInfo(E).error_set.error_names` list above.

## Cheat sheet

| Expression | Result |
|---|---|
| `error{A, B}` | declares an error set type |
| `error.A` | an error value |
| `E!T` | error union: `E` or `T` |
| `!T` | error union with inferred set |
| `A \|\| B` | merged error set (compile time) |
| `anyerror` | the global, open error set |
| `@errorName(e)` | `[]const u8` name, e.g. `"Denied"` |
| `{t}` on an error | `Denied` |
| `{!d}` on `E!i32` | `5` or `error.Denied` |
| `@intFromError(e)` | its internal id — don't persist it |
| `@typeInfo(E).error_set.error_names` | `?[]const [:0]const u8`, `null` for `anyerror` |

---

Next: [4.2 — `try` & `catch`](./02-try-catch.md)
