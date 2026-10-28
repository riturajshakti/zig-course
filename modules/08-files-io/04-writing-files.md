# 8.4 — Writing Files

Everything here writes to `/tmp/`, so nothing lands in your project.

## One call: `writeFile`

```zig
const std = @import("std");

fn show(io: std.Io, gpa: std.mem.Allocator, label: []const u8, path: []const u8) !void {
    const text = try std.Io.Dir.cwd().readFileAlloc(io, path, gpa, .limited(4096));
    defer gpa.free(text);
    std.debug.print("{s} ({d} bytes): {s}\n", .{ label, text.len, text });
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const cwd = std.Io.Dir.cwd();
    const path = "/tmp/zig-course-write.txt";

    try cwd.writeFile(io, .{ .sub_path = path, .data = "first write\n" });
    try show(io, init.gpa, "after writeFile ", path);

    try cwd.writeFile(io, .{ .sub_path = path, .data = "second\n" });
    try show(io, init.gpa, "written again   ", path);

    const excl = cwd.writeFile(io, .{
        .sub_path = path,
        .data = "nope\n",
        .flags = .{ .exclusive = true },
    });
    std.debug.print("exclusive = true -> {any}\n", .{excl});

    cwd.deleteFile(io, "/tmp/zig-course-write-new.txt") catch {};
    try cwd.writeFile(io, .{
        .sub_path = "/tmp/zig-course-write-new.txt",
        .data = "brand new\n",
        .flags = .{ .exclusive = true },
    });
    try show(io, init.gpa, "exclusive on new", "/tmp/zig-course-write-new.txt");
}
```

```
after writeFile  (12 bytes): first write

written again    (7 bytes): second

exclusive = true -> error.PathAlreadyExists
exclusive on new (10 bytes): brand new
```

One options struct, no file handle, no flush. The second write **truncated** —
12 bytes became 7, not 12 with 7 on top.

| `.flags` field | Default | Effect |
|---|---|---|
| `truncate` | `true` | existing contents are discarded |
| `exclusive` | `false` | `true` → `error.PathAlreadyExists` if it exists |
| `read` | `false` | also open for reading |
| `permissions` | `.default_file` | mode bits |

`writeFile` is the right call whenever the data is already one slice in memory.

## Many writes: `createFile` + writer + flush

```zig
const std = @import("std");

const Row = struct { name: []const u8, hits: u32, rate: f64 };

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const path = "/tmp/zig-course-report.csv";

    const rows = [_]Row{
        .{ .name = "index", .hits = 10234, .rate = 0.9812 },
        .{ .name = "search", .hits = 871, .rate = 0.4409 },
    };

    var file = try std.Io.Dir.cwd().createFile(io, path, .{});
    defer file.close(io);

    var buf: [1024]u8 = undefined;
    var fw = file.writer(io, &buf);
    const out = &fw.interface;

    try out.writeAll("route,hits,rate\n");
    for (rows) |r| try out.print("{s},{d},{d:.2}\n", .{ r.name, r.hits, r.rate });
    try out.flush();

    const text = try std.Io.Dir.cwd().readFileAlloc(io, path, init.gpa, .limited(4096));
    defer init.gpa.free(text);
    std.debug.print("{s}", .{text});
}
```

```
route,hits,rate
index,10234,0.98
search,871,0.44
```

Four lines you always write, in this order:

```
var file = try std.Io.Dir.cwd().createFile(io, path, .{});
defer file.close(io);
var fw = file.writer(io, &buf);
try fw.interface.flush();
```

That last line can equivalently be `defer fw.interface.flush() catch {};`.

## Forget the flush and the file is empty — **broken on purpose**

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const path = "/tmp/zig-course-noflush.txt";

    {
        var file = try std.Io.Dir.cwd().createFile(io, path, .{});
        defer file.close(io);
        var buf: [1024]u8 = undefined;
        var fw = file.writer(io, &buf);
        try fw.interface.writeAll("these bytes never leave the buffer\n");
    }

    const text = try std.Io.Dir.cwd().readFileAlloc(io, path, init.gpa, .limited(4096));
    defer init.gpa.free(text);
    std.debug.print("file is {d} bytes\n", .{text.len});
}
```

```
file is 0 bytes
```

Exit code 0. The file exists, is 0 bytes, and nothing warned you. `close()` does
**not** flush — it closes the descriptor and the buffered bytes are simply
discarded. This is the single most common 0.17 bug.

The 35 bytes were in `buf`, on the stack, and went out of scope. One line fixes
it:

```
defer fw.interface.flush() catch {};
```

Put it immediately after creating the writer, before any `writeAll`.

> **Ordering with `defer`.** Defers run LIFO, so a flush deferred *after*
> `file.close(io)` runs *before* the close — which is what you want. Declare the
> close first, then the writer, then the flush.

## Append

`createFile` with `truncate = false` keeps the contents; then point the writer
at the end.

```zig
const std = @import("std");

