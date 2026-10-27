# 5.8 — Common Leaks

Six mistakes that account for nearly every leak in real Zig code. Each one is a
runnable program that leaks, the real report, and the fix. **All the leaks on
this page are deliberate.**

| # | Mistake | Fix |
|---|---|---|
| 1 | overwriting a pointer before freeing it | free the old value first |
| 2 | early `return` past an allocation | `errdefer` |
| 3 | `deinit`ing a container but not its elements | loop, then `deinit` |
| 4 | freeing with the wrong allocator | track which allocator owns what |
| 5 | allocating in a loop, freeing after it | `defer` **inside** the loop body |
| 6 | forgetting `deinit` entirely | `defer` on the line after you create it |

## 1. Overwriting a pointer

```zig
const std = @import("std");

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer {
        const status = gpa_s.deinit();
        std.debug.print("deinit status: {t}\n", .{status});
    }
    const gpa = gpa_s.allocator();

    var first = try gpa.dupe(u8, "first");
    std.debug.print("first  = {s}\n", .{first});
    first = try gpa.dupe(u8, "second");
    std.debug.print("second = {s}\n", .{first});
    gpa.free(first);
}
```

```
first  = first
second = second
error(DebugAllocator): memory address 0x104ec0000 leaked: 
...
/path/h1.zig:11:29: 0x104d7f5eb in main (h1)
    var first = try gpa.dupe(u8, "first");
                            ^
```

The assignment on line 13 dropped the only pointer to the first allocation. The
report names **line 11** — the allocation that is now unreachable. There is one
`free` and two `dupe`s.

```zig
const std = @import("std");

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer {
        const status = gpa_s.deinit();
        std.debug.print("deinit status: {t}\n", .{status});
    }
    const gpa = gpa_s.allocator();

    var first = try gpa.dupe(u8, "first");
    std.debug.print("first  = {s}\n", .{first});
    gpa.free(first);
    first = try gpa.dupe(u8, "second");
    defer gpa.free(first);
    std.debug.print("second = {s}\n", .{first});
}
```

```
first  = first
second = second
deinit status: ok
```

This is the leak hiding in every "update a cached field" method:

```zig
fn setName(self: *Thing, gpa: std.mem.Allocator, name: []const u8) !void {
    gpa.free(self.name);                    // <- the line people forget
    self.name = try gpa.dupe(u8, name);
}
```

## 2. Early return past an allocation

```zig
const std = @import("std");

fn parseAge(gpa: std.mem.Allocator, input: []const u8) !u32 {
    const trimmed = try gpa.dupe(u8, std.mem.trim(u8, input, " "));
    const n = try std.fmt.parseInt(u32, trimmed, 10);
    gpa.free(trimmed);
    return n;
}

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer {
        const status = gpa_s.deinit();
        std.debug.print("deinit status: {t}\n", .{status});
    }
    const gpa = gpa_s.allocator();

    std.debug.print("\"  42 \" -> {!d}\n", .{parseAge(gpa, "  42 ")});
    std.debug.print("\"abc\"  -> {!d}\n", .{parseAge(gpa, "abc")});
}
```

```
"  42 " -> 42
"abc"  -> error.InvalidCharacter
error(DebugAllocator): memory address 0x100600000 leaked: 
...
/path/h3.zig:4:33: 0x1004aebbb in parseAge (h3)
    const trimmed = try gpa.dupe(u8, std.mem.trim(u8, input, " "));
                                ^
```

**The happy path frees correctly, so tests pass.** Only the error path leaks —
`try` on line 5 returned before line 6 ran. This is the most common leak in
production Zig, because the leaking path is the one nobody exercises.

The fix is `defer` here (the memory is scratch, freed on both paths):

