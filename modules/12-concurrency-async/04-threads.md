# 12.4 — Threads

`std.Thread` is the raw OS thread: no pool, no scheduler, no `io`. One
`spawn` is one `pthread_create`.

| | `std.Thread.spawn` | `io.async` |
|---|---|---|
| maps to | one OS thread | a pool slot (or inline) |
| join | `t.join()` | `f.await(io)` |
| return value | none — write through a pointer | typed, from the `Future` |
| cancellable | no | yes |
| needs `io` | no | yes |
| cost | ~13 µs here, 16 MiB virtual stack | amortised across the pool |

## The basics

```zig
const std = @import("std");

fn hello(n: u32, ids: []std.Thread.Id) void {
    ids[n] = std.Thread.getCurrentId();
}

pub fn main() !void {
    std.debug.print("cpus = {d}\n", .{try std.Thread.getCpuCount()});
    std.debug.print("default_stack_size = {d} MiB\n", .{std.Thread.SpawnConfig.default_stack_size / (1 << 20)});

    var ids: [2]std.Thread.Id = .{ 0, 0 };
    const t1 = try std.Thread.spawn(.{}, hello, .{ 0, &ids });
    const t2 = try std.Thread.spawn(.{ .stack_size = 64 * 1024 }, hello, .{ 1, &ids });
    t1.join();
    t2.join();

    std.debug.print("main id != t1 id != t2 id: {}\n", .{ids[0] != ids[1] and ids[0] != std.Thread.getCurrentId()});
}
```

```
cpus = 12
default_stack_size = 16 MiB
main id != t1 id != t2 id: true
```

| API | Signature |
|---|---|
| `std.Thread.spawn` | `(SpawnConfig, comptime fn, args_tuple) SpawnError!Thread` |
| `t.join()` | blocks, releases the handle. **Exactly once** |
| `t.detach()` | releases the handle now; you can never join |
| `std.Thread.getCpuCount()` | `CpuCountError!usize` |
| `std.Thread.getCurrentId()` | `Id` |
| `std.Thread.yield()` | `YieldError!void` |
| `std.Thread.max_name_len` | 63 on macOS, 15 on Linux, 31 on Windows |

`SpawnConfig`:

| Field | Default | Note |
|---|---|---|
| `stack_size` | `16 * 1024 * 1024` | virtual reservation, not resident memory |
| `allocator` | `null` | only used on targets without native threads (WASI) |

16 MiB is large because it is *reserved* address space; only touched pages
become resident. Shrink it when you spawn hundreds of threads and know the call
depth is shallow.

> `std.Thread.Mutex` **no longer exists** in 0.17 — the sync primitives moved to
> `std.Io`. See 12.5.

## Thread names

```zig
const std = @import("std");

const Slot = struct {
    ready: std.Io.Semaphore = .{},
    handle: std.Thread = undefined,
};

fn body(io: std.Io, slot: *Slot, name: []const u8) void {
    slot.ready.wait(io) catch return;
    slot.handle.setName(io, name) catch |e| {
        std.debug.print("setName: {t}\n", .{e});
        return;
    };
    var buf: [std.Thread.max_name_len:0]u8 = undefined;
    std.debug.print("self-named: {?s}\n", .{slot.handle.getName(&buf) catch null});
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    std.debug.print("max_name_len here = {d}\n", .{std.Thread.max_name_len});

    var slot: Slot = .{};
    const t = try std.Thread.spawn(.{}, body, .{ io, &slot, "worker-1" });
    slot.handle = t;
    std.debug.print("naming from another thread: {!}\n", .{t.setName(io, "nope")});
    slot.ready.post(io);
    t.join();
}
```

```
max_name_len here = 63
naming from another thread: error.Unsupported
self-named: worker-1
```

Two things worth knowing:

- **`setName` takes `io`** — `t.setName(io, name)`, not `t.setName(name)`.
- **On macOS a thread can only name itself.** `pthread_setname_np` there takes
  no thread argument, so naming from the spawner returns `error.Unsupported`.
  On Linux either direction works. The semaphore above is just there to hand
  the thread its own handle before it runs.

## Passing pointers safely

Thread arguments are copied into the new thread's context, so pass **pointers
to storage that outlives the thread**. The classic bug is a pointer into the
spawning function's frame when that function returns before `join`.

Here each thread owns a disjoint slice and a disjoint output field — so no lock
is needed at all:

