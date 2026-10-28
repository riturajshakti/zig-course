# 15.7 — Measuring Performance

**`std.time.Timer` is gone in 0.17.** Time is I/O now, so a benchmark needs
`io`, which means `pub fn main(init: std.process.Init)`.

## Timing

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const io = init.io;

    const t0 = std.Io.Clock.Timestamp.now(io, .awake);
    var acc: u64 = 0;
    for (0..5_000_000) |i| acc +%= i *% 2654435761;
    const t1 = std.Io.Clock.Timestamp.now(io, .awake);
    std.mem.doNotOptimizeAway(acc);

    const d = t0.durationTo(t1);
    std.debug.print("ns = {d}\n", .{@as(i64, @intCast(d.raw.toNanoseconds()))});
    std.debug.print("us = {d}\n", .{d.raw.toMicroseconds()});
    std.debug.print("ms = {d}\n", .{d.raw.toMilliseconds()});
    std.debug.print("{{f}} = {f}\n", .{d.raw});

    const real = std.Io.Clock.Timestamp.now(io, .real);
    std.debug.print("\nwall clock unix seconds = {d}\n", .{real.raw.toSeconds()});
    std.debug.print("clocks: .awake (monotonic, excludes sleep) / .real (wall) / .boot\n", .{});
}
```

```
ns = 3822833
us = 3822
ms = 3
{f} = 3.822ms

wall clock unix seconds = 1793171355
clocks: .awake (monotonic, excludes sleep) / .real (wall) / .boot
```

| What you remember | 0.17.0 |
|---|---|
| `var t = try std.time.Timer.start();` | `const t0 = std.Io.Clock.Timestamp.now(io, .awake);` |
| `t.read()` | `t0.durationTo(t1)` |
| `Clock.monotonic` | **`Clock.awake`** |
| `std.time.timestamp()` | `…now(io, .real).raw.toSeconds()` |
| `std.fmt.fmtDuration(ns)` | `{f}` on the `Duration`'s `.raw` |

Three details that will cost you a compile:

- **`Timestamp.now(io, .awake)` is NOT fallible.** `try` on it is a compile
  error. Reading a clock can't fail.
- **`Clock.Duration` is tagged** — you need `.raw` to reach `toMicroseconds()`,
  `toNanoseconds()` and `{f}`.
- **`io` comes first**, but the *clock* comes first on `Clock.now(clock, io)`.
  Both spellings exist; `Timestamp.now(io, clock)` is the one to remember.

| Clock | Measures | Use for |
|---|---|---|
| **`.awake`** | monotonic, **excludes** system suspend | **benchmarks** — the default |
| `.boot` | monotonic, includes suspend | uptime, long timeouts |
| `.real` | wall clock, can jump backwards | timestamps in logs, never durations |
| `.cpu_process` | CPU time used by this process | separating CPU from I/O wait |
| `.cpu_thread` | CPU time used by this thread | per-worker accounting |

**Never measure a duration with `.real`.** NTP can step it backwards and your
duration goes negative.

## Benchmark hygiene

```zig
const std = @import("std");

const Stats = struct {
    min: u64,
    max: u64,
    mean: u64,
    median: u64,

    fn spread(s: Stats) f64 {
        return @as(f64, @floatFromInt(s.max)) / @as(f64, @floatFromInt(s.min));
    }
};

fn measure(io: std.Io, warmup: usize, reps: usize, samples: []u64, comptime f: fn (u64) u64) Stats {
    var sink: u64 = 0;
    for (0..warmup) |i| sink +%= f(i);
    std.mem.doNotOptimizeAway(sink);

    for (samples[0..reps], 0..) |*s, i| {
        const t0 = std.Io.Clock.Timestamp.now(io, .awake);
        const r = f(i);
        const t1 = std.Io.Clock.Timestamp.now(io, .awake);
        std.mem.doNotOptimizeAway(r);
        s.* = @intCast(t0.durationTo(t1).raw.toMicroseconds());
    }

    const used = samples[0..reps];
    std.mem.sort(u64, used, {}, std.sort.asc(u64));
    var total: u64 = 0;
    for (used) |s| total += s;
    return .{
        .min = used[0],
        .max = used[used.len - 1],
        .mean = total / used.len,
        .median = used[used.len / 2],
    };
}

var table: []u64 = undefined;

