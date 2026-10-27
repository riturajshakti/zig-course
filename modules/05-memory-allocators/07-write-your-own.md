# 5.7 — Write an Allocator

Implementing `std.mem.Allocator` is **four functions and a struct**. Nothing
about it is privileged — `DebugAllocator` and `ArenaAllocator` are ordinary Zig
code doing exactly what you're about to do. This is the lesson that makes
allocators stop feeling like magic.

## The two fields and the four functions

```zig
ptr: *anyopaque,                 // your state, type-erased
vtable: *const VTable,           // four function pointers
```

```zig
pub const VTable = struct {
    alloc:  *const fn (*anyopaque, len: usize, alignment: Alignment, ret_addr: usize) ?[*]u8,
    resize: *const fn (*anyopaque, memory: []u8, alignment: Alignment, new_len: usize, ret_addr: usize) bool,
    remap:  *const fn (*anyopaque, memory: []u8, alignment: Alignment, new_len: usize, ret_addr: usize) ?[*]u8,
    free:   *const fn (*anyopaque, memory: []u8, alignment: Alignment, ret_addr: usize) void,
};
```

| Function | Returns | Failure is | Contract |
|---|---|---|---|
| `alloc` | `?[*]u8` | `null` | `len` bytes at `alignment`, or nothing |
| `resize` | `bool` | `false` | grow/shrink **in place**, never move |
| `remap` | `?[*]u8` | `null` | grow/shrink, **may move** and copy |
| `free` | `void` | can't fail | release it |

**0.17 has four, not three** — `remap` is new, and omitting it is a compile
error. Two more things that will trip you up if you write this from memory:

- `alignment` is a **`std.mem.Alignment`** (an enum), not a `u8` log2 value.
- The callbacks are raw: no `error.OutOfMemory`, no slices returned. Failure is
  `null`/`false`, and you return a many-item pointer `[*]u8`. The nice
  `alloc(T, n) ![]T` API you've been using is a thin wrapper
  `std.mem.Allocator` puts on top.

## A CountingAllocator that forwards everything

Start with a pass-through: it counts, then hands the real work to a backing
allocator.

```zig
const std = @import("std");

const CountingAllocator = struct {
    backing: std.mem.Allocator,
    bytes: usize = 0,
    allocs: usize = 0,

    fn allocator(self: *CountingAllocator) std.mem.Allocator {
        return .{
            .ptr = self,
            .vtable = &.{ .alloc = alloc, .resize = resize, .remap = remap, .free = free },
        };
    }
    fn alloc(ctx: *anyopaque, len: usize, a: std.mem.Alignment, ra: usize) ?[*]u8 {
        const self: *CountingAllocator = @ptrCast(@alignCast(ctx));
        const p = self.backing.rawAlloc(len, a, ra) orelse return null;
        self.bytes += len;
        self.allocs += 1;
        return p;
    }
    fn resize(ctx: *anyopaque, m: []u8, a: std.mem.Alignment, new_len: usize, ra: usize) bool {
        const self: *CountingAllocator = @ptrCast(@alignCast(ctx));
        return self.backing.rawResize(m, a, new_len, ra);
    }
    fn remap(ctx: *anyopaque, m: []u8, a: std.mem.Alignment, new_len: usize, ra: usize) ?[*]u8 {
        const self: *CountingAllocator = @ptrCast(@alignCast(ctx));
        return self.backing.rawRemap(m, a, new_len, ra);
    }
    fn free(ctx: *anyopaque, m: []u8, a: std.mem.Alignment, ra: usize) void {
        const self: *CountingAllocator = @ptrCast(@alignCast(ctx));
        self.backing.rawFree(m, a, ra);
    }
};

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer _ = gpa_s.deinit();

    var counter: CountingAllocator = .{ .backing = gpa_s.allocator() };
    const gpa = counter.allocator();

    const a = try gpa.alloc(u8, 10);
    defer gpa.free(a);
    const b = try gpa.alloc(u8, 20);
    defer gpa.free(b);

    std.debug.print("allocs={d} bytes={d}\n", .{ counter.allocs, counter.bytes });
}
```

```
allocs=2 bytes=30
```

Three mechanisms to understand:

**`.ptr = self`** — assigning a `*CountingAllocator` to a `*anyopaque` field.
This is the type erasure that lets one `Allocator` type represent every
allocator. It is why `Allocator` is 16 bytes regardless of how much state the
implementation keeps.

