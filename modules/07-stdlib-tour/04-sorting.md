# 7.4 — Sorting & Search

Every sort in Zig takes the same four arguments:

```zig
std.mem.sort(T, items, context, lessThanFn);
//           │  │      │        └─ fn (@TypeOf(context), T, T) bool
//           │  │      └─ anything; `{}` when you don't need it
//           │  └─ a mutable slice, sorted in place
//           └─ the element type
```

There is no `items.sort()`. Sorting an `ArrayList` means sorting `list.items`.

## Operations table

| Operation | Code |
|---|---|
| sort ascending | `std.mem.sort(T, s, {}, std.sort.asc(T))` |
| sort descending | `std.mem.sort(T, s, {}, std.sort.desc(T))` |
| sort, custom | `std.mem.sort(T, s, ctx, myLessThan)` |
| sort, faster, order of equals undefined | `std.mem.sortUnstable(T, s, {}, …)` |
| **stable** algorithm directly | `std.sort.block(T, s, {}, …)` |
| **unstable** algorithm directly | `std.sort.pdq(T, s, {}, …)` |
| tiny slices / nearly sorted | `std.sort.insertion(T, s, {}, …)` |
| no extra memory, no recursion | `std.sort.heap(T, s, {}, …)` |
| already sorted? | `std.sort.isSorted(T, s, {}, …)` → `bool` |
| find in a **sorted** slice | `std.sort.binarySearch(T, s, needle, orderFn)` → `?usize` |
| first index `>=` needle | `std.sort.lowerBound(T, s, needle, orderFn)` → `usize` |
| first index `>` needle | `std.sort.upperBound(T, s, needle, orderFn)` → `usize` |
| range of equal items | `std.sort.equalRange(T, s, needle, orderFn)` → `.{ lo, hi }` |
| index of smallest / largest | `std.sort.argMin` / `argMax` → `?usize` |
| smallest / largest **value** | `std.mem.min(T, s)` / `std.mem.max(T, s)` → `T` |
| index of min / max | `std.mem.indexOfMin` / `indexOfMax` → `usize` |
| both at once | `std.mem.indexOfMinMax(T, s)` → `.{ min_i, max_i }` |
| linear search | `std.mem.indexOfScalar(T, s, x)` → `?usize` |
| reverse | `std.mem.reverse(T, s)` |

> `std.mem.min` / `max` return a **plain `T`**, not an optional — they assert the
> slice is non-empty. Print them with `{d}`. `argMin` / `argMax` *do* return
> `?usize` (`null` for an empty slice), so those take `{?d}`.

## The algorithms

```zig
const std = @import("std");

pub fn main() void {
    var a = [_]i32{ 5, 2, 9, 1, 7 };
    std.mem.sort(i32, &a, {}, std.sort.asc(i32));
    std.debug.print("mem.sort asc        {any}\n", .{a});

    var b = [_]i32{ 5, 2, 9, 1, 7 };
    std.mem.sort(i32, &b, {}, std.sort.desc(i32));
    std.debug.print("mem.sort desc       {any}\n", .{b});

    var c = [_]i32{ 5, 2, 9, 1, 7 };
    std.mem.sortUnstable(i32, &c, {}, std.sort.asc(i32));
    std.debug.print("mem.sortUnstable    {any}\n", .{c});

    var d = [_]u8{ 'd', 'a', 'c', 'b' };
    std.sort.block(u8, &d, {}, std.sort.asc(u8));
    std.debug.print("sort.block (stable) [{s}]\n", .{d});

    var e = [_]u8{ 'd', 'a', 'c', 'b' };
    std.sort.pdq(u8, &e, {}, std.sort.asc(u8));
    std.debug.print("sort.pdq (unstable) [{s}]\n", .{e});

    var f = [_]u8{ 'd', 'a', 'c', 'b' };
    std.sort.insertion(u8, &f, {}, std.sort.asc(u8));
    std.debug.print("sort.insertion      [{s}]\n", .{f});

    var g = [_]u8{ 'd', 'a', 'c', 'b' };
    std.sort.heap(u8, &g, {}, std.sort.asc(u8));
    std.debug.print("sort.heap           [{s}]\n", .{g});

    std.debug.print("isSorted(a)         {}\n", .{std.sort.isSorted(i32, &a, {}, std.sort.asc(i32))});
    std.debug.print("isSorted(b)         {}\n", .{std.sort.isSorted(i32, &b, {}, std.sort.asc(i32))});
}
```

```
mem.sort asc        { 1, 2, 5, 7, 9 }
mem.sort desc       { 9, 7, 5, 2, 1 }
mem.sortUnstable    { 1, 2, 5, 7, 9 }
sort.block (stable) [abcd]
sort.pdq (unstable) [abcd]
sort.insertion      [abcd]
sort.heap           [abcd]
isSorted(a)         true
isSorted(b)         false
```

