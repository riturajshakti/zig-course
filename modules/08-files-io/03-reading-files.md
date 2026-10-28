# 8.3 — Reading Files

Three ways in, in increasing order of control: whole file into memory, line by
line, byte by byte.

## `std.Io.Dir.cwd()`

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const cwd = std.Io.Dir.cwd();
    std.debug.print("type   {s}\n", .{@typeName(@TypeOf(cwd))});
    std.debug.print("handle {d}\n", .{cwd.handle});

    var buf: [1024]u8 = undefined;
    const n = try std.process.currentPath(init.io, &buf);
    std.debug.print("cwd is {s}\n", .{buf[0..n]});
}
```

```
type   Io.Dir
handle -2
```

(The third line prints whatever directory you ran from.)

A `Dir` is just a file descriptor. `cwd()` is comptime-callable on POSIX, costs
nothing, and **must not be closed**. Every path below is absolute (`/tmp/...`),
so `cwd()` is only acting as the resolution root.

## Whole file: `readFileAlloc`

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const gpa = init.gpa;
    const cwd = std.Io.Dir.cwd();

    try cwd.writeFile(io, .{
        .sub_path = "/tmp/zig-course-read.txt",
        .data = "alpha\nbeta\ngamma\n",
    });

    const whole = try cwd.readFileAlloc(io, "/tmp/zig-course-read.txt", gpa, .limited(64 * 1024));
    defer gpa.free(whole);
    std.debug.print("{d} bytes:\n{s}", .{ whole.len, whole });

    var small: [8]u8 = undefined;
    const got = try cwd.readFile(io, "/tmp/zig-course-read.txt", &small);
    std.debug.print("readFile into [8]u8 -> {d} bytes: {s}\n", .{ got.len, got });
}
```

```
17 bytes:
alpha
beta
gamma
readFile into [8]u8 -> 8 bytes: alpha
be
```

Argument order is `(io, path, gpa, limit)` — `io` first, allocator **third**.
That trips everyone coming from 0.14.

| Function | Allocates | On a file bigger than you asked for |
|---|---|---|
| `readFileAlloc(io, p, gpa, limit)` | yes, you `free` | `error.StreamTooLong` |
| `readFile(io, p, &buf)` | no | **silently truncates** to the buffer |

`readFile` giving you 8 of 17 bytes with no error is a real footgun. Compare
`got.len` against your buffer length if you care.

## Why the limit exists

`readFileAlloc` allocates based on a size the **file** reports. Without a cap,
`myprog /dev/zero` or a 40 GB log is an instant OOM, and on a file that grows
while you read it the allocation is unbounded. The limit makes the worst case a
number you chose.

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const gpa = init.gpa;
    const cwd = std.Io.Dir.cwd();

    try cwd.writeFile(io, .{ .sub_path = "/tmp/zig-course-limit.txt", .data = "0123456789" });

    for ([_]std.Io.Limit{ .limited(9), .limited(10), .limited(11), .unlimited }) |lim| {
        const r = cwd.readFileAlloc(io, "/tmp/zig-course-limit.txt", gpa, lim);
        if (r) |text| {
            defer gpa.free(text);
            std.debug.print("ok   ({d} bytes) {s}\n", .{ text.len, text });
        } else |e| {
            std.debug.print("fail {t}\n", .{e});
        }
    }
}
```

```
fail StreamTooLong
fail StreamTooLong
ok   (10 bytes) 0123456789
ok   (10 bytes) 0123456789
```

**`.limited(10)` on a 10-byte file fails.** The stdlib doc says "If **reached**
or exceeded, `error.StreamTooLong`" — the limit is exclusive. Always give a byte
of headroom; in practice pick a round number far above your real maximum.

| Limit | Means |
|---|---|
| `.limited(n)` | fail if the file reaches `n` bytes |
| `.unlimited` | no cap — only for files you created |
| `.nothing` | zero bytes |

## Line by line

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const cwd = std.Io.Dir.cwd();

    try cwd.writeFile(io, .{
        .sub_path = "/tmp/zig-course-lines.txt",
        .data = "one\ntwo\n\nfour\nno trailing newline",
    });

    var file = try cwd.openFile(io, "/tmp/zig-course-lines.txt", .{});
    defer file.close(io);

    var buf: [4096]u8 = undefined;
    var fr = file.reader(io, &buf);
    const r = &fr.interface;

    var n: usize = 0;
    while (true) {
        const line = r.takeDelimiterInclusive('\n') catch |e| switch (e) {
            error.EndOfStream => break,
            else => return e,
        };
        n += 1;
        std.debug.print("{d}: [{s}]\n", .{ n, std.mem.trimEnd(u8, line, "\n") });
    }
    std.debug.print("{d} lines\n", .{n});
}
```

