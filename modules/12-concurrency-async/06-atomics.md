# 12.6 — Atomics

`std.atomic.Value(T)` is a one-field `extern struct` wrapping the `@atomic*`
builtins. No `io`, no allocation, no cancellation — just instructions.

```
pub fn Value(comptime T: type) type {
    return extern struct { raw: T, ... };
}
```

`.raw` is public so you can reach it for a non-atomic access. Doing that in
concurrent code is a data race; it exists for initialisation and for
`io.futexWait`-style APIs that want the address.

## Every operation

```zig
const std = @import("std");

pub fn main() void {
    var v: std.atomic.Value(u32) = .init(10);

    std.debug.print("load         {d}\n", .{v.load(.monotonic)});
    v.store(20, .monotonic);
    std.debug.print("after store  {d}\n", .{v.load(.monotonic)});
    std.debug.print("fetchAdd(5)  returns {d}, now {d}\n", .{ v.fetchAdd(5, .monotonic), v.load(.monotonic) });
    std.debug.print("fetchSub(3)  returns {d}, now {d}\n", .{ v.fetchSub(3, .monotonic), v.load(.monotonic) });
    std.debug.print("swap(99)     returns {d}, now {d}\n", .{ v.swap(99, .acq_rel), v.load(.monotonic) });
    std.debug.print("fetchMax(50) returns {d}, now {d}\n", .{ v.fetchMax(50, .monotonic), v.load(.monotonic) });
    std.debug.print("fetchOr(0x6) returns {d}, now {d}\n", .{ v.fetchOr(0x6, .monotonic), v.load(.monotonic) });

    std.debug.print("cmpxchg 103->7 (wrong expect) -> {?d}\n", .{v.cmpxchgStrong(42, 7, .acq_rel, .monotonic)});
    std.debug.print("cmpxchg 103->7 (right expect) -> {?d}, now {d}\n", .{ v.cmpxchgStrong(103, 7, .acq_rel, .monotonic), v.load(.monotonic) });

    std.debug.print("bitSet(3)    -> {d}, now {d}\n", .{ v.bitSet(3, .monotonic), v.load(.monotonic) });
    std.debug.print("cache_line   = {d} bytes\n", .{std.atomic.cache_line});
}
```

```
load         10
after store  20
fetchAdd(5)  returns 20, now 25
fetchSub(3)  returns 25, now 22
swap(99)     returns 22, now 99
fetchMax(50) returns 99, now 99
fetchOr(0x6) returns 99, now 103
cmpxchg 103->7 (wrong expect) -> 103
cmpxchg 103->7 (right expect) -> null, now 7
bitSet(3)    -> 0, now 15
cache_line   = 128 bytes
```

| Method | Returns | Note |
|---|---|---|
| `.init(x)` | `Value(T)` | the only constructor |
| `load(order)` | current value | order must not be `.release`/`.acq_rel` |
| `store(x, order)` | `void` | order must not be `.acquire`/`.acq_rel` |
| `swap(x, order)` | **old** value | |
| `fetchAdd` / `fetchSub` | **old** value | wraps on overflow; never panics |
| `fetchMin` / `fetchMax` | **old** value | one-instruction lock-free min/max |
| `fetchAnd` / `fetchOr` / `fetchXor` / `fetchNand` | **old** value | |
| `cmpxchgStrong(expect, new, ok, fail)` | **`?T`** — `null` on success, the actual value on failure | |
| `cmpxchgWeak(...)` | same | may fail spuriously; **only valid inside a retry loop** |
| `bitSet` / `bitReset` / `bitToggle` | `u1`, the old bit | |
| `rmw(.Add, x, order)` | old value | generic form of the `fetch*` family |

All `fetch*` return the value **before** the operation, like
`__atomic_fetch_add`, not like `++x`.

## Memory orderings

| Ordering | Guarantees | Use for |
|---|---|---|
| `.unordered` | atomicity only, no ordering at all | almost never |
| `.monotonic` | atomicity; no ordering w.r.t. other variables | counters and statistics you only read at the end |
| `.acquire` | on a **load**: nothing after it moves before it | reading a flag that publishes other data |
| `.release` | on a **store**: nothing before it moves after it | setting the flag that publishes that data |
| `.acq_rel` | both, for read-modify-write | `swap`, `cmpxchg`, lock acquire/release |
| `.seq_cst` | single global total order across all `seq_cst` ops | when you're unsure; the only one that composes trivially |

