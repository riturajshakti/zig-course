# 14.7 — Tests & Release

Three layers, each cheaper than the one above it:

| Layer | Needs | Count | Catches |
|---|---|---|---|
| pure unit (`router`, `api`, `config`) | nothing | 7 | path parsing, status mapping, config validation |
| handler (`handlers`, `store`) | `testing.allocator` + a `Threaded` io | 5 | business logic, **leaks**, arena/store lifetimes |
| end to end (`e2e_test`) | a loopback port | 1 (12 calls) | HTTP framing, status codes, real wire bytes |

## Handler tests with no socket

Because `dispatch` takes a `Request` value and returns a `Response` value
(14.1), the whole API is testable without a port. One fixture holds the three
things a request needs:

```zig
const TestEnv = struct {
    threaded: std.Io.Threaded,
    arena_s: std.heap.ArenaAllocator,
    store: Store,

    fn init() TestEnv {
        return .{
            .threaded = .init(std.testing.allocator, .{}),
            .arena_s = .init(std.testing.allocator),
            .store = .init(std.testing.allocator),
        };
    }
    fn deinit(e: *TestEnv) void {
        e.store.deinit();
        e.arena_s.deinit();
        e.threaded.deinit();
    }
    fn call(e: *TestEnv, req: Request) !Response {
        defer _ = e.arena_s.reset(.retain_capacity);
        return dispatch(e.arena_s.allocator(), e.threaded.io(), &e.store, req);
    }
};
```

| Piece | Why a test needs it |
|---|---|
| `std.Io.Threaded.init(testing.allocator, .{})` | `Store` takes `io` for its mutex; `init.io` doesn't exist in a test |
| `testing.allocator` for the arena's backing | the arena's pages are checked for leaks too |
| `testing.allocator` for the store | **a leaked item fails the test** |
| `defer reset` inside `call` | one `call` = one request, including the arena reset |

`deinit` order mirrors the server's (14.6): store last.

The three tests this enables:

```zig
test "handlers without a socket" {
    const testing = std.testing;
    var env: TestEnv = .init();
    defer env.deinit();

    const health = try env.call(.{ .method = .GET, .target = "/health" });
    try testing.expectEqual(std.http.Status.ok, health.status);

    const empty = try env.call(.{ .method = .GET, .target = "/items" });
    try testing.expectEqualStrings("[]", empty.body);

    const created = try env.call(.{
        .method = .POST,
        .target = "/items",
        .body = "{\"title\":\"from a unit test\"}",
    });
    try testing.expectEqual(std.http.Status.created, created.status);

    const one = try env.call(.{ .method = .GET, .target = "/items/1" });
    try testing.expectEqualStrings("{\"id\":\"1\",\"title\":\"from a unit test\",\"done\":false}", one.body);

    const gone = try env.call(.{ .method = .DELETE, .target = "/items/1" });
    try testing.expectEqual(std.http.Status.no_content, gone.status);
    try testing.expectEqualStrings("", gone.body);
}

test "handler error set maps onto the api errors" {
    const testing = std.testing;
    var env: TestEnv = .init();
    defer env.deinit();

    try testing.expectError(error.NotFound, env.call(.{ .method = .GET, .target = "/items/7" }));
    try testing.expectError(error.NotFound, env.call(.{ .method = .GET, .target = "/nope" }));
    try testing.expectError(error.MethodNotAllowed, env.call(.{ .method = .PUT, .target = "/items" }));
    try testing.expectError(error.BadRequest, env.call(.{ .method = .POST, .target = "/items", .body = "oops" }));
    try testing.expectError(error.BadRequest, env.call(.{ .method = .POST, .target = "/items", .body = "{\"title\":\"\"}" }));
}

test "the arena is reset between requests but the store is not" {
    var env: TestEnv = .init();
    defer env.deinit();

    for (0..50) |i| {
        var buf: [64]u8 = undefined;
        const body = try std.fmt.bufPrint(&buf, "{{\"title\":\"item {d}\"}}", .{i});
        _ = try env.call(.{ .method = .POST, .target = "/items", .body = body });
    }
    try std.testing.expectEqual(@as(u32, 50), env.store.items.count());
    try std.testing.expect(env.arena_s.queryCapacity() < 8 * 1024);
}
```

That last test is the one worth stealing. Fifty requests, fifty items kept, and
the arena still under 8 KiB — it asserts 14.3's and 14.4's invariants *as a
test*, so the day someone stores an arena-owned string the capacity assertion
or the leak check fires.

