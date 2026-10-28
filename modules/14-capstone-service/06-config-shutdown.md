# 14.6 — Config & Shutdown

## Where config comes from

| Source | How | Wins over |
|---|---|---|
| defaults | struct field defaults | — |
| environment | `init.environ_map.get("ITEMSVC_PORT")` | defaults |
| args | `init.minimal.args.iterate()` | both |

| Variable | Flag | Default |
|---|---|---|
| `ITEMSVC_HOST` | `--host=` | `127.0.0.1` |
| `ITEMSVC_PORT` | `--port=` | `8080` |
| `ITEMSVC_MAX_REQUESTS` | `--max-requests=` | `0` (unlimited) |
| — | — | `body_limit = 64 KiB` |

## `src/config.zig`

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

pub fn main(init: std.process.Init) !void {
    const cfg = Config.load(init.environ_map, init.minimal.args) catch |e|
        std.process.fatal("configuration: {t}", .{e});
    std.debug.print("host={s} port={d} max_requests={d} body_limit={d}\n", .{
        cfg.host, cfg.port, cfg.max_requests, cfg.body_limit,
    });
}
```

```sh
zig build-exe config.zig -femit-bin=itemsvc
```

```
$ ./itemsvc
host=127.0.0.1 port=8080 max_requests=0 body_limit=65536
$ ITEMSVC_PORT=9000 ITEMSVC_HOST=0.0.0.0 ./itemsvc
host=0.0.0.0 port=9000 max_requests=0 body_limit=65536
$ ITEMSVC_PORT=9000 ./itemsvc --port=7777 --max-requests=5
host=127.0.0.1 port=7777 max_requests=5 body_limit=65536
$ ./itemsvc --port=http
error: configuration: BadPort
$ ITEMSVC_HOST=localhost ./itemsvc
error: configuration: BadHost
$ ./itemsvc --verbose
error: configuration: UnknownFlag
```

### Four details

**`init.environ_map` is a `*std.process.Environ.Map`.** `get(key)` returns
`?[]const u8` pointing into memory the runtime owns for the whole process, so
`cfg.host` can be a borrowed slice — no `dupe`, no allocator in this file at
all.

**`init.minimal.args` is *const*.** `args.iterate()` takes it by value; passing
`&init.minimal.args` to a `*std.process.Args` parameter fails with *"cast
discards const qualifier"*. Take `args: std.process.Args` by value.

**`_ = try cfg.address()` at the end of `load`.** Validate at startup, not at
`listen`. `ITEMSVC_HOST=localhost` is a config error reported in milliseconds,
not a confusing socket error after the store is already built. Note that
`Ip4Address.parse` means **no DNS** — `localhost` is rejected. That is
intentional for a bind address.

**`std.process.fatal`** prints `error: <msg>` and exits 1 with no stack trace.
Compare the alternative: plain `try` in `main` dumps a 7-frame trace through
`std.fmt.parseInt` for a typo'd port. A config error is the user's, not yours.

## Graceful shutdown

The problem: `accept` blocks. Setting a flag does nothing until the next
connection arrives, which may be never.

| Step | Call |
|---|---|
| 1. mark intent | `stop_flag.store(true, .release)` — from a signal handler or a request |
| 2. wake the parked `accept` | `task.cancel(io)` → `accept` returns `error.Canceled` |
| 3. finish in-flight work | nothing to do: `cancel` waits for the task, and the in-flight request is inside it |
| 4. close the listener | `listener.deinit(io)` |
| 5. release everything | arena, then store |

```zig
const std = @import("std");

const port = 39256;

var stop_flag: std.atomic.Value(bool) = .init(false);

fn onSignal(_: std.posix.SIG) callconv(.c) void {
    stop_flag.store(true, .release);
}

fn installSignalHandlers() void {
    const act: std.posix.Sigaction = .{
        .handler = .{ .handler = onSignal },
        .mask = std.posix.sigemptyset(),
        .flags = 0,
    };
    std.posix.sigaction(std.posix.SIG.INT, &act, null);
    std.posix.sigaction(std.posix.SIG.TERM, &act, null);
}

