# 15.6 — Syscalls & the OS

**`std.posix` in 0.17 is deliberately thin — about 54 functions.** Most of what
you remember from 0.13 is gone, because file and socket I/O moved to `std.Io`.
What's left is the stuff `std.Io` can't abstract: memory mapping, process and
resource introspection, signals, and the tracing interfaces.

## What `std.posix` still gives you

```zig
const std = @import("std");

pub fn main() !void {
    const un = std.posix.uname();
    std.debug.print("uname:\n", .{});
    std.debug.print("  sysname  = {s}\n", .{std.mem.sliceTo(&un.sysname, 0)});
    std.debug.print("  release  = {s}\n", .{std.mem.sliceTo(&un.release, 0)});
    std.debug.print("  machine  = {s}\n", .{std.mem.sliceTo(&un.machine, 0)});

    std.debug.print("\ngetppid() = {d}\n", .{std.posix.getppid()});

    var host: [std.posix.HOST_NAME_MAX]u8 = undefined;
    std.debug.print("gethostname() = {d} chars\n", .{(try std.posix.gethostname(&host)).len});

    std.debug.print("\nrlimits (soft / hard):\n", .{});
    inline for (.{ .NOFILE, .STACK, .CORE }) |which| {
        const l = try std.posix.getrlimit(which);
        std.debug.print("  {t:<7} {d} / {d}\n", .{ which, l.cur, l.max });
    }

    const ru = std.posix.getrusage(std.posix.rusage.SELF);
    std.debug.print("\ngetrusage(SELF):\n", .{});
    std.debug.print("  maxrss      = {d} B\n", .{ru.maxrss});
    std.debug.print("  minflt      = {d}  majflt = {d}\n", .{ ru.minflt, ru.majflt });
    std.debug.print("  user time   = {d} s {d} us\n", .{ ru.utime.sec, @as(u64, @intCast(ru.utime.usec)) });

    std.debug.print("\npage size = {d}\n", .{std.heap.pageSize()});
    const b = @import("builtin");
    std.debug.print("target    = {t}-{t}\n", .{ b.cpu.arch, b.os.tag });
}
```

```
uname:
  sysname  = Darwin
  release  = 25.5.0
  machine  = arm64

getppid() = 9564
gethostname() = 26 chars

rlimits (soft / hard):
  NOFILE  10240 / 10240
  STACK   8372224 / 67092480
  CORE    0 / 9223372036854775807

getrusage(SELF):
  maxrss      = 1916928 B
  minflt      = 16049  majflt = 151
  user time   = 1 s 45121 us

page size = 16384
target    = aarch64-macos
```

Everything there is operationally useful in a backend service:

| Call | Tells you | Why you'd care in production |
|---|---|---|
| `uname()` | kernel name / release / arch | log it at startup; it's the first thing you want in a bug report |
| `getrlimit(.NOFILE)` | max open fds | **10240 here.** A connection-per-fd server dies at this number. Check it at boot and log a warning |
| `getrlimit(.STACK)` | 8 MB soft, 64 MB hard | why deep recursion dies at a specific depth |
| `getrusage(.SELF)` | peak RSS, page faults, CPU time | **`majflt`** is the one to watch — major faults mean you're hitting disk |
| `std.heap.pageSize()` | **16384 on Apple silicon**, 4096 on x86-64 Linux | never hardcode 4096 |
| `getppid()` | parent pid | detect "my supervisor died" |

`std.heap.pageSize()` is a **function**, not a constant — it reads the real
value at runtime, because the same binary can run on hosts with different page
sizes.

Note the C-string handling: `un.sysname` is a fixed `[256]u8` with a NUL
somewhere inside, so `std.mem.sliceTo(&un.sysname, 0)` is how you get a usable
`[]const u8`.

### The inventory

Probed on this toolchain, one program per name:

| Still in `std.posix` | |
|---|---|
| **memory** | `mmap` `munmap` `mremap` `msync` `madvise` `mincore` |
| **process / limits** | `getppid` `getrlimit` `setrlimit` `getrusage` `uname` `gethostname` `prctl` `sched_getaffinity` |
| **signals** | `sigaction` `sigprocmask` `sigfillset` `sigemptyset` `sigaddset` `kill` `raise` `sigaltstack` |
| **polling** | `poll` `ppoll` |
| **tracing** | `ptrace` `perf_event_open` (Linux) |
| **misc** | `read` `openat` `sysctl` `setsockopt` `getpeername` `memfd_create` `sync` `fdatasync` `tcgetattr` `tcsetattr` |

| **Gone** — moved into `std.Io` | Use instead |
|---|---|
| `posix.open` | `std.Io.Dir.cwd().openFile(io, path, .{})` |
| `posix.close` | `file.close(io)` |
| `posix.write` | a `*std.Io.Writer` |
| `posix.lseek` | positional `read`/`write` on `std.Io.File` |
| `posix.socket` / `connect` / `bind` | `std.Io.net` |
| `posix.getpid` | not exposed; `getppid` is |
| `posix.fork` / `execve` / `pipe` / `dup2` | `std.process.Child` |

**If you're writing from memory and `std.posix.X` doesn't exist, the answer is
almost always `std.Io`.** That's the 0.17 design: one portable I/O interface,
and `std.posix` for the things that are genuinely OS-specific.

## `mmap`

```zig
const std = @import("std");

pub fn main() !void {
    const page = std.heap.pageSize();
    std.debug.print("page size = {d}\n\n", .{page});

    const anon = try std.posix.mmap(
        null,
        4 * page,
        .{ .READ = true, .WRITE = true },
        .{ .TYPE = .PRIVATE, .ANONYMOUS = true },
        -1,
        0,
    );
    defer std.posix.munmap(anon);

    std.debug.print("mmap anonymous:\n", .{});
    std.debug.print("  len        = {d} ({d} pages)\n", .{ anon.len, anon.len / page });
    std.debug.print("  addr mod page = {d}  (page-aligned)\n", .{@intFromPtr(anon.ptr) % page});
    std.debug.print("  zero-filled? {}\n", .{std.mem.allEqual(u8, anon, 0)});
    std.debug.print("  slice type = {s}\n", .{@typeName(@TypeOf(anon))});

    anon[0] = 0xAB;
    anon[anon.len - 1] = 0xCD;
    std.debug.print("  wrote first=0x{x} last=0x{x}\n", .{ anon[0], anon[anon.len - 1] });

    var resident: [4]u8 = undefined;
    try std.posix.mincore(anon.ptr, anon.len, &resident);
    std.debug.print("\nmincore (page resident bitmap) = {any}\n", .{resident});

    try std.posix.madvise(anon.ptr, anon.len, std.posix.MADV.DONTNEED);
    std.debug.print("madvise DONTNEED -> ok\n", .{});
    try std.posix.mincore(anon.ptr, anon.len, &resident);
    std.debug.print("mincore after DONTNEED        = {any}\n", .{resident});

    const ro = try std.posix.mmap(null, page, .{ .READ = true }, .{ .TYPE = .PRIVATE, .ANONYMOUS = true }, -1, 0);
    defer std.posix.munmap(ro);
    std.debug.print("\nread-only mapping: len={d} first byte={d}\n", .{ ro.len, ro[0] });
}
```

```
page size = 16384

mmap anonymous:
  len        = 65536 (4 pages)
  addr mod page = 0  (page-aligned)
  zero-filled? true
  slice type = []align(16384) u8
  wrote first=0xab last=0xcd

mincore (page resident bitmap) = { 135, 135, 131, 135 }
madvise DONTNEED -> ok
mincore after DONTNEED        = { 133, 133, 129, 133 }

read-only mapping: len=16384 first byte=0
```

**The 0.17 signature trap:** on macOS the PROT flags are a **packed struct**,
not OR-able constants:

```zig
.{ .READ = true, .WRITE = true }     // yes
std.posix.PROT.READ | std.posix.PROT.WRITE   // what every tutorial says — won't compile
```

Same for the flags argument: `.{ .TYPE = .PRIVATE, .ANONYMOUS = true }`.

Three things the output proves:

- **The return type is `[]align(16384) u8`** — a slice, carrying the page
  alignment in its type. You index it like any other slice, and `munmap` takes
  the slice back (no length bookkeeping).
