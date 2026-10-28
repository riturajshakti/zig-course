# 7.7 — Time & Random

## ⚠️ `std.time.Timer` is gone

This is the biggest break in the module. In 0.15 you wrote:

```zig
var timer = try std.time.Timer.start();   // does not compile in 0.17
const ns = timer.read();
```

In 0.17 `std.time` holds **only unit constants** — no `Timer`, no
`timestamp()`, no `milliTimestamp()`. Reading a clock is I/O, so it needs `io`:

```zig
const start: std.Io.Clock.Timestamp = .now(io, .awake);
// … work …
const elapsed = start.durationTo(.now(io, .awake));
```

Which means **any program that measures time needs
`pub fn main(init: std.process.Init)`**.

| 0.13–0.15 | 0.17.0 |
|---|---|
| `std.time.Timer.start()` | `std.Io.Clock.Timestamp.now(io, .awake)` |
| `timer.read()` | `start.durationTo(.now(io, .awake))` |
| `std.time.timestamp()` | `std.Io.Clock.Timestamp.now(io, .real).raw.toSeconds()` |
| `std.time.milliTimestamp()` | the same, via `.toNanoseconds()` |
| `std.time.sleep(ns)` | `duration.sleep(io)` |
| `Clock.monotonic` | `Clock.awake` |
| `Duration.fromMs(n)` | `Duration.fromMilliseconds(n)` |
| `std.fmt.fmtDuration(ns)` | `{f}` on a `std.Io.Duration` |
| `std.crypto.random.bytes(buf)` | `io.random(buf)` / `try io.randomSecure(buf)` |

## Operations table

| Operation | Code |
|---|---|
| read a monotonic clock | `std.Io.Clock.Timestamp.now(io, .awake)` |
| read wall-clock time | `std.Io.Clock.Timestamp.now(io, .real)` |
| CPU time used | `std.Io.Clock.Timestamp.now(io, .cpu_process)` |
| elapsed between two | `a.durationTo(b)` → `Clock.Duration` |
| elapsed since one | `t.untilNow(io)` |
| the raw duration | `d.raw` → `std.Io.Duration` |
| duration → µs / ms / s / ns | `d.raw.toMicroseconds()` / `toMilliseconds()` / `toSeconds()` / `toNanoseconds()` |
| duration from a number | `std.Io.Duration.fromMilliseconds(50)` |
| print a duration | `{f}` on `d.raw` → `1.5s`, `1.234us` |
| sleep | `try duration.sleep(io)` |
| a deadline | `Clock.Timestamp.fromNow(io, d)` then `try t.wait(io)` |
| unit constants | `std.time.ns_per_ms`, `s_per_day`, … |
| seeded PRNG | `var p: std.Random.DefaultPrng = .init(42);` |
| get the interface | `const r = p.random();` |
| entropy for a seed | `io.random(std.mem.asBytes(&seed))` |
| cryptographic bytes | `try io.randomSecure(&buf)` |
| any integer | `r.int(u8)` / `r.int(i32)` |
| inclusive range | `r.intRangeAtMost(u8, 1, 6)` |
| exclusive range | `r.intRangeLessThan(u8, 0, 10)` |
| `0..n-1` | `r.uintLessThan(u32, n)` |
| float in `[0,1)` | `r.float(f64)` |
| normal distribution | `r.floatNorm(f64)` |
| coin flip | `r.boolean()` |
| a random enum value | `r.enumValue(E)` |
| weighted pick | `r.weightedIndex(u32, &.{ 1, 1, 98 })` |
| fill bytes | `r.bytes(&buf)` |
| shuffle in place | `r.shuffle(T, slice)` |

## Measuring elapsed time

```zig
const std = @import("std");

fn work(n: u64) u64 {
    var acc: u64 = 0;
    for (0..n) |i| acc +%= i * i;
    return acc;
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;

    const start: std.Io.Clock.Timestamp = .now(io, .awake);
    const r = work(2_000_000);
    const end: std.Io.Clock.Timestamp = .now(io, .awake);

    const elapsed = start.durationTo(end);
    std.debug.print("result    = {d}\n", .{r});
    std.debug.print("elapsed   = {f}\n", .{elapsed.raw});
    std.debug.print("micros    = {d}\n", .{elapsed.raw.toMicroseconds()});
    std.debug.print("millis    = {d}\n", .{elapsed.raw.toMilliseconds()});
    std.debug.print("nanos     = {d}\n", .{elapsed.raw.toNanoseconds()});
}
```

```
result    = 2666664666667000000
elapsed   = 3.117ms
micros    = 3117
millis    = 3
nanos     = 3117709
```

