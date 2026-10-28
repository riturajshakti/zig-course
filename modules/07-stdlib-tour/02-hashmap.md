# 7.2 — HashMap

Two names cover almost everything:

| Type | Keys | Hashing |
|---|---|---|
| `std.StringHashMap(V)` | `[]const u8` | content-hashed, compared with `mem.eql` |
| `std.AutoHashMap(K, V)` | ints, enums, structs, pointers… | derived from the type automatically |

**`AutoHashMap` will not take `[]const u8`.** It would hash the pointer, not the
bytes, so two equal strings at different addresses would be different keys. Zig
makes you pick the right one by giving them separate names.

## The asymmetry that will trip you

`ArrayList` went unmanaged in 0.17. **`HashMap` did not.** It takes the
allocator once, at `.init`, and stores it.

```zig
var list: std.ArrayList(u32) = .empty;        // no allocator
defer list.deinit(gpa);                       // allocator per call

var map: std.StringHashMap(u32) = .init(gpa); // allocator here
defer map.deinit();                           // and never again
```

Mixing these up is the most common 0.17 compile error in container code. If you
want the unmanaged form, it has its own name: `StringHashMapUnmanaged` /
`AutoHashMapUnmanaged`, with `.empty` and an allocator on every method.

```zig
const std = @import("std");

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    var ages: std.StringHashMap(u32) = .init(gpa);
    defer ages.deinit();

    try ages.put("alice", 30);
    try ages.put("bob", 25);

    std.debug.print("get(alice)   = {?d}\n", .{ages.get("alice")});
    std.debug.print("get(nobody)  = {?d}\n", .{ages.get("nobody")});
    std.debug.print("count()      = {d}\n", .{ages.count()});
    std.debug.print("capacity()   = {d}\n", .{ages.capacity()});
    std.debug.print("contains(bob)= {}\n", .{ages.contains("bob")});
}
```

```
get(alice)   = 30
get(nobody)  = null
count()      = 2
capacity()   = 8
contains(bob)= true
```

`get` returns `?V` — a missing key is `null`, never a zero value or an exception.
Print it with `{?d}` / `{?s}`.

## Operations table

| Operation | Code | Notes |
|---|---|---|
| create | `var m: std.StringHashMap(V) = .init(gpa);` | managed |
| create unmanaged | `var m: std.StringHashMapUnmanaged(V) = .empty;` | allocator per call |
| insert / overwrite | `try m.put(k, v)` | silently replaces |
| insert, must be new | `try m.putNoClobber(k, v)` | **panics** on a duplicate |
| insert, no alloc | `m.putAssumeCapacity(k, v)` | after `ensureTotalCapacity` |
| replace, return old | `try m.fetchPut(k, v)` → `?KV` | |
| read | `m.get(k)` → `?V` | copies the value |
| read a pointer | `m.getPtr(k)` → `?*V` | **update in place** |
| read the key | `m.getKey(k)` → `?K` | the *stored* key |
| read the pair | `m.getEntry(k)` → `?Entry` | |
| find-or-insert | `const g = try m.getOrPut(k)` | one lookup, not two |
| find-or-default | `try m.getOrPutValue(k, dflt)` → `Entry` | |
| update | `if (m.getPtr(k)) \|p\| p.* = v;` | |
| exists | `m.contains(k)` → `bool` | |
| size | `m.count()` | |
| slots allocated | `m.capacity()` | |
| remove | `m.remove(k)` → `bool` | `false` if absent |
| remove, return pair | `m.fetchRemove(k)` → `?KV` | `.key` and `.value` |
| remove by key pointer | `m.removeByPtr(key_ptr)` | from an iterator |
| iterate pairs | `var it = m.iterator();` → `e.key_ptr.*`, `e.value_ptr.*` | |
| iterate keys | `var it = m.keyIterator();` → `k.*` | |
| iterate values | `var it = m.valueIterator();` → `v.*` | |
| empty, keep memory | `m.clearRetainingCapacity()` | |
| empty, free memory | `m.clearAndFree()` | |
| reserve | `try m.ensureTotalCapacity(n)` | |
| reserve n more | `try m.ensureUnusedCapacity(n)` | |
| copy | `var c = try m.clone();` | shallow |
| free | `m.deinit()` | **frees neither keys nor values** |

There is no `keys()` or `values()` method that hands you a slice — that's an
`ArrayHashMap` feature. On a `HashMap` you iterate (see "Sorted output" below).

