# 12.7 — Concurrent Echo Server

Everything from this module in one program: `listen`, `accept`, one
`io.concurrent` task per connection, per-connection buffers, and a shutdown
that happens after a fixed number of clients so the program terminates.

| Step | Call |
|---|---|
| address | `const addr: std.Io.net.IpAddress = .{ .ip4 = .loopback(39230) };` |
| listen | `var server = try addr.listen(io, .{ .reuse_address = true });` + `defer server.deinit(io)` |
| accept | `const stream = try server.accept(io);` → `std.Io.net.Stream` |
| handle | `try conns.concurrent(io, handle, .{ io, stream })` |
| read | `var r = s.reader(io, &rbuf); r.interface.takeDelimiterInclusive('\n')` |
| write | `var w = s.writer(io, &wbuf); w.interface.writeAll(line); w.interface.flush()` |
| close | `s.close(io)` |
| connect | `addr.connect(io, .{ .mode = .stream })` — **`mode` has no default** |

Each program below runs a server **and** its clients in one process, so you can
`zig run` it and watch both sides. In a real service `main` would be the server
alone, looping on `accept` forever.

## The server

```zig
const std = @import("std");

const port = 39230;

fn handle(io: std.Io, stream: std.Io.net.Stream, id: u32) void {
    var s = stream;
    defer s.close(io);

    var rbuf: [256]u8 = undefined;
    var wbuf: [256]u8 = undefined;
    var r = s.reader(io, &rbuf);
    var w = s.writer(io, &wbuf);

    var lines: u32 = 0;
    while (true) {
        const line = r.interface.takeDelimiterInclusive('\n') catch break;
        w.interface.print("conn{d}: {s}", .{ id, line }) catch break;
        w.interface.flush() catch break;
        lines += 1;
    }
    std.debug.print("conn{d} closed after {d} line(s)\n", .{ id, lines });
}

fn client(io: std.Io, msg: []const u8) void {
    const addr: std.Io.net.IpAddress = .{ .ip4 = .loopback(port) };
    var stream = addr.connect(io, .{ .mode = .stream }) catch return;
    defer stream.close(io);

    var wbuf: [128]u8 = undefined;
    var rbuf: [128]u8 = undefined;
    var w = stream.writer(io, &wbuf);
    var r = stream.reader(io, &rbuf);

    w.interface.print("{s}\n", .{msg}) catch return;
    w.interface.flush() catch return;
    const reply = r.interface.takeDelimiterInclusive('\n') catch return;
    std.debug.print("client {s} got: {s}", .{ msg, reply });
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const addr: std.Io.net.IpAddress = .{ .ip4 = .loopback(port) };
    var server = try addr.listen(io, .{ .reuse_address = true });
    defer server.deinit(io);

    const msgs: []const []const u8 = &.{ "alpha", "bravo", "charlie" };
    var clients: [3]std.Io.Future(void) = undefined;
    for (&clients, msgs) |*c, m| c.* = try io.concurrent(client, .{ io, m });

    var conns: [3]std.Io.Future(void) = undefined;
    for (&conns, 0..) |*h, i| {
        const stream = try server.accept(io);
        h.* = try io.concurrent(handle, .{ io, stream, @as(u32, @intCast(i)) });
    }

    for (&clients) |*c| c.await(io);
    for (&conns) |*h| h.await(io);
    std.debug.print("accepted 3 clients, shutting down\n", .{});
}
```

```
client alpha got: conn0: alpha
client bravo got: conn1: bravo
conn0 closed after 1 line(s)
conn1 closed after 1 line(s)
client charlie got: conn2: charlie
conn2 closed after 1 line(s)
accepted 3 clients, shutting down
```

**The lines interleave differently every run** — three `handle` tasks and three
`client` tasks print from different threads. What does not vary: every client
gets its own echo back, and all three connections report closing.

Things that will bite you:

| Trap | Fix |
|---|---|
| `io.async` instead of `io.concurrent` for the clients | the client might run inline before anything is accepting → deadlock |
| accepting before spawning clients | `accept` blocks forever; nothing is connecting yet |
| `addr.connect(io, .{})` | compile error — `ConnectOptions.mode` has **no default** |
| forgetting `w.interface.flush()` | the writer is buffered; the peer blocks on a read that never arrives |
| `takeDelimiterExclusive` in a `while` loop | spins forever on an empty line instead of signalling EOF |
| `net.Socket` has **no `localAddress()`** | bind a fixed port instead of asking for port 0 |
| `error.AddressInUse` on the **second** run | `ListenOptions.reuse_address` defaults to **`false`**; set it `true` or wait out TIME_WAIT |
| `stream` is passed by value | `var s = stream;` first, because `close`/`reader`/`writer` need `*Stream` |