## End to end over loopback

Twelve calls through the real HTTP stack, against the real server:

```zig
    var server = try Server.init(io, gpa, .{ .port = port, .max_requests = calls.len });
    defer server.deinit();

    var task = io.async(Server.run, .{&server});
    defer task.await(io);
```

`max_requests = calls.len` is what makes this terminate: `run` exits on its own
after the twelfth response, `task.await` joins it, and `deinit` closes the
listener — no cancelation needed, no infinite accept loop, no sleep-and-hope.

**Collect failures, don't `try` inside the loop.** If an assertion returns early
on call 5, the server is still parked in `accept` waiting for calls 6–12 and the
test **hangs forever** instead of failing. Count mismatches, print them, assert
once at the end:

```zig
        const ok = got == want[0] and std.mem.eql(u8, want[1], body.written());
        if (!ok) {
            failures += 1;
            std.debug.print("FAIL {t} {s} -> {d} {s}\n", .{ c.method, c.path, got, body.written() });
        }
    }
    try testing.expectEqual(@as(usize, 0), failures);
```

## `zig build test`

`root.zig` is a test-only root that pulls every module in:

```zig
const std = @import("std");

pub const api = @import("api.zig");
pub const config = @import("config.zig");
pub const handlers = @import("handlers.zig");
pub const router = @import("router.zig");
pub const server = @import("server.zig");
pub const store = @import("store.zig");

test {
    std.testing.refAllDecls(@This());
}

test {
    _ = @import("e2e_test.zig");
}
```

> One formatting note before you run `zig fmt` on this project: it rewrites
> `@intFromEnum(x)` to `@backingInt(x)` in place. Both spellings compile and
> mean the same thing, so a `zig fmt --check` failure on that line is cosmetic.

`refAllDecls` is **load-bearing**: without it the tests inside the imported
files are silently skipped and the run still prints "All 2 tests passed". The
second `test` block exists because `e2e_test.zig` is not a `pub const` above —
an `_ = @import(...)` inside a `test` is how you pull in a file that exports
nothing.

```sh
zig build test
```

```
Build Summary: 3/3 steps succeeded; 15/15 tests passed
test success
+- run test 15 pass (15 total) 487ms MaxRSS:4M
   +- compile test debug native success 3s MaxRSS:537M
```

Per-test names come from running the root directly:

```sh
zig test src/root.zig
```

```
1/15 root.test_0...OK
2/15 root.test_1...OK
3/15 e2e_test.test.end to end over loopback...OK
4/15 api.test.every api error maps to a 4xx or 5xx...OK
5/15 api.test.internal errors never leak their name...OK
6/15 config.test.defaults...OK
7/15 config.test.address validates the host...OK
8/15 handlers.test.handlers without a socket...OK
9/15 handlers.test.handler error set maps onto the api errors...OK
10/15 handlers.test.the arena is reset between requests but the store is not...OK
11/15 router.test.static routes...OK
12/15 router.test.404 and 405...OK
13/15 router.test.path param...OK
14/15 store.test.crud round trip, leak-checked...OK
15/15 store.test.ids keep rising after a delete...OK
All 15 tests passed.
```

Every one of those uses `testing.allocator`, so **15/15 passing is also a
leak-free claim** for the store, the arena, and the HTTP client.

## Release builds, measured

```sh
zig build -Doptimize=ReleaseSafe -Dstrip
```

| Mode | Bytes | `-Dstrip` |
|---|---|---|
| Debug | 2,583,528 | 1,289,096 |
| ReleaseSafe | 655,048 | **371,400** |
| ReleaseFast | 536,472 | 305,928 |
| ReleaseSmall | 494,744 | 256,312 |

**Ship `ReleaseSafe`.** For a network service the 20% over `ReleaseFast` buys
bounds checks, overflow checks and `unreachable` turning into a panic instead of
undefined behaviour — on a path fed by untrusted input. `ReleaseFast` is for
code whose inputs you already validated.

One mode-dependent behaviour matters here: `DebugAllocator`'s leak detection
**disappears** in `ReleaseFast`/`ReleaseSmall`, because its `safety` default
follows `std.debug.runtime_safety`. `deinit()` returns `.ok` on a real leak. In
`ReleaseSafe` it still reports. `std.testing.allocator` keeps checking in all
four modes, which is why the test suite is the real leak gate.

