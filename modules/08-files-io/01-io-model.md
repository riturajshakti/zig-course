# 8.1 — The `std.Io` Model

Lesson 1.4 introduced `io` as the thing writers need. Now the whole model: what
that value is, what it abstracts, and which implementation you get.

## What `io` actually is

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    std.debug.print("io          = {s}\n", .{@typeName(@TypeOf(io))});
    std.debug.print("io.userdata = {s}\n", .{@typeName(@TypeOf(io.userdata))});
    std.debug.print("io.vtable   = {s}\n", .{@typeName(@TypeOf(io.vtable))});
    std.debug.print("@sizeOf(Io) = {d}\n", .{@sizeOf(std.Io)});
}
```

```
io          = Io
io.userdata = ?*anyopaque
io.vtable   = *const Io.VTable
@sizeOf(Io) = 16
```

Sixteen bytes: a type-erased pointer and a function table. Structurally
identical to `std.mem.Allocator`. Copy it, store it, pass it by value — it is
not a handle you have to close.

## What it abstracts

Every operation that leaves your process goes through that one vtable.

| Area | Reached through |
|---|---|
| filesystem | `std.Io.Dir`, `std.Io.File` |
| terminal / stdio | `std.Io.File.stdout()`, `.stderr()`, `.stdin()` |
| network | `std.Io.net` — sockets, listen, connect |
| time & clocks | `std.Io.Timestamp`, `std.Io.sleep`, `std.Io.Duration` |
| concurrency | `io.async`, `io.concurrent`, `std.Io.Group` |
| sync primitives | `std.Io.Mutex`, `Condition`, `Semaphore`, `RwLock` |

That is the point of the redesign: one interface for everything that blocks, so
one swap changes the execution model for all of it.

## Why it is a parameter

Three concrete payoffs. Each is a real property of the code below, not a slogan.

### 1. The signature is honest

```zig
const std = @import("std");

fn pure(a: i32, b: i32) i32 {
    return a + b;
}

fn needsWriter(out: *std.Io.Writer, label: []const u8) !void {
    try out.print("[{s}]\n", .{label});
}

fn needsIo(io: std.Io, path: []const u8) !void {
    try std.Io.Dir.cwd().writeFile(io, .{ .sub_path = path, .data = "touched\n" });
}

pub fn main(init: std.process.Init) !void {
    var buf: [256]u8 = undefined;
    var w = std.Io.File.stdout().writerStreaming(init.io, &buf);
    const out = &w.interface;
    defer out.flush() catch {};

    try out.print("pure: {d}\n", .{pure(2, 3)});
    try needsWriter(out, "writer only");
    try needsIo(init.io, "/tmp/zig-course-model.txt");
    try out.writeAll("wrote /tmp/zig-course-model.txt\n");
}
```

```
pure: 5
[writer only]
wrote /tmp/zig-course-model.txt
```

| Signature | Can it touch the outside world? |
|---|---|
| `fn pure(a, b) i32` | no — enforced by the compiler |
| `fn f(out: *std.Io.Writer, ...)` | writes text somewhere the caller chose |
| `fn f(io: std.Io, ...)` | files, sockets, clocks, spawning work |

Pass the narrowest of the three that does the job.

### 2. No function colouring

In most languages `async` infects every caller: an async function can only be
called from an async function, and a blocking library cannot be composed with an
async one. Zig has no `async` keyword in the signature. The concurrency lives in
the `io` **value**, so one function body serves both models:

```zig
const std = @import("std");

fn work(io: std.Io, name: []const u8) void {
    std.Io.sleep(io, .fromMilliseconds(100), .awake) catch {};
    std.debug.print("{s} done\n", .{name});
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const t0 = std.Io.Timestamp.now(io, .awake);

    var a = io.async(work, .{ io, "a" });
    var b = io.async(work, .{ io, "b" });
    a.await(io);
    b.await(io);

    const t1 = std.Io.Timestamp.now(io, .awake);
    std.debug.print("elapsed {d} ms\n", .{t0.durationTo(t1).toMilliseconds()});
}
```

```
a done
b done
elapsed 105 ms
```

Two 100 ms sleeps in 105 ms. `work` is an ordinary function with an ordinary
signature — no `async`, no `await` in its body, no `Future` return type. Call it
directly and it blocks; hand it to `io.async` and it doesn't. Nothing about
`work` changed.

> `std.Io.sleep` takes `(io, duration, clock)` — duration **before** clock. And
> `Timestamp.now` returns a plain `Timestamp`, not an error union, so `try` on
> it is a compile error: `expected error union type, found 'Io.Timestamp'`.

### 3. Testability

```zig
const std = @import("std");

fn wordCount(io: std.Io, gpa: std.mem.Allocator, path: []const u8) !usize {
    const text = try std.Io.Dir.cwd().readFileAlloc(io, path, gpa, .limited(1 << 16));
    defer gpa.free(text);
    var it = std.mem.tokenizeAny(u8, text, " \n\t");
    var n: usize = 0;
    while (it.next()) |_| n += 1;
    return n;
}

pub fn main(init: std.process.Init) !void {
    try std.Io.Dir.cwd().writeFile(init.io, .{
        .sub_path = "/tmp/zig-course-words.txt",
        .data = "the quick brown fox\njumps over\n",
    });
    std.debug.print("{d} words\n", .{try wordCount(init.io, init.gpa, "/tmp/zig-course-words.txt")});
}

