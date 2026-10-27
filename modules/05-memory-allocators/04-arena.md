# 5.4 — Arena

**Allocate as much as you like, never free anything, throw it all away at
once.** The arena is the allocator that makes manual memory management feel
easy, and it is the backbone of every Zig server you will read.

## No `free` calls anywhere

```zig
const std = @import("std");

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer {
        const status = gpa_s.deinit();
        std.debug.print("deinit status: {t}\n", .{status});
    }

    var arena_s: std.heap.ArenaAllocator = .init(gpa_s.allocator());
    defer arena_s.deinit();
    const arena = arena_s.allocator();

    for (0..5) |i| {
        const s = try std.fmt.allocPrint(arena, "row {d}", .{i});
        std.debug.print("{s} @ {*}\n", .{ s, s.ptr });
    }
    const nums = try arena.alloc(u64, 100);
    nums[0] = 1;
    const node = try arena.create(struct { x: i32 });
    node.x = 5;

    std.debug.print("no free calls at all; capacity = {d} bytes\n", .{arena_s.queryCapacity()});
}
```

```
row 0 @ u8@100c40018
row 1 @ u8@100c4001d
row 2 @ u8@100c40022
row 3 @ u8@100c40027
row 4 @ u8@100c4002c
no free calls at all; capacity = 1368 bytes
deinit status: ok
```

Two things to notice.

**The addresses are adjacent**: `…018`, `…01d`, `…022` — five bytes apart, the
exact length of `"row 0"`. An arena is a **bump pointer**: allocation is "return
the current offset, then add the size". No free lists, no size classes, no
metadata per allocation.

**The DebugAllocator reports `ok`** even though nothing was freed by name. The
arena's `deinit` returned every page to the backing allocator in one go.

## How it's wired

The arena is **not** a source of memory — it's a strategy layered over one:

```zig
var arena_s: std.heap.ArenaAllocator = .init(gpa_s.allocator());
defer arena_s.deinit();
const arena = arena_s.allocator();
```

| | |
|---|---|
| `.init(backing)` | takes an `Allocator` — usually a DebugAllocator or `page_allocator` |
| `arena_s.deinit()` | frees **everything**; returns `void`, so no `_ =` needed |
| `arena_s.allocator()` | the `Allocator` you hand to functions |
| `arena.free(x)` | legal, and almost always a no-op |

Still two variables, still `var` for the state and `const` for the interface.
The `defer arena_s.deinit()` is the only cleanup line in the program.

## `reset` — the per-request server pattern

`deinit` destroys the arena. `reset` empties it but keeps it usable:

```zig
const std = @import("std");

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer _ = gpa_s.deinit();
    const gpa = gpa_s.allocator();

    var arena_s: std.heap.ArenaAllocator = .init(gpa);
    defer arena_s.deinit();
    const arena = arena_s.allocator();

    std.debug.print("start         : capacity={d}\n", .{arena_s.queryCapacity()});

    for (0..3) |req| {
        for (0..50) |i| {
            _ = try std.fmt.allocPrint(arena, "request {d} item {d}", .{ req, i });
        }
        const cap_before = arena_s.queryCapacity();
        _ = arena_s.reset(.retain_capacity);
        std.debug.print("req {d} done    : capacity {d} -> {d} (retained)\n", .{ req, cap_before, arena_s.queryCapacity() });
    }

    _ = try arena.alloc(u8, 10);
    _ = arena_s.reset(.free_all);
    std.debug.print("free_all      : capacity={d}\n", .{arena_s.queryCapacity()});

    _ = try arena.alloc(u8, 9000);
    const big = arena_s.queryCapacity();
    _ = arena_s.reset(.{ .retain_with_limit = 1024 });
    std.debug.print("limit 1024    : capacity {d} -> {d}\n", .{ big, arena_s.queryCapacity() });
}
```

```
start         : capacity=0
req 0 done    : capacity 1566 -> 1566 (retained)
req 1 done    : capacity 1566 -> 1566 (retained)
req 2 done    : capacity 1566 -> 1566 (retained)
free_all      : capacity=0
limit 1024    : capacity 13538 -> 1024
```

| Mode | Capacity after | Use when |
|---|---|---|
| `.retain_capacity` | unchanged | **a loop** — reuse the pages, zero syscalls per iteration |
| `.free_all` | 0 | you're done for a while and want the memory back |
| `.{ .retain_with_limit = n }` | ≤ n | a loop, but cap the high-water mark |

`reset` returns `bool` — `false` means the capacity-retaining re-allocation
failed and the arena fell back to empty. It is still safe to use either way,
which is why `_ = arena_s.reset(…)` is the idiomatic call.

