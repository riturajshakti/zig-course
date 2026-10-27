# 4.4 — Error Traces

An **error return trace** is not a stack trace. A stack trace says "here is where
the program is". An error return trace says "here are the `return`s that carried
this error up to you" — and those are different lists, because the error travels
*after* the failing frame has already gone.

## What an escaping error prints

```zig
const std = @import("std");

fn readByte() !u8 {
    return error.EndOfStream;
}

fn readHeader() !u32 {
    const b = try readByte();
    return b;
}

fn loadFile() !u32 {
    return try readHeader();
}

pub fn main() !void {
    const n = try loadFile();
    std.debug.print("n = {d}\n", .{n});
}
```

```sh
zig run trace.zig
```

**Deliberate failure.** Real output, exit code **1**:

```
error: EndOfStream
/tmp/trace.zig:4:5: 0x10448357f in readByte (trace)
    return error.EndOfStream;
    ^
/tmp/trace.zig:8:15: 0x104483647 in readHeader (trace)
    const b = try readByte();
              ^
/tmp/trace.zig:13:12: 0x104483737 in loadFile (trace)
    return try readHeader();
           ^
/tmp/trace.zig:17:15: 0x10448384b in main (trace)
    const n = try loadFile();
              ^
```

Four entries, bottom-up order of *origin* first:

| Entry | Points at |
|---|---|
| `readByte` line 4 | the `return error.X` that **created** the error |
| `readHeader` line 8 | the `try` that forwarded it |
| `loadFile` line 13 | the `try` that forwarded it |
| `main` line 17 | the `try` that let it escape |

Every `try` adds a frame. No exception object, no `.stack` string — the runtime
recorded return addresses and resolved them against your debug info. Paths are
absolute and the hex addresses are ASLR-randomized, so **yours will differ** on
every run.

## It records only the error that propagated

```zig
const std = @import("std");

fn leaf() !u32 {
    return error.Deep;
}

fn retried() !u32 {
    if (leaf()) |v| return v else |_| {}
    return try leaf();
}

pub fn main() !void {
    const v = try retried();
    std.debug.print("{d}\n", .{v});
}
```

**Deliberate failure**, exit code 1:

```
error: Deep
/tmp/retry.zig:4:5: 0x10051f57f in leaf (retry)
    return error.Deep;
    ^
/tmp/retry.zig:9:12: 0x10051f6d3 in retried (retry)
    return try leaf();
           ^
/tmp/retry.zig:13:15: 0x10051f7e7 in main (retry)
    const v = try retried();
              ^
```

`leaf()` failed **twice**. The first failure was caught and discarded, so it is
not in the trace — only the second, which was `try`-ed and actually travelled.
Line 9 appears, line 8 does not. Handling an error ends its trace.

## Reading the trace yourself

```zig
const std = @import("std");

fn inner() !void {
    return error.Nope;
}

fn middle() !void {
    try inner();
}

pub fn main() void {
    middle() catch |e| {
        std.debug.print("caught {t}; return trace so far:\n", .{e});
        if (@errorReturnTrace()) |trace| {
            std.debug.dumpErrorReturnTrace(trace);
        } else {
            std.debug.print("(no trace in this build mode)\n", .{});
        }
    };
}
```

```
caught Nope; return trace so far:
/tmp/dump.zig:4:5: 0x104f4b81f in inner (dump)
    return error.Nope;
    ^
/tmp/dump.zig:8:5: 0x104f4b88b in middle (dump)
    try inner();
    ^
```

`@errorReturnTrace()` is a builtin returning `?*std.builtin.StackTrace` — `null`
when the build mode doesn't track traces. Log it inside a `catch` and you get the
origin of a failure you decided to swallow.

> **Two different `StackTrace` types, and they don't cast.** `dumpStackTrace`
> takes `*const std.debug.StackTrace` (a `return_addresses` list, for *stack*
> traces from `captureCurrentStackTrace` / `dumpCurrentStackTrace`), while
> `@errorReturnTrace()` gives `*std.builtin.StackTrace`
> (an `instruction_addresses` list). Passing the latter to `dumpStackTrace` is a
> compile error:
>
> ```
> error: expected type '*const debug.StackTrace', found '*lang.StackTrace'
> ```
>
> Use **`dumpErrorReturnTrace`** for `@errorReturnTrace()`, and
> `std.debug.dumpCurrentStackTrace(.{})` when you want where-am-I-now.

## Formatting errors

```zig
const std = @import("std");

const Err = error{ Bad, Worse };

fn num(fail: bool) Err!i32 {
    if (fail) return error.Bad;
    return 42;
}

fn text(fail: bool) Err![]const u8 {
    if (fail) return error.Worse;
    return "hello";
}

pub fn main() void {
    std.debug.print("{!d}\n", .{num(false)});
    std.debug.print("{!d}\n", .{num(true)});
    std.debug.print("{!s}\n", .{text(false)});
    std.debug.print("{!s}\n", .{text(true)});

    const e: Err = error.Worse;
    std.debug.print("{t}\n", .{e});
    std.debug.print("{s}\n", .{@errorName(e)});

    const u: Err!i32 = error.Bad;
    std.debug.print("{!}\n", .{u});
    std.debug.print("{!any}\n", .{u});
}
```

