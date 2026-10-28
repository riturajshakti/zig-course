# 8.6 — CLI Programs

Args, stdin, environment, exit codes. Everything a real command-line tool needs.

> **`std.process.argsAlloc` is gone.** So are `argsWithAllocator` and
> `std.os.argv`. Arguments come from `init.minimal.args`.

> **Use `writerStreaming` for stdout and stderr in a CLI program.** Not
> `writer`. The reason is at the bottom of this page, and it is not cosmetic.

## Arguments: `iterate()`

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    var buf: [1024]u8 = undefined;
    var w = std.Io.File.stdout().writerStreaming(init.io, &buf);
    const out = &w.interface;
    defer out.flush() catch {};

    var it = init.minimal.args.iterate();
    const exe = it.next() orelse "?";
    try out.print("exe basename: {s}\n", .{std.fs.path.basename(exe)});
    var n: usize = 1;
    while (it.next()) |arg| : (n += 1) try out.print("arg[{d}] = {s}\n", .{ n, arg });
    try out.print("{d} argument(s) after the program name\n", .{n - 1});
}
```

```sh
zig run args.zig -- --verbose input.txt -n 3
```

```
exe basename: args
arg[1] = --verbose
arg[2] = input.txt
arg[3] = -n
arg[4] = 3
4 argument(s) after the program name
```

**The `--` is required.** `zig run args.zig --verbose` fails with
`unrecognized file extension`, because `zig` claims the flag for itself. A
compiled binary needs no `--`.

`arg[0]` is the executable path — under `zig run` that is somewhere in the build
cache, so `basename` is the only part worth printing.

## Arguments: `toSlice()` for random access

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const args = try init.minimal.args.toSlice(init.arena.allocator());
    var buf: [1024]u8 = undefined;
    var w = std.Io.File.stdout().writerStreaming(init.io, &buf);
    const out = &w.interface;
    defer out.flush() catch {};

    try out.print("{d} args, type {s}\n", .{ args.len, @typeName(@TypeOf(args)) });
    for (args[1..], 1..) |a, i| try out.print("  [{d}] {s}\n", .{ i, a });
    if (args.len > 2) try out.print("random access: args[2] = {s}\n", .{args[2]});
}
```

```sh
zig run toslice.zig -- alpha beta gamma
```

```
4 args, type []const [:0]const u8
  [1] alpha
  [2] beta
  [3] gamma
random access: args[2] = beta
```

`toSlice(arena)` wants an **arena**, because on Windows it must split and own the
command line and there is nothing to free individually. `init.arena.allocator()`
— remember `init.arena` is a `*ArenaAllocator`, not an `Allocator`.

| | `iterate()` | `toSlice(arena)` |
|---|---|---|
| allocates | no (POSIX) | yes |
| access | forward only | indexed, `.len` |
| element type | `?[:0]const u8` | `[]const [:0]const u8` |
| use for | streaming flag parsing | lookahead, counting, re-scanning |

## A real flag parser

```zig
const std = @import("std");

const Options = struct {
    verbose: bool = false,
    count: u32 = 1,
    name: []const u8 = "world",
    rest: std.ArrayList([]const u8) = .empty,
};

fn parse(gpa: std.mem.Allocator, it: *std.process.Args.Iterator) !Options {
    var o: Options = .{};
    _ = it.next();
    while (it.next()) |arg| {
        if (std.mem.eql(u8, arg, "-v") or std.mem.eql(u8, arg, "--verbose")) {
            o.verbose = true;
        } else if (std.mem.eql(u8, arg, "-n") or std.mem.eql(u8, arg, "--count")) {
            const v = it.next() orelse return error.MissingValue;
            o.count = std.fmt.parseInt(u32, v, 10) catch return error.BadNumber;
        } else if (std.mem.startsWith(u8, arg, "--name=")) {
            o.name = arg["--name=".len..];
        } else if (std.mem.startsWith(u8, arg, "-")) {
            return error.UnknownFlag;
        } else {
            try o.rest.append(gpa, arg);
        }
    }
    return o;
}

pub fn main(init: std.process.Init) !void {
    var buf: [1024]u8 = undefined;
    var w = std.Io.File.stdout().writerStreaming(init.io, &buf);
    const out = &w.interface;
    defer out.flush() catch {};

    var it = init.minimal.args.iterate();
    var o = parse(init.gpa, &it) catch |e| {
        var ebuf: [256]u8 = undefined;
        var ew = std.Io.File.stderr().writerStreaming(init.io, &ebuf);
        try ew.interface.print("error: {t}\n", .{e});
        try ew.interface.flush();
        return;
    };
    defer o.rest.deinit(init.gpa);

    try out.print("verbose={} count={d} name={s} rest={d}\n", .{ o.verbose, o.count, o.name, o.rest.items.len });
    for (0..o.count) |_| try out.print("hello, {s}!\n", .{o.name});
    for (o.rest.items) |f| try out.print("  file: {s}\n", .{f});
}
```