```zig
const std = @import("std");

const Chunk = struct {
    data: []const u32,
    sum: u64 = 0,
};

fn sumChunk(c: *Chunk) void {
    var s: u64 = 0;
    for (c.data) |v| s += v;
    c.sum = s;
}

pub fn main(init: std.process.Init) !void {
    const gpa = init.gpa;
    const n = 1_000_000;
    const data = try gpa.alloc(u32, n);
    defer gpa.free(data);
    for (data, 0..) |*v, i| v.* = @intCast(i % 1000);

    const parts = 4;
    var chunks: [parts]Chunk = undefined;
    var threads: [parts]std.Thread = undefined;
    const per = n / parts;

    for (&chunks, 0..) |*c, i| c.* = .{ .data = data[i * per ..][0..per] };
    for (&threads, &chunks) |*t, *c| t.* = try std.Thread.spawn(.{}, sumChunk, .{c});
    for (&threads) |t| t.join();

    var total: u64 = 0;
    for (chunks) |c| total += c.sum;

    var check: u64 = 0;
    for (data) |v| check += v;
    std.debug.print("parallel {d}, serial {d}, equal {}\n", .{ total, check, total == check });
}
```

```
parallel 499500000, serial 499500000, equal true
```

| Pattern | Safe? |
|---|---|
| `&local` in a fn that `join`s before returning | yes — `join` is the lifetime bound |
| `&local` in a fn that `detach`es | **no** — dangling as soon as it returns |
| heap pointer freed after `join` | yes |
| `*T` to disjoint elements of one array | yes — no overlap, no lock |
| two threads writing the same `*T` | no — needs a mutex (12.5) or an atomic (12.6) |

## `detach`

```zig
const std = @import("std");

var finished: std.atomic.Value(u32) = .init(0);

fn fire(io: std.Io) void {
    std.Io.sleep(io, .fromMilliseconds(20), .awake) catch {};
    _ = finished.fetchAdd(1, .monotonic);
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const t = try std.Thread.spawn(.{}, fire, .{io});
    t.detach();
    std.debug.print("detached; cannot join, must sync some other way\n", .{});

    while (finished.load(.acquire) == 0) try std.Io.sleep(io, .fromMilliseconds(5), .awake);
    std.debug.print("observed finished = {d}\n", .{finished.load(.acquire)});
}
```

```
detached; cannot join, must sync some other way
observed finished = 1
```

A detached thread still dies when the process exits, so you need your own
completion signal anyway. In practice: don't detach. `join` or use `io.async`.

## When raw threads beat `io.async`

Pool dispatch is not free. In a Debug build, `Threaded`'s bookkeeping costs
more than `pthread_create`; with optimisation it wins by an order of magnitude.

```zig
const std = @import("std");

var hits: std.atomic.Value(u32) = .init(0);

fn tick() void {
    _ = hits.fetchAdd(1, .monotonic);
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const n = 500;

    var t0 = std.Io.Timestamp.now(io, .awake);
    var threads: [n]std.Thread = undefined;
    for (&threads) |*t| t.* = try std.Thread.spawn(.{}, tick, .{});
    for (&threads) |t| t.join();
    var t1 = std.Io.Timestamp.now(io, .awake);
    const thread_us = t0.durationTo(t1).toMicroseconds();

    t0 = std.Io.Timestamp.now(io, .awake);
    var g: std.Io.Group = .init;
    for (0..n) |_| g.async(io, tick, .{});
    try g.await(io);
    t1 = std.Io.Timestamp.now(io, .awake);
    const group_us = t0.durationTo(t1).toMicroseconds();

    std.debug.print("{d} fresh threads : {d} us\n", .{ n, thread_us });
    std.debug.print("{d} Group.async   : {d} us\n", .{ n, group_us });
    std.debug.print("hits = {d}\n", .{hits.load(.monotonic)});
}
```

```sh
zig run spawnbench.zig
```

```
500 fresh threads : 6653 us
500 Group.async   : 27892 us
hits = 1000
```

```sh
zig run -Ofast spawnbench.zig
```

```
500 fresh threads : 6232 us
500 Group.async   : 1468 us
hits = 1000
```

Debug: 4x slower. `-Ofast`: 4x faster. Timings vary between runs; the
Debug/release flip does not. **Never benchmark concurrency in Debug.**

Legitimate reasons to reach for a raw thread:

| Reason | Why `io.async` doesn't fit |
|---|---|
| a thread that lives for the whole process (log flusher, signal handler, watchdog) | it would permanently occupy a pool slot |
| you need a specific `stack_size` for one task | `Threaded`'s `stack_size` is pool-wide |
| the thread must be named, pinned, or priority-tuned | no `Future` equivalent |
| blocking on a C library that doesn't know about `io` | a pool worker stuck in C blocks other tasks |
| no `io` available (a library's deepest internals) | `spawn` needs nothing |

Everything else: use `io.async`. You get cancellation, typed results, and code
that still works when the `io` implementation changes.

---

Next: [12.5 — Mutexes & Locks](./05-mutexes.md)
