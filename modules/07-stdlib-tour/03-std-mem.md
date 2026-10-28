# 7.3 — std.mem

`std.mem` is where the methods would be if Zig had methods. Everything that
operates on a slice lives here: compare, search, copy, split, join, trim, and
reinterpret raw bytes.

All of it is generic over the element type. `mem.eql(u8, …)` compares strings;
`mem.eql(i32, …)` compares integer slices with the same code.

## Operations table

| Operation | Code | Allocates |
|---|---|---|
| equal | `mem.eql(T, a, b)` → `bool` | no |
| three-way compare | `mem.order(T, a, b)` → `.lt`/`.eq`/`.gt` | no |
| sort comparator | `mem.lessThan(T, a, b)` → `bool` | no |
| first differing index | `mem.indexOfDiff(T, a, b)` → `?usize` | no |
| prefix / suffix | `mem.startsWith(T, s, x)` / `endsWith` | no |
| find subslice | `mem.indexOf(T, s, sub)` → `?usize` | no |
| find from the end | `mem.lastIndexOf(T, s, sub)` | no |
| find one element | `mem.indexOfScalar(T, s, x)` | no |
| find any of a set | `mem.indexOfAny(T, s, set)` | no |
| find from an offset | `mem.indexOfPos(T, s, start, sub)` | no |
| occurrences | `mem.count(T, s, sub)` → `usize` | no |
| at least n of | `mem.containsAtLeast(T, s, n, sub)` | no |
| max / min value | `mem.max(T, s)` / `mem.min(T, s)` → `T` | no |
| index of max / min | `mem.indexOfMax(T, s)` / `indexOfMin` | no |
| copy, low to high | `mem.copyForwards(T, dst, src)` | no |
| copy, high to low | `mem.copyBackwards(T, dst, src)` | no |
| copy, no overlap | `@memcpy(dst, src)` | no |
| fill | `@memset(dst, value)` | no |
| swap two values | `mem.swap(T, &a, &b)` | no |
| reverse in place | `mem.reverse(T, s)` | no |
| rotate left by n | `mem.rotate(T, s, n)` | no |
| trim both ends | `mem.trim(T, s, set)` | no |
| trim one end | `mem.trimStart` / `mem.trimEnd` | no |
| split, keep empties | `mem.splitScalar(T, s, x)` | no |
| split on a subslice | `mem.splitSequence(T, s, sub)` | no |
| split on a set | `mem.splitAny(T, s, set)` | no |
| split, skip empties | `mem.tokenizeScalar` / `tokenizeAny` / `tokenizeSequence` | no |
| split from the end | `mem.splitBackwardsScalar(T, s, x)` | no |
| join with separator | `try mem.join(gpa, sep, slices)` | **yes** |
| join, nul-terminated | `try mem.joinZ(gpa, sep, slices)` | **yes** |
| concatenate | `try mem.concat(gpa, T, slices)` | **yes** |
| replace → new | `try mem.replaceOwned(T, gpa, s, from, to)` | **yes** |
| replace → buffer | `mem.replace(T, s, from, to, &out)` → count | no |
| replace one element | `mem.replaceScalar(T, s, from, to)` | no |
| copy to the heap | `try gpa.dupe(T, s)` | **yes** |
| zero-initialize | `mem.zeroes(T)` → `T` | no |
| value → bytes | `mem.asBytes(&v)` → `*[N]u8` | no |
| value → byte array | `mem.toBytes(v)` → `[N]u8` | no |
| bytes → value ptr | `mem.bytesAsValue(T, bytes)` → `*T` | no |
| bytes → value copy | `mem.bytesToValue(T, bytes)` → `T` | no |
| bytes → slice of T | `mem.bytesAsSlice(T, bytes)` | no |
| read an int, endian | `mem.readInt(T, bytes, .little)` | no |
| truncate at sentinel | `mem.sliceTo(ptr, 0)` | no |
| C string → slice | `mem.span(c_ptr)` | no |
| C string length | `mem.len(c_ptr)` | no |

> **The `indexOf*` family are aliases now.** In 0.17 the real names are
> `find`, `findLast`, `findScalar`, `findAny`, `findMax`, `findMin`, and
> `mem.zig` ends with lines like `pub const indexOf = find;`. Both spellings
> compile and mean the same thing — but when you grep the stdlib for an
> implementation, search for `find`.

