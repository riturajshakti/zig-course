# 12.5 — Mutexes & Locks

**`std.Thread.Mutex` no longer exists.** In 0.17 every synchronisation
primitive lives under `std.Io`, because blocking on one is an `io` operation —
which makes it a cancellation point.

| Primitive | Init | Lock / wait | Release |
|---|---|---|---|
| `std.Io.Mutex` | `.init` | `try m.lock(io)` · `m.tryLock()` | `m.unlock(io)` |
| `std.Io.RwLock` | `.init` | `try l.lock(io)` / `try l.lockShared(io)` · `tryLock(io)` / `tryLockShared(io)` | `unlock(io)` / `unlockShared(io)` |
| `std.Io.Condition` | `.init` | `try c.wait(io, &m)` · `try c.waitTimeout(io, &m, t)` | `c.signal(io)` · `c.broadcast(io)` |
| `std.Io.Semaphore` | `.{}` or `.{ .permits = n }` | `try s.wait(io)` · `try s.waitTimeout(io, t)` | `s.post(io)` |

Every blocking call above returns `Cancelable!…`, so it needs `try` or `catch`.
Each has an `…Uncancelable` twin that returns `void`. `unlock`, `signal`,
`broadcast` and `post` never block, so they never fail — but they still take
`io`.

## Why the lock is needed

```zig
const std = @import("std");

var mtx: std.Io.Mutex = .init;
var guarded: u64 = 0;
var unguarded: u64 = 0;

fn bumpBoth(io: std.Io) void {
    for (0..20_000) |_| {
        unguarded += 1;
        mtx.lock(io) catch return;
        guarded += 1;
        mtx.unlock(io);
    }
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const tasks = 6;

    var futures: [tasks]std.Io.Future(void) = undefined;
    for (&futures) |*f| f.* = try io.concurrent(bumpBoth, .{io});
    for (&futures) |*f| f.await(io);

    std.debug.print("expected   {d}\n", .{tasks * 20_000});
    std.debug.print("guarded    {d}\n", .{guarded});
    std.debug.print("unguarded  {d} (lost updates: {})\n", .{ unguarded, unguarded != tasks * 20_000 });
}
```

```
expected   120000
guarded    120000
unguarded  116806 (lost updates: true)
```

`guarded` is always exactly 120000. The `unguarded` number is different every
run — it was 117067, 117413, 116438 on three further runs. `x += 1` is a
load, an add and a store; two tasks interleaving those lose an increment.

## `tryLock` — never blocks, takes no `io`

```zig
const std = @import("std");

var mtx: std.Io.Mutex = .init;

pub fn main(init: std.process.Init) !void {
    const io = init.io;

    std.debug.print("tryLock on free mutex   -> {}\n", .{mtx.tryLock()});
    std.debug.print("tryLock while held      -> {}\n", .{mtx.tryLock()});
    mtx.unlock(io);

    try mtx.lock(io);
    std.debug.print("tryLock after lock(io)  -> {}\n", .{mtx.tryLock()});
    mtx.unlock(io);
    std.debug.print("tryLock after unlock    -> {}\n", .{mtx.tryLock()});
    mtx.unlock(io);
}
```

```
tryLock on free mutex   -> true
tryLock while held      -> false
tryLock after lock(io)  -> false
tryLock after unlock    -> true
```

`tryLock()` is the only lock operation with no `io` and no error — it cannot
block, so it cannot be cancelled. Use it for "update the cache if nobody else
is already doing it", never in a spin loop.

## `RwLock` — many readers, one writer