```
1: [one]
2: [two]
3: []
4: [four]
4 lines
```

This is the pattern. Four things about it are load-bearing:

1. **`takeDelimiterInclusive`, not `takeDelimiterExclusive`.** The exclusive
   version does not consume the delimiter, so on an empty line it returns an
   empty slice forever and your loop spins. Lesson 8.6 reproduces the hang.
2. **`while (true)` with an explicit `catch |e| switch`,** not `while (...) |x|`
   with an `else` clause. EOF is an error here, not a `null`.
3. **The line keeps its `\n`** — `trimEnd(u8, line, "\n")` to drop it.
4. **The slice points into `buf`** and is invalid after the next call. Copy it
   if you need to keep it.

### The final line without a newline is silently dropped

`no trailing newline` never appeared above. It is in the buffer when EOF hits,
so collect it on the way out:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const cwd = std.Io.Dir.cwd();

    try cwd.writeFile(io, .{
        .sub_path = "/tmp/zig-course-lines2.txt",
        .data = "one\ntwo\nno trailing newline",
    });

    var file = try cwd.openFile(io, "/tmp/zig-course-lines2.txt", .{});
    defer file.close(io);

    var buf: [4096]u8 = undefined;
    var fr = file.reader(io, &buf);
    const r = &fr.interface;

    var n: usize = 0;
    while (true) {
        const line = r.takeDelimiterInclusive('\n') catch |e| switch (e) {
            error.EndOfStream => {
                const rest = r.buffered();
                if (rest.len > 0) {
                    n += 1;
                    std.debug.print("{d}: [{s}] (no newline)\n", .{ n, rest });
                    r.toss(rest.len);
                }
                break;
            },
            else => return e,
        };
        n += 1;
        std.debug.print("{d}: [{s}]\n", .{ n, std.mem.trimEnd(u8, line, "\n") });
    }
    std.debug.print("{d} lines\n", .{n});
}
```

```
1: [one]
2: [two]
3: [no trailing newline] (no newline)
3 lines
```

### Or: read it all and split

For a file you know is small, this is simpler and has no EOF edge cases:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    try std.Io.Dir.cwd().writeFile(io, .{
        .sub_path = "/tmp/zig-course-split.txt",
        .data = "one\ntwo\n\nfour\nno trailing newline",
    });

    const text = try std.Io.Dir.cwd().readFileAlloc(
        io,
        "/tmp/zig-course-split.txt",
        init.gpa,
        .limited(1 << 20),
    );
    defer init.gpa.free(text);

    std.debug.print("-- splitScalar (keeps blanks) --\n", .{});
    var a = std.mem.splitScalar(u8, text, '\n');
    while (a.next()) |l| std.debug.print("[{s}]\n", .{l});

    std.debug.print("-- tokenizeScalar (drops blanks) --\n", .{});
    var b = std.mem.tokenizeScalar(u8, text, '\n');
    while (b.next()) |l| std.debug.print("[{s}]\n", .{l});
}
```

```
-- splitScalar (keeps blanks) --
[one]
[two]
[]
[four]
[no trailing newline]
-- tokenizeScalar (drops blanks) --
[one]
[two]
[four]
[no trailing newline]
```

`splitScalar` keeps empty fields, `tokenizeScalar` collapses them. Both get the
unterminated last line for free.

| Approach | Memory | Use when |
|---|---|---|
| `readFileAlloc` + `splitScalar` | whole file | small file, you want random access |
| `takeDelimiterInclusive` | one buffer | big or streaming input, logs, stdin |

## Fixed bytes and binary