And one static Linux binary from the same macOS laptop:

```sh
zig build -Doptimize=ReleaseSafe -Dtarget=x86_64-linux-musl -Dstrip
file zig-out/bin/itemsvc
```

```
zig-out/bin/itemsvc: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, stripped
```

**470,640 bytes, no libc, no SDK, no container to build it.** `FROM scratch`
plus that file is the whole image.

## The complete service

Eight files. Copy them into a directory and `zig build run` works.

### `build.zig.zon`

```zig
.{
    .name = .itemsvc,
    .version = "0.1.0",
    .minimum_zig_version = "0.17.0",
    .fingerprint = 0x700676e732499b83,
    .paths = .{ "build.zig", "build.zig.zon", "src" },
    .dependencies = .{},
}
```

### `build.zig`

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    const strip = b.option(bool, "strip", "Omit debug info") orelse false;

    const exe = b.addExecutable(.{
        .name = "itemsvc",
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            .target = target,
            .optimize = optimize,
            .strip = strip,
        }),
    });
    b.installArtifact(exe);

    const run = b.addRunArtifact(exe);
    run.addPassthruArgs();
    b.step("run", "Run the service").dependOn(&run.step);

    const unit = b.addTest(.{
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/root.zig"),
            .target = target,
            .optimize = optimize,
        }),
    });
    b.step("test", "Run all tests").dependOn(&b.addRunArtifact(unit).step);
}
```

### `src/main.zig`

```zig
const std = @import("std");
const Config = @import("config.zig").Config;
const Server = @import("server.zig").Server;

pub fn main(init: std.process.Init) !void {
    const cfg = Config.load(init.environ_map, init.minimal.args) catch |e|
        std.process.fatal("configuration: {t}", .{e});

    var server = try Server.init(init.io, init.gpa, cfg);
    defer server.deinit();

    std.log.info("listening on {s}:{d} (max_requests={d})", .{ cfg.host, cfg.port, cfg.max_requests });

    server.run();
    std.log.info("stopped after {d} requests", .{server.served});
}
```

### `src/config.zig`

```zig
const std = @import("std");

pub const Config = struct {
    host: []const u8 = "127.0.0.1",
    port: u16 = 8080,
    max_requests: usize = 0,
    body_limit: usize = 64 * 1024,

    pub const Error = error{ BadHost, BadPort, BadMaxRequests, UnknownFlag };

    pub fn load(env: *std.process.Environ.Map, args: std.process.Args) Error!Config {
        var cfg: Config = .{};
        if (env.get("ITEMSVC_HOST")) |v| cfg.host = v;
        if (env.get("ITEMSVC_PORT")) |v| cfg.port = std.fmt.parseInt(u16, v, 10) catch return error.BadPort;
        if (env.get("ITEMSVC_MAX_REQUESTS")) |v| cfg.max_requests = std.fmt.parseInt(usize, v, 10) catch return error.BadMaxRequests;

        var it = args.iterate();
        _ = it.next();
        while (it.next()) |arg| {
            if (value(arg, "--host=")) |v| {
                cfg.host = v;
            } else if (value(arg, "--port=")) |v| {
                cfg.port = std.fmt.parseInt(u16, v, 10) catch return error.BadPort;
            } else if (value(arg, "--max-requests=")) |v| {
                cfg.max_requests = std.fmt.parseInt(usize, v, 10) catch return error.BadMaxRequests;
            } else return error.UnknownFlag;
        }
        _ = try cfg.address();
        return cfg;
    }

    pub fn address(cfg: Config) Error!std.Io.net.IpAddress {
        const ip4 = std.Io.net.Ip4Address.parse(cfg.host, cfg.port) catch return error.BadHost;
        return .{ .ip4 = ip4 };
    }

    fn value(arg: []const u8, prefix: []const u8) ?[]const u8 {
        return if (std.mem.startsWith(u8, arg, prefix)) arg[prefix.len..] else null;
    }
};

test "defaults" {
    const cfg: Config = .{};
    try std.testing.expectEqual(@as(u16, 8080), cfg.port);
    try std.testing.expectEqualStrings("127.0.0.1", cfg.host);
    try std.testing.expectEqual(@as(usize, 0), cfg.max_requests);
}

