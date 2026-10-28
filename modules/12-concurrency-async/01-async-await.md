# 12.1 — async & await

`io.async(fn, args)` returns a `std.Io.Future(T)`. `future.await(io)` gives you
the `T`. That is the whole API surface.

| Runtime | Spawn | Join | Keyword needed? |
|---|---|---|---|
| Node | `fn()` returns a Promise | `await p` | `async` on every caller |
| Go | `go fn()` | channel / `WaitGroup` | no |
| Python | `asyncio.create_task(f())` | `await t` | `async`/`await` on every caller |
| **Zig 0.17** | `io.async(fn, .{args})` | `fut.await(io)` | **none — `fn` is an ordinary function** |

## Sequential vs async, measured

```zig
const std = @import("std");

fn work(io: std.Io, name: []const u8, ms: i64) void {
    std.Io.sleep(io, .fromMilliseconds(ms), .awake) catch {};
    std.debug.print("{s} finished after {d} ms\n", .{ name, ms });
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;

    const t0 = std.Io.Timestamp.now(io, .awake);
    work(io, "seq-a", 100);
    work(io, "seq-b", 100);
    const t1 = std.Io.Timestamp.now(io, .awake);

    var a = io.async(work, .{ io, "par-a", 100 });
    var b = io.async(work, .{ io, "par-b", 100 });
    a.await(io);
    b.await(io);
    const t2 = std.Io.Timestamp.now(io, .awake);

    std.debug.print("sequential: {d} ms\n", .{t0.durationTo(t1).toMilliseconds()});
    std.debug.print("async:      {d} ms\n", .{t1.durationTo(t2).toMilliseconds()});
}
```

```
seq-a finished after 100 ms
seq-b finished after 100 ms
par-a finished after 100 ms
par-b finished after 100 ms
sequential: 209 ms
async:      105 ms
```

Same `work` function both times. Calling it directly blocks; handing it to
`io.async` does not.

> `fromMilliseconds` takes an **`i64`**, so `ms: u64` is a compile error.
> `sleep` is `(io, duration, clock)` — duration first — and the monotonic clock
> is spelled **`.awake`**. `Timestamp.now` is *not* fallible: `try` on it fails
> to compile.

## Why there is no `async` keyword

Look at `work`'s signature again:

```
fn work(io: std.Io, name: []const u8, ms: i64) void
```

No `async`. No `Future` return type. No `Promise`. The concurrency lives in the
**`io` value**, not in the function's type — so the same body serves both
models and the caller decides:

```zig
const std = @import("std");

fn download(io: std.Io, url: []const u8) usize {
    std.Io.sleep(io, .fromMilliseconds(40), .awake) catch {};
    return url.len;
}

fn totalSequential(io: std.Io, urls: []const []const u8) usize {
    var n: usize = 0;
    for (urls) |u| n += download(io, u);
    return n;
}

fn totalConcurrent(io: std.Io, urls: []const []const u8) usize {
    var futures: [3]std.Io.Future(usize) = undefined;
    for (urls, &futures) |u, *f| f.* = io.async(download, .{ io, u });
    var n: usize = 0;
    for (&futures) |*f| n += f.await(io);
    return n;
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const urls: []const []const u8 = &.{ "a.example", "bb.example", "ccc.example" };

    var t0 = std.Io.Timestamp.now(io, .awake);
    const s = totalSequential(io, urls);
    var t1 = std.Io.Timestamp.now(io, .awake);
    std.debug.print("sequential {d} bytes, {d} ms\n", .{ s, t0.durationTo(t1).toMilliseconds() });

    t0 = std.Io.Timestamp.now(io, .awake);
    const c = totalConcurrent(io, urls);
    t1 = std.Io.Timestamp.now(io, .awake);
    std.debug.print("concurrent {d} bytes, {d} ms\n", .{ c, t0.durationTo(t1).toMilliseconds() });
}
```

```
sequential 30 bytes, 133 ms
concurrent 30 bytes, 45 ms
```

`download` was written once and never touched. This is the problem that
function colouring creates in every other language:

| | JS / Python | Zig 0.17 |
|---|---|---|
| sync caller of an async fn | impossible (or `asyncio.run`, blocking the loop) | just call it |
| async caller of a sync fn | blocks the whole event loop | fine — `io` handles it |
| two libraries, different models | cannot compose; need `sync-rpc`, `nest_asyncio`, `anyio` | compose freely |
| changing a leaf to do I/O | `async` virally rewrites every caller | add one `io` parameter |