`Stream.reader(io, &buf)` and `.writer(io, &buf)` return a `Stream.Reader` /
`Stream.Writer` whose `.interface` is a **field**, not a method — so
`r.interface.takeDelimiterInclusive(...)`, no parentheses after `interface`.
`takeDelimiterInclusive` keeps the `\n`, which is why the echo above writes
`line` straight back with no fixup.

## Variation: per-connection arena, multi-line sessions

Buffers sized per connection and freed with the connection — the standard
server shape. Each client sends three lines this time.

```zig
const std = @import("std");

const port = 39231;
const max_clients = 4;

var served: std.atomic.Value(u32) = .init(0);
var echoed: std.atomic.Value(u32) = .init(0);

fn connection(io: std.Io, gpa: std.mem.Allocator, stream: std.Io.net.Stream) void {
    var s = stream;
    defer s.close(io);

    var arena: std.heap.ArenaAllocator = .init(gpa);
    defer arena.deinit();

    const rbuf = arena.allocator().alloc(u8, 1024) catch return;
    const wbuf = arena.allocator().alloc(u8, 1024) catch return;
    var r = s.reader(io, rbuf);
    var w = s.writer(io, wbuf);

    _ = served.fetchAdd(1, .monotonic);
    while (true) {
        const line = r.interface.takeDelimiterInclusive('\n') catch break;
        w.interface.writeAll(line) catch break;
        w.interface.flush() catch break;
        _ = echoed.fetchAdd(1, .monotonic);
    }
}

fn client(io: std.Io, n: u32) void {
    const addr: std.Io.net.IpAddress = .{ .ip4 = .loopback(port) };
    var stream = addr.connect(io, .{ .mode = .stream }) catch return;
    defer stream.close(io);

    var wbuf: [256]u8 = undefined;
    var rbuf: [256]u8 = undefined;
    var w = stream.writer(io, &wbuf);
    var r = stream.reader(io, &rbuf);

    for (0..3) |i| {
        w.interface.print("c{d}-msg{d}\n", .{ n, i }) catch return;
        w.interface.flush() catch return;
        const back = r.interface.takeDelimiterInclusive('\n') catch return;
        std.debug.assert(std.mem.endsWith(u8, back, "\n"));
    }
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const addr: std.Io.net.IpAddress = .{ .ip4 = .loopback(port) };
    var server = try addr.listen(io, .{ .reuse_address = true });
    defer server.deinit(io);

    var clients: std.Io.Group = .init;
    defer clients.cancel(io);
    for (0..max_clients) |i| try clients.concurrent(io, client, .{ io, @as(u32, @intCast(i)) });

    var conns: std.Io.Group = .init;
    for (0..max_clients) |_| {
        const stream = try server.accept(io);
        try conns.concurrent(io, connection, .{ io, init.gpa, stream });
    }

    try clients.await(io);
    try conns.await(io);

    std.debug.print("connections served: {d}\n", .{served.load(.monotonic)});
    std.debug.print("lines echoed:       {d}\n", .{echoed.load(.monotonic)});
}
```

```
connections served: 4
lines echoed:       12
```

Deterministic, because only the totals are printed. Four connections, three
lines each.

| Choice | Why |
|---|---|
| `std.Io.Group` instead of a `Future` array | connection count is a runtime value; `Group` grows |
| per-connection `ArenaAllocator` | one `arena.deinit()` releases every allocation that connection made |
| counters are `atomic.Value` | handlers run on different threads; a plain `u32` would lose updates |
| `defer clients.cancel(io)` | if `accept` fails mid-loop, the client tasks are still torn down |
| `try conns.await(io)` | `Group.await` is cancelable, hence `try` |

The arena is the per-request pattern from module 5, applied per *connection*
instead of per request: allocate freely in the handler, never free
individually, drop it all when the socket closes.

## Variation: proving the concurrency

A handler that takes 100 ms. Four clients. If connections were handled one at
a time this takes 400 ms.

