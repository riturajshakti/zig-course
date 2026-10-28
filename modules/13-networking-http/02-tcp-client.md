# 13.2 — TCP Client

`connect` gives you a `net.Stream`. A `Stream` is not a reader or a writer; you
ask it for one, with a buffer you own — exactly like `std.Io.File` in module 8.

```zig
var stream = try addr.connect(io, .{ .mode = .stream });
defer stream.close(io);

var rbuf: [128]u8 = undefined;
var wbuf: [128]u8 = undefined;
var r = stream.reader(io, &rbuf);   // .interface is a FIELD
var w = stream.writer(io, &wbuf);
```

| Trap | Reality |
|---|---|
| `ConnectOptions.mode` | **no default** — `.{}` is a compile error |
| `stream.reader(io, &buf)` | returns a `Stream.Reader`; the `*Io.Reader` is `&r.interface` |
| writes | **buffered** — nothing leaves the socket until `flush()` |
| buffer size | the hard limit for one delimited read (`error.StreamTooLong` past it) |
| `close(io)` | takes `io`, returns `void`, cannot fail |

## A full round trip in one program

Server on `io.async`, client on the main task.

```zig
const std = @import("std");
const net = std.Io.net;

const port = 39243;

fn server(io: std.Io) void {
    const addr: net.IpAddress = .{ .ip4 = .loopback(port) };
    var listener = addr.listen(io, .{ .reuse_address = true }) catch return;
    defer listener.deinit(io);

    var stream = listener.accept(io) catch return;
    defer stream.close(io);

    var rbuf: [128]u8 = undefined;
    var wbuf: [128]u8 = undefined;
    var r = stream.reader(io, &rbuf);
    var w = stream.writer(io, &wbuf);

    const line = r.interface.takeDelimiterInclusive('\n') catch return;
    w.interface.print("echo: {s}", .{line}) catch return;
    w.interface.flush() catch return;
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;

    var task = io.async(server, .{io});
    defer task.await(io);
    io.sleep(.fromMilliseconds(50), .awake) catch {};

    const addr: net.IpAddress = .{ .ip4 = .loopback(port) };
    var stream = try addr.connect(io, .{ .mode = .stream });
    defer stream.close(io);

    var wbuf: [128]u8 = undefined;
    var w = stream.writer(io, &wbuf);
    try w.interface.writeAll("ping\n");
    try w.interface.flush();

    var rbuf: [128]u8 = undefined;
    var r = stream.reader(io, &rbuf);
    std.debug.print("{s}", .{try r.interface.takeDelimiterInclusive('\n')});
}
```

```
echo: ping
```

The `io.sleep(.fromMilliseconds(50), .awake)` is a crude start barrier so the
`listen` call lands before `connect`. A real client would retry on
`error.ConnectionRefused` instead.

## Forgetting `flush()` looks exactly like a hang

This is the #1 socket bug in 0.17. The same client, with and without the flush:

```zig
const std = @import("std");
const net = std.Io.net;

const port = 39245;

fn server(io: std.Io) void {
    const addr: net.IpAddress = .{ .ip4 = .loopback(port) };
    var listener = addr.listen(io, .{ .reuse_address = true }) catch return;
    defer listener.deinit(io);

    for (0..2) |_| {
        var stream = listener.accept(io) catch return;
        defer stream.close(io);
        var rbuf: [128]u8 = undefined;
        var r = stream.reader(io, &rbuf);
        if (r.interface.takeDelimiterInclusive('\n')) |line| {
            std.debug.print("server read: {s}", .{line});
        } else |err| {
            std.debug.print("server read: {t}\n", .{err});
        }
    }
}

fn sendLine(io: std.Io, do_flush: bool) !void {
    const addr: net.IpAddress = .{ .ip4 = .loopback(port) };
    var stream = try addr.connect(io, .{ .mode = .stream });
    defer stream.close(io);
    var wbuf: [128]u8 = undefined;
    var w = stream.writer(io, &wbuf);
    try w.interface.writeAll("with a flush\n");
    if (do_flush) try w.interface.flush();
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    var task = io.async(server, .{io});
    io.sleep(.fromMilliseconds(50), .awake) catch {};

    try sendLine(io, true);
    io.sleep(.fromMilliseconds(50), .awake) catch {};
    try sendLine(io, false);
    task.await(io);
}
```

```
server read: with a flush
server read: EndOfStream
```

No error on the client side. `writeAll` succeeded, `close` succeeded, exit code
0 — and the 13 bytes were discarded with the buffer. The peer saw a connection
that opened and closed with nothing in it.

Put the flush next to the writer and the problem disappears:

```zig
var w = stream.writer(io, &wbuf);
defer w.interface.flush() catch {};
```

...with the caveat from module 8: if knowing the bytes landed matters, also
call `try w.interface.flush()` explicitly, because `defer` swallows the error.

## Framing #1 — newline delimited

`takeDelimiterInclusive` returns a slice **into the reader's buffer**, up to and
including the delimiter.

