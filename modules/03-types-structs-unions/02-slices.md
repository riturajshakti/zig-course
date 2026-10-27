# 3.2 — Slices

A slice is **a pointer plus a length** — 16 bytes on a 64-bit machine, pointing
at memory someone else owns. `[]i32` accepts any length, where `[3]i32` accepts
exactly three. That is the whole reason slices exist.

```zig
const std = @import("std");

pub fn main() void {
    var arr = [_]i32{ 5, 3, 9, 1, 7 };
    const s: []i32 = &arr;
    const mid: []i32 = arr[2..4];

    std.debug.print("@sizeOf([]i32)   = {d} bytes (ptr + len)\n", .{@sizeOf([]i32)});
    std.debug.print("@sizeOf([5]i32)  = {d} bytes\n", .{@sizeOf([5]i32)});
    std.debug.print("s.len={d} mid.len={d}\n", .{ s.len, mid.len });
    std.debug.print("mid.ptr - s.ptr  = {d} elements\n", .{mid.ptr - s.ptr});
    std.debug.print("mid.ptr[0]       = {d}\n", .{mid.ptr[0]});
    std.debug.print("@TypeOf(s.ptr)   = {s}\n", .{@typeName(@TypeOf(s.ptr))});
}
```

```
@sizeOf([]i32)   = 16 bytes (ptr + len)
@sizeOf([5]i32)  = 20 bytes
s.len=5 mid.len=2
mid.ptr - s.ptr  = 2 elements
mid.ptr[0]       = 9
@TypeOf(s.ptr)   = [*]i32
```

| | Array `[5]i32` | Slice `[]i32` |
|---|---|---|
| size of the value | 20 bytes (the data) | 16 bytes (ptr + len) |
| owns its data | yes | **no** — it borrows |
| length known at | compile time | runtime, in `.len` |
| copying it copies | all 5 elements | just ptr + len |
| `.ptr` | n/a | `[*]i32`, a many-item pointer |

`s.len` is a real field here, read at runtime; on an array it was a compile-time
constant.

## Slicing

```zig
const std = @import("std");

pub fn main() void {
    var arr = [_]i32{ 5, 3, 9, 1, 7 };

    const all: []i32 = &arr;
    const mid = arr[1..4];
    const tail = arr[2..];
    const head = arr[0..2];
    const empty = arr[2..2];

    std.debug.print("all   = {any} len={d}\n", .{ all, all.len });
    std.debug.print("mid   = {any} len={d}\n", .{ mid, mid.len });
    std.debug.print("tail  = {any} len={d}\n", .{ tail, tail.len });
    std.debug.print("head  = {any} len={d}\n", .{ head, head.len });
    std.debug.print("empty = {any} len={d}\n", .{ empty, empty.len });

    std.debug.print("@TypeOf(all)  = {s}\n", .{@typeName(@TypeOf(all))});
    std.debug.print("@TypeOf(mid)  = {s}\n", .{@typeName(@TypeOf(mid))});
}
```

```
all   = { 5, 3, 9, 1, 7 } len=5
mid   = { 3, 9, 1 } len=3
tail  = { 9, 1, 7 } len=3
head  = { 5, 3 } len=2
empty = {  } len=0
@TypeOf(all)  = []i32
@TypeOf(mid)  = *[3]i32
```

**Look at the last line.** `arr[1..4]` is *not* a slice — it is `*[3]i32`, a
pointer to a 3-element array. Zig noticed both bounds were compile-time known
and gave you back the length in the type. Use runtime bounds, or annotate, to
get a real slice:

```zig
const std = @import("std");

pub fn main() void {
    var arr = [_]i32{ 5, 3, 9, 1, 7 };
    var i: usize = 1;
    _ = &i;

    const comptime_bounds = arr[1..4];
    const runtime_bounds = arr[i..4];
    const forced: []i32 = arr[1..4];

    std.debug.print("arr[1..4]      -> {s}\n", .{@typeName(@TypeOf(comptime_bounds))});
    std.debug.print("arr[i..4]      -> {s}\n", .{@typeName(@TypeOf(runtime_bounds))});
    std.debug.print("[]i32 = arr[..] -> {s}\n", .{@typeName(@TypeOf(forced))});
    std.debug.print("values: {any} {any} {any}\n", .{ comptime_bounds, runtime_bounds, forced });
}
```

```
arr[1..4]      -> *[3]i32
arr[i..4]      -> []i32
[]i32 = arr[..] -> []i32
values: { 3, 9, 1 } { 3, 9, 1 } { 3, 9, 1 }
```