```zig
const std = @import("std");

var lock: std.Io.RwLock = .init;
var value: u32 = 0;
var reads_observed: std.atomic.Value(u32) = .init(0);

fn reader(io: std.Io) void {
    lock.lockShared(io) catch return;
    defer lock.unlockShared(io);
    std.Io.sleep(io, .fromMilliseconds(50), .awake) catch {};
    if (value == 0) _ = reads_observed.fetchAdd(1, .monotonic);
}

fn writer(io: std.Io) void {
    lock.lock(io) catch return;
    defer lock.unlock(io);
    value = 99;
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const t0 = std.Io.Timestamp.now(io, .awake);

    var rs: [4]std.Io.Future(void) = undefined;
    for (&rs) |*f| f.* = try io.concurrent(reader, .{io});
    for (&rs) |*f| f.await(io);

    const t1 = std.Io.Timestamp.now(io, .awake);
    var w = try io.concurrent(writer, .{io});
    w.await(io);

    std.debug.print("4 x 50ms shared readers took {d} ms\n", .{t0.durationTo(t1).toMilliseconds()});
    std.debug.print("all 4 saw value == 0: {}\n", .{reads_observed.load(.monotonic) == 4});
    std.debug.print("after writer: value = {d}\n", .{value});
}
```

```
4 x 50ms shared readers took 55 ms
all 4 saw value == 0: true
after writer: value = 99
```

55 ms, not 200 — the four readers genuinely overlapped. A plain `Mutex` would
have serialised them. Note `tryLock` on `RwLock` **does** take `io` (unlike
`Mutex.tryLock`).

Use `RwLock` when reads dominate *and* the critical section is long enough to
pay for the extra bookkeeping. For a short read of one word, an atomic (12.6)
is cheaper than either lock.

## `Condition` — wait for a state change

A condition variable is always: a mutex, a predicate, and a `while` loop. Never
an `if` — wakeups can be spurious.

```zig
const std = @import("std");

const Queue = struct {
    mtx: std.Io.Mutex = .init,
    not_empty: std.Io.Condition = .init,
    items: [8]u32 = undefined,
    len: usize = 0,
    closed: bool = false,
};

var q: Queue = .{};

fn producer(io: std.Io) void {
    for (1..6) |i| {
        std.Io.sleep(io, .fromMilliseconds(15), .awake) catch {};
        q.mtx.lock(io) catch return;
        q.items[q.len] = @intCast(i * 100);
        q.len += 1;
        q.not_empty.signal(io);
        q.mtx.unlock(io);
    }
    q.mtx.lock(io) catch return;
    q.closed = true;
    q.not_empty.broadcast(io);
    q.mtx.unlock(io);
}

fn consumer(io: std.Io) u32 {
    var sum: u32 = 0;
    while (true) {
        q.mtx.lock(io) catch return sum;
        while (q.len == 0 and !q.closed) q.not_empty.wait(io, &q.mtx) catch {
            q.mtx.unlock(io);
            return sum;
        };
        if (q.len == 0) {
            q.mtx.unlock(io);
            return sum;
        }
        q.len -= 1;
        sum += q.items[q.len];
        q.mtx.unlock(io);
    }
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    var p = try io.concurrent(producer, .{io});
    var c = try io.concurrent(consumer, .{io});
    const total = c.await(io);
    p.await(io);
    std.debug.print("consumer summed {d} (expected 1500)\n", .{total});
}
```

```
consumer summed 1500 (expected 1500)
```

| Rule | Why |
|---|---|
| hold the mutex when you call `wait` | `wait` unlocks it for you and re-locks before returning |
| `while (!predicate) wait(...)` | spurious wakeups are allowed by design |
| `signal` wakes one, `broadcast` wakes all | use `broadcast` for a state *everyone* cares about, e.g. "closed" |
| `wait` can return `error.Canceled` | and the mutex **is** re-locked before it does, so you must still unlock |
| needs `io.concurrent`, not `io.async` | with `io.async` the producer could run inline before the consumer exists |

`std.Io` also ships `Queue(T)` with `putOne`/`getOne`, which is this pattern
pre-built — closer to a Go channel. Reach for it before hand-rolling.

## `Semaphore` — cap the concurrency

