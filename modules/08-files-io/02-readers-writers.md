# 8.2 — Readers & Writers

One generic `std.Io.Writer` and one generic `std.Io.Reader`. Everything else —
files, stdout, memory, sockets, hashes — is a wrapper that hands you one.

## The `.interface` field

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    var fbuf: [256]u8 = undefined;
    var sbuf: [256]u8 = undefined;
    var mbuf: [256]u8 = undefined;

    var file = try std.Io.Dir.cwd().createFile(init.io, "/tmp/zig-course-shapes.txt", .{});
    defer file.close(init.io);

    var fw = file.writer(init.io, &fbuf);
    var sw = std.Io.File.stdout().writerStreaming(init.io, &sbuf);
    var mw = std.Io.Writer.fixed(&mbuf);

    try fw.interface.writeAll("to the file\n");
    try fw.interface.flush();
    try mw.writeAll("to memory");

    const out = &sw.interface;
    try out.print("file writer    {s}\n", .{@typeName(@TypeOf(fw))});
    try out.print("stdout writer  {s}\n", .{@typeName(@TypeOf(sw))});
    try out.print("fixed writer   {s}\n", .{@typeName(@TypeOf(mw))});
    try out.print("&fw.interface  {s}\n", .{@typeName(@TypeOf(&fw.interface))});
    try out.print("&mw            {s}\n", .{@typeName(@TypeOf(&mw))});
    try out.print("mw.buffered()  {s}\n", .{mw.buffered()});
    try out.flush();
}
```

```
file writer    Io.File.Writer
stdout writer  Io.File.Writer
fixed writer   Io.Writer
&fw.interface  *Io.Writer
&mw            *Io.Writer
mw.buffered()  to memory
```

The asymmetry in one table — memorise it or you will fight the compiler:

| You have | Get the `*std.Io.Writer` with |
|---|---|
| `file.writer(io, &buf)` | `&w.interface` |
| `File.stdout().writer(io, &buf)` | `&w.interface` |
| `std.Io.Writer.fixed(&buf)` | `&w` — **it IS a Writer** |
| `Writer.Allocating.init(gpa)` | `&aw.writer` |
| `file.reader(io, &buf)` | `&r.interface` |

`Writer.fixed` has no `.interface`; reaching for one is a compile error.

## `*std.Io.Writer` is the portable parameter type

Write every helper against the interface and it works with all of them:

```zig
const std = @import("std");

const Row = struct { []const u8, u32 };

fn render(out: *std.Io.Writer, rows: []const Row) !void {
    for (rows) |row| try out.print("{s:<6}{d:>5}\n", .{ row[0], row[1] });
}

pub fn main(init: std.process.Init) !void {
    const rows = [_]Row{ .{ "cpu", 93 }, .{ "mem", 41 } };

    var mem: [128]u8 = undefined;
    var fixed = std.Io.Writer.fixed(&mem);
    try render(&fixed, &rows);

    var alloc: std.Io.Writer.Allocating = .init(init.gpa);
    defer alloc.deinit();
    try render(&alloc.writer, &rows);

    var sbuf: [256]u8 = undefined;
    var sw = std.Io.File.stdout().writerStreaming(init.io, &sbuf);
    const out = &sw.interface;
    defer out.flush() catch {};

    try out.print("fixed      {d} bytes\n{s}", .{ fixed.buffered().len, fixed.buffered() });
    try out.print("allocating {d} bytes\n{s}", .{ alloc.written().len, alloc.written() });
}
```

```
fixed      24 bytes
cpu      93
mem      41
allocating 24 bytes
cpu      93
mem      41
```

One `render`, three destinations, zero generics.

## Writing: the methods

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    var buf: [1024]u8 = undefined;
    var w = std.Io.File.stdout().writerStreaming(init.io, &buf);
    const out = &w.interface;
    defer out.flush() catch {};

    try out.writeAll("writeAll\n");
    try out.print("print {d} {s}\n", .{ 7, "args" });
    try out.writeByte('X');
    try out.writeByte('\n');
    try out.splatByteAll('-', 20);
    try out.writeByte('\n');
    try out.splatBytesAll("ab", 4);
    try out.writeByte('\n');
    try out.printHex("zig", .lower);
    try out.writeByte('\n');
    try out.printBase64("zig");
    try out.writeByte('\n');
    try out.printByteSize(1536 * 1024, .binary, .{});
    try out.writeByte('\n');
    const n = try out.write("write returns a count\n");
    try out.print("write -> {d}\n", .{n});
}
```

