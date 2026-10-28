# 15.5 — Zero-Copy Parsing

A slice is a pointer plus a length. **Parsing can be just arithmetic on
pointers you already have** — no `dupe`, no `allocPrint`, no allocator
parameter at all. This is where Zig's memory model stops being a tax and starts
being an advantage.

## Split and tokenize: the primitives

```zig
const std = @import("std");

fn show(comptime label: []const u8, it_init: anytype) void {
    var it = it_init;
    std.debug.print("{s:<28}", .{label});
    var n: usize = 0;
    while (it.next()) |p| : (n += 1) std.debug.print(" [{s}]", .{p});
    std.debug.print("   ({d} parts)\n", .{n});
}

pub fn main() !void {
    const s = "a,,b,c";
    std.debug.print("input = \"{s}\"\n", .{s});
    show("splitScalar ','", std.mem.splitScalar(u8, s, ','));
    show("tokenizeScalar ','", std.mem.tokenizeScalar(u8, s, ','));

    const t = "  GET   /path  ";
    std.debug.print("\ninput = \"{s}\"\n", .{t});
    show("splitAny \" \"", std.mem.splitAny(u8, t, " "));
    show("tokenizeAny \" \\t\"", std.mem.tokenizeAny(u8, t, " \t"));

    const u = "k1=v1&&k2=v2";
    std.debug.print("\ninput = \"{s}\"\n", .{u});
    show("splitSequence \"&&\"", std.mem.splitSequence(u8, u, "&&"));

    const v = "2024-01-15T10:30:00Z";
    std.debug.print("\ninput = \"{s}\"\n", .{v});
    show("splitAny \"-T:Z\"", std.mem.splitAny(u8, v, "-T:Z"));
    show("tokenizeAny \"-T:Z\"", std.mem.tokenizeAny(u8, v, "-T:Z"));

    std.debug.print("\nsliceTo(\"abc\\x00def\", 0) = \"{s}\"\n", .{std.mem.sliceTo("abc\x00def", 0)});
    std.debug.print("trim(\"  x  \")             = \"{s}\"\n", .{std.mem.trim(u8, "  x  ", " ")});
    std.debug.print("trimStart / trimEnd       = \"{s}\" / \"{s}\"\n", .{
        std.mem.trimStart(u8, "..x..", "."), std.mem.trimEnd(u8, "..x..", "."),
    });
    const line = "name: value";
    const c = std.mem.indexOfScalar(u8, line, ':').?;
    std.debug.print("split at first ':'        = \"{s}\" / \"{s}\"\n", .{ line[0..c], std.mem.trimStart(u8, line[c + 1 ..], " ") });
}
```

```
input = "a,,b,c"
splitScalar ','              [a] [] [b] [c]   (4 parts)
tokenizeScalar ','           [a] [b] [c]   (3 parts)

input = "  GET   /path  "
splitAny " "                 [] [] [GET] [] [] [/path] [] []   (8 parts)
tokenizeAny " \t"            [GET] [/path]   (2 parts)

input = "k1=v1&&k2=v2"
splitSequence "&&"           [k1=v1] [k2=v2]   (2 parts)

input = "2024-01-15T10:30:00Z"
splitAny "-T:Z"              [2024] [01] [15] [10] [30] [00] []   (7 parts)
tokenizeAny "-T:Z"           [2024] [01] [15] [10] [30] [00]   (6 parts)

sliceTo("abc\x00def", 0) = "abc"
trim("  x  ")             = "x"
trimStart / trimEnd       = "x.." / "..x"
split at first ':'        = "name" / "value"
```

**`split*` keeps empty fields, `tokenize*` collapses them.** That one line
decides which you want:

| Function | On `"a,,b,c"` | Use for |
|---|---|---|
| `splitScalar(u8, s, ',')` | `a`, ``, `b`, `c` | **CSV** — an empty field is data |
| `tokenizeScalar(u8, s, ',')` | `a`, `b`, `c` | lists where blanks are noise |
| `splitAny(u8, s, " \t")` | splits on *any* byte in the set | rarely what you want |
| `tokenizeAny(u8, s, " \t")` | collapses runs of **any** delimiter | **whitespace-separated** fields |
| `splitSequence(u8, s, "\r\n")` | splits on the **whole** string | **HTTP lines**, multi-byte delimiters |
| `std.mem.indexOfScalar` + slicing | one split, at the first occurrence | `name: value` — the value may contain `:` |

