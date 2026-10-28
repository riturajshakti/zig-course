# 13.1 — Addresses & Sockets

Everything network lives under `std.Io.net`. No `std.net` any more.

| Type | Is |
|---|---|
| `net.IpAddress` | `union(enum) { ip4: Ip4Address, ip6: Ip6Address }` |
| `net.Ip4Address` | `struct { bytes: [4]u8, port: u16 }` |
| `net.Socket` | a handle plus its `address` — connectionless (UDP) |
| `net.Stream` | a connected byte stream — gives you a reader and a writer |
| `net.Server` | a listening socket; `accept` returns a `Stream` |
| `net.HostName` | a *validated* host name; does DNS |
| `net.Protocol` | `enum(u32)` of IANA numbers — `.tcp = 6`, `.udp = 17` |

## Building addresses

```zig
const std = @import("std");
const net = std.Io.net;

pub fn main() !void {
    const list = [_]net.IpAddress{
        .{ .ip4 = .loopback(8080) },
        try net.IpAddress.parseIp4("10.0.0.7", 443),
        try net.IpAddress.parseIp6("::1", 9000),
        try net.IpAddress.parse("192.168.1.1", 53),
        try net.IpAddress.parse("fe80::1", 53),
        try net.IpAddress.parseLiteral("127.0.0.1:3000"),
        try net.IpAddress.parseLiteral("[2001:db8::5]:3000"),
    };

    var buf: [64]u8 = undefined;
    for (list) |addr| {
        const text = try std.fmt.bufPrint(&buf, "{f}", .{addr});
        std.debug.print("{t:<4} {s:<24} port={d}\n", .{ addr, text, addr.getPort() });
    }
}
```

```
ip4  127.0.0.1:8080           port=8080
ip4  10.0.0.7:443             port=443
ip6  [::1]:9000               port=9000
ip4  192.168.1.1:53           port=53
ip6  [fe80::1]:53             port=53
ip4  127.0.0.1:3000           port=3000
ip6  [2001:db8::5]:3000       port=3000
```

| Constructor | Takes | Notes |
|---|---|---|
| `.{ .ip4 = .loopback(p) }` | port | `127.0.0.1` — what you want in tests |
| `.{ .ip6 = .loopback(p) }` | port | `::1` |
| `parseIp4(text, port)` | address text **without** port | strict, IPv4 only |
| `parseIp6(text, port)` | address text **without** port | strict, IPv6 only |
| `parse(text, port)` | either family | tries v4, falls back to v6 |
| `parseLiteral(text)` | `"host:port"` | port is **inside** the string |
| `resolve(io, text, port)` | either family + `%scope` | needs `io` — an ioctl resolves the interface name |

`IpAddress` has a `format` method, so `{f}` prints `127.0.0.1:8080` /
`[::1]:9000`. Width is ignored on `{f}` — `bufPrint` first if you want a column.

## Ports and bytes

```zig
const std = @import("std");
const net = std.Io.net;

pub fn main() !void {
    var addr: net.IpAddress = .{ .ip4 = .loopback(0) };
    std.debug.print("start   {f}\n", .{addr});
    addr.setPort(39500);
    std.debug.print("setPort {f}  getPort={d}\n", .{ addr, addr.getPort() });

    switch (addr) {
        .ip4 => |v4| std.debug.print("bytes   {any}\n", .{v4.bytes}),
        .ip6 => |v6| std.debug.print("bytes   {any}\n", .{v6.bytes}),
    }

    const bad = [_][]const u8{ "999.1.1.1", "not-an-ip", "", "127.0.0.1" };
    for (bad) |text| {
        std.debug.print("parse(\"{s}\") -> ", .{text});
        if (net.IpAddress.parse(text, 80)) |ok| {
            std.debug.print("{f}\n", .{ok});
        } else |err| {
            std.debug.print("{t}\n", .{err});
        }
    }

    std.debug.print("parseLiteral(\"1.2.3.4\") -> {any}\n", .{net.IpAddress.parseLiteral("1.2.3.4")});
    std.debug.print("parseLiteral(\"::1\")     -> {any}\n", .{net.IpAddress.parseLiteral("::1")});
}
```

