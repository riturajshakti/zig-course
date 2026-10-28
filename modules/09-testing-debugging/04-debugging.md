# 9.4 — Debugging

Two output paths, and they are not interchangeable:

| | `std.debug.print` | `std.log.*` |
|---|---|---|
| destination | stderr, unbuffered | stderr via `std.options.logFn` |
| needs `io` / flush | no / no | no / no |
| has levels | no | `err` `warn` `info` `debug` |
| filterable at compile time | no | yes — by level and by scope |
| present in a release build | **always** | only levels at or above `log_level` |
| intended for | temporary debugging | the program's real diagnostics |

**Rule: `std.debug.print` is for lines you will delete. `std.log` is for lines
you will ship.** A `std.debug.print` left in a release binary is a bug.

## `std.log` levels

```zig
const std = @import("std");

pub fn main() void {
    std.log.err("disk is full: {d} bytes free", .{0});
    std.log.warn("retrying in {d}ms", .{250});
    std.log.info("listening on port {d}", .{8080});
    std.log.debug("cache key = {s}", .{"user:17"});
}
```

```sh
zig run lg1.zig
```

```
error: disk is full: 0 bytes free
warning: retrying in 250ms
info: listening on port 8080
debug: cache key = user:17
```

The default level **depends on the build mode** — `.debug` in Debug, `.info`
in the three release modes:

```sh
zig run -Ofast lg1.zig
```

```
error: disk is full: 0 bytes free
warning: retrying in 250ms
info: listening on port 8080
```

The `debug` line is gone, and not at runtime: `std.log.debug` compiles to
nothing when the level excludes it, so its arguments are never even evaluated.
Logging is free when it's off.

## `std.options.log_level`

Override by declaring a **public** `std_options` in your root file. The name is
exact; a typo silently does nothing.

```zig
const std = @import("std");

pub const std_options: std.Options = .{
    .log_level = .warn,
};

pub fn main() void {
    std.log.err("shown", .{});
    std.log.warn("shown", .{});
    std.log.info("hidden", .{});
    std.log.debug("hidden", .{});
}
```

```sh
zig run lg2.zig
```

```
error: shown
warning: shown
```

## Scopes

`std.log.scoped(.name)` gives a namespaced logger, and scopes get **their own
levels** — the per-subsystem verbosity knob.

```zig
const std = @import("std");

pub const std_options: std.Options = .{
    .log_level = .info,
    .log_scope_levels = &.{
        .{ .scope = .db, .level = .debug },
        .{ .scope = .http, .level = .err },
    },
};

const db = std.log.scoped(.db);
const http = std.log.scoped(.http);

pub fn main() void {
    db.debug("SELECT 1", .{});
    db.info("connected", .{});
    http.info("hidden: http is err-only", .{});
    http.err("502 upstream", .{});
    std.log.info("default scope at info", .{});
    std.log.debug("hidden: default scope at info", .{});
}
```

```sh
zig run lg3.zig
```

```
debug(db): SELECT 1
info(db): connected
error(http): 502 upstream
info: default scope at info
```

Four of six lines survive. The scope appears in parentheses; the unscoped
functions use `.default`, which prints no parentheses.

## A custom `logFn`

One function replaces the whole formatter. Structured JSON logs in 15 lines:

```zig
const std = @import("std");

pub const std_options: std.Options = .{
    .log_level = .debug,
    .logFn = jsonLog,
};

fn jsonLog(
    comptime level: std.log.Level,
    comptime scope: @EnumLiteral(),
    comptime format: []const u8,
    args: anytype,
) void {
    var buf: [512]u8 = undefined;
    const msg = std.fmt.bufPrint(&buf, format, args) catch "<message too long>";
    std.debug.print(
        "{{\"level\":\"{s}\",\"scope\":\"{t}\",\"msg\":\"{s}\"}}\n",
        .{ level.asText(), scope, msg },
    );
}

const http = std.log.scoped(.http);

pub fn main() void {
    std.log.info("server started", .{});
    http.warn("slow request: {d}ms", .{1830});
    std.log.err("shutting down", .{});
}
```

```sh
zig run lg4.zig
```