`splitAny " "` on `"  GET   /path  "` giving **8 parts** is the trap. Use
`tokenizeAny` for human-written whitespace, and never reach for `splitAny` on a
multi-byte delimiter — `splitAny(u8, s, "\r\n")` splits on `\r` *or* `\n`
separately, so every CRLF line gives you a phantom empty field.

Every one of these returns slices **into the input**. No allocator, no copies.

## Parsing a log line

```zig
const std = @import("std");

const LogLine = struct {
    ip: []const u8,
    method: []const u8,
    path: []const u8,
    status: u16,
    bytes: u64,
};

const Error = error{ Malformed, BadStatus, BadBytes };

fn parseLine(line: []const u8) Error!LogLine {
    var it = std.mem.tokenizeAny(u8, line, " ");
    const ip = it.next() orelse return error.Malformed;
    const method = it.next() orelse return error.Malformed;
    const path = it.next() orelse return error.Malformed;
    const status_s = it.next() orelse return error.Malformed;
    const bytes_s = it.next() orelse return error.Malformed;

    return .{
        .ip = ip,
        .method = method,
        .path = path,
        .status = std.fmt.parseInt(u16, status_s, 10) catch return error.BadStatus,
        .bytes = std.fmt.parseInt(u64, bytes_s, 10) catch return error.BadBytes,
    };
}

pub fn main() !void {
    const log =
        \\10.0.0.1 GET /api/users 200 1543
        \\10.0.0.2 POST /api/login 401 89
        \\10.0.0.1 GET /health 200 2
        \\garbage
        \\10.0.0.3 DELETE /api/users/7 500 xyz
    ;

    var lines = std.mem.splitScalar(u8, log, '\n');
    while (lines.next()) |line| {
        if (parseLine(line)) |r| {
            std.debug.print("{s:<10} {s:<7} {s:<16} {d} {d:>5} B", .{ r.ip, r.method, r.path, r.status, r.bytes });
            const inside = @intFromPtr(r.path.ptr) >= @intFromPtr(log.ptr) and
                @intFromPtr(r.path.ptr) < @intFromPtr(log.ptr) + log.len;
            std.debug.print("   path points into source: {}\n", .{inside});
        } else |e| {
            std.debug.print("{s:<10} -> {t}\n", .{ if (line.len > 9) line[0..9] else line, e });
        }
    }
}
```

```
10.0.0.1   GET     /api/users       200  1543 B   path points into source: true
10.0.0.2   POST    /api/login       401    89 B   path points into source: true
10.0.0.1   GET     /health          200     2 B   path points into source: true
garbage    -> Malformed
10.0.0.3   -> BadBytes
```

**`parseLine` has no allocator parameter**, which is the whole claim: by
signature alone, it cannot allocate and cannot fail with `OutOfMemory`. Its
error set is three cases you can actually handle.

Compare the Node.js habit — `line.split(' ')` allocates an array plus a string
per field, five objects for the GC per line. Here it's five `{ptr, len}` pairs
in registers.

## An HTTP-ish request parser, zero allocations

