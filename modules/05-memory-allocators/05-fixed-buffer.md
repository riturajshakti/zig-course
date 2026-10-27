# 5.5 — FixedBuffer

An allocator backed by **memory you already have**. No heap, no syscalls, no
cleanup — and a hard ceiling that returns `error.OutOfMemory` when you hit it.

## The allocations land inside your array

```zig
const std = @import("std");

pub fn main() !void {
    var buf: [256]u8 = undefined;
    var fba: std.heap.FixedBufferAllocator = .init(&buf);
    const alloc = fba.allocator();

    const a = try alloc.alloc(u8, 10);
    const b = try std.fmt.allocPrint(alloc, "n={d}", .{42});
    const c = try alloc.create(u64);
    c.* = 7;

    std.debug.print("&buf   = {*}\n", .{&buf});
    std.debug.print("a      = {*}\n", .{a.ptr});
    std.debug.print("b      = {*} \"{s}\"\n", .{ b.ptr, b });
    std.debug.print("c      = {*} {d}\n", .{ c, c.* });
    std.debug.print("used   = {d} / {d} bytes\n", .{ fba.end_index, buf.len });
}
```

```
&buf   = [256]u8@16b8fa378
a      = u8@16b8fa378
b      = u8@16b8fa382 "n=42"
c      = u64@16b8fa388 7
used   = 24 / 256 bytes
```

**`a` and `&buf` are the same address.** `b` is 10 bytes later, `c` is aligned
up to a multiple of 8. `allocPrint` and `create` ran without touching the heap
once — and `buf` is a stack array, so the address starts `0x16b…` (stack), not
`0x100…` (heap).

No `deinit`, no `free`, nothing to leak. When `buf` goes out of scope the
"allocator" is gone.

| | |
|---|---|
| `.init(&buf)` | takes a `[]u8` — stack array, static array, or heap slice |
| `.allocator()` | the `Allocator`, same as everywhere |
| `fba.end_index` | bytes used so far |
| `fba.reset()` | back to zero, reuse the buffer |
| cleanup | **none** |

## Hitting the ceiling

```zig
const std = @import("std");

pub fn main() !void {
    var buf: [100]u8 = undefined;
    var fba: std.heap.FixedBufferAllocator = .init(&buf);
    const alloc = fba.allocator();

    var n: usize = 0;
    while (true) : (n += 1) {
        _ = alloc.alloc(u8, 30) catch |err| {
            std.debug.print("alloc #{d} of 30 bytes failed: {t}\n", .{ n, err });
            std.debug.print("used {d} / {d}\n", .{ fba.end_index, buf.len });
            break;
        };
        std.debug.print("alloc #{d} ok, used {d} / {d}\n", .{ n, fba.end_index, buf.len });
    }

    fba.reset();
    std.debug.print("after reset: used {d}\n", .{fba.end_index});
    const again = try alloc.alloc(u8, 90);
    std.debug.print("reused the same buffer: {d} bytes\n", .{again.len});
}
```

**Deliberate `error.OutOfMemory`:**

```
alloc #0 ok, used 30 / 100
alloc #1 ok, used 60 / 100
alloc #2 ok, used 90 / 100
alloc #3 of 30 bytes failed: OutOfMemory
used 90 / 100
after reset: used 0
reused the same buffer: 90 bytes
```

Three fit, the fourth doesn't — 10 bytes remain and 30 were asked for. This is
the cheapest way to **test your OOM error paths**: make the buffer small enough
and every `try` on an allocation becomes reachable. No mocking, no fault
injection library.

Notice it doesn't crash or wrap around. `error.OutOfMemory` is just an error,
and `reset()` makes the buffer usable again.

## Freeing is mostly a no-op

Like the arena, a FixedBufferAllocator is a bump pointer — it can only reclaim
the **last** allocation:

```zig
const std = @import("std");

pub fn main() !void {
    var buf: [64]u8 = undefined;
    var fba: std.heap.FixedBufferAllocator = .init(&buf);
    const alloc = fba.allocator();

    const a = try alloc.alloc(u8, 20);
    std.debug.print("a: used={d}\n", .{fba.end_index});
    const b = try alloc.alloc(u8, 20);
    std.debug.print("b: used={d}\n", .{fba.end_index});

    alloc.free(a);
    std.debug.print("free(a) [not last]: used={d}\n", .{fba.end_index});
    alloc.free(b);
    std.debug.print("free(b) [was last]: used={d}\n", .{fba.end_index});
}
```

```
a: used=20
b: used=40
free(a) [not last]: used=40
free(b) [was last]: used=20
```

Freeing `a` while `b` sits above it does nothing. Freeing `b` rewinds by 20.
Don't plan around individual frees — use `reset()` at a natural boundary, or
size the buffer for the peak.

## The caller-provides-buffer API

The most useful thing about a FixedBufferAllocator: it lets a function use
allocator-based stdlib code (`ArrayList`, `allocPrint`, `join`) while
**allocating nothing**:

```zig
const std = @import("std");

fn render(out: []u8, name: []const u8, score: u32) ![]const u8 {
    var fba: std.heap.FixedBufferAllocator = .init(out);
    const alloc = fba.allocator();

    var list: std.ArrayList([]const u8) = .empty;
    try list.append(alloc, try std.fmt.allocPrint(alloc, "name={s}", .{name}));
    try list.append(alloc, try std.fmt.allocPrint(alloc, "score={d}", .{score}));
    try list.append(alloc, "ok=true");
    return std.mem.join(alloc, "&", list.items);
}

pub fn main() !void {
    var scratch: [512]u8 = undefined;
    std.debug.print("{s}\n", .{try render(&scratch, "raj", 99)});
    std.debug.print("{s}\n", .{try render(&scratch, "zig", 17)});

    var tiny: [20]u8 = undefined;
    const r = render(&tiny, "overflow", 1);
    std.debug.print("tiny buffer -> {!s}\n", .{r});
}
```

