# 12.3 — Cancellation

Node has no cancellation (`AbortController` is a convention, not enforcement).
Go passes a `context.Context` by hand and hopes everyone checks it. Python
raises `CancelledError` at `await` points.

Zig takes Python's model and puts it in the type system: **`error.Canceled` is
a member of `std.Io.Cancelable`, and every `io` function that can block
returns it.** If a function can be cancelled, its error set says so.

```zig
pub const Cancelable = error{ Canceled };
```

| Term | Meaning |
|---|---|
| **cancellation point** | a call to an `io` function that can return `error.Canceled` |
| **cancellation request** | `future.cancel(io)` or `group.cancel(io)` |
| the guarantee | the task's **next** cancellation point returns `error.Canceled` — once |
| the trap | later points do **not** re-signal. Swallowing `error.Canceled` is almost always a bug |

## `future.cancel(io)`

Same as `await`, plus a cancellation request first. It still blocks until the
task actually finishes, and returns whatever the task returned.

```zig
const std = @import("std");

fn longSleep(io: std.Io) std.Io.Cancelable!u32 {
    try std.Io.sleep(io, .fromMilliseconds(5000), .awake);
    return 42;
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const t0 = std.Io.Timestamp.now(io, .awake);

    var f = try io.concurrent(longSleep, .{io});
    try std.Io.sleep(io, .fromMilliseconds(50), .awake);
    const r = f.cancel(io);

    const t1 = std.Io.Timestamp.now(io, .awake);
    std.debug.print("result = {!d} after {d} ms\n", .{ r, t0.durationTo(t1).toMilliseconds() });
}
```

```
result = error.Canceled after 55 ms
```

Not 5000 ms. The `sleep` inside the task was the cancellation point, and the
`try` propagated it out as the task's result.

> This uses `io.concurrent`, not `io.async`. With `io.async` the task might run
> inline and finish before you could cancel it.

## Cleanup still runs

Cancellation is an ordinary Zig error, so `defer` and `errdefer` behave
normally — no special unwinding, no "cancellation-safe" discipline to learn.

```zig
const std = @import("std");

fn worker(io: std.Io, gpa: std.mem.Allocator, out: *usize) std.Io.Cancelable!void {
    const buf = gpa.alloc(u8, 1024) catch return;
    defer {
        gpa.free(buf);
        std.debug.print("worker: buffer freed\n", .{});
    }
    errdefer std.debug.print("worker: errdefer ran\n", .{});

    var i: usize = 0;
    while (i < 100) : (i += 1) {
        try std.Io.sleep(io, .fromMilliseconds(10), .awake);
        out.* = i;
    }
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    var progress: usize = 0;

    var f = try io.concurrent(worker, .{ io, init.gpa, &progress });
    try std.Io.sleep(io, .fromMilliseconds(60), .awake);
    const r = f.cancel(io);

    std.debug.print("cancel returned {!}, progress reached {d}\n", .{ r, progress });
}
```

```
worker: errdefer ran
worker: buffer freed
cancel returned error.Canceled, progress reached 4
```

`errdefer` fires (cancellation is an error path), then `defer`, in reverse
order. No leak, and `init.gpa`'s leak checker confirms it.

> `progress reached 4` varies between runs — it depends on how many 10 ms
> sleeps completed before the 60 ms mark. The fact that both cleanups run does
> not vary.

## CPU-bound tasks: `io.checkCancel`

A task with no `io` calls has no cancellation points, so a cancellation request
is never observed. Insert one yourself.

```zig
const std = @import("std");

fn grind(io: std.Io, done: *usize) std.Io.Cancelable!u64 {
    var acc: u64 = 0;
    var round: usize = 0;
    while (true) : (round += 1) {
        try io.checkCancel();
        var i: u64 = 0;
        while (i < 2_000_000) : (i += 1) acc = acc +% i;
        done.* = round + 1;
    }
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    var rounds: usize = 0;

    var f = try io.concurrent(grind, .{ io, &rounds });
    try std.Io.sleep(io, .fromMilliseconds(100), .awake);
    const r = f.cancel(io);

    std.debug.print("grind -> {!d}, completed {d} rounds\n", .{ r, rounds });
}
```

