# 13.4 — HTTP Client

`std.http.Client` is a struct literal with two required fields and a connection
pool inside it.

```zig
var client: std.http.Client = .{ .allocator = gpa, .io = io };
defer client.deinit();
```

No `init`. The pool, the TLS CA bundle and the proxy config are all lazily
built, so `deinit()` is still mandatory.

## `fetch` — the one-shot path

```zig
const std = @import("std");

const port = 39252;
const url = std.fmt.comptimePrint("http://127.0.0.1:{d}/hello", .{port});

fn serveOne(io: std.Io) void {
    const addr: std.Io.net.IpAddress = .{ .ip4 = .loopback(port) };
    var listener = addr.listen(io, .{ .reuse_address = true }) catch return;
    defer listener.deinit(io);
    var stream = listener.accept(io) catch return;
    defer stream.close(io);

    var rbuf: [4096]u8 = undefined;
    var wbuf: [4096]u8 = undefined;
    var sr = stream.reader(io, &rbuf);
    var sw = stream.writer(io, &wbuf);
    var http: std.http.Server = .init(&sr.interface, &sw.interface);
    var req = http.receiveHead() catch return;
    req.respond("hi there", .{}) catch return;
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    var task = io.async(serveOne, .{io});
    defer task.await(io);
    io.sleep(.fromMilliseconds(50), .awake) catch {};

    var client: std.http.Client = .{ .allocator = init.gpa, .io = io };
    defer client.deinit();

    var body: std.Io.Writer.Allocating = .init(init.gpa);
    defer body.deinit();

    const res = try client.fetch(.{ .location = .{ .url = url }, .response_writer = &body.writer });
    std.debug.print("status  {d} {t}\n", .{ @intFromEnum(res.status), res.status });
    std.debug.print("body    {s}\n", .{body.written()});
}
```

```
status  200 ok
body    hi there
```

Three things to internalise:

- **`fetch` does not return a body.** `FetchResult` has exactly one field,
  `status`. The body goes wherever `response_writer` points, or is discarded.
- **`response_writer` is a `*std.Io.Writer`.** `Writer.Allocating` is the usual
  choice (`&body.writer` — the field is `writer`, not `interface`), but a
  `Writer.fixed(&buf)` or a file writer works identically.
