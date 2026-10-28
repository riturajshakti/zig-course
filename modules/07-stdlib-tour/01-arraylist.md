# 7.1 — ArrayList

`std.ArrayList(T)` is a growable array: a `ptr`, a `len` and a `capacity`.

**In 0.17 it is *unmanaged*.** It does not store an allocator. You create it from
the constant `.empty` and hand the allocator to every method that might grow or
shrink it.

```zig
const std = @import("std");

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    var list: std.ArrayList(u32) = .empty;
    defer list.deinit(gpa);

    try list.append(gpa, 10);
    try list.append(gpa, 20);
    try list.append(gpa, 30);

    std.debug.print("items    = {any}\n", .{list.items});
    std.debug.print("len      = {d}\n", .{list.items.len});
    std.debug.print("capacity = {d}\n", .{list.capacity});
    std.debug.print("items[1] = {d}\n", .{list.items[1]});
}
```

```
items    = { 10, 20, 30 }
len      = 3
capacity = 33
items[1] = 20
```

> `capacity` is 33, not 4. The allocator rounds the request up to a size class
> and `ArrayList` keeps whatever it got. Never assume a capacity number.

### Why unmanaged

The old `ArrayList.init(gpa)` stored the allocator in every list. A struct with
twelve lists carried twelve identical copies of the same 16-byte pointer pair,
and nothing stopped you from `deinit`-ing with a *different* allocator than you
allocated with.

0.17 moved the allocator to the call site:

| 0.13–0.15 (everywhere online) | 0.17.0 |
|---|---|
| `var l = std.ArrayList(T).init(gpa);` | `var l: std.ArrayList(T) = .empty;` |
| `try l.append(x)` | `try l.append(gpa, x)` |
| `l.deinit()` | `l.deinit(gpa)` |

The list is now two words plus a length — cheap to embed, and the allocator is
visible in every signature that can allocate. Methods that *cannot* allocate
(`pop`, `orderedRemove`, `clearRetainingCapacity`) still take no allocator, which
is how you tell at a glance which calls can fail.

The managed form still exists as `std.ArrayList(T).Managed` if you want it, but
unmanaged is the default everywhere in the stdlib.

## Operations table

| Operation | Code | Allocates |
|---|---|---|
| create | `var l: std.ArrayList(T) = .empty;` | no |
| create with room | `var l: std.ArrayList(T) = try .initCapacity(gpa, n);` | yes |
| create over a buffer | `var l: std.ArrayList(T) = .initBuffer(&buf);` | **never** |
| append one | `try l.append(gpa, x)` | maybe |
| append many | `try l.appendSlice(gpa, &.{ a, b })` | maybe |
| append, no alloc | `l.appendAssumeCapacity(x)` | no |
| append, fail if full | `try l.appendBounded(x)` | no |
| insert at | `try l.insert(gpa, i, x)` | maybe |
| insert slice at | `try l.insertSlice(gpa, i, items)` | maybe |
| reserve a slot | `const p = try l.addOne(gpa); p.* = x;` | maybe |
| reserve n slots | `const s = try l.addManyAsSlice(gpa, n)` | maybe |
| format & append (`u8` only) | `try l.print(gpa, "{d}", .{n})` | maybe |
| read | `l.items[i]` | no |
| last | `l.getLast()` (panics if empty) / `l.last()` → `?T` | no |
| pointer to last | `l.lastPtr()` → `?*T` | no |
| update | `l.items[i] = x` | no |
| update in a loop | `for (l.items) \|*p\| p.* = …` | no |
| length | `l.items.len` | no |
| capacity | `l.capacity` | no |
| remove, keep order | `l.orderedRemove(i)` → `T`, O(n) | no |
| remove, fast | `l.swapRemove(i)` → `T`, O(1), **scrambles** | no |
| remove last | `l.pop()` → `?T` | no |
| replace a range | `try l.replaceRange(gpa, start, len, new)` | maybe |
| truncate | `l.shrinkRetainingCapacity(n)` | no |
| truncate + release | `l.shrinkAndFree(gpa, n)` | no |
| empty it, keep buffer | `l.clearRetainingCapacity()` | no |
| empty it, free buffer | `l.clearAndFree(gpa)` | frees |
| reserve capacity | `try l.ensureTotalCapacity(gpa, n)` | maybe |
| reserve n more | `try l.ensureUnusedCapacity(gpa, n)` | maybe |
| iterate | `for (l.items) \|v\|` | no |
| iterate with index | `for (l.items, 0..) \|v, i\|` | no |
| search | `std.mem.indexOfScalar(T, l.items, x)` → `?usize` | no |
| contains | `std.mem.indexOfScalar(…) != null` | no |
| max / min | `std.mem.max(T, l.items)` / `min` | no |
| sort | `std.mem.sort(T, l.items, {}, std.sort.asc(T))` | no |
| reverse | `std.mem.reverse(T, l.items)` | no |
| copy | `var c = try l.clone(gpa);` | yes |
| hand over the buffer | `const s = try l.toOwnedSlice(gpa);` | transfers |
| nul-terminated buffer | `try l.toOwnedSliceSentinel(gpa, 0)` | transfers |
| free | `l.deinit(gpa)` | frees |

