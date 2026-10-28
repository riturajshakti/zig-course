# 9.5 — Release Modes

Four modes. The difference is **not only speed** — they change what your program
*does*. The same source either panics on an integer overflow or silently wraps,
depending on a flag you pass to the compiler.

| `-O` | `builtin.mode` | `runtime_safety` | Optimised | Ships |
|---|---|---|---|---|
| `-Odebug` (default) | `.debug` | `true` | no | never |
| `-Osafe` | `.safe` | `true` | yes | **servers, CLIs, anything handling input** |
| `-Ofast` | `.fast` | `false` | yes | hot numeric paths you have benchmarked |
| `-Osmall` | `.small` | `false` | for size | embedded, wasm |

> **The names are lowercase in 0.17.** `-Odebug`, `-Osafe`, `-Ofast`, `-Osmall`.
> The old `-ODebug` / `-OReleaseSafe` / `-OReleaseFast` / `-OReleaseSmall`
> spellings are still accepted, but `builtin.mode` now prints `debug`, `safe`,
> `fast`, `small`, and `std.Options` fields switch on those tags.

```zig
const std = @import("std");
const builtin = @import("builtin");

pub fn main() void {
    std.debug.print("mode = {t}\n", .{builtin.mode});
    std.debug.print("runtime_safety = {}\n", .{std.debug.runtime_safety});
}
```

```sh
for m in debug safe fast small; do zig run -O$m mode.zig; done
```

```
mode = debug
runtime_safety = true
mode = safe
runtime_safety = true
mode = fast
runtime_safety = false
mode = small
runtime_safety = false
```

**The real split is `runtime_safety`, and it falls between `safe` and `fast`.**
Everything below follows from that one boolean.

## 1. Integer overflow: panic vs wrap

```zig
const std = @import("std");
const builtin = @import("builtin");

fn bump(n: u8, by: u8) u8 {
    return n + by;
}

pub fn main() void {
    std.debug.print("mode = {t}\n", .{builtin.mode});
    std.debug.print("250 + 10 = {d}\n", .{bump(250, 10)});
}
```

```sh
for m in debug safe fast small; do zig run -O$m ov.zig; done
```

**The first two panics are deliberate.** Real output:

```
=== -Odebug ===
mode = debug
thread 16631020 panic: integer overflow
/path/ov.zig:5:14: 0x10258dd37 in bump (ov)
    return n + by;
             ^
/path/ov.zig:10:47: 0x10258d1ab in main (ov)
    std.debug.print("250 + 10 = {d}\n", .{bump(250, 10)});
                                              ^
exit=134

=== -Osafe ===
mode = safe
thread 16631443 panic: integer overflow
/opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/Io/Threaded.zig:2404:11: 0x102d759ff in main (ov)
    thread.cancel_protection = new;
          ^
???:?:?: 0x180407dff in start (/usr/lib/dyld)
exit=134

=== -Ofast ===
mode = fast
250 + 10 = 4
exit=0

=== -Osmall ===
mode = small
250 + 10 = 4
exit=0
```

`250 + 10 = 4`. One compiler flag turned a hard crash into a wrong answer and a
zero exit code.

Note the `-Osafe` trace: it **blames `std/Io/Threaded.zig`**, a file your
program never mentions. Inlining destroyed the frame for `bump`, and the panic
got attributed to a neighbouring inlined function. Safety is intact — the
location is not. **Reproduce a release panic in `-Odebug` before you trust the
trace.**

If you *want* wrapping, say so explicitly with `+%`, `-%`, `*%` — then it wraps
in all four modes and the behaviour is in the source instead of the build
command.

## 2. Every other safety check, same story

```zig
const std = @import("std");
const builtin = @import("builtin");

fn at(items: []const u32, i: usize) u32 {
    return items[i];
}

pub fn main() void {
    const data = [_]u32{ 10, 20, 30 };
    std.debug.print("mode = {t}\n", .{builtin.mode});
    std.debug.print("data[5] = {d}\n", .{at(&data, 5)});
}
```

```sh
for m in debug safe fast small; do zig run -O$m oob.zig; done
```

**Deliberate**, trimmed:

```
=== -Odebug ===
mode = debug
thread 16641465 panic: index out of bounds: index 5, len 3
/path/oob.zig:5:17: 0x1006e2423 in at (oob)
    return items[i];
                ^
=== -Osafe ===
mode = safe
thread 16641655 panic: index out of bounds: index 5, len 3
/path/oob.zig:5:17: 0x104bada0b in at (oob)
    return items[i];
                ^
=== -Ofast ===
mode = fast
data[5] = 0
exit=0
=== -Osmall ===
mode = small
data[5] = 0
exit=0
```

`data[5] = 0` is a read of whatever happened to sit past the array. It could as
easily have been a segfault, or someone else's data.

| Check | Debug / ReleaseSafe | ReleaseFast / ReleaseSmall |
|---|---|---|
| integer overflow | panic | **wraps** |
| array / slice index | panic | **reads out of bounds** |
| `x.?` on null | panic | **UB** |
| `catch unreachable` on an error | panic | **UB** |
| `unreachable` | panic | **UB** |
| lossy `@intCast` | panic | **truncates** |
| division by zero | panic | **UB** |
| wrong field on a bare `union` | panic | **garbage** |
| `std.debug.assert` | checked | **compiled out** |

## 3. `unreachable`: a panic, or a licence to miscompile

```zig
const std = @import("std");
const builtin = @import("builtin");

fn grade(score: u32) []const u8 {
    return switch (score) {
        90...100 => "A",
        80...89 => "B",
        0...79 => "C",
        else => unreachable,
    };
}

pub fn main() void {
    std.debug.print("mode = {t}\n", .{builtin.mode});
    std.debug.print("95 -> {s}\n", .{grade(95)});
    std.debug.print("500 -> {s}\n", .{grade(500)});
    std.debug.print("survived\n", .{});
}
```

```sh
for m in debug safe fast small; do zig run -O$m un.zig; done
```

**Deliberate UB. Never run this unguarded** — reaching `unreachable` in
`-Ofast` is allowed to do anything, including hang. On this machine:

```
=== -Odebug ===
mode = debug
95 -> A
thread 16632450 panic: reached unreachable code
/path/un.zig:9:17: 0x104c4a077 in grade (un)
        else => unreachable,
                ^
/path/un.zig:16:44: 0x104c491cf in main (un)
    std.debug.print("500 -> {s}\n", .{grade(500)});
                                           ^
=== -Osafe ===
mode = safe
95 -> A
thread 16632865 panic: reached unreachable code
/opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/Io/Threaded.zig:2402:26: 0x100c85ae3 in callMainWithArgs (un)
    const thread = Thread.current orelse return .unblocked;
                         ^
=== -Ofast ===
mode = fast
95 -> A
500 -> C
survived
=== -Osmall ===
mode = small
95 -> A
500 -> C
survived
```

`grade(500)` returned `"C"` and the program carried on. `unreachable` is a
**promise to the optimiser**, and in `-Ofast` the optimiser took it: told score
500 can't exist, it folded the `else` into the nearest arm. Same misattributed
`Threaded.zig` trace in `-Osafe`.

**`unreachable` means "I have proved this cannot happen", not "this shouldn't
happen".** For input you don't control, return an error.

## 4. Leak detection silently disappears

The nastiest one, because nothing tells you.

```zig
const std = @import("std");
const builtin = @import("builtin");

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    const gpa = gpa_s.allocator();

    const leaked = try gpa.alloc(u8, 1024);
    _ = leaked;

    std.debug.print("mode = {t}\n", .{builtin.mode});
    std.debug.print("deinit status = {t}\n", .{gpa_s.deinit()});
}
```

```sh
for m in debug safe fast small; do zig run -O$m lk.zig; done
```

**The leak is deliberate and identical in all four runs:**

```
=== -Odebug ===
mode = debug
error(DebugAllocator): memory address 0x1029e0000 leaked: 
deinit status = leak

=== -Osafe ===
mode = safe
error(DebugAllocator): memory address 0x102a00000 leaked: 
(empty stack trace)
deinit status = leak

=== -Ofast ===
mode = fast
deinit status = ok

=== -Osmall ===
mode = small
deinit status = ok
```