```zig
const std = @import("std");

var gate: std.Io.Semaphore = .{ .permits = 2 };
var in_flight: std.atomic.Value(u32) = .init(0);
var peak: std.atomic.Value(u32) = .init(0);

fn limited(io: std.Io) void {
    gate.wait(io) catch return;
    defer gate.post(io);

    const now = in_flight.fetchAdd(1, .acq_rel) + 1;
    _ = peak.fetchMax(now, .acq_rel);
    std.Io.sleep(io, .fromMilliseconds(40), .awake) catch {};
    _ = in_flight.fetchSub(1, .acq_rel);
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    var fs: [8]std.Io.Future(void) = undefined;
    for (&fs) |*f| f.* = try io.concurrent(limited, .{io});
    for (&fs) |*f| f.await(io);
    std.debug.print("8 tasks, semaphore permits = 2, peak concurrency = {d}\n", .{peak.load(.acquire)});
}
```

```
8 tasks, semaphore permits = 2, peak concurrency = 2
```

Eight tasks, never more than two inside. This is the connection-pool /
rate-limit primitive — `p-limit` in Node, a buffered channel in Go.

`Semaphore` initialises with a plain struct literal (`.{}`), not `.init`,
because all its fields have defaults. `permits` starts at 0, so `.{}` is a
gate that blocks until someone `post`s.

## Deadlock, and how cancellation gets you out

Two mutexes, two tasks, opposite acquisition order. The sleep makes it certain
rather than occasional.

```zig
const std = @import("std");

var a: std.Io.Mutex = .init;
var b: std.Io.Mutex = .init;

fn ab(io: std.Io) std.Io.Cancelable!void {
    try a.lock(io);
    defer a.unlock(io);
    try std.Io.sleep(io, .fromMilliseconds(30), .awake);
    try b.lock(io);
    defer b.unlock(io);
}

fn ba(io: std.Io) std.Io.Cancelable!void {
    try b.lock(io);
    defer b.unlock(io);
    try std.Io.sleep(io, .fromMilliseconds(30), .awake);
    try a.lock(io);
    defer a.unlock(io);
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;

    var f1 = try io.concurrent(ab, .{io});
    var f2 = try io.concurrent(ba, .{io});
    try std.Io.sleep(io, .fromMilliseconds(200), .awake);
    std.debug.print("after 200 ms both still blocked -> deadlock\n", .{});
    std.debug.print("f1 = {!}\n", .{f1.cancel(io)});
    std.debug.print("f2 = {!}\n", .{f2.cancel(io)});
}
```

```
after 200 ms both still blocked -> deadlock
f1 = error.Canceled
f2 = void
```

Read that second line: cancelling `f1` made its `try b.lock(io)` fail, which
ran `defer a.unlock(io)` — which released the lock `f2` was waiting for, so
`f2` completed normally. **Cancelable locks turn a hard hang into a
recoverable condition.** In Go or C this program hangs forever.

### The fix: one global lock order

```zig
const std = @import("std");

var a: std.Io.Mutex = .init;
var b: std.Io.Mutex = .init;
var done: std.atomic.Value(u32) = .init(0);

fn bothInOrder(io: std.Io) std.Io.Cancelable!void {
    try a.lock(io);
    defer a.unlock(io);
    try std.Io.sleep(io, .fromMilliseconds(10), .awake);
    try b.lock(io);
    defer b.unlock(io);
    _ = done.fetchAdd(1, .monotonic);
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const t0 = std.Io.Timestamp.now(io, .awake);

    var fs: [4]std.Io.Future(std.Io.Cancelable!void) = undefined;
    for (&fs) |*f| f.* = try io.concurrent(bothInOrder, .{io});
    for (&fs) |*f| try f.await(io);

    const t1 = std.Io.Timestamp.now(io, .awake);
    std.debug.print("all 4 completed in {d} ms, done = {d}\n", .{ t0.durationTo(t1).toMilliseconds(), done.load(.monotonic) });
}
```