**`l.items` is a plain slice**, so every `std.mem` and `std.sort` function works
on it directly. There is no `l.get(i)`, no `l.sort()`, no `l.contains(x)` — and
no need for them.

## Adding

```zig
const std = @import("std");

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    var l: std.ArrayList(u32) = .empty;
    defer l.deinit(gpa);

    try l.append(gpa, 1);
    std.debug.print("append(1)            {any}\n", .{l.items});

    try l.appendSlice(gpa, &.{ 2, 3, 4 });
    std.debug.print("appendSlice 2,3,4    {any}\n", .{l.items});

    try l.insert(gpa, 0, 99);
    std.debug.print("insert(0, 99)        {any}\n", .{l.items});

    try l.insertSlice(gpa, 2, &.{ 7, 8 });
    std.debug.print("insertSlice(2, 7,8)  {any}\n", .{l.items});

    const slot = try l.addOne(gpa);
    slot.* = 500;
    std.debug.print("addOne -> 500        {any}\n", .{l.items});

    const many = try l.addManyAsSlice(gpa, 2);
    many[0] = 61;
    many[1] = 62;
    std.debug.print("addManyAsSlice(2)    {any}\n", .{l.items});

    std.debug.print("len                  {d}\n", .{l.items.len});
}
```

```
append(1)            { 1 }
appendSlice 2,3,4    { 1, 2, 3, 4 }
insert(0, 99)        { 99, 1, 2, 3, 4 }
insertSlice(2, 7,8)  { 99, 1, 7, 8, 2, 3, 4 }
addOne -> 500        { 99, 1, 7, 8, 2, 3, 4, 500 }
addManyAsSlice(2)    { 99, 1, 7, 8, 2, 3, 4, 500, 61, 62 }
len                  10
```

`addOne` and `addManyAsSlice` grow the list and hand you **pointers into it** to
fill — no copy. That is how you build elements in place instead of constructing
a temporary and appending it.

## Removing and clearing

```zig
const std = @import("std");

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    var l: std.ArrayList(u8) = .empty;
    defer l.deinit(gpa);
    try l.appendSlice(gpa, "abcdef");
    std.debug.print("start                [{s}]\n", .{l.items});

    const a = l.orderedRemove(1);
    std.debug.print("orderedRemove(1)={c}  [{s}]\n", .{ a, l.items });

    const b = l.swapRemove(0);
    std.debug.print("swapRemove(0)={c}     [{s}]\n", .{ b, l.items });

    const c = l.pop();
    std.debug.print("pop()={?c}            [{s}]\n", .{ c, l.items });

    l.shrinkRetainingCapacity(2);
    std.debug.print("shrinkRetaining(2)   [{s}] cap={d}\n", .{ l.items, l.capacity });

    try l.replaceRange(gpa, 0, 1, "XY");
    std.debug.print("replaceRange(0,1,XY) [{s}]\n", .{l.items});

    l.clearRetainingCapacity();
    std.debug.print("clearRetaining       [{s}] len={d} cap={d}\n", .{ l.items, l.items.len, l.capacity });

    l.clearAndFree(gpa);
    std.debug.print("clearAndFree         [{s}] len={d} cap={d}\n", .{ l.items, l.items.len, l.capacity });

    var empty: std.ArrayList(u8) = .empty;
    defer empty.deinit(gpa);
    std.debug.print("pop() when empty     {?c}\n", .{empty.pop()});
}
```