```zig
const std = @import("std");

fn parseAge(gpa: std.mem.Allocator, input: []const u8) !u32 {
    const trimmed = try gpa.dupe(u8, std.mem.trim(u8, input, " "));
    defer gpa.free(trimmed);
    return std.fmt.parseInt(u32, trimmed, 10);
}

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer {
        const status = gpa_s.deinit();
        std.debug.print("deinit status: {t}\n", .{status});
    }
    const gpa = gpa_s.allocator();

    std.debug.print("\"  42 \" -> {!d}\n", .{parseAge(gpa, "  42 ")});
    std.debug.print("\"abc\"  -> {!d}\n", .{parseAge(gpa, "abc")});
}
```

```
"  42 " -> 42
"abc"  -> error.InvalidCharacter
deinit status: ok
```

**Rule: put the cleanup on the line immediately after the allocation.** `defer`
if the function frees it on success, `errdefer` if the function returns it (see
5.6). Never in the middle of the function body where a `try` can jump over it.

## 3. `deinit`ing the container but not the elements

```zig
const std = @import("std");

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer {
        const status = gpa_s.deinit();
        std.debug.print("deinit status: {t}\n", .{status});
    }
    const gpa = gpa_s.allocator();

    var list: std.ArrayList([]u8) = .empty;
    defer list.deinit(gpa);

    for (0..3) |i| {
        try list.append(gpa, try std.fmt.allocPrint(gpa, "item {d}", .{i}));
    }
    for (list.items) |it| std.debug.print("{s}\n", .{it});
}
```

```
item 0
item 1
item 2
error(DebugAllocator): memory address 0x104ee0000 leaked: 
...
/path/h4.zig:15:52: 0x104d93637 in main (h4)
        try list.append(gpa, try std.fmt.allocPrint(gpa, "item {d}", .{i}));
```

**Three reports** — one per `allocPrint`. `list.deinit(gpa)` freed the list's
backing array of three `[]u8` *headers*, not the bytes each one points at. The
container owns its storage; it has no idea its elements are owned pointers.

```zig
const std = @import("std");

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer {
        const status = gpa_s.deinit();
        std.debug.print("deinit status: {t}\n", .{status});
    }
    const gpa = gpa_s.allocator();

    var list: std.ArrayList([]u8) = .empty;
    defer {
        for (list.items) |it| gpa.free(it);
        list.deinit(gpa);
    }

    for (0..3) |i| {
        try list.append(gpa, try std.fmt.allocPrint(gpa, "item {d}", .{i}));
    }
    for (list.items) |it| std.debug.print("{s}\n", .{it});
}
```

```
item 0
item 1
item 2
deinit status: ok
```

**Elements first, container second** — reversed, you'd be reading freed memory
to find the pointers. The same applies to `HashMap` when you own the keys or the
values, and to any struct holding allocated fields.

| Element type | Needs the loop |
|---|---|
| `ArrayList(u32)` | no — values, not pointers |
| `ArrayList([]u8)` | **yes**, if the slices are owned |
| `ArrayList([]const u8)` of string literals | no — literals aren't allocated |
| `StringHashMap(V)` with duped keys | **yes**, free the keys |

That third row is the nasty one: `ArrayList([]const u8)` can hold a mix of
literals and allocations, and the type can't tell you which. Keep them separate.

## 4. Freeing with the wrong allocator

```zig
const std = @import("std");

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer {
        const status = gpa_s.deinit();
        std.debug.print("deinit status: {t}\n", .{status});
    }
    const gpa = gpa_s.allocator();

    var arena_s: std.heap.ArenaAllocator = .init(gpa);
    defer arena_s.deinit();
    const arena = arena_s.allocator();

    const from_gpa = try gpa.dupe(u8, "allocated by gpa");
    std.debug.print("{s}\n", .{from_gpa});
    arena.free(from_gpa);
    std.debug.print("freed it with the arena instead\n", .{});
}
```

**Deliberate panic:**

```
allocated by gpa
thread 16465917 panic: attempt to use null value
/opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/heap/ArenaAllocator.zig:615:39: 0x1008a2c4f in free (h6)
    const node = arena.loadFirstNode().?;
                                      ^
/opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/mem/Allocator.zig:165:25: 0x1008f00bb in rawFree (h6)
    return a.vtable.free(a.ptr, memory, alignment, ret_addr);
                        ^
/path/h6.zig:17:15: 0x1009976ef in main (h6)
    arena.free(from_gpa);
              ^
```