```
all 4 completed in 48 ms, done = 4
```

Every path takes `a` before `b`. No cycle, therefore no deadlock — the only
general fix. Four tasks × 10 ms serialised behind `a` ≈ 48 ms.

| Avoiding deadlock | |
|---|---|
| **lock ordering** | pick a global order (address, id, name) and never deviate. The real fix |
| **one lock** | if two mutexes always travel together, they are one mutex |
| **`tryLock` + back off** | release everything and retry; livelock risk |
| **timeout / cancel** | detects it, doesn't prevent it. Good for a watchdog |
| **no I/O under a lock** | a `sleep` or socket read inside a critical section is how 30 ms becomes 30 s |

## Lock granularity

One shared mutex serialises everything. Per-slot mutexes do not — provided the
data each task touches sits on its **own cache line**, or false sharing eats
the win.

```zig
const std = @import("std");

const stripes = 8;
const rounds = 200_000;

const Padded = struct {
    mtx: std.Io.Mutex align(std.atomic.cache_line) = .init,
    value: u64 = 0,
};

var one_lock: std.Io.Mutex = .init;
var one_data: [stripes]Padded = @splat(.{});
var many: [stripes]Padded = @splat(.{});

fn coarse(io: std.Io, slot: usize) void {
    for (0..rounds) |_| {
        one_lock.lock(io) catch return;
        one_data[slot].value += 1;
        one_lock.unlock(io);
    }
}

fn fine(io: std.Io, slot: usize) void {
    for (0..rounds) |_| {
        many[slot].mtx.lock(io) catch return;
        many[slot].value += 1;
        many[slot].mtx.unlock(io);
    }
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;

    var t0 = std.Io.Timestamp.now(io, .awake);
    var fs: [stripes]std.Io.Future(void) = undefined;
    for (&fs, 0..) |*f, i| f.* = try io.concurrent(coarse, .{ io, i });
    for (&fs) |*f| f.await(io);
    var t1 = std.Io.Timestamp.now(io, .awake);
    const coarse_us = t0.durationTo(t1).toMicroseconds();

    t0 = std.Io.Timestamp.now(io, .awake);
    for (&fs, 0..) |*f, i| f.* = try io.concurrent(fine, .{ io, i });
    for (&fs) |*f| f.await(io);
    t1 = std.Io.Timestamp.now(io, .awake);
    const fine_us = t0.durationTo(t1).toMicroseconds();

    var ca: u64 = 0;
    var fa: u64 = 0;
    for (one_data, many) |c, f| {
        ca += c.value;
        fa += f.value;
    }

    std.debug.print("1 shared mutex   : {d} us, total {d}\n", .{ coarse_us, ca });
    std.debug.print("{d} private mutexes: {d} us, total {d}\n", .{ stripes, fine_us, fa });
}
```

```sh
zig run -Ofast granularity.zig
```

```
1 shared mutex   : 15796 us, total 1600000
8 private mutexes: 629 us, total 1600000
```

~25x, same total. Both numbers vary run to run; the ratio holds.

> `std.atomic.cache_line` is **128** bytes on this Apple Silicon machine, not
> 64. Drop the `align` and the striped version gets *slower* than the shared
> one: the eight `Padded` structs land on the same line and the cores fight
> over it even though they never touch each other's data.

## Picking a primitive

| Shared state | Reach for |
|---|---|
| one integer or flag | `std.atomic.Value(T)` — 12.6, no lock at all |
| a struct, short critical section | `Io.Mutex` |
| read-mostly, long reads | `Io.RwLock` |
| "wait until X" | `Io.Condition` + a mutex, or `Io.Queue(T)` |
| "at most N at once" | `Io.Semaphore` |
| per-task scratch, combined at the end | **nothing** — see 12.4's chunk example |

The last row is the one people skip. The fastest lock is the one you deleted by
giving each task its own data.

---

Next: [12.6 — Atomics](./06-atomics.md)
