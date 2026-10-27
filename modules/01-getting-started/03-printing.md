# 1.3 — Printing Properly

`std.debug.print` goes to stderr and is unbuffered. Real program output goes to
**stdout**, and in 0.17 that means getting a writer — which needs `io` and a
buffer you own.

> **Where tutorials will mislead you.** Online you will see
> `std.io.getStdOut().writer()`, `std.fs.File.stdout()`, or `stdout.writer()`
> with no arguments. All three are gone. Here is what 0.17.0 actually says:
>
> ```
> old1.zig:4:27: error: root source file struct 'fs' has no member named 'File'
>     const stdout = std.fs.File.stdout();
>                           ^~~~
> ```
> ```
> old2.zig:4:24: error: root source file struct 'std' has no member named 'io'
>     const stdout = std.io.getStdOut().writer();
>                        ^~
> ```
> Lowercase `std.io` does not exist. It is `std.Io`, capital `I`.

## The incantation

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    var buf: [1024]u8 = undefined;
    var w = std.Io.File.stdout().writer(init.io, &buf);
    const out = &w.interface;

    try out.print("you WILL see this\n", .{});
    try out.flush();
}
```

```
you WILL see this
```

Line by line:

| Line | Why |
|---|---|
| `main(init: std.process.Init)` | the runtime hands you `io` — see lesson 1.4 |
| `var buf: [1024]u8 = undefined` | **you** own the buffer; the stdlib allocates nothing |
| `.writer(init.io, &buf)` | a writer needs an I/O implementation *and* storage |
| `&w.interface` | `w` is a `File.Writer`; `.interface` is the generic `*std.Io.Writer` inside it |
| `try out.flush()` | without this, nothing happens |

`w` must be `var`, not `const` — the writer mutates its own buffer state.

## The flush trap

This is the single most common 0.17 mistake. Here is the broken program:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    var buf: [1024]u8 = undefined;
    var w = std.Io.File.stdout().writer(init.io, &buf);
    const out = &w.interface;

    try out.print("you will NOT see this\n", .{});
}
```

```sh
zig run noflush.zig
```

```
```

Nothing. Not a warning, not an error — **exit code `0` and zero bytes written**:

```sh
zig run noflush.zig 2>/dev/null | wc -c
```

```
       0
```

The fix is the one missing line:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    var buf: [1024]u8 = undefined;
    var w = std.Io.File.stdout().writer(init.io, &buf);
    const out = &w.interface;

    try out.print("you WILL see this\n", .{});
    try out.flush();
}
```

```
you WILL see this
```

### Why buffering is the default now

`print` copies formatted bytes into *your* `buf`. Nothing reaches the operating
system until the buffer fills or you flush it. A write syscall costs roughly a
microsecond; a `memcpy` into a stack array costs nanoseconds. Printing a million
lines unbuffered is a million syscalls.

Older Zig hid this by giving you an unbuffered writer, so every `print` was a
syscall and the slow path was the default. 0.17 inverted that: the fast path is
the default, and the cost is that you must say when to drain. `std.debug.print`
is the exception — it stays unbuffered on purpose, because a program that
crashes mid-execution must not lose its diagnostics.

### Flush with `defer`

Writing `flush()` at every exit point is how you forget it. Put it next to the
writer instead:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    var buf: [1024]u8 = undefined;
    var w = std.Io.File.stdout().writer(init.io, &buf);
    const out = &w.interface;
    defer out.flush() catch {};

    try out.print("line 1\n", .{});
    try out.print("line 2\n", .{});
}
```

```
line 1
line 2
```

`defer` runs on every exit from the scope, including an early error return.
`catch {}` is needed because `defer` cannot propagate an error — you are
deciding that a failed flush during teardown is not worth crashing over. Use
this pattern everywhere.

### The buffer size is not a limit

A small buffer just means more flushes, never a truncated or failed write:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    var buf: [8]u8 = undefined;
    var w = std.Io.File.stdout().writer(init.io, &buf);
    const out = &w.interface;

    try out.print("a very long line that does not fit in eight bytes\n", .{});
    try out.flush();
}
```

```
a very long line that does not fit in eight bytes
```

Eight bytes of buffer, forty-nine bytes of output. The writer drains whenever
it runs out of room. 1–4 KiB is a sane default.

## stdout vs stderr

`stderr()` mirrors `stdout()` exactly: same arguments, same `.interface`, and
its **own separate** `flush()`.

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    var obuf: [256]u8 = undefined;
    var ebuf: [256]u8 = undefined;
    var ow = std.Io.File.stdout().writer(init.io, &obuf);
    var ew = std.Io.File.stderr().writer(init.io, &ebuf);
    const out = &ow.interface;
    const err = &ew.interface;

    try out.print("this is stdout\n", .{});
    try err.print("this is stderr\n", .{});
    try out.flush();
    try err.flush();
}
```

```sh
zig run streams.zig
```

```
this is stdout
this is stderr
```

```sh
zig run streams.zig 2>/dev/null
```

```
this is stdout
```

```sh
zig run streams.zig 1>/dev/null
```

```
this is stderr
```

| Stream | For | API |
|---|---|---|
| stdout | the program's actual output (pipeable data) | `std.Io.File.stdout()` |
| stderr | logs, errors, progress, debugging | `std.Io.File.stderr()` or `std.debug.print` |

### One writer per stream

A subtle trap. `writer()` does **positional** writes — each writer tracks its own
offset starting at zero. Two writers on the same stream will overwrite each
other when output is redirected to a file:

```zig
const std = @import("std");

fn banner(io: std.Io, text: []const u8) !void {
    var buf: [256]u8 = undefined;
    var w = std.Io.File.stdout().writer(io, &buf);
    const out = &w.interface;
    try out.print("== {s} ==\n", .{text});
    try out.flush();
}

pub fn main(init: std.process.Init) !void {
    try banner(init.io, "first");
    try banner(init.io, "second");
}
```