Not a leak — a **crash**, because the arena looked for bookkeeping it never
created. With other allocator pairings you can get silent corruption instead,
which is worse.

Nothing in the type system prevents this: every `Allocator` is the same type.
The defences are conventions:

| Defence | How |
|---|---|
| store the allocator next to the data | `Thing { gpa: Allocator, buf: []u8 }` |
| one allocator per scope | don't have `gpa` and `arena` both in scope if you can avoid it |
| name them for their role | `arena` for scratch, `gpa` for owned state |
| arena data is never freed individually | if it came from the arena, don't call `free` at all |

## 5. Allocating in a loop, freeing after it

```zig
const std = @import("std");

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer {
        const status = gpa_s.deinit();
        std.debug.print("deinit status: {t}\n", .{status});
    }
    const gpa = gpa_s.allocator();

    var total: usize = 0;
    {
        for (0..1000) |i| {
            const line = try std.fmt.allocPrint(gpa, "line {d}", .{i});
            total += line.len;
            if (i == 999) gpa.free(line);
        }
    }
    std.debug.print("processed {d} bytes\n", .{total});
}
```

**Deliberate leak** — 1000 allocations, 1 free:

```
processed 7890 bytes
error(DebugAllocator): memory address 0x1022a0000 leaked: 
...
deinit status: leak
```

```sh
zig run loopleak.zig 2>&1 | grep -c "leaked:"
```

```
999
```

**999 leak reports.** This is unbounded growth in a server loop: memory climbs
with request count and the process is OOM-killed hours later.

The fix is one word in the right place:

```zig
const std = @import("std");

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer {
        const status = gpa_s.deinit();
        std.debug.print("deinit status: {t}\n", .{status});
    }
    const gpa = gpa_s.allocator();

    var total: usize = 0;
    for (0..1000) |i| {
        const line = try std.fmt.allocPrint(gpa, "line {d}", .{i});
        defer gpa.free(line);
        total += line.len;
    }
    std.debug.print("processed {d} bytes, no leak\n", .{total});
}
```

```
processed 7890 bytes, no leak
deinit status: ok
```

**`defer` is scoped to the enclosing block, and a loop body is a block** — so
this `defer` runs 1000 times, once per iteration, not once at the end of `main`.
That's the behavior you want, and it's the difference between a leak and a
constant memory profile.

If the per-iteration work is more involved, reach for an arena and
`reset(.retain_capacity)` (lesson 5.4) instead of tracking individual frees.

## 6. Forgetting `deinit` on a container

```zig
const std = @import("std");

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer {
        const status = gpa_s.deinit();
        std.debug.print("deinit status: {t}\n", .{status});
    }
    const gpa = gpa_s.allocator();

    var list: std.ArrayList(u32) = .empty;
    for (0..100) |i| try list.append(gpa, @intCast(i));
    std.debug.print("list has {d} items\n", .{list.items.len});
}
```

**Deliberate leak** — no `list.deinit(gpa)`:

```
list has 100 items
error(DebugAllocator): memory address 0x1026c0000 leaked: 
/opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/array_list.zig:1375:56: 0x102527a0f in ensureTotalCapacityPrecise (h10)
                const new_memory = try gpa.alignedAlloc(T, alignment, new_capacity);
                                                       ^
...
/path/h10.zig:12:37: 0x102af3683 in main (h10)
    for (0..100) |i| try list.append(gpa, @intCast(i));
                                    ^
deinit status: leak
```

One report, not a hundred — the `ArrayList` holds one backing array. The report
points at `append`, since that's where the surviving allocation happened.

The habit that prevents it: **write the `defer` on the line after you create the
container**, before you write anything that uses it.

```zig
var list: std.ArrayList(u32) = .empty;
defer list.deinit(gpa);
```

0.17's unmanaged containers make this easy to forget because `.empty` allocates
nothing — there's no `init(gpa)` call to remind you. `.empty` still needs a
matching `deinit(gpa)`.

