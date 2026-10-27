# 1.4 — The `io` Parameter

Every writer in lesson 1.3 needed an `io`. This lesson is about what that value
*is*, where to get it, and why 0.17 forces you to pass it around.

## Why 0.17 made `io` explicit

This is a genuinely new idea, so it is worth the paragraphs.

### The problem with ambient I/O

In Node, Go, Python and older Zig, a function that does I/O just does it:

```js
function save(data) { fs.writeFileSync("out.txt", data); }   // Node
```

Nothing in that signature says it touches the disk. Two consequences follow,
and both are what 0.17 is attacking:

**1. The signature lies about what the function does.** `save(data)` looks
pure. You cannot tell from the type whether it writes a file, opens a socket, or
blocks for thirty seconds. The only way to know is to read the body, and the
bodies of everything it calls.

**2. The concurrency model is baked into the function, not chosen by the
caller.** `fs.writeFileSync` is *the* blocking implementation, decided by
whoever wrote the function. If you later want that same code to run on a thread
pool, or on `io_uring`, or on green threads, you cannot — you must rewrite it.
This is the "function colouring" problem: in most languages `async` infects
every caller, and a library written for blocking I/O and a library written for
async I/O cannot be composed.

### The 0.17 answer

`io` is an **interface value** — a vtable, like an allocator. A function that
needs I/O takes it as a parameter:

```
fn save(io: std.Io, data: []const u8) !void { ... }
```

Now:

- **The signature is honest.** `io` in the parameter list means "this performs
  I/O". No `io` means it cannot, enforced by the compiler.
- **The caller picks the implementation.** The same `save` runs blocking, on a
  thread pool, on `io_uring`, or against an in-memory fake, depending on which
  `io` you hand it. The function never changes.
- **There is no `async` keyword to colour anything.** Async-ness lives in the
  `io` value, not in the function's type, so blocking and async code compose.

This mirrors a pattern you already know from Zig: allocators are explicit
parameters for exactly these reasons. `io` is that idea applied to I/O. The
cost is one extra parameter threaded through your call graph; the payoff is that
I/O becomes swappable and testable.

### The payoff, concretely

```zig
const std = @import("std");

fn report(out: *std.Io.Writer, name: []const u8, n: u32) !void {
    try out.print("{s}: {d}\n", .{ name, n });
}

pub fn main(init: std.process.Init) !void {
    var buf: [256]u8 = undefined;
    var w = std.Io.File.stdout().writer(init.io, &buf);
    defer w.interface.flush() catch {};
    try report(&w.interface, "live", 1);
}

test "report writes to memory, no io needed" {
    var mem: [64]u8 = undefined;
    var fixed = std.Io.Writer.fixed(&mem);
    try report(&fixed, "test", 7);
    try std.testing.expectEqualStrings("test: 7\n", fixed.buffered());
}
```

```sh
zig run report.zig
```

```
live: 1
```

```sh
zig test report.zig
```

```
1/1 report.test.report writes to memory, no io needed...OK
All 1 tests passed.
```

One `report` function. In `main` it writes to the terminal; in the test it
writes into a 64-byte stack array with **no files, no `io`, and no fixtures** —
`std.Io.Writer.fixed` is a writer backed by memory, and `buffered()` returns
what was written. That is what explicit I/O buys you.

## Getting `io`: `std.process.Init` (preferred)

Change `main`'s signature and the runtime hands you everything:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const gpa = init.gpa;
    const arena = init.arena.allocator();

    var buf: [512]u8 = undefined;
    var w = std.Io.File.stdout().writer(io, &buf);
    const out = &w.interface;
    defer out.flush() catch {};

    const a = try std.fmt.allocPrint(gpa, "gpa string {d}", .{1});
    defer gpa.free(a);

    const b = try std.fmt.allocPrint(arena, "arena string {d}", .{2});

    try out.print("{s}\n", .{a});
    try out.print("{s}\n", .{b});

    var it = init.minimal.args.iterate();
    var n: usize = 0;
    while (it.next()) |arg| : (n += 1) {
        try out.print("arg[{d}] = {s}\n", .{ n, arg });
    }
}
```

```sh
zig run init.zig -- alpha beta
```

```
gpa string 1
arena string 2
arg[0] = /Users/raj/.cache/zig/o/f6fc4790fbdb547c236c823ea45233de/init
arg[1] = alpha
arg[2] = beta
```

Note the `--`: `zig run` needs it to stop parsing arguments as its own. `arg[0]`
is the binary, which under `zig run` lives in the build cache.

Also note the `b` from the arena is **never freed**, and that is correct — the
arena is released wholesale at process exit.

### What's in `Init`

| Field | Type | Use |
|---|---|---|
| `init.io` | `std.Io` | every writer, reader, file, socket |
| `init.gpa` | `std.mem.Allocator` | general allocation; leak-checked in Debug |
| `init.arena` | `*std.heap.ArenaAllocator` | **not an Allocator** — see below |
| `init.minimal.args` | `Args` | command-line arguments, via `.iterate()` |
| `init.environ_map` | `*Environ.Map` | environment variables |

Confirm it yourself:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    std.debug.print("init.io is a {s}\n", .{@typeName(@TypeOf(init.io))});
    std.debug.print("init.gpa is a {s}\n", .{@typeName(@TypeOf(init.gpa))});
    std.debug.print("init.arena is a {s}\n", .{@typeName(@TypeOf(init.arena))});
}
```