```
grind -> error.Canceled, completed 52 rounds
```

The round count varies run to run; `error.Canceled` does not. This is Go's
`select { case <-ctx.Done(): }` with the check made explicit and cheap.

## Timeouts: `std.Io.Timeout`

```zig
pub const Timeout = union(enum) { none, duration: Clock.Duration, deadline: Clock.Timestamp };
```

Note `duration` is a **`Clock.Duration`** — a `{ raw, clock }` pair — not an
`Io.Duration`. Writing `.{ .duration = .fromMilliseconds(80) }` is a compile
error: *`struct 'Io.Clock.Duration' has no member named 'fromMilliseconds'`*.

```zig
const std = @import("std");

var mtx: std.Io.Mutex = .init;
var cond: std.Io.Condition = .init;
var ready: bool = false;

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const budget: std.Io.Timeout = .{ .duration = .{ .raw = .fromMilliseconds(80), .clock = .awake } };

    try mtx.lock(io);
    defer mtx.unlock(io);

    const t0 = std.Io.Timestamp.now(io, .awake);
    const r = cond.waitTimeout(io, &mtx, budget);
    const t1 = std.Io.Timestamp.now(io, .awake);

    std.debug.print("waitTimeout -> {!}, ready = {}, after {d} ms\n", .{ r, ready, t0.durationTo(t1).toMilliseconds() });
    std.debug.print("as deadline: {t}\n", .{budget.toDeadline(io)});
    try budget.sleep(io);
    std.debug.print("budget.sleep(io) returned\n", .{});
}
```

```
waitTimeout -> error.Timeout, ready = false, after 81 ms
as deadline: deadline
budget.sleep(io) returned
```

| Member | Use |
|---|---|
| `.none` | wait forever |
| `.{ .duration = .{ .raw = …, .clock = .awake } }` | relative budget |
| `.{ .deadline = ts }` | absolute instant; survives being passed down a call chain |
| `.toDeadline(io)` | convert relative → absolute, so retries share one overall budget |
| `.sleep(io)` | sleep out the timeout |

`error.Timeout` is a **separate** error from `error.Canceled`. Functions that
take a timeout return `Cancelable || Timeout.Error`.

## Racing a task against a clock: `std.Io.Select`

`Io.Timeout` only applies to functions that accept one. To put a deadline on an
*arbitrary* task, race it — Zig's `Promise.race` / Go's `select`.

```zig
const std = @import("std");

const Outcome = union(enum) {
    work: std.Io.Cancelable!i64,
    timeout: void,
};

fn job(io: std.Io, ms: i64) std.Io.Cancelable!i64 {
    try std.Io.sleep(io, .fromMilliseconds(ms), .awake);
    return ms;
}

fn after(io: std.Io, ms: i64) void {
    std.Io.sleep(io, .fromMilliseconds(ms), .awake) catch {};
}

fn race(io: std.Io, job_ms: i64, budget_ms: i64) !void {
    var buf: [2]Outcome = undefined;
    var sel: std.Io.Select(Outcome) = .init(io, &buf);
    defer sel.cancelDiscard();

    try sel.concurrent(.work, job, .{ io, job_ms });
    try sel.concurrent(.timeout, after, .{ io, budget_ms });

    switch (try sel.await()) {
        .work => |r| std.debug.print("job({d}) within {d} ms budget -> {!d}\n", .{ job_ms, budget_ms, r }),
        .timeout => std.debug.print("job({d}) exceeded {d} ms budget -> error.Timeout\n", .{ job_ms, budget_ms }),
    }
}

pub fn main(init: std.process.Init) !void {
    try race(init.io, 30, 200);
    try race(init.io, 400, 200);
}
```