```
writeAll
print 7 args
X
--------------------
abababab
7a6967
emln
1.5MiB
write returns a count
write -> 22
```

| Method | Does |
|---|---|
| `writeAll(bytes)` | all of it, or an error |
| `write(bytes)` | writes **some**, returns the count — you must loop |
| `print(fmt, args)` | formatted, same specifiers as lesson 1.3 |
| `writeByte(b)` | one byte |
| `splatByteAll(b, n)` | `b` repeated `n` times |
| `splatBytesAll(s, n)` | `s` repeated `n` times |
| `printHex(bytes, .lower)` | hex dump of bytes |
| `printBase64(bytes)` | base64 |
| `printByteSize(n, .binary, .{})` | human sizes — `.binary` → `1.5MiB`, `.decimal` → `1.572864MB` |
| `flush()` | drain the buffer to the destination |

`printByteSize` takes **four** arguments — value, units, options — a two-arg
call is a compile error.

## Buffering, measured

The buffer is not a limit, it is a syscall budget. This writer counts its own
drains, so you can watch the cost change:

```zig
const std = @import("std");

const Counting = struct {
    writer: std.Io.Writer,
    drains: usize = 0,

    fn drain(w: *std.Io.Writer, data: []const []const u8, splat: usize) std.Io.Writer.Error!usize {
        const self: *Counting = @alignCast(@fieldParentPtr("writer", w));
        self.drains += 1;
        _ = splat;
        w.end = 0;
        var n: usize = 0;
        for (data) |d| n += d.len;
        return n;
    }

    fn init(buffer: []u8) Counting {
        return .{ .writer = .{ .buffer = buffer, .vtable = &.{ .drain = drain } } };
    }
};

fn emit(out: *std.Io.Writer) !void {
    for (0..100) |i| try out.print("line {d}\n", .{i});
    try out.flush();
}

pub fn main(init: std.process.Init) !void {
    var sbuf: [256]u8 = undefined;
    var sw = std.Io.File.stdout().writerStreaming(init.io, &sbuf);
    const out = &sw.interface;
    defer out.flush() catch {};

    for ([_]usize{ 16, 64, 512, 4096 }) |size| {
        var storage: [4096]u8 = undefined;
        var c = Counting.init(storage[0..size]);
        try emit(&c.writer);
        try out.print("buffer {d:>5} -> {d:>3} drains\n", .{ size, c.drains });
    }
}
```

```
buffer    16 ->  43 drains
buffer    64 ->  12 drains
buffer   512 ->   2 drains
buffer  4096 ->   1 drains
```

Same 790 bytes of output. A `drain` is one write syscall for a real file — so
that column is 43 syscalls versus 1.

Note also what a `Writer` is at the bottom: a `buffer` slice and a vtable whose
only required member is `drain`. Implementing one is thirty lines.

### Sizing

| Size | Use |
|---|---|
| none (`&.{}`) | unbuffered — every write is a syscall |
| 64 B | interactive prompts where latency beats throughput |
| **1–4 KiB** | **the default; stop thinking about it** |
| 64 KiB+ | bulk copies, large file writes |

A too-small buffer never truncates or fails — it only drains more often.

### `flush()`

Buffered means **nothing is written until you say so**. Zero bytes, exit code 0,
no warning. Put the flush next to the writer:

```
defer out.flush() catch {};
```

`catch {}` is required because `defer` cannot propagate an error. Two caveats:

- A deferred flush runs **LIFO**, so if you defer stdout's flush before
  stderr's, stderr drains first and the interleaving on a terminal is not
  source order.
- If the flush is the operation whose failure matters (a file you must know
  landed), call `try out.flush()` explicitly at the end *as well*.

`Writer.Allocating`'s flush is a documented no-op — it has nowhere to drain to.

## In-memory writers

### `Writer.fixed` — no allocator, no `io`

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    var tiny: [8]u8 = undefined;
    var fixed = std.Io.Writer.fixed(&tiny);
    fixed.writeAll("more than eight bytes") catch |e| {
        std.debug.print("fixed overflowed: {t}\n", .{e});
    };
    std.debug.print("fixed holds: {s}\n", .{fixed.buffered()});

    var alloc: std.Io.Writer.Allocating = .init(init.gpa);
    defer alloc.deinit();
    try alloc.writer.writeAll("grows as needed, no limit");
    const owned = try alloc.toOwnedSlice();
    defer init.gpa.free(owned);
    std.debug.print("owned ({d}): {s}\n", .{ owned.len, owned });
}
```

```
fixed overflowed: WriteFailed
fixed holds: more tha
owned (25): grows as needed, no limit
```

A full `fixed` writer fails with `error.WriteFailed` **after a partial write** —
the first 8 bytes are there. It is not transactional.

### `Writer.Allocating` — grows

| Call | Gives you |
|---|---|
| `.init(gpa)` | empty, grows on demand |
| `.initCapacity(gpa, n)` | pre-sized |
| `.fromArrayList(gpa, &list)` | takes over an existing `ArrayList(u8)` |
| `aw.written()` | a borrowed view of the bytes |
| `aw.toOwnedSlice()` | transfers ownership — **you** free it |
| `aw.toArrayList()` | hands back an `ArrayList(u8)` |
| `aw.deinit()` | required unless you took ownership |

`&aw.writer` is the `*std.Io.Writer` — the field is named `writer`, not
`interface`.

## Reading

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const cwd = std.Io.Dir.cwd();

    try cwd.writeFile(io, .{
        .sub_path = "/tmp/zig-course-bytes.txt",
        .data = "HEADER:0123456789rest of the stream",
    });

    var file = try cwd.openFile(io, "/tmp/zig-course-bytes.txt", .{});
    defer file.close(io);

    var buf: [256]u8 = undefined;
    var fr = file.reader(io, &buf);
    const r = &fr.interface;

    std.debug.print("length       {d}\n", .{try file.length(io)});
    std.debug.print("peek(6)      {s}\n", .{try r.peek(6)});
    std.debug.print("take(7)      {s}\n", .{try r.take(7)});
    std.debug.print("takeByte     {c}\n", .{try r.takeByte()});
    std.debug.print("takeArray(4) {s}\n", .{try r.takeArray(4)});
    try r.discardAll(5);
    std.debug.print("after discard, takeByte {c}\n", .{try r.takeByte()});
    const rest = try r.allocRemaining(init.gpa, .limited(1024));
    defer init.gpa.free(rest);
    std.debug.print("allocRemaining [{s}]\n", .{rest});
    std.debug.print("at EOF: {any}\n", .{r.takeByte()});
}
```

```
length       35
peek(6)      HEADER
take(7)      HEADER:
takeByte     0
takeArray(4) 1234
after discard, takeByte r
allocRemaining [est of the stream]
at EOF: error.EndOfStream
```

| Method | Does |
|---|---|
| `peek(n)` | look at `n` bytes, **cursor stays** |
| `take(n)` | `n` bytes, cursor advances |
| `takeByte()` | one byte |
| `takeArray(n)` | `*[n]u8`, `n` comptime-known |
| `takeDelimiterInclusive(d)` | up to and including `d` — see 8.3 |
| `discardAll(n)` | skip `n` bytes |
| `toss(n)` | drop `n` already-buffered bytes |
| `buffered()` | what is sitting in the buffer right now |
| `allocRemaining(gpa, limit)` | the rest, heap-allocated |
| `streamRemaining(w)` | the rest, straight into a writer |

