# 11.3 — C Strings

Zig strings carry a length. C strings carry a NUL byte somewhere ahead. Nearly
every C-interop bug lives in that gap.

| | Zig `[]const u8` | C `const char *` |
|---|---|---|
| Length | stored, `s.len` | found by scanning for `\0` |
| Size of the value | 16 bytes (ptr + len) | 8 bytes (ptr) |
| Can contain `\0` | yes | no — it *is* the terminator |
| Length cost | O(1) | O(n), every call |
| Can be NULL | no | yes |
| Zig type | `[]const u8` | `[*:0]const u8` or `?[*:0]const u8` |

## Both directions in one program

```zig
const std = @import("std");
const c = std.c;

extern fn strlen(s: [*:0]const u8) usize;
extern fn strchr(s: [*:0]const u8, ch: c_int) ?[*:0]const u8;

pub fn main() !void {
    const zig_str: []const u8 = "hello";
    const c_str: [*:0]const u8 = "hello";

    std.debug.print("zig  {s} len={d} type={s}\n", .{ zig_str, zig_str.len, @typeName(@TypeOf(zig_str)) });
    std.debug.print("cstr {s} strlen={d} type={s}\n", .{ c_str, strlen(c_str), @typeName(@TypeOf(c_str)) });

    const back: []const u8 = std.mem.span(c_str);
    std.debug.print("span -> len={d} type={s}\n", .{ back.len, @typeName(@TypeOf(back)) });

    const raw: [*c]const u8 = c_str;
    const s2 = std.mem.sliceTo(raw, 0);
    std.debug.print("sliceTo -> {s} len={d} type={s}\n", .{ s2, s2.len, @typeName(@TypeOf(s2)) });

    const lit: [:0]const u8 = "abc";
    std.debug.print("literal [:0] len={d} byte[3]={d}\n", .{ lit.len, lit[3] });

    if (strchr(c_str, 'l')) |p| {
        std.debug.print("strchr('l') -> {s}\n", .{std.mem.span(p)});
    }

    const home = std.mem.span(c.getenv("HOME") orelse "?");
    std.debug.print("getenv HOME = {s}\n", .{home});
}
```

```sh
zig run -lc strs.zig
```

```
zig  hello len=5 type=[]const u8
cstr hello strlen=5 type=[*:0]const u8
span -> len=5 type=[]const u8
sliceTo -> hello len=5 type=[:0]const u8
literal [:0] len=3 byte[3]=0
strchr('l') -> llo
getenv HOME = /Users/raj
```

Five facts in that output:

- **A Zig string literal is already `[:0]const u8`.** `"abc".len == 3`, but
  `"abc"[3] == 0` — the NUL is there, just not counted. So **literals pass to C
  for free**, no conversion and no allocation.
- **`std.mem.span` turns a sentinel pointer into a slice** by scanning. The
  result is `[]const u8`, length 5, pointing at the same bytes.
- **`std.mem.sliceTo(ptr, 0)`** does the same for a `[*c]` pointer, and returns
  a `[:0]const u8` — it keeps the sentinel in the type.
- **`{s}` works directly on `[*:0]const u8`** — no conversion needed just to
  print.
- **`strchr` returns a pointer into the middle** of the same buffer, so
  `span` on it gives `llo`. No copying anywhere.

## The three conversion functions

| From | To | Use | Allocates |
|---|---|---|---|
| `[*:0]T` | `[:0]T` | `std.mem.span(p)` | no |
| `[*c]T` / `[*]T` | `[:0]T` | `std.mem.sliceTo(p, 0)` | no |
| `[*]T` + known length | `[]T` | `p[0..n]` | no |
| `[]const u8` (runtime) | `[:0]u8` | `allocSentinel` + `@memcpy` | **yes** |
| `[]const u8` (comptime) | `[:0]const u8` | it already is one | no |
| `ArrayList(u8)` | `[:0]u8` | `toOwnedSliceSentinel(gpa, 0)` | transfers |
| formatted (heap) | `[:0]u8` | `std.fmt.allocPrintSentinel` | **yes** |
| formatted (stack) | `[:0]u8` | `std.fmt.bufPrintSentinel` | no |