**`deinit()` returned `.ok` on a 1 KiB leak.** `DebugAllocator`'s `safety`
option defaults to `std.debug.runtime_safety`, so in `-Ofast`/`-Osmall` the
bookkeeping is never recorded and there is nothing to report. No warning, no
compile error — the leak checker just evaporates.

Two consequences:

- `if (gpa_s.deinit() == .leak) @panic("leaked")` is **not a production leak
  check**. It is a development-only assertion.
- A memory bug that reproduces only in release cannot be found by switching to
  `DebugAllocator` in release. You need an external tool (`leaks`, `heaptrack`)
  or a mode-independent allocator.

Force it on regardless of mode:

```zig
const std = @import("std");
const builtin = @import("builtin");

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{ .safety = true }) = .init;
    const gpa = gpa_s.allocator();
    const leaked = try gpa.alloc(u8, 1024);
    _ = leaked;
    std.debug.print("mode = {t}\n", .{builtin.mode});
    std.debug.print("deinit status = {t}\n", .{gpa_s.deinit()});
}
```

```sh
zig run -Ofast so.zig
```

**Deliberate leak — now reported even in `-Ofast`:**

```
mode = fast
error(DebugAllocator): memory address 0x100560000 leaked: 
(empty stack trace)

deinit status = leak
```

Good news from 9.3: **`std.testing.allocator` is not affected.** It catches
leaks and fails the test in all four modes; only the trace quality degrades.

## 5. Bare unions change size and silently return garbage

```zig
const std = @import("std");
const builtin = @import("builtin");

const Bare = union { int: u32, float: f32 };
const Tagged = union(enum) { int: u32, float: f32 };

pub fn main() void {
    var x: i32 = undefined;
    _ = &x;
    std.debug.print("mode = {t}\n", .{builtin.mode});
    std.debug.print("@sizeOf(Bare)   = {d}\n", .{@sizeOf(Bare)});
    std.debug.print("@sizeOf(Tagged) = {d}\n", .{@sizeOf(Tagged)});
    std.debug.print("undefined i32   = {d}\n", .{x});
}
```

```sh
for m in debug safe fast small; do zig run -O$m sz.zig; done
```

```
mode = debug
@sizeOf(Bare)   = 8
@sizeOf(Tagged) = 8
undefined i32   = -1431655766
mode = safe
@sizeOf(Bare)   = 8
@sizeOf(Tagged) = 8
undefined i32   = -1431655766
mode = fast
@sizeOf(Bare)   = 4
@sizeOf(Tagged) = 8
undefined i32   = 0
mode = small
@sizeOf(Bare)   = 4
@sizeOf(Tagged) = 8
undefined i32   = 0
```

Two separate surprises:

**`@sizeOf(Bare)` is 8 in Debug, 4 in ReleaseFast.** A bare `union` carries a
*hidden* safety tag when `runtime_safety` is on. Your struct layouts change
between modes — so never `@sizeOf` a bare union into a wire format or a C ABI
boundary. (`union(enum)` is 8 everywhere: its tag is a real, declared field.)

**`undefined` is `-1431655766` in Debug, `0` in ReleaseFast.** Debug fills
uninitialised memory with the `0xAA` poison pattern; release leaves it alone, so
it usually reads as zero. A program that reads an `undefined` value "works" in
release and fails in Debug, or the reverse. Reading `undefined` is UB in every
mode — both numbers are wrong answers, and the poison is deliberately ugly so
you notice.

And the companion: reading the wrong field of a bare union.

```zig
const std = @import("std");
const builtin = @import("builtin");

const Bare = union { int: u32, float: f32 };

pub fn main() void {
    var v: Bare = .{ .int = 1065353216 };
    _ = &v;
    std.debug.print("mode = {t}\n", .{builtin.mode});
    std.debug.print("read .float = {d}\n", .{v.float});
}
```

```sh
for m in debug safe fast small; do zig run -O$m bu.zig; done
```

**Deliberate:**

```
=== -Odebug ===
mode = debug
thread 16637281 panic: access of union field 'float' while field 'int' is active
/path/bu.zig:10:47: 0x10063f8c7 in main (bu)
    std.debug.print("read .float = {d}\n", .{v.float});
                                              ^
=== -Osafe ===
mode = safe
thread 16637553 panic: access of union field 'float' while field 'int' is active
=== -Ofast ===
mode = fast
read .float = 1
=== -Osmall ===
mode = small
read .float = 1
```