(Your timings will differ — that's the point of measuring.)

There are **two** Duration types and the difference matters:

| Type | What it is |
|---|---|
| `std.Io.Clock.Duration` | a duration **tagged with which clock** it came from |
| `std.Io.Duration` | the plain nanosecond count, reachable as `.raw` |

`durationTo` gives you the tagged one. The constructors
(`fromMilliseconds`, `fromSeconds`), the converters (`toMicroseconds`) and the
`format` method all live on the **plain** one, so you go through `.raw`. The tag
exists so the compiler stops you subtracting a wall-clock timestamp from a
monotonic one.

`+%=` in `work` is wrapping addition — plain `+` would panic on overflow in
Debug.

## Which clock

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const io = init.io;

    const real: std.Io.Clock.Timestamp = .now(io, .real);
    const secs = real.raw.toSeconds();
    std.debug.print("unix seconds  = {d}\n", .{secs});
    std.debug.print("unix millis   = {d}\n", .{@divTrunc(real.raw.toNanoseconds(), std.time.ns_per_ms)});

    const es = std.time.epoch.EpochSeconds{ .secs = @intCast(secs) };
    const day = es.getEpochDay();
    const yd = day.calculateYearDay();
    const md = yd.calculateMonthDay();
    const ds = es.getDaySeconds();
    std.debug.print("date (UTC)    = {d:0>4}-{d:0>2}-{d:0>2}\n", .{ yd.year, md.month.numeric(), md.day_index + 1 });
    std.debug.print("time (UTC)    = {d:0>2}:{d:0>2}:{d:0>2}\n", .{ ds.getHoursIntoDay(), ds.getMinutesIntoHour(), ds.getSecondsIntoMinute() });

    const awake: std.Io.Clock.Timestamp = .now(io, .awake);
    const cpu: std.Io.Clock.Timestamp = .now(io, .cpu_process);
    std.debug.print("awake uptime  = {f}\n", .{std.Io.Duration{ .nanoseconds = awake.raw.toNanoseconds() }});
    std.debug.print("cpu_process   = {f}\n", .{std.Io.Duration{ .nanoseconds = cpu.raw.toNanoseconds() }});
}
```

```
unix seconds  = 1793166988
unix millis   = 1793166988416
date (UTC)    = 2026-10-28
time (UTC)    = 05:56:28
awake uptime  = 1w1d18h34m29.351s
cpu_process   = 1.369s
```

| Clock | Measures | Use for |
|---|---|---|
| `.real` | wall-clock time since the Unix epoch | timestamps, dates, logs |
| `.awake` | time since an arbitrary point, excluding suspend | **benchmarks, timeouts** |
| `.boot` | same but including time suspended | uptime |
| `.cpu_process` | CPU time this process has burned | profiling |
| `.cpu_thread` | CPU time this thread has burned | per-thread profiling |

**Never benchmark with `.real`.** It jumps when NTP corrects it or an admin
changes the clock, so a duration can come out negative. `.awake` only ever moves
forward.

`std.time.epoch` is the only date arithmetic in the stdlib: it decomposes Unix
seconds into UTC year/month/day. There is **no** timezone database, no date
parsing, no formatting — those need a library.

## Sleeping, deadlines, and unit constants

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const io = init.io;

    std.debug.print("ns_per_us  = {d}\n", .{std.time.ns_per_us});
    std.debug.print("ns_per_ms  = {d}\n", .{std.time.ns_per_ms});
    std.debug.print("ns_per_s   = {d}\n", .{std.time.ns_per_s});
    std.debug.print("ms_per_s   = {d}\n", .{std.time.ms_per_s});
    std.debug.print("s_per_min  = {d}\n", .{std.time.s_per_min});
    std.debug.print("s_per_hour = {d}\n", .{std.time.s_per_hour});
    std.debug.print("s_per_day  = {d}\n", .{std.time.s_per_day});

    const before: std.Io.Clock.Timestamp = .now(io, .awake);
    const nap: std.Io.Clock.Duration = .{ .raw = .fromMilliseconds(50), .clock = .awake };
    try nap.sleep(io);
    const after: std.Io.Clock.Timestamp = .now(io, .awake);
    std.debug.print("slept 50ms -> measured {f}\n", .{before.durationTo(after).raw});

    const deadline: std.Io.Clock.Timestamp = .fromNow(io, .{ .raw = .fromMilliseconds(30), .clock = .awake });
    try deadline.wait(io);
    std.debug.print("waited until a deadline 30ms out\n", .{});
}
```

