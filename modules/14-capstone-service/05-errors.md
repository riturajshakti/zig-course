# 14.5 — Error Handling

Handlers `return error.X`. One function turns the error set into a status and a
stable JSON code. Nothing else in the service writes a status.

## The mapping

| Error | Status | Body |
|---|---|---|
| `NotFound` | 404 | `{"error":"not_found"}` |
| `BadRequest`, `SyntaxError`, `UnexpectedToken`, `MissingField`, `InvalidCharacter`, `Overflow` | 400 | `{"error":"bad_request"}` |
| `MethodNotAllowed` | 405 | `{"error":"method_not_allowed"}` |
| `PayloadTooLarge`, `StreamTooLong` | 413 | `{"error":"payload_too_large"}` |
| **anything else** | 500 | `{"error":"internal"}` |

The `else` arm is the load-bearing one: `OutOfMemory`, `Canceled`,
`WriteFailed`, and every error a future handler invents all become
`{"error":"internal"}` without anyone touching this file.

## `src/api.zig`

```zig
const std = @import("std");

pub const Failure = struct { status: std.http.Status, code: []const u8 };

pub fn classify(err: anyerror) Failure {
    return switch (err) {
        error.NotFound => .{ .status = .not_found, .code = "not_found" },
        error.MethodNotAllowed => .{ .status = .method_not_allowed, .code = "method_not_allowed" },
        error.PayloadTooLarge, error.StreamTooLong => .{ .status = .payload_too_large, .code = "payload_too_large" },
        error.BadRequest,
        error.SyntaxError,
        error.UnexpectedToken,
        error.UnexpectedEndOfInput,
        error.MissingField,
        error.InvalidCharacter,
        error.InvalidNumber,
        error.Overflow,
        => .{ .status = .bad_request, .code = "bad_request" },
        else => .{ .status = .internal_server_error, .code = "internal" },
    };
}

pub fn errorBody(buf: []u8, code: []const u8) []const u8 {
    return std.fmt.bufPrint(buf, "{{\"error\":\"{s}\"}}", .{code}) catch "{\"error\":\"internal\"}";
}
```

`anyerror`, not a closed set, because `classify` is the layer where every
error set in the program converges — including ones from `std.json` and
`std.Io` that you do not control. `switch` on `anyerror` **requires** an `else`,
which is exactly the safe default you want.

`errorBody` writes into a caller-supplied `[64]u8` on the stack: the error path
must work when the failure *was* `OutOfMemory`. The `catch` fallback is a
string literal for the same reason.

## Error sets per layer

| Layer | Error set | Shape |
|---|---|---|
| `router.zig` | none | returns `.not_found` / `.method_not_allowed` as **values** |
| `store.zig` | `error{ NotFound, OutOfMemory, Canceled }` | explicit, documents what callers handle |
| `handlers.zig` | inferred (`!Response`) | the union of the store's set plus `BadRequest`, `MethodNotAllowed` and `std.json`'s set |
| `server.zig` | none out | catches everything and calls `classify` |

Routing returns values rather than errors because "no such path" is a normal
outcome, not a failure — but at the `handlers.zig` boundary those two cases
*become* errors, so the 404 path and the store's 404 path share one exit:

```zig
.not_found => error.NotFound,
.method_not_allowed => error.MethodNotAllowed,
```

The inferred set in the middle is deliberate. Writing it out by hand would mean
re-listing `std.json.ParseError`'s members, and it would break the moment the
stdlib adds one. Explicit sets are for leaves (`store.zig`), inferred sets for
the layer that merges them.

## It runs