```zig
const std = @import("std");

extern fn strlen(s: [*:0]const u8) usize;

pub fn main() void {
    const p: [*:0]const u8 = "sentinel pointer";
    const cp: [*c]const u8 = "c pointer";
    const many: [*]const u8 = "no sentinel in type";

    const a = std.mem.span(p);
    const b = std.mem.sliceTo(cp, 0);
    const d = many[0..5];

    std.debug.print("span    {s:<20} len={d} type={s}\n", .{ a, a.len, @typeName(@TypeOf(a)) });
    std.debug.print("sliceTo {s:<20} len={d} type={s}\n", .{ b, b.len, @typeName(@TypeOf(b)) });
    std.debug.print("p[0..n] {s:<20} len={d} type={s}\n", .{ d, d.len, @typeName(@TypeOf(d)) });
    std.debug.print("strlen on the literal = {d}\n", .{strlen(p)});
}
```

```sh
zig run -lc convert.zig
```

```
span    sentinel pointer     len=16 type=[:0]const u8
sliceTo c pointer            len=9 type=[:0]const u8
p[0..n] no se                len=5 type=*const [5]u8
strlen on the literal = 16
```

Note `p[0..n]` with **comptime-known** bounds gives `*const [5]u8`, a pointer to
an array, not a slice. It still prints with `{s}` and coerces to `[]const u8`
wherever one is wanted.

## `dupeZ` is gone — use `allocSentinel`

Older code and every tutorial reaches for `allocator.dupeZ(u8, s)`. It no longer
exists:

```zig
const std = @import("std");

pub fn main() void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();
    _ = gpa.dupeZ(u8, "x") catch {};
}
```

```sh
zig run dupez.zig
```

```
dupez.zig:7:12: error: no field or member function named 'dupeZ' in 'mem.Allocator'
    _ = gpa.dupeZ(u8, "x") catch {};
        ~~~^~~~~~
/opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/mem/Allocator.zig:1:1: note: struct declared here
//! The standard memory allocation interface.
^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

The replacement is `allocSentinel` plus a copy. Write it once as a helper:

```zig
const std = @import("std");

extern fn strlen(s: [*:0]const u8) usize;
extern fn puts(s: [*:0]const u8) c_int;

fn toC(gpa: std.mem.Allocator, s: []const u8) ![:0]u8 {
    const out = try gpa.allocSentinel(u8, s.len, 0);
    @memcpy(out, s);
    return out;
}

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    var buf: [32]u8 = undefined;
    const runtime = buf[0..5];
    @memcpy(runtime, "world");

    const z = try toC(gpa, runtime);
    defer gpa.free(z);
    std.debug.print("allocSentinel -> {s} len={d} strlen={d} type={s}\n", .{ z, z.len, strlen(z), @typeName(@TypeOf(z)) });
    _ = puts(z);

    const p = try std.fmt.allocPrintSentinel(gpa, "n={d}", .{7}, 0);
    defer gpa.free(p);
    std.debug.print("allocPrintSentinel -> {s} strlen={d}\n", .{ p, strlen(p) });
}
```

```sh
zig run -lc allocs.zig
```

```
allocSentinel -> world len=5 strlen=5 type=[:0]u8
allocPrintSentinel -> n=7 strlen=3
world
```

| Old | 0.17.0 |
|---|---|
| `gpa.dupeZ(u8, s)` | `allocSentinel(u8, s.len, 0)` + `@memcpy` |
| `std.fmt.allocPrintZ(gpa, …)` | `std.fmt.allocPrintSentinel(gpa, fmt, args, 0)` |
| `std.fmt.bufPrintZ(&buf, …)` | `std.fmt.bufPrintSentinel(&buf, fmt, args, 0)` |
| `list.toOwnedSliceSentinel(0)` | `list.toOwnedSliceSentinel(gpa, 0)` |

`allocSentinel(u8, 5, 0)` allocates **6** bytes and gives you a `[:0]u8` of
length 5 — the sentinel is beyond `len`, so `@memcpy(out, s)` with equal lengths
is correct and does not clobber it. `gpa.free(z)` on the `[:0]u8` frees all six;
you never account for the extra byte yourself.

## When does a Zig string need conversion?

Only when its length isn't comptime-known. This is the decision in one table:

| You have | Pass to C directly? |
|---|---|
| `"literal"` | **yes** — it's `[:0]const u8` |
| `[:0]const u8` / `[:0]u8` | **yes** — sentinel is in the type |
| `[]const u8` from `argv`, env, or `std.mem.span` | often yes, but the type won't say so — convert |
| `[]const u8` from a file read, a slice, `ArrayList.items` | **no** — convert |
| `buf[0..n]` with runtime `n` | **no** — convert |

```zig
const std = @import("std");

