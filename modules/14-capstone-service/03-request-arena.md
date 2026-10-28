# 14.3 — Per-Request Arena

One allocator per request, reset when the response is written. Handlers
allocate as much as they want and free nothing.

## The shape

```zig
var arena_s: std.heap.ArenaAllocator = .init(gpa);
defer arena_s.deinit();

while (...) {
    defer _ = arena_s.reset(.retain_capacity);   // ← the whole request, freed
    const stream = try listener.accept(io);
    defer stream.close(io);
    ...
}
```

| Line | Why exactly there |
|---|---|
| `.init(gpa)` **outside** the loop | one arena for the server's lifetime, not one per request |
| `defer ... reset` **first in the body** | runs last, after `stream.close`, and runs on the error path too |
| `.retain_capacity` | keeps the pages; after request 1 the arena never asks the OS again |
| `deinit()` outside | returns the pages to the gpa at shutdown |

## It holds capacity flat

```zig
const std = @import("std");

const port = 39252;
const n_requests = 4;

fn handle(arena: std.mem.Allocator, target: []const u8) ![]const u8 {
    var parts: std.ArrayList([]const u8) = .empty;
    var it = std.mem.tokenizeScalar(u8, target, '/');
    while (it.next()) |seg| try parts.append(arena, try std.fmt.allocPrint(arena, "\"{s}\"", .{seg}));

    const joined = try std.mem.join(arena, ",", parts.items);
    return std.fmt.allocPrint(arena, "{{\"path\":{s},\"segments\":[{s}]}}", .{
        try std.json.Stringify.valueAlloc(arena, target, .{}),
        joined,
    });
}

fn serve(io: std.Io, gpa: std.mem.Allocator) void {
    const addr: std.Io.net.IpAddress = .{ .ip4 = .loopback(port) };
    var listener = addr.listen(io, .{ .reuse_address = true }) catch return;
    defer listener.deinit(io);

    var arena_s: std.heap.ArenaAllocator = .init(gpa);
    defer arena_s.deinit();

    for (0..n_requests) |_| {
        defer _ = arena_s.reset(.retain_capacity);

        var stream = listener.accept(io) catch return;
        defer stream.close(io);

        var rbuf: [4096]u8 = undefined;
        var wbuf: [4096]u8 = undefined;
        var sr = stream.reader(io, &rbuf);
        var sw = stream.writer(io, &wbuf);
        var http: std.http.Server = .init(&sr.interface, &sw.interface);
        var req = http.receiveHead() catch continue;

        const body = handle(arena_s.allocator(), req.head.target) catch {
            req.respond("{\"error\":\"internal\"}", .{ .status = .internal_server_error, .keep_alive = false }) catch {};
            continue;
        };
        std.debug.print("[server] {s} arena capacity={d}\n", .{ req.head.target, arena_s.queryCapacity() });
        req.respond(body, .{
            .keep_alive = false,
            .extra_headers = &.{.{ .name = "content-type", .value = "application/json" }},
        }) catch continue;
    }
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const gpa = init.gpa;

    var task = io.async(serve, .{ io, gpa });
    defer task.await(io);
    io.sleep(.fromMilliseconds(50), .awake) catch {};

    var client: std.http.Client = .{ .allocator = gpa, .io = io };
    defer client.deinit();

    for ([_][]const u8{ "/items", "/items/1", "/items/1/tags", "/a/b/c/d" }) |p| {
        const url = try std.fmt.allocPrint(gpa, "http://127.0.0.1:{d}{s}", .{ port, p });
        defer gpa.free(url);
        var body: std.Io.Writer.Allocating = .init(gpa);
        defer body.deinit();
        const res = try client.fetch(.{ .location = .{ .url = url }, .response_writer = &body.writer });
        std.debug.print("[client] {d} {s}\n", .{ @intFromEnum(res.status), body.written() });
    }
}
```

```
[server] /items arena capacity=990
[client] 200 {"path":"/items","segments":["items"]}
[server] /items/1 arena capacity=990
[client] 200 {"path":"/items/1","segments":["items","1"]}
[server] /items/1/tags arena capacity=990
[client] 200 {"path":"/items/1/tags","segments":["items","1","tags"]}
[server] /a/b/c/d arena capacity=990
[client] 200 {"path":"/a/b/c/d","segments":["a","b","c","d"]}
```

**Count the `free` calls in `handle`: zero.** An `ArrayList`, up to four
`allocPrint`s, a `join`, a `Stringify.valueAlloc`, and not one `defer`. It also
cannot leak on the error path, because the `defer reset` is above the `try`s.

**Capacity is 990 on every request.** After the first one, serving a request
costs zero allocations from the OS — the bump pointer just starts over at the
front of the pages it already owns.

## Measured against freeing individually

Same work twice: build three strings per "request", 20 000 times.