- **Anonymous pages are zero-filled** (`allEqual(…, 0)` is `true`) and the
  kernel guarantees it. You don't pay for the zeroing until you touch a page.
- **`mincore` reports residency per page** and `madvise(.DONTNEED)` changes it —
  bit 0 of each byte is the "resident" flag, and it flipped from set to clear in
  every page. The exact byte values vary run to run (the other bits are
  implementation-defined hints), so expect different numbers; the low bit is the
  part you can rely on.

| When to `mmap` instead of `alloc` | |
|---|---|
| **a large read-only file you search** | the page cache *is* your buffer; no copy, no RSS for untouched pages |
| **sharing memory between processes** | `.TYPE = .SHARED` |
| **a huge sparse region** | reserve address space, commit pages on touch |
| **you need guaranteed page alignment** | the return type says `align(16384)` |
| **you're writing an allocator** | this is what `page_allocator` does underneath |

| When **not** to | |
|---|---|
| anything under a few pages | 16 KB minimum granularity here; use the allocator |
| a file you read once, sequentially | `readFileAlloc` is simpler and no slower |
| you need it portable to Windows | use `std.Io` / the allocator |

### Mapping a file

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const path = "/tmp/zig-mmap-demo.txt";
    const cwd = std.Io.Dir.cwd();

    try cwd.writeFile(io, .{ .sub_path = path, .data = "MMAPPED FILE CONTENT -- hello from the page cache\n" });

    const file = try cwd.openFile(io, path, .{ .mode = .read_only });
    defer file.close(io);

    const size = (try file.stat(io)).size;
    std.debug.print("file size = {d} B, fd = {d}\n", .{ size, file.handle });

    const mapped = try std.posix.mmap(
        null,
        size,
        .{ .READ = true },
        .{ .TYPE = .PRIVATE },
        file.handle,
        0,
    );
    defer std.posix.munmap(mapped);

    std.debug.print("mmap len  = {d}, addr mod page = {d}\n", .{ mapped.len, @intFromPtr(mapped.ptr) % std.heap.pageSize() });
    std.debug.print("contents  = {s}", .{mapped[0..size]});
    std.debug.print("read via pointer, no read() call, no buffer copy\n", .{});

    std.debug.print("\nbyte 0 = '{c}'  byte 7 = '{c}'\n", .{ mapped[0], mapped[7] });
    std.debug.print("indexOf \"page\" = {?d}\n", .{std.mem.indexOf(u8, mapped[0..size], "page")});
}
```

```
file size = 50 B, fd = 3
mmap len  = 50, addr mod page = 0
contents  = MMAPPED FILE CONTENT -- hello from the page cache
read via pointer, no read() call, no buffer copy

byte 0 = 'M'  byte 7 = ' '
indexOf "page" = 39
```

**This is the bridge between the two worlds**: `std.Io` opens the file
portably, `file.handle` hands you the raw fd (`std.posix.fd_t`), and
`std.posix.mmap` takes it from there. `std.Io.File.handle` is the escape hatch
for every syscall `std.Io` doesn't wrap.

After the mapping, `std.mem.indexOf` works on it like any slice — a 50 GB file
becomes a `[]const u8` you can search without a 50 GB buffer.

## File descriptors and `poll`

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const path = "/tmp/zig-fd-demo.txt";
    const cwd = std.Io.Dir.cwd();
    try cwd.writeFile(io, .{ .sub_path = path, .data = "line one\nline two\n" });

    const f = try cwd.openFile(io, path, .{ .mode = .read_only });
    defer f.close(io);
    std.debug.print("openFile fd            = {d}\n", .{f.handle});
    std.debug.print("stdin/out/err fds      = {d} {d} {d}\n", .{
        std.Io.File.stdin().handle, std.Io.File.stdout().handle, std.Io.File.stderr().handle,
    });

    var buf: [64]u8 = undefined;
    const n = try std.posix.read(f.handle, &buf);
    std.debug.print("posix.read({d}) -> {d} bytes: \"{s}\"\n", .{ f.handle, n, buf[0 .. n - 1] });

    var fds = [_]std.posix.pollfd{
        .{ .fd = f.handle, .events = std.posix.POLL.IN, .revents = 0 },
    };
    const ready = try std.posix.poll(&fds, 0);
    std.debug.print("\npoll(timeout=0) ready  = {d}, revents = 0x{x}\n", .{ ready, fds[0].revents });
    std.debug.print("  POLL.IN set?         = {}\n", .{fds[0].revents & std.posix.POLL.IN != 0});

    const st = try f.stat(io);
    std.debug.print("\nstat: size={d} kind={t}\n", .{ st.size, st.kind });
}
```