```zig
const std = @import("std");

const Request = struct {
    method: []const u8,
    target: []const u8,
    version: []const u8,
    headers: [16]Header,
    header_count: usize,

    const Header = struct { name: []const u8, value: []const u8 };

    fn get(r: *const Request, name: []const u8) ?[]const u8 {
        for (r.headers[0..r.header_count]) |h| {
            if (std.ascii.eqlIgnoreCase(h.name, name)) return h.value;
        }
        return null;
    }
};

const Error = error{ NoRequestLine, BadRequestLine, BadHeader, TooManyHeaders };

fn parseRequest(raw: []const u8) Error!Request {
    var lines = std.mem.splitSequence(u8, raw, "\r\n");
    const first = lines.next() orelse return error.NoRequestLine;

    var parts = std.mem.splitScalar(u8, first, ' ');
    const method = parts.next() orelse return error.BadRequestLine;
    const target = parts.next() orelse return error.BadRequestLine;
    const version = parts.next() orelse return error.BadRequestLine;

    var req: Request = .{
        .method = method,
        .target = target,
        .version = version,
        .headers = undefined,
        .header_count = 0,
    };

    while (lines.next()) |line| {
        if (line.len == 0) break;
        const colon = std.mem.indexOfScalar(u8, line, ':') orelse return error.BadHeader;
        if (req.header_count == req.headers.len) return error.TooManyHeaders;
        req.headers[req.header_count] = .{
            .name = line[0..colon],
            .value = std.mem.trim(u8, line[colon + 1 ..], " \t"),
        };
        req.header_count += 1;
    }
    return req;
}

pub fn main() !void {
    const raw = "POST /api/items?page=2 HTTP/1.1\r\n" ++
        "Host: api.example.com\r\n" ++
        "Content-Type: application/json\r\n" ++
        "Content-Length: 27\r\n" ++
        "\r\n" ++
        "{\"name\":\"widget\",\"qty\":3}";

    const req = try parseRequest(raw);
    std.debug.print("method  = {s}\n", .{req.method});
    std.debug.print("target  = {s}\n", .{req.target});
    std.debug.print("version = {s}\n", .{req.version});
    std.debug.print("headers = {d}\n", .{req.header_count});
    for (req.headers[0..req.header_count]) |h| {
        std.debug.print("  {s} = \"{s}\"\n", .{ h.name, h.value });
    }
    std.debug.print("\nget(\"content-type\") = {?s}\n", .{req.get("content-type")});
    std.debug.print("get(\"Missing\")      = {?s}\n", .{req.get("Missing")});

    var q = std.mem.splitScalar(u8, req.target, '?');
    std.debug.print("\npath  = {s}\n", .{q.next().?});
    std.debug.print("query = {?s}\n", .{q.next()});

    var inside: usize = 0;
    for (req.headers[0..req.header_count]) |h| {
        if (@intFromPtr(h.value.ptr) >= @intFromPtr(raw.ptr) and
            @intFromPtr(h.value.ptr) < @intFromPtr(raw.ptr) + raw.len) inside += 1;
    }
    std.debug.print("\nheader values pointing into `raw`: {d}/{d}  (0 allocations)\n", .{ inside, req.header_count });
    std.debug.print("@sizeOf(Request) = {d} B, all stack\n", .{@sizeOf(Request)});
}
```

```
method  = POST
target  = /api/items?page=2
version = HTTP/1.1
headers = 3
  Host = "api.example.com"
  Content-Type = "application/json"
  Content-Length = "27"

get("content-type") = application/json
get("Missing")      = null

path  = /api/items
query = page=2

header values pointing into `raw`: 3/3  (0 allocations)
@sizeOf(Request) = 568 B, all stack
```

The techniques worth stealing:

| Technique | Why |
|---|---|
| **fixed `[16]Header` + a count**, not an `ArrayList` | no allocator; a request with 200 headers gets `error.TooManyHeaders` instead of eating memory |
| `splitSequence(…, "\r\n")` | the delimiter is two bytes; `splitAny` would give phantom empties |
| `indexOfScalar(':')` then slice | a header value may contain `:` (`Host: example.com:8443`) — splitting on `:` would truncate it |
| `std.mem.trim(u8, v, " \t")` | strips optional whitespace without copying — just moves `ptr` and shrinks `len` |
| `std.ascii.eqlIgnoreCase` for lookup | HTTP header names are case-insensitive, and this needs no lowercased copy |
| `get()` returning `?[]const u8` | a missing header is not an error |

**`TooManyHeaders` is a feature, not a limitation.** A fixed cap is a
denial-of-service defence you get for free; an `ArrayList` would happily
allocate for as many headers as an attacker sends.

## Measured: zero-copy vs allocating