```
name=raj&score=99&ok=true
name=zig&score=17&ok=true
tiny buffer -> error.OutOfMemory
```

`render` takes a buffer instead of an allocator. The caller owns the memory and
can reuse it (note the same `scratch` serving both calls), nothing is freed, and
an undersized buffer is a clean error instead of a crash. The signature says
everything: **`out: []u8` means "I write into your memory and return a slice of
it."**

The cost is that the result points into `out` — it dies when `out` does, and the
second `render` overwrote the first result. Compare the two styles:

| | `fn f(gpa: Allocator) ![]u8` | `fn f(out: []u8) ![]const u8` |
|---|---|---|
| Who owns the result | caller, must `free` | caller already did |
| Max size | heap | fixed at the call site |
| Overflow | grows | `error.OutOfMemory` |
| Result outlives the call | yes | only while `out` lives |
| Works with no heap at all | no | **yes** |

## The alternatives: `page_allocator` and `smp_allocator`

Two more allocators that need no state variable at all — they're `const`
globals:

```zig
const std = @import("std");

pub fn main() !void {
    std.debug.print("pageSize() = {d}\n", .{std.heap.pageSize()});

    const page = std.heap.page_allocator;
    const p1 = try page.alloc(u8, 1);
    defer page.free(p1);
    const p2 = try page.alloc(u8, 1);
    defer page.free(p2);
    std.debug.print("page_allocator, two 1-byte allocs:\n", .{});
    std.debug.print("  p1 = {*}\n", .{p1.ptr});
    std.debug.print("  p2 = {*}\n", .{p2.ptr});
    std.debug.print("  distance = {d} bytes\n", .{@intFromPtr(p2.ptr) - @intFromPtr(p1.ptr)});

    const smp = std.heap.smp_allocator;
    const s1 = try smp.alloc(u8, 1);
    defer smp.free(s1);
    const s2 = try smp.alloc(u8, 1);
    defer smp.free(s2);
    std.debug.print("smp_allocator, two 1-byte allocs:\n", .{});
    std.debug.print("  s1 = {*}\n", .{s1.ptr});
    std.debug.print("  s2 = {*}\n", .{s2.ptr});
    std.debug.print("  distance = {d} bytes\n", .{@intFromPtr(s2.ptr) - @intFromPtr(s1.ptr)});
}
```

```
pageSize() = 16384
page_allocator, two 1-byte allocs:
  p1 = u8@10119c000
  p2 = u8@1011a0000
  distance = 16384 bytes
smp_allocator, two 1-byte allocs:
  s1 = u8@1011b0000
  s2 = u8@1011b0008
  distance = 8 bytes
```

**`page_allocator` burned 16 KiB per byte.** Every allocation is its own `mmap`
— one syscall each, rounded up to a page (16 KiB on Apple silicon, usually 4 KiB
on x86 Linux). It is not a general-purpose allocator; it is the *source* other
allocators sit on top of. `smp_allocator` packed the two allocations 8 bytes
apart, as a real allocator should.

### Choosing

| Allocator | State needed | Thread-safe | Leak detection | Use for |
|---|---|---|---|---|
| `DebugAllocator(.{})` | `var` + `deinit` | yes | **yes** | development, tests, general use |
| `ArenaAllocator` | `var` + `deinit` | no | n/a | per-request / per-task work |
| `FixedBufferAllocator` | `var`, a buffer | no (use `threadSafeAllocator()`) | n/a | no-heap, embedded, OOM tests |
| `page_allocator` | none | yes | no | backing other allocators; huge blocks |
| `smp_allocator` | none | yes | no | **release builds, multi-threaded** |
| `c_allocator` | none | yes | no | when you link libc and want `malloc` |
| `std.testing.allocator` | none | yes | **yes, fails the test** | inside `test` blocks |

The production pair worth remembering:

```zig
const gpa = if (@import("builtin").mode == .Debug)
    debug_state.allocator()
else
    std.heap.smp_allocator;
```

Develop with DebugAllocator so leaks get caught; ship with `smp_allocator` for
throughput. Lesson 5.3 showed why you can't just leave the DebugAllocator in —
its checks compile away in `ReleaseFast` anyway.

### A buffer that isn't on the stack

`.init` takes any `[]u8`, so you can back a FixedBufferAllocator with one big
heap block and then sub-allocate from it with zero further syscalls:

```zig
const std = @import("std");

pub fn main() !void {
    const block = try std.heap.page_allocator.alloc(u8, 64 * 1024);
    defer std.heap.page_allocator.free(block);

    var fba: std.heap.FixedBufferAllocator = .init(block);
    const alloc = fba.allocator();

    for (0..4) |i| {
        const s = try std.fmt.allocPrint(alloc, "item {d}", .{i});
        std.debug.print("{s} (used {d}/{d})\n", .{ s, fba.end_index, block.len });
    }
}
```

```
item 0 (used 6/65536)
item 1 (used 12/65536)
item 2 (used 18/65536)
item 3 (used 24/65536)
```

One 64 KiB `mmap`, then every allocation is a pointer bump. That's the shape of
most high-performance allocators underneath.

---

Next: [5.6 — Ownership Rules](./06-ownership.md)