```
openFile fd            = 3
stdin/out/err fds      = 0 1 2
posix.read(3) -> 18 bytes: "line one
line two"

poll(timeout=0) ready  = 1, revents = 0x1
  POLL.IN set?         = true

stat: size=18 kind=file
```

| Fact | |
|---|---|
| **fds 0/1/2 are always stdin/stdout/stderr** | the first file you open is 3 |
| `std.Io.File.handle` is `std.posix.fd_t` (an `i32`) | the universal escape hatch |
| `std.posix.read` **can return fewer bytes than asked** | always loop, or use `std.Io`, which handles it |
| `poll` **mutates** `revents` in place | check `revents & POLL.IN`, not the return value alone |
| `poll`'s return is a **count** of ready fds | 0 means timeout |
| `POLL.IN` here is a bare constant, unlike mmap's PROT | 0.17 is not uniform about this — check the source |

**`poll` is not how you should write a server in 0.17.** `std.Io`'s `async` /
`concurrent` is the portable concurrency layer (module 12), and it uses the
best mechanism per platform — `kqueue` on macOS, `epoll`/`io_uring` on Linux.
Reach for `poll` directly only when you're integrating with a C library that
hands you raw fds.

## `std.Io` is the portable layer

The rule that keeps your code compiling on three platforms:

| You want | Use | Portable? |
|---|---|---|
| open / read / write / close a file | `std.Io.File` | **yes** |
| directories, walking, stat | `std.Io.Dir` | **yes** |
| TCP / UDP | `std.Io.net` | **yes** |
| concurrency, timers, cancellation | `io.async`, `io.concurrent`, `std.Io.Clock` | **yes** |
| spawn a process | `std.process.Child` | **yes** |
| memory mapping | `std.posix.mmap` | POSIX only (Windows differs) |
| rlimits, rusage, uname | `std.posix` | POSIX only |
| `epoll` / `io_uring` / `signalfd` / `eventfd` | **`std.os.linux`** | **Linux only** |
| `kqueue` | `std.posix` / `std.c` | BSD + macOS only |

**`std.os.linux.*` will not compile on a macOS host** — it's gated on the
target OS. If you need Linux-specific syscalls while developing on a Mac, build
for Linux (`-Dtarget=x86_64-linux-musl`) and your code compiles; you just can't
run it locally without a VM or container.

```zig
const builtin = @import("builtin");
if (builtin.os.tag == .linux) {
    // std.os.linux.* is only legal inside a comptime-known Linux branch
}
```

`builtin.os.tag` is comptime-known, so the dead branch is never compiled.

## Seeing the syscalls

| Platform | Command | Note |
|---|---|---|
| Linux | `strace -c ./myprog` | counts per syscall; the single best "why is this slow" tool |
| Linux | `strace -f -e trace=openat,read,write ./myprog` | follow children, filter |
| macOS | `sudo dtruss ./myprog` | needs sudo, and SIP blocks it for system binaries |
| macOS | `sudo dtrace -n 'syscall:::entry /pid == $target/ {@[probefunc] = count();}' -c ./myprog` | aggregated counts |
| either | `std.posix.getrusage(.SELF)` from inside | no privileges needed, works in prod |

What to look for: **the same syscall thousands of times.** An unbuffered writer
doing one `write(2)` per line, or a config file re-opened per request, shows up
instantly in `strace -c` and never shows up in a profiler that only samples
user time.

This is also the concrete payoff of 0.17's buffered writers: `out.print()` a
thousand times then `flush()` once is **one** `write` syscall, not a thousand.
Forgetting the `flush()` is the famous trap; the reason it exists is here.

---

Next: [15.7 — Measuring Performance](./07-performance.md)