```
start                [abcdef]
orderedRemove(1)=b  [acdef]
swapRemove(0)=a     [fcde]
pop()=e            [fcd]
shrinkRetaining(2)   [fc] cap=137
replaceRange(0,1,XY) [XYc]
clearRetaining       [] len=0 cap=137
clearAndFree         [] len=0 cap=0
pop() when empty     null
```

Three things worth reading twice:

- **`swapRemove(0)` turned `acdef` into `fcde`** — it moves the *last* element
  into the hole instead of shifting. O(1), and the order is gone. Use it when
  order doesn't matter; `orderedRemove` is O(n) and keeps it.
- **`pop()` returns `?T`**, so an empty list gives `null` rather than panicking.
- **`clearRetainingCapacity` keeps the buffer** (cap stays 137) so refilling
  allocates nothing; `clearAndFree` returns it to the allocator (cap 0). Both
  leave a valid list — `deinit` afterwards is still correct.

## Reading, updating, iterating, searching, sorting

```zig
const std = @import("std");

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    var l: std.ArrayList(i32) = .empty;
    defer l.deinit(gpa);
    try l.appendSlice(gpa, &.{ 40, 10, 30, 20 });

    std.debug.print("items[0]       = {d}\n", .{l.items[0]});
    std.debug.print("getLast()      = {d}\n", .{l.getLast()});
    std.debug.print("last()         = {?d}\n", .{l.last()});
    std.debug.print("len            = {d}\n", .{l.items.len});

    l.items[0] = 99;
    std.debug.print("after items[0]=99  {any}\n", .{l.items});

    l.lastPtr().?.* = -1;
    std.debug.print("after lastPtr      {any}\n", .{l.items});

    for (l.items) |v| std.debug.print("  value {d}\n", .{v});
    for (l.items, 0..) |v, i| std.debug.print("  [{d}] {d}\n", .{ i, v });
    for (l.items) |*p| p.* += 1000;
    std.debug.print("after *p += 1000   {any}\n", .{l.items});

    std.debug.print("indexOfScalar 1010 = {?d}\n", .{std.mem.indexOfScalar(i32, l.items, 1010)});
    std.debug.print("contains 1010      = {}\n", .{std.mem.indexOfScalar(i32, l.items, 1010) != null});
    std.debug.print("max                = {d}\n", .{std.mem.max(i32, l.items)});
    std.debug.print("min                = {d}\n", .{std.mem.min(i32, l.items)});

    std.mem.sort(i32, l.items, {}, std.sort.asc(i32));
    std.debug.print("sort asc           {any}\n", .{l.items});
    std.mem.sort(i32, l.items, {}, std.sort.desc(i32));
    std.debug.print("sort desc          {any}\n", .{l.items});
    std.mem.reverse(i32, l.items);
    std.debug.print("reverse            {any}\n", .{l.items});
}
```

```
items[0]       = 40
getLast()      = 20
last()         = 20
len            = 4
after items[0]=99  { 99, 10, 30, 20 }
after lastPtr      { 99, 10, 30, -1 }
  value 99
  value 10
  value 30
  value -1
  [0] 99
  [1] 10
  [2] 30
  [3] -1
after *p += 1000   { 1099, 1010, 1030, 999 }
indexOfScalar 1010 = 1
contains 1010      = true
max                = 1099
min                = 999
sort asc           { 999, 1010, 1030, 1099 }
sort desc          { 1099, 1030, 1010, 999 }
reverse            { 999, 1010, 1030, 1099 }
```