fn append(io: std.Io, path: []const u8, line: []const u8) !void {
    var file = try std.Io.Dir.cwd().createFile(io, path, .{ .truncate = false });
    defer file.close(io);
    var buf: [256]u8 = undefined;
    var fw = file.writer(io, &buf);
    fw.pos = try file.length(io);
    try fw.interface.writeAll(line);
    try fw.interface.flush();
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const path = "/tmp/zig-course-append.log";
    std.Io.Dir.cwd().deleteFile(io, path) catch {};

    try append(io, path, "boot\n");
    try append(io, path, "ready\n");
    try append(io, path, "shutdown\n");

    const text = try std.Io.Dir.cwd().readFileAlloc(io, path, init.gpa, .limited(4096));
    defer init.gpa.free(text);
    std.debug.print("{s}", .{text});
}
```

```
boot
ready
shutdown
```

`fw.pos = try file.length(io)` is the whole trick: a positional writer's offset
is a plain field you can set. Drop that line and each call overwrites from byte
0, leaving only `shutdown\n`.

A streaming writer does it with a method instead:

```
var fw = file.writerStreaming(io, &buf);
try fw.seekTo(try file.length(io));
```

| Goal | Flags | Then |
|---|---|---|
| overwrite | `.{}` | write from offset 0 |
| append | `.{ .truncate = false }` | `fw.pos = try file.length(io)` |
| create or fail | `.{ .exclusive = true }` | — |
| truncate an existing file to 0 | — | `try file.setLength(io, 0)` |

## Atomic replace

A half-written config file is worse than an old one. `createFileAtomic` writes
to a temporary file and swaps it in with a single rename, so a reader sees
either the old bytes or the new bytes, never a mix.

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const cwd = std.Io.Dir.cwd();
    const path = "/tmp/zig-course-config.json";

    try cwd.writeFile(io, .{ .sub_path = path, .data = "{\"version\":1}\n" });

    var atomic = try cwd.createFileAtomic(io, path, .{ .replace = true });
    defer atomic.deinit(io);

    var buf: [512]u8 = undefined;
    var fw = atomic.file.writer(io, &buf);
    try fw.interface.print("{{\"version\":{d}}}\n", .{2});
    try fw.interface.flush();

    const before = try cwd.readFileAlloc(io, path, init.gpa, .limited(1024));
    defer init.gpa.free(before);
    std.debug.print("before replace: {s}", .{before});

    try atomic.replace(io);

    const after = try cwd.readFileAlloc(io, path, init.gpa, .limited(1024));
    defer init.gpa.free(after);
    std.debug.print("after replace:  {s}", .{after});
}
```

```
before replace: {"version":1}
after replace:  {"version":2}
```

The read *between* the write and the replace still sees version 1 — that is the
guarantee, demonstrated.

| Step | Why |
|---|---|
| `createFileAtomic(io, path, .{ .replace = true })` | `.replace` must match how you finish, or `replace()` asserts |
| `atomic.file.writer(io, &buf)` | write through the `.file` field |
| `try fw.interface.flush()` | **before** `replace()` — same trap, worse consequences |
| `try atomic.replace(io)` | the rename; the file is now live |
| `defer atomic.deinit(io)` | cleans up the temp file if you returned early |

`deinit` after a successful `replace` is harmless, so the `defer` is
unconditional — an error path between create and replace leaves no debris.

Use `atomic.link(io)` instead of `replace(io)` when the destination must **not**
already exist; pass `.replace = false` in that case.

## Formatted output and `{f}`

A type with a `format` method writes itself into any writer:

```zig
const std = @import("std");

const Point = struct {
    x: i32,
    y: i32,
    pub fn format(p: Point, w: *std.Io.Writer) !void {
        try w.print("({d},{d})", .{ p.x, p.y });
    }
};

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const path = "/tmp/zig-course-points.txt";
    const pts = [_]Point{ .{ .x = 0, .y = 0 }, .{ .x = 3, .y = -4 } };

    var file = try std.Io.Dir.cwd().createFile(io, path, .{});
    defer file.close(io);
    var buf: [256]u8 = undefined;
    var fw = file.writer(io, &buf);
    defer fw.interface.flush() catch {};

    for (pts, 0..) |p, i| try fw.interface.print("{d}: {f}\n", .{ i, p });
    try fw.interface.flush();

    const text = try std.Io.Dir.cwd().readFileAlloc(io, path, init.gpa, .limited(1024));
    defer init.gpa.free(text);
    std.debug.print("{s}", .{text});
}
```

```
0: (0,0)
1: (3,-4)
```

Signature is `pub fn format(self: T, w: *std.Io.Writer) !void` — two parameters.
Older tutorials show four and will not compile.

## Build in memory, write once

When the content needs to be assembled before you know its size, format into an
`Allocating` writer and hand the result to `writeFile`:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const path = "/tmp/zig-course-assembled.md";

    var doc: std.Io.Writer.Allocating = .init(init.gpa);
    defer doc.deinit();
    const w = &doc.writer;

    try w.writeAll("# Report\n\n");
    for ([_][]const u8{ "parse", "check", "emit" }, 1..) |stage, i| {
        try w.print("{d}. {s}\n", .{ i, stage });
    }
    try w.print("\n{d} stages, {d} bytes so far\n", .{ 3, doc.written().len });

    try std.Io.Dir.cwd().writeFile(io, .{ .sub_path = path, .data = doc.written() });

    const text = try std.Io.Dir.cwd().readFileAlloc(io, path, init.gpa, .limited(4096));
    defer init.gpa.free(text);
    std.debug.print("{s}", .{text});
}
```

```
# Report

1. parse
2. check
3. emit

3 stages, 36 bytes so far
```

No flush anywhere — `Allocating` has nothing to drain to, and `writeFile` does
its own. The tradeoff: the whole document is in memory. For a large file, write
straight to the file's writer and let the buffer do its job.

## Which call

| Situation | Use |
|---|---|
| data is one slice already | `writeFile(io, .{ .sub_path, .data })` |
| many writes, formatted | `createFile` + `writer` + **flush** |
| must not clobber a good file | `createFileAtomic` + `replace` |
| appending to a log | `.{ .truncate = false }` + `fw.pos = length` |
| must not already exist | `.{ .exclusive = true }` |
| size unknown until assembled | `Writer.Allocating` then `writeFile` |

---

Next: [8.5 — Directories](./05-directories.md)