```
42
error.Bad
hello
error.Worse
Worse
Worse
error.Bad
error.Bad
```

| Specifier | On | Prints |
|---|---|---|
| `{!d}` | `E!i32` | `42` or `error.Bad` |
| `{!s}` | `E![]const u8` | `hello` or `error.Worse` |
| `{!}` `{!any}` | `E!T` | payload via `{any}`, else `error.Bad` |
| `{t}` | a plain error value | `Worse` — **no `error.` prefix** |
| `@errorName(e)` | a plain error value | `"Worse"` as a `[]const u8` |

The `!` is a *prefix modifier*: `{!d}` means "unwrap the error union, then format
the payload with `d`". `{d}` on an error union is a compile error. For logging,
`{t}` is the one you want inside `catch |e|`; `{!d}` is for printing a call
result without unwrapping it first.

## Panic vs error — different thing entirely

An error is a value you must handle. A panic is a bug, and it aborts.

```zig
const std = @import("std");

fn pick(items: []const u32, i: usize) u32 {
    return items[i];
}

pub fn main() void {
    const xs = [_]u32{ 10, 20, 30 };
    std.debug.print("xs[1] = {d}\n", .{pick(&xs, 1)});
    std.debug.print("xs[9] = {d}\n", .{pick(&xs, 9)});
}
```

**Deliberate failure.** Real output, exit code **134** (SIGABRT):

```
xs[1] = 20
thread 16426381 panic: index out of bounds: index 9, len 3
/tmp/panic.zig:4:17: 0x1044d0c3b in pick (panic)
    return items[i];
                ^
/tmp/panic.zig:10:44: 0x1044cf883 in main (panic)
    std.debug.print("xs[9] = {d}\n", .{pick(&xs, 9)});
                                           ^
/opt/.../std/start.zig:788:64: 0x1044cf7bf in callMain (panic)
    if (fn_info.param_types.len == 0) return wrapMain(root.main());
                                                               ^
???:?:?: 0x180407dff in start (/usr/lib/dyld)
```

| | Error return trace | Panic stack trace |
|---|---|---|
| Says | path the error travelled | frames currently on the stack |
| Stops at | `main` | the process entry point (`start`, dyld) |
| Includes runtime internals | no | yes (`callMain`, `start`) |
| Exit code | `1` | `134` |
| Catchable | yes | **no** |
| Header | `error: EndOfStream` | `thread N panic: ...` |

The panic trace includes `callMain` and dyld's `start`; the error trace stops at
your `main` because the runtime's frames never `try`-ed anything.

## Release-mode differences

Same `trace.zig` as at the top, built four ways:

```sh
zig run -O Debug        trace.zig
zig run -O ReleaseSafe  trace.zig
zig run -O ReleaseFast  trace.zig
zig run -O ReleaseSmall trace.zig
```

```
error: EndOfStream
/tmp/trace.zig:4:5: 0x1041bb57f in readByte (trace)
    return error.EndOfStream;
    ^
/tmp/trace.zig:8:15: 0x1041bb647 in readHeader (trace)
    const b = try readByte();
              ^
/tmp/trace.zig:13:12: 0x1041bb737 in loadFile (trace)
    return try readHeader();
           ^
/tmp/trace.zig:17:15: 0x1041bb84b in main (trace)
    const n = try loadFile();
              ^
```

```
error: EndOfStream
```

```
error: EndOfStream
```

```
error: EndOfStream
```

| Mode | Return trace | Exit code |
|---|---|---|
| `Debug` | full, with source lines | 1 |
| `ReleaseSafe` | **none** — just `error: Name` | 1 |
| `ReleaseFast` | none | 1 |
| `ReleaseSmall` | none | 1 |

Only `Debug` tracks error return traces. Each `try` on the error path otherwise
costs a stored return address, so release builds drop the bookkeeping entirely —
and `@errorReturnTrace()` returns `null` there, which is why the `if (...) |t|`
in the dump example is required rather than decorative.

**Operational consequence for a backend:** error return traces are a development
tool. In production (`ReleaseSafe`/`ReleaseFast`) you get only the error name, so
your own logging has to carry the context — log `{t}` plus the request id at the
boundary where you handle the error, not where it was created.

```zig
const std = @import("std");

const ReqErr = error{ BadJson, NoSuchUser };

fn handle(id: u32, path: []const u8) ReqErr!void {
    if (std.mem.eql(u8, path, "/users/0")) return error.NoSuchUser;
    if (std.mem.eql(u8, path, "/bad")) return error.BadJson;
    std.debug.print("req {d}: 200 {s}\n", .{ id, path });
}

pub fn main() void {
    const reqs = [_][]const u8{ "/users/7", "/users/0", "/bad" };
    for (reqs, 1..) |p, i| {
        handle(@intCast(i), p) catch |e| {
            std.debug.print("req {d}: 500 {s} err={t}\n", .{ i, p, e });
        };
    }
}
```

```
req 1: 200 /users/7
req 2: 500 /users/0 err=NoSuchUser
req 3: 500 /bad err=BadJson
```

That log line survives `ReleaseFast`. A trace does not.

---

Next: [4.5 — Optionals](./05-optionals.md)
