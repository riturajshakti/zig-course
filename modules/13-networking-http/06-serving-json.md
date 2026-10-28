# 13.6 — Serving JSON

Two functions do all the work:

| Direction | Call |
|---|---|
| Zig value → JSON text | `std.json.Stringify.valueAlloc(gpa, value, .{})` |
| JSON text → Zig value | `std.json.parseFromSlice(T, gpa, bytes, .{})` |

`parseFromSlice` returns a `Parsed(T)` — `.value` is your data, `.deinit()`
frees everything it allocated for it.

## Options, in one program

```zig
const std = @import("std");

const Todo = struct { id: u32, title: []const u8, note: ?[]const u8 = null };

pub fn main(init: std.process.Init) !void {
    const gpa = init.gpa;
    const value: Todo = .{ .id = 7, .title = "ship it", .note = null };

    const compact = try std.json.Stringify.valueAlloc(gpa, value, .{});
    defer gpa.free(compact);
    std.debug.print("default        {s}\n", .{compact});

    const indented = try std.json.Stringify.valueAlloc(gpa, value, .{ .whitespace = .indent_2 });
    defer gpa.free(indented);
    std.debug.print("indent_2\n{s}\n", .{indented});

    const no_null = try std.json.Stringify.valueAlloc(gpa, value, .{ .emit_null_optional_fields = false });
    defer gpa.free(no_null);
    std.debug.print("skip nulls     {s}\n", .{no_null});

    const inputs = [_][]const u8{
        "{\"id\":1,\"title\":\"a\"}",
        "{\"id\":2,\"title\":\"b\",\"extra\":true}",
        "{\"id\":\"3\",\"title\":\"c\"}",
        "{\"title\":\"no id\"}",
    };
    for (inputs) |raw| {
        const strict = std.json.parseFromSlice(Todo, gpa, raw, .{});
        if (strict) |p| {
            defer p.deinit();
            std.debug.print("strict  ok   {s}\n", .{raw});
        } else |err| {
            std.debug.print("strict  {t:<22} {s}\n", .{ err, raw });
        }
        const loose = std.json.parseFromSlice(Todo, gpa, raw, .{ .ignore_unknown_fields = true });
        if (loose) |p| {
            defer p.deinit();
            std.debug.print("  loose ok   id={d}\n", .{p.value.id});
        } else |err| {
            std.debug.print("  loose {t}\n", .{err});
        }
    }
}
```

```
default        {"id":7,"title":"ship it","note":null}
indent_2
{
  "id": 7,
  "title": "ship it",
  "note": null
}
skip nulls     {"id":7,"title":"ship it"}
strict  ok   {"id":1,"title":"a"}
  loose ok   id=1
strict  UnknownField           {"id":2,"title":"b","extra":true}
  loose ok   id=2
strict  ok   {"id":"3","title":"c"}
  loose ok   id=3
strict  MissingField           {"title":"no id"}
  loose MissingField
```

| Behaviour | Default | Flip with |
|---|---|---|
| unknown field | `error.UnknownField` | `.ignore_unknown_fields = true` |
| missing field with no default | `error.MissingField` | give the field a Zig default |
| `null` optional field | emitted as `null` | `.emit_null_optional_fields = false` |
| output | compact, one line | `.whitespace = .indent_2` / `.indent_4` / `.indent_tab` |

Two surprises worth memorising: **`"3"` parses into a `u32`** — JSON strings
coerce to numbers — and `UnknownField` is an *error* by default, which makes
Zig's parser stricter than `JSON.parse` in both directions.

## A GET + POST endpoint pair