test "wordCount against a temp file" {
    const io = std.testing.io;
    try std.Io.Dir.cwd().writeFile(io, .{
        .sub_path = "/tmp/zig-course-words-test.txt",
        .data = "a b c\n",
    });
    try std.testing.expectEqual(
        @as(usize, 3),
        try wordCount(io, std.testing.allocator, "/tmp/zig-course-words-test.txt"),
    );
}
```

```sh
zig run words.zig
```

```
6 words
```

```sh
zig test words.zig
```

```
1/1 words.test.wordCount against a temp file...OK
All 1 tests passed.
```

`std.testing.io` is the `io` for tests. The function under test never knew the
difference — and when a function only emits text, you can drop `io` entirely
and use `std.Io.Writer.fixed` (lesson 8.2).

## Getting `io`

### `std.process.Init` — the default

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    std.debug.print("io          {s}\n", .{@typeName(@TypeOf(init.io))});
    std.debug.print("gpa         {s}\n", .{@typeName(@TypeOf(init.gpa))});
    std.debug.print("arena       {s}\n", .{@typeName(@TypeOf(init.arena))});
    std.debug.print("environ_map {s}\n", .{@typeName(@TypeOf(init.environ_map))});
    std.debug.print("args        {s}\n", .{@typeName(@TypeOf(init.minimal.args))});
}
```

```
io          Io
gpa         mem.Allocator
arena       *heap.ArenaAllocator
environ_map *process.Environ.Map
args        process.Args
```

| Field | Notes |
|---|---|
| `init.io` | the implementation chosen for your target |
| `init.gpa` | leak-checked in Debug |
| `init.arena` | **a pointer to an ArenaAllocator** — call `init.arena.allocator()` |
| `init.environ_map` | lesson 8.6 |
| `init.minimal.args` | lesson 8.6 |

### `std.Io.Threaded` — by hand

```zig
const std = @import("std");

fn greet(io: std.Io, label: []const u8) !void {
    var buf: [256]u8 = undefined;
    var w = std.Io.File.stdout().writerStreaming(io, &buf);
    const out = &w.interface;
    defer out.flush() catch {};
    try std.Io.sleep(io, .fromMilliseconds(20), .awake);
    try out.print("{s}\n", .{label});
}

pub fn main() !void {
    var threaded: std.Io.Threaded = .init(std.heap.page_allocator, .{});
    defer threaded.deinit();
    try greet(threaded.io(), "same function, io built by hand");
}
```

```
same function, io built by hand
```

`.init` takes `(allocator, options)`; `.io()` produces the interface value;
`deinit()` must run. This is what `std.process.Init` does for you. Build it
yourself for a library entry point, a test harness, or to tune the pool:

| `Threaded.InitOptions` | Default |
|---|---|
| `stack_size` | `std.Thread.SpawnConfig.default_stack_size` |
| `async_limit` | one less than the logical core count |
| `concurrent_limit` | `.unlimited` |

### The implementations

| Implementation | Mechanism | Status on 0.17.0 |
|---|---|---|
| `std.Io.Threaded` | thread pool, blocking syscalls | **works everywhere** — the default |
| `std.Io.Evented` | stackful fibers over an event loop | **does not compile on 0.17.0** |
| `std.Io.Uring` | `io_uring` | Linux only |
| `std.Io.Kqueue` | `kqueue` | BSD/macOS |

`Evented` is where the design is heading — green threads, one OS thread,
`io.async` without a thread per task. On 0.17.0 it is bit-rotted and fails to
build:

```
.../std/Io/Dispatch.zig:439:14: error: no field named 'processReplacePath' in struct 'Io.VTable'
            .processReplacePath = processReplacePath,
             ^~~~~~~~~~~~~~~~~~
```

Its vtable literal references a field the current `Io.VTable` no longer has.
Use `Threaded` today. **The code you write does not change when `Evented`
lands** — that is the entire argument for explicit `io`.

## Passing `io` down

One rule: take it as the first parameter and hand it along. Do not stash it in a
global, and do not rebuild one per function.

```zig
const std = @import("std");

const Store = struct {
    io: std.Io,
    gpa: std.mem.Allocator,
    root: []const u8,

    fn put(s: Store, key: []const u8, value: []const u8) !void {
        var buf: [256]u8 = undefined;
        const path = try std.fmt.bufPrint(&buf, "{s}/{s}", .{ s.root, key });
        try std.Io.Dir.cwd().writeFile(s.io, .{ .sub_path = path, .data = value });
    }

    fn get(s: Store, key: []const u8) ![]u8 {
        var buf: [256]u8 = undefined;
        const path = try std.fmt.bufPrint(&buf, "{s}/{s}", .{ s.root, key });
        return std.Io.Dir.cwd().readFileAlloc(s.io, path, s.gpa, .limited(1 << 20));
    }
};

pub fn main(init: std.process.Init) !void {
    const root = "/tmp/zig-course-store";
    try std.Io.Dir.cwd().createDirPath(init.io, root);

    const store: Store = .{ .io = init.io, .gpa = init.gpa, .root = root };
    try store.put("greeting", "hello from the store\n");

    const got = try store.get("greeting");
    defer init.gpa.free(got);
    std.debug.print("{s}", .{got});
}
```

```
hello from the store
```

Storing `io` in a struct next to the allocator is idiomatic — it is the same
16-byte value, and a type that does I/O should say so in its fields.

---

Next: [8.2 — Readers & Writers](./02-readers-writers.md)
