# 5.3 — DebugAllocator

Your default allocator during development. It tracks every allocation and
**tells you, with a stack trace and the exact line number, what you leaked.**

> `std.heap.GeneralPurposeAllocator` was renamed to `std.heap.DebugAllocator`.
> Every tutorial you find online still says `GeneralPurposeAllocator(.{})` —
> that name is gone in 0.17.

## The four lines

```zig
const std = @import("std");

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer {
        const status = gpa_s.deinit();
        std.debug.print("deinit status: {t}\n", .{status});
    }
    const gpa = gpa_s.allocator();

    const ok = try gpa.alloc(u8, 16);
    gpa.free(ok);
    std.debug.print("allocated and freed 16 bytes\n", .{});
}
```

```
allocated and freed 16 bytes
deinit status: ok
```

| Line | Why |
|---|---|
| `var gpa_s: std.heap.DebugAllocator(.{}) = .init` | the **state**. `var`, not `const`; `.init` not `.init()` |
| `defer … gpa_s.deinit()` | checks for leaks at scope exit, returns `.ok` or `.leak` |
| `const gpa = gpa_s.allocator()` | the 16-byte `std.mem.Allocator` you pass around |

`deinit()` returns a `std.heap.Check` enum, so the short form you'll usually
write discards it:

```zig
defer _ = gpa_s.deinit();
```

Keep the `_ =` — an unused return value is a compile error. Capture the status
only when you want to print it or fail a test on it.

## Watch a real leak get caught

```zig
const std = @import("std");

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer {
        const status = gpa_s.deinit();
        std.debug.print("deinit status: {t}\n", .{status});
    }
    const gpa = gpa_s.allocator();

    const leaked = try gpa.alloc(u8, 32);
    _ = leaked;
    std.debug.print("allocated 32 bytes, never freed\n", .{});
}
```

**Deliberate leak.** Real output:

```
allocated 32 bytes, never freed
error(DebugAllocator): memory address 0x1042a0000 leaked: 
/path/leak.zig:11:33: 0x1041575eb in main (leak)
    const leaked = try gpa.alloc(u8, 32);
                                ^
/opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/start.zig:788:64: 0x104157977 in callMain (leak)
    if (fn_info.param_types.len == 0) return wrapMain(root.main());
                                                               ^
???:?:?: 0x180407dff in start (/usr/lib/dyld)

deinit status: leak
```

Read that again: **it printed the source line that allocated the memory, with a
caret under the call.** That is the whole value proposition. No valgrind, no
sanitizer build, no guessing.

(`_ = leaked;` is only there to dodge the unused-variable compile error. The
addresses are ASLR-randomized and the paths are yours — the line number and the
source snippet are the parts that matter.)

### A leak in a loop, allocated three frames down

```zig
const std = @import("std");

fn makeName(gpa: std.mem.Allocator, who: []const u8) ![]u8 {
    return std.fmt.allocPrint(gpa, "hello, {s}", .{who});
}

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer {
        const status = gpa_s.deinit();
        std.debug.print("deinit status: {t}\n", .{status});
    }
    const gpa = gpa_s.allocator();

    for (0..3) |i| {
        const s = try makeName(gpa, "zig");
        _ = s;
        std.debug.print("iteration {d}: allocated, forgot to free\n", .{i});
    }
}
```

**Deliberate leak.** Three iterations, three separate reports (first one shown):

```
iteration 0: allocated, forgot to free
iteration 1: allocated, forgot to free
iteration 2: allocated, forgot to free
error(DebugAllocator): memory address 0x104460000 leaked: 
/opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/mem/Allocator.zig:507:55: 0x10430f54f in print__func_878 (leak2)
    var aw = try std.Io.Writer.Allocating.initCapacity(a, format.len);
                                                      ^
/opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/fmt.zig:626:21: 0x10430f433 in allocPrint__func_853 (leak2)
    return gpa.print(fmt, args);
                    ^
/path/leak2.zig:4:30: 0x10430b5b3 in makeName (leak2)
    return std.fmt.allocPrint(gpa, "hello, {s}", .{who});
                             ^
/path/leak2.zig:16:31: 0x10430b6e7 in main (leak2)
        const s = try makeName(gpa, "zig");
                              ^
...
(additional stack frames may have been skipped...)
```

**One report per leaked allocation**, and the trace walks outward: the stdlib
frame that called `rawAlloc`, then `allocPrint`, then *your* `makeName`, then
the `main` loop that called it. Skip the stdlib frames at the top and read the
first line with your own filename in it — line 16 is where the loop forgot its
`defer`.

The fix is one line:

```zig
const s = try makeName(gpa, "zig");
defer gpa.free(s);
```

## Double-free detection

The naive double free is **not** a clean report:

```zig
const std = @import("std");

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer _ = gpa_s.deinit();
    const gpa = gpa_s.allocator();

    const buf = try gpa.alloc(u8, 8);
    gpa.free(buf);
    std.debug.print("freed once\n", .{});
    gpa.free(buf);
    std.debug.print("freed twice\n", .{});
}
```

**Deliberate crash:**

```
freed once
Segmentation fault at address 0x103000000
/opt/homebrew/Cellar/zig/0.17.0/lib/zig/compiler_rt.zig:601:26: 0x102ecff84 in memsetSmallPowerOf2__func_19 (compiler_rt)
        @as(*align(1) T, @ptrCast(d)).* = splatted;
                         ^
/opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/mem/Allocator.zig:475:5: 0x102e1805b in free__func_398 (df)
    @memset(bytes, undefined);
    ^
/path/df.zig:11:13: 0x102ebf663 in main (df)
    gpa.free(buf);
            ^
```