```zig
const std = @import("std");

const port = 39260;
const base = std.fmt.comptimePrint("http://127.0.0.1:{d}", .{port});

const Todo = struct { id: u32, title: []const u8, done: bool };
const NewTodo = struct { title: []const u8 };
const ApiError = struct { @"error": []const u8, detail: []const u8 };

const json_ct: std.http.Header = .{ .name = "content-type", .value = "application/json" };

fn sendJson(req: *std.http.Server.Request, gpa: std.mem.Allocator, status: std.http.Status, value: anytype) !void {
    const text = try std.json.Stringify.valueAlloc(gpa, value, .{});
    defer gpa.free(text);
    try req.respond(text, .{ .status = status, .extra_headers = &.{json_ct} });
}

fn handle(req: *std.http.Server.Request, gpa: std.mem.Allocator, next_id: *u32) !void {
    const method = req.head.method;
    const target = try gpa.dupe(u8, req.head.target);
    defer gpa.free(target);

    if (method == .GET and std.mem.eql(u8, target, "/todos")) {
        const list = [_]Todo{
            .{ .id = 1, .title = "learn zig", .done = true },
            .{ .id = 2, .title = "ship service", .done = false },
        };
        return sendJson(req, gpa, .ok, list);
    }

    if (method == .POST and std.mem.eql(u8, target, "/todos")) {
        var body_buf: [4096]u8 = undefined;
        const reader = try req.readerExpectContinue(&body_buf);
        const raw = try reader.allocRemaining(gpa, .limited(64 * 1024));
        defer gpa.free(raw);

        const parsed = std.json.parseFromSlice(NewTodo, gpa, raw, .{}) catch |err| {
            return sendJson(req, gpa, .bad_request, ApiError{
                .@"error" = "invalid_json",
                .detail = @errorName(err),
            });
        };
        defer parsed.deinit();

        defer next_id.* += 1;
        return sendJson(req, gpa, .created, Todo{
            .id = next_id.*,
            .title = parsed.value.title,
            .done = false,
        });
    }

    return sendJson(req, gpa, .not_found, ApiError{ .@"error" = "no_route", .detail = target });
}

fn serve(io: std.Io, gpa: std.mem.Allocator, n: usize) void {
    const addr: std.Io.net.IpAddress = .{ .ip4 = .loopback(port) };
    var listener = addr.listen(io, .{ .reuse_address = true }) catch return;
    defer listener.deinit(io);
    var stream = listener.accept(io) catch return;
    defer stream.close(io);

    var rbuf: [8192]u8 = undefined;
    var wbuf: [8192]u8 = undefined;
    var sr = stream.reader(io, &rbuf);
    var sw = stream.writer(io, &wbuf);
    var http: std.http.Server = .init(&sr.interface, &sw.interface);

    var next_id: u32 = 3;
    for (0..n) |_| {
        var req = http.receiveHead() catch return;
        handle(&req, gpa, &next_id) catch return;
    }
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const gpa = init.gpa;

    const calls = [_]std.http.Client.FetchOptions{
        .{ .location = .{ .url = base ++ "/todos" } },
        .{ .location = .{ .url = base ++ "/todos" }, .payload = "{\"title\":\"write tests\"}" },
        .{ .location = .{ .url = base ++ "/todos" }, .payload = "{oops" },
        .{ .location = .{ .url = base ++ "/nope" } },
    };

    var task = io.async(serve, .{ io, gpa, calls.len });
    defer task.await(io);
    io.sleep(.fromMilliseconds(50), .awake) catch {};

    var client: std.http.Client = .{ .allocator = gpa, .io = io };
    defer client.deinit();

    for (calls) |c| {
        var body: std.Io.Writer.Allocating = .init(gpa);
        defer body.deinit();
        var o = c;
        o.response_writer = &body.writer;
        const res = try client.fetch(o);
        std.debug.print("{d} {s}\n", .{ @intFromEnum(res.status), body.written() });
    }
}
```

```
200 [{"id":1,"title":"learn zig","done":true},{"id":2,"title":"ship service","done":false}]
201 {"id":3,"title":"write tests","done":false}
400 {"error":"invalid_json","detail":"SyntaxError"}
404 {"error":"no_route","detail":"/nope"}
```

What that program establishes:

- **A Zig array stringifies to a JSON array**, so a collection endpoint is just
  `sendJson(req, gpa, .ok, slice_of_structs)`.
- **`sendJson` takes `anytype`**, so one helper serves both success and error
  shapes — a 400 body is as structured as a 200.
- **`@"error"` is a quoted identifier**, which is how you emit the field name
  `"error"` despite it being a Zig keyword.
- **`@errorName(err)`** turns a parse failure into a client-safe string. Never
  send the raw body back; it is attacker-controlled.

## The invalidation trap

This is the one that costs an afternoon. In the program above:

```zig
const target = try gpa.dupe(u8, req.head.target);   // copy FIRST
defer gpa.free(target);
...
const reader = try req.readerExpectContinue(&body_buf);   // invalidates req.head
```

Swap those two lines — use `req.head.target` *after* creating the body reader —
and you do not get an error. You get:

```
Segmentation fault at address 0xaaaaaaaaaaaaaaaa
... in utf8ValidateSliceImpl
... in std/json/Stringify.zig:506
```

`req.head`'s slices point into the stream reader's buffer. Initialising the
body stream reuses that buffer, and the dangling `target` becomes stack-poison
bytes (`0xAA`) that the JSON writer then tries to validate as UTF-8.

**Rule: read everything you need out of `req.head` before touching the body.**
Copy the bits you keep.

## Content type

`respond` writes `content-length` for you but never a `content-type` — browsers
and `fetch()` clients will sniff or refuse. Always pass it:

```zig
.extra_headers = &.{.{ .name = "content-type", .value = "application/json" }},
```

Hoisting it into a `const` as `json_ct` above keeps it to one place. There is no
helper for this and no default; it is a deliberate "you decide" in the stdlib.