```zig
const std = @import("std");

const Header = extern struct { magic: u32, version: u16, flags: u16 };

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const cwd = std.Io.Dir.cwd();
    const path = "/tmp/zig-course-binary.bin";

    const h: Header = .{ .magic = 0x5A49_4721, .version = 17, .flags = 0b101 };
    try cwd.writeFile(io, .{ .sub_path = path, .data = std.mem.asBytes(&h) });

    var file = try cwd.openFile(io, path, .{});
    defer file.close(io);
    var buf: [64]u8 = undefined;
    var fr = file.reader(io, &buf);
    const r = &fr.interface;

    const got = try r.takeStructPointer(Header);
    std.debug.print("magic   0x{X}\n", .{got.magic});
    std.debug.print("version {d}\n", .{got.version});
    std.debug.print("flags   0b{b}\n", .{got.flags});

    try fr.seekTo(0);
    const le = try r.takeInt(u32, .little);
    const rest = try r.takeArray(4);
    std.debug.print("as u32 LE 0x{X}\n", .{le});
    std.debug.print("next 4 bytes {any}\n", .{rest.*});
}
```

```
magic   0x5A494721
version 17
flags   0b101
as u32 LE 0x5A494721
next 4 bytes { 17, 0, 5, 0 }
```

`extern struct` keeps C field order, so the layout is what you wrote. A plain
`struct` lets Zig reorder fields and is not safe to `asBytes` across programs.
Note `fr.seekTo(0)` on the **File.Reader**, not on the interface — rewinding is
a positional-reader feature.

## Errors

```zig
const std = @import("std");

fn load(io: std.Io, gpa: std.mem.Allocator, path: []const u8) ![]u8 {
    return std.Io.Dir.cwd().readFileAlloc(io, path, gpa, .limited(1 << 20));
}

pub fn main(init: std.process.Init) !void {
    const paths = [_][]const u8{
        "/tmp/zig-course-read.txt",
        "/tmp/definitely-not-here.txt",
        "/tmp",
        "/etc/sudoers",
    };
    for (paths) |p| {
        if (load(init.io, init.gpa, p)) |text| {
            defer init.gpa.free(text);
            std.debug.print("{s:<30} ok, {d} bytes\n", .{ p, text.len });
        } else |e| switch (e) {
            error.FileNotFound => std.debug.print("{s:<30} no such file\n", .{p}),
            error.IsDir => std.debug.print("{s:<30} that's a directory\n", .{p}),
            error.AccessDenied, error.PermissionDenied => std.debug.print("{s:<30} not allowed\n", .{p}),
            else => std.debug.print("{s:<30} {t}\n", .{ p, e }),
        }
    }
}
```

```
/tmp/zig-course-read.txt       ok, 17 bytes
/tmp/definitely-not-here.txt   no such file
/tmp                           that's a directory
/etc/sudoers                   not allowed
```

Four real outcomes, no mocking. The ones worth handling by name:

| Error | Cause |
|---|---|
| `FileNotFound` | no such path, or a missing parent directory |
| `IsDir` | the path is a directory |
| `AccessDenied` / `PermissionDenied` | both exist; match both or you will miss one |
| `StreamTooLong` | hit your `.limited(n)` |
| `SymLinkLoop` | circular symlink |
| `NameTooLong` | component over the OS limit |

`AccessDenied` and `PermissionDenied` being distinct members is easy to miss —
handle them together.

### Default-on-missing

```zig
const std = @import("std");

fn readOr(io: std.Io, gpa: std.mem.Allocator, path: []const u8, fallback: []const u8) ![]u8 {
    return std.Io.Dir.cwd().readFileAlloc(io, path, gpa, .limited(1 << 16)) catch |e| switch (e) {
        error.FileNotFound => try gpa.dupe(u8, fallback),
        else => e,
    };
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const gpa = init.gpa;
    try std.Io.Dir.cwd().writeFile(io, .{
        .sub_path = "/tmp/zig-course-present.conf",
        .data = "mode=real\n",
    });

    for ([_][]const u8{ "/tmp/zig-course-present.conf", "/tmp/zig-course-absent.conf" }) |p| {
        const cfg = try readOr(io, gpa, p, "mode=default\n");
        defer gpa.free(cfg);
        std.debug.print("{s:<30} {s}", .{ std.fs.path.basename(p), cfg });
    }
}
```

```
zig-course-present.conf        mode=real
zig-course-absent.conf         mode=default
```

`gpa.dupe` on the fallback keeps the return type one thing the caller always
frees — never mix "you own this" and "you don't" in one signature.

---

Next: [8.4 — Writing Files](./04-writing-files.md)