The cost is real: `io` must be threaded through the call graph. It is the same
trade you already accept for allocators.

## Returning values and errors

A `Future(T)` carries whatever the function returns — including an error union.
Nothing special happens to errors; `await` hands you the error union and you
`try` it where you want.

```zig
const std = @import("std");

fn square(io: std.Io, n: u32) u32 {
    std.Io.sleep(io, .fromMilliseconds(30), .awake) catch {};
    return n * n;
}

fn parse(io: std.Io, text: []const u8) !u32 {
    std.Io.sleep(io, .fromMilliseconds(30), .awake) catch {};
    return std.fmt.parseInt(u32, text, 10);
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;

    var f: std.Io.Future(u32) = io.async(square, .{ io, 7 });
    std.debug.print("square(7) = {d}\n", .{f.await(io)});

    var good = io.async(parse, .{ io, "123" });
    var bad = io.async(parse, .{ io, "12x" });
    std.debug.print("good = {!d}\n", .{good.await(io)});
    std.debug.print("bad  = {!d}\n", .{bad.await(io)});

    var again = io.async(square, .{ io, 9 });
    _ = again.await(io);
    std.debug.print("await twice = {d}\n", .{again.await(io)});
}
```

```
square(7) = 49
good = 123
bad  = error.InvalidCharacter
await twice = 81
```

| Fact | Detail |
|---|---|
| `Future(T)` type | `T` is exactly the function's return type, errors included |
| `await` is idempotent | second call returns the cached result, no re-run |
| `await` is **not** threadsafe | one owner per future |
| errors don't need `try` at the `await` | you get the union; `try` it when you want it |
| the future must stay alive | it holds the result slot — don't let it go out of scope before `await` |

## Fan-out over an array

```zig
const std = @import("std");

fn fetch(io: std.Io, id: u32) u32 {
    std.Io.sleep(io, .fromMilliseconds(50), .awake) catch {};
    return id * 10;
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const t0 = std.Io.Timestamp.now(io, .awake);

    var futures: [8]std.Io.Future(u32) = undefined;
    for (&futures, 0..) |*f, i| f.* = io.async(fetch, .{ io, @as(u32, @intCast(i)) });

    var total: u32 = 0;
    for (&futures) |*f| total += f.await(io);

    const t1 = std.Io.Timestamp.now(io, .awake);
    std.debug.print("total = {d} in {d} ms\n", .{ total, t0.durationTo(t1).toMilliseconds() });
}
```

```
total = 280 in 55 ms
```

Eight 50 ms tasks in 55 ms. Two loops: spawn everything, then await
everything. Awaiting inside the spawn loop would serialize them.

## `std.Io.Group` — when you don't need the results

`Group` is Zig's `WaitGroup` / `asyncio.gather`. Tasks must return something
coercible to `Cancelable!void`, and the group is awaited as a whole.

```zig
const std = @import("std");

var done: std.atomic.Value(u32) = .init(0);

fn task(io: std.Io, ms: i64) void {
    std.Io.sleep(io, .fromMilliseconds(ms), .awake) catch {};
    _ = done.fetchAdd(1, .monotonic);
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    var g: std.Io.Group = .init;
    for (0..5) |i| g.async(io, task, .{ io, @as(i64, @intCast(20 + i * 10)) });
    try g.await(io);
    std.debug.print("group finished, done = {d}\n", .{done.load(.monotonic)});
}
```

```
group finished, done = 5
```

| | `Future(T)` | `Group` |
|---|---|---|
| return values | yes, typed | no — must coerce to `Cancelable!void` |
| task count | fixed at compile time (you declare the array) | dynamic, add in a loop |
| storage | you own the `Future` | the group owns everything |
| await | per-future | all-or-nothing |
| closest analogue | `Promise` | `WaitGroup` / `asyncio.gather` |

## Shapes to remember

| Code | Means |
|---|---|
| `var f = io.async(work, .{ io, x })` | spawn; `f` is a `Future(@TypeOf(work(...)))` |
| `f.await(io)` | block until done, return the value |
| `io.async(f, .{})` | zero-arg function: the tuple is still required |
| `var fs: [N]std.Io.Future(T) = undefined` | fan-out storage |
| `var g: std.Io.Group = .init; g.async(io, f, args)` | fire-and-collect |
| `try g.await(io)` | `Group.await` is cancelable, so it needs `try` |

---

Next: [12.2 — concurrent](./02-concurrent.md)