```sh
zig build-exe cli.zig && ./cli -v -n 2 --name=zig a.txt b.txt
```

```
verbose=true count=2 name=zig rest=2
hello, zig!
hello, zig!
  file: a.txt
  file: b.txt
```

```sh
./cli
```

```
verbose=false count=1 name=world rest=0
hello, world!
```

```sh
./cli -n abc
```

```
error: BadNumber
```

```sh
./cli --wat
```

```
error: UnknownFlag
```

Four shapes in one loop:

| Pattern | Example | How |
|---|---|---|
| boolean flag | `-v`, `--verbose` | `mem.eql` on both spellings |
| separated value | `-n 2` | `it.next()` inside the branch |
| attached value | `--name=zig` | `startsWith` then slice past the `=` |
| positional | `a.txt` | the `else` branch |
| unknown flag | `--wat` | starts with `-` and matched nothing |

The iterator type is `*std.process.Args.Iterator` — useful when your parser
lives in another function, as here.

## stdin

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    var inbuf: [4096]u8 = undefined;
    var r = std.Io.File.stdin().readerStreaming(io, &inbuf);

    var outbuf: [4096]u8 = undefined;
    var w = std.Io.File.stdout().writerStreaming(io, &outbuf);
    const out = &w.interface;
    defer out.flush() catch {};

    var lines: usize = 0;
    var words: usize = 0;
    var bytes: usize = 0;
    while (true) {
        const line = r.interface.takeDelimiterInclusive('\n') catch |e| switch (e) {
            error.EndOfStream => break,
            else => return e,
        };
        lines += 1;
        bytes += line.len;
        var it = std.mem.tokenizeAny(u8, line, " \t\n");
        while (it.next()) |_| words += 1;
    }
    try out.print("{d} lines  {d} words  {d} bytes\n", .{ lines, words, bytes });
}
```

```sh
printf 'hello world\n\nthe quick brown fox\n' | ./wc
```

```
3 lines  6 words  33 bytes
```

```sh
printf '' | ./wc
```

```
0 lines  0 words  0 bytes
```

```sh
./wc < wc.zig
```

```
27 lines  110 words  859 bytes
```

Three details, each of which will otherwise cost you an hour:

1. **`readerStreaming`, not `reader`.** A pipe is not seekable, so the
   positional reader is wrong.
2. **`r.interface.takeDelimiterInclusive`** — `.interface` on a `File.Reader`,
   same as writers.
3. **`while (true)` + `catch |e| switch`,** and `takeDelimiter**Inclusive**`.

### The hang — **broken on purpose**

`takeDelimiterExclusive` in a `while (...) |x| ... else` loop does not terminate:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    var inbuf: [4096]u8 = undefined;
    var r = std.Io.File.stdin().readerStreaming(init.io, &inbuf);
    var n: usize = 0;
    while (r.interface.takeDelimiterExclusive('\n')) |line| {
        n += 1;
        std.debug.print("{d}: {s}\n", .{ n, line });
        if (n > 10) {
            std.debug.print("... runaway, bailing out\n", .{});
            break;
        }
    } else |_| {}
    std.debug.print("total {d}\n", .{n});
}
```

```sh
printf 'a\n\nb\n' | ./spin
```

```
1: a
2: 
3: 
4: 
5: 
6: 
7: 
8: 
9: 
10: 
11: 
... runaway, bailing out
total 11
```

Three lines of input, eleven iterations, and without the `n > 10` escape hatch
it never stops. The exclusive variant does **not consume the delimiter**, so on
reaching an empty line it returns an empty slice, the cursor does not move, and
the next call returns the same empty slice forever. It never signals
`EndOfStream`.

Use `takeDelimiterInclusive` and trim, or `takeDelimiter` (which returns
`?[]const u8` and gives you `null` at EOF).