**`@ptrCast(@alignCast(ctx))`** — the inverse, in every callback. `@alignCast`
restores the alignment information `*anyopaque` lost, then `@ptrCast`
reinterprets the pointer. You will type this line in every allocator you write.
Order matters: `@alignCast` first, inside.

**`rawAlloc`/`rawResize`/`rawRemap`/`rawFree`** — to forward to a backing
allocator you call the `raw*` methods, not `alloc`/`free`. The `raw*` family
takes and returns the same low-level types as the vtable, so forwarding is
one line. Calling `self.backing.alloc(u8, len)` would be wrong — you'd lose the
caller's alignment and get back a slice you can't return.

Note `ra` (`ret_addr`): the caller's return address, which is what
DebugAllocator uses to build those leak stack traces. Pass it through unchanged.

## Building it up: live bytes and peak usage

```zig
const std = @import("std");

const StatsAllocator = struct {
    backing: std.mem.Allocator,
    allocs: usize = 0,
    frees: usize = 0,
    live: usize = 0,
    peak: usize = 0,
    total: usize = 0,

    fn allocator(self: *StatsAllocator) std.mem.Allocator {
        return .{
            .ptr = self,
            .vtable = &.{ .alloc = alloc, .resize = resize, .remap = remap, .free = free },
        };
    }

    fn note(self: *StatsAllocator) void {
        if (self.live > self.peak) self.peak = self.live;
    }

    fn alloc(ctx: *anyopaque, len: usize, a: std.mem.Alignment, ra: usize) ?[*]u8 {
        const self: *StatsAllocator = @ptrCast(@alignCast(ctx));
        const p = self.backing.rawAlloc(len, a, ra) orelse return null;
        self.allocs += 1;
        self.total += len;
        self.live += len;
        self.note();
        return p;
    }

    fn resize(ctx: *anyopaque, m: []u8, a: std.mem.Alignment, new_len: usize, ra: usize) bool {
        const self: *StatsAllocator = @ptrCast(@alignCast(ctx));
        if (!self.backing.rawResize(m, a, new_len, ra)) return false;
        self.live = self.live - m.len + new_len;
        self.note();
        return true;
    }

    fn remap(ctx: *anyopaque, m: []u8, a: std.mem.Alignment, new_len: usize, ra: usize) ?[*]u8 {
        const self: *StatsAllocator = @ptrCast(@alignCast(ctx));
        const p = self.backing.rawRemap(m, a, new_len, ra) orelse return null;
        self.live = self.live - m.len + new_len;
        if (new_len > m.len) self.total += new_len - m.len;
        self.note();
        return p;
    }

    fn free(ctx: *anyopaque, m: []u8, a: std.mem.Alignment, ra: usize) void {
        const self: *StatsAllocator = @ptrCast(@alignCast(ctx));
        self.backing.rawFree(m, a, ra);
        self.frees += 1;
        self.live -= m.len;
    }

    fn report(self: *StatsAllocator) void {
        std.debug.print("allocs={d} frees={d} live={d} peak={d} total={d}\n", .{
            self.allocs, self.frees, self.live, self.peak, self.total,
        });
    }
};

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer _ = gpa_s.deinit();

    var stats: StatsAllocator = .{ .backing = gpa_s.allocator() };
    const gpa = stats.allocator();

    var list: std.ArrayList(u64) = .empty;
    defer list.deinit(gpa);
    for (0..100) |i| try list.append(gpa, i);
    std.debug.print("after 100 appends: ", .{});
    stats.report();

    const s = try std.fmt.allocPrint(gpa, "sum={d}", .{list.items.len});
    std.debug.print("after allocPrint : ", .{});
    stats.report();
    gpa.free(s);
    std.debug.print("after free       : ", .{});
    stats.report();
}
```

```
after 100 appends: allocs=4 frees=3 live=1120 peak=1776 total=2256
after allocPrint : allocs=5 frees=3 live=1127 peak=1776 total=2263
after free       : allocs=5 frees=4 live=1120 peak=1776 total=2263
```

**This is a real profiler in 60 lines**, and it just taught us something about
`ArrayList`: 100 appends cost **4 allocations and 3 frees** — the list grew and
copied three times — with a `peak` of 1776 bytes against 1120 live. That extra
656 bytes is an old array still alive while the new one was being filled.

Wrapping an allocator to observe it is the single most useful custom-allocator
technique. Variations worth writing yourself:

| Wrapper | Change to `alloc` |
|---|---|
| fail after N allocations | `if (self.n >= self.limit) return null;` — test OOM paths |
| log every call | `std.debug.print` before forwarding |
| enforce a byte budget | `if (self.live + len > self.cap) return null;` |
| slow allocator | sleep before forwarding, to expose races |

## A real allocator: bump from a buffer

No backing allocator — this one manages memory itself. It is
`FixedBufferAllocator` with the serial numbers filed off:

```zig
const std = @import("std");

const BumpAllocator = struct {
    buffer: []u8,
    offset: usize = 0,
    high_water: usize = 0,

    fn init(buffer: []u8) BumpAllocator {
        return .{ .buffer = buffer };
    }

    fn allocator(self: *BumpAllocator) std.mem.Allocator {
        return .{
            .ptr = self,
            .vtable = &.{
                .alloc = alloc,
                .resize = std.mem.Allocator.noResize,
                .remap = std.mem.Allocator.noRemap,
                .free = std.mem.Allocator.noFree,
            },
        };
    }

    fn reset(self: *BumpAllocator) void {
        self.offset = 0;
    }

    fn alloc(ctx: *anyopaque, len: usize, a: std.mem.Alignment, ra: usize) ?[*]u8 {
        _ = ra;
        const self: *BumpAllocator = @ptrCast(@alignCast(ctx));
        const base = @intFromPtr(self.buffer.ptr);
        const start = a.forward(base + self.offset) - base;
        const end = start + len;
        if (end > self.buffer.len) return null;
        self.offset = end;
        if (end > self.high_water) self.high_water = end;
        return self.buffer.ptr + start;
    }
};

pub fn main() !void {
    var backing: [256]u8 = undefined;
    var bump: BumpAllocator = .init(&backing);
    const gpa = bump.allocator();

    const a = try gpa.alloc(u8, 3);
    const b = try gpa.create(u64);
    b.* = 99;
    const c = try std.fmt.allocPrint(gpa, "hi {d}", .{b.*});

    std.debug.print("a   offset {d} align {d}\n", .{ @intFromPtr(a.ptr) - @intFromPtr(&backing), @alignOf(u8) });
    std.debug.print("u64 offset {d} align {d} (padded!)\n", .{ @intFromPtr(b) - @intFromPtr(&backing), @alignOf(u64) });
    std.debug.print("c   offset {d} \"{s}\"\n", .{ @intFromPtr(c.ptr) - @intFromPtr(&backing), c });
    std.debug.print("used {d} / {d}\n", .{ bump.offset, backing.len });

    gpa.free(a);
    std.debug.print("after free(a): used {d} (noFree does nothing)\n", .{bump.offset});

    const big = gpa.alloc(u8, 1000);
    std.debug.print("alloc 1000 from 256: {!s}\n", .{big});

    bump.reset();
    std.debug.print("after reset : used {d}, high_water {d}\n", .{ bump.offset, bump.high_water });
}
```

```
a   offset 0 align 1
u64 offset 8 align 8 (padded!)
c   offset 22 "hi 99"
used 27 / 256
after free(a): used 27 (noFree does nothing)
alloc 1000 from 256: error.OutOfMemory
after reset : used 0, high_water 27
```

Three payoffs in that output:

**`a.forward(...)` is how you honour alignment.** `a` was 3 bytes at offset 0,
so the next free byte is offset 3 — but a `u64` needs 8-byte alignment, so the
allocator rounded up and wasted 5 bytes. `Alignment.forward(addr)` rounds an
address up; the whole alignment calculation is that one call. (Alignment is
relative to the *absolute* address, which is why `base` is added and subtracted.)

**`std.mem.Allocator.noResize` / `noRemap` / `noFree` are provided stubs.** When
your allocator genuinely can't do an operation, use these instead of writing
empty functions. `noFree` is why `gpa.free(a)` changed nothing.

**Returning `null` becomes `error.OutOfMemory`** for free. The 1000-byte request
exceeded the buffer, `alloc` returned `null`, and the `Allocator` wrapper turned
it into the error the caller expects. You never write `error.OutOfMemory`
yourself.

## A pool allocator: fixed-size slots with reuse

The classic interview exercise. Same-size objects, a free list, O(1) alloc and
free, and memory reused instead of growing:

```zig
const std = @import("std");

fn PoolAllocator(comptime slot_size: usize, comptime slot_count: usize) type {
    return struct {
        const Self = @This();

        slots: [slot_count][slot_size]u8 align(16) = undefined,
        free_list: [slot_count]bool = @splat(true),
        in_use: usize = 0,
        rejected: usize = 0,

        fn allocator(self: *Self) std.mem.Allocator {
            return .{
                .ptr = self,
                .vtable = &.{
                    .alloc = alloc,
                    .resize = std.mem.Allocator.noResize,
                    .remap = std.mem.Allocator.noRemap,
                    .free = free,
                },
            };
        }

        fn alloc(ctx: *anyopaque, len: usize, a: std.mem.Alignment, ra: usize) ?[*]u8 {
            _ = ra;
            const self: *Self = @ptrCast(@alignCast(ctx));
            if (len > slot_size or a.toByteUnits() > 16) {
                self.rejected += 1;
                return null;
            }
            for (&self.free_list, 0..) |*is_free, i| {
                if (is_free.*) {
                    is_free.* = false;
                    self.in_use += 1;
                    return &self.slots[i];
                }
            }
            self.rejected += 1;
            return null;
        }

        fn free(ctx: *anyopaque, m: []u8, a: std.mem.Alignment, ra: usize) void {
            _ = a;
            _ = ra;
            const self: *Self = @ptrCast(@alignCast(ctx));
            const base = @intFromPtr(&self.slots);
            const i = (@intFromPtr(m.ptr) - base) / slot_size;
            std.debug.assert(!self.free_list[i]);
            self.free_list[i] = true;
            self.in_use -= 1;
        }
    };
}

const Connection = struct { id: u32, active: bool };

pub fn main() !void {
    var pool: PoolAllocator(@sizeOf(Connection), 4) = .{};
    const gpa = pool.allocator();

    var conns: [4]*Connection = undefined;
    for (&conns, 0..) |*c, i| {
        c.* = try gpa.create(Connection);
        c.*.* = .{ .id = @intCast(i), .active = true };
        std.debug.print("created id={d} in_use={d}\n", .{ c.*.id, pool.in_use });
    }

    const fifth = gpa.create(Connection);
    std.debug.print("5th from a 4-slot pool: {!}\n", .{fifth});

    gpa.destroy(conns[1]);
    std.debug.print("destroyed one: in_use={d}\n", .{pool.in_use});

    const reused = try gpa.create(Connection);
    std.debug.print("reused slot? same addr as conns[1]: {}\n", .{reused == conns[1]});
    std.debug.print("in_use={d} rejected={d}\n", .{ pool.in_use, pool.rejected });
}
```

```
created id=0 in_use=1
created id=1 in_use=2
created id=2 in_use=3
created id=3 in_use=4
5th from a 4-slot pool: error.OutOfMemory
destroyed one: in_use=3
reused slot? same addr as conns[1]: true
in_use=4 rejected=1
```

`reused == conns[1]` is `true` — the freed slot came straight back. That's the
property you want for connection handles, particles, ECS components: a bounded
number of same-sized objects, churning constantly, with **no fragmentation and
no syscalls ever**.

Note how the pool handles requests it can't serve: `len > slot_size` or a bigger
alignment than it guarantees → `null`. **An allocator may refuse**; the contract
is only that it returns valid memory *or* nothing.

The free-list-as-`[N]bool` is O(n) on alloc. Production pools store the free
list *inside* the free slots themselves as an intrusive linked list, making both
operations O(1) with zero extra memory. `std.heap.MemoryPool(T)` in the standard
library does exactly that — read it once you've written this one.

## Checklist for your own allocator

| Requirement | Why |
|---|---|
| all four vtable fields set | `remap` is not optional in 0.17 |
| `@ptrCast(@alignCast(ctx))` in every callback | restore your type from `*anyopaque` |
| honour `alignment` with `a.forward(addr)` | a misaligned `u64` is undefined behavior |
| return `null`/`false` on failure, never panic | the wrapper converts it to `error.OutOfMemory` |
| forward with `raw*`, not `alloc`/`free` | the `raw*` family matches the vtable types |
| use `noResize`/`noRemap`/`noFree` stubs | don't hand-write no-ops |
| `allocator()` takes `*Self` | the vtable stores a pointer to your state, so it must not move |

That last one is a real trap: `Allocator.ptr` points at your struct, so if you
copy or move the state after calling `allocator()`, the vtable points at the old
location. Keep the state in a `var` that outlives the `Allocator`, and pass the
`Allocator` around — never the state by value.

---

Next: [5.8 — Common Leaks](./08-common-leaks.md)