fn work(seed: u64) u64 {
    var h: u64 = seed;
    var idx: usize = seed % table.len;
    for (0..2_000_000) |_| {
        h = (h ^ table[idx]) *% 0x9E3779B97F4A7C15;
        idx = h % table.len;
    }
    return h;
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const gpa = init.gpa;
    var samples: [21]u64 = undefined;

    table = try gpa.alloc(u64, 8 * 1024 * 1024);
    defer gpa.free(table);
    var rng = std.Random.DefaultPrng.init(1);
    for (table) |*t| t.* = rng.random().int(u64);
    std.debug.print("random-access over a {d} MB table ({d} values)\n\n", .{ table.len * 8 / 1048576, table.len });

    const cold = measure(io, 0, 21, &samples, work);
    std.debug.print("no warmup, 21 reps:\n", .{});
    std.debug.print("  min={d} median={d} mean={d} max={d}  max/min={d:.2}x\n", .{ cold.min, cold.median, cold.mean, cold.max, cold.spread() });

    const warm = measure(io, 5, 21, &samples, work);
    std.debug.print("\n5 warmup, 21 reps:\n", .{});
    std.debug.print("  min={d} median={d} mean={d} max={d}  max/min={d:.2}x\n", .{ warm.min, warm.median, warm.mean, warm.max, warm.spread() });

    const one = measure(io, 5, 1, &samples, work);
    std.debug.print("\nsingle run (what a naive benchmark reports): {d} us\n", .{one.min});
    std.debug.print("vs the median of 21:                         {d} us\n", .{warm.median});
}
```

**`zig run -Ofast`**, four consecutive runs on an idle machine:

```
random-access over a 64 MB table (8388608 values)

no warmup, 21 reps:
  min=8123 median=8831 mean=8757 max=9192  max/min=1.13x

5 warmup, 21 reps:
  min=7964 median=8748 mean=8630 max=9056  max/min=1.14x

single run (what a naive benchmark reports): 9000 us
vs the median of 21:                         8748 us
```

| Run | min | median | mean | max | max/min |
|---|---|---|---|---|---|
| 1 | 7993 | 8721 | 8613 | 9110 | 1.14x |
| 2 | 7987 | 8712 | 8627 | 9042 | 1.13x |
| 3 | 8035 | 8771 | **9370** | **21015** | **2.62x** |
| 4 | 7964 | 8748 | 8630 | 9056 | 1.14x |

**Run 3 is the lesson.** One sample out of 21 took 21 ms instead of 8.7 — a
scheduler hiccup, some other process, a frequency change. It dragged the **mean
from 8630 to 9370 (+9%)** while the **median moved 0.3%**. If you report means
from small samples, you are reporting your machine's mood.

| Rule | Why |
|---|---|
| **report min or median, never the mean** | one outlier destroys a mean; the min is the least-disturbed run |
| **`std.mem.doNotOptimizeAway(result)`** | without it the optimizer deletes your benchmark |
| **make the input depend on the iteration** | otherwise the loop is invariant and gets hoisted out entirely |
| **state the mode** | Debug numbers are meaningless; always say `-Ofast` or `-Osafe` |
| **run the whole thing 2+ times** | catches a machine that was busy |
| **report the spread, not just the number** | `max/min=1.14x` tells the reader how much to trust it |
| warmup | matters when there are caches to fill, JIT-like lazy work, or page faults; here it changed ~1% |

**The most common way a Zig microbenchmark lies:** in an earlier draft of 15.4,
ReleaseFast noticed the benchmarked function ignored its argument, hoisted the
whole loop out of the repeat loop, and reported **0 us** for all four variants.
A benchmark reporting 0 is not fast; it is not running.

## Release modes, measured

Four modes, one `hello world`, install path, unstripped and `-fstrip`:

```sh
zig build-exe hello.zig -O ReleaseSmall -fstrip
```

| Mode | Size | Stripped | vs Debug |
|---|---|---|---|
| **Debug** | 2,001,512 | 845,656 | — |
| **ReleaseSafe** | 468,912 | 206,880 | 4.3x smaller |
| **ReleaseFast** | 395,072 | 190,192 | 5.1x smaller |
| **ReleaseSmall** | 165,088 | 165,088 | **12.1x smaller** |

(ReleaseSmall already omits the debug info, which is why `-fstrip` changes
nothing.) A `ReleaseSmall` + `-fstrip` + static musl Linux binary lands around
**134 KB** — a complete, dependency-free service binary.

### Speed is *not* ordered the way you expect

```zig
const std = @import("std");

fn sumIndexed(data: []const u32, idx: []const u32) u64 {
    var total: u64 = 0;
    for (idx) |i| total += data[i % data.len];
    return total;
}