```
start   127.0.0.1:0
setPort 127.0.0.1:39500  getPort=39500
bytes   { 127, 0, 0, 1 }
parse("999.1.1.1") -> ParseFailed
parse("not-an-ip") -> ParseFailed
parse("") -> ParseFailed
parse("127.0.0.1") -> 127.0.0.1:80
parseLiteral("1.2.3.4") -> .{ .ip4 = .{ .bytes = { 1, 2, 3, 4 }, .port = 0 } }
parseLiteral("::1")     -> error.InvalidAddress
```

Two things worth knowing:

- `parse` collapses every v4 and v6 failure into a single **`ParseFailed`**.
  Use `parseIp4` directly if you want the precise error (`NonCanonical`,
  `Incomplete`, `Overflow`, …).
- `parseLiteral` **requires brackets** for IPv6, even without a port —
  `"::1"` fails, `"[::1]"` works. `"1.2.3.4"` with no colon gets port 0.

`getPort`/`setPort` are native-endian; the byte-swap to network order happens
inside the `io` implementation, not in your code.

## `Socket.Mode` — stream vs datagram

`Mode` is the socket type, `Protocol` is the wire protocol.

| `Socket.Mode` | Protocol default | Gives you |
|---|---|---|
| `.stream` | `.tcp` | ordered reliable bytes — `listen`/`connect` → `Stream` |
| `.dgram` | `.udp` | packets — `bind` → `Socket`, then `send`/`receive` |
| `.seqpacket` | — | reliable ordered *records* |
| `.raw` | — | raw protocol access |
| `.rdm` | — | reliable datagrams, unordered |

`listen` only accepts connection-oriented modes. `.dgram` there is a
`error.SocketModeUnsupported` — lesson 13.3 shows it.

### UDP: `bind` + `send` + `receive`

```zig
const std = @import("std");
const net = std.Io.net;

const server_port = 39241;
const client_port = 39242;

fn receiveOne(io: std.Io) void {
    const addr: net.IpAddress = .{ .ip4 = .loopback(server_port) };
    const sock = addr.bind(io, .{ .mode = .dgram }) catch return;
    defer sock.close(io);

    var buf: [256]u8 = undefined;
    const msg = sock.receive(io, &buf) catch return;
    std.debug.print("recv {d} bytes from {f}: {s}\n", .{ msg.data.len, msg.from, msg.data });
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;

    var task = io.async(receiveOne, .{io});
    io.sleep(.fromMilliseconds(50), .awake) catch {};

    const mine: net.IpAddress = .{ .ip4 = .loopback(client_port) };
    const sock = try mine.bind(io, .{ .mode = .dgram });
    defer sock.close(io);

    const dest: net.IpAddress = .{ .ip4 = .loopback(server_port) };
    try sock.send(io, &dest, "one datagram, no connection");
    std.debug.print("sent to {f}\n", .{dest});
    task.await(io);
}
```

```
sent to 127.0.0.1:39241
recv 27 bytes from 127.0.0.1:39242: one datagram, no connection
```

`receive` returns an `IncomingMessage`: `.from` (who sent it), `.data` (a slice
**into your buffer**), and `.flags` — check `.flags.trunc` to learn the
datagram was larger than the buffer you supplied. There is no reader/writer
here; a datagram socket has no stream to buffer.

`BindOptions.mode` has **no default** — omitting it is a compile error, same as
`ConnectOptions.mode` in the next lesson.

## DNS: `net.HostName`

`HostName` is a validated name, not a string. Construct it with `init`, which
checks RFC 1123 rules up front.

```zig
const std = @import("std");
const net = std.Io.net;

pub fn main(init: std.process.Init) !void {
    const io = init.io;

    const host: net.HostName = try .init("localhost");
    var slots: [16]net.HostName.LookupResult = undefined;
    var q: std.Io.Queue(net.HostName.LookupResult) = .init(&slots);

    var task = io.async(net.HostName.lookup, .{ host, io, &q, net.HostName.LookupOptions{ .port = 80 } });
    defer task.await(io) catch {};

    while (q.getOne(io)) |result| switch (result) {
        .address => |a| std.debug.print("address        {f}\n", .{a}),
        .canonical_name => |n| std.debug.print("canonical_name {s}\n", .{n.bytes}),
    } else |_| {}

    std.debug.print("validate(\"ok-host.example\") {any}\n", .{net.HostName.validate("ok-host.example")});
    std.debug.print("validate(\"bad_host\")        {any}\n", .{net.HostName.validate("bad_host")});
}
```