Incoming, `req.head.content_type` is an `?[]const u8` you should actually check
before parsing — a `POST` with `text/plain` is a `415 unsupported_media_type`,
not a JSON parse error.

## Arena per request

A handler allocates a dozen short-lived things: the duped target, the raw body,
the parsed tree, the stringified response. Tracking each one's `free` is noise.
Give the handler an arena, reset it per request, and the whole handler needs
zero `defer`:

```zig
const std = @import("std");

const port = 39261;
const base = std.fmt.comptimePrint("http://127.0.0.1:{d}", .{port});

const Echo = struct { target: []const u8, body_len: usize, words: []const []const u8 };

fn handle(req: *std.http.Server.Request, arena: std.mem.Allocator) !void {
    const target = try arena.dupe(u8, req.head.target);

    var body_buf: [4096]u8 = undefined;
    const reader = try req.readerExpectContinue(&body_buf);
    const raw = try reader.allocRemaining(arena, .limited(64 * 1024));

    var words: std.ArrayList([]const u8) = .empty;
    var it = std.mem.tokenizeAny(u8, raw, " \n");
    while (it.next()) |w| try words.append(arena, try arena.dupe(u8, w));

    const text = try std.json.Stringify.valueAlloc(arena, Echo{
        .target = target,
        .body_len = raw.len,
        .words = words.items,
    }, .{});

    try req.respond(text, .{
        .extra_headers = &.{.{ .name = "content-type", .value = "application/json" }},
    });
}

fn serve(io: std.Io, gpa: std.mem.Allocator, n: usize) void {
    const addr: std.Io.net.IpAddress = .{ .ip4 = .loopback(port) };
    var listener = addr.listen(io, .{ .reuse_address = true }) catch return;
    defer listener.deinit(io);
    var stream = listener.accept(io) catch return;
    defer stream.close(io);

    var rbuf: [8192]u8 = undefined;
    var wbuf: [8192]u8 = undefined;
    var sr = stream.reader(io, &rbuf);
    var sw = stream.writer(io, &wbuf);
    var http: std.http.Server = .init(&sr.interface, &sw.interface);

    for (0..n) |_| {
        var req = http.receiveHead() catch return;

        var arena: std.heap.ArenaAllocator = .init(gpa);
        defer arena.deinit();

        handle(&req, arena.allocator()) catch return;
    }
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const gpa = init.gpa;

    const payloads = [_][]const u8{ "alpha beta gamma", "one", "" };

    var task = io.async(serve, .{ io, gpa, payloads.len });
    defer task.await(io);
    io.sleep(.fromMilliseconds(50), .awake) catch {};

    var client: std.http.Client = .{ .allocator = gpa, .io = io };
    defer client.deinit();

    for (payloads) |p| {
        var body: std.Io.Writer.Allocating = .init(gpa);
        defer body.deinit();
        const res = try client.fetch(.{
            .location = .{ .url = base ++ "/echo" },
            .method = .POST,
            .payload = p,
            .response_writer = &body.writer,
        });
        std.debug.print("{d} {s}\n", .{ @intFromEnum(res.status), body.written() });
    }
}
```

```
200 {"target":"/echo","body_len":16,"words":["alpha","beta","gamma"]}
200 {"target":"/echo","body_len":3,"words":["one"]}
200 {"target":"/echo","body_len":0,"words":[]}
```

Count the `defer`s in `handle`: **none**. One `arena.deinit()` in the loop frees
the duped target, the body, the ArrayList's backing array, every duped word and
the response text. That is the per-request allocator pattern from module 5,
and it is why `std.heap.ArenaAllocator` exists.

In a real service you would hoist the arena out of the loop and
`_ = arena.reset(.retain_capacity)` per request instead, so the first request's
pages get reused instead of returned to the OS every time.

| Shape | Lifetime | Use when |
|---|---|---|
| `gpa` + matched `free` | explicit | the value outlives the request |
| arena per request, `deinit` | end of request | simplest correct default |
| arena hoisted, `reset(.retain_capacity)` | end of request, pages kept | hot path |
| `FixedBufferAllocator` on the stack | end of request | hard memory cap, no heap at all |

## Where this leaves you

You now have every piece of an HTTP service: a listener, a `receiveHead` loop,
a routing `switch`, JSON in and out, structured errors, and an allocator
strategy. What is missing is everything a framework would have given you —
and that is what you build next:

| Missing | Module 14 adds |
|---|---|
| a route table instead of `if` chains | a comptime-checked dispatcher |
| shared state across requests | a store behind a mutex |
| graceful shutdown | listener `shutdown` + draining |
| tests against handlers | `Writer.fixed` in place of a socket, no `io` needed |
| config, logging, metrics | the unglamorous 80% |

---

Next: [14 — Capstone: A JSON HTTP Service](../14-capstone-service/)