fn sumIndexedUnsafe(data: []const u32, idx: []const u32) u64 {
    @setRuntimeSafety(false);
    var total: u64 = 0;
    for (idx) |i| total += data[i % data.len];
    return total;
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const gpa = init.gpa;
    const data = try gpa.alloc(u32, 4096);
    defer gpa.free(data);
    const idx = try gpa.alloc(u32, 30_000_000);
    defer gpa.free(idx);
    var rng = std.Random.DefaultPrng.init(11);
    for (data) |*d| d.* = rng.random().int(u16);
    for (idx) |*d| d.* = rng.random().int(u32);

    var a: u64 = std.math.maxInt(u64);
    var b: u64 = std.math.maxInt(u64);
    for (0..5) |_| {
        var t0 = std.Io.Clock.Timestamp.now(io, .awake);
        var r = sumIndexed(data, idx);
        var t1 = std.Io.Clock.Timestamp.now(io, .awake);
        std.mem.doNotOptimizeAway(r);
        a = @min(a, @as(u64, @intCast(t0.durationTo(t1).raw.toMicroseconds())));

        t0 = std.Io.Clock.Timestamp.now(io, .awake);
        r = sumIndexedUnsafe(data, idx);
        t1 = std.Io.Clock.Timestamp.now(io, .awake);
        std.mem.doNotOptimizeAway(r);
        b = @min(b, @as(u64, @intCast(t0.durationTo(t1).raw.toMicroseconds())));
    }
    std.debug.print("{t:<6} safety on: {d:>6} us   @setRuntimeSafety(false): {d:>6} us\n", .{ @import("builtin").mode, a, b });
}
```

Run as `zig run -O Debug h5.zig`, `-O ReleaseSafe`, etc. Best of 5, three runs:

```
debug  safety on:  92305 us   @setRuntimeSafety(false):  37987 us
safe   safety on:   8021 us   @setRuntimeSafety(false):   4304 us
fast   safety on:   5325 us   @setRuntimeSafety(false):   5370 us
small  safety on:  16391 us   @setRuntimeSafety(false):  16403 us
```

| Mode | With safety | `@setRuntimeSafety(false)` | vs ReleaseFast |
|---|---|---|---|
| Debug | 92,305 / 93,578 / 92,326 | 37,987 / 38,083 / 37,832 | **~18x slower** |
| ReleaseSafe | 8,021 / 8,199 / 8,996 | 4,304 / 4,331 / 5,437 | ~1.6x slower |
| ReleaseFast | 5,325 / 4,528 / 5,308 | 5,370 / 4,555 / 5,382 | — |
| ReleaseSmall | 16,391 / 16,386 | 16,403 / 16,366 | **~3.3x slower** |

One machine, one workload (30M bounds-checked random indexes) — your ratios
will differ. Four conclusions that do generalize:

1. **Debug is ~18x slower than ReleaseFast here.** Never quote a Debug number
   and never ship Debug. This alone explains most "Zig is slow" claims online.
2. **ReleaseSafe costs ~1.6x on a bounds-check-heavy loop** — and nothing at all
   on loops without checks (a pure-arithmetic variant of this benchmark measured
   40,313 us safe vs 41,446 fast: identical within noise). **The cost of safety
   depends entirely on your code**, so measure your code.
3. **ReleaseSmall is ~3.3x slower than ReleaseFast.** It optimizes for size and
   will happily not inline or not unroll. It is not "ReleaseFast but smaller."
4. **`@setRuntimeSafety(false)` works per function.** In ReleaseSafe it
   recovered the full 1.6x on this one loop while the rest of the program keeps
   its checks — far better than switching the whole program to ReleaseFast.

### Behavior changes, not just speed

```zig
const std = @import("std");

pub fn main() !void {
    const mode = @import("builtin").mode;
    std.debug.print("mode = {t}\n", .{mode});
    std.debug.print("runtime_safety = {}\n", .{std.debug.runtime_safety});

    var dbg: std.heap.DebugAllocator(.{}) = .init;
    const gpa = dbg.allocator();
    const leaked = try gpa.alloc(u8, 32);
    leaked[0] = 1;
    std.debug.print("DebugAllocator deinit on a REAL 32-byte leak: {t}\n", .{dbg.deinit()});

    var a: u8 = 250;
    a +%= 10;
    std.debug.print("250 +%= 10 (wrapping) = {d}\n", .{a});
}
```

`zig run -O Debug` then `-O ReleaseFast`:

```
--- Debug ---
mode = debug
runtime_safety = true
DebugAllocator deinit on a REAL 32-byte leak: leak
250 +%= 10 (wrapping) = 4