By default the allocator has already unmapped that page, so `free`'s
"poison the bytes" `@memset` faults before any check runs. Useful (it names
line 11) but ugly.

Two config flags turn it into a proper diagnostic:

```zig
const std = @import("std");

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{
        .never_unmap = true,
        .retain_metadata = true,
    }) = .init;
    defer _ = gpa_s.deinit();
    const gpa = gpa_s.allocator();

    const buf = try gpa.alloc(u8, 8);
    gpa.free(buf);
    std.debug.print("freed once\n", .{});
    gpa.free(buf);
    std.debug.print("survived the double free\n", .{});
}
```

**Deliberate double free**, now diagnosed instead of fatal:

```
freed once
error(DebugAllocator): Double free detected. Allocation: 
/path/df2.zig:11:30: 0x100aa35e3 in main (df2)
    const buf = try gpa.alloc(u8, 8);
                             ^
...
 First free: 
/path/df2.zig:12:13: 0x100aa363f in main (df2)
    gpa.free(buf);
            ^
...
 Second free: 
/path/df2.zig:14:13: 0x100aa3663 in main (df2)
    gpa.free(buf);
            ^
...
survived the double free
```

**Three stack traces: where it was allocated, where it was first freed, where it
was freed again.** That is the single most useful debugging output in the
standard library. Note the program then *continues* — the report is logged, not
fatal.

`never_unmap` keeps pages mapped and `retain_metadata` keeps the bookkeeping
alive, so both cost memory. Turn them on when you're chasing a specific bug, not
permanently.

## The config options worth knowing

```zig
const std = @import("std");

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{ .verbose_log = true }) = .init;
    defer _ = gpa_s.deinit();
    const gpa = gpa_s.allocator();

    const a = try gpa.alloc(u8, 10);
    const b = try gpa.create(u64);
    gpa.destroy(b);
    gpa.free(a);
}
```

```
info(DebugAllocator): small alloc 10 bytes at 0x1051c0000
info(DebugAllocator): small alloc 8 bytes at 0x1051e0000
info(DebugAllocator): small free 8 bytes at u8@1051e0000
info(DebugAllocator): small free 10 bytes at u8@1051c0000
```

Every allocation and free, in order. Unusable in production, excellent for
"which of these 40 allocations survives?"

### Forcing OutOfMemory on purpose

`enable_memory_limit` adds a settable byte cap — the easiest way to test your
error paths without a real OOM:

```zig
const std = @import("std");

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{ .enable_memory_limit = true }) = .init;
    defer _ = gpa_s.deinit();
    gpa_s.requested_memory_limit = 1024;
    const gpa = gpa_s.allocator();

    var held: [8][]u8 = undefined;
    var n: usize = 0;
    while (n < 8) : (n += 1) {
        held[n] = gpa.alloc(u8, 200) catch |err| {
            std.debug.print("alloc #{d} failed: {t}\n", .{ n, err });
            std.debug.print("requested so far = {d} / limit 1024\n", .{gpa_s.total_requested_bytes});
            break;
        };
    }
    for (held[0..n]) |h| gpa.free(h);
    std.debug.print("freed {d} blocks\n", .{n});
}
```

```
alloc #5 failed: OutOfMemory
requested so far = 1000 / limit 1024
freed 5 blocks
```

Five × 200 = 1000 bytes fit; the sixth would exceed 1024 and fails. Both
`requested_memory_limit` and `total_requested_bytes` only exist when
`enable_memory_limit` is true — otherwise they are `void`.

| Option | Default | What it buys you |
|---|---|---|
| `safety` | `runtime_safety` | all leak and double-free checking |
| `verbose_log` | `false` | log every alloc/free |
| `never_unmap` | `false` | turns use-after-free segfaults into messages |
| `retain_metadata` | `false` | double-free reports (pair with `never_unmap`) |
| `enable_memory_limit` | `false` | `requested_memory_limit`, `total_requested_bytes` |
| `thread_safe` | `!single_threaded` | internal mutex |
| `stack_trace_frames` | 6 (Debug) | depth of the traces above |

## The trap: leak detection disappears in release builds

Same leaking program from the top of this lesson, one flag different:

```sh
zig run -OReleaseFast leak.zig
```

```
allocated 32 bytes, never freed
deinit status: ok
```

**It says `ok`. The leak is still there.** `safety` defaults to
`std.debug.runtime_safety`, which is `false` in `ReleaseFast` and
`ReleaseSmall`, so all the tracking compiles away. That's by design — the
tracking costs memory and time — but it means:

| Build mode | Leaks caught |
|---|---|
| `Debug` (default) | yes |
| `ReleaseSafe` | yes |
| `ReleaseFast` | **no** |
| `ReleaseSmall` | **no** |

So: **develop and test in Debug, deploy in release.** And for production
workloads prefer an allocator chosen for throughput — `std.heap.smp_allocator`
(lesson 5.5) or an arena (lesson 5.4) — rather than a DebugAllocator with its
safety switched off.

In tests you get this for free: `std.testing.allocator` is a DebugAllocator and
a leak **fails the test** (lesson 5.8).

---

Next: [5.4 — Arena](./04-arena.md)