## Insert and update

```zig
const std = @import("std");

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    var m: std.StringHashMap(u32) = .init(gpa);
    defer m.deinit();

    try m.put("a", 1);
    std.debug.print("put a=1              get(a)={?d}\n", .{m.get("a")});

    try m.put("a", 2);
    std.debug.print("put a=2 (overwrite)  get(a)={?d} count={d}\n", .{ m.get("a"), m.count() });

    const old = try m.fetchPut("a", 3);
    std.debug.print("fetchPut a=3         old={?d} new={?d}\n", .{ if (old) |kv| kv.value else null, m.get("a") });

    try m.putNoClobber("c", 20);
    std.debug.print("putNoClobber c=20    get(c)={?d}\n", .{m.get("c")});

    if (m.getPtr("c")) |p| p.* += 5;
    std.debug.print("getPtr(c).* += 5     get(c)={?d}\n", .{m.get("c")});

    const gop = try m.getOrPut("d");
    if (!gop.found_existing) gop.value_ptr.* = 100;
    std.debug.print("getOrPut d (new)     found={} value={d}\n", .{ gop.found_existing, gop.value_ptr.* });

    const gop2 = try m.getOrPut("d");
    if (!gop2.found_existing) gop2.value_ptr.* = 999 else gop2.value_ptr.* += 1;
    std.debug.print("getOrPut d (again)   found={} value={d}\n", .{ gop2.found_existing, gop2.value_ptr.* });

    const e = try m.getOrPutValue("z", 7);
    std.debug.print("getOrPutValue z=7    value={d}\n", .{e.value_ptr.*});
    std.debug.print("count                {d}\n", .{m.count()});
}
```

```
put a=1              get(a)=1
put a=2 (overwrite)  get(a)=2 count=1
fetchPut a=3         old=2 new=3
putNoClobber c=20    get(c)=20
getPtr(c).* += 5     get(c)=25
getOrPut d (new)     found=false value=100
getOrPut d (again)   found=true value=101
getOrPutValue z=7    value=7
count                4
```

### `getOrPut` is the one to learn

The obvious counter increment does **two** hash lookups:

```zig
const n = m.get(k) orelse 0;
try m.put(k, n + 1);
```

`getOrPut` does one. It returns a struct with three fields:

| Field | Meaning |
|---|---|
| `found_existing` | `bool` — was the key already there |
| `value_ptr` | `*V` — pointer to the slot, **uninitialized if new** |
| `key_ptr` | `*K` — pointer to the stored key |

```zig
const gop = try m.getOrPut(k);
if (gop.found_existing) gop.value_ptr.* += 1 else gop.value_ptr.* = 1;
```

**You must write `value_ptr.*` when `found_existing` is false.** The slot is
reserved but holds garbage, and the entry already counts toward `count()`.
Reading it without writing is undefined behavior, not a zero.

> ⚠️ **`putNoClobber` panics on a duplicate key** — it does not return an error.
> Internally it is `assert(!self.contains(key))`, so a duplicate aborts the
> process with `reached unreachable code`. Its error union is only
> `error{OutOfMemory}`. Use `getOrPut` and check `found_existing` if a duplicate
> is a case you need to *handle*.

## Remove and clear

```zig
const std = @import("std");

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    var m: std.StringHashMap(u32) = .init(gpa);
    defer m.deinit();
    try m.put("a", 1);
    try m.put("b", 2);
    try m.put("c", 3);
    std.debug.print("count            = {d}\n", .{m.count()});

    std.debug.print("remove(b)        = {}\n", .{m.remove("b")});
    std.debug.print("remove(zzz)      = {}\n", .{m.remove("zzz")});
    std.debug.print("count            = {d}\n", .{m.count()});

    if (m.fetchRemove("c")) |kv| {
        std.debug.print("fetchRemove(c)   = {s} -> {d}\n", .{ kv.key, kv.value });
    }
    std.debug.print("fetchRemove(zzz) = {}\n", .{m.fetchRemove("zzz") == null});
    std.debug.print("count            = {d}\n", .{m.count()});

    try m.put("x", 10);
    try m.put("y", 20);
    m.clearRetainingCapacity();
    std.debug.print("clearRetaining   count={d} cap={d}\n", .{ m.count(), m.capacity() });

    try m.put("z", 1);
    m.clearAndFree();
    std.debug.print("clearAndFree     count={d} cap={d}\n", .{ m.count(), m.capacity() });
}
```