--- ReleaseFast ---
mode = fast
runtime_safety = false
DebugAllocator deinit on a REAL 32-byte leak: ok
250 +%= 10 (wrapping) = 4
```

**The same real 32-byte leak reports `leak` in Debug and `ok` in ReleaseFast.**
`DebugAllocator`'s `safety` option defaults to `std.debug.runtime_safety`, which
is `false` in both release-fast and release-small. Your leak checking silently
disappears in the build you ship.

| Changes between modes | Debug / ReleaseSafe | ReleaseFast / ReleaseSmall |
|---|---|---|
| integer overflow | **panics** | **silently wraps** |
| out-of-bounds index | **panics** | undefined behavior |
| `unreachable` reached | panics | UB — may **hang**, not crash |
| wrong field of a bare `union` | **panics** | returns garbage |
| `@sizeOf(union{int:u32,float:f32})` | **8** (hidden safety tag) | **4** |
| `DebugAllocator` leak detection | works | **silently reports `ok`** |
| `std.testing.allocator` leak detection | works | **works** (all four modes) |

**`std.testing.allocator` keeps leak detection in all four modes**, which is why
your tests are the right place to catch leaks — not your production build.

| Mode | Use for |
|---|---|
| **Debug** | development, `zig build test` — leak reports with line numbers |
| **ReleaseSafe** | **the default for shipping a backend service.** Keeps bounds and overflow checks; a bug is a crash with a stack trace, not a silent corruption |
| **ReleaseFast** | a measured hot path where you've proven safety costs you, and you have fuzzing/tests to cover what the checks were covering |
| **ReleaseSmall** | embedded, container images, cold-start-sensitive serverless — accept ~3x slower |

**Ship ReleaseSafe unless you have a measurement that says otherwise.** That is
the answer an interviewer wants: a 1.6x worst case for keeping memory-safety
checks on in a networked service is a good trade, and you can disable them per
function where the profile says so.

## Reading `zig build --summary`

```sh
zig build --summary all
```

```
Build Summary: 3/3 steps succeeded
install success
+- install svc success
   +- compile exe svc debug native success 1s MaxRSS:245M
```

```sh
zig build test --summary all
```

```
Build Summary: 3/3 steps succeeded; 1/1 tests passed
test success
+- run test 1 pass (1 total) 421ms MaxRSS:2M
   +- compile test debug native success 1s MaxRSS:257M
```

```sh
zig build --summary all -Doptimize=ReleaseFast
```

```
Build Summary: 3/3 steps succeeded
install success
+- install svc success
   +- compile exe svc fast native success 4s MaxRSS:310M
```

| Column | Means |
|---|---|
| `3/3 steps succeeded` | the dependency graph; a `cached` step did no work |
| `1/1 tests passed` | appears only when a test step ran |
| `debug native` / `fast native` | **mode and target**, so you can't misread a Debug number as a release one |
| `1s` / `4s` | wall time **for that step** — this is where you find the slow part of your build |
| `MaxRSS:245M` | peak compiler memory; a comptime explosion shows up here first |

**ReleaseFast took 4s where Debug took 1s** — that's LLVM optimization, and the
reason you develop in Debug and only build release in CI. `MaxRSS:310M` for a
hello-world is the compiler, not your program; if it reaches multiple GB, you
have runaway comptime or generic instantiation (module 6.6).

Useful flags:

| Flag | Does |
|---|---|
| `--summary all` | show every step, including cached ones |
| `--summary new` | only steps that actually ran |
| `--summary failures` | CI default — quiet unless something breaks |
| `-freference-trace` | full instantiation chain on a comptime error |
| `--prominent-compile-errors` | easier-to-spot errors in a noisy log |
| `-fstrip` | drop debug info (see the size table) |

## What actually matters in a service

Honest ordering of where backend latency goes, biggest first:

| Rank | Cost | Typical magnitude | Fix |
|---|---|---|---|
| 1 | a network round trip | **1–100 ms** | batch, cache, or don't make it |
| 2 | a disk read that missed the page cache | ~0.1–10 ms | `getrusage().majflt`, index better, cache |
| 3 | an N+1 query pattern | n × rank 1 | join, or one round trip |
| 4 | a `write(2)` per log line | ~1–5 us each, ×10⁵ | **buffer and `flush()`** — 0.17 makes you do this |
| 5 | allocating per field while parsing | ~4x the parse (15.5) | zero-copy, or an arena |
| 6 | cache misses on a hot loop | ~5.8x the loop (15.4) | shrink the row, then consider SoA |
| 7 | bounds checks | ~1.6x a check-heavy loop | `@setRuntimeSafety(false)` on that function |
| 8 | everything you'd micro-optimize first | < 1.1x | leave it alone |

**Ranks 1–4 are about syscalls and round trips; 5–7 are about memory.** Nothing
in that list is "my loop isn't unrolled." Zig gives you excellent tools for
ranks 5–7, and that is genuinely where Zig beats Go and Node — but only after
you've fixed 1–4, which are language-independent.

The thing to say in an interview: **"I'd measure before changing anything, and
I'd expect the answer to be I/O."** Then mention `strace -c`, a p99 rather than
a mean, and `getrusage` for major faults — because those are what you'd actually
reach for.

| Measure | Not |
|---|---|
| **p99 / p99.9 latency** | the mean |
| requests completed under load | a microbenchmark of one function |
| RSS over hours | RSS at startup |
| syscalls per request (`strace -c`) | instructions per loop |
| allocations per request | allocator speed |

---

Next: [15.8 — Zig vs C, Go & Rust](./08-zig-vs-others.md)