```
ns_per_us  = 1000
ns_per_ms  = 1000000
ns_per_s   = 1000000000
ms_per_s   = 1000
s_per_min  = 60
s_per_hour = 3600
s_per_day  = 86400
slept 50ms -> measured 55.019ms
waited until a deadline 30ms out
```

**A 50 ms sleep took 55 ms.** Sleeping guarantees a *minimum*, never an exact
duration — the OS reschedules you when it gets around to it. Never build a clock
out of accumulated sleeps; compute an absolute deadline and wait for it, which is
what `fromNow` + `wait` does.

Note `sleep` and `wait` can return `error.Canceled`, which is how module 12's
cancellation works.

## Random numbers

A PRNG is a value you own. `.init(seed)` creates it, `.random()` hands you the
`std.Random` interface with all the methods.

```zig
const std = @import("std");

const Suit = enum { hearts, spades, clubs, diamonds };

pub fn main() void {
    var prng: std.Random.DefaultPrng = .init(42);
    const r = prng.random();

    std.debug.print("int u8            = {d}\n", .{r.int(u8)});
    std.debug.print("int i32           = {d}\n", .{r.int(i32)});
    std.debug.print("intRangeAtMost 1-6= {d} {d} {d} {d} {d}\n", .{
        r.intRangeAtMost(u8, 1, 6), r.intRangeAtMost(u8, 1, 6),
        r.intRangeAtMost(u8, 1, 6), r.intRangeAtMost(u8, 1, 6),
        r.intRangeAtMost(u8, 1, 6),
    });
    std.debug.print("intRangeLessThan  = {d}\n", .{r.intRangeLessThan(u8, 0, 10)});
    std.debug.print("uintLessThan 100  = {d}\n", .{r.uintLessThan(u32, 100)});
    std.debug.print("uintAtMost 100    = {d}\n", .{r.uintAtMost(u32, 100)});
    std.debug.print("float f64 [0,1)   = {d:.6}\n", .{r.float(f64)});
    std.debug.print("float f32 [0,1)   = {d:.6}\n", .{r.float(f32)});
    std.debug.print("floatNorm (gauss) = {d:.6}\n", .{r.floatNorm(f64)});
    std.debug.print("floatExp          = {d:.6}\n", .{r.floatExp(f64)});
    std.debug.print("boolean           = {} {} {}\n", .{ r.boolean(), r.boolean(), r.boolean() });
    std.debug.print("enumValue         = {t}\n", .{r.enumValue(Suit)});
    std.debug.print("weightedIndex     = {d}\n", .{r.weightedIndex(u32, &.{ 1, 1, 98 })});

    var buf: [8]u8 = undefined;
    r.bytes(&buf);
    std.debug.print("bytes             = {any}\n", .{buf});

    var deck = [_]u8{ 1, 2, 3, 4, 5, 6, 7, 8 };
    r.shuffle(u8, &deck);
    std.debug.print("shuffle           = {any}\n", .{deck});
}
```

```
int u8            = 159
int i32           = 1466906513
intRangeAtMost 1-6= 4 5 3 3 3
intRangeLessThan  = 2
uintLessThan 100  = 93
uintAtMost 100    = 10
float f64 [0,1)   = 0.936840
float f32 [0,1)   = 0.684419
floatNorm (gauss) = 0.494782
floatExp          = 0.068277
boolean           = true false false
enumValue         = hearts
weightedIndex     = 2
bytes             = { 123, 29, 72, 226, 137, 203, 132, 119 }
shuffle           = { 2, 8, 6, 3, 1, 4, 7, 5 }
```

**Those numbers are reproducible.** Run that program a hundred times and seed 42
gives the identical sequence every time, which is exactly what you want in a
test.

Range functions, since the naming is easy to misread:

| Call | Range |
|---|---|
| `r.intRangeAtMost(u8, 1, 6)` | 1–6 **inclusive** — a die |
| `r.intRangeLessThan(u8, 0, 10)` | 0–9 |
| `r.uintLessThan(u32, n)` | `0..n-1` — a valid index into a length-`n` slice |
| `r.uintAtMost(u32, n)` | `0..n` |
| `r.float(f64)` | `[0, 1)` |

The `…Biased` variants (`uintLessThanBiased`) are faster but slightly skewed for
ranges that don't divide evenly into the generator's output. Use the unbiased
default unless you've measured that it matters.

