# 13.3 — TCP Server

Three calls, and you own everything in between.

```zig
var listener = try addr.listen(io, .{});   // bind + listen in one
defer listener.deinit(io);
var stream = try listener.accept(io);      // BLOCKS until a client arrives
defer stream.close(io);
```

`listen` does the `socket`+`bind`+`listen` sequence for you — there is no
separate `bind` step for stream sockets. `deinit(io)` closes the listening
socket (and invalidates the `Server`); `close(io)` closes one connection.

## `ListenOptions`

| Field | Default | Does |
|---|---|---|
| `kernel_backlog` | 128 | pending connections the kernel queues for you; past it, clients see `ConnectionRefused` |
| `reuse_address` | `false` | sets `SO_REUSEADDR` **and** `SO_REUSEPORT` on POSIX |
| `mode` | `.stream` | only connection-oriented modes allowed |
| `protocol` | `.tcp` | `.sctp`, `.dccp`, `.tp` also valid |

```zig
const std = @import("std");
const net = std.Io.net;

fn tryListen(io: std.Io, label: []const u8, addr: net.IpAddress, opts: net.IpAddress.ListenOptions) void {
    if (addr.listen(io, opts)) |server| {
        var s = server;
        defer s.deinit(io);
        std.debug.print("{s} -> ok\n", .{label});
    } else |err| {
        std.debug.print("{s} -> {t}\n", .{ label, err });
    }
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const addr: net.IpAddress = .{ .ip4 = .loopback(39249) };

    var held = try addr.listen(io, .{ .reuse_address = true });
    defer held.deinit(io);
    std.debug.print("first,  reuse_address=true  -> ok (held open)\n", .{});

    tryListen(io, "second, reuse_address=true ", addr, .{ .reuse_address = true });
    tryListen(io, "second, reuse_address=false", addr, .{});
    tryListen(io, "127.0.0.1:80               ", .{ .ip4 = .loopback(80) }, .{});
    tryListen(io, "mode=.dgram                ", .{ .ip4 = .loopback(39250) }, .{ .mode = .dgram });
}
```

```
first,  reuse_address=true  -> ok (held open)
second, reuse_address=true  -> ok
second, reuse_address=false -> AddressInUse
127.0.0.1:80                -> AccessDenied
mode=.dgram                 -> SocketModeUnsupported
```

Read that second line carefully. `reuse_address: true` sets `SO_REUSEPORT`, so a
**second live listener binds the same port** and the kernel load-balances new
connections between them. That is a feature (zero-downtime restarts, one
listener per core) and a foot-gun (a stale process silently steals half your
traffic). `false` gives you the `AddressInUse` you expected.

## No `localAddress()`

`net.Socket` has an `address` field, but there is no call in 0.17 to ask the
kernel what it actually bound. Bind port 0 and you can never learn which
ephemeral port you got — so **pick a fixed port** in tests and examples, or
pass the port in from config.

## Serving N connections, then returning

A server in a lesson must terminate. Count and stop:

```zig
const std = @import("std");
const net = std.Io.net;

const port = 39248;
const want = 3;

fn serve(io: std.Io) usize {
    const addr: net.IpAddress = .{ .ip4 = .loopback(port) };
    var listener = addr.listen(io, .{ .reuse_address = true, .kernel_backlog = 16 }) catch return 0;
    defer listener.deinit(io);

    var served: usize = 0;
    while (served < want) : (served += 1) {
        var stream = listener.accept(io) catch break;
        defer stream.close(io);

        var rbuf: [128]u8 = undefined;
        var wbuf: [128]u8 = undefined;
        var r = stream.reader(io, &rbuf);
        var w = stream.writer(io, &wbuf);

        const line = r.interface.takeDelimiterInclusive('\n') catch continue;
        w.interface.print("#{d} {s}", .{ served, line }) catch continue;
        w.interface.flush() catch continue;
    }
    return served;
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    var task = io.async(serve, .{io});
    io.sleep(.fromMilliseconds(50), .awake) catch {};

    const addr: net.IpAddress = .{ .ip4 = .loopback(port) };
    for (0..want) |i| {
        var stream = try addr.connect(io, .{ .mode = .stream });
        defer stream.close(io);

        var wbuf: [128]u8 = undefined;
        var w = stream.writer(io, &wbuf);
        try w.interface.print("client {d}\n", .{i});
        try w.interface.flush();

        var rbuf: [128]u8 = undefined;
        var r = stream.reader(io, &rbuf);
        std.debug.print("reply: {s}", .{try r.interface.takeDelimiterInclusive('\n')});
    }
    std.debug.print("server served {d}, loop ended, process exits\n", .{task.await(io)});
}
```