```
job(30) within 200 ms budget -> 30
job(400) exceeded 200 ms budget -> error.Timeout
```

| `Select` piece | Note |
|---|---|
| `std.Io.Select(U)` | `U` is a **tagged union**; each field is one racer's result type |
| `.init(io, &buf)` | `buf` is `[]U`, sized for all racers |
| `.concurrent(.tag, fn, args)` | `fn`'s return type must match that field's type |
| `.await()` | takes **no `io`** — it was captured in `init`. Returns the first `U` |
| `.cancelDiscard()` | cancels and drops the losers. `defer` it |
| `.cancel()` | use instead when a loser can return something that needs freeing; call in a loop until `null` |

## Why `lock(io)` can fail

`std.Io.Mutex.lock` is `Cancelable!void` for exactly this reason: a task
blocked waiting for a mutex is blocked in `io`, so it is a cancellation point.

```zig
const std = @import("std");

var mtx: std.Io.Mutex = .init;

fn wantsLock(io: std.Io) std.Io.Cancelable!void {
    try mtx.lock(io);
    defer mtx.unlock(io);
    std.debug.print("got the lock (should not happen)\n", .{});
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;

    try mtx.lock(io);
    var f = try io.concurrent(wantsLock, .{io});
    try std.Io.sleep(io, .fromMilliseconds(50), .awake);

    std.debug.print("blocked task cancel -> {!}\n", .{f.cancel(io)});
    mtx.unlock(io);

    std.debug.print("tryLock now -> {}\n", .{mtx.tryLock()});
    mtx.unlock(io);
}
```

```
blocked task cancel -> error.Canceled
tryLock now -> true
```

The task never entered the critical section, so its `defer mtx.unlock` never
ran — correct, because the `try` failed *before* the lock was acquired. Put the
`defer` after the `try` and this is automatic.

If you genuinely must not be interrupted, every primitive has an
`…Uncancelable` twin: `lockUncancelable(io)`, `Condition.waitUncancelable`,
`Semaphore.waitUncancelable`. They return `void`.

## Cooperative shutdown

You don't always want cancellation. An atomic flag plus a loop condition is
often clearer, and lets the task finish its current unit of work.

```zig
const std = @import("std");

var stop: std.atomic.Value(bool) = .init(false);

fn poller(io: std.Io, ticks: *u32) void {
    while (!stop.load(.acquire)) {
        std.Io.sleep(io, .fromMilliseconds(10), .awake) catch break;
        ticks.* += 1;
    }
    std.debug.print("poller exiting cleanly\n", .{});
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    var ticks: u32 = 0;

    var f = try io.concurrent(poller, .{ io, &ticks });
    try std.Io.sleep(io, .fromMilliseconds(100), .awake);
    stop.store(true, .release);
    f.await(io);

    std.debug.print("ticks > 0: {}\n", .{ticks > 0});
}
```

```
poller exiting cleanly
ticks > 0: true
```

| Mechanism | Latency | Task finishes current work? | Needs |
|---|---|---|---|
| `future.cancel(io)` | immediate at next cancellation point | no, it aborts | `Cancelable` in the signature |
| atomic flag | up to one loop iteration | yes | an atomic and a loop check |
| `group.cancel(io)` | immediate, propagates to all members | no | nothing extra |

## Rules

1. Don't swallow `error.Canceled`. `catch {}` around a cancellation point means
   the cancellation silently vanishes and the next point will not re-signal it.
   Use `std.Io.recancel(io)` if you must handle it and then re-arm.
2. `defer` after the `try`, never before — cleanup should only run for
   resources you actually acquired.
3. `cancel` blocks until the task is done. It requests, it does not kill.
4. `error.Timeout` ≠ `error.Canceled`.
5. `Group.cancel(io)` propagates to every member and returns only once they
   have all finished.

---

Next: [12.4 — Threads](./04-threads.md)