```
{"level":"info","scope":"default","msg":"server started"}
{"level":"warning","scope":"http","msg":"slow request: 1830ms"}
{"level":"error","scope":"default","msg":"shutting down"}
```

| Detail | Why |
|---|---|
| all three of `level`, `scope`, `format` are `comptime` | the signature is fixed by `std.Options`; copy it exactly |
| `scope: @EnumLiteral()` | not an `enum` type — `{t}` prints its name |
| `level.asText()` | `"error"`, `"warning"`, `"info"`, `"debug"` |
| `{{` and `}}` | escaped braces for literal JSON |
| no `io` parameter | `logFn` can be called from anywhere, including a panic handler |

Forward to the default instead of replacing it by calling
`std.log.defaultLog(level, scope, format, args)` from your own `logFn`.

## `@panic` and `std.debug.panic`

`@panic` takes a string; `std.debug.panic` takes a format string and args.

```zig
const std = @import("std");

fn configValue(key: []const u8) []const u8 {
    if (std.mem.eql(u8, key, "port")) return "8080";
    std.debug.panic("unknown config key: {s}", .{key});
}

pub fn main() void {
    std.debug.print("port = {s}\n", .{configValue("port")});
    std.debug.print("host = {s}\n", .{configValue("host")});
}
```

```sh
zig run p1.zig; echo "exit=$?"
```

**Deliberate panic:**

```
port = 8080
thread 16620780 panic: unknown config key: host
/path/p1.zig:5:20: 0x1005e2073 in configValue (p1)
    std.debug.panic("unknown config key: {s}", .{key});
                   ^
/path/p1.zig:10:50: 0x1005e11df in main (p1)
    std.debug.print("host = {s}\n", .{configValue("host")});
                                                 ^
/opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/start.zig:788:64: 0x1005e1123 in callMain (p1)
    if (fn_info.param_types.len == 0) return wrapMain(root.main());
                                                               ^
???:?:?: 0x180407dff in start (/usr/lib/dyld)
exit=134
```

Reading a panic trace:

| Part | Means |
|---|---|
| `thread 16620780 panic: <message>` | thread id + the reason |
| **first** frame after it | where it happened |
| each following frame | its caller, outward |
| `callMain` / `start` | the runtime's entry; everything below is not yours |
| `???:?:?` | no debug info (system dylib) |
| `exit=134` | 128 + SIGABRT(6) — panics abort |

Frames print **innermost first**, opposite to a Python traceback. The hex
addresses are ASLR-randomised and differ every run; read the source lines.

Panic messages the compiler can produce on its own, all with the same layout:

| Message | Cause |
|---|---|
| `integer overflow` | `a + b` past the type's range |
| `index out of bounds: index 5, len 3` | slice/array index |
| `reached unreachable code` | `unreachable`, or a failed `std.debug.assert` |
| `division by zero` | `a / 0` |
| `attempt to unwrap null` | `x.?` on a null optional |
| `attempt to unwrap error: Foo` | `catch unreachable` on an error |
| `access of union field 'f' while field 'g' is active` | wrong field on a bare union |
| `cast truncated bits` | a lossy `@intCast` |

## `std.debug.assert`

`assert(cond)` is `if (!cond) unreachable;` — not a function call, not a
message, and **removed entirely in `-Ofast`/`-Osmall`**.

```zig
const std = @import("std");

fn mid(buf: []const u8) u8 {
    std.debug.assert(buf.len > 0);
    return buf[buf.len / 2];
}

pub fn main() void {
    std.debug.print("mid = {c}\n", .{mid("abcde")});
    std.debug.print("mid = {c}\n", .{mid("")});
}
```

```sh
zig run as1.zig
```

**Deliberate panic:**

```
mid = c
thread 16621457 panic: reached unreachable code
/opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/debug.zig:442:14: 0x1022c1067 in assert (as1)
    if (!ok) unreachable; // assertion failure
             ^
/path/as1.zig:4:21: 0x1023cc4d7 in mid (as1)
    std.debug.assert(buf.len > 0);
                    ^
/path/as1.zig:10:41: 0x1023cb86f in main (as1)
    std.debug.print("mid = {c}\n", .{mid("")});
                                        ^
```