test "address validates the host" {
    const testing = std.testing;
    const ok = try (Config{ .host = "10.0.0.7", .port = 9000 }).address();
    try testing.expectEqualSlices(u8, &.{ 10, 0, 0, 7 }, &ok.ip4.bytes);
    try testing.expectEqual(@as(u16, 9000), ok.ip4.port);

    try testing.expectError(error.BadHost, (Config{ .host = "10.0.0" }).address());
    try testing.expectError(error.BadHost, (Config{ .host = "10.0.0.999" }).address());
    try testing.expectError(error.BadHost, (Config{ .host = "localhost" }).address());
}
```

### `src/router.zig`

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

test "static routes" {
    const testing = std.testing;
    try testing.expectEqual(Route.health, route(.GET, "/health"));
    try testing.expectEqual(Route.list_items, route(.GET, "/items"));
    try testing.expectEqual(Route.create_item, route(.POST, "/items"));
    try testing.expectEqual(Route.list_items, route(.GET, "/items?limit=5"));
}

test "path param" {
    try std.testing.expectEqualStrings("abc", route(.GET, "/items/abc").get_item);
    try std.testing.expectEqualStrings("abc", route(.DELETE, "/items/abc").delete_item);
}

test "404 and 405" {
    const testing = std.testing;
    try testing.expectEqual(Route.not_found, route(.GET, "/"));
    try testing.expectEqual(Route.not_found, route(.GET, "/items/"));
    try testing.expectEqual(Route.not_found, route(.GET, "/items/1/tags"));
    try testing.expectEqual(Route.method_not_allowed, route(.POST, "/health"));
    try testing.expectEqual(Route.method_not_allowed, route(.PUT, "/items"));
    try testing.expectEqual(Route.method_not_allowed, route(.PATCH, "/items/1"));
}
```

### `src/api.zig`

```zig
const std = @import("std");

pub const Error = error{
    NotFound,
    BadRequest,
    MethodNotAllowed,
    PayloadTooLarge,
    Internal,
};

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

test "every api error maps to a 4xx or 5xx" {
    const testing = std.testing;
    try testing.expectEqual(std.http.Status.not_found, classify(error.NotFound).status);
    try testing.expectEqual(std.http.Status.bad_request, classify(error.SyntaxError).status);
    try testing.expectEqual(std.http.Status.method_not_allowed, classify(error.MethodNotAllowed).status);
    try testing.expectEqual(std.http.Status.payload_too_large, classify(error.PayloadTooLarge).status);
    try testing.expectEqual(std.http.Status.internal_server_error, classify(error.Canceled).status);
}

test "internal errors never leak their name" {
    var buf: [64]u8 = undefined;
    const f = classify(error.DatabaseOnFire);
    try std.testing.expectEqualStrings("{\"error\":\"internal\"}", errorBody(&buf, f.code));
}
```

### `src/store.zig`