```zig
const std = @import("std");
const net = std.Io.net;

const port = 39247;

fn server(io: std.Io) void {
    const addr: net.IpAddress = .{ .ip4 = .loopback(port) };
    var listener = addr.listen(io, .{ .reuse_address = true }) catch return;
    defer listener.deinit(io);
    var stream = listener.accept(io) catch return;
    defer stream.close(io);

    var wbuf: [64]u8 = undefined;
    var w = stream.writer(io, &wbuf);
    for (0..3) |i| w.interface.print("chunk {d}\n", .{i}) catch return;
    w.interface.flush() catch return;
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    var task = io.async(server, .{io});
    defer task.await(io);
    io.sleep(.fromMilliseconds(50), .awake) catch {};

    const addr: net.IpAddress = .{ .ip4 = .loopback(port) };
    var stream = try addr.connect(io, .{ .mode = .stream });
    defer stream.close(io);
    try stream.shutdown(io, .send);

    var rbuf: [16]u8 = undefined;
    var r = stream.reader(io, &rbuf);
    const all = try r.interface.allocRemaining(init.gpa, .limited(4096));
    defer init.gpa.free(all);
    std.debug.print("read {d} bytes until EOF:\n{s}", .{ all.len, all });
}
```

```
read 24 bytes until EOF:
chunk 0
chunk 1
chunk 2
```

Two things to notice. `shutdown(io, .send)` half-closes: the client promises to
send nothing more, so a server reading to EOF gets one, while the client can
still read the reply. And `allocRemaining` read 24 bytes through a **16-byte**
reader buffer — the buffer limits one delimited read, not the total.

| Loop until EOF | How |
|---|---|
| line at a time | `takeDelimiterInclusive('\n')`, `catch error.EndOfStream => break` |
| everything, heap | `allocRemaining(gpa, .limited(n))` |
| everything, straight through | `streamRemaining(other_writer)` |

Never use `takeDelimiterExclusive` in a `while ... else` loop: on an empty line
it spins instead of signalling EOF.

## Framing #2 — length prefixed

Binary protocols read an integer header, then exactly that many bytes. The
reader has `takeInt` / `take` for precisely this.

```zig
const std = @import("std");
const net = std.Io.net;

const port = 39246;
const messages = [_][]const u8{ "hello", "a longer frame", "" };

fn server(io: std.Io) void {
    const addr: net.IpAddress = .{ .ip4 = .loopback(port) };
    var listener = addr.listen(io, .{ .reuse_address = true }) catch return;
    defer listener.deinit(io);
    var stream = listener.accept(io) catch return;
    defer stream.close(io);

    var rbuf: [256]u8 = undefined;
    var r = stream.reader(io, &rbuf);
    for (0..messages.len) |_| {
        const len = r.interface.takeInt(u32, .big) catch return;
        const payload = r.interface.take(len) catch return;
        std.debug.print("frame len={d:<2} payload=\"{s}\"\n", .{ len, payload });
    }
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    var task = io.async(server, .{io});
    io.sleep(.fromMilliseconds(50), .awake) catch {};

    const addr: net.IpAddress = .{ .ip4 = .loopback(port) };
    var stream = try addr.connect(io, .{ .mode = .stream });
    defer stream.close(io);

    var wbuf: [256]u8 = undefined;
    var w = stream.writer(io, &wbuf);
    for (messages) |m| {
        try w.interface.writeInt(u32, @intCast(m.len), .big);
        try w.interface.writeAll(m);
    }
    try w.interface.flush();
    task.await(io);
}
```

```
frame len=5  payload="hello"
frame len=14 payload="a longer frame"
frame len=0  payload=""
```

The whole batch went out in **one** `flush`, so three frames arrived in one TCP
segment — and the reader still split them correctly, because the length prefix
says where each one ends. That is the point of framing: TCP gives you a byte
stream, not messages.

`.big` is network byte order. `take(len)` needs `len <= rbuf.len`, so a
length-prefixed reader's buffer must be as large as your largest frame — or
read in pieces with `streamExact64`.

| Reader method | For |
|---|---|
| `takeInt(T, .big)` | fixed-width header field |
| `take(n)` | exactly `n` bytes, borrowed from the buffer |
| `takeArray(n)` | `*[n]u8`, `n` comptime-known |
| `peek(n)` | look without consuming — decide the frame type first |
| `discardAll(n)` | skip a field you do not care about |

## Connection errors

```zig
const std = @import("std");
const net = std.Io.net;

pub fn main(init: std.process.Init) !void {
    const io = init.io;

    const closed: net.IpAddress = .{ .ip4 = .loopback(39244) };
    if (closed.connect(io, .{ .mode = .stream })) |stream| {
        stream.close(io);
        std.debug.print("connected (unexpected)\n", .{});
    } else |err| {
        std.debug.print("127.0.0.1:39244 -> {t}\n", .{err});
    }

    const missing: net.HostName = try .init("no-such-host.invalid");
    std.debug.print("bad DNS name    -> {any}\n", .{missing.connect(io, 80, .{ .mode = .stream })});
}
```

```
127.0.0.1:39244 -> ConnectionRefused
bad DNS name    -> error.UnknownHostName
```

| Error | Means |
|---|---|
| `ConnectionRefused` | nothing listening, or the kernel backlog is full |
| `ConnectionResetByPeer` | the peer sent RST — it died mid-connection |
| `NetworkUnreachable` / `HostUnreachable` | routing says no |
| `Timeout` | SYN went unanswered |
| `UnknownHostName` | DNS, not TCP — from `HostName.connect` |
| `ProcessFdQuotaExceeded` | you leaked sockets; `close(io)` every `Stream` |

**`ConnectOptions.timeout` is not implemented** in 0.17's `Threaded`
implementation — setting it panics with `TODO implement
netConnectIpPosix with timeout`. Until it lands, bound a connect by racing it
against `io.sleep` with `io.async`, or accept the kernel's default SYN timeout.

---

Next: [13.3 — TCP Server](./03-tcp-server.md)