## Compare and search

```zig
const std = @import("std");
const mem = std.mem;

pub fn main() void {
    const a = "apple";
    const b = "apricot";

    std.debug.print("eql(a,a)            = {}\n", .{mem.eql(u8, a, a)});
    std.debug.print("eql(a,b)            = {}\n", .{mem.eql(u8, a, b)});
    std.debug.print("order(a,b)          = {t}\n", .{mem.order(u8, a, b)});
    std.debug.print("order(b,a)          = {t}\n", .{mem.order(u8, b, a)});
    std.debug.print("order(a,a)          = {t}\n", .{mem.order(u8, a, a)});
    std.debug.print("lessThan(a,b)       = {}\n", .{mem.lessThan(u8, a, b)});
    std.debug.print("indexOfDiff(a,b)    = {?d}\n", .{mem.indexOfDiff(u8, a, b)});

    const x = [_]i32{ 1, 2, 3 };
    const y = [_]i32{ 1, 2, 4 };
    std.debug.print("eql on i32 arrays   = {}\n", .{mem.eql(i32, &x, &y)});
    std.debug.print("order on i32 arrays = {t}\n", .{mem.order(i32, &x, &y)});

    std.debug.print("startsWith          = {}\n", .{mem.startsWith(u8, b, "apr")});
    std.debug.print("endsWith            = {}\n", .{mem.endsWith(u8, b, "cot")});
    std.debug.print("indexOf \"ric\"       = {?d}\n", .{mem.indexOf(u8, b, "ric")});
    std.debug.print("indexOf \"zzz\"       = {?d}\n", .{mem.indexOf(u8, b, "zzz")});
    std.debug.print("lastIndexOf \"o\"     = {?d}\n", .{mem.lastIndexOf(u8, b, "o")});
    std.debug.print("indexOfScalar 'c'   = {?d}\n", .{mem.indexOfScalar(u8, b, 'c')});
    std.debug.print("count \"p\"           = {d}\n", .{mem.count(u8, b, "p")});
}
```

```
eql(a,a)            = true
eql(a,b)            = false
order(a,b)          = lt
order(b,a)          = gt
order(a,a)          = eq
lessThan(a,b)       = true
indexOfDiff(a,b)    = 2
eql on i32 arrays   = false
order on i32 arrays = lt
startsWith          = true
endsWith            = true
indexOf "ric"       = 2
indexOf "zzz"       = null
lastIndexOf "o"     = 5
indexOfScalar 'c'   = 4
count "p"           = 1
```

| Function | Returns | Use for |
|---|---|---|
| `eql` | `bool` | `==`, which doesn't compile on slices |
| `order` | `.lt` / `.eq` / `.gt` | three-way branching in a `switch` |
| `lessThan` | `bool` | passing to `std.mem.sort` |
| `indexOfDiff` | `?usize` | diffing — `null` means identical |

`order` returns an enum you can `switch` on exhaustively, which is why it's
better than two comparisons:

```zig
switch (mem.order(u8, a, b)) { .lt => …, .eq => …, .gt => … }
```

## Copying, overlap, and reordering

```zig
const std = @import("std");
const mem = std.mem;

pub fn main() void {
    var dst: [6]u8 = undefined;
    mem.copyForwards(u8, &dst, "abcdef");
    std.debug.print("copyForwards      [{s}]\n", .{dst});

    var ov1: [6]u8 = "abcdef".*;
    mem.copyForwards(u8, ov1[0..4], ov1[2..6]);
    std.debug.print("shift left  (fwd) [{s}]\n", .{ov1});

    var ov2: [6]u8 = "abcdef".*;
    mem.copyBackwards(u8, ov2[2..6], ov2[0..4]);
    std.debug.print("shift right (bwd) [{s}]\n", .{ov2});

    var exact: [6]u8 = undefined;
    @memcpy(&exact, "123456");
    std.debug.print("@memcpy           [{s}]\n", .{exact});

    var filled: [6]u8 = undefined;
    @memset(&filled, '*');
    std.debug.print("@memset           [{s}]\n", .{filled});

    var sw1: i32 = 1;
    var sw2: i32 = 2;
    mem.swap(i32, &sw1, &sw2);
    std.debug.print("swap              {d} {d}\n", .{ sw1, sw2 });

    var rev = [_]u8{ 'a', 'b', 'c', 'd' };
    mem.reverse(u8, &rev);
    std.debug.print("reverse           [{s}]\n", .{rev});

    var rot = [_]u8{ 'a', 'b', 'c', 'd', 'e' };
    mem.rotate(u8, &rot, 2);
    std.debug.print("rotate(2)         [{s}]\n", .{rot});
}
```