**`.retain_capacity` in a loop is the whole trick.** After the first iteration
the arena never asks the OS for memory again — notice capacity stays at 1566
across all three requests.

### The actual server shape

```zig
const std = @import("std");

const Request = struct { path: []const u8, body: []const u8 };

fn handle(arena: std.mem.Allocator, req: Request) ![]const u8 {
    const parts = try arena.alloc([]const u8, 3);
    parts[0] = try std.fmt.allocPrint(arena, "path={s}", .{req.path});
    parts[1] = try std.fmt.allocPrint(arena, "len={d}", .{req.body.len});
    parts[2] = try arena.dupe(u8, "status=200");
    return std.mem.join(arena, " ", parts);
}

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer {
        const status = gpa_s.deinit();
        std.debug.print("deinit status: {t}\n", .{status});
    }
    const gpa = gpa_s.allocator();

    var arena_s: std.heap.ArenaAllocator = .init(gpa);
    defer arena_s.deinit();

    const requests = [_]Request{
        .{ .path = "/users", .body = "{}" },
        .{ .path = "/users/42", .body = "{\"name\":\"raj\"}" },
        .{ .path = "/health", .body = "" },
    };

    for (requests) |req| {
        defer _ = arena_s.reset(.retain_capacity);
        const resp = try handle(arena_s.allocator(), req);
        std.debug.print("{s}\n", .{resp});
    }
}
```

```
path=/users len=2 status=200
path=/users/42 len=14 status=200
path=/health len=0 status=200
deinit status: ok
```

Count the cleanup in `handle`: **zero.** Four allocations, one of them a slice of
slices, one a `join` that allocates again — and not a single `free` or `defer`.
The `defer _ = arena_s.reset(.retain_capacity)` at the top of the loop body
cleans up the entire request no matter how `handle` exits, including on error.

This is the pattern in module 14, and it is the answer to "how do you manage
memory in a Zig web service?" in an interview:

> Long-lived state (the store, the config, the connection pool) uses the
> general-purpose allocator and is freed explicitly. Everything scoped to one
> request — parsed JSON, formatted strings, intermediate buffers — comes from a
> per-request arena that gets reset when the response is written. Request
> handlers never free anything individually, so they can't leak.

## Why it's so much faster

```zig
const std = @import("std");

const N = 200_000;

fn withGpa(gpa: std.mem.Allocator) !u64 {
    var acc: u64 = 0;
    for (0..N) |i| {
        const p = try gpa.alloc(u8, 32);
        p[0] = @truncate(i);
        acc += p[0];
        gpa.free(p);
    }
    return acc;
}

fn withArena(backing: std.mem.Allocator) !u64 {
    var arena_s: std.heap.ArenaAllocator = .init(backing);
    defer arena_s.deinit();
    const arena = arena_s.allocator();
    var acc: u64 = 0;
    for (0..N) |i| {
        const p = try arena.alloc(u8, 32);
        p[0] = @truncate(i);
        acc += p[0];
    }
    return acc;
}

fn micros(io: std.Io, from: std.Io.Clock.Timestamp) i64 {
    return from.durationTo(.now(io, .awake)).raw.toMicroseconds();
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer _ = gpa_s.deinit();
    const gpa = gpa_s.allocator();

    const t0: std.Io.Clock.Timestamp = .now(io, .awake);
    const a = try withGpa(gpa);
    const gpa_us = micros(io, t0);

    const t1: std.Io.Clock.Timestamp = .now(io, .awake);
    const b = try withArena(gpa);
    const arena_us = micros(io, t1);

    std.debug.print("{d} allocs, alloc+free each : {d} us (sum={d})\n", .{ N, gpa_us, a });
    std.debug.print("{d} allocs, arena, free once: {d} us (sum={d})\n", .{ N, arena_us, b });
}
```

```sh
zig run -OReleaseSafe bench.zig
```

```
200000 allocs, alloc+free each : 450744 us (sum=25493856)
200000 allocs, arena, free once: 1086 us (sum=25493856)
```

**~400x.** (Exact numbers vary per run and per machine; the order of magnitude
does not.) An arena allocation is an add and a bounds check. A general-purpose
allocation walks size classes, updates free lists, and — in a DebugAllocator —
captures a stack trace.

Timing in 0.17 needs `io`: `std.time.Timer` no longer exists, and
`std.time` is now only unit constants. Use
`std.Io.Clock.Timestamp.now(io, .awake)` and `durationTo`.