```zig
const std = @import("std");

const port = 39232;
const clients = 4;
const work_ms = 100;

fn slowHandler(io: std.Io, stream: std.Io.net.Stream) void {
    var s = stream;
    defer s.close(io);
    var rbuf: [128]u8 = undefined;
    var wbuf: [128]u8 = undefined;
    var r = s.reader(io, &rbuf);
    var w = s.writer(io, &wbuf);

    const line = r.interface.takeDelimiterInclusive('\n') catch return;
    std.Io.sleep(io, .fromMilliseconds(work_ms), .awake) catch {};
    w.interface.writeAll(line) catch return;
    w.interface.flush() catch return;
}

fn client(io: std.Io) void {
    const addr: std.Io.net.IpAddress = .{ .ip4 = .loopback(port) };
    var stream = addr.connect(io, .{ .mode = .stream }) catch return;
    defer stream.close(io);
    var wbuf: [128]u8 = undefined;
    var rbuf: [128]u8 = undefined;
    var w = stream.writer(io, &wbuf);
    var r = stream.reader(io, &rbuf);
    w.interface.writeAll("ping\n") catch return;
    w.interface.flush() catch return;
    _ = r.interface.takeDelimiterInclusive('\n') catch return;
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const addr: std.Io.net.IpAddress = .{ .ip4 = .loopback(port) };
    var server = try addr.listen(io, .{ .reuse_address = true });
    defer server.deinit(io);

    var cg: std.Io.Group = .init;
    defer cg.cancel(io);
    for (0..clients) |_| try cg.concurrent(io, client, .{io});

    const t0 = std.Io.Timestamp.now(io, .awake);
    var hg: std.Io.Group = .init;
    for (0..clients) |_| {
        const s = try server.accept(io);
        try hg.concurrent(io, slowHandler, .{ io, s });
    }
    try hg.await(io);
    const t1 = std.Io.Timestamp.now(io, .awake);

    try cg.await(io);
    std.debug.print("{d} clients x {d} ms handler, concurrent: {d} ms\n", .{ clients, work_ms, t0.durationTo(t1).toMilliseconds() });
    std.debug.print("(sequential would be ~{d} ms)\n", .{clients * work_ms});
}
```

```
4 clients x 100 ms handler, concurrent: 105 ms
(sequential would be ~400 ms)
```

105 ms. The accept loop never waits for a handler — it spawns and immediately
accepts again. That one line is the entire difference between a server that
handles one client at a time and one that doesn't.

## Connection lifecycle

```
addr.listen(io, .{ .reuse_address = true })
                             -> Server     (socket bound + listening)
  server.accept(io)          -> Stream     (one connection; blocks)
    s.reader(io, &rbuf)      -> Reader     (.interface is a FIELD)
    s.writer(io, &wbuf)      -> Writer     (buffered — flush!)
    s.close(io)                             (defer it, first thing in the handler)
  server.deinit(io)                         (defer it in main)
```

| Concern | Production answer |
|---|---|
| unbounded connections | `Io.Semaphore` around the handler, or `concurrent_limit` on `Threaded` (12.2, 12.5) |
| slow-loris clients | `Io.Timeout` on reads, or race the handler with a clock (12.3) |
| graceful shutdown | flag + `Group.cancel(io)`; handlers see `error.Canceled` and their `defer s.close(io)` runs |
| handler panics | don't — return errors; a panic takes the whole process down |
| per-connection state | arena, as above |

Shutdown deserves one more look: because `accept`, `read` and `write` are all
cancellation points, `conns.cancel(io)` unblocks every handler *and* blocks
until they have all run their `defer`s. There is no "drain" phase to write by
hand.

## The interview answer

> `addr.listen(io, .{ .reuse_address = true })` gives a `Server`; `server.accept(io)` blocks and
> returns a `Stream`. The accept loop does nothing but accept — each
> connection goes to `io.concurrent` (not `io.async`, which may run inline and
> stall the loop), so the handler owns its socket, its buffers and its arena,
> and closes all of it with a `defer`. Reads and writes go through
> `stream.reader(io, &buf).interface` and `.writer(io, &buf).interface`, and
> the writer is buffered so every response ends in a `flush`. Shutdown is a
> `Group.cancel(io)`: every handler is parked in an `io` call, so they all get
> `error.Canceled`, run their cleanup, and `cancel` returns only once they are
> done.

---

Next: [Module 13 — Networking & HTTP](../13-networking-http/)
