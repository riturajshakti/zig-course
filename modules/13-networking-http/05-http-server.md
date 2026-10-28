# 13.5 — HTTP Server

`std.http.Server` is a **head parser plus a response writer**. That is all. It
does not own a socket, does not accept connections, does not route.

```zig
var sr = stream.reader(io, &rbuf);          // net.Stream.Reader — you own rbuf
var sw = stream.writer(io, &wbuf);          // net.Stream.Writer — you own wbuf
var http: std.http.Server = .init(&sr.interface, &sw.interface);
```

`init` takes a `*std.Io.Reader` and a `*std.Io.Writer`. Nothing in that
signature mentions sockets — hand it a file reader, a `Writer.fixed`, or a
pipe, and it parses HTTP out of that instead. The socket is your business; the
protocol is its business.

## Buffer ownership

| Buffer | Owned by | Sized for |
|---|---|---|
| `rbuf` (stream reader) | you | the **largest request head** you will accept |
| `wbuf` (stream writer) | you | response head + small bodies; it just drains more often if small |
| body buffer (`readerExpectContinue(&buf)`) | you, per request | a chunk of the request body, not all of it |
| streaming buffer (`respondStreaming(&buf, …)`) | you | one chunk of the response |

The head must fit in `rbuf` in its entirety, because `Head.parse` works on a
contiguous slice of it. Undersize it and you get an explicit error, not a
truncation:

```zig
const std = @import("std");

const port = 39262;
const base = std.fmt.comptimePrint("http://127.0.0.1:{d}", .{port});

fn serve(io: std.Io, read_buf: []u8) void {
    const addr: std.Io.net.IpAddress = .{ .ip4 = .loopback(port) };
    var listener = addr.listen(io, .{ .reuse_address = true }) catch return;
    defer listener.deinit(io);
    var stream = listener.accept(io) catch return;
    defer stream.close(io);

    var wbuf: [4096]u8 = undefined;
    var sr = stream.reader(io, read_buf);
    var sw = stream.writer(io, &wbuf);
    var http: std.http.Server = .init(&sr.interface, &sw.interface);

    if (http.receiveHead()) |r| {
        var req = r;
        std.debug.print("server: head parsed, target={s}\n", .{req.head.target});
        req.respond("ok", .{}) catch return;
    } else |err| {
        std.debug.print("server: receiveHead -> {t}\n", .{err});
    }
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    var tiny: [48]u8 = undefined;

    var task = io.async(serve, .{ io, &tiny });
    defer task.await(io);
    io.sleep(.fromMilliseconds(50), .awake) catch {};

    var client: std.http.Client = .{ .allocator = init.gpa, .io = io };
    defer client.deinit();

    var body: std.Io.Writer.Allocating = .init(init.gpa);
    defer body.deinit();
    const res = client.fetch(.{
        .location = .{ .url = base ++ "/a/fairly/long/path/for/a/48/byte/buffer" },
        .response_writer = &body.writer,
    });
    std.debug.print("client: {any}\n", .{res});
}
```

```
server: receiveHead -> HttpHeadersOversize
client: error.ReadFailed
```