The message is always `reached unreachable code` — the condition is not
reported. The **second** frame is the one you want. If you need the value, use
`std.debug.panic` or return an error instead.

In `-Ofast` the same program (guard any run of this, it is undefined behaviour):

```
mid = c
exit=133
```

No message. `exit=133` is 128 + SIGTRAP. The assert was compiled out, `buf[0/2]`
read past a zero-length slice, and the program died on an illegal instruction.
**`assert` documents an invariant; it does not enforce one in production.** For
a condition that could be false at runtime, return an error.

## Error return traces

`@errorReturnTrace()` returns the trace Zig builds as an error propagates
through `try`. Print it with **`std.debug.dumpErrorReturnTrace`**.

```zig
const std = @import("std");

fn readPort(s: []const u8) !u16 {
    return std.fmt.parseInt(u16, s, 10);
}

fn loadConfig(s: []const u8) !u16 {
    return readPort(s);
}

pub fn main() void {
    const port = loadConfig("80x80") catch {
        std.debug.print("could not load config, trace follows:\n", .{});
        if (@errorReturnTrace()) |trace| std.debug.dumpErrorReturnTrace(trace);
        return;
    };
    std.debug.print("port = {d}\n", .{port});
}
```

```sh
zig run er1.zig
```

```
could not load config, trace follows:
/opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/fmt.zig:583:24: 0x10287a96f in charToDigit (er1)
    if (value >= base) return error.InvalidCharacter;
                       ^
...
/path/er1.zig:4:5: 0x10293389f in readPort (er1)
    return std.fmt.parseInt(u16, s, 10);
    ^
/path/er1.zig:8:5: 0x10293392b in loadConfig (er1)
    return readPort(s);
    ^
```

The trace starts at the **original `return error.X`**, four frames inside
`std.fmt`, and walks outward to your `catch`. That is the thing a bare
`error.InvalidCharacter` can never tell you.

> **Do not use `std.debug.dumpStackTrace` for this.** There are two
> incompatible `StackTrace` types in 0.17:
>
> ```
> er2.zig:9:67: error: expected type '*const debug.StackTrace', found '*lang.StackTrace'
>         if (@errorReturnTrace()) |trace| std.debug.dumpStackTrace(trace);
>                                                                   ^~~~~
> ```
> `@errorReturnTrace()` yields `*std.builtin.StackTrace` (a
> `.instruction_addresses` list); `dumpStackTrace` wants
> `*const std.debug.StackTrace` (`.return_addresses` + `.skipped`). Error return
> trace → `dumpErrorReturnTrace`. Full stop.

For a trace of *where you are now*, with no error involved, use
`dumpCurrentStackTrace`:

```zig
const std = @import("std");

fn inner() void {
    std.debug.print("where am I?\n", .{});
    std.debug.dumpCurrentStackTrace(.{});
}

fn outer() void {
    inner();
}

pub fn main() void {
    outer();
}
```

```sh
zig run cs.zig
```

```
where am I?
/path/cs.zig:5:36: 0x102e13c3b in inner (cs)
    std.debug.dumpCurrentStackTrace(.{});
                                   ^
/path/cs.zig:9:10: 0x102e13997 in outer (cs)
    inner();
         ^
/path/cs.zig:13:10: 0x102e1381f in main (cs)
    outer();
         ^
/opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/start.zig:788:64: 0x102e137bf in callMain (cs)
    if (fn_info.param_types.len == 0) return wrapMain(root.main());
                                                               ^
???:?:?: 0x180407dff in start (/usr/lib/dyld)
```

| Want | Call | Argument |
|---|---|---|
| the trace of an error you just caught | `std.debug.dumpErrorReturnTrace(t)` | `@errorReturnTrace()` unwrapped |
| the call stack right here | `std.debug.dumpCurrentStackTrace(.{})` | options struct |
| a trace you captured earlier | `std.debug.dumpStackTrace(st)` | `*const std.debug.StackTrace` |
| a trace inside a format string | `std.debug.FormatStackTrace{ .stack_trace = st }` with `{f}` | — |

## `lldb` on a Zig binary