## The tradeoff: memory only goes up

An arena cannot reclaim memory in the middle. Free the *most recent*
allocation and it does get reused:

```zig
const std = @import("std");

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer _ = gpa_s.deinit();
    const gpa = gpa_s.allocator();

    var arena_s: std.heap.ArenaAllocator = .init(gpa);
    defer arena_s.deinit();
    const arena = arena_s.allocator();

    for (0..5) |i| {
        const tmp = try arena.alloc(u8, 100_000);
        arena.free(tmp);
        std.debug.print("iteration {d}: freed 100 KB, arena capacity = {d}\n", .{ i, arena_s.queryCapacity() });
    }
}
```

```
iteration 0: freed 100 KB, arena capacity = 150038
iteration 1: freed 100 KB, arena capacity = 150038
iteration 2: freed 100 KB, arena capacity = 150038
iteration 3: freed 100 KB, arena capacity = 150038
iteration 4: freed 100 KB, arena capacity = 150038
```

Flat — because `tmp` was the last thing allocated, so `free` just moved the bump
pointer back. Now allocate **anything** after it, so the freed block is no
longer on top:

```zig
const std = @import("std");

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer _ = gpa_s.deinit();
    const gpa = gpa_s.allocator();

    var arena_s: std.heap.ArenaAllocator = .init(gpa);
    defer arena_s.deinit();
    const arena = arena_s.allocator();

    const keep = try arena.alloc(u8, 8);
    _ = keep;

    for (0..5) |i| {
        const tmp = try arena.alloc(u8, 100_000);
        const pin = try arena.alloc(u8, 8);
        _ = pin;
        arena.free(tmp);
        std.debug.print("iteration {d}: freed 100 KB, capacity = {d}\n", .{ i, arena_s.queryCapacity() });
    }
}
```

```
iteration 0: freed 100 KB, capacity = 150162
iteration 1: freed 100 KB, capacity = 525368
iteration 2: freed 100 KB, capacity = 525368
iteration 3: freed 100 KB, capacity = 525368
iteration 4: freed 100 KB, capacity = 1238214
```

**1.2 MB held for 8 bytes of live data.** Eight 8-byte pins made 500 KB of
freed space unreclaimable. In a long-running loop with no `reset`, this is
unbounded growth that no leak detector will report — the arena *owns* that
memory and will release it at `deinit`. It just happens to be at process exit.

| Use an arena when | Don't when |
|---|---|
| work has a clear end (a request, a frame, a CLI run) | the process runs forever with no natural boundary |
| you allocate many small things with the same lifetime | lifetimes differ wildly |
| the output is copied out or written before reset | you need peak memory bounded by *live* data |
| a parse tree, an AST, intermediate buffers | a long-lived cache with eviction |

**The discipline: an arena needs an owner who resets it on a schedule.** A
request, a loop iteration, a function call. If you can't name the moment it
resets, use a general-purpose allocator.

## Returning data out of an arena

Anything allocated from an arena dies at `reset`/`deinit`. To hand a result to
the outside you must copy it out with the *caller's* allocator:

```zig
const std = @import("std");

fn parse(gpa: std.mem.Allocator, input: []const u8) ![]u8 {
    var arena_s: std.heap.ArenaAllocator = .init(gpa);
    defer arena_s.deinit();
    const arena = arena_s.allocator();

    var list: std.ArrayList([]const u8) = .empty;
    var it = std.mem.splitScalar(u8, input, ',');
    while (it.next()) |tok| {
        try list.append(arena, try std.fmt.allocPrint(arena, "[{s}]", .{tok}));
    }
    const scratch = try std.mem.join(arena, "", list.items);
    return gpa.dupe(u8, scratch);
}

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer {
        const status = gpa_s.deinit();
        std.debug.print("deinit status: {t}\n", .{status});
    }
    const gpa = gpa_s.allocator();

    const out = try parse(gpa, "a,b,c");
    defer gpa.free(out);
    std.debug.print("result = {s}\n", .{out});
}
```

```
result = [a][b][c]
deinit status: ok
```

`parse` does all its messy intermediate work in a throwaway arena — an
`ArrayList`, three `allocPrint`s, a `join`, none of them freed individually —
then `dupe`s the single final answer into the caller's allocator. One
allocation escapes, one `defer gpa.free(out)` in the caller, and the
DebugAllocator confirms `ok`.

**Scratch arena inside, caller's allocator for the return value.** Memorize
that shape; it's the cleanest way to write an allocating function in Zig.

---

Next: [5.5 — FixedBuffer](./05-fixed-buffer.md)