const Server = struct {
    io: std.Io,
    listener: std.Io.net.Server,
    arena_s: std.heap.ArenaAllocator,
    served: usize = 0,

    fn init(io: std.Io, gpa: std.mem.Allocator) !Server {
        const addr: std.Io.net.IpAddress = .{ .ip4 = .loopback(port) };
        return .{
            .io = io,
            .listener = try addr.listen(io, .{ .reuse_address = true }),
            .arena_s = .init(gpa),
        };
    }

    fn deinit(s: *Server) void {
        s.listener.deinit(s.io);
        s.arena_s.deinit();
        std.debug.print("[server] listener closed, arena released after {d} requests\n", .{s.served});
    }

    fn run(s: *Server) void {
        while (!stop_flag.load(.acquire)) {
            var stream = s.listener.accept(s.io) catch |e| {
                std.debug.print("[server] accept returned {t}; draining\n", .{e});
                break;
            };
            defer {
                stream.close(s.io);
                _ = s.arena_s.reset(.retain_capacity);
                s.served += 1;
            }

            var rbuf: [4096]u8 = undefined;
            var wbuf: [4096]u8 = undefined;
            var sr = stream.reader(s.io, &rbuf);
            var sw = stream.writer(s.io, &wbuf);
            var http: std.http.Server = .init(&sr.interface, &sw.interface);
            var req = http.receiveHead() catch continue;

            std.Io.sleep(s.io, .fromMilliseconds(30), .awake) catch {};
            req.respond("{\"status\":\"ok\"}", .{
                .keep_alive = false,
                .extra_headers = &.{.{ .name = "content-type", .value = "application/json" }},
            }) catch {};
        }
        std.debug.print("[server] accept loop exited, stop_flag={}\n", .{stop_flag.load(.acquire)});
    }
};

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const gpa = init.gpa;
    installSignalHandlers();

    var server = try Server.init(io, gpa);
    defer server.deinit();

    var task = io.async(Server.run, .{&server});
    io.sleep(.fromMilliseconds(50), .awake) catch {};

    var client: std.http.Client = .{ .allocator = gpa, .io = io };
    defer client.deinit();

    for (0..2) |i| {
        var body: std.Io.Writer.Allocating = .init(gpa);
        defer body.deinit();
        const url = std.fmt.comptimePrint("http://127.0.0.1:{d}/health", .{port});
        const res = try client.fetch(.{ .location = .{ .url = url }, .response_writer = &body.writer });
        std.debug.print("[client] request {d} -> {d} {s}\n", .{ i, @intFromEnum(res.status), body.written() });
    }

    std.debug.print("[main] sending SIGTERM to ourselves\n", .{});
    std.posix.raise(std.posix.SIG.TERM) catch {};
    task.cancel(io);
    std.debug.print("[main] accept task joined\n", .{});
}
```

```
[client] request 0 -> 200 {"status":"ok"}
[client] request 1 -> 200 {"status":"ok"}
[main] sending SIGTERM to ourselves
[server] accept returned Canceled; draining
[server] accept loop exited, stop_flag=true
[main] accept task joined
[server] listener closed, arena released after 2 requests
```

Each handler sleeps 30 ms, so both requests were genuinely in flight. The order
of the last four lines is the whole lesson:

1. `accept` was parked; `cancel` made it return `error.Canceled`
2. the loop exited with `stop_flag=true` — not mid-request
3. `cancel` **returned only after the task finished** ("accept task joined")
4. *then* `deinit` closed the listener and freed the arena

## `task.cancel(io)` vs `task.await(io)`

| | `await(io)` | `cancel(io)` |
|---|---|---|
| waits for the task | yes | **yes** |
| requests cancelation | no | yes — the next cancelation point returns `error.Canceled` |
| use for | a task that ends on its own (`max_requests` reached) | a task parked in `accept` |

`cancel` is documented as "equivalent to `await` but places a cancelation
request", which is exactly graceful shutdown: *stop taking new work, then let
the current work finish.* Both are idempotent, so `defer task.await(io)` plus an
explicit `task.cancel(io)` on the shutdown path is safe.

A cancelation point is any `io` call that can return `error.Canceled` — `accept`,
`io.sleep`, `Mutex.lock`. Only the **next** one signals; after that the task
runs normally, which is why `break`ing out of the loop on `error.Canceled` is
required rather than optional.

## `deinit` ordering

```zig
pub fn deinit(s: *Server) void {
    s.listener.deinit(s.io);   // 1. stop accepting
    s.arena_s.deinit();        // 2. request memory
    s.store.deinit();          // 3. long-lived state
}
```

Reverse order of acquisition, and each line explains the one above it:

| Order | Reason it is not later |
|---|---|
| listener first | a connection accepted after the store is freed would use freed memory |
| arena second | nothing in it outlives the request; freeing it cannot affect the store |
| store last | it is the thing requests read, so it must outlive every request |

The arena **holds gpa pages**, so `arena_s.deinit()` must happen before the gpa
itself goes away. With `init.gpa` the runtime handles that; with your own
`DebugAllocator`, `defer _ = gpa_s.deinit()` is declared *first* in `main` so it
runs *last*.

## Signal handling notes

```zig
fn onSignal(_: std.posix.SIG) callconv(.c) void {
    stop_flag.store(true, .release);
}
```

| Point | Detail |
|---|---|
| handler parameter | on macOS it's **`std.posix.SIG` (an enum)**, not `i32` — `fn (i32)` is a compile error here |
| handler body | an atomic store and nothing else. No allocation, no logging, no locks — a handler runs on a borrowed stack between two arbitrary instructions |
| `sigemptyset()` | a function call in 0.17, not a constant |
| `SIGPIPE` | `std.Io.Threaded` already ignores it, so a client disconnecting mid-write is a `WriteFailed` error, not a dead process |
| waking the loop | a handler **cannot** call `task.cancel(io)`. Set the flag; the main task must notice and cancel |

The last row is the structural consequence: a real deployment runs `main` as the
supervisor — it blocks on something cancelable (a sleep loop that polls the
flag, or `io.sleep` cancelled by a self-pipe write), and on wake calls
`task.cancel(io)`. Keeping the handler to one atomic store is what makes the
rest of the shutdown path ordinary, debuggable code.

A `SIGKILL` cannot be handled. That is fine: in-memory state is lost either way,
and 14.1 said persistence is out of scope.

---

Next: [14.7 — Tests & Release](./07-tests-release.md)