```zig
const std = @import("std");

const port = 39254;

const Failure = struct { status: std.http.Status, code: []const u8 };

fn classify(err: anyerror) Failure {
    return switch (err) {
        error.NotFound => .{ .status = .not_found, .code = "not_found" },
        error.BadRequest, error.InvalidCharacter, error.SyntaxError, error.UnexpectedToken, error.MissingField => .{ .status = .bad_request, .code = "bad_request" },
        error.MethodNotAllowed => .{ .status = .method_not_allowed, .code = "method_not_allowed" },
        error.PayloadTooLarge, error.StreamTooLong => .{ .status = .payload_too_large, .code = "payload_too_large" },
        else => .{ .status = .internal_server_error, .code = "internal" },
    };
}

fn errorResponse(req: *std.http.Server.Request, err: anyerror) void {
    const f = classify(err);
    std.debug.print("[server] {t} -> {d} {s}\n", .{ err, @intFromEnum(f.status), f.code });

    var buf: [128]u8 = undefined;
    const body = std.fmt.bufPrint(&buf, "{{\"error\":\"{s}\"}}", .{f.code}) catch "{\"error\":\"internal\"}";
    req.respond(body, .{
        .status = f.status,
        .keep_alive = false,
        .extra_headers = &.{.{ .name = "content-type", .value = "application/json" }},
    }) catch {};
}

const NewItem = struct { title: []const u8 };

fn handle(arena: std.mem.Allocator, req: *std.http.Server.Request) ![]const u8 {
    const target = try arena.dupe(u8, req.head.target);

    if (std.mem.eql(u8, target, "/items")) {
        if (req.head.method != .POST) return error.MethodNotAllowed;
        if ((req.head.content_length orelse 0) > 1024) return error.PayloadTooLarge;

        var bbuf: [1024]u8 = undefined;
        const reader = try req.readerExpectContinue(&bbuf);
        const raw = try reader.allocRemaining(arena, .limited(1024));

        const parsed = try std.json.parseFromSlice(NewItem, arena, raw, .{ .ignore_unknown_fields = true });
        if (parsed.value.title.len == 0) return error.BadRequest;
        return std.fmt.allocPrint(arena, "{{\"id\":\"1\",\"title\":\"{s}\"}}", .{parsed.value.title});
    }
    if (std.mem.startsWith(u8, target, "/items/")) return error.NotFound;
    if (std.mem.eql(u8, target, "/boom")) return error.DatabaseOnFire;
    return error.NotFound;
}

fn serve(io: std.Io, gpa: std.mem.Allocator, n: usize) void {
    const addr: std.Io.net.IpAddress = .{ .ip4 = .loopback(port) };
    var listener = addr.listen(io, .{ .reuse_address = true }) catch return;
    defer listener.deinit(io);

    var arena_s: std.heap.ArenaAllocator = .init(gpa);
    defer arena_s.deinit();

    for (0..n) |_| {
        defer _ = arena_s.reset(.retain_capacity);
        var stream = listener.accept(io) catch return;
        defer stream.close(io);

        var rbuf: [4096]u8 = undefined;
        var wbuf: [4096]u8 = undefined;
        var sr = stream.reader(io, &rbuf);
        var sw = stream.writer(io, &wbuf);
        var http: std.http.Server = .init(&sr.interface, &sw.interface);
        var req = http.receiveHead() catch continue;

        if (handle(arena_s.allocator(), &req)) |body| {
            std.debug.print("[server] ok -> 201\n", .{});
            req.respond(body, .{
                .status = .created,
                .keep_alive = false,
                .extra_headers = &.{.{ .name = "content-type", .value = "application/json" }},
            }) catch {};
        } else |err| errorResponse(&req, err);
    }
}

const Call = struct { method: std.http.Method, path: []const u8, body: ?[]const u8 };

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const gpa = init.gpa;

    const calls = [_]Call{
        .{ .method = .POST, .path = "/items", .body = "{\"title\":\"real work\"}" },
        .{ .method = .POST, .path = "/items", .body = "{\"title\":\"\"}" },
        .{ .method = .POST, .path = "/items", .body = "not json" },
        .{ .method = .GET, .path = "/items", .body = null },
        .{ .method = .GET, .path = "/items/42", .body = null },
        .{ .method = .GET, .path = "/boom", .body = null },
    };

    var task = io.async(serve, .{ io, gpa, calls.len });
    defer task.await(io);
    io.sleep(.fromMilliseconds(50), .awake) catch {};

    var client: std.http.Client = .{ .allocator = gpa, .io = io };
    defer client.deinit();

    for (calls) |c| {
        const url = try std.fmt.allocPrint(gpa, "http://127.0.0.1:{d}{s}", .{ port, c.path });
        defer gpa.free(url);
        var body: std.Io.Writer.Allocating = .init(gpa);
        defer body.deinit();
        const res = try client.fetch(.{
            .location = .{ .url = url },
            .method = c.method,
            .payload = c.body,
            .response_writer = &body.writer,
        });
        std.debug.print("[client] {t:<6} {s:<10} -> {d} {s}\n", .{ c.method, c.path, @intFromEnum(res.status), body.written() });
    }
}
```

```
[server] ok -> 201
[client] POST   /items     -> 201 {"id":"1","title":"real work"}
[server] BadRequest -> 400 bad_request
[client] POST   /items     -> 400 {"error":"bad_request"}
[server] SyntaxError -> 400 bad_request
[client] POST   /items     -> 400 {"error":"bad_request"}
[server] MethodNotAllowed -> 405 method_not_allowed
[client] GET    /items     -> 405 {"error":"method_not_allowed"}
[server] NotFound -> 404 not_found
[client] GET    /items/42  -> 404 {"error":"not_found"}
[server] DatabaseOnFire -> 500 internal
[client] GET    /boom      -> 500 {"error":"internal"}
```

## Never leaking internals

The last two lines are the whole discipline:

```
[server] DatabaseOnFire -> 500 internal
[client] GET /boom      -> 500 {"error":"internal"}
```