```
init.io is a Io
init.gpa is a mem.Allocator
init.arena is a *heap.ArenaAllocator
```

### The `init.arena` trap

`init.arena` is a *pointer to an ArenaAllocator*, not an `Allocator`. Pass it
directly and you get:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const s = try std.fmt.allocPrint(init.arena, "oops", .{});
    _ = s;
}
```

```
arena.zig:4:42: error: expected type 'mem.Allocator', found '*heap.ArenaAllocator'
    const s = try std.fmt.allocPrint(init.arena, "oops", .{});
                                     ~~~~^~~~~~
```

Fix: `init.arena.allocator()`.

| Allocator | Lifetime | When |
|---|---|---|
| `init.gpa` | you `free` each allocation | long-lived data, anything you must release |
| `init.arena.allocator()` | freed all at once at exit | scratch data you never need to free individually |

## Getting `io`: `std.Io.Threaded` (manual)

Sometimes you need to build the `io` yourself — a library entry point, a test
harness, or when you want to configure the event loop:

```zig
const std = @import("std");

pub fn main() !void {
    var threaded: std.Io.Threaded = .init(std.heap.page_allocator, .{});
    defer threaded.deinit();
    const io = threaded.io();

    var buf: [256]u8 = undefined;
    var w = std.Io.File.stdout().writer(io, &buf);
    const out = &w.interface;
    defer out.flush() catch {};

    try out.print("io built by hand\n", .{});
}
```

```
io built by hand
```

`std.Io.Threaded` is one concrete implementation of the `std.Io` interface — it
backs blocking operations with a thread pool. `.init` takes two arguments, an
allocator and an options struct, and `.io()` produces the interface value.
`deinit()` must run, hence the `defer`.

This is the mechanism `std.process.Init` sets up for you. Seeing it by hand is
the point: `io` is an ordinary value you can construct, not compiler magic.

## Which to use

| Situation | `main` signature |
|---|---|
| only `std.debug.print` | `pub fn main() void` or `!void` |
| **any real I/O, allocation, or args** | `pub fn main(init: std.process.Init) !void` |
| you must configure or own the event loop | `pub fn main() !void` + `std.Io.Threaded` |

Default to `std.process.Init`. Reach for `Threaded` only when you genuinely need
to own the loop.

## The rule for your own functions

```zig
const std = @import("std");

fn pure(a: i32, b: i32) i32 {
    return a + b;
}

fn needsWriter(out: *std.Io.Writer, label: []const u8) !void {
    try out.print("[{s}]\n", .{label});
}

fn needsIo(io: std.Io, path: []const u8) !void {
    var f = try std.Io.Dir.cwd().createFile(io, path, .{});
    defer f.close(io);
}

pub fn main(init: std.process.Init) !void {
    var buf: [256]u8 = undefined;
    var w = std.Io.File.stdout().writer(init.io, &buf);
    const out = &w.interface;
    defer out.flush() catch {};

    try out.print("pure: {d}\n", .{pure(2, 3)});
    try needsWriter(out, "writer only");
    try needsIo(init.io, "/tmp/zig-io-demo.txt");
    try out.print("made /tmp/zig-io-demo.txt\n", .{});
}
```

```
pure: 5
[writer only]
made /tmp/zig-io-demo.txt
```

Three tiers, and the signature tells you which is which:

- No `io`, no writer → **cannot** do I/O. Trivially testable.
- Takes `*std.Io.Writer` → writes somewhere the caller chose. Prefer this for
  anything that only emits text; it is the most testable shape.
- Takes `std.Io` → needs to open files, sockets, or spawn work.

Pass the narrowest thing that does the job.

---

Next: [1.5 — Reading the Source](./05-reading-stdlib.md)