Using the wrong one is a **compile error**, not a subtle bug:

```zig
const std = @import("std");
var v: std.atomic.Value(u32) = .init(0);
pub fn main() void {
    std.debug.print("{d}\n", .{v.load(.release)});
}
```

```
/opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/atomic.zig:21:46: error: @atomicLoad atomic ordering must not be release or acq_rel
            return @atomicLoad(T, &self.raw, order);
                                             ^~~~~
f6.zig:4:38: note: called inline here
    std.debug.print("{d}\n", .{v.load(.release)});
                               ~~~~~~^~~~~~~~~~
```

> **`@fence` was removed in 0.17** — `error: invalid builtin function: '@fence'`.
> Standalone fences are gone; put the ordering on the operation that needs it.
> That is strictly better: `release` on the store that publishes is local and
> checkable, where a loose `@fence` was neither.

## release / acquire in practice

The one pattern worth memorising: a `.release` store publishes everything
written before it, and a matching `.acquire` load makes all of it visible.

```zig
const std = @import("std");

var payload: u64 = 0;
var flag: std.atomic.Value(bool) = .init(false);
var seen: std.atomic.Value(u64) = .init(0);

fn publish(io: std.Io) void {
    std.Io.sleep(io, .fromMilliseconds(10), .awake) catch {};
    payload = 0xCAFE;
    flag.store(true, .release);
}

fn consume(io: std.Io) void {
    while (!flag.load(.acquire)) std.Io.sleep(io, .fromMilliseconds(1), .awake) catch return;
    seen.store(payload, .monotonic);
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    var c = try io.concurrent(consume, .{io});
    var p = try io.concurrent(publish, .{io});
    c.await(io);
    p.await(io);
    std.debug.print("consumer saw payload = 0x{X}\n", .{seen.load(.monotonic)});
}
```

```
consumer saw payload = 0xCAFE
```

`payload` is a **plain `u64`** with no atomic wrapper, read and written from
two different tasks — and that is correct *only* because of the
release/acquire pair on `flag`. Downgrade either to `.monotonic` and the
compiler (or the CPU) may reorder `payload = 0xCAFE` after the flag store, and
the consumer can legally see 0.

## cmpxchg: lock-free read-modify-write

`cmpxchgWeak` in a loop is how you build any atomic operation the hardware
doesn't give you directly:

```zig
const std = @import("std");

var high: std.atomic.Value(u32) = .init(0);

fn casMax(v: u32) void {
    var cur = high.load(.monotonic);
    while (v > cur) cur = high.cmpxchgWeak(cur, v, .acq_rel, .monotonic) orelse return;
}

fn bidder(_: std.Io, base: u32) void {
    for (0..2000) |i| casMax(base + @as(u32, @intCast(i)));
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    var fs: [6]std.Io.Future(void) = undefined;
    for (&fs, 0..) |*f, i| f.* = try io.concurrent(bidder, .{ io, @as(u32, @intCast(i * 1000)) });
    for (&fs) |*f| f.await(io);
    std.debug.print("lock-free max across 6 tasks = {d}\n", .{high.load(.monotonic)});
    std.debug.print("fetchMax does the same in one call: {d}\n", .{high.fetchMax(9999, .acq_rel)});
    std.debug.print("now {d}\n", .{high.load(.monotonic)});
}
```

```
lock-free max across 6 tasks = 6999
fetchMax does the same in one call: 6999
now 9999
```

The CAS loop shape, which never changes:

```
var cur = x.load(.monotonic);
while (needsChange(cur)) cur = x.cmpxchgWeak(cur, newValue(cur), .acq_rel, .monotonic) orelse break;
```

`orelse` catches the `null` that means success; on failure `cmpxchg` hands back
the *actual* value so you retry against fresh data without reloading.

| | `cmpxchgStrong` | `cmpxchgWeak` |
|---|---|---|
| spurious failure | never | allowed |
| cost | may be a loop internally | single instruction (LL/SC) |
| use | a one-shot "claim this slot" | inside your own retry loop |

Use `fetchMax` when it exists. The CAS loop is for operations that don't.

## Benchmarked: mutex vs atomic vs neither

The lesson everyone gets backwards. Measured, `-Ofast`, 8 tasks × 500 000
increments:

```zig
const std = @import("std");

const tasks = 8;
const rounds = 500_000;

var mtx: std.Io.Mutex = .init;
var plain: u64 = 0;
var shared: std.atomic.Value(u64) = .init(0);

const Local = struct { n: u64 align(std.atomic.cache_line) = 0 };
var locals: [tasks]Local = @splat(.{});

fn viaMutex(io: std.Io) void {
    for (0..rounds) |_| {
        mtx.lock(io) catch return;
        plain += 1;
        mtx.unlock(io);
    }
}

fn viaSharedAtomic(_: std.Io) void {
    for (0..rounds) |_| _ = shared.fetchAdd(1, .monotonic);
}

fn viaLocalThenCombine(_: std.Io, slot: usize) void {
    var n: std.atomic.Value(u64) = .init(0);
    for (0..rounds) |_| _ = n.fetchAdd(1, .monotonic);
    locals[slot].n = n.load(.monotonic);
}

fn time(io: std.Io, comptime f: anytype) !i64 {
    const t0 = std.Io.Timestamp.now(io, .awake);
    var fs: [tasks]std.Io.Future(void) = undefined;
    for (&fs) |*fu| fu.* = try io.concurrent(f, .{io});
    for (&fs) |*fu| fu.await(io);
    return t0.durationTo(std.Io.Timestamp.now(io, .awake)).toMilliseconds();
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;

    const m = try time(io, viaMutex);
    const a = try time(io, viaSharedAtomic);

    const t0 = std.Io.Timestamp.now(io, .awake);
    var fs: [tasks]std.Io.Future(void) = undefined;
    for (&fs, 0..) |*fu, i| fu.* = try io.concurrent(viaLocalThenCombine, .{ io, i });
    for (&fs) |*fu| fu.await(io);
    var combined: u64 = 0;
    for (locals) |l| combined += l.n;
    const l = t0.durationTo(std.Io.Timestamp.now(io, .awake)).toMilliseconds();

    std.debug.print("expected               {d}\n", .{tasks * rounds});
    std.debug.print("Io.Mutex               {d}  {d} ms\n", .{ plain, m });
    std.debug.print("one shared atomic      {d}  {d} ms\n", .{ shared.load(.monotonic), a });
    std.debug.print("padded locals + sum    {d}  {d} ms\n", .{ combined, l });
}
```

```sh
zig run -Ofast counters.zig
```

```
expected               4000000
Io.Mutex               4000000  25 ms
one shared atomic      4000000  119 ms
padded locals + sum    4000000  1 ms
```

All three get 4000000. The timings are the point, and the ordering is stable
across runs (the exact milliseconds are not):

- **The shared atomic is 5x slower than the mutex.** "Lock-free is faster" is
  false under contention. Eight cores doing `lock xadd` on one address bounce
  that cache line between them continuously; the mutex, by contrast, lets one
  task hold the line and run uncontended for a stretch.
- **The padded-local version does the exact same 4 000 000 atomic RMWs and is
  ~100x faster than the shared one.** Identical instruction count, identical
  ordering, *different address* — one private cache line each. The cost was
  never the atomic. It was the sharing.

| Rule | |
|---|---|
| one word, low contention | atomic. Cheapest thing there is |
| one word, high contention | shard it, then combine. Not a "faster lock" |
| a struct, or any invariant across fields | mutex. Atomics cannot make two writes one unit |
| hot counters in a loop | accumulate in a local, publish once at the end |
| anything performance-critical | **measure in release mode.** Debug inverts every result here |

## Where atomics are genuinely the answer

| Use | Shape |
|---|---|
| shutdown flag | `Value(bool)`, `.release` store / `.acquire` load — 12.3 |
| statistics you read at the end | `fetchAdd(1, .monotonic)` |
| "did anyone else already do this?" | `cmpxchgStrong(false, true, …) == null` |
| lazy one-time init | `cmpxchgStrong` on a state enum: `.uninit` -> `.busy` -> `.ready` |
| reference counts | `fetchSub(1, .acq_rel) == 1` means you were last |
| high-water marks | `fetchMax` |

Everything else: start with a mutex. It is easier to reason about, easier to
get right, and — as measured above — frequently faster.

---

Next: [12.7 — Concurrent Echo Server](./07-echo-server.md)