```
count            = 3
remove(b)        = true
remove(zzz)      = false
count            = 2
fetchRemove(c)   = c -> 3
fetchRemove(zzz) = true
count            = 1
clearRetaining   count=0 cap=8
clearAndFree     count=0 cap=0
```

`remove` tells you whether anything was there; `fetchRemove` hands you the pair
on the way out — which is exactly what you need when the key or value is
heap-allocated and you have to free it.

## Iterating

```zig
const std = @import("std");

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    var m: std.StringHashMap(u32) = .init(gpa);
    defer m.deinit();
    try m.put("one", 1);
    try m.put("two", 2);
    try m.put("three", 3);

    std.debug.print("iterator():\n", .{});
    var it = m.iterator();
    while (it.next()) |e| std.debug.print("  {s} -> {d}\n", .{ e.key_ptr.*, e.value_ptr.* });

    std.debug.print("keyIterator():\n", .{});
    var ki = m.keyIterator();
    while (ki.next()) |k| std.debug.print("  {s}\n", .{k.*});

    std.debug.print("valueIterator():\n", .{});
    var vi = m.valueIterator();
    while (vi.next()) |v| std.debug.print("  {d}\n", .{v.*});

    std.debug.print("mutate via value_ptr:\n", .{});
    var mi = m.iterator();
    while (mi.next()) |e| e.value_ptr.* *= 10;
    var mi2 = m.iterator();
    while (mi2.next()) |e| std.debug.print("  {s} -> {d}\n", .{ e.key_ptr.*, e.value_ptr.* });

    var sum: u32 = 0;
    var si = m.valueIterator();
    while (si.next()) |v| sum += v.*;
    std.debug.print("sum = {d}\n", .{sum});
}
```

```
iterator():
  one -> 1
  three -> 3
  two -> 2
keyIterator():
  one
  three
  two
valueIterator():
  1
  3
  2
mutate via value_ptr:
  one -> 10
  three -> 30
  two -> 20
sum = 60
```

Three things:

- **Every iterator yields pointers**, so you write `e.key_ptr.*`, not `e.key`.
  That is also how you mutate values while iterating — `e.value_ptr.* *= 10`.
- **The order is not insertion order** (`one, three, two`) and is not stable
  across versions, rebuilds, or even inserts. Never depend on it; if you need
  order, use `std.StringArrayHashMap` (insertion-ordered) or sort a copy.
- **Inserting or removing during iteration invalidates the iterator.** Collect
  the keys to change first, then apply the changes.

## ⚠️ The key-lifetime trap: keys are NOT copied

`put` stores the **slice you handed it** — pointer and length. It does not copy
the bytes. If that memory changes or dies, the map is quietly corrupt.

```zig
const std = @import("std");

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    var m: std.StringHashMap(u32) = .init(gpa);
    defer m.deinit();

    var buf: [8]u8 = undefined;
    const key = try std.fmt.bufPrint(&buf, "user{d}", .{1});
    try m.put(key, 42);
    std.debug.print("right after put: {?d}\n", .{m.get("user1")});

    @memcpy(buf[0..5], "zzzzz");
    std.debug.print("after buf overwritten:\n", .{});
    std.debug.print("  get(\"user1\") = {?d}\n", .{m.get("user1")});
    var it = m.iterator();
    while (it.next()) |e| std.debug.print("  stored key is now \"{s}\"\n", .{e.key_ptr.*});
}
```

```
right after put: 42
after buf overwritten:
  get("user1") = null
  stored key is now "zzzzz"
```

No crash, no warning. The entry is still *there* — `count()` is 1 — but it is
now filed under `"zzzzz"` while sitting in the hash bucket for `"user1"`, so it
is unreachable by **either** name. This is the single most common HashMap bug in
Zig, and it happens whenever a key comes from a reused buffer, a parsed line, or
anything you're about to free.

`deinit()` makes the same point from the other side: it frees the map's own
table and **nothing else**. Heap-allocated keys and values are yours to free.

| Where the key lives | Safe? | What to do |
|---|---|---|
| string literal | ✅ | nothing — it lives in the binary |
| a buffer you reuse | ❌ | `gpa.dupe` the key |
| a slice into a file you'll free | ❌ | `gpa.dupe`, or keep the file buffer alive |
| `allocPrint`/`dupe` result | ✅ *if* you free it at `deinit` | own it deliberately |
| you'd rather not think about it | ✅ | `std.BufSet` / `std.BufMap` — they copy |