Zig emits standard DWARF, so `lldb` works with no setup. Build with
`zig build-exe` (keeps the binary; `zig run` discards it):

```zig
const std = @import("std");

fn runningTotal(items: []const u32) u32 {
    var total: u32 = 0;
    for (items) |n| {
        total += n;
    }
    return total;
}

pub fn main() void {
    const data = [_]u32{ 10, 20, 30 };
    const t = runningTotal(&data);
    std.debug.print("total = {d}\n", .{t});
}
```

```sh
zig build-exe sum.zig
lldb ./sum
```

```
(lldb) breakpoint set --file sum.zig --line 6
Breakpoint 1: where = sum`sum.runningTotal + 116 at sum.zig:6:9, address = 0x0000000100132418
(lldb) run
Process 76682 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x0000000100132418 sum`sum.runningTotal(items=[]const u32 @ 0x000000016fdfe2e8) at sum.zig:6:9
   3   	fn runningTotal(items: []const u32) u32 {
   4   	    var total: u32 = 0;
   5   	    for (items) |n| {
-> 6   	        total += n;
    	        ^
   7   	    }
   8   	    return total;
   9   	}
(lldb) frame variable
([]const u32) items = {
  ptr = 0x000000010014a1cc
  len = 3
}
(unsigned int) total = 0
(unsigned int) n = 10
```

Zig slices show as `{ ptr, len }`, the loop capture `n` is a real local, and
function names are mangled as `<module>.<fn>` — `sum.runningTotal`.

| Command | Short | Does |
|---|---|---|
| `breakpoint set --file f.zig --line N` | `b f.zig:N` | break at a line |
| `breakpoint set --name pkg.fnName` | `b pkg.fnName` | break at a function |
| `run` | `r` | start |
| `continue` | `c` | resume |
| `next` / `step` / `finish` | `n` / `s` | over / into / out |
| `frame variable` | `fr v` | all locals in this frame |
| `print expr` | `p expr` | evaluate |
| `bt` | | full backtrace |
| `frame select N` | `f N` | switch frames |
| `quit` | `q` | exit |

A panicking program stops in `__pthread_kill` on SIGABRT, which is useless on
its own — but Zig's own trace has already printed above it, and `bt` plus
`frame select` gets you to your code.

**Debugging a test binary** needs the binary without running it:

```sh
zig test --test-no-exec -femit-bin=tt-test tt.zig
lldb ./tt-test
```

```
(lldb) breakpoint set --file tt.zig --line 4
Breakpoint 1: where = tt-test`tt.halve + 16 at tt.zig:4:14, address = 0x0000000100157510
(lldb) run
    frame #0: 0x0000000100157510 tt-test`tt.halve(n=11) at tt.zig:4:14
(lldb) bt
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
  * frame #0: 0x0000000100157510 tt-test`tt.halve(n=11) at tt.zig:4:14
    frame #1: 0x000000010015746c tt-test`tt.test.halve at tt.zig:8:41
    frame #2: 0x00000001001331ac tt-test`test_runner.mainTerminal(init=<unavailable>) at test_runner.zig:295:25
    frame #3: 0x0000000100132c8c tt-test`test_runner.main(init=process.Init.Minimal @ 0x000000016fdfe2f0) at test_runner.zig:74:28
```

Your test shows up as a frame named `tt.test.halve`. Combine with
`--test-filter` to stop at the one test you care about.

## What to reach for

| Symptom | Tool |
|---|---|
| "what is this value here" | `std.debug.print` |
| ongoing diagnostics you'll ship | `std.log` + scopes |
| machine-readable logs | a custom `logFn` |
| "how did this error get here" | `dumpErrorReturnTrace` |
| "who called me" | `dumpCurrentStackTrace` |
| it crashed | read the panic trace, innermost frame first |
| an invariant you believe is impossible | `std.debug.assert` (Debug/ReleaseSafe only) |
| a condition that can genuinely be false | return an error, not `assert` |
| you need to step through it | `zig build-exe` + `lldb` |
| it only fails in release | lesson 9.5 |

---

Next: [9.5 — Release Modes](./05-release-modes.md)