## Seeding

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const io = init.io;

    var fixed: std.Random.DefaultPrng = .init(42);
    var fixed2: std.Random.DefaultPrng = .init(42);
    var other: std.Random.DefaultPrng = .init(43);
    std.debug.print("seed 42  -> {d}\n", .{fixed.random().int(u16)});
    std.debug.print("seed 42  -> {d} (identical)\n", .{fixed2.random().int(u16)});
    std.debug.print("seed 43  -> {d} (different)\n", .{other.random().int(u16)});

    var seed: u64 = undefined;
    io.random(std.mem.asBytes(&seed));
    var varying: std.Random.DefaultPrng = .init(seed);
    std.debug.print("io-seeded -> {d} (changes every run)\n", .{varying.random().int(u16)});

    var secure: [4]u8 = undefined;
    try io.randomSecure(&secure);
    std.debug.print("randomSecure bytes -> {any}\n", .{secure});

    var pcg: std.Random.Pcg = .init(42);
    var sfc: std.Random.Sfc64 = .init(42);
    var xo: std.Random.Xoshiro256 = .init(42);
    std.debug.print("Pcg(42)        = {d}\n", .{pcg.random().int(u16)});
    std.debug.print("Sfc64(42)      = {d}\n", .{sfc.random().int(u16)});
    std.debug.print("Xoshiro256(42) = {d}\n", .{xo.random().int(u16)});
}
```

```
seed 42  -> 26783
seed 42  -> 26783 (identical)
seed 43  -> 29821 (different)
io-seeded -> 40392 (changes every run)
randomSecure bytes -> { 178, 193, 206, 147 }
Pcg(42)        = 57719
Sfc64(42)      = 9487
Xoshiro256(42) = 26783
```

Only the two `io.random*` lines change between runs; everything seeded with a
constant is fixed. `Xoshiro256(42)` matches `DefaultPrng(42)` because
`DefaultPrng` **is** `Xoshiro256`.

> **`std.crypto.random` is gone in 0.17.** Entropy comes from `io` now:
> `io.random(buf)` for a seed, `try io.randomSecure(buf)` when the bytes
> themselves must be unpredictable. `randomSecure` returns an error union
> because it can fail if the OS entropy source is unavailable.

| Need | Use |
|---|---|
| reproducible tests, simulations | `DefaultPrng.init(fixed_seed)` |
| different every run | `io.random(asBytes(&seed))`, then `DefaultPrng.init(seed)` |
| **tokens, keys, passwords, nonces** | `try io.randomSecure(&buf)` — never a PRNG |
| fastest general-purpose | `DefaultPrng` (Xoshiro256) |

**Never generate a secret with `DefaultPrng`.** Observe a few outputs and its
entire future output is predictable — it is designed for speed, not secrecy.

## Timing something real

```zig
const std = @import("std");

fn linear(items: []const u32, needle: u32) bool {
    return std.mem.indexOfScalar(u32, items, needle) != null;
}

fn binary(items: []const u32, needle: u32) bool {
    return std.sort.binarySearch(u32, items, needle, struct {
        fn ord(n: u32, item: u32) std.math.Order {
            return std.math.order(n, item);
        }
    }.ord) != null;
}

fn bench(io: std.Io, label: []const u8, items: []const u32, f: *const fn ([]const u32, u32) bool) void {
    const start: std.Io.Clock.Timestamp = .now(io, .awake);
    var hits: usize = 0;
    for (0..2000) |i| {
        if (f(items, @intCast(i * 7 % 50_000))) hits += 1;
    }
    const d = start.durationTo(.now(io, .awake));
    std.debug.print("{s:<8} {f:>10}  hits={d}\n", .{ label, d.raw, hits });
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const gpa = init.gpa;

    const items = try gpa.alloc(u32, 50_000);
    defer gpa.free(items);
    for (items, 0..) |*p, i| p.* = @intCast(i);

    bench(io, "linear", items, linear);
    bench(io, "binary", items, binary);
}
```

```
linear   9.361ms  hits=2000
binary   234.583us  hits=2000
```

2000 lookups in a sorted 50,000-element slice: **~40x faster** with a binary
search, which is lesson 7.4's claim turned into a number. Both found all 2000
needles, so the comparison is honest.

Three habits for benchmarks that mean something:

1. **Use the result.** `hits` is printed, so the optimizer can't delete the loop.
   `std.mem.doNotOptimizeAway(x)` is the explicit version.
2. **Measure with `.awake`**, never `.real`.
3. **Run it in ReleaseFast** before believing a ratio —
   `zig run -O ReleaseFast bench.zig`. Debug builds carry bounds checks and
   overflow checks that distort everything.

---

That's the standard library tour. Next up is real I/O: files, directories and
CLI programs.

Next: [08 — Files & I/O](../08-files-io/)