`std.mem.sort` and `sortUnstable` are thin wrappers — this is literally all they
are:

| Wrapper | Calls | Stable | Extra memory |
|---|---|---|---|
| `std.mem.sort` | `std.sort.block` | **yes** | O(1) — in-place merge |
| `std.mem.sortUnstable` | `std.sort.pdq` | no | O(log n) stack |
| — | `std.sort.insertion` | yes | O(1), O(n²) worst case |
| — | `std.sort.heap` | no | O(1), never recurses |

Default to `std.mem.sort`. Reach for `sortUnstable` when the elements have no
meaningful identity beyond the sort key (plain numbers) and the slice is large.

## Stable vs unstable, actually observed

"Stable" means equal elements keep their original relative order. Tag each item
with its input position, sort by a key with many duplicates, and look at what
happened to the tags:

```zig
const std = @import("std");

const Item = struct { key: u8, seq: u32 };

fn byKey(_: void, a: Item, b: Item) bool {
    return a.key < b.key;
}

fn seqsForKey0(items: []const Item, out: []u32) void {
    var n: usize = 0;
    for (items) |it| {
        if (it.key == 0) {
            out[n] = it.seq;
            n += 1;
        }
    }
}

pub fn main() void {
    var base: [40]Item = undefined;
    for (&base, 0..) |*it, i| it.* = .{ .key = @intCast(i % 2), .seq = @intCast(i) };

    var a = base;
    std.sort.block(Item, &a, {}, byKey);
    var b = base;
    std.sort.pdq(Item, &b, {}, byKey);

    var sa: [20]u32 = undefined;
    var sb: [20]u32 = undefined;
    seqsForKey0(&a, &sa);
    seqsForKey0(&b, &sb);

    std.debug.print("input: 40 items, key = i % 2, seq = i\n", .{});
    std.debug.print("seq order inside key 0 after sorting by key:\n", .{});
    std.debug.print("  block (stable)   {any}\n", .{sa});
    std.debug.print("  pdq   (unstable) {any}\n", .{sb});
    std.debug.print("  same order? {}\n", .{std.mem.eql(u32, &sa, &sb)});
}
```

```
input: 40 items, key = i % 2, seq = i
seq order inside key 0 after sorting by key:
  block (stable)   { 0, 2, 4, 6, 8, 10, 12, 14, 16, 18, 20, 22, 24, 26, 28, 30, 32, 34, 36, 38 }
  pdq   (unstable) { 20, 14, 2, 38, 4, 36, 6, 34, 8, 32, 10, 30, 12, 28, 26, 0, 16, 24, 22, 18 }
  same order? false
```

Both outputs are correctly sorted by `key` — but `pdq` shuffled the ties
arbitrarily. Why it matters: **stability is what makes multi-pass sorting work.**
Sort by salary, then stably sort by department, and you get departments in order
with salaries still ordered inside each one. With `pdq` the first pass is lost.