```zig
const std = @import("std");

const Fields = struct { ip: []const u8, method: []const u8, path: []const u8, status: u16 };

fn splitZeroCopy(line: []const u8) ?Fields {
    var it = std.mem.tokenizeAny(u8, line, " ");
    return .{
        .ip = it.next() orelse return null,
        .method = it.next() orelse return null,
        .path = it.next() orelse return null,
        .status = std.fmt.parseInt(u16, it.next() orelse return null, 10) catch return null,
    };
}

fn splitAllocating(gpa: std.mem.Allocator, line: []const u8) !?Fields {
    var it = std.mem.tokenizeAny(u8, line, " ");
    const ip = try gpa.dupe(u8, it.next() orelse return null);
    errdefer gpa.free(ip);
    const method = try gpa.dupe(u8, it.next() orelse return null);
    errdefer gpa.free(method);
    const path = try gpa.dupe(u8, it.next() orelse return null);
    errdefer gpa.free(path);
    const status = std.fmt.parseInt(u16, it.next() orelse return null, 10) catch return null;
    return .{ .ip = ip, .method = method, .path = path, .status = status };
}

fn freeFields(gpa: std.mem.Allocator, f: Fields) void {
    gpa.free(f.ip);
    gpa.free(f.method);
    gpa.free(f.path);
}

const REPS = 5;

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const gpa = init.gpa;

    var log: std.ArrayList(u8) = .empty;
    defer log.deinit(gpa);
    const N = 200_000;
    for (0..N) |i| {
        try log.print(gpa, "10.0.{d}.{d} GET /api/resource/{d} 200\n", .{ (i / 256) % 256, i % 256, i });
    }
    std.debug.print("input: {d} lines, {d} bytes\n\n", .{ N, log.items.len });

    var best_zc: u64 = std.math.maxInt(u64);
    var best_al: u64 = std.math.maxInt(u64);
    var best_ar: u64 = std.math.maxInt(u64);
    var acc: u64 = 0;

    for (0..REPS) |_| {
        var t0 = std.Io.Clock.Timestamp.now(io, .awake);
        var lines = std.mem.splitScalar(u8, log.items, '\n');
        while (lines.next()) |line| {
            if (line.len == 0) continue;
            if (splitZeroCopy(line)) |f| acc += f.status + f.path.len;
        }
        var t1 = std.Io.Clock.Timestamp.now(io, .awake);
        std.mem.doNotOptimizeAway(acc);
        best_zc = @min(best_zc, @as(u64, @intCast(t0.durationTo(t1).raw.toMicroseconds())));

        t0 = std.Io.Clock.Timestamp.now(io, .awake);
        lines = std.mem.splitScalar(u8, log.items, '\n');
        while (lines.next()) |line| {
            if (line.len == 0) continue;
            if (try splitAllocating(gpa, line)) |f| {
                acc += f.status + f.path.len;
                freeFields(gpa, f);
            }
        }
        t1 = std.Io.Clock.Timestamp.now(io, .awake);
        std.mem.doNotOptimizeAway(acc);
        best_al = @min(best_al, @as(u64, @intCast(t0.durationTo(t1).raw.toMicroseconds())));

        t0 = std.Io.Clock.Timestamp.now(io, .awake);
        var arena: std.heap.ArenaAllocator = .init(gpa);
        const aa = arena.allocator();
        lines = std.mem.splitScalar(u8, log.items, '\n');
        while (lines.next()) |line| {
            if (line.len == 0) continue;
            if (try splitAllocating(aa, line)) |f| acc += f.status + f.path.len;
        }
        arena.deinit();
        t1 = std.Io.Clock.Timestamp.now(io, .awake);
        std.mem.doNotOptimizeAway(acc);
        best_ar = @min(best_ar, @as(u64, @intCast(t0.durationTo(t1).raw.toMicroseconds())));
    }

    std.debug.print("zero-copy (slices into buffer) : {d:>6} us   0 allocations\n", .{best_zc});
    std.debug.print("dupe + free per field (gpa)    : {d:>6} us   {d} allocations\n", .{ best_al, N * 3 });
    std.debug.print("dupe into an arena, free once  : {d:>6} us   {d} allocations\n", .{ best_ar, N * 3 });
    std.debug.print("\nchecksum {d}\n", .{acc});
}
```

**`zig run -Ofast`** (ReleaseFast), 200k lines / 8.1 MB, best of 5 per phase,
four consecutive runs:

```
input: 200000 lines, 8112474 bytes

zero-copy (slices into buffer) :   3833 us   0 allocations
dupe + free per field (gpa)    :  16483 us   600000 allocations
dupe into an arena, free once  :   7081 us   600000 allocations
```

| Strategy | Times across 4 runs (us) | vs zero-copy | Allocations |
|---|---|---|---|
| **zero-copy** | 3833 / 3716 / 4064 / 3964 | — | **0** |
| **arena**, freed once | 7081 / 7033 / 6926 / 6870 | ~1.8x slower | 600k |
| `dupe` + `free` each | 16483 / 16996 / 16270 / 15608 | **~4.2x slower** | 600k |

Measured on one machine (arm64 macOS, ReleaseFast); expect different absolute
numbers elsewhere. What generalizes:

- **Zero-copy beats everything** because the only work is pointer arithmetic.
- **An arena recovers most of the loss.** 1.8x, not 4.2x — this is why the
  per-request arena is the idiomatic server pattern. If you *must* own the
  strings, own them in an arena.
- **Per-field `dupe`/`free` is the expensive one.** 600k round trips through the
  allocator's free lists, and ~8 MB of pointless copying.

**Don't over-read this.** 4.2x on a parse loop is not 4.2x on your service: a
real request also does I/O, JSON, a database round trip. Parsing is rarely the
bottleneck — but it is a bottleneck you can delete entirely, for free, by
changing a signature.

## The lifetime rule

Zero-copy means **the parse result borrows the buffer**. Reuse the buffer and
your parsed data silently changes:

```zig
const std = @import("std");

fn firstToken(line: []const u8) []const u8 {
    var it = std.mem.tokenizeAny(u8, line, " ");
    return it.next() orelse "";
}

pub fn main() !void {
    var buf: [32]u8 = undefined;

    const n1 = (try std.fmt.bufPrint(&buf, "alpha first-record", .{})).len;
    const borrowed = firstToken(buf[0..n1]);
    std.debug.print("borrowed while valid : \"{s}\"\n", .{borrowed});

    const n2 = (try std.fmt.bufPrint(&buf, "ZZZZZ second-record", .{})).len;
    _ = n2;
    std.debug.print("after buffer reuse   : \"{s}\"   <- same pointer, new bytes\n", .{borrowed});

    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer std.debug.print("deinit status: {t}\n", .{dbg.deinit()});
    const gpa = dbg.allocator();

    const n3 = (try std.fmt.bufPrint(&buf, "gamma third-record", .{})).len;
    const owned = try gpa.dupe(u8, firstToken(buf[0..n3]));
    defer gpa.free(owned);
    _ = try std.fmt.bufPrint(&buf, "XXXXX fourth-record", .{});
    std.debug.print("duped, survives      : \"{s}\"\n", .{owned});
}
```

```
borrowed while valid : "alpha"
after buffer reuse   : "ZZZZZ"   <- same pointer, new bytes
duped, survives      : "gamma"
deinit status: ok
```

**`"alpha"` became `"ZZZZZ"` with no crash, no leak, and no warning.** This is
the cost of zero-copy, and the reason Rust has lifetimes. In Zig you carry the
rule yourself:

| If the parse result… | Then |
|---|---|
| is consumed before the buffer is reused | **borrow** — this is the common case in a request handler |
| outlives the buffer (stored in a cache, returned up) | **`dupe` it**, into an arena if there are many |
| goes into a `HashMap` key | **`dupe` it** — the map does not copy keys |
| is read on another thread | `dupe` it, or prove the buffer outlives the thread |

Three habits that make this safe in practice:

1. **Read the whole message into one buffer, parse it, finish with it, reuse the
   buffer.** That's the loop shape of every high-performance server. The
   lifetime is one iteration, which is short enough to hold in your head.
2. **Name the buffer in the type.** A function returning views into its
   argument should say so: `/// The returned slices alias \`raw\`.`
3. **One arena per request.** Anything that needs to outlive the read buffer
   but not the request goes there, and `arena.deinit()` at the end of the
   handler makes the lifetime structural instead of a thing you remember.

---

Next: [15.6 — Syscalls & the OS](./06-syscalls.md)