The server log names the error; the client gets five words that reveal nothing
about the implementation. Three rules follow:

| Rule | Why |
|---|---|
| the client body is built from `f.code`, never from `{t}` on the error | `{t}` on `error.DatabaseOnFire` tells an attacker what you run |
| `f.code` values are a closed vocabulary (`not_found`, `bad_request`, `method_not_allowed`, `payload_too_large`, `internal`) | clients can branch on them; they survive refactors |
| log at `err` level **only** for 5xx | a 404 is not an incident; filling the log with them hides the real ones |

That third rule in the final service:

```zig
if (f.status == .internal_server_error)
    std.log.err("unhandled: {t} on {t} {s}", .{ err, req.head.method, req.head.target });
```

Note also that `std.json.parseFromSlice`'s `SyntaxError` becomes a 400 without
`handle` mentioning JSON errors at all — it just `try`s, and `classify` knows.

## Reading the body: the trap

```zig
const target = try arena.dupe(u8, req.head.target);   // ← before the next line
const reader = try req.readerExpectContinue(&bbuf);
```

`readerExpectContinue` initialises the body stream over the same buffer the head
was parsed into, and the stdlib documents it as invalidating the strings inside
`Head`. Route on `req.head.target` *after* that call and you are routing on the
body. Copy it into the arena first — one `dupe` that costs nothing because the
arena is already there.

Also note the order of the two checks:

```zig
if ((req.head.content_length orelse 0) > 1024) return error.PayloadTooLarge;
...
const raw = try reader.allocRemaining(arena, .limited(1024));
```

The header check rejects a declared-oversize body before reading a byte; the
`.limited()` catches a lying or chunked one, failing with `StreamTooLong` —
which `classify` also maps to 413. Belt and braces, and the limit lives in the
config rather than as a literal in the final version.

## `errdefer` mid-request

The arena means handlers can't leak request memory. But a handler that touches
**long-lived** state before failing still has to undo it:

```zig
const std = @import("std");

const Audit = struct {
    gpa: std.mem.Allocator,
    entries: std.ArrayList([]const u8) = .empty,

    fn deinit(a: *Audit) void {
        for (a.entries.items) |e| a.gpa.free(e);
        a.entries.deinit(a.gpa);
    }
};

fn handleLeaky(arena: std.mem.Allocator, audit: *Audit, title: []const u8) ![]const u8 {
    const note = try std.fmt.allocPrint(audit.gpa, "created {s}", .{title});
    try audit.entries.append(audit.gpa, note);
    if (title.len == 0) return error.BadRequest;
    return std.fmt.allocPrint(arena, "{{\"title\":\"{s}\"}}", .{title});
}

fn handleSafe(arena: std.mem.Allocator, audit: *Audit, title: []const u8) ![]const u8 {
    const note = try std.fmt.allocPrint(audit.gpa, "created {s}", .{title});
    errdefer audit.gpa.free(note);
    try audit.entries.append(audit.gpa, note);
    errdefer _ = audit.entries.pop();
    if (title.len == 0) return error.BadRequest;
    return std.fmt.allocPrint(arena, "{{\"title\":\"{s}\"}}", .{title});
}

fn run(
    label: []const u8,
    handler: *const fn (std.mem.Allocator, *Audit, []const u8) anyerror![]const u8,
) !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    const gpa = gpa_s.allocator();

    var arena_s: std.heap.ArenaAllocator = .init(gpa);
    var audit: Audit = .{ .gpa = gpa };

    for ([_][]const u8{ "ok one", "", "ok two" }) |title| {
        defer _ = arena_s.reset(.retain_capacity);
        _ = handler(arena_s.allocator(), &audit, title) catch {};
    }

    std.debug.print("{s}: audit holds {d} entries\n", .{ label, audit.entries.items.len });
    audit.deinit();
    arena_s.deinit();
    std.debug.print("{s}: gpa deinit -> {t}\n", .{ label, gpa_s.deinit() });
}

pub fn main() !void {
    try run("no errdefer", handleLeaky);
    try run("errdefer   ", handleSafe);
}
```

```
no errdefer: audit holds 3 entries
no errdefer: gpa deinit -> ok
errdefer   : audit holds 2 entries
errdefer   : gpa deinit -> ok
```

Two requests succeeded, one failed with a 400 — and the version without
`errdefer` recorded **three** audit entries. Note both report `gpa deinit -> ok`:
this is not a leak, it is *committed state for a request that returned an
error*, and no allocator will ever tell you about it. The arena protects you
from leaking; only `errdefer` protects you from half-applied writes.

`Store.create` in 14.4 is the same shape, which is why its three allocations
each carry one.

---

Next: [14.6 — Config & Shutdown](./06-config-shutdown.md)
