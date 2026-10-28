# 14.2 — Routing

`route()` takes a method and a target and returns a value. No allocator, no
`io`, no HTTP objects — so you can test every path in a `test` block.

## The route table

| Method | Path | `Route` |
|---|---|---|
| `GET` | `/health` | `.health` |
| `GET` | `/items` | `.list_items` |
| `POST` | `/items` | `.create_item` |
| `GET` | `/items/:id` | `.{ .get_item = id }` |
| `DELETE` | `/items/:id` | `.{ .delete_item = id }` |
| any | unknown path | `.not_found` |
| wrong | known path | `.method_not_allowed` |

A tagged union, not a function pointer: the param rides **inside** the route, so
there is no out-param, no `?[]const u8` to unwrap, and `switch` on the result is
exhaustive — add a route and the compiler finds every place that must handle it.

## `src/router.zig`

```zig
const std = @import("std");

pub const Route = union(enum) {
    health,
    list_items,
    get_item: []const u8,
    create_item,
    delete_item: []const u8,
    not_found,
    method_not_allowed,
};

pub fn route(method: std.http.Method, target: []const u8) Route {
    const path = if (std.mem.findScalar(u8, target, '?')) |q| target[0..q] else target;

    if (std.mem.eql(u8, path, "/health"))
        return if (method == .GET) .health else .method_not_allowed;

    if (std.mem.eql(u8, path, "/items")) return switch (method) {
        .GET => .list_items,
        .POST => .create_item,
        else => .method_not_allowed,
    };

    if (std.mem.startsWith(u8, path, "/items/")) {
        const id = path["/items/".len..];
        if (id.len == 0 or std.mem.findScalar(u8, id, '/') != null) return .not_found;
        return switch (method) {
            .GET => .{ .get_item = id },
            .DELETE => .{ .delete_item = id },
            else => .method_not_allowed,
        };
    }
    return .not_found;
}
```

Four things worth naming:

| Line | Why |
|---|---|
| `findScalar(u8, target, '?')` | `req.head.target` is the **raw** request target — `/items?limit=2`, query included. Nothing strips it for you |
| `path["/items/".len..]` | comptime-known length; no magic `7` |
| `id.len == 0` | `/items/` is a 404, not an empty id |
| `findScalar(u8, id, '/')` | `/items/1/tags` is a 404, not `id = "1/tags"` |

**`id` is a slice into `target`** — zero allocation, and it lives exactly as long
as the buffer `target` points into. 14.1 explained why that buffer dies when you
read the request body; that is the one rule this file depends on from outside.

## Tests, no socket

```zig
const std = @import("std");
const testing = std.testing;

pub const Route = union(enum) {
    health,
    list_items,
    get_item: []const u8,
    create_item,
    delete_item: []const u8,
    not_found,
    method_not_allowed,
};

pub fn route(method: std.http.Method, target: []const u8) Route {
    const path = if (std.mem.findScalar(u8, target, '?')) |q| target[0..q] else target;

    if (std.mem.eql(u8, path, "/health"))
        return if (method == .GET) .health else .method_not_allowed;

    if (std.mem.eql(u8, path, "/items")) return switch (method) {
        .GET => .list_items,
        .POST => .create_item,
        else => .method_not_allowed,
    };

    if (std.mem.startsWith(u8, path, "/items/")) {
        const id = path["/items/".len..];
        if (id.len == 0 or std.mem.findScalar(u8, id, '/') != null) return .not_found;
        return switch (method) {
            .GET => .{ .get_item = id },
            .DELETE => .{ .delete_item = id },
            else => .method_not_allowed,
        };
    }
    return .not_found;
}

test "static paths" {
    try testing.expectEqual(Route.health, route(.GET, "/health"));
    try testing.expectEqual(Route.list_items, route(.GET, "/items"));
    try testing.expectEqual(Route.create_item, route(.POST, "/items"));
}

test "query string is stripped" {
    try testing.expectEqual(Route.list_items, route(.GET, "/items?limit=2&offset=1"));
    try testing.expectEqual(Route.health, route(.GET, "/health?x"));
}

test "path param is a slice of the target" {
    const target = "/items/abc123";
    const r = route(.GET, target);
    try testing.expectEqualStrings("abc123", r.get_item);
    try testing.expectEqual(target.ptr + 7, r.get_item.ptr);
}

test "wrong method is 405, unknown path is 404" {
    try testing.expectEqual(Route.method_not_allowed, route(.POST, "/health"));
    try testing.expectEqual(Route.method_not_allowed, route(.PUT, "/items"));
    try testing.expectEqual(Route.method_not_allowed, route(.PATCH, "/items/1"));
    try testing.expectEqual(Route.not_found, route(.GET, "/nope"));
    try testing.expectEqual(Route.not_found, route(.GET, "/items/"));
    try testing.expectEqual(Route.not_found, route(.GET, "/items/1/tags"));
}
```

```sh
zig test router.zig
```

```
1/4 router.test.static paths...OK
2/4 router.test.query string is stripped...OK
3/4 router.test.path param is a slice of the target...OK
4/4 router.test.wrong method is 405, unknown path is 404...OK
All 4 tests passed.
```

`expectEqual(Route.health, ...)` compares the whole tagged union, tag and
payload. For the slice payload use `expectEqualStrings` — `expectEqual` on two
`[]const u8` compares **pointers**, which is also exactly why
`expectEqual(target.ptr + 7, r.get_item.ptr)` is the right assertion for
"no copy was made".

## Wired to the socket

Same `route()`, now behind real HTTP:

```zig
const std = @import("std");

const port = 39251;

pub const Route = union(enum) {
    health,
    list_items,
    get_item: []const u8,
    create_item,
    delete_item: []const u8,
    not_found,
    method_not_allowed,
};

pub fn route(method: std.http.Method, target: []const u8) Route {
    const path = if (std.mem.findScalar(u8, target, '?')) |q| target[0..q] else target;

    if (std.mem.eql(u8, path, "/health"))
        return if (method == .GET) .health else .method_not_allowed;

    if (std.mem.eql(u8, path, "/items")) return switch (method) {
        .GET => .list_items,
        .POST => .create_item,
        else => .method_not_allowed,
    };

    if (std.mem.startsWith(u8, path, "/items/")) {
        const id = path["/items/".len..];
        if (id.len == 0 or std.mem.findScalar(u8, id, '/') != null) return .not_found;
        return switch (method) {
            .GET => .{ .get_item = id },
            .DELETE => .{ .delete_item = id },
            else => .method_not_allowed,
        };
    }
    return .not_found;
}

fn reply(req: *std.http.Server.Request, r: Route) void {
    var buf: [128]u8 = undefined;
    const pair: struct { std.http.Status, []const u8 } = switch (r) {
        .health => .{ .ok, "{\"route\":\"health\"}" },
        .list_items => .{ .ok, "{\"route\":\"list_items\"}" },
        .create_item => .{ .created, "{\"route\":\"create_item\"}" },
        .get_item => |id| .{ .ok, std.fmt.bufPrint(&buf, "{{\"route\":\"get_item\",\"id\":\"{s}\"}}", .{id}) catch "{}" },
        .delete_item => .{ .no_content, "" },
        .not_found => .{ .not_found, "{\"error\":\"not_found\"}" },
        .method_not_allowed => .{ .method_not_allowed, "{\"error\":\"method_not_allowed\"}" },
    };
    req.respond(pair[1], .{
        .status = pair[0],
        .keep_alive = false,
        .extra_headers = &.{.{ .name = "content-type", .value = "application/json" }},
    }) catch {};
}

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
        var req = http.receiveHead() catch continue;
        reply(&req, route(req.head.method, req.head.target));
    }
}

const Call = struct { std.http.Method, []const u8 };

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const gpa = init.gpa;

    const calls = [_]Call{
        .{ .GET, "/health" },
        .{ .POST, "/health" },
        .{ .GET, "/items" },
        .{ .GET, "/items?limit=2&done=true" },
        .{ .POST, "/items" },
        .{ .PUT, "/items" },
        .{ .GET, "/items/42" },
        .{ .DELETE, "/items/42" },
        .{ .PATCH, "/items/42" },
        .{ .GET, "/items/" },
        .{ .GET, "/items/42/tags" },
        .{ .GET, "/nope" },
    };

    var task = io.async(serve, .{ io, calls.len });
    defer task.await(io);
    io.sleep(.fromMilliseconds(50), .awake) catch {};

    var client: std.http.Client = .{ .allocator = gpa, .io = io };
    defer client.deinit();

    for (calls) |c| {
        const url = try std.fmt.allocPrint(gpa, "http://127.0.0.1:{d}{s}", .{ port, c[1] });
        defer gpa.free(url);
        var body: std.Io.Writer.Allocating = .init(gpa);
        defer body.deinit();
        const res = try client.fetch(.{
            .location = .{ .url = url },
            .method = c[0],
            .payload = if (c[0].requestHasBody()) "{}" else null,
            .response_writer = &body.writer,
        });
        std.debug.print("{t:<6} {s:<22} -> {d} {s}\n", .{ c[0], c[1], @intFromEnum(res.status), body.written() });
    }
}
```

```
GET    /health                -> 200 {"route":"health"}
POST   /health                -> 405 {"error":"method_not_allowed"}
GET    /items                 -> 200 {"route":"list_items"}
GET    /items?limit=2&done=true -> 200 {"route":"list_items"}
POST   /items                 -> 201 {"route":"create_item"}
PUT    /items                 -> 405 {"error":"method_not_allowed"}
GET    /items/42              -> 200 {"route":"get_item","id":"42"}
DELETE /items/42              -> 204 
PATCH  /items/42              -> 405 {"error":"method_not_allowed"}
GET    /items/                -> 404 {"error":"not_found"}
GET    /items/42/tags         -> 404 {"error":"not_found"}
GET    /nope                  -> 404 {"error":"not_found"}
```

## Three traps in that program

**`.delete_item => .{ .no_content, "" }` has no capture.** Writing
`|_|` to discard it is a compile error in 0.17: *"discard of capture; omit it
instead"*. Omit the `|...|` entirely.

**`.payload = if (c[0].requestHasBody()) "{}" else null`.** `client.fetch` with
a body-bearing method and a `null` payload hits an `assert` inside the stdlib —
a `reached unreachable code` panic, not an error you can catch. `PUT` must be
given a payload even when you only want to see the 405.

**204 must have an empty body.** `respond` will happily send content with a
`.no_content` status; the client discards it and the response is malformed.

## Why not a table of function pointers

```zig
const Handler = *const fn (std.mem.Allocator, []const u8) anyerror![]const u8;
const table = [_]struct { std.http.Method, []const u8, Handler }{ ... };
```

It reads nicely and costs you the thing that matters: with a tagged union,
`switch (route(...))` is exhaustive, so adding `.patch_item` to `Route` turns
every incomplete `switch` into a compile error. A table of pointers turns the
same change into a 404 at runtime. Keep routing as data the compiler checks.

---

Next: [14.3 — Per-Request Arena](./03-request-arena.md)