```
address        [::1]:80
address        127.0.0.1:80
validate("ok-host.example") void
validate("bad_host")        error.InvalidHostName
```

`lookup` is **streaming**: it pushes zero or more `.address` results plus
exactly one `.canonical_name` into an `std.Io.Queue` you own, and closes the
queue when done. That is why it must run under `io.async` — a single blocking
call could not hand you results as they arrive. `error.Closed` from `getOne` is
the normal end of the loop, which is why the `else |_| {}` is there.

Underscores are illegal in host names — `bad_host` fails validation before any
DNS traffic happens.

### Connecting by name

`HostName.connect` fans out across every resolved address concurrently and
returns the first one that answers — "happy eyeballs", built in.

```zig
const std = @import("std");
const net = std.Io.net;

const port = 39240;

fn serveOne(io: std.Io) void {
    const addr: net.IpAddress = .{ .ip4 = .loopback(port) };
    var server = addr.listen(io, .{ .reuse_address = true }) catch return;
    defer server.deinit(io);
    var stream = server.accept(io) catch return;
    defer stream.close(io);
    var buf: [64]u8 = undefined;
    var w = stream.writer(io, &buf);
    w.interface.writeAll("hello from localhost\n") catch return;
    w.interface.flush() catch return;
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;

    var task = io.async(serveOne, .{io});
    defer task.await(io);
    io.sleep(.fromMilliseconds(50), .awake) catch {};

    const host: net.HostName = try .init("localhost");
    var stream = try host.connect(io, port, .{ .mode = .stream });
    defer stream.close(io);

    var rbuf: [64]u8 = undefined;
    var r = stream.reader(io, &rbuf);
    std.debug.print("via DNS name: {s}", .{try r.interface.takeDelimiterInclusive('\n')});
}
```

```
via DNS name: hello from localhost
```

| Need | Call |
|---|---|
| name → addresses | `host.lookup(io, &queue, .{ .port = p })` |
| name → one connected stream | `host.connect(io, port, .{ .mode = .stream })` |
| name → every stream | `host.connectMany(io, port, &queue, opts)` |
| name from a URL | `net.HostName.fromUri(uri, &buf)` |
| validity check only | `net.HostName.validate(bytes)` |

## IPv4 vs IPv6

```zig
const std = @import("std");
const net = std.Io.net;

pub fn main() !void {
    const v4: net.IpAddress = .{ .ip4 = .loopback(80) };
    const v6: net.IpAddress = .{ .ip6 = .loopback(80) };
    std.debug.print("v4  {f}  size={d}\n", .{ v4, @sizeOf(net.Ip4Address) });
    std.debug.print("v6  {f}  size={d}\n", .{ v6, @sizeOf(net.Ip6Address) });
    std.debug.print("union size={d}  Family={t}\n", .{ @sizeOf(net.IpAddress), @as(net.IpAddress.Family, v6) });

    const mapped = try net.IpAddress.parseIp6("::ffff:192.0.2.128", 443);
    std.debug.print("mapped parsed as {t}: {f}\n", .{ mapped, mapped });
    std.debug.print("fromIp6 normalises to {t}: {f}\n", .{
        net.IpAddress.fromIp6(mapped.ip6),
        net.IpAddress.fromIp6(mapped.ip6),
    });
}
```

```
v4  127.0.0.1:80  size=6
v6  [::1]:80  size=28
union size=32  Family=ip6
mapped parsed as ip6: [::ffff:192.0.2.128]:443
fromIp6 normalises to ip4: 192.0.2.128:443
```

`IpAddress.fromIp6` collapses an IPv4-mapped v6 address back to `.ip4`, which
is what you want before comparing a client address against a v4 allowlist. A
listener bound to `::` can accept v4 clients and hand you mapped addresses, so
normalise before you compare.

---

Next: [13.2 — TCP Client](./02-tcp-client.md)