- **`status` is an `enum(u10)`.** `@intFromEnum(res.status)` gives `200`,
  `{t}` gives `ok`. Comparisons are on the enum: `if (res.status == .ok)`.
  (`zig fmt` rewrites `@intFromEnum` to `@backingInt` — same builtin, newer
  spelling, so don't be surprised when your file changes under you.)

`std.fmt.comptimePrint` builds the URL from the port constant at comptime, so
there is one place to change the port.

## `FetchOptions`

| Field | Default | Use |
|---|---|---|
| `location` | required | `.{ .url = "http://..." }` or `.{ .uri = parsed }` |
| `method` | `GET`, or `POST` when `payload` is set | any `std.http.Method` |
| `payload` | `null` | the request body, as bytes |
| `response_writer` | `null` (discard) | where the response body goes |
| `extra_headers` | `&.{}` | survive a cross-domain redirect |
| `privileged_headers` | `&.{}` | **stripped** on a cross-domain redirect — put credentials here |
| `headers` | `.{}` | overrides for `host`, `user-agent`, `accept-encoding`, … |
| `redirect_behavior` | 3 hops | `.init(n)`, `.not_allowed`, `.unhandled` |
| `keep_alive` | `true` | pool the connection for reuse |
| `redirect_buffer` | heap | ≥8000 bytes recommended by RFC 9110 |

## Methods, headers, bodies

```zig
const std = @import("std");

const port = 39253;
const base = std.fmt.comptimePrint("http://127.0.0.1:{d}", .{port});

fn serve(io: std.Io, gpa: std.mem.Allocator, n: usize) void {
    const addr: std.Io.net.IpAddress = .{ .ip4 = .loopback(port) };
    var listener = addr.listen(io, .{ .reuse_address = true }) catch return;
    defer listener.deinit(io);

    for (0..n) |_| {
        var stream = listener.accept(io) catch return;
        defer stream.close(io);
        var rbuf: [4096]u8 = undefined;
        var wbuf: [4096]u8 = undefined;
        var sr = stream.reader(io, &rbuf);
        var sw = stream.writer(io, &wbuf);
        var http: std.http.Server = .init(&sr.interface, &sw.interface);
        var req = http.receiveHead() catch return;

        var seen: std.Io.Writer.Allocating = .init(gpa);
        defer seen.deinit();
        seen.writer.print("{t} {s}", .{ req.head.method, req.head.target }) catch return;

        var it = req.iterateHeaders();
        while (it.next()) |h| {
            if (std.ascii.startsWithIgnoreCase(h.name, "x-")) {
                seen.writer.print(" [{s}={s}]", .{ h.name, h.value }) catch return;
            }
        }

        var body_buf: [1024]u8 = undefined;
        const body_reader = req.readerExpectContinue(&body_buf) catch return;
        const payload = body_reader.allocRemaining(gpa, .limited(4096)) catch return;
        defer gpa.free(payload);
        if (payload.len > 0) seen.writer.print(" body=\"{s}\"", .{payload}) catch return;

        req.respond(seen.written(), .{
            .status = if (req.head.method == .POST) .created else .ok,
            .extra_headers = &.{.{ .name = "x-served-by", .value = "zig" }},
        }) catch return;
    }
}

fn call(client: *std.http.Client, gpa: std.mem.Allocator, opts: std.http.Client.FetchOptions) !void {
    var body: std.Io.Writer.Allocating = .init(gpa);
    defer body.deinit();
    var o = opts;
    o.response_writer = &body.writer;
    o.keep_alive = false;
    const res = try client.fetch(o);
    std.debug.print("{d} {t:<8} <- {s}\n", .{ @intFromEnum(res.status), res.status, body.written() });
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const gpa = init.gpa;

    var task = io.async(serve, .{ io, gpa, 4 });
    defer task.await(io);
    io.sleep(.fromMilliseconds(50), .awake) catch {};

    var client: std.http.Client = .{ .allocator = gpa, .io = io };
    defer client.deinit();

    try call(&client, gpa, .{ .location = .{ .url = base ++ "/users" } });
    try call(&client, gpa, .{ .location = .{ .url = base ++ "/users?page=2" }, .method = .PUT, .payload = "x" });
    try call(&client, gpa, .{
        .location = .{ .url = base ++ "/users" },
        .payload = "{\"name\":\"ada\"}",
    });
    try call(&client, gpa, .{
        .location = .{ .url = base ++ "/admin" },
        .method = .DELETE,
        .extra_headers = &.{ .{ .name = "x-api-key", .value = "s3cr3t" }, .{ .name = "x-trace", .value = "abc" } },
    });
}
```

```
200 ok       <- GET /users
200 ok       <- PUT /users?page=2 body="x"
201 created  <- POST /users body="{"name":"ada"}"
200 ok       <- DELETE /admin [x-api-key=s3cr3t] [x-trace=abc]
```

Note the third call sets no `method` and still arrives as a `POST` — `fetch`
infers `POST` the moment `payload` is non-null.

## The keep-alive deadlock

`o.keep_alive = false` in that program is **load-bearing**. Remove it and the
run hangs after the first response.

| | Client `keep_alive: true` (default) | Client `keep_alive: false` |
|---|---|---|
| Server closes after 1 request | **second `fetch` hangs** | works |
| Server loops `receiveHead` on one stream | works, 1 TCP connection | works, N connections |

With keep-alive the client returns the connection to its pool and the *next*
`fetch` to the same host reuses it — writing request #2 into a socket the server
already closed and is no longer reading. Nothing errors; both sides wait.

Pick one side and be consistent:

```zig
// client-side opt out
o.keep_alive = false;
```

```zig
// server-side: one stream, many requests
var http: std.http.Server = .init(&sr.interface, &sw.interface);
for (0..want) |_| {
    var req = http.receiveHead() catch return;
    req.respond("...", .{}) catch return;
}
```

Here is the server-side version working — three requests, **one** TCP connection:

```zig
const std = @import("std");

const port = 39254;
const base = std.fmt.comptimePrint("http://127.0.0.1:{d}", .{port});
const want = 3;

fn serve(io: std.Io) void {
    const addr: std.Io.net.IpAddress = .{ .ip4 = .loopback(port) };
    var listener = addr.listen(io, .{ .reuse_address = true }) catch return;
    defer listener.deinit(io);
    var stream = listener.accept(io) catch return;
    defer stream.close(io);

    var rbuf: [4096]u8 = undefined;
    var wbuf: [4096]u8 = undefined;
    var sr = stream.reader(io, &rbuf);
    var sw = stream.writer(io, &wbuf);
    var http: std.http.Server = .init(&sr.interface, &sw.interface);

    for (0..want) |i| {
        var req = http.receiveHead() catch return;
        var buf: [64]u8 = undefined;
        const text = std.fmt.bufPrint(&buf, "request {d} on one connection", .{i}) catch return;
        req.respond(text, .{}) catch return;
    }
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    var task = io.async(serve, .{io});
    defer task.await(io);
    io.sleep(.fromMilliseconds(50), .awake) catch {};

    var client: std.http.Client = .{ .allocator = init.gpa, .io = io };
    defer client.deinit();

    for (0..want) |_| {
        var body: std.Io.Writer.Allocating = .init(init.gpa);
        defer body.deinit();
        const res = try client.fetch(.{ .location = .{ .url = base ++ "/x" }, .response_writer = &body.writer });
        std.debug.print("{d} {s}\n", .{ @intFromEnum(res.status), body.written() });
    }
}
```

```
200 request 0 on one connection
200 request 1 on one connection
200 request 2 on one connection
```

Only one `accept` call in the whole program.

## Response headers need the lower-level path

`fetch` throws the head away. For headers, `content_type`, or the reason
phrase, use `client.request` → `sendBodiless` → `receiveHead`:

```zig
const std = @import("std");

const port = 39255;
const base = std.fmt.comptimePrint("http://127.0.0.1:{d}", .{port});

fn serve(io: std.Io) void {
    const addr: std.Io.net.IpAddress = .{ .ip4 = .loopback(port) };
    var listener = addr.listen(io, .{ .reuse_address = true }) catch return;
    defer listener.deinit(io);
    var stream = listener.accept(io) catch return;
    defer stream.close(io);

    var rbuf: [4096]u8 = undefined;
    var wbuf: [4096]u8 = undefined;
    var sr = stream.reader(io, &rbuf);
    var sw = stream.writer(io, &wbuf);
    var http: std.http.Server = .init(&sr.interface, &sw.interface);
    var req = http.receiveHead() catch return;
    req.respond("{\"id\":1}", .{
        .extra_headers = &.{
            .{ .name = "content-type", .value = "application/json" },
            .{ .name = "x-request-id", .value = "r-42" },
            .{ .name = "cache-control", .value = "no-store" },
        },
    }) catch return;
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    var task = io.async(serve, .{io});
    defer task.await(io);
    io.sleep(.fromMilliseconds(50), .awake) catch {};

    var client: std.http.Client = .{ .allocator = init.gpa, .io = io };
    defer client.deinit();

    const uri = try std.Uri.parse(base ++ "/items/1");
    var req = try client.request(.GET, uri, .{});
    defer req.deinit();
    try req.sendBodiless();

    var redirect_buf: [1024]u8 = undefined;
    var res = try req.receiveHead(&redirect_buf);

    std.debug.print("status         {d} {s}\n", .{ @intFromEnum(res.head.status), res.head.reason });
    std.debug.print("version        {t}\n", .{res.head.version});
    std.debug.print("content_type   {?s}\n", .{res.head.content_type});
    std.debug.print("content_length {?d}\n", .{res.head.content_length});
    std.debug.print("keep_alive     {any}\n", .{res.head.keep_alive});

    var it: std.http.HeaderIterator = .init(res.head.bytes);
    while (it.next()) |h| std.debug.print("  {s}: {s}\n", .{ h.name, h.value });

    var body_buf: [1024]u8 = undefined;
    const body = res.reader(&body_buf);
    const text = try body.allocRemaining(init.gpa, .limited(4096));
    defer init.gpa.free(text);
    std.debug.print("body           {s}\n", .{text});
}
```

```
status         200 OK
version        HTTP/1.1
content_type   application/json
content_length 8
keep_alive     true
  content-length: 8
  content-type: application/json
  x-request-id: r-42
  cache-control: no-store
body           {"id":1}
```

| Step | Call |
|---|---|
| parse a URL | `std.Uri.parse(text)` |
| open | `client.request(method, uri, .{})` — `defer req.deinit()` returns the connection to the pool |
| send, no body | `req.sendBodiless()` |
| send, with body | `req.sendBodyComplete(bytes)`, or `sendBody(&buf)` for a streaming `BodyWriter` |
| read head | `req.receiveHead(&redirect_buf)` |
| named fields | `res.head.status` / `.reason` / `.content_type` / `.content_length` |
| all headers | `std.http.HeaderIterator.init(res.head.bytes)` |
| read body | `res.reader(&buf)` → a `*std.Io.Reader` |
| decompress too | `res.readerDecompressing(&buf, &decomp_buf)` |

`res.head.bytes` is the raw head, so the iterator is a zero-copy walk over it.
Those slices point into the response reader's buffer and die when you start
reading the body — `dupe` anything you keep.

## Redirects

```zig
const std = @import("std");

const port = 39256;
const base = std.fmt.comptimePrint("http://127.0.0.1:{d}", .{port});

fn serve(io: std.Io, n: usize) void {
    const addr: std.Io.net.IpAddress = .{ .ip4 = .loopback(port) };
    var listener = addr.listen(io, .{ .reuse_address = true }) catch return;
    defer listener.deinit(io);

    for (0..n) |_| {
        var stream = listener.accept(io) catch return;
        defer stream.close(io);
        var rbuf: [4096]u8 = undefined;
        var wbuf: [4096]u8 = undefined;
        var sr = stream.reader(io, &rbuf);
        var sw = stream.writer(io, &wbuf);
        var http: std.http.Server = .init(&sr.interface, &sw.interface);
        var req = http.receiveHead() catch return;

        if (std.mem.eql(u8, req.head.target, "/old")) {
            req.respond("", .{
                .status = .found,
                .keep_alive = false,
                .extra_headers = &.{.{ .name = "location", .value = "/new" }},
            }) catch return;
        } else {
            req.respond("the real resource", .{ .keep_alive = false }) catch return;
        }
    }
}

fn get(client: *std.http.Client, gpa: std.mem.Allocator, label: []const u8, behavior: ?std.http.Client.Request.RedirectBehavior) !void {
    var body: std.Io.Writer.Allocating = .init(gpa);
    defer body.deinit();
    const res = client.fetch(.{
        .location = .{ .url = base ++ "/old" },
        .response_writer = &body.writer,
        .redirect_behavior = behavior,
        .keep_alive = false,
    });
    if (res) |ok| {
        std.debug.print("{s:<18} {d} \"{s}\"\n", .{ label, @intFromEnum(ok.status), body.written() });
    } else |err| {
        std.debug.print("{s:<18} {t}\n", .{ label, err });
    }
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    var task = io.async(serve, .{ io, 6 });
    defer task.await(io);
    io.sleep(.fromMilliseconds(50), .awake) catch {};

    var client: std.http.Client = .{ .allocator = init.gpa, .io = io };
    defer client.deinit();

    try get(&client, init.gpa, "default (follow)", null);
    try get(&client, init.gpa, "init(1)", .init(1));
    try get(&client, init.gpa, "not_allowed", .not_allowed);
    try get(&client, init.gpa, "unhandled", .unhandled);
}
```

```
default (follow)   200 "the real resource"
init(1)            200 "the real resource"
not_allowed        TooManyHttpRedirects
unhandled          302 ""
```

| `RedirectBehavior` | Effect |
|---|---|
| default (`null`) | follow up to 3 hops, then `TooManyHttpRedirects` |
| `.init(n)` | follow up to `n` |
| `.not_allowed` | the first 3xx is `error.TooManyHttpRedirects` |
| `.unhandled` | hand the 3xx back with its `location` header intact |

A request **with a payload** defaults to `.unhandled`, because re-sending a body
after a redirect is not something the client will silently do for you.

Each followed redirect is a fresh request, so the server above needs 6 accepts
for 4 client calls. `privileged_headers` exist for exactly this: an `authorization`
header you do *not* want replayed to whatever host the `location` points at.

## TLS and the public internet

`https://` works — `client.request` lazily scans the system CA bundle on the
first TLS request. Nothing in this module needs it, and every example here runs
against `127.0.0.1` so it works offline. Two knobs when you do go outbound:
`std.options.http_disable_tls` compiles TLS out entirely, and
`client.initDefaultProxies(arena, &environ_map)` picks up `http_proxy` /
`https_proxy` from the environment.

---

Next: [13.5 — HTTP Server](./05-http-server.md)