`for (l.items) |v|` copies each element; `for (l.items) |*p|` gives a pointer you
can write through. Note `std.mem.max` returns a **plain `i32`**, not an optional
— it asserts the slice is non-empty, so it prints with `{d}`.

## Capacity, cloning, and giving the buffer away

```zig
const std = @import("std");

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    var a: std.ArrayList(u32) = .empty;
    defer a.deinit(gpa);
    std.debug.print(".empty              cap={d}\n", .{a.capacity});
    try a.ensureTotalCapacity(gpa, 16);
    std.debug.print("ensureTotalCapacity cap={d}\n", .{a.capacity});
    for (0..5) |i| a.appendAssumeCapacity(@intCast(i));
    std.debug.print("5x appendAssumeCap  {any} cap={d}\n", .{ a.items, a.capacity });

    var b: std.ArrayList(u32) = try .initCapacity(gpa, 4);
    defer b.deinit(gpa);
    std.debug.print("initCapacity(4)     cap={d} len={d}\n", .{ b.capacity, b.items.len });

    var buf: [4]u32 = undefined;
    var c: std.ArrayList(u32) = .initBuffer(&buf);
    try c.appendBounded(7);
    try c.appendBounded(8);
    std.debug.print("initBuffer (no heap) {any}\n", .{c.items});

    var d: std.ArrayList(u32) = .empty;
    defer d.deinit(gpa);
    try d.appendSlice(gpa, &.{ 1, 2, 3 });
    var e = try d.clone(gpa);
    defer e.deinit(gpa);
    e.items[0] = 100;
    std.debug.print("clone: d={any} e={any}\n", .{ d.items, e.items });

    var f: std.ArrayList(u32) = .empty;
    try f.appendSlice(gpa, &.{ 5, 6 });
    const owned = try f.toOwnedSlice(gpa);
    defer gpa.free(owned);
    std.debug.print("toOwnedSlice={any} list now len={d} cap={d}\n", .{ owned, f.items.len, f.capacity });
}
```

```
.empty              cap=0
ensureTotalCapacity cap=56
5x appendAssumeCap  { 0, 1, 2, 3, 4 } cap=56
initCapacity(4)     cap=4 len=0
initBuffer (no heap) { 7, 8 }
clone: d={ 1, 2, 3 } e={ 100, 2, 3 }
toOwnedSlice={ 5, 6 } list now len=0 cap=0
```

Four distinct lifetimes in one program:

- **`ensureTotalCapacity` + `appendAssumeCapacity`** — reserve once, then append
  with no error handling at all. `appendAssumeCapacity` returns `void`.
- **`.initBuffer(&buf)` never touches the heap.** Pair it with `appendBounded`,
  which returns `error.OutOfMemory` when the fixed buffer is full. No allocator,
  no `deinit`.
- **`clone(gpa)` is a deep copy of the buffer** — but only one level deep. For
  `ArrayList([]const u8)` the slices are still shared.
- **`toOwnedSlice(gpa)` transfers ownership.** `f` is reset to length 0,
  capacity 0, so it needs no `deinit` — but the slice now needs `gpa.free`.
  This is the normal way a builder function returns its result.

## ⚠️ Appending invalidates every pointer into the list

The one trap unique to a growable array: when it outgrows its buffer it
allocates a new one and copies. Any pointer or slice you were holding now points
at freed memory.

```zig
const std = @import("std");

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    var l: std.ArrayList(u32) = try .initCapacity(gpa, 2);
    defer l.deinit(gpa);
    try l.append(gpa, 1);
    try l.append(gpa, 2);

    const stale = @intFromPtr(&l.items[0]);
    try l.append(gpa, 3);
    const fresh = @intFromPtr(&l.items[0]);

    std.debug.print("len=2 cap=2, then append ->\n", .{});
    std.debug.print("  same buffer?  {}\n", .{stale == fresh});
    std.debug.print("  new capacity: {d}\n", .{l.capacity});
}
```