```zig
const std = @import("std");

pub const Item = struct {
    id: []const u8,
    title: []const u8,
    done: bool = false,
};

pub const Store = struct {
    gpa: std.mem.Allocator,
    mutex: std.Io.Mutex = .init,
    items: std.StringHashMap(Item),
    next_id: u32 = 1,

    pub const Error = error{ NotFound, OutOfMemory, Canceled };

    pub fn init(gpa: std.mem.Allocator) Store {
        return .{ .gpa = gpa, .items = .init(gpa) };
    }

    pub fn deinit(s: *Store) void {
        var it = s.items.valueIterator();
        while (it.next()) |item| {
            s.gpa.free(item.id);
            s.gpa.free(item.title);
        }
        s.items.deinit();
    }

    pub fn create(s: *Store, io: std.Io, title: []const u8, done: bool) Error!Item {
        try s.mutex.lock(io);
        defer s.mutex.unlock(io);

        const id = try std.fmt.allocPrint(s.gpa, "{d}", .{s.next_id});
        errdefer s.gpa.free(id);
        const owned = try s.gpa.dupe(u8, title);
        errdefer s.gpa.free(owned);

        const item: Item = .{ .id = id, .title = owned, .done = done };
        try s.items.put(id, item);
        s.next_id += 1;
        return item;
    }

    pub fn get(s: *Store, io: std.Io, id: []const u8) Error!Item {
        try s.mutex.lock(io);
        defer s.mutex.unlock(io);
        return s.items.get(id) orelse error.NotFound;
    }

    pub fn remove(s: *Store, io: std.Io, id: []const u8) Error!void {
        try s.mutex.lock(io);
        defer s.mutex.unlock(io);
        const kv = s.items.fetchRemove(id) orelse return error.NotFound;
        s.gpa.free(kv.value.id);
        s.gpa.free(kv.value.title);
    }

    pub fn list(s: *Store, io: std.Io, arena: std.mem.Allocator) Error![]Item {
        try s.mutex.lock(io);
        defer s.mutex.unlock(io);

        const out = try arena.alloc(Item, s.items.count());
        var i: usize = 0;
        var it = s.items.valueIterator();
        while (it.next()) |item| : (i += 1) out[i] = item.*;
        std.mem.sort(Item, out, {}, byId);
        return out;
    }

    fn byId(_: void, a: Item, b: Item) bool {
        if (a.id.len != b.id.len) return a.id.len < b.id.len;
        return std.mem.lessThan(u8, a.id, b.id);
    }
};

const TestIo = struct {
    threaded: std.Io.Threaded,

    fn init() TestIo {
        return .{ .threaded = .init(std.testing.allocator, .{}) };
    }
    fn deinit(t: *TestIo) void {
        t.threaded.deinit();
    }
};

test "crud round trip, leak-checked" {
    const testing = std.testing;
    var t: TestIo = .init();
    defer t.deinit();
    const io = t.threaded.io();

    var store: Store = .init(testing.allocator);
    defer store.deinit();

    const a = try store.create(io, "first", false);
    const b = try store.create(io, "second", true);
    try testing.expectEqualStrings("1", a.id);
    try testing.expectEqualStrings("2", b.id);

    try testing.expectEqualStrings("second", (try store.get(io, "2")).title);
    try testing.expectError(error.NotFound, store.get(io, "99"));

    var arena_s: std.heap.ArenaAllocator = .init(testing.allocator);
    defer arena_s.deinit();
    const all = try store.list(io, arena_s.allocator());
    try testing.expectEqual(@as(usize, 2), all.len);
    try testing.expectEqualStrings("1", all[0].id);

    try store.remove(io, "1");
    try testing.expectError(error.NotFound, store.remove(io, "1"));
    try testing.expectEqual(@as(u32, 1), store.items.count());
}

test "ids keep rising after a delete" {
    var t: TestIo = .init();
    defer t.deinit();
    const io = t.threaded.io();

    var store: Store = .init(std.testing.allocator);
    defer store.deinit();

    _ = try store.create(io, "x", false);
    try store.remove(io, "1");
    const next = try store.create(io, "y", false);
    try std.testing.expectEqualStrings("2", next.id);
}
```

### `src/handlers.zig`