(With fewer than ~24 elements `pdq` delegates to insertion sort and *happens* to
look stable. Never rely on that — it's why the demo needs 40 items.)

## Custom comparators

A comparator is `fn (context, a, b) bool` returning "does `a` come before `b`".

```zig
const std = @import("std");

const Emp = struct {
    name: []const u8,
    dept: u8,
    salary: u32,
};

fn bySalaryDesc(_: void, a: Emp, b: Emp) bool {
    return a.salary > b.salary;
}

fn byName(_: void, a: Emp, b: Emp) bool {
    return std.mem.lessThan(u8, a.name, b.name);
}

fn byDeptThenSalary(_: void, a: Emp, b: Emp) bool {
    if (a.dept != b.dept) return a.dept < b.dept;
    return a.salary > b.salary;
}

fn show(label: []const u8, items: []const Emp) void {
    std.debug.print("{s}\n", .{label});
    for (items) |e| std.debug.print("  {s:<6} dept={d} salary={d}\n", .{ e.name, e.dept, e.salary });
}

pub fn main() void {
    const base = [_]Emp{
        .{ .name = "ada", .dept = 2, .salary = 120 },
        .{ .name = "bob", .dept = 1, .salary = 90 },
        .{ .name = "cleo", .dept = 2, .salary = 150 },
        .{ .name = "dan", .dept = 1, .salary = 90 },
    };

    var a = base;
    std.mem.sort(Emp, &a, {}, bySalaryDesc);
    show("by salary desc:", &a);

    var b = base;
    std.mem.sort(Emp, &b, {}, byName);
    show("by name:", &b);

    var c = base;
    std.mem.sort(Emp, &c, {}, byDeptThenSalary);
    show("by dept, then salary desc:", &c);
}
```

```
by salary desc:
  cleo   dept=2 salary=150
  ada    dept=2 salary=120
  bob    dept=1 salary=90
  dan    dept=1 salary=90
by name:
  ada    dept=2 salary=120
  bob    dept=1 salary=90
  cleo   dept=2 salary=150
  dan    dept=1 salary=90
by dept, then salary desc:
  bob    dept=1 salary=90
  dan    dept=1 salary=90
  cleo   dept=2 salary=150
  ada    dept=2 salary=120
```

Patterns worth memorizing:

| Want | Body |
|---|---|
| ascending by field | `return a.f < b.f;` |
| descending by field | `return a.f > b.f;` |
| by string field | `return std.mem.lessThan(u8, a.s, b.s);` |
| **multi-key** | `if (a.f != b.f) return a.f < b.f; return a.g > b.g;` |
| by computed value | `return score(a) > score(b);` |

> A comparator must be a **strict** ordering: `lessThan(x, x)` must be `false`.
> Writing `<=` produces an inconsistent order and the sort may read out of
> bounds or loop. Use `<` / `>` only.

## Context: sorting by something outside the elements

The `context` parameter is why Zig needs no closures. It's `{}` above; make it a
value and every comparison can see it.

```zig
const std = @import("std");

fn byDistance(origin: i32, a: i32, b: i32) bool {
    return @abs(a - origin) < @abs(b - origin);
}

fn byPriority(order: []const u8, a: u8, b: u8) bool {
    const ia = std.mem.indexOfScalar(u8, order, a) orelse order.len;
    const ib = std.mem.indexOfScalar(u8, order, b) orelse order.len;
    return ia < ib;
}

fn lenThenAlpha(_: void, a: []const u8, b: []const u8) bool {
    if (a.len != b.len) return a.len < b.len;
    return std.mem.lessThan(u8, a, b);
}

fn printWords(label: []const u8, w: []const []const u8) void {
    std.debug.print("{s}", .{label});
    for (w) |x| std.debug.print(" {s}", .{x});
    std.debug.print("\n", .{});
}

pub fn main() void {
    var nums = [_]i32{ 1, 50, 20, 30, 45 };
    std.mem.sort(i32, &nums, @as(i32, 40), byDistance);
    std.debug.print("closest to 40:    {any}\n", .{nums});

    var nums2 = [_]i32{ 1, 50, 20, 30, 45 };
    std.mem.sort(i32, &nums2, @as(i32, 0), byDistance);
    std.debug.print("closest to 0:     {any}\n", .{nums2});

    var letters = [_]u8{ 'a', 'b', 'c', 'd' };
    std.mem.sort(u8, &letters, @as([]const u8, "dcba"), byPriority);
    std.debug.print("custom priority:  [{s}]\n", .{letters});

    var words = [_][]const u8{ "pear", "fig", "apple", "kiwi" };
    std.mem.sort([]const u8, &words, {}, struct {
        fn lt(_: void, a: []const u8, b: []const u8) bool {
            return std.mem.lessThan(u8, a, b);
        }
    }.lt);
    printWords("alphabetical:    ", &words);

    var words2 = [_][]const u8{ "pear", "fig", "apple", "kiwi" };
    std.mem.sort([]const u8, &words2, {}, lenThenAlpha);
    printWords("by length, then: ", &words2);

    var ci = [_][]const u8{ "Banana", "apple", "Cherry" };
    std.mem.sort([]const u8, &ci, {}, struct {
        fn lt(_: void, a: []const u8, b: []const u8) bool {
            return std.ascii.lessThanIgnoreCase(a, b);
        }
    }.lt);
    printWords("case-insensitive:", &ci);
}
```

```
closest to 40:    { 45, 50, 30, 20, 1 }
closest to 0:     { 1, 20, 30, 45, 50 }
custom priority:  [dcba]
alphabetical:     apple fig kiwi pear
by length, then:  fig kiwi pear apple
case-insensitive: apple Banana Cherry
```

Same comparator, two different origins, two different orders — the context is
how you parameterize a sort. (50 and 30 are both 10 away from 40, so their
relative order is whatever the stable sort preserved from the input.)

### Sorting strings

`[]const u8` as the **element type** means a slice of strings:

```zig
std.mem.sort([]const u8, &words, {}, cmp);
```

and the comparator takes two `[]const u8`. There is no built-in
`std.sort.asc([]const u8)` — `asc` uses `<`, which doesn't work on slices. Write
`std.mem.lessThan(u8, a, b)` for byte order, or
`std.ascii.lessThanIgnoreCase(a, b)` to fold case.

The `struct { fn lt(…) … }.lt` form declares a one-off function inline. It's the
Zig idiom for a comparator you use once.

## Binary search

`binarySearch` requires a **sorted** slice and takes an *order* function, not a
`lessThan`. The signature is unusual: the needle travels in `context`, and the
function compares it against one item at a time.

```zig
fn orderU32(needle: u32, item: u32) std.math.Order {
    return std.math.order(needle, item);
}
```

```zig
const std = @import("std");

fn orderU32(needle: u32, item: u32) std.math.Order {
    return std.math.order(needle, item);
}

fn byLen(needle: usize, item: []const u8) std.math.Order {
    return std.math.order(needle, item.len);
}

pub fn main() void {
    const sorted = [_]u32{ 10, 20, 20, 20, 30, 40 };

    std.debug.print("binarySearch 30   = {?d}\n", .{std.sort.binarySearch(u32, &sorted, @as(u32, 30), orderU32)});
    std.debug.print("binarySearch 35   = {?d}\n", .{std.sort.binarySearch(u32, &sorted, @as(u32, 35), orderU32)});
    std.debug.print("lowerBound 20     = {d}\n", .{std.sort.lowerBound(u32, &sorted, @as(u32, 20), orderU32)});
    std.debug.print("upperBound 20     = {d}\n", .{std.sort.upperBound(u32, &sorted, @as(u32, 20), orderU32)});
    const r = std.sort.equalRange(u32, &sorted, @as(u32, 20), orderU32);
    std.debug.print("equalRange 20     = {d}..{d} ({d} matches)\n", .{ r[0], r[1], r[1] - r[0] });
    std.debug.print("lowerBound 35     = {d} (insert here)\n", .{std.sort.lowerBound(u32, &sorted, @as(u32, 35), orderU32)});

    const words = [_][]const u8{ "a", "bb", "ccc", "dddd" };
    std.debug.print("search by length 3= {?d}\n", .{std.sort.binarySearch([]const u8, &words, @as(usize, 3), byLen)});

    const nums = [_]i32{ 5, 2, 9, 1 };
    std.debug.print("argMin            = {?d}\n", .{std.sort.argMin(i32, &nums, {}, std.sort.asc(i32))});
    std.debug.print("argMax            = {?d}\n", .{std.sort.argMax(i32, &nums, {}, std.sort.asc(i32))});
    std.debug.print("mem.min           = {d}\n", .{std.mem.min(i32, &nums)});
    std.debug.print("mem.max           = {d}\n", .{std.mem.max(i32, &nums)});
    std.debug.print("mem.indexOfMin    = {d}\n", .{std.mem.indexOfMin(i32, &nums)});
    std.debug.print("mem.indexOfMax    = {d}\n", .{std.mem.indexOfMax(i32, &nums)});
    const mm = std.mem.indexOfMinMax(i32, &nums);
    std.debug.print("indexOfMinMax     = min@{d} max@{d}\n", .{ mm[0], mm[1] });
}
```

```
binarySearch 30   = 4
binarySearch 35   = null
lowerBound 20     = 1
upperBound 20     = 4
equalRange 20     = 1..4 (3 matches)
lowerBound 35     = 5 (insert here)
search by length 3= 2
argMin            = 3
argMax            = 2
mem.min           = 1
mem.max           = 9
mem.indexOfMin    = 3
mem.indexOfMax    = 2
indexOfMinMax     = min@3 max@2
```

Reading the duplicate-handling results on `{ 10, 20, 20, 20, 30, 40 }`:

| Call | Result | Means |
|---|---|---|
| `binarySearch(30)` | `4` | the index of 30 |
| `binarySearch(20)` | *some* index in 1..3 | **any** match, not guaranteed the first |
| `lowerBound(20)` | `1` | first index where `item >= 20` |
| `upperBound(20)` | `4` | first index where `item > 20` |
| `equalRange(20)` | `1..4` | the whole run — `hi - lo` = 3 duplicates |
| `lowerBound(35)` | `5` | where 35 **would** be inserted |

So: `binarySearch` for "is it present", `equalRange` for "all the matches",
`lowerBound` for "where does this go" — the insertion point for keeping a slice
sorted.

`byLen` shows the needle and the element need not be the same type: the needle
is a `usize` length, the elements are strings. The slice just has to be sorted
*by the same criterion the order function uses*.

| Searching | Use | Needs sorted |
|---|---|---|
| small slice, one lookup | `std.mem.indexOfScalar` — O(n) | no |
| large slice, many lookups | `std.sort.binarySearch` — O(log n) | **yes** |
| by key, repeatedly | a `HashMap` — O(1) | no |

Sorting to enable one binary search is a loss; sorting once and searching many
times is the win.

---

Next: [7.5 — Formatting](./05-formatting.md)