```
copyForwards      [abcdef]
shift left  (fwd) [cdefef]
shift right (bwd) [ababcd]
@memcpy           [123456]
@memset           [******]
swap              2 1
reverse           [dcba]
rotate(2)         [cdeab]
```

### Which copy to use

**`@memcpy` requires that the two regions do not overlap.** Zig checks this at
runtime in Debug and **panics**:

```
thread 16630199 panic: @memcpy arguments alias
```

That is what `copyForwards` and `copyBackwards` are for — and the direction
matters:

| Moving data | Use | Why |
|---|---|---|
| no overlap at all | `@memcpy(dst, src)` | fastest, asserts non-overlap |
| **shifting left** (dst before src) | `copyForwards` | writes a slot only after reading it |
| **shifting right** (dst after src) | `copyBackwards` | same, from the other end |
| not sure | match the direction to the shift | the wrong one smears bytes |

`@memcpy` also requires the lengths to match exactly, while
`copyForwards`/`copyBackwards` assert `dst.len >= src.len`.

## Splitting

```zig
const std = @import("std");
const mem = std.mem;

pub fn main() void {
    const line = "GET /a/b HTTP/1.1";

    var sp = mem.splitScalar(u8, "a,b,,c", ',');
    std.debug.print("splitScalar ',':   ", .{});
    while (sp.next()) |p| std.debug.print("[{s}]", .{p});
    std.debug.print("\n", .{});

    var tk = mem.tokenizeScalar(u8, "a,b,,c", ',');
    std.debug.print("tokenizeScalar ',':", .{});
    while (tk.next()) |p| std.debug.print("[{s}]", .{p});
    std.debug.print("\n", .{});

    var sq = mem.splitSequence(u8, "a::b::c", "::");
    std.debug.print("splitSequence '::':", .{});
    while (sq.next()) |p| std.debug.print("[{s}]", .{p});
    std.debug.print("\n", .{});

    var ta = mem.tokenizeAny(u8, "one two\tthree", " \t");
    std.debug.print("tokenizeAny ' \\t':  ", .{});
    while (ta.next()) |p| std.debug.print("[{s}]", .{p});
    std.debug.print("\n", .{});

    var sa = mem.splitAny(u8, "a;b,c", ";,");
    std.debug.print("splitAny ';,':     ", .{});
    while (sa.next()) |p| std.debug.print("[{s}]", .{p});
    std.debug.print("\n", .{});

    var bw = mem.splitBackwardsScalar(u8, "a/b/c", '/');
    std.debug.print("splitBackwards '/':", .{});
    while (bw.next()) |p| std.debug.print("[{s}]", .{p});
    std.debug.print("\n", .{});

    var req = mem.splitScalar(u8, line, ' ');
    const method = req.next().?;
    const path = req.next().?;
    const proto = req.rest();
    std.debug.print("method={s} path={s} proto={s}\n", .{ method, path, proto });

    var first = mem.splitScalar(u8, "k=v=w", '=');
    std.debug.print("first={s} rest={s}\n", .{ first.first(), first.rest() });
    std.debug.print("peek after reset:  {?s}\n", .{mem.splitScalar(u8, "x,y", ',').peek()});
}
```

```
splitScalar ',':   [a][b][][c]
tokenizeScalar ',':[a][b][c]
splitSequence '::':[a][b][c]
tokenizeAny ' \t':  [one][two][three]
splitAny ';,':     [a][b][c]
splitBackwards '/':[c][b][a]
method=GET path=/a/b proto=HTTP/1.1
first=k rest=v=w
peek after reset:  x
```

**split keeps empty fields; tokenize skips them.** That one difference decides
which you want:

| Input | Want | Use |
|---|---|---|
| `"a,b,,c"` CSV | `a`, `b`, `""`, `c` — the empty column is data | `splitScalar` |
| `"one  two"` prose | `one`, `two` — runs of spaces are one separator | `tokenizeAny` |

Iterator methods, all zero-allocation:

| Method | Does |
|---|---|
| `.next()` | `?[]const u8`, advances |
| `.peek()` | `?[]const u8`, doesn't advance |
| `.first()` | the first piece, resets to the start |
| `.rest()` | everything not yet consumed — **split once on the first separator** |
| `.reset()` | back to the beginning |

Every piece is a **slice into the input**. Parsing a 100 MB log with
`splitScalar` allocates nothing — and every piece dies when the input does.

## Building: join, concat, replace

```zig
const std = @import("std");
const mem = std.mem;

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    const j = try mem.join(gpa, "-", &.{ "a", "b", "c" });
    defer gpa.free(j);
    std.debug.print("join \"-\"          = \"{s}\"\n", .{j});

    const jz = try mem.joinZ(gpa, "/", &.{ "usr", "local" });
    defer gpa.free(jz);
    std.debug.print("joinZ \"/\"         = \"{s}\" type={s}\n", .{ jz, @typeName(@TypeOf(jz)) });

    const c = try mem.concat(gpa, u8, &.{ "foo", "bar" });
    defer gpa.free(c);
    std.debug.print("concat            = \"{s}\"\n", .{c});

    const ci = try mem.concat(gpa, i32, &.{ &.{ 1, 2 }, &.{ 3, 4 } });
    defer gpa.free(ci);
    std.debug.print("concat i32        = {any}\n", .{ci});

    const r = try mem.replaceOwned(u8, gpa, "a.b.c", ".", "/");
    defer gpa.free(r);
    std.debug.print("replaceOwned      = \"{s}\"\n", .{r});

    const grown = try mem.replaceOwned(u8, gpa, "a.b", ".", "<->");
    defer gpa.free(grown);
    std.debug.print("replaceOwned grow = \"{s}\"\n", .{grown});

    var inplace: [5]u8 = undefined;
    const n = mem.replace(u8, "a.b.c", ".", "/", &inplace);
    std.debug.print("replace           = \"{s}\" ({d} done)\n", .{ inplace, n });

    var sc: [5]u8 = "a.b.c".*;
    mem.replaceScalar(u8, &sc, '.', ':');
    std.debug.print("replaceScalar     = \"{s}\"\n", .{sc});

    const d = try gpa.dupe(u8, "copied");
    defer gpa.free(d);
    std.debug.print("dupe              = \"{s}\"\n", .{d});

    const padded = "  hi  ";
    std.debug.print("trim              = \"{s}\"\n", .{mem.trim(u8, padded, " ")});
    std.debug.print("trimStart         = \"{s}\"\n", .{mem.trimStart(u8, padded, " ")});
    std.debug.print("trimEnd           = \"{s}\"\n", .{mem.trimEnd(u8, padded, " ")});
    std.debug.print("trim whitespace   = \"{s}\"\n", .{mem.trim(u8, "\t x \n", " \t\n\r")});
    std.debug.print("trim a set        = \"{s}\"\n", .{mem.trim(u8, "xxhixx", "x")});
}
```

```
join "-"          = "a-b-c"
joinZ "/"         = "usr/local" type=[:0]u8
concat            = "foobar"
concat i32        = { 1, 2, 3, 4 }
replaceOwned      = "a/b/c"
replaceOwned grow = "a<->b"
replace           = "a/b/c" (2 done)
replaceScalar     = "a:b:c"
dupe              = "copied"
trim              = "hi"
trimStart         = "hi  "
trimEnd           = "  hi"
trim whitespace   = "x"
trim a set        = "hi"
```

- **`concat` is generic**, not a string function — `concat(gpa, i32, …)` works.
- **`replaceOwned` handles a longer replacement** (`"."` → `"<->"` grew the
  string); `replace` writes into a buffer you sized yourself, so it only works
  when the replacement is the same length or shorter.
- **`trim`'s second argument is a SET of bytes, not a prefix.**
  `trim(u8, "xxhixx", "x")` strips every leading and trailing `x`. This is the
  most commonly misread signature in `std.mem`.