## Environment variables

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const env = init.environ_map;
    var buf: [1024]u8 = undefined;
    var w = std.Io.File.stdout().writerStreaming(init.io, &buf);
    const out = &w.interface;
    defer out.flush() catch {};

    try out.print("MY_FLAG    {?s}\n", .{env.get("MY_FLAG")});
    try out.print("NOT_SET    {?s}\n", .{env.get("NOT_SET_ANYWHERE")});
    try out.print("has PATH   {}\n", .{env.contains("PATH")});
    try out.print("has NOPE   {}\n", .{env.contains("NOPE")});

    const level = env.get("LOG_LEVEL") orelse "info";
    try out.print("LOG_LEVEL  {s} (default info)\n", .{level});

    try env.put("ADDED", "yes");
    try out.print("ADDED      {?s}\n", .{env.get("ADDED")});
    _ = env.swapRemove("ADDED");
    try out.print("after remove {?s}\n", .{env.get("ADDED")});
}
```

```sh
MY_FLAG=on LOG_LEVEL=debug ./env
```

```
MY_FLAG    on
NOT_SET    null
has PATH   true
has NOPE   false
LOG_LEVEL  debug (default info)
ADDED      yes
after remove null
```

```sh
env -u MY_FLAG -u LOG_LEVEL ./env
```

```
MY_FLAG    null
NOT_SET    null
has PATH   true
has NOPE   false
LOG_LEVEL  info (default info)
ADDED      yes
after remove null
```

| Call | Returns |
|---|---|
| `env.get(k)` | `?[]const u8` — pair with `orelse` for a default |
| `env.contains(k)` | `bool` |
| `env.put(k, v)` | allocates, affects this process only |
| `env.swapRemove(k)` | `bool` |
| `env.count()` | how many |
| `env.keys()` / `env.values()` | slices for iterating |

`init.environ_map` is **not threadsafe** and `put` does not propagate to child
processes unless you pass the map to `std.process.Child` explicitly.

## Exit codes

Declare `main` as returning `u8` and return the code.

```zig
const std = @import("std");

fn run(init: std.process.Init) !u8 {
    var buf: [512]u8 = undefined;
    var w = std.Io.File.stdout().writerStreaming(init.io, &buf);
    const out = &w.interface;
    defer out.flush() catch {};

    var ebuf: [512]u8 = undefined;
    var ew = std.Io.File.stderr().writerStreaming(init.io, &ebuf);
    const err = &ew.interface;
    defer err.flush() catch {};

    var it = init.minimal.args.iterate();
    _ = it.next();
    const path = it.next() orelse {
        try err.writeAll("usage: wc <file>\n");
        return 2;
    };

    const text = std.Io.Dir.cwd().readFileAlloc(init.io, path, init.gpa, .limited(1 << 20)) catch |e| {
        try err.print("cannot read {s}: {t}\n", .{ path, e });
        return 1;
    };
    defer init.gpa.free(text);

    try out.print("{d}\n", .{std.mem.countScalar(u8, text, '\n')});
    return 0;
}

pub fn main(init: std.process.Init) !u8 {
    return run(init);
}
```

```sh
./lc /tmp/zig-course-read.txt; echo "exit=$?"
```

```
3
exit=0
```

```sh
./lc; echo "exit=$?"
```

```
usage: wc <file>
exit=2
```

```sh
./lc /tmp/nope-nope.txt; echo "exit=$?"
```

```
cannot read /tmp/nope-nope.txt: FileNotFound
exit=1
```

| `main` signature | Exit code |
|---|---|
| `!void` | 0, or **1** on any returned error (plus a stack trace on stderr) |
| `u8` / `!u8` | whatever you return |
| `void` | always 0 |

Returning `!u8` lets you still use `try` for the unexpected and an explicit
`return 2` for the expected. Convention: `0` success, `1` runtime failure, `2`
bad usage.

## stdout vs stderr

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    var obuf: [512]u8 = undefined;
    var ebuf: [512]u8 = undefined;
    var ow = std.Io.File.stdout().writerStreaming(init.io, &obuf);
    var ew = std.Io.File.stderr().writerStreaming(init.io, &ebuf);
    const out = &ow.interface;
    const log = &ew.interface;
    defer out.flush() catch {};
    defer log.flush() catch {};

    try log.writeAll("scanning...\n");
    for ([_][]const u8{ "alpha", "beta", "gamma" }) |w| try out.print("{s}\n", .{w});
    try log.writeAll("done, 3 results\n");
}
```

```sh
./streams
```

```
scanning...
done, 3 results
alpha
beta
gamma
```

```sh
./streams 2>/dev/null | sort -r
```

```
gamma
beta
alpha
```

```sh
./streams 2>&1 1>/dev/null
```

```
scanning...
done, 3 results
```

The data pipes cleanly into `sort` because the logs are not in it. That is the
whole rule:

| Stream | Carries |
|---|---|
| **stdout** | the data — only what a pipe should receive |
| **stderr** | progress, warnings, errors, usage, anything a human reads |

