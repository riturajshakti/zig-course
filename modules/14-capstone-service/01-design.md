# 14.1 — Design & Layout

## The API

| Method | Path | Success | Failure |
|---|---|---|---|
| `GET` | `/health` | `200 {"status":"ok"}` | — |
| `GET` | `/items` | `200 [{...}, ...]` | — |
| `GET` | `/items/:id` | `200 {...}` | `404 {"error":"not_found"}` |
| `POST` | `/items` | `201 {...}` | `400 {"error":"bad_request"}` |
| `DELETE` | `/items/:id` | `204` (no body) | `404 {"error":"not_found"}` |

An `Item` is three fields:

```zig
pub const Item = struct {
    id: []const u8,
    title: []const u8,
    done: bool = false,
};
```

## It runs today

Before any structure, the whole round trip in one file. The server answers a
fixed number of requests and exits; the client is in the same program, so there
is nothing to start in another terminal:

```zig
const std = @import("std");

const port = 39250;
const requests_to_serve = 3;

fn serve(io: std.Io) void {
    const addr: std.Io.net.IpAddress = .{ .ip4 = .loopback(port) };
    var listener = addr.listen(io, .{ .reuse_address = true }) catch return;
    defer listener.deinit(io);

    for (0..requests_to_serve) |_| {
        var stream = listener.accept(io) catch return;
        defer stream.close(io);

        var rbuf: [4096]u8 = undefined;
        var wbuf: [4096]u8 = undefined;
        var sr = stream.reader(io, &rbuf);
        var sw = stream.writer(io, &wbuf);

        var http: std.http.Server = .init(&sr.interface, &sw.interface);
        var req = http.receiveHead() catch continue;
        std.debug.print("[server] {t} {s}\n", .{ req.head.method, req.head.target });

        req.respond("{\"status\":\"ok\"}", .{
            .status = .ok,
            .keep_alive = false,
            .extra_headers = &.{.{ .name = "content-type", .value = "application/json" }},
        }) catch continue;
    }
}

fn call(gpa: std.mem.Allocator, client: *std.http.Client, path: []const u8) !void {
    const url = try std.fmt.allocPrint(gpa, "http://127.0.0.1:{d}{s}", .{ port, path });
    defer gpa.free(url);

    var body: std.Io.Writer.Allocating = .init(gpa);
    defer body.deinit();

    const res = try client.fetch(.{ .location = .{ .url = url }, .response_writer = &body.writer });
    std.debug.print("[client] GET {s} -> {d} {s}\n", .{ path, @intFromEnum(res.status), body.written() });
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const gpa = init.gpa;

    var task = io.async(serve, .{io});
    defer task.await(io);

    io.sleep(.fromMilliseconds(50), .awake) catch {};

    var client: std.http.Client = .{ .allocator = gpa, .io = io };
    defer client.deinit();

    for ([_][]const u8{ "/health", "/items", "/items/1" }) |p| try call(gpa, &client, p);
}
```

```sh
zig run service.zig
```

```
[server] GET /health
[client] GET /health -> 200 {"status":"ok"}
[server] GET /items
[client] GET /items -> 200 {"status":"ok"}
[server] GET /items/1
[client] GET /items/1 -> 200 {"status":"ok"}
```

Four lines of that are the entire 0.17 server API, and none of them match what
you'll find online:

| Line | What it is |
|---|---|
| `.{ .ip4 = .loopback(port) }` | an `IpAddress` tagged union — `.loopback(p)`, `.unspecified(p)`, or `Ip4Address.parse("10.0.0.1", p)` |
| `addr.listen(io, .{ .reuse_address = true })` | a `std.Io.net.Server`; `deinit(io)` closes it. Without `reuse_address`, a restart inside the TIME_WAIT window fails |
| `stream.reader(io, &rbuf)` | buffer size is **the longest request head you accept** |
| `.init(&sr.interface, &sw.interface)` | `http.Server` is just a reader and a writer — it owns no socket |

`io.async(serve, .{io})` runs the accept loop on another task so the same
program can be its own client. `keep_alive = false` closes after each response,
which keeps `requests_to_serve` meaning what it says.

## Layout

```
itemsvc/
├── build.zig
├── build.zig.zon
└── src/
    ├── main.zig        entry point: config -> server -> run
    ├── config.zig      env + args -> Config, validated at startup
    ├── server.zig      the socket, the accept loop, the arena, HTTP framing
    ├── handlers.zig    Request -> Response, pure except for the store
    ├── router.zig      (method, target) -> Route — no allocator, no io
    ├── store.zig       the only long-lived state
    ├── api.zig         anyerror -> (status, error code)
    ├── root.zig        test-only root that pulls in every module
    └── e2e_test.zig    drives the real server over loopback
```

Each file in one line, with what it is **not** allowed to touch:

| File | Owns | Must not |
|---|---|---|
| `router.zig` | path → route decision | allocate, touch `io`, know about HTTP types beyond `Method` |
| `api.zig` | error → status + code | know any handler, allocate |
| `store.zig` | items, ids, the mutex | know about HTTP at all |
| `handlers.zig` | request → response **values** | touch a socket or `http.Server` |
| `server.zig` | socket, arena, HTTP framing | contain business logic |
| `config.zig` | defaults, env, args | be read anywhere but `main` |

That `handlers.zig` line is the one that pays off. Because a handler takes a
plain `Request` struct and returns a plain `Response` struct, **every handler is
unit-testable with no socket** (14.7). `server.zig` is the only file that knows
a `std.http.Server` exists, and it is the only file you cannot test without a
port.

## The request lifecycle

```
accept                      -> a Stream
  reader/writer over it     -> *Io.Reader, *Io.Writer
  http.receiveHead()        -> Request (head.method, head.target)
  arena.dupe(target)        -> survives reading the body  ← see below
  read body (if any)        -> arena-allocated bytes
  router.route()            -> Route (tagged union)
  handlers.dispatch()       -> Response | error
  on error: api.classify()  -> status + {"error":"code"}
  req.respond()
  stream.close()
  arena.reset(.retain_capacity)   ← the whole request's memory, freed at once
```

Two of those lines are not negotiable:

**`arena.dupe(target)` before reading the body.** `req.head.target` points into
the server's read buffer, and initialising the body stream reuses that buffer.
The stdlib says so out loud: `readerExpectNone` "invalidates the string memory
inside `Head`". Route on a target you copied, or you route on whatever the body
overwrote it with.

**`reset(.retain_capacity)` in a `defer`, not at the end of the happy path.** It
has to run when the handler returns an error too, which is the entire reason it
is a `defer` on the loop body.

## Deliberately not built

| Not here | Because |
|---|---|
| persistence | the memory story is the lesson; a DB would hide it |
| auth | orthogonal, and every shop does it differently |
| keep-alive / pipelining | `keep_alive = false` keeps "requests served" countable |
| one task per connection | 12.7 covers that; here one request at a time keeps the mutex story honest |
| `PUT`/`PATCH` | `POST` + `DELETE` already exercise body parsing and 404s |
| TLS | a reverse proxy's job |

The accept loop being sequential is a deliberate simplification, **not** a
missing feature: `store.zig` is still written mutex-first in 14.4, so moving to
`io.async` per connection is a change to `server.zig` alone.

---

Next: [14.2 — Routing](./02-routing.md)