```
len=2 cap=2, then append ->
  same buffer?  false
  new capacity: 36
```

Had that program kept a `*u32` instead of an integer and dereferenced it after
the append, it would not print garbage — under `DebugAllocator` the old pages
are unmapped, so it **segfaults**:

```
Segmentation fault at address 0x100800000
```

Rules that keep you safe:

| Do | Don't |
|---|---|
| hold **indices** across appends | hold `*T` or `[]T` across appends |
| `l.items[i]` at the point of use | cache `const p = &l.items[i]` |
| `ensureTotalCapacity` first, then take pointers | take pointers, then append |
| `std.ArrayList([]const u8)` of *slices* | expect element addresses to be stable |

If you need addresses that never move, that's a different data structure —
a `std.SinglyLinkedList`, or an `ArrayList` of pointers to individually
`create`d items.

## A list of structs

```zig
const std = @import("std");

const Task = struct {
    id: u32,
    title: []const u8,
    done: bool,
};

fn byId(_: void, a: Task, b: Task) bool {
    return a.id < b.id;
}

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    var tasks: std.ArrayList(Task) = .empty;
    defer tasks.deinit(gpa);

    try tasks.append(gpa, .{ .id = 3, .title = "ship", .done = false });
    try tasks.append(gpa, .{ .id = 1, .title = "design", .done = true });
    try tasks.append(gpa, .{ .id = 2, .title = "build", .done = false });

    std.mem.sort(Task, tasks.items, {}, byId);
    for (tasks.items) |t| std.debug.print("  #{d} {s} done={}\n", .{ t.id, t.title, t.done });

    for (tasks.items) |*t| {
        if (t.id == 2) t.done = true;
    }

    var open: usize = 0;
    for (tasks.items) |t| {
        if (!t.done) open += 1;
    }
    std.debug.print("open = {d}\n", .{open});

    var found: ?Task = null;
    for (tasks.items) |t| {
        if (std.mem.eql(u8, t.title, "build")) found = t;
    }
    std.debug.print("found build: {?d}\n", .{if (found) |t| t.id else null});
}
```

```
  #1 design done=true
  #2 build done=false
  #3 ship done=false
open = 1
found build: 2
```

`.{ .id = 3, … }` coerces to `Task` because `append`'s parameter type is known
at the call site. The `title` slices point at string literals here — if they
pointed at heap memory, freeing each one would be *your* job, because
`deinit(gpa)` frees only the list's own buffer, never what the elements point at.

## `ArrayList(u8)` is the string builder

`ArrayList(u8)` gets an extra method the other instantiations don't: `print`,
which formats straight into the list.

```zig
const std = @import("std");

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    var sb: std.ArrayList(u8) = .empty;
    defer sb.deinit(gpa);

    try sb.appendSlice(gpa, "items:");
    for (0..4) |i| try sb.print(gpa, " {d}", .{i * i});
    try sb.append(gpa, '.');

    std.debug.print("{s}\n", .{sb.items});
    std.debug.print("len={d}\n", .{sb.items.len});

    const owned = try sb.toOwnedSliceSentinel(gpa, 0);
    defer gpa.free(owned);
    std.debug.print("sentinel: {s} (type {s})\n", .{ owned, @typeName(@TypeOf(owned)) });
}
```

```
items: 0 1 4 9.
len=15
sentinel: items: 0 1 4 9. (type [:0]u8)
```

`print` is `comptime assert(T == u8)` internally — calling it on an
`ArrayList(u32)` is a **compile** error, not a runtime one. `toOwnedSliceSentinel(gpa, 0)`
gives you a `[:0]u8`, which is what C APIs want.

| Building a string | Use |
|---|---|
| known pieces, one shot | `std.mem.concat` / `std.mem.join` |
| one formatted result | `std.fmt.allocPrint` |
| **incremental, in a loop** | `ArrayList(u8)` + `append`/`appendSlice`/`print` |
| fixed upper bound, no heap | `std.fmt.bufPrint` into a stack array |

---

Next: [7.2 — HashMap](./02-hashmap.md)