extern fn strlen(s: [*:0]const u8) usize;

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    std.debug.print("literal          strlen={d}\n", .{strlen("direct")});

    const sentinel: [:0]const u8 = "also direct";
    std.debug.print("[:0]const u8     strlen={d}\n", .{strlen(sentinel)});

    var list: std.ArrayList(u8) = .empty;
    defer list.deinit(gpa);
    try list.appendSlice(gpa, "built at runtime");
    const owned = try list.toOwnedSliceSentinel(gpa, 0);
    defer gpa.free(owned);
    std.debug.print("ArrayList        strlen={d}\n", .{strlen(owned)});

    var stack: [64]u8 = undefined;
    const formatted = try std.fmt.bufPrintSentinel(&stack, "id-{d}", .{99}, 0);
    std.debug.print("bufPrintSentinel strlen={d} text={s}\n", .{ strlen(formatted), formatted });
}
```

```sh
zig run -lc needconv.zig
```

```
literal          strlen=6
[:0]const u8     strlen=11
ArrayList        strlen=16
bufPrintSentinel strlen=5 text=id-99
```

**`std.fmt.bufPrintSentinel` is the no-allocation path** — formats into a stack
buffer and returns `[:0]u8`, ready for C. Reach for it before an allocator when
the upper bound is known. (It is *not* called `bufPrintZ`; that name does not
exist in 0.17 and gives `fmt has no member named 'bufPrintZ'`.)

Attempting to pass a plain slice is a compile error, not a crash:

```zig
const std = @import("std");
extern fn strlen(s: [*:0]const u8) usize;
pub fn main() void {
    var buf: [8]u8 = undefined;
    @memcpy(buf[0..3], "abc");
    const slice: []const u8 = buf[0..3];
    std.debug.print("{d}\n", .{strlen(slice)});
}
```

```sh
zig run -lc badslice.zig
```

```
badslice.zig:7:39: error: expected type '[*:0]const u8', found '[]const u8'
    std.debug.print("{d}\n", .{strlen(slice)});
                                      ^~~~~
badslice.zig:2:21: note: parameter type declared here
extern fn strlen(s: [*:0]const u8) usize;
                    ^~~~~~~~~~~~~
```

The error points at your `extern` declaration, which is the right place to look.
The type system caught the buffer overrun that C would have let you ship.

## Reading a C-owned string

When C returns a string, the question is always *who frees it*. The type does
not tell you — only the documentation does.

```zig
const std = @import("std");
const c = std.c;

extern fn strdup(s: [*:0]const u8) ?[*:0]u8;

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    const borrowed = std.mem.span(c.getenv("SHELL") orelse "none");
    std.debug.print("borrowed (never free)  = {s}\n", .{borrowed});

    const owned = strdup("malloc'd by C") orelse return error.OutOfMemory;
    defer c.free(owned);
    std.debug.print("C-owned (c.free)       = {s}\n", .{std.mem.span(owned)});

    const copied = try gpa.dupe(u8, std.mem.span(owned));
    defer gpa.free(copied);
    std.debug.print("copied into Zig (free) = {s}\n", .{copied});
}
```

```sh
zig run -lc owned.zig
```

```
borrowed (never free)  = /bin/zsh
C-owned (c.free)       = malloc'd by C
copied into Zig (free) = malloc'd by C
```

| C returns | Who frees | Zig side |
|---|---|---|
| a pointer to static/internal data (`getenv`, `strerror`) | **nobody** | `std.mem.span`, use it, don't store it |
| `malloc`'d memory (`strdup`, most `*_new` functions) | **you**, with `c.free` | `defer c.free(p)` |
| memory from the library's own allocator | **you**, with *its* free fn | `defer lib_free(p)` |
| a pointer into a buffer you passed in | you, by owning the buffer | nothing extra |

**Never `gpa.free` something C malloc'd, and never `c.free` something your
allocator made.** They are different heaps; mixing them corrupts both. If a
C-owned string must outlive the C call, `gpa.dupe` it — that is what the third
line above does.

`strerror`'s result is the classic trap: it points into libc's static storage
and the next `strerror` call overwrites it. Copy it immediately or format it
immediately.

## Embedded NULs

A Zig slice can contain `\0`. A C string cannot. Converting one loses data
**silently**:

```zig
const std = @import("std");