Running out of input is always `error.EndOfStream`, never a short read you have
to detect.

### Reader buffer size is a hard limit for delimited reads

`take`, `peek` and the delimiter methods return a slice **into the reader's
buffer**, so a line longer than the buffer cannot be returned:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    var data: std.Io.Writer.Allocating = .init(init.gpa);
    defer data.deinit();
    try data.writer.writeAll("short\n");
    try data.writer.splatByteAll('y', 100);
    try data.writer.writeByte('\n');
    try std.Io.Dir.cwd().writeFile(init.io, .{
        .sub_path = "/tmp/zig-course-longline.txt",
        .data = data.written(),
    });

    var f = try std.Io.Dir.cwd().openFile(init.io, "/tmp/zig-course-longline.txt", .{});
    defer f.close(init.io);
    var rb: [16]u8 = undefined;
    var fr = f.reader(init.io, &rb);
    std.debug.print("line1: {s}", .{try fr.interface.takeDelimiterInclusive('\n')});
    std.debug.print("line2: {any}\n", .{fr.interface.takeDelimiterInclusive('\n')});
}
```

```
line1: short
line2: error.StreamTooLong
```

An explicit error, not a silent truncation. 4 KiB handles ordinary text; size
for your longest expected line.

## Positional vs streaming

`file.writer()` writes at an explicit **offset** it tracks itself, starting at
0. `file.writerStreaming()` appends at the OS file position.

```zig
const std = @import("std");

fn twice(io: std.Io, file: std.Io.File, streaming: bool) !void {
    for ([_][]const u8{ "first\n", "second\n" }) |chunk| {
        var buf: [64]u8 = undefined;
        var w = if (streaming) file.writerStreaming(io, &buf) else file.writer(io, &buf);
        try w.interface.writeAll(chunk);
        try w.interface.flush();
    }
}

fn show(io: std.Io, gpa: std.mem.Allocator, label: []const u8, path: []const u8) !void {
    const text = try std.Io.Dir.cwd().readFileAlloc(io, path, gpa, .limited(1024));
    defer gpa.free(text);
    std.debug.print("{s} -> {d} bytes\n{s}", .{ label, text.len, text });
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const cwd = std.Io.Dir.cwd();

    {
        var f = try cwd.createFile(io, "/tmp/zig-course-positional.txt", .{});
        defer f.close(io);
        try twice(io, f, false);
    }
    try show(io, init.gpa, "writer()         ", "/tmp/zig-course-positional.txt");

    {
        var f = try cwd.createFile(io, "/tmp/zig-course-streaming.txt", .{});
        defer f.close(io);
        try twice(io, f, true);
    }
    try show(io, init.gpa, "writerStreaming()", "/tmp/zig-course-streaming.txt");
}
```

```
writer()          -> 7 bytes
second
writerStreaming() -> 13 bytes
first
second
```

`first\n` is gone. Both positional writers started at offset 0 and the second
overwrote the first.

| | `writer(io, &buf)` | `writerStreaming(io, &buf)` |
|---|---|---|
| writes at | an offset it owns, from 0 | wherever the OS position is |
| two writers on one file | **they overwrite** | they append in order |
| works on a pipe | no — pipes are not seekable | yes |
| `seekTo(n)` / `w.pos = n` | yes | no |

**Rules.** One writer per stream, passed around as `*std.Io.Writer` (lesson
1.3). If you truly need a second, use `writerStreaming`. For **stdout and
stderr in a CLI program, always `writerStreaming`** — your process inherits a
terminal or a redirect that other programs have already written to, and a
positional writer starting at 0 clobbers it. Lesson 8.6 shows the damage.

For readers the same split exists — `file.reader(io, &buf)` is positional and
seekable, `readerStreaming` is not. **stdin must use `readerStreaming`**, since
a pipe has no offset to seek to.

---

Next: [8.3 — Reading Files](./03-reading-files.md)