> `trimLeft` / `trimRight` were renamed **`trimStart` / `trimEnd`** in 0.17.

## Raw bytes

This is the group you reach for in binary protocols and file formats.

```zig
const std = @import("std");
const mem = std.mem;

const Header = struct {
    magic: u16,
    len: u16,
};

pub fn main() void {
    const zeroed = mem.zeroes(Header);
    std.debug.print("zeroes(Header)     magic={d} len={d}\n", .{ zeroed.magic, zeroed.len });

    const zarr = mem.zeroes([4]u8);
    std.debug.print("zeroes([4]u8)      {any}\n", .{zarr});

    const h = Header{ .magic = 0x5A49, .len = 3 };
    const bytes = mem.asBytes(&h);
    std.debug.print("asBytes            {any} ({d} bytes)\n", .{ bytes.*, bytes.len });

    const back = mem.bytesAsValue(Header, bytes);
    std.debug.print("bytesAsValue       magic=0x{x} len={d}\n", .{ back.magic, back.len });

    var raw = [_]u8{ 0x01, 0x02, 0x03, 0x04 };
    const pair = mem.bytesAsSlice(u16, raw[0..4]);
    std.debug.print("bytesAsSlice u16   {any}\n", .{pair});

    std.debug.print("toBytes            {any}\n", .{mem.toBytes(@as(u16, 0x0102))});
    std.debug.print("bytesToValue u16   0x{x}\n", .{mem.bytesToValue(u16, &[_]u8{ 0x02, 0x01 })});

    std.debug.print("readInt le         0x{x}\n", .{mem.readInt(u16, &[_]u8{ 0x02, 0x01 }, .little)});
    std.debug.print("readInt be         0x{x}\n", .{mem.readInt(u16, &[_]u8{ 0x02, 0x01 }, .big)});

    const c_like = [_]u8{ 'h', 'i', 0, 'x', 'x' };
    std.debug.print("sliceTo(…, 0)      \"{s}\" len={d}\n", .{ mem.sliceTo(&c_like, 0), mem.sliceTo(&c_like, 0).len });

    const cstr: [*:0]const u8 = "from C";
    std.debug.print("span               \"{s}\" len={d}\n", .{ mem.span(cstr), mem.span(cstr).len });
    std.debug.print("len                {d}\n", .{mem.len(cstr)});
}
```

```
zeroes(Header)     magic=0 len=0
zeroes([4]u8)      { 0, 0, 0, 0 }
asBytes            { 73, 90, 3, 0 } (4 bytes)
bytesAsValue       magic=0x5a49 len=3
bytesAsSlice u16   { 513, 1027 }
toBytes            { 2, 1 }
bytesToValue u16   0x102
readInt le         0x102
readInt be         0x201
sliceTo(…, 0)      "hi" len=2
span               "from C" len=6
len                6
```

Read the numbers carefully — they are all about **byte order**:

- `asBytes` on `magic = 0x5A49` gave `{ 73, 90, … }` = `{ 0x49, 0x5A }`. The low
  byte came first because this machine is little-endian. Writing that struct to
  a file produces a file that a big-endian machine misreads.
- `readInt(…, .little)` and `readInt(…, .big)` on the *same two bytes* gave
  `0x102` and `0x201`. **Use `readInt`/`writeInt` with an explicit endianness
  for anything that leaves the process** — a file, a socket, a checksum.
  `bytesToValue` and `@bitCast` use native order, which is only safe in-process.
- `mem.zeroes(T)` is the "all fields zero" constructor. It works on structs,
  arrays and integers, and it's the honest alternative to `undefined` when zero
  is a meaningful default.

| Crossing a boundary | Use |
|---|---|
| in-process, same memory | `asBytes` / `bytesAsValue` / `@bitCast` |
| file, socket, wire format | `readInt` / `writeInt` with `.little` or `.big` |
| C API expecting `char*` | `mem.span(ptr)` / `mem.sliceTo(ptr, 0)` |

`span` needs a sentinel-terminated pointer (`[*:0]const u8`) and finds the
length by scanning; `sliceTo` does the same for a slice or array that happens to
contain a sentinel.

---

Next: [7.4 — Sorting & Search](./04-sorting.md)