```zig
const std = @import("std");
const router = @import("router.zig");
const Store = @import("store.zig").Store;
const Item = @import("store.zig").Item;

pub const NewItem = struct { title: []const u8, done: bool = false };

pub const Response = struct {
    status: std.http.Status,
    body: []const u8,
};

pub const Request = struct {
    method: std.http.Method,
    target: []const u8,
    body: []const u8 = "",
};

pub fn dispatch(arena: std.mem.Allocator, io: std.Io, store: *Store, req: Request) !Response {
    return switch (router.route(req.method, req.target)) {
        .health => .{ .status = .ok, .body = "{\"status\":\"ok\"}" },

        .list_items => .{
            .status = .ok,
            .body = try std.json.Stringify.valueAlloc(arena, try store.list(io, arena), .{}),
        },

        .get_item => |id| .{
            .status = .ok,
            .body = try std.json.Stringify.valueAlloc(arena, try store.get(io, id), .{}),
        },

        .create_item => blk: {
            const parsed = std.json.parseFromSlice(NewItem, arena, req.body, .{
                .ignore_unknown_fields = true,
            }) catch return error.BadRequest;
            if (parsed.value.title.len == 0) return error.BadRequest;

            const item = try store.create(io, parsed.value.title, parsed.value.done);
            break :blk .{
                .status = .created,
                .body = try std.json.Stringify.valueAlloc(arena, item, .{}),
            };
        },

        .delete_item => |id| blk: {
            try store.remove(io, id);
            break :blk .{ .status = .no_content, .body = "" };
        },

        .not_found => error.NotFound,
        .method_not_allowed => error.MethodNotAllowed,
    };
}

const TestEnv = struct {
    threaded: std.Io.Threaded,
    arena_s: std.heap.ArenaAllocator,
    store: Store,

    fn init() TestEnv {
        return .{
            .threaded = .init(std.testing.allocator, .{}),
            .arena_s = .init(std.testing.allocator),
            .store = .init(std.testing.allocator),
        };
    }
    fn deinit(e: *TestEnv) void {
        e.store.deinit();
        e.arena_s.deinit();
        e.threaded.deinit();
    }
    fn call(e: *TestEnv, req: Request) !Response {
        defer _ = e.arena_s.reset(.retain_capacity);
        return dispatch(e.arena_s.allocator(), e.threaded.io(), &e.store, req);
    }
};

test "handlers without a socket" {
    const testing = std.testing;
    var env: TestEnv = .init();
    defer env.deinit();

    const health = try env.call(.{ .method = .GET, .target = "/health" });
    try testing.expectEqual(std.http.Status.ok, health.status);

    const empty = try env.call(.{ .method = .GET, .target = "/items" });
    try testing.expectEqualStrings("[]", empty.body);

    const created = try env.call(.{
        .method = .POST,
        .target = "/items",
        .body = "{\"title\":\"from a unit test\"}",
    });
    try testing.expectEqual(std.http.Status.created, created.status);

    const one = try env.call(.{ .method = .GET, .target = "/items/1" });
    try testing.expectEqualStrings("{\"id\":\"1\",\"title\":\"from a unit test\",\"done\":false}", one.body);

    const gone = try env.call(.{ .method = .DELETE, .target = "/items/1" });
    try testing.expectEqual(std.http.Status.no_content, gone.status);
    try testing.expectEqualStrings("", gone.body);
}

test "handler error set maps onto the api errors" {
    const testing = std.testing;
    var env: TestEnv = .init();
    defer env.deinit();

    try testing.expectError(error.NotFound, env.call(.{ .method = .GET, .target = "/items/7" }));
    try testing.expectError(error.NotFound, env.call(.{ .method = .GET, .target = "/nope" }));
    try testing.expectError(error.MethodNotAllowed, env.call(.{ .method = .PUT, .target = "/items" }));
    try testing.expectError(error.BadRequest, env.call(.{ .method = .POST, .target = "/items", .body = "oops" }));
    try testing.expectError(error.BadRequest, env.call(.{ .method = .POST, .target = "/items", .body = "{\"title\":\"\"}" }));
}

test "the arena is reset between requests but the store is not" {
    var env: TestEnv = .init();
    defer env.deinit();

    for (0..50) |i| {
        var buf: [64]u8 = undefined;
        const body = try std.fmt.bufPrint(&buf, "{{\"title\":\"item {d}\"}}", .{i});
        _ = try env.call(.{ .method = .POST, .target = "/items", .body = body });
    }
    try std.testing.expectEqual(@as(u32, 50), env.store.items.count());
    try std.testing.expect(env.arena_s.queryCapacity() < 8 * 1024);
}
```

### `src/server.zig`