`*[3]i32` coerces to `[]i32` automatically, so this rarely bites you — except in
`@TypeOf` output and error messages, where it is confusing the first time.

(`_ = &i;` is there because taking the address of `i` is what tells Zig the
value is not comptime-known. Without it, `i` would be folded to a constant.)

## `[]T` vs `[]const T`

The `const` is on the **elements**, not the slice variable.

```zig
const std = @import("std");

fn total(xs: []const i32) i32 {
    var t: i32 = 0;
    for (xs) |x| t += x;
    return t;
}

fn scale(xs: []i32, by: i32) void {
    for (xs) |*x| x.* *= by;
}

pub fn main() void {
    var arr = [_]i32{ 1, 2, 3, 4 };

    std.debug.print("total(&arr)      = {d}\n", .{total(&arr)});
    std.debug.print("total(arr[1..3]) = {d}\n", .{total(arr[1..3])});

    scale(&arr, 10);
    std.debug.print("after scale      = {any}\n", .{arr});

    scale(arr[0..2], 2);
    std.debug.print("after partial    = {any}\n", .{arr});
}
```

```
total(&arr)      = 10
total(arr[1..3]) = 5
after scale      = { 10, 20, 30, 40 }
after partial    = { 20, 40, 30, 40 }
```

`[]i32` coerces to `[]const i32` freely. The reverse never happens — this is a
deliberate error:

```zig
const std = @import("std");

fn scale(xs: []i32, by: i32) void {
    for (xs) |*x| x.* *= by;
}

pub fn main() void {
    const arr = [_]i32{ 1, 2, 3, 4 };
    scale(&arr, 10);
    std.debug.print("{any}\n", .{arr});
}
```

```
s6.zig:9:11: error: expected type '[]i32', found '*const [4]i32'
    scale(&arr, 10);
          ^~~~
s6.zig:9:11: note: cast discards const qualifier
s6.zig:3:14: note: parameter type declared here
fn scale(xs: []i32, by: i32) void {
```

**Rule:** take `[]const T` unless you intend to write. It documents intent in the
signature and the compiler enforces it.

## ⚠️ The dangling-slice trap

**This is the single most common way to corrupt memory in Zig.** A slice borrows;
it does not own. Return a slice into a local and the local dies first.

```zig
const std = @import("std");

fn badLabel(n: i32) []const u8 {
    var buf: [16]u8 = undefined;
    const s = std.fmt.bufPrint(&buf, "item-{d}", .{n}) catch "?";
    std.debug.print("inside badLabel({d}): \"{s}\"\n", .{ n, s });
    return s;
}

pub fn main() void {
    const a = badLabel(1);
    const b = badLabel(2);
    std.debug.print("a = \"{s}\"\n", .{a});
    std.debug.print("b = \"{s}\"\n", .{b});
    std.debug.print("a.ptr == b.ptr : {}\n", .{a.ptr == b.ptr});
}
```

```
inside badLabel(1): "item-1"
inside badLabel(2): "item-2"
a = "������"
b = "������"
a.ptr == b.ptr : true
```

Deliberately broken, and it **compiles without a single warning**. Read the
output carefully:

- Inside the function the string is perfect.
- Outside, both are six `0xAA` bytes — Zig's debug stack-poison pattern. (Your
  terminal shows them as `�` because `0xAA` is not valid UTF-8.)
- `a.ptr == b.ptr` is **`true`**: two supposedly independent strings point at the
  same recycled stack slot. In a bigger program `b` would silently overwrite `a`
  and you would be debugging it for an afternoon.

There is no borrow checker here. Two fixes:

```zig
const std = @import("std");

fn labelInto(buf: []u8, n: i32) ![]const u8 {
    return std.fmt.bufPrint(buf, "item-{d}", .{n});
}

fn labelAlloc(gpa: std.mem.Allocator, n: i32) ![]const u8 {
    return std.fmt.allocPrint(gpa, "item-{d}", .{n});
}

pub fn main(init: std.process.Init) !void {
    const gpa = init.gpa;

    var buf_a: [16]u8 = undefined;
    var buf_b: [16]u8 = undefined;
    const a = try labelInto(&buf_a, 1);
    const b = try labelInto(&buf_b, 2);
    std.debug.print("caller-owned: a=\"{s}\" b=\"{s}\" same_ptr={}\n", .{ a, b, a.ptr == b.ptr });

    const c = try labelAlloc(gpa, 3);
    defer gpa.free(c);
    const d = try labelAlloc(gpa, 4);
    defer gpa.free(d);
    std.debug.print("heap-owned:   c=\"{s}\" d=\"{s}\" same_ptr={}\n", .{ c, d, c.ptr == d.ptr });
}
```