The hidden tag is what makes the Debug panic possible; with it gone, the bits
`1065353216` reinterpret as the float `1.0` and nothing complains. **Use
`union(enum)` unless you are deliberately type-punning.**

## 6. What you gain: speed and size

```zig
const std = @import("std");
const builtin = @import("builtin");

fn collatzSteps(start: u64) u64 {
    var n = start;
    var steps: u64 = 0;
    while (n != 1) : (steps += 1) {
        n = if (n % 2 == 0) n / 2 else 3 * n + 1;
    }
    return steps;
}

pub fn main(init: std.process.Init) void {
    const io = init.io;
    const t0 = std.Io.Clock.Timestamp.now(io, .awake);

    var worst: u64 = 0;
    for (1..300_000) |i| {
        const s = collatzSteps(@intCast(i));
        if (s > worst) worst = s;
    }

    const dt = t0.durationTo(std.Io.Clock.Timestamp.now(io, .awake));
    std.debug.print("{t}: worst = {d} steps in {d} ms\n", .{
        builtin.mode, worst, @divTrunc(dt.raw.toMicroseconds(), 1000),
    });
}
```

```sh
for m in debug safe fast small; do zig run -O$m bench.zig; done
```

```
debug: worst = 442 steps in 113 ms
safe: worst = 442 steps in 39 ms
fast: worst = 442 steps in 27 ms
small: worst = 442 steps in 27 ms
```

Debug → ReleaseSafe is **~2.8x**. ReleaseSafe → ReleaseFast is a further ~1.4x.
(Timings vary a millisecond or two per run; the ratios are what matter.)
**Most of the win is in turning on the optimiser, not in turning off safety** —
and this is a tight arithmetic loop, the case that flatters `-Ofast` most. For
I/O-bound code the difference is in the noise.

Binary size, for a hello-world:

```sh
for m in debug safe fast small; do
  zig build-exe -O$m -femit-bin=hw-$m hw.zig
  printf "%-6s %s bytes\n" "$m" "$(stat -f%z hw-$m)"
done
```

```
debug  2001496 bytes
safe   468904 bytes
fast   395048 bytes
small  165080 bytes
```