```zig
const std = @import("std");
const api = @import("api.zig");
const handlers = @import("handlers.zig");
const Config = @import("config.zig").Config;
const Store = @import("store.zig").Store;

pub const Server = struct {
    io: std.Io,
    gpa: std.mem.Allocator,
    cfg: Config,
    store: Store,
    listener: std.Io.net.Server,
    arena_s: std.heap.ArenaAllocator,
    stopping: std.atomic.Value(bool) = .init(false),
    served: usize = 0,

    pub fn init(io: std.Io, gpa: std.mem.Allocator, cfg: Config) !Server {
        return .{
            .io = io,
            .gpa = gpa,
            .cfg = cfg,
            .store = .init(gpa),
            .listener = try (try cfg.address()).listen(io, .{ .reuse_address = true }),
            .arena_s = .init(gpa),
        };
    }

    pub fn deinit(s: *Server) void {
        s.listener.deinit(s.io);
        s.arena_s.deinit();
        s.store.deinit();
    }

    pub fn run(s: *Server) void {
        while (!s.stopping.load(.acquire)) {
            if (s.cfg.max_requests != 0 and s.served >= s.cfg.max_requests) break;

            var stream = s.listener.accept(s.io) catch break;
            defer {
                stream.close(s.io);
                _ = s.arena_s.reset(.retain_capacity);
                s.served += 1;
            }
            s.serveConnection(stream);
        }
    }

    pub fn stop(s: *Server) void {
        s.stopping.store(true, .release);
    }

    fn serveConnection(s: *Server, stream: std.Io.net.Stream) void {
        var rbuf: [8192]u8 = undefined;
        var wbuf: [8192]u8 = undefined;
        var sr = stream.reader(s.io, &rbuf);
        var sw = stream.writer(s.io, &wbuf);

        var http: std.http.Server = .init(&sr.interface, &sw.interface);
        var req = http.receiveHead() catch return;

        const arena = s.arena_s.allocator();
        if (s.handle(arena, &req)) |res| {
            req.respond(res.body, .{
                .status = res.status,
                .keep_alive = false,
                .extra_headers = &.{.{ .name = "content-type", .value = "application/json" }},
            }) catch {};
        } else |err| {
            const f = api.classify(err);
            if (f.status == .internal_server_error)
                std.log.err("unhandled: {t} on {t} {s}", .{ err, req.head.method, req.head.target });
            var buf: [64]u8 = undefined;
            req.respond(api.errorBody(&buf, f.code), .{
                .status = f.status,
                .keep_alive = false,
                .extra_headers = &.{.{ .name = "content-type", .value = "application/json" }},
            }) catch {};
        }
    }

    fn handle(s: *Server, arena: std.mem.Allocator, req: *std.http.Server.Request) !handlers.Response {
        const method = req.head.method;
        const target = try arena.dupe(u8, req.head.target);

        var body: []const u8 = "";
        if (method.requestHasBody()) {
            if ((req.head.content_length orelse 0) > s.cfg.body_limit) return error.PayloadTooLarge;
            var bbuf: [4096]u8 = undefined;
            const reader = try req.readerExpectContinue(&bbuf);
            body = try reader.allocRemaining(arena, .limited(s.cfg.body_limit));
        }

        return handlers.dispatch(arena, s.io, &s.store, .{
            .method = method,
            .target = target,
            .body = body,
        });
    }
};
```

### `src/root.zig`

```zig
const std = @import("std");

pub const api = @import("api.zig");
pub const config = @import("config.zig");
pub const handlers = @import("handlers.zig");
pub const router = @import("router.zig");
pub const server = @import("server.zig");
pub const store = @import("store.zig");

test {
    std.testing.refAllDecls(@This());
}

test {
    _ = @import("e2e_test.zig");
}
```

### `src/e2e_test.zig`

```zig
const std = @import("std");
const testing = std.testing;
const Config = @import("config.zig").Config;
const Server = @import("server.zig").Server;

const port = 39257;

const Call = struct {
    method: std.http.Method,
    path: []const u8,
    payload: ?[]const u8 = null,
};

test "end to end over loopback" {
    var threaded: std.Io.Threaded = .init(testing.allocator, .{});
    defer threaded.deinit();
    const io = threaded.io();
    const gpa = testing.allocator;

    const calls = [_]Call{
        .{ .method = .GET, .path = "/health" },
        .{ .method = .GET, .path = "/items" },
        .{ .method = .POST, .path = "/items", .payload = "{\"title\":\"write the service\"}" },
        .{ .method = .POST, .path = "/items", .payload = "{\"title\":\"ship it\",\"done\":true}" },
        .{ .method = .GET, .path = "/items" },
        .{ .method = .GET, .path = "/items/1" },
        .{ .method = .GET, .path = "/items/404" },
        .{ .method = .POST, .path = "/items", .payload = "{}" },
        .{ .method = .PUT, .path = "/items", .payload = "{}" },
        .{ .method = .DELETE, .path = "/items/1" },
        .{ .method = .DELETE, .path = "/items/1" },
        .{ .method = .GET, .path = "/items" },
    };

    var server = try Server.init(io, gpa, .{ .port = port, .max_requests = calls.len });
    defer server.deinit();

    var task = io.async(Server.run, .{&server});
    defer task.await(io);

    var client: std.http.Client = .{ .allocator = gpa, .io = io };
    defer client.deinit();

    const expected = [_]struct { u16, []const u8 }{
        .{ 200, "{\"status\":\"ok\"}" },
        .{ 200, "[]" },
        .{ 201, "{\"id\":\"1\",\"title\":\"write the service\",\"done\":false}" },
        .{ 201, "{\"id\":\"2\",\"title\":\"ship it\",\"done\":true}" },
        .{ 200, "[{\"id\":\"1\",\"title\":\"write the service\",\"done\":false},{\"id\":\"2\",\"title\":\"ship it\",\"done\":true}]" },
        .{ 200, "{\"id\":\"1\",\"title\":\"write the service\",\"done\":false}" },
        .{ 404, "{\"error\":\"not_found\"}" },
        .{ 400, "{\"error\":\"bad_request\"}" },
        .{ 405, "{\"error\":\"method_not_allowed\"}" },
        .{ 204, "" },
        .{ 404, "{\"error\":\"not_found\"}" },
        .{ 200, "[{\"id\":\"2\",\"title\":\"ship it\",\"done\":true}]" },
    };

    var failures: usize = 0;
    for (calls, expected) |c, want| {
        const url = try std.fmt.allocPrint(gpa, "http://127.0.0.1:{d}{s}", .{ port, c.path });
        defer gpa.free(url);

        var body: std.Io.Writer.Allocating = .init(gpa);
        defer body.deinit();

        const res = try client.fetch(.{
            .location = .{ .url = url },
            .method = c.method,
            .payload = c.payload,
            .response_writer = &body.writer,
        });
        const got = @intFromEnum(res.status);
        const ok = got == want[0] and std.mem.eql(u8, want[1], body.written());
        if (!ok) {
            failures += 1;
            std.debug.print("FAIL {t} {s} -> {d} {s}\n", .{ c.method, c.path, got, body.written() });
        }
    }
    try testing.expectEqual(@as(usize, 0), failures);
}
```