```
caller-owned: a="item-1" b="item-2" same_ptr=false
heap-owned:   c="item-3" d="item-4" same_ptr=false
```

| Fix | Signature | Who frees |
|---|---|---|
| caller passes the buffer | `fn f(buf: []u8, ...) ![]const u8` | nobody — it is the caller's stack |
| function allocates | `fn f(gpa: Allocator, ...) ![]const u8` | the caller, with `gpa.free` |

A function taking an `Allocator` is Zig's way of saying *"the thing I return is
yours to free."* That convention is the whole of Zig's memory discipline.

**The checklist:** a returned slice must point into (1) the caller's buffer,
(2) heap memory the caller will free, or (3) static/comptime data like a string
literal. Never a local.

## Bounds are checked

```zig
const std = @import("std");

pub fn main() void {
    var arr = [_]i32{ 1, 2, 3 };
    const s: []i32 = &arr;
    var i: usize = 5;
    _ = &i;
    std.debug.print("s[5] = {d}\n", .{s[i]});
}
```

```
thread 16421601 panic: index out of bounds: index 5, len 3
s9.zig:8:40: 0x104d9d253 in main (s9)
    std.debug.print("s[5] = {d}\n", .{s[i]});
                                       ^
```

A panic, not silent corruption — in Debug and ReleaseSafe. `ReleaseFast` removes
the check.
(The thread id and the `0x...` addresses differ on every run — only the
message and the source location are stable.)


## Sentinel slices `[:0]T`

A sentinel slice guarantees one extra terminator element past `.len`. This is how
Zig talks to C without a separate type.

```zig
const std = @import("std");

pub fn main() void {
    const lit = "nul-terminated";
    const z: [:0]const u8 = lit;
    const plain: []const u8 = lit;

    std.debug.print("z.len={d} plain.len={d}\n", .{ z.len, plain.len });
    std.debug.print("z[z.len] == 0 : {}\n", .{z[z.len] == 0});
    std.debug.print("@TypeOf(lit)  = {s}\n", .{@typeName(@TypeOf(lit))});
    std.debug.print("@TypeOf(z)    = {s}\n", .{@typeName(@TypeOf(z))});

    var buf: [8:0]u8 = @splat('x');
    buf[0] = 'A';
    std.debug.print("buf={s} buf.len={d} buf[8]={d}\n", .{ buf, buf.len, buf[8] });

    std.debug.print("sliceTo(z,'-')   = {s}\n", .{std.mem.sliceTo(z, '-')});
    std.debug.print("span(c_string)   = {s}\n", .{std.mem.span(@as([*:0]const u8, lit))});
}
```

```
z.len=14 plain.len=14
z[z.len] == 0 : true
@TypeOf(lit)  = *const [14:0]u8
@TypeOf(z)    = [:0]const u8
buf=Axxxxxxx buf.len=8 buf[8]=0
sliceTo(z,'-')   = nul
span(c_string)   = nul-terminated
```

**`.len` excludes the sentinel**, but `z[z.len]` is legal and reads the `0`. That
is the one place indexing at `.len` does not panic.

| Type | Meaning |
|---|---|
| `[]const u8` | 14 bytes, nothing promised after them |
| `[:0]const u8` | 14 bytes **and** a `0` at index 14 |
| `[*:0]const u8` | C's `char*` — no length at all, scan to the `0` |
| `std.mem.span(p)` | `[*:0]T` → `[:0]T` by finding the sentinel |
| `std.mem.sliceTo(s, x)` | truncate at the first `x` |

`[:0]const u8` coerces down to `[]const u8` for free. Going the other way needs a
re-allocation (`gpa.dupeZ`).

## `std.mem` — reading