Debug is 4x ReleaseSafe (debug info and no inlining) and 12x ReleaseSmall. (The
exact byte counts shift slightly with your build path, which is embedded in the
debug info — the ratios don't.)
ReleaseSmall drops stack-trace machinery entirely — which is why a leak report
there says `Cannot print stack trace: stack tracing is disabled`.

## 7. Per-function control: `@setRuntimeSafety`

Safety is per-scope, so you can keep `-Osafe` everywhere and disable checks in
the one function you've proved correct.

```zig
const std = @import("std");
const builtin = @import("builtin");

fn checkedAdd(a: u8, b: u8) u8 {
    return a + b;
}

fn uncheckedAdd(a: u8, b: u8) u8 {
    @setRuntimeSafety(false);
    return a + b;
}

pub fn main() void {
    std.debug.print("mode = {t}\n", .{builtin.mode});
    std.debug.print("unchecked 250+10 = {d}\n", .{uncheckedAdd(250, 10)});
    std.debug.print("checked 250+10   = {d}\n", .{checkedAdd(250, 10)});
}
```

```sh
zig run srs.zig
```

**The panic is deliberate** — in a plain `-Odebug` build:

```
mode = debug
unchecked 250+10 = 4
thread 16641118 panic: integer overflow
/path/srs.zig:5:14: 0x102b61d63 in checkedAdd (srs)
    return a + b;
             ^
/path/srs.zig:16:61: 0x102b611d7 in main (srs)
    std.debug.print("checked 250+10   = {d}\n", .{checkedAdd(250, 10)});
```

Two functions, one file, one build: one wraps, one panics. This beats
`-Ofast`-the-whole-program — profile, find the hot function, disable safety
there, leave the other 99% checked.

## 8. Running your tests in release mode

Behaviour differences are real bugs, so test for them. `zig test` takes `-O`:

```sh
zig test -Osafe mytests.zig
zig test -Ofast mytests.zig
```

| What this catches | How |
|---|---|
| code that only works because a safety check caught it | passes in `debug`, fails in `fast` |
| `builtin.mode`-dependent logic | run both |
| an `undefined` read | `0xAA` in debug vs `0` in fast |
| a bare-union size assumption | `@sizeOf` differs |

Guard genuinely mode-specific tests rather than letting them fail:

```zig
const std = @import("std");
const builtin = @import("builtin");

test "wrapping behaviour is explicit, not mode-dependent" {
    const a: u8 = 250;
    try std.testing.expectEqual(4, a +% 10);
}

test "safety-check-dependent test" {
    if (!std.debug.runtime_safety) return error.SkipZigTest;
    try std.testing.expect(builtin.mode == .debug or builtin.mode == .safe);
}
```

```sh
zig test rm.zig && zig test -Ofast rm.zig
```

```
1/2 rm.test.wrapping behaviour is explicit, not mode-dependent...OK
2/2 rm.test.safety-check-dependent test...OK
All 2 tests passed.
1/2 rm.test.wrapping behaviour is explicit, not mode-dependent...OK
2/2 rm.test.safety-check-dependent test...SKIP
1 passed; 1 skipped; 0 failed.
```

## Which mode to ship

**`-Osafe`, unless you have a benchmark that says otherwise.**

| Situation | Mode | Why |
|---|---|---|
| a server, a CLI, anything parsing untrusted input | `-Osafe` | an overflow should crash, not corrupt |
| development and CI | `-Odebug` | full traces, poisoned memory, leak reports |
| a profiled hot numeric loop | `-Ofast`, or `-Osafe` + `@setRuntimeSafety(false)` | the second is better: scoped |
| embedded, wasm, a size budget | `-Osmall` | 165 KB vs 469 KB here |
| handling money, auth, or crypto | `-Osafe`, never `-Ofast` | a silently wrapped integer is a vulnerability |

`-Ofast` is **not** the default release mode in Zig the way `--release` is in
Cargo. Choosing it is choosing C semantics: an integer overflow becomes
undefined behaviour and an out-of-bounds read becomes a security hole. Reach for
`@setRuntimeSafety(false)` on the measured hot path instead, and keep the rest
of the program checked.

## Module summary

| Fact | Lesson |
|---|---|
| `test` blocks live beside the code and see private declarations | 9.1 |
| test bodies aren't even analysed in a normal build | 9.1 |
| another file's tests need `test { _ = @import(...); }` | 9.1 |
| a nested type's tests need `refAllDecls(@This())` | 9.1 |
| every matcher returns an error union — always `try` | 9.2 |
| argument order is `(expected, actual)` | 9.2 |
| `expectEqual` on slices compares **pointers** | 9.2 |
| `expect` prints no diagnostic; prefer a typed matcher | 9.2 |
| `return error.SkipZigTest` to skip; it doesn't affect the exit code | 9.2 |
| `errdefer` in a test body is your failure-diagnostic hook | 9.2 |
| **a leak fails the test — and prints "All N tests passed" while exiting 1** | 9.3 |
| check the **exit code** in CI, never the summary line | 9.3 |
| `checkAllAllocationFailures` tests every OOM path | 9.3 |
| `testing.allocator` catches leaks in all four modes | 9.3 |
| `std.debug.print` is temporary, `std.log` is shipped | 9.4 |
| `dumpErrorReturnTrace` for errors, `dumpStackTrace` is a different type | 9.4 |
| panic frames print innermost first | 9.4 |
| `assert` is compiled out in `-Ofast`/`-Osmall` | 9.4, 9.5 |
| overflow panics in Debug/ReleaseSafe, **wraps** in ReleaseFast/ReleaseSmall | 9.5 |
| `DebugAllocator` leak detection vanishes in `-Ofast`/`-Osmall` | 9.5 |
| `@sizeOf` a bare union, and `undefined`'s value, differ by mode | 9.5 |
| ship `-Osafe`; use `@setRuntimeSafety(false)` on measured hot spots | 9.5 |

---

Next: [Module 10 — Build System](../10-build-system/)