### Running it

```sh
zig build
ITEMSVC_PORT=39258 ITEMSVC_MAX_REQUESTS=8 ./zig-out/bin/itemsvc
```

```
info: listening on 127.0.0.1:39258 (max_requests=8)
```

In another shell:

```
$ curl -s http://127.0.0.1:39258/health
{"status":"ok"}
$ curl -s -XPOST -d '{"title":"finish module 14"}' http://127.0.0.1:39258/items
{"id":"1","title":"finish module 14","done":false}
$ curl -s -XPOST -d '{"title":"apply for jobs","done":true}' http://127.0.0.1:39258/items
{"id":"2","title":"apply for jobs","done":true}
$ curl -s http://127.0.0.1:39258/items
[{"id":"1","title":"finish module 14","done":false},{"id":"2","title":"apply for jobs","done":true}]
$ curl -s http://127.0.0.1:39258/items/1
{"id":"1","title":"finish module 14","done":false}
$ curl -s -o /dev/null -w '%{http_code}\n' -XDELETE http://127.0.0.1:39258/items/1
204
$ curl -s http://127.0.0.1:39258/items
[{"id":"2","title":"apply for jobs","done":true}]
$ curl -s http://127.0.0.1:39258/items/1
{"error":"not_found"}
```

```
info: stopped after 8 requests
```

Drop `ITEMSVC_MAX_REQUESTS` and it serves until you stop it.

## What you can say about this

| Question | Answer from this code |
|---|---|
| "How do you manage memory in a Zig service?" | arena per request, reset in a `defer`; gpa for the store, which dupes everything it keeps |
| "How do you avoid leaks?" | handlers never free, so they can't forget; `testing.allocator` fails the build on a leak in any of the 15 tests |
| "How do errors become responses?" | handlers return Zig errors; one `classify(anyerror)` maps them; the `else` arm is a logged 500 with a fixed body |
| "How do you test an HTTP service?" | handlers take and return values, so 5 tests need no socket; one e2e test drives the real server with `max_requests` so it terminates |
| "Shared state?" | `std.Io.Mutex` inside the store, `lock(io)` is fallible so `Canceled` is in the store's error set |
| "How do you ship it?" | `ReleaseSafe` + `-Dtarget=x86_64-linux-musl` → one 470 KB static binary, `FROM scratch` |

### Where to take it next

| Next step | Touches |
|---|---|
| one task per connection (`io.async` in `run`) | `server.zig` only — the mutex is already there |
| `PUT`/`PATCH` | `Route` (the compiler then finds every incomplete `switch`), `handlers`, `store` |
| persistence | `store.zig` behind the same four methods |
| metrics endpoint | a counter struct next to the store; `/metrics` in the router |
| keep-alive | `keep_alive = true` + a `receiveHead` loop per connection in `serveConnection` |

The first and the last are the two an interviewer is most likely to ask you to
sketch. Both are changes to `server.zig` alone, and that is the point of the
module boundaries in 14.1.

---

Module 14 complete. Next: [Module 15 — Zig for Systems & Backend Work](../15-systems-backend/)