```zig
const std = @import("std");
const mem = std.mem;

pub fn main() void {
    const arr = [_]i32{ 5, 3, 9, 1, 7 };
    const s: []const i32 = &arr;

    std.debug.print("slice           {any}\n", .{s});
    std.debug.print("len             {d}\n", .{s.len});
    std.debug.print("max             {d}\n", .{mem.max(i32, s)});
    std.debug.print("min             {d}\n", .{mem.min(i32, s)});
    std.debug.print("indexOfMax      {d}\n", .{mem.indexOfMax(i32, s)});
    std.debug.print("indexOfMin      {d}\n", .{mem.indexOfMin(i32, s)});
    std.debug.print("indexOfScalar 9 {?d}\n", .{mem.indexOfScalar(i32, s, 9)});
    std.debug.print("indexOfScalar 4 {?d}\n", .{mem.indexOfScalar(i32, s, 4)});
    std.debug.print("indexOf {{9,1}}   {?d}\n", .{mem.indexOf(i32, s, &[_]i32{ 9, 1 })});
    std.debug.print("startsWith      {}\n", .{mem.startsWith(i32, s, &[_]i32{ 5, 3 })});
    std.debug.print("endsWith        {}\n", .{mem.endsWith(i32, s, &[_]i32{ 1, 7 })});
    std.debug.print("count 7         {d}\n", .{mem.count(i32, s, &[_]i32{7})});
}
```

```
slice           { 5, 3, 9, 1, 7 }
len             5
max             9
min             1
indexOfMax      2
indexOfMin      3
indexOfScalar 9 2
indexOfScalar 4 null
indexOf {9,1}   2
startsWith      true
endsWith        true
count 7         1
```

Two specifier traps in there:

- **`max`/`min` return a plain `i32`, not an optional** — `{d}`, and they
  *assert* the slice is non-empty rather than returning `null`.
- **`indexOf*` return `?usize`** — `{?d}`, and `null` when not found.

## `std.mem` — mutating

```zig
const std = @import("std");
const mem = std.mem;

fn show(label: []const u8, s: []const i32) void {
    std.debug.print("{s:<22} {any}\n", .{ label, s });
}

pub fn main() void {
    var arr = [_]i32{ 5, 3, 9, 1, 7 };
    const s: []i32 = &arr;
    show("start", s);

    mem.sort(i32, s, {}, comptime std.sort.asc(i32));
    show("sort asc", s);

    mem.sort(i32, s, {}, comptime std.sort.desc(i32));
    show("sort desc", s);

    mem.reverse(i32, s);
    show("reverse", s);

    mem.rotate(i32, s, 2);
    show("rotate 2", s);

    mem.swap(i32, &s[0], &s[4]);
    show("swap 0,4", s);

    var dst: [5]i32 = undefined;
    mem.copyForwards(i32, &dst, s);
    show("copyForwards -> dst", &dst);

    @memset(s[1..4], 0);
    show("memset mid to 0", s);
}
```

```
start                  { 5, 3, 9, 1, 7 }
sort asc               { 1, 3, 5, 7, 9 }
sort desc              { 9, 7, 5, 3, 1 }
reverse                { 1, 3, 5, 7, 9 }
rotate 2               { 5, 7, 9, 1, 3 }
swap 0,4               { 3, 7, 9, 1, 5 }
copyForwards -> dst    { 3, 7, 9, 1, 5 }
memset mid to 0        { 3, 0, 0, 0, 5 }
```

`mem.sort` takes `(T, slice, context, lessThanFn)`. With no context you pass `{}`
— the empty `void` value — and `comptime std.sort.asc(i32)` builds the
comparator. The `comptime` keyword is required: the function must be known at
compile time.

Note `s` is `const` while the elements change. The slice (ptr+len) never moves;
what it points at does.

## `std.mem` — allocating

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const gpa = init.gpa;
    const arr = [_]i32{ 5, 3, 9, 1, 7 };

    const owned = try gpa.dupe(i32, &arr);
    defer gpa.free(owned);
    std.mem.sort(i32, owned, {}, comptime std.sort.asc(i32));
    std.debug.print("dupe+sort = {any}  original = {any}\n", .{ owned, arr });

    const fresh = try gpa.alloc(i32, 4);
    defer gpa.free(fresh);
    @memset(fresh, 42);
    std.debug.print("alloc(4)  = {any} len={d}\n", .{ fresh, fresh.len });

    const grown = try gpa.realloc(try gpa.dupe(i32, &arr), 7);
    defer gpa.free(grown);
    @memset(grown[5..], 0);
    std.debug.print("realloc 7 = {any}\n", .{grown});
}
```

```
dupe+sort = { 1, 3, 5, 7, 9 }  original = { 5, 3, 9, 1, 7 }
alloc(4)  = { 42, 42, 42, 42 } len=4
realloc 7 = { 5, 3, 9, 1, 7, 0, 0 }
```

`dupe` is the standard "I need to keep this past the lifetime of the input"
move — and the standard fix for the dangling-slice trap.

## Splitting and windowing

```zig
const std = @import("std");