## `testing.allocator` catches all of this for free

Inside a `test` block, `std.testing.allocator` is a leak-checking allocator and
**a leak fails the test**:

```zig
const std = @import("std");

fn joinWords(gpa: std.mem.Allocator, words: []const []const u8) ![]u8 {
    var list: std.ArrayList(u8) = .empty;
    errdefer list.deinit(gpa);
    for (words, 0..) |w, i| {
        if (i > 0) try list.append(gpa, '-');
        try list.appendSlice(gpa, w);
    }
    return list.toOwnedSlice(gpa);
}

fn leakyJoin(gpa: std.mem.Allocator, words: []const []const u8) ![]u8 {
    var list: std.ArrayList(u8) = .empty;
    for (words, 0..) |w, i| {
        if (i > 0) try list.append(gpa, '-');
        try list.appendSlice(gpa, w);
    }
    return gpa.dupe(u8, list.items);
}

test "joinWords does not leak" {
    const out = try joinWords(std.testing.allocator, &.{ "a", "b", "c" });
    defer std.testing.allocator.free(out);
    try std.testing.expectEqualStrings("a-b-c", out);
}

test "leakyJoin leaks" {
    const out = try leakyJoin(std.testing.allocator, &.{ "a", "b", "c" });
    defer std.testing.allocator.free(out);
    try std.testing.expectEqualStrings("a-b-c", out);
}
```

```sh
zig test leaks.zig
```

```
1/2 leaks.test.joinWords does not leak...OK
2/2 leaks.test.leakyJoin leaks...OK
[SafeAllocator] (err): leaked [addr: 104f28010, len: 129 (0x81) align: 1] allocated at: 
...
/path/leaks.zig:17:29: 0x102f5f493 in leakyJoin (test)
        try list.appendSlice(gpa, w);
                            ^
/path/leaks.zig:29:30: 0x102f5f87b in test.leakyJoin leaks (test)
    const out = try leakyJoin(std.testing.allocator, &.{ "a", "b", "c" });
                             ^
All 2 tests passed.
1 errors were logged.
1 tests leaked memory.
error: the following test command failed with exit code 1:
```

Read that carefully. **"All 2 tests passed" — and then a non-zero exit code.**
Both assertions were correct; `leakyJoin` returned `"a-b-c"` as promised. It
just never freed the `ArrayList`'s own buffer after `dupe`ing the contents out
of it. The leak is reported *per test*, so you know which one.

(Note the `[SafeAllocator]` prefix: `std.testing.allocator` isn't a
DebugAllocator in 0.17, so the message format differs slightly. The useful part
— the allocation's stack trace — is the same.)

`joinWords` is the correct version of the same function: `toOwnedSlice` hands
the buffer out with no copy, and `errdefer` covers the failure path.

**This is why every allocating function needs a test using
`std.testing.allocator`.** It costs three lines and it catches all six mistakes
on this page, including the error-path leak in #2 — which nothing else will.

## The whole module on one page

| Rule | Lesson |
|---|---|
| Stack by default; heap when the size is runtime-known or it must outlive the scope | 5.1 |
| Never return a pointer or slice into a local | 5.1 |
| If a function allocates, it takes an `Allocator` parameter | 5.2 |
| `alloc`→`free`, `create`→`destroy`, same allocator both times | 5.2 |
| `DebugAllocator` in dev — it names the leaking line | 5.3 |
| Arena for per-request work; `reset(.retain_capacity)` in the loop | 5.4 |
| `FixedBufferAllocator` for no-heap paths and OOM tests | 5.5 |
| Cleanup goes on the line after the allocation | 5.6, 5.8 |
| `defer` if you free it, `errdefer` if you return it | 5.6 |
| `dupe` any borrowed slice you intend to store | 5.6 |
| `deinit` frees elements, *then* the container | 5.6, 5.8 |
| Test every allocating function with `std.testing.allocator` | 5.8 |

---

Next: [Module 06 — Comptime & Generics](../06-comptime-generics/)