### The owned-key pattern

```zig
const std = @import("std");

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    var m: std.StringHashMap(u32) = .init(gpa);
    defer {
        var it = m.keyIterator();
        while (it.next()) |k| gpa.free(k.*);
        m.deinit();
    }

    const words = [_][]const u8{ "the", "cat", "the", "mat", "the" };
    for (words) |w| {
        const gop = try m.getOrPut(w);
        if (gop.found_existing) {
            gop.value_ptr.* += 1;
        } else {
            gop.key_ptr.* = try gpa.dupe(u8, w);
            gop.value_ptr.* = 1;
        }
    }

    var it = m.iterator();
    while (it.next()) |e| std.debug.print("  {s} = {d}\n", .{ e.key_ptr.*, e.value_ptr.* });
    std.debug.print("distinct = {d}\n", .{m.count()});
}
```

```
  cat = 1
  the = 3
  mat = 1
distinct = 3
```

This is the canonical Zig word counter, and it is leak-clean. Two details make
it work:

- **`gop.key_ptr.* = try gpa.dupe(u8, w)` only on the new branch.** The lookup
  used the borrowed `w`; the *stored* key is then overwritten with a copy the
  map owns. Duplicating on every word would leak the ones already present.
- **The `defer` block frees keys before `deinit()`.** After `deinit` the
  iterator is gone, so the order is not optional.

## Key types: `AutoHashMap`, sets, `BufSet`

```zig
const std = @import("std");

const Point = struct { x: i32, y: i32 };
const Color = enum { red, green, blue };

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    var ints: std.AutoHashMap(u32, []const u8) = .init(gpa);
    defer ints.deinit();
    try ints.put(1, "one");
    try ints.put(2, "two");
    std.debug.print("AutoHashMap(u32,...)   get(2)={?s}\n", .{ints.get(2)});

    var pts: std.AutoHashMap(Point, u8) = .init(gpa);
    defer pts.deinit();
    try pts.put(.{ .x = 1, .y = 2 }, 'A');
    std.debug.print("key = a struct         get={?c}\n", .{pts.get(.{ .x = 1, .y = 2 })});
    std.debug.print("different struct       get={?c}\n", .{pts.get(.{ .x = 9, .y = 9 })});

    var en: std.AutoHashMap(Color, u32) = .init(gpa);
    defer en.deinit();
    try en.put(.red, 1);
    std.debug.print("key = an enum          get(.red)={?d}\n", .{en.get(.red)});

    var set: std.AutoHashMap(u32, void) = .init(gpa);
    defer set.deinit();
    try set.put(10, {});
    try set.put(20, {});
    try set.put(10, {});
    std.debug.print("AutoHashMap(T, void)   count={d} contains(10)={}\n", .{ set.count(), set.contains(10) });

    var ss: std.BufSet = .init(gpa);
    defer ss.deinit();
    try ss.insert("alpha");
    try ss.insert("alpha");
    std.debug.print("BufSet (copies keys)   count={d} contains={}\n", .{ ss.count(), ss.contains("alpha") });
}
```

```
AutoHashMap(u32,...)   get(2)=two
key = a struct         get=A
different struct       get=null
key = an enum          get(.red)=1
AutoHashMap(T, void)   count=2 contains(10)=true
BufSet (copies keys)   count=1 contains=true
```

- **A struct key works with no extra code** — `AutoHashMap` hashes all its
  fields. It must contain no pointers or slices, or you're back to hashing
  addresses.
- **`HashMap(K, void)` is Zig's hash set.** `{}` is the `void` value. There is no
  separate `HashSet` type.
- **`std.BufSet` and `std.BufMap` duplicate their keys internally** and free them
  in `deinit`. Slower, but they make the lifetime trap impossible. `BufMap`'s
  values are `[]const u8` and are copied too — it's what `std.process` uses for
  environment variables.

## Sorted output, and getting keys out

The usual ending for a map: dump it in a defined order.