pub fn main() void {
    const data = [_]i32{ 1, 0, 2, 3, 0, 4 };

    var it = std.mem.splitScalar(i32, &data, 0);
    while (it.next()) |chunk| std.debug.print("chunk={any}\n", .{chunk});

    var w = std.mem.window(i32, &data, 3, 1);
    while (w.next()) |win| std.debug.print("window={any}\n", .{win});
}
```

```
chunk={ 1 }
chunk={ 2, 3 }
chunk={ 4 }
window={ 1, 0, 2 }
window={ 0, 2, 3 }
window={ 2, 3, 0 }
window={ 3, 0, 4 }
```

Both iterators yield **slices into the original** — zero copies, zero
allocations. And zero lifetime safety: they dangle the moment `data` does.

## Operations table

| Operation | Code |
|---|---|
| from an array | `const s: []T = &arr;` |
| whole array | `arr[0..]` |
| range | `arr[a..b]` (end exclusive) |
| open-ended | `arr[2..]` |
| empty | `arr[2..2]` |
| length | `s.len` |
| raw pointer | `s.ptr` → `[*]T` |
| last element | `s[s.len - 1]` |
| index | `s[i]` — panics out of bounds |
| iterate | `for (s) \|v\|` |
| iterate mutably | `for (s) \|*p\| p.* = ...;` |
| compare | `std.mem.eql(T, a, b)` |
| order | `std.mem.order(T, a, b)` → `.lt`/`.eq`/`.gt` |
| find element | `std.mem.indexOfScalar(T, s, x)` → `?usize` |
| find subslice | `std.mem.indexOf(T, s, sub)` → `?usize` |
| find from end | `std.mem.lastIndexOf(T, s, sub)` → `?usize` |
| find any of | `std.mem.indexOfAny(T, s, set)` → `?usize` |
| prefix / suffix | `std.mem.startsWith` / `endsWith` |
| count | `std.mem.count(T, s, sub)` |
| max / min value | `std.mem.max(T, s)` / `min` — plain `T`, asserts non-empty |
| max / min index | `std.mem.indexOfMax(T, s)` / `indexOfMin` |
| sort ascending | `std.mem.sort(T, s, {}, comptime std.sort.asc(T))` |
| sort descending | `... comptime std.sort.desc(T))` |
| reverse | `std.mem.reverse(T, s)` |
| rotate left | `std.mem.rotate(T, s, n)` |
| swap two | `std.mem.swap(T, &s[i], &s[j])` |
| trim ends | `std.mem.trim(T, s, values)` |
| trim one end | `std.mem.trimStart` / `trimEnd` |
| fill | `@memset(s, value)` |
| copy, no overlap | `@memcpy(dst, src)` — lengths must match |
| copy, may overlap | `std.mem.copyForwards` / `copyBackwards` |
| heap copy | `try gpa.dupe(T, s)` → **you free it** |
| heap copy + sentinel | `try gpa.allocSentinel(u8, s.len, 0)` then `@memcpy` → `[:0]u8` |
| allocate | `try gpa.alloc(T, n)` |
| resize | `try gpa.realloc(s, n)` |
| free | `gpa.free(s)` |
| split on element | `std.mem.splitScalar(T, s, delim)` |
| split on subslice | `std.mem.splitSequence(T, s, delim)` |
| skip empties | `std.mem.tokenizeScalar(T, s, delim)` |
| sliding windows | `std.mem.window(T, s, size, step)` |
| truncate at value | `std.mem.sliceTo(s, sentinel)` |

> **Changed in 0.17.0:**
>
> | Old | 0.17.0 |
> |---|---|
> | `std.mem.trimLeft` / `trimRight` | **`trimStart`** / **`trimEnd`** |
> | `gpa.dupeZ(u8, s)` | **gone** — `gpa.allocSentinel(u8, s.len, 0)` + `@memcpy` |
> | `[_]u8{'-'} ** 8` | **gone** — `@splat` (see lesson 3.1) |
>
> Also `indexOf`, `lastIndexOf`, `indexOfMax` and friends are now aliases marked
> *deprecated in favor of* `find`, `findLast`, `findMax`. The old names still
> work today; the `find*` names are where the stdlib is going.

---

Next: [3.3 — Strings](./03-strings.md)