```
reply: #0 client 0
reply: #1 client 1
reply: #2 client 2
server served 3, loop ended, process exits
```

Notice `io.async(serve, .{io})` returns an `std.Io.Future(usize)` because
`serve` returns a `usize` — `task.await(io)` hands the value back. A `void`
function gives you `Future(void)` and `await` returns nothing.

This loop is **serial**: one connection is read, answered and closed before the
next `accept`. Fine for a lesson, useless for a service — one slow client
blocks everybody.

## Connection lifecycle

| Stage | Call | Notes |
|---|---|---|
| bind + listen | `addr.listen(io, opts)` | one call |
| wait | `listener.accept(io)` | blocks; `error.SocketNotListening` after shutdown |
| read | `stream.reader(io, &buf)` | buffer sized for your longest frame |
| write | `stream.writer(io, &buf)` | **flush** |
| half close | `stream.shutdown(io, .send)` | signal EOF, keep reading |
| close one client | `stream.close(io)` | cannot fail |
| stop listening | `listener.deinit(io)` | makes a blocked `accept` return `SocketNotListening` |

`shutdown` on the **listener's** socket is the documented cancellation
mechanism: another task calls it, and the `accept` that is blocked right now
returns `error.SocketNotListening` instead of hanging forever.

## One task per connection

Hand each accepted `Stream` to its own `io.async` and the `accept` loop goes
straight back to waiting.

```zig
const std = @import("std");
const net = std.Io.net;

const port = 39251;
const want = 4;

fn handle(io: std.Io, stream: net.Stream, id: usize) void {
    defer stream.close(io);
    var rbuf: [128]u8 = undefined;
    var wbuf: [128]u8 = undefined;
    var r = stream.reader(io, &rbuf);
    var w = stream.writer(io, &wbuf);
    const line = r.interface.takeDelimiterInclusive('\n') catch return;
    w.interface.print("handler {d} saw {s}", .{ id, line }) catch return;
    w.interface.flush() catch return;
}

fn serve(io: std.Io) void {
    const addr: net.IpAddress = .{ .ip4 = .loopback(port) };
    var listener = addr.listen(io, .{ .reuse_address = true }) catch return;
    defer listener.deinit(io);

    var tasks: [want]std.Io.Future(void) = undefined;
    var n: usize = 0;
    while (n < want) : (n += 1) {
        const stream = listener.accept(io) catch break;
        tasks[n] = io.async(handle, .{ io, stream, n });
    }
    for (tasks[0..n]) |*t| t.await(io);
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    var server_task = io.async(serve, .{io});
    io.sleep(.fromMilliseconds(50), .awake) catch {};

    const addr: net.IpAddress = .{ .ip4 = .loopback(port) };
    for (0..want) |i| {
        var stream = try addr.connect(io, .{ .mode = .stream });
        defer stream.close(io);
        var wbuf: [128]u8 = undefined;
        var w = stream.writer(io, &wbuf);
        try w.interface.print("req {d}\n", .{i});
        try w.interface.flush();
        var rbuf: [128]u8 = undefined;
        var r = stream.reader(io, &rbuf);
        std.debug.print("{s}", .{try r.interface.takeDelimiterInclusive('\n')});
    }
    server_task.await(io);
}
```

```
handler 0 saw req 0
handler 1 saw req 1
handler 2 saw req 2
handler 3 saw req 3
```

Two details that matter in real code:

- **The handler owns the stream.** `accept` returns a value, the handler takes
  it by value and `defer stream.close(io)` lives in the handler — not in the
  accept loop. Closing it in both is a double close.
- **Each handler declares its own buffers.** They are on the handler task's
  stack, so two connections never share a read buffer. A single shared buffer
  hoisted out of the loop would corrupt under concurrency.

The fixed `tasks` array is the lesson-sized version of a real bound: a service
needs a connection limit anyway, and an array of `Future` plus a free-slot
search is a perfectly good one. Unbounded `io.async` per connection is how you
get a fork bomb from a port scan.

---

Next: [13.4 — HTTP Client](./04-http-client.md)