extern fn strlen(s: [*:0]const u8) usize;

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    const zig_str = "a\x00b";
    std.debug.print("zig len     = {d}\n", .{zig_str.len});

    const z = try gpa.allocSentinel(u8, zig_str.len, 0);
    defer gpa.free(z);
    @memcpy(z, zig_str);
    std.debug.print("zig slice   = {d} bytes\n", .{z.len});
    std.debug.print("C sees      = {d} bytes\n", .{strlen(z)});
    std.debug.print("round trip  = {d} bytes\n", .{std.mem.span(@as([*:0]const u8, z)).len});

    std.debug.print("has NUL?    = {}\n", .{std.mem.indexOfScalar(u8, zig_str, 0) != null});
}
```

```sh
zig run -lc nul.zig
```

```
zig len     = 3
zig slice   = 3 bytes
C sees      = 1 bytes
round trip  = 1 bytes
has NUL?    = true
```

**Three bytes in, one byte out.** `allocSentinel` succeeded, `@memcpy` copied
all three bytes, and the data is intact in Zig's view — but `strlen` stops at
index 1 and the round trip through `span` gives you back `"a"`. No error, no
panic, no warning.

Where this actually bites: file paths, user input, network data, and anything
decoded from a binary format. If a filename could contain a NUL, passing it to
`open()` silently opens a *different* file.

```zig
const std = @import("std");

fn toCChecked(gpa: std.mem.Allocator, s: []const u8) ![:0]u8 {
    if (std.mem.indexOfScalar(u8, s, 0) != null) return error.EmbeddedNul;
    const out = try gpa.allocSentinel(u8, s.len, 0);
    @memcpy(out, s);
    return out;
}

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    const ok = try toCChecked(gpa, "clean/path");
    defer gpa.free(ok);
    std.debug.print("clean  -> {s}\n", .{ok});

    std.debug.print("dirty  -> {!s}\n", .{toCChecked(gpa, "bad\x00path")});
}
```

```sh
zig run toc.zig
```

```
clean  -> clean/path
dirty  -> error.EmbeddedNul
```

One `indexOfScalar` turns silent truncation into an error you can handle. Use
the checked version at every boundary where the bytes came from outside your
program.

## Operations summary

| Task | Code |
|---|---|
| literal → C | pass it; already `[:0]const u8` |
| `[:0]u8` → C | pass it |
| `[]const u8` → C (heap) | `allocSentinel(u8, s.len, 0)` + `@memcpy` |
| `[]const u8` → C (stack) | `std.fmt.bufPrintSentinel(&buf, "{s}", .{s}, 0)` |
| formatted → C (heap) | `std.fmt.allocPrintSentinel(gpa, fmt, args, 0)` |
| built up → C | `ArrayList(u8)` + `toOwnedSliceSentinel(gpa, 0)` |
| `[*:0]T` → slice | `std.mem.span(p)` |
| `[*c]T` → slice | `std.mem.sliceTo(p, 0)` |
| `[*]T` + len → slice | `p[0..n]` |
| C string → owned Zig copy | `gpa.dupe(u8, std.mem.span(p))` |
| C string length | `std.mem.span(p).len` (or `strlen`, same scan) |
| compare with a Zig string | `std.mem.eql(u8, std.mem.span(p), "x")` |
| reject embedded NUL | `std.mem.indexOfScalar(u8, s, 0) != null` |
| free a C-malloc'd string | `std.c.free(p)` |

---

Next: [11.4 — Linking Libraries](./04-linking.md)