```zig
const std = @import("std");

const n_requests = 20_000;

fn render(a: std.mem.Allocator, id: u32) ![]const u8 {
    const id_str = try std.fmt.allocPrint(a, "{d}", .{id});
    const title = try std.fmt.allocPrint(a, "item number {d}", .{id});
    const json = try std.fmt.allocPrint(a, "{{\"id\":\"{s}\",\"title\":\"{s}\"}}", .{ id_str, title });
    a.free(title);
    a.free(id_str);
    return json;
}

fn withArena(gpa: std.mem.Allocator) !usize {
    var arena_s: std.heap.ArenaAllocator = .init(gpa);
    defer arena_s.deinit();
    var total: usize = 0;
    for (0..n_requests) |i| {
        defer _ = arena_s.reset(.retain_capacity);
        const body = try render(arena_s.allocator(), @intCast(i));
        total += body.len;
    }
    return total;
}

fn withGpa(gpa: std.mem.Allocator) !usize {
    var total: usize = 0;
    for (0..n_requests) |i| {
        const body = try render(gpa, @intCast(i));
        defer gpa.free(body);
        total += body.len;
    }
    return total;
}

fn micros(io: std.Io, from: std.Io.Clock.Timestamp) u64 {
    return @intCast(from.durationTo(.now(io, .awake)).raw.toMicroseconds());
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer _ = gpa_s.deinit();
    const gpa = gpa_s.allocator();

    const t0: std.Io.Clock.Timestamp = .now(io, .awake);
    const a = try withGpa(gpa);
    const gpa_us = micros(io, t0);

    const t1: std.Io.Clock.Timestamp = .now(io, .awake);
    const b = try withArena(gpa);
    const arena_us = micros(io, t1);

    std.debug.print("{d} requests, free each alloc : {d:>7} us (bytes={d})\n", .{ n_requests, gpa_us, a });
    std.debug.print("{d} requests, arena reset     : {d:>7} us (bytes={d})\n", .{ n_requests, arena_us, b });
}
```

```sh
zig run -OReleaseSafe bench.zig
```

```
20000 requests, free each alloc :  464669 us (bytes=817780)
20000 requests, arena reset     :    1616 us (bytes=817780)
```

**~290x**, identical bytes produced. (Numbers move per run and per machine; the
order of magnitude doesn't.) The arena version is also *shorter* — the
`a.free(title)` / `a.free(id_str)` lines exist only to make the gpa version
correct, and they are precisely the lines that rot when someone adds an early
`return` later.

The reason is not that `free` is slow. It's that:

| Per allocation | gpa | arena |
|---|---|---|
| work | walk size classes, update free list, record a trace in Debug | add to an offset, bounds check |
| per-request teardown | N frees, each one of the above | one pointer assignment |
| can leak | yes, if a path skips a `free` | **no** |

That last row is why this is idiomatic rather than merely fast.

## What must outlive the request

Everything from the arena dies at `reset`. So anything the **store** keeps must
come from the gpa. Put an arena-owned key into a long-lived map and you get
this:

```zig
const std = @import("std");

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer _ = gpa_s.deinit();
    const gpa = gpa_s.allocator();

    var arena_s: std.heap.ArenaAllocator = .init(gpa);
    defer arena_s.deinit();

    var wrong: std.StringHashMap([]const u8) = .init(gpa);
    defer wrong.deinit();
    var right: std.StringHashMap([]const u8) = .init(gpa);
    defer {
        var it = right.iterator();
        while (it.next()) |e| {
            gpa.free(e.key_ptr.*);
            gpa.free(e.value_ptr.*);
        }
        right.deinit();
    }

    for (0..3) |req| {
        defer _ = arena_s.reset(.retain_capacity);
        const arena = arena_s.allocator();

        const id = try std.fmt.allocPrint(arena, "id-{d}", .{req});
        const title = try std.fmt.allocPrint(arena, "title {d}", .{req});

        try wrong.put(id, title);
        try right.put(try gpa.dupe(u8, id), try gpa.dupe(u8, title));
    }

    std.debug.print("arena-owned keys: count={d} (3 puts)\n", .{wrong.count()});
    var wit = wrong.iterator();
    while (wit.next()) |e| std.debug.print("  {s} -> {s}\n", .{ e.key_ptr.*, e.value_ptr.* });

    std.debug.print("gpa-owned keys:   count={d} (3 puts)\n", .{right.count()});
    var rit = right.iterator();
    while (rit.next()) |e| std.debug.print("  {s} -> {s}\n", .{ e.key_ptr.*, e.value_ptr.* });
}
```

```
arena-owned keys: count=2 (3 puts)
  id-2 -> title 2
  id-2 -> title 2
gpa-owned keys:   count=3 (3 puts)
  id-0 -> title 0
  id-1 -> title 1
  id-2 -> title 2
```

Three `put`s, **two** entries, both of them `id-2`. No crash, no panic, and the
DebugAllocator reports nothing, because nothing leaked — the arena simply handed
request 2 the same addresses it gave request 0, and the map's keys changed under
it. One of the three got the same hash slot twice and was overwritten; the other
two now alias the same bytes.

That is the worst class of bug: silent, data-dependent, and invisible to the
leak checker. **`StringHashMap` does not copy keys.** The rule:

| Lives for | Allocator |
|---|---|
| this request only — parsed JSON, response body, formatted strings, intermediate buffers | **arena** |
| longer than this request — anything stored, anything in the config | **gpa**, with an explicit owner who frees it |

14.4 is that rule applied to the store.

## `reset` modes

| Mode | After | Use |
|---|---|---|
| `.retain_capacity` | capacity unchanged | **per request** — this is the one |
| `.free_all` | 0 | idle service that should give memory back |
| `.{ .retain_with_limit = n }` | ≤ n | per request, but cap the high-water mark from one huge upload |

`reset` returns `bool` (`false` = the retaining re-allocation failed and it fell
back to empty). It is safe either way, hence `_ = arena_s.reset(...)`.

For a service that accepts large bodies, `.{ .retain_with_limit = 64 * 1024 }`
is the safer default: with plain `.retain_capacity` one 10 MB request leaves the
arena holding 10 MB for the rest of the process's life.

---

Next: [14.4 — Store & State](./04-store.md)