```zig
const std = @import("std");

const Entry = struct { key: []const u8, value: u32 };

fn byKey(_: void, a: Entry, b: Entry) bool {
    return std.mem.lessThan(u8, a.key, b.key);
}

fn byValueDesc(_: void, a: Entry, b: Entry) bool {
    return a.value > b.value;
}

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    var m: std.StringHashMap(u32) = .init(gpa);
    defer m.deinit();
    try m.put("pear", 5);
    try m.put("apple", 9);
    try m.put("fig", 2);

    var rows: std.ArrayList(Entry) = try .initCapacity(gpa, m.count());
    defer rows.deinit(gpa);
    var it = m.iterator();
    while (it.next()) |e| rows.appendAssumeCapacity(.{ .key = e.key_ptr.*, .value = e.value_ptr.* });

    std.mem.sort(Entry, rows.items, {}, byKey);
    std.debug.print("sorted by key:\n", .{});
    for (rows.items) |r| std.debug.print("  {s} = {d}\n", .{ r.key, r.value });

    std.mem.sort(Entry, rows.items, {}, byValueDesc);
    std.debug.print("sorted by value desc:\n", .{});
    for (rows.items) |r| std.debug.print("  {s} = {d}\n", .{ r.key, r.value });

    var keys: std.ArrayList([]const u8) = .empty;
    defer keys.deinit(gpa);
    var ki = m.keyIterator();
    while (ki.next()) |k| try keys.append(gpa, k.*);
    std.mem.sort([]const u8, keys.items, {}, struct {
        fn lt(_: void, a: []const u8, b: []const u8) bool {
            return std.mem.lessThan(u8, a, b);
        }
    }.lt);
    std.debug.print("sorted keys:", .{});
    for (keys.items) |k| std.debug.print(" {s}", .{k});
    std.debug.print("\n", .{});
}
```

```
sorted by key:
  apple = 9
  fig = 2
  pear = 5
sorted by value desc:
  apple = 9
  pear = 5
  fig = 2
sorted keys: apple fig pear
```

`initCapacity(gpa, m.count())` sizes the list exactly, so `appendAssumeCapacity`
can't fail. The anonymous `struct { fn lt… }.lt` is the inline-comparator idiom —
lesson 7.4 covers comparators properly.

> `{s}` on a `[][]const u8` is a **compile** error
> (`expected type '[]const u8', found '[][]const u8'`). Loop and print each one.

## Unmanaged, capacity, clone

```zig
const std = @import("std");

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    var m: std.StringHashMapUnmanaged(u32) = .empty;
    defer m.deinit(gpa);
    try m.put(gpa, "a", 1);
    try m.put(gpa, "b", 2);
    std.debug.print("unmanaged: get(a)={?d} count={d}\n", .{ m.get("a"), m.count() });

    var c: std.StringHashMap(u32) = .init(gpa);
    defer c.deinit();
    try c.ensureTotalCapacity(16);
    std.debug.print("ensureTotalCapacity(16) cap={d}\n", .{c.capacity()});
    c.putAssumeCapacity("k", 9);
    std.debug.print("putAssumeCapacity       get(k)={?d}\n", .{c.get("k")});

    var cl = try c.clone();
    defer cl.deinit();
    try cl.put("k", 100);
    std.debug.print("clone: original={?d} copy={?d}\n", .{ c.get("k"), cl.get("k") });
}
```

```
unmanaged: get(a)=1 count=2
ensureTotalCapacity(16) cap=32
putAssumeCapacity       get(k)=9
clone: original=9 copy=100
```

`ensureTotalCapacity(16)` gave 32 slots: the table keeps a maximum load factor
(80% by default) and rounds to a power of two, so asking for room for 16 entries
allocates 32 buckets. `getPtr` results are stable until the next insert that
grows the table — the same invalidation rule as `ArrayList`.

## Which map?

| Need | Type |
|---|---|
| string keys | `std.StringHashMap(V)` |
| int / enum / struct keys | `std.AutoHashMap(K, V)` |
| a set | `std.AutoHashMap(K, void)` |
| **insertion order preserved** | `std.StringArrayHashMap(V)` / `std.AutoArrayHashMap(K, V)` |
| `keys()` / `values()` as slices | the `ArrayHashMap` family |
| keys copied for you | `std.BufSet` / `std.BufMap` |
| embedded in a struct, no stored allocator | `…HashMapUnmanaged` |
| sorted keys | a `HashMap` plus a sort, or a tree from a library |

`ArrayHashMap` stores entries in a flat array alongside the index, which is why
it keeps insertion order and can give you `keys()` and `values()` directly. It's
slightly slower to look up and slower to delete (`swapRemove` scrambles,
`orderedRemove` is O(n)) — a good default when you will iterate more than you
will probe.

---

Next: [7.3 — std.mem](./03-std-mem.md)
