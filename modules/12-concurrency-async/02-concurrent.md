# 12.2 — concurrent

`io.async` is a *request*: "run this, I'll collect it later." The
implementation may honour it by running the task on a worker thread — or by
running it **inline, right there, to completion**, before `async` even returns.

`io.concurrent` is a *demand*: "this must make progress independently of me."
It can fail.

| | `io.async` | `io.concurrent` |
|---|---|---|
| returns | `Future(T)` | `ConcurrentError!Future(T)` |
| may run inline | **yes** | no |
| fails when no unit of concurrency | never | `error.ConcurrencyUnavailable` |
| works on a single-threaded `io` | yes | no |
| use for | fan-out where any order is fine | tasks that must coexist (server + client, producer + consumer) |

## Proving the difference

`async_limit = .nothing` gives `Threaded` no worker threads at all.

```zig
const std = @import("std");

fn slow(io: std.Io) void {
    std.Io.sleep(io, .fromMilliseconds(100), .awake) catch {};
}

pub fn main(init: std.process.Init) !void {
    var threaded: std.Io.Threaded = .init(init.gpa, .{ .async_limit = .nothing });
    defer threaded.deinit();
    const io = threaded.io();

    var t0 = std.Io.Timestamp.now(io, .awake);
    var a = io.async(slow, .{io});
    var t1 = std.Io.Timestamp.now(io, .awake);
    std.debug.print("io.async returned after      {d} ms\n", .{t0.durationTo(t1).toMilliseconds()});
    a.await(io);

    t0 = std.Io.Timestamp.now(io, .awake);
    var c = try io.concurrent(slow, .{io});
    t1 = std.Io.Timestamp.now(io, .awake);
    std.debug.print("io.concurrent returned after {d} ms\n", .{t0.durationTo(t1).toMilliseconds()});
    c.await(io);
}
```

```
io.async returned after      102 ms
io.concurrent returned after 0 ms
```

With no pool, `io.async` ran the task inline and only returned when it was
done. `io.concurrent` refused to do that — it spawned a thread.

**The consequence:** anything where task A must be alive while you do B
(accepting a socket your own client is about to connect to, a consumer waiting
on a producer) needs `io.concurrent`. With `io.async` it can deadlock.

## When concurrency is unavailable

```zig
const std = @import("std");

fn ping(io: std.Io) u32 {
    std.Io.sleep(io, .fromMilliseconds(20), .awake) catch {};
    return 1;
}

fn try_both(gpa: std.mem.Allocator, label: []const u8, opts: std.Io.Threaded.InitOptions) !void {
    var threaded: std.Io.Threaded = .init(gpa, opts);
    defer threaded.deinit();
    const io = threaded.io();

    var a = io.async(ping, .{io});
    std.debug.print("{s}: io.async     -> {d}\n", .{ label, a.await(io) });

    if (io.concurrent(ping, .{io})) |fut| {
        var f = fut;
        std.debug.print("{s}: io.concurrent -> {d}\n", .{ label, f.await(io) });
    } else |err| {
        std.debug.print("{s}: io.concurrent -> {t}\n", .{ label, err });
    }
}

pub fn main(init: std.process.Init) !void {
    try try_both(init.gpa, "default   ", .{});
    try try_both(init.gpa, "no threads", .{ .async_limit = .nothing, .concurrent_limit = .nothing });
}
```

```
default   : io.async     -> 1
default   : io.concurrent -> 1
no threads: io.async     -> 1
no threads: io.concurrent -> ConcurrencyUnavailable
```

`io.async` always works. That is why libraries should prefer it: a library
built on `io.async` runs on a single-threaded `io` too.

## Concurrency vs parallelism

| | Concurrency | Parallelism |
|---|---|---|
| means | several tasks in flight | several tasks executing *at the same instant* |
| needs | one core | several cores |
| wins on | I/O-bound work (sleep, sockets, disk) | CPU-bound work |
| Node | the event loop gives you this | needs `worker_threads` |
| Go | goroutines | `GOMAXPROCS` > 1 |
| Zig `Threaded` | both, via the worker pool | both |

`Threaded` backs tasks with OS threads, so on this 12-core machine `io.async`
gives you real parallelism:

```zig
const std = @import("std");

fn spin(n: u64) u64 {
    var acc: u64 = 0;
    var i: u64 = 0;
    while (i < n) : (i += 1) acc = acc +% i *% 2654435761;
    return acc;
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const work = 40_000_000;
    const tasks = 8;

    var t0 = std.Io.Timestamp.now(io, .awake);
    var acc: u64 = 0;
    for (0..tasks) |_| acc +%= spin(work);
    var t1 = std.Io.Timestamp.now(io, .awake);
    const serial_ms = t0.durationTo(t1).toMilliseconds();

    t0 = std.Io.Timestamp.now(io, .awake);
    var futures: [tasks]std.Io.Future(u64) = undefined;
    for (&futures) |*f| f.* = io.async(spin, .{work});
    var acc2: u64 = 0;
    for (&futures) |*f| acc2 +%= f.await(io);
    t1 = std.Io.Timestamp.now(io, .awake);
    const async_ms = t0.durationTo(t1).toMilliseconds();

    std.debug.print("checksums equal: {}\n", .{acc == acc2});
    std.debug.print("serial {d} ms, io.async {d} ms\n", .{ serial_ms, async_ms });
    std.debug.print("cpus = {d}\n", .{try std.Thread.getCpuCount()});
}
```

```
checksums equal: true
serial 347 ms, io.async 73 ms
cpus = 12
```

~4.8x on 8 tasks. The numbers vary run to run — four more runs gave 3.7x to
5.3x — but the speedup is always there.

> `spin` takes no `io`. Tasks do not have to be I/O functions — `io.async` runs
> anything. And note `+%`/`*%`: plain `+` would **panic** on overflow in Debug.
>
> Build this one in Debug (plain `zig run`). In `-Ofast` the optimiser proves
> `spin`'s result unused-but-equal and deletes the loops, printing `0 ms` twice.

## Tuning the pool

`std.Io.Threaded.init(gpa, options)`:

| Option | Default | Effect |
|---|---|---|
| `async_limit` | one less than logical core count | pool cap for `io.async`. **Past it, tasks run inline** |
| `concurrent_limit` | `.unlimited` | pool cap for `io.concurrent`. Past it → `error.ConcurrencyUnavailable` |
| `stack_size` | 16 MiB (`std.Thread.SpawnConfig.default_stack_size`) | per worker thread |

`Io.Limit` is `.unlimited`, `.nothing`, or `.limited(n)`.

```zig
const std = @import("std");

fn sleeper(io: std.Io) void {
    std.Io.sleep(io, .fromMilliseconds(100), .awake) catch {};
}

fn run(gpa: std.mem.Allocator, label: []const u8, limit: std.Io.Limit) !void {
    var threaded: std.Io.Threaded = .init(gpa, .{ .async_limit = limit });
    defer threaded.deinit();
    const io = threaded.io();

    const t0 = std.Io.Timestamp.now(io, .awake);
    var futures: [4]std.Io.Future(void) = undefined;
    for (&futures) |*f| f.* = io.async(sleeper, .{io});
    for (&futures) |*f| f.await(io);
    const t1 = std.Io.Timestamp.now(io, .awake);

    std.debug.print("{s:<22} {d} ms\n", .{ label, t0.durationTo(t1).toMilliseconds() });
}

pub fn main(init: std.process.Init) !void {
    try run(init.gpa, "async_limit = .nothing", .nothing);
    try run(init.gpa, "async_limit = 1", .limited(1));
    try run(init.gpa, "async_limit = 3", .limited(3));
}
```

```
async_limit = .nothing 415 ms
async_limit = 1        209 ms
async_limit = 3        105 ms
```

Four 100 ms sleeps. With `.nothing` they are fully serial; with 3 workers the
main thread handles the fourth and everything overlaps. **This is the knob that
turns `io.async` from parallel to sequential without touching a line of task
code** — exactly the promise of explicit `io`.

## Which to reach for

| Situation | Use |
|---|---|
| fan out N independent jobs, collect results | `io.async` |
| writing a reusable library | `io.async` — never force a thread on your caller |
| server that must accept while a handler runs | `io.concurrent` |
| producer and consumer that wait on each other | `io.concurrent` |
| want to cap parallelism | `async_limit` / `concurrent_limit`, or a `Semaphore` (12.5) |
| don't care about return values | `std.Io.Group` |

---

Next: [12.3 — Cancellation](./03-cancellation.md)