Note the interleaving in the first run: both `log` writes appear before any
`out` write even though the source alternates. Each writer has its **own**
buffer, and the deferred flushes run LIFO — `log` was deferred last, so it
drains first. Relative order between two buffered streams is not source order.
Flush explicitly at the point where ordering matters.

## Why `writerStreaming`, not `writer` — **broken on purpose**

`writer()` is positional and starts at offset 0. Your process inherits stdout
from the shell, and that descriptor may already be at a nonzero offset, so a
positional writer **overwrites what is already there**.

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    var buf: [256]u8 = undefined;
    var w = std.Io.File.stdout().writer(init.io, &buf);
    try w.interface.writeAll("FROM ZIG\n");
    try w.interface.flush();
}
```

```sh
zig build-exe clobber.zig
{ echo "SHELL LINE ONE"; ./clobber; } > out.txt; cat out.txt
```

```
FROM ZIG
E ONE
```

`SHELL LINE ON` is gone — overwritten by the 9 bytes the Zig program wrote at
offset 0. Change one word:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    var buf: [256]u8 = undefined;
    var w = std.Io.File.stdout().writerStreaming(init.io, &buf);
    try w.interface.writeAll("FROM ZIG\n");
    try w.interface.flush();
}
```

```sh
{ echo "SHELL LINE ONE"; ./fixed; } > out.txt; cat out.txt
```

```
SHELL LINE ONE
FROM ZIG
```

| | stdout / stderr / stdin | a file you opened |
|---|---|---|
| writing | **`writerStreaming(io, &buf)`** | `writer(io, &buf)` is fine |
| reading | **`readerStreaming(io, &buf)`** | `reader(io, &buf)`, seekable |

The positional writer is correct for files — you own the offset and seeking is
useful. For an inherited stream you do not own the offset, so stream it.

## The CLI skeleton

```zig
const std = @import("std");

const Config = struct { verbose: bool = false, path: ?[]const u8 = null };

fn usage(err: *std.Io.Writer) !void {
    try err.writeAll("usage: tool [-v] <file>\n");
}

fn run(init: std.process.Init, out: *std.Io.Writer, err: *std.Io.Writer) !u8 {
    var cfg: Config = .{};
    var it = init.minimal.args.iterate();
    _ = it.next();
    while (it.next()) |arg| {
        if (std.mem.eql(u8, arg, "-v")) {
            cfg.verbose = true;
        } else if (std.mem.eql(u8, arg, "-h")) {
            try usage(err);
            return 0;
        } else if (std.mem.startsWith(u8, arg, "-")) {
            try err.print("unknown flag: {s}\n", .{arg});
            return 2;
        } else {
            cfg.path = arg;
        }
    }

    const path = cfg.path orelse {
        try usage(err);
        return 2;
    };
    if (cfg.verbose) try err.print("reading {s}\n", .{path});

    const text = std.Io.Dir.cwd().readFileAlloc(init.io, path, init.gpa, .limited(1 << 20)) catch |e| {
        try err.print("{s}: {t}\n", .{ path, e });
        return 1;
    };
    defer init.gpa.free(text);

    try out.print("{d} bytes, {d} lines\n", .{ text.len, std.mem.countScalar(u8, text, '\n') });
    return 0;
}

pub fn main(init: std.process.Init) !u8 {
    var obuf: [4096]u8 = undefined;
    var ebuf: [1024]u8 = undefined;
    var ow = std.Io.File.stdout().writerStreaming(init.io, &obuf);
    var ew = std.Io.File.stderr().writerStreaming(init.io, &ebuf);
    defer ow.interface.flush() catch {};
    defer ew.interface.flush() catch {};
    return run(init, &ow.interface, &ew.interface);
}
```

```sh
./tool -v /tmp/zig-course-read.txt; echo "exit=$?"
```

```
reading /tmp/zig-course-read.txt
17 bytes, 3 lines
exit=0
```

```sh
./tool -h; echo "exit=$?"
```

```
usage: tool [-v] <file>
exit=0
```

```sh
./tool -x; echo "exit=$?"
```

```
unknown flag: -x
exit=2
```

```sh
./tool; echo "exit=$?"
```

```
usage: tool [-v] <file>
exit=2
```

Why this shape:

- `main` owns the two writers and their flushes; nothing below creates one.
- `run` takes `*std.Io.Writer` twice, so it is testable against
  `std.Io.Writer.fixed` with no `io` and no process at all.
- Every exit is an explicit `u8`.

---

That is Module 08. You can now read and write files, walk directories, and
build a command-line tool that behaves correctly in a pipeline.

Next: [Module 09 — Testing & Debugging](../09-testing-debugging/)