4 KiB handles ordinary browser traffic; 8 KiB is the usual production figure
(and the one RFC 9110 suggests for the client's redirect buffer too). That
number **is** your maximum header size limit — there is no separate config for
it, which also means a hostile client cannot make you allocate.

## `receiveHead` and `req.head`

```zig
var req = try http.receiveHead();
```

| `req.head` field | Type | Note |
|---|---|---|
| `.method` | `std.http.Method` | enum: `.GET`, `.POST`, …; `{t}` prints `GET` |
| `.target` | `[]const u8` | the raw request target — path **and** query, undecoded |
| `.version` | `std.http.Version` | `.@"HTTP/1.1"` |
| `.content_type` | `?[]const u8` | |
| `.content_length` | `?u64` | `null` on chunked or bodiless |
| `.transfer_encoding` | `.none` / `.chunked` | |
| `.keep_alive` | `bool` | what the client asked for |
| `.expect` | `?[]const u8` | `"100-continue"` if present |

`Method` carries real behaviour, not just a tag:
`.requestHasBody()`, `.responseHasBody()`, `.safe()`, `.idempotent()`,
`.cacheable()`. A generic handler can branch on those instead of listing verbs.

**`req.head`'s slices point into `rbuf`, and are invalidated the moment you
initialise the request body stream.** Copy `target` before calling
`readerExpectContinue`, or you will read freed memory — see 13.6 for the
segfault that teaches you this.

## There is no router — you write the switch

```zig
const std = @import("std");

const port = 39257;
const base = std.fmt.comptimePrint("http://127.0.0.1:{d}", .{port});

fn route(req: *std.http.Server.Request) !void {
    const target = req.head.target;
    const path = if (std.mem.findScalar(u8, target, '?')) |i| target[0..i] else target;

    if (req.head.method == .GET and std.mem.eql(u8, path, "/")) {
        return req.respond("root\n", .{});
    }
    if (req.head.method == .GET and std.mem.eql(u8, path, "/health")) {
        return req.respond("{\"ok\":true}", .{
            .extra_headers = &.{.{ .name = "content-type", .value = "application/json" }},
        });
    }
    if (std.mem.startsWith(u8, path, "/users/")) {
        const id = path["/users/".len..];
        if (id.len == 0) return req.respond("missing id\n", .{ .status = .bad_request });
        var buf: [64]u8 = undefined;
        return req.respond(try std.fmt.bufPrint(&buf, "user {s}\n", .{id}), .{});
    }
    if (std.mem.eql(u8, path, "/teapot")) {
        return req.respond("short and stout\n", .{ .status = .teapot });
    }
    return req.respond("no route\n", .{ .status = .not_found });
}

fn serve(io: std.Io, n: usize) void {
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

    for (0..n) |_| {
        var req = http.receiveHead() catch return;
        route(&req) catch return;
    }
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const paths = [_][]const u8{ "/", "/health", "/users/42", "/teapot", "/nope", "/users/" };

    var task = io.async(serve, .{ io, paths.len });
    defer task.await(io);
    io.sleep(.fromMilliseconds(50), .awake) catch {};

    var client: std.http.Client = .{ .allocator = init.gpa, .io = io };
    defer client.deinit();

    for (paths) |p| {
        var url_buf: [128]u8 = undefined;
        const url = try std.fmt.bufPrint(&url_buf, base ++ "{s}", .{p});
        var body: std.Io.Writer.Allocating = .init(init.gpa);
        defer body.deinit();
        const res = try client.fetch(.{ .location = .{ .url = url }, .response_writer = &body.writer });
        const trimmed = std.mem.trimEnd(u8, body.written(), "\n");
        std.debug.print("{s:<10} {d} {t:<12} {s}\n", .{ p, @intFromEnum(res.status), res.status, trimmed });
    }
}
```

```
/          200 ok           root
/health    200 ok           {"ok":true}
/users/42  200 ok           user 42
/teapot    418 teapot       short and stout
/nope      404 not_found    no route
/users/    400 bad_request  missing id
```

`target` is **raw**: `/users?page=2` arrives with the query attached and
`%20` undecoded. Splitting on `'?'` is yours to do, as is percent-decoding
(`std.Uri.percentDecodeInPlace`) and path-traversal rejection. There is no
framework quietly doing it first.

Six requests, one `accept`, one `receiveHead` loop — the keep-alive shape from
13.4.

## `respond`

```zig
try req.respond(content, .{
    .status = .created,
    .reason = null,
    .keep_alive = true,
    .extra_headers = &.{.{ .name = "content-type", .value = "application/json" }},
    .transfer_encoding = null,
});
```

| Option | Default | Notes |
|---|---|---|
| `status` | `.ok` | any `std.http.Status`; asserts it is not `.@"continue"` |
| `reason` | `null` | `null` → the enum's standard phrase |
| `keep_alive` | `true` | `false` sends `connection: close` |
| `extra_headers` | `&.{}` | asserts no `:` in a name and no `\r\n` anywhere |
| `transfer_encoding` | `null` | `null` → `content-length` is computed for you |
| `version` | `.@"HTTP/1.1"` | |

`respond` writes the head, writes the body, discards any unread request body,
and flushes. `respondUnflushed` does the same without the flush — use it to
batch several pipelined responses into one syscall, then flush once yourself.

`respond` also handles `HEAD` for you by omitting the body while keeping the
`content-length`, and answers an `expect: 100-continue` before reading.

## `respondStreaming` — chunked, or a known length

For a body you generate incrementally, `respondStreaming` returns an
`http.BodyWriter`:

```zig
const std = @import("std");

const port = 39258;
const base = std.fmt.comptimePrint("http://127.0.0.1:{d}", .{port});

fn serve(io: std.Io, n: usize) void {
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

    for (0..n) |_| {
        var req = http.receiveHead() catch return;
        var chunk_buf: [64]u8 = undefined;
        var body = req.respondStreaming(&chunk_buf, .{
            .respond_options = .{
                .extra_headers = &.{.{ .name = "content-type", .value = "text/plain" }},
            },
        }) catch return;
        for (0..4) |i| {
            body.writer.print("row {d}\n", .{i}) catch return;
            body.flush() catch return;
        }
        body.end() catch return;
    }
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    var task = io.async(serve, .{ io, 1 });
    defer task.await(io);
    io.sleep(.fromMilliseconds(50), .awake) catch {};

    var client: std.http.Client = .{ .allocator = init.gpa, .io = io };
    defer client.deinit();

    const uri = try std.Uri.parse(base ++ "/stream");
    var req = try client.request(.GET, uri, .{});
    defer req.deinit();
    try req.sendBodiless();
    var rb: [1024]u8 = undefined;
    var res = try req.receiveHead(&rb);

    std.debug.print("transfer_encoding {t}\n", .{res.head.transfer_encoding});
    std.debug.print("content_length    {?d}\n", .{res.head.content_length});

    var body_buf: [256]u8 = undefined;
    const r = res.reader(&body_buf);
    while (r.takeDelimiterInclusive('\n')) |line| {
        std.debug.print("  got {s}", .{line});
    } else |err| switch (err) {
        error.EndOfStream => {},
        else => return err,
    }
}
```

```
transfer_encoding chunked
content_length    null
  got row 0
  got row 1
  got row 2
  got row 3
```

| You want | Pass |
|---|---|
| chunked (length unknown) | `.{}` — the default |
| `content-length: N` | `.{ .content_length = N }` |
| no framing at all (close signals end) | `.{ .respond_options = .{ .transfer_encoding = .none } }` |

| `BodyWriter` call | Does |
|---|---|
| `body.writer` | the `std.Io.Writer` you write into |
| `body.flush()` | emit the buffered bytes as one chunk — this is what makes it streaming |
| `body.end()` | terminating `0\r\n\r\n` (or verify the content-length); **mandatory** |
| `body.elide` | `true` on a `HEAD` request — skip the expensive work |

Without the per-iteration `flush()` you get the same four rows in one chunk.
`end()` is not optional: skip it and a chunked client waits forever for the
terminator.

## Reading a request body

```zig
const std = @import("std");

const port = 39259;
const base = std.fmt.comptimePrint("http://127.0.0.1:{d}", .{port});

fn serve(io: std.Io, gpa: std.mem.Allocator, n: usize) void {
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

    for (0..n) |_| {
        var req = http.receiveHead() catch return;

        var head: std.Io.Writer.Allocating = .init(gpa);
        defer head.deinit();
        head.writer.print("method={t} target={s} version={t} ct={?s} len={?d}\n", .{
            req.head.method, req.head.target, req.head.version,
            req.head.content_type, req.head.content_length,
        }) catch return;

        var body_buf: [1024]u8 = undefined;
        const reader = req.readerExpectContinue(&body_buf) catch return;
        const payload = reader.allocRemaining(gpa, .limited(64 * 1024)) catch return;
        defer gpa.free(payload);
        head.writer.print("body({d})={s}", .{ payload.len, payload }) catch return;

        req.respond(head.written(), .{}) catch return;
    }
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const gpa = init.gpa;

    var task = io.async(serve, .{ io, gpa, 3 });
    defer task.await(io);
    io.sleep(.fromMilliseconds(50), .awake) catch {};

    var client: std.http.Client = .{ .allocator = gpa, .io = io };
    defer client.deinit();

    const calls = [_]std.http.Client.FetchOptions{
        .{ .location = .{ .url = base ++ "/empty" } },
        .{ .location = .{ .url = base ++ "/create" }, .payload = "{\"name\":\"ada\"}" },
        .{
            .location = .{ .url = base ++ "/patch" },
            .method = .PATCH,
            .payload = "a=1&b=2",
            .extra_headers = &.{.{ .name = "content-type", .value = "application/x-www-form-urlencoded" }},
        },
    };

    for (calls) |c| {
        var body: std.Io.Writer.Allocating = .init(gpa);
        defer body.deinit();
        var o = c;
        o.response_writer = &body.writer;
        const res = try client.fetch(o);
        std.debug.print("--- {d}\n{s}\n", .{ @intFromEnum(res.status), body.written() });
    }
}
```

```
--- 200
method=GET target=/empty version=HTTP/1.1 ct=null len=null
body(0)=
--- 200
method=POST target=/create version=HTTP/1.1 ct=null len=14
body(14)={"name":"ada"}
--- 200
method=PATCH target=/patch version=HTTP/1.1 ct=application/x-www-form-urlencoded len=7
body(7)=a=1&b=2
```

Everything from `req.head` is read into `head` **before** the body reader is
created — that ordering is deliberate, see the invalidation warning above.

| Call | For |
|---|---|
| `req.readerExpectContinue(&buf)` | answers `expect: 100-continue`, then gives you a `*Io.Reader` |
| `req.readerExpectNone(&buf)` | assumes no `expect` header; cannot fail |
| `.allocRemaining(gpa, .limited(n))` | the whole body, bounded — **always bound it** |
| `.streamRemaining(w)` | straight into a writer, no intermediate allocation |
| `.takeDelimiterInclusive('\n')` | line at a time out of a large upload |

A bodiless `GET` is not a special case: the reader returns 0 bytes. And the
limit on `allocRemaining` is your request-size cap — a server without one is a
memory-exhaustion bug waiting for one `content-length: 99999999999`.

## Status codes

`std.http.Status` is an `enum(u10)` with the IANA registry in it, so the tag
name documents the handler: `.ok`, `.created`, `.no_content`, `.bad_request`,
`.unauthorized`, `.forbidden`, `.not_found`, `.method_not_allowed`, `.conflict`,
`.unprocessable_entity`, `.too_many_requests`, `.internal_server_error`,
`.service_unavailable`, and yes, `.teapot`.

| Need | Expression |
|---|---|
| the number | `@intFromEnum(status)` |
| the tag | `{t}` → `not_found` |
| the standard phrase | `status.phrase()` → `?[]const u8` |
| the class | `status.class()` → `.success`, `.client_error`, … |
| from a number | `@enumFromInt(code)` |

---

Next: [13.6 — Serving JSON](./06-serving-json.md)