```sh
zig run twowriters.zig > out.txt; cat out.txt
```

```
== second ==
```

`== first ==` was overwritten. Create one writer and pass the `*std.Io.Writer`
around instead:

```zig
const std = @import("std");

fn banner(out: *std.Io.Writer, text: []const u8) !void {
    try out.print("== {s} ==\n", .{text});
}

pub fn main(init: std.process.Init) !void {
    var buf: [256]u8 = undefined;
    var w = std.Io.File.stdout().writer(init.io, &buf);
    const out = &w.interface;
    defer out.flush() catch {};

    try banner(out, "first");
    try banner(out, "second");
}
```

```
== first ==
== second ==
```

Helper functions should take `out: *std.Io.Writer`, never recreate a writer. (If
you genuinely need an appending writer, `writerStreaming(io, &buf)` uses
streaming writes instead of positional ones and is safe to create more than
once.)

## Format specifiers

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    var buf: [1024]u8 = undefined;
    var w = std.Io.File.stdout().writer(init.io, &buf);
    const out = &w.interface;
    defer out.flush() catch {};

    const n: u8 = 65;
    const bytes = [_]u8{ 'z', 'i', 'g' };

    try out.print("{d}\n", .{n});
    try out.print("{c}\n", .{n});
    try out.print("{x}\n", .{n});
    try out.print("{b}\n", .{n});
    try out.print("{o}\n", .{n});
    try out.print("{s}\n", .{bytes});
    try out.print("{any}\n", .{bytes});
    try out.print("{s}\n", .{&bytes});
    try out.print("{any}\n", .{@as([]const u8, &bytes)});
}
```

```
65
A
41
1000001
101
zig
{ 122, 105, 103 }
zig
{ 122, 105, 103 }
```

Same `u8` five ways; same three bytes as text or as numbers.

```zig
const std = @import("std");

const Color = enum { red, green, blue };

const Point = struct {
    x: i32,
    y: i32,
    pub fn format(p: Point, w: *std.Io.Writer) !void {
        try w.print("({d}, {d})", .{ p.x, p.y });
    }
};

pub fn main(init: std.process.Init) !void {
    var buf: [1024]u8 = undefined;
    var w = std.Io.File.stdout().writer(init.io, &buf);
    const out = &w.interface;
    defer out.flush() catch {};

    const pi: f64 = 3.14159265;
    const some: ?i32 = 7;
    const none: ?i32 = null;
    const good: anyerror!i32 = 1;
    const bad: anyerror!i32 = error.Boom;

    try out.print("[{d:.3}]\n", .{pi});
    try out.print("[{d:>8}]\n", .{42});
    try out.print("[{d:<8}]\n", .{42});
    try out.print("[{d:0>8}]\n", .{42});
    try out.print("[{s:^9}]\n", .{"zig"});
    try out.print("{?d} {?d}\n", .{ some, none });
    try out.print("{!d} {!d}\n", .{ good, bad });
    try out.print("{t} {t}\n", .{ Color.green, Color.blue });
    try out.print("{f}\n", .{Point{ .x = 3, .y = -4 }});
    try out.print("{any}\n", .{Point{ .x = 3, .y = -4 }});
}
```

```
[3.142]
[      42]
[42      ]
[00000042]
[   zig   ]
7 null
1 error.Boom
green blue
(3, -4)
.{ .x = 3, .y = -4 }
```

| Specifier | Prints | Example output |
|---|---|---|
| `{d}` | decimal | `65` |
| `{c}` | the byte as a character | `A` |
| `{x}` `{b}` `{o}` | hex, binary, octal | `41` `1000001` `101` |
| `{s}` | bytes as a UTF-8 string | `zig` |
| `{any}` | Zig literal syntax, recursively | `{ 122, 105, 103 }` |
| `{d:.3}` | 3 decimal places | `3.142` |
| `{d:>8}` `{d:<8}` | right / left pad to width 8 | `      42` / `42      ` |
| `{d:0>8}` | pad with a chosen fill char | `00000042` |
| `{s:^9}` | centered | `   zig   ` |
| `{?d}` | unwraps an optional, else `null` | `7` / `null` |
| `{!d}` | unwraps an error union, else the error | `1` / `error.Boom` |
| `{t}` | an enum's or error's tag name | `green` |
| `{f}` | calls the type's own `pub fn format` | `(3, -4)` |

Two worth highlighting:

- **`{f}` is how you get custom formatting.** The signature changed in 0.17 to
  `pub fn format(self: T, w: *std.Io.Writer) !void` — two parameters, no `fmt`
  string and no options. Older tutorials show four.
- **`{any}` on a struct prints Zig literal syntax**, leading dot included:
  `.{ .x = 3, .y = -4 }`. It is the debug-dump specifier and it needs no
  `format` method.

A mismatched specifier is a **compile** error, not a runtime surprise:

```
error: invalid format string 'd' for type '[3]u8'
```

## Without a writer: `bufPrint`

Formatting into a stack buffer needs no `io` at all:

```zig
const std = @import("std");

pub fn main() !void {
    var buf: [64]u8 = undefined;
    const s = try std.fmt.bufPrint(&buf, "{s} v{d}.{d}", .{ "zig", 0, 17 });
    std.debug.print("{s} ({d} bytes)\n", .{ s, s.len });
}
```

```
zig v0.17 (9 bytes)
```

`bufPrint` returns a slice of `buf` holding exactly the bytes written, and fails
with `error.NoSpaceLeft` if the buffer is too small. No allocator, no flush.

---

Next: [1.4 — The `io` Parameter](./04-the-io-parameter.md)
