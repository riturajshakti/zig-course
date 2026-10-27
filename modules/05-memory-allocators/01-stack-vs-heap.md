# 5.1 — Stack vs Heap

Node, Python and Go decide this for you. Zig does not: **every allocation is a
line you wrote.** This lesson is about which of the two places a value lives in,
and how to tell.

## Addresses tell you where you are

```zig
const std = @import("std");

pub fn main() !void {
    var n: u64 = 7;
    var arr: [4]u8 = .{ 1, 2, 3, 4 };
    const slice = arr[1..3];

    std.debug.print("&n    = {*}\n", .{&n});
    std.debug.print("&arr  = {*}\n", .{&arr});
    std.debug.print("slice = {*} len={d}\n", .{ slice.ptr, slice.len });

    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer _ = gpa_s.deinit();
    const gpa = gpa_s.allocator();

    const heap = try gpa.alloc(u8, 4);
    defer gpa.free(heap);
    std.debug.print("heap  = {*} len={d}\n", .{ heap.ptr, heap.len });
}
```

```
&n    = u64@16fad6370
&arr  = [4]u8@16fad63ec
slice = u8@16fad63ed len=2
heap  = u8@100580000 len=4
```

Three addresses start `0x16f…`, one starts `0x100…`. The stack lives high and
grows down; the heap lives low. (Exact values shift every run — macOS
randomizes both — but the *two clusters* are always there.)

| | Stack | Heap |
|---|---|---|
| Who manages it | the compiler, automatically | **you**, via an allocator |
| Lifetime | ends when the scope ends | until you `free` |
| Size known at | compile time | runtime |
| Cost | free (a register adjustment) | a function call, maybe a syscall |
| Typical size | 8 MiB, fixed | gigabytes |
| Can be returned from a function | **no** | yes |
| Fails by | overflow → crash | `error.OutOfMemory` |

## What goes on the stack

Everything, by default. Locals, parameters, struct values, fixed-size arrays:

```zig
const std = @import("std");

const Big = struct {
    data: [256]u8,
    tag: u32,
};

fn byValue(b: Big) u32 {
    return b.tag;
}

fn byPointer(b: *const Big) u32 {
    return b.tag;
}

fn mutate(b: *Big) void {
    b.tag = 999;
}

pub fn main() !void {
    var b: Big = .{ .data = undefined, .tag = 7 };

    std.debug.print("@sizeOf(Big)  = {d}\n", .{@sizeOf(Big)});
    std.debug.print("@sizeOf(*Big) = {d}\n", .{@sizeOf(*Big)});
    std.debug.print("byValue   = {d}\n", .{byValue(b)});
    std.debug.print("byPointer = {d}\n", .{byPointer(&b)});
    mutate(&b);
    std.debug.print("after mutate = {d}\n", .{b.tag});
}
```

```
@sizeOf(Big)  = 260
@sizeOf(*Big) = 8
byValue   = 7
byPointer = 7
after mutate = 999
```

`byValue(b)` copies 260 bytes. `byPointer(&b)` copies 8. Neither touches the
heap — a pointer to a stack value is a perfectly good pointer, as long as the
stack frame it points into is still alive.

| Signature | Copies | Can mutate the caller's value |
|---|---|---|
| `fn f(b: Big)` | the whole struct | no (it's a copy) |
| `fn f(b: *const Big)` | 8 bytes | no |
| `fn f(b: *Big)` | 8 bytes | yes |

Rule of thumb: pass by value up to ~16 bytes, by `*const` above that, by `*`
only when you intend to mutate.

## The bug: returning a pointer to a local

**Deliberate bug** — this one does not compile:

```zig
const std = @import("std");

fn stackCounter() *u64 {
    var n: u64 = 42;
    return &n;
}

pub fn main() !void {
    const p = stackCounter();
    std.debug.print("p.* = {d}\n", .{p.*});
}
```

```
a2.zig:5:13: error: returning address of expired local variable 'n'
    return &n;
            ^
a2.zig:4:9: note: declared runtime-known here
    var n: u64 = 42;
        ^
```

Zig 0.17 catches the obvious form at **compile time**. In C this compiles with a
warning at best.

### The form it does *not* catch

Launder the pointer through a slice and the check loses track of it:

**Deliberate bug**, and this one *does* compile:

```zig
const std = @import("std");

fn badLabel() []const u8 {
    var buf: [16]u8 = undefined;
    const s = std.fmt.bufPrint(&buf, "id-{d}", .{99}) catch unreachable;
    return s;
}

fn noise() u64 {
    var pad: [4096]u8 = undefined;
    for (&pad, 0..) |*b, i| b.* = @truncate(i);
    var sum: u64 = 0;
    for (pad) |b| sum += b;
    return sum;
}

pub fn main() !void {
    const s = badLabel();
    _ = noise();
    std.debug.print("label = \"{s}\"\n", .{s});
}
```

```
label = "h?"
```

It compiles. It runs. It prints garbage. `badLabel` returned a slice pointing
into its own dead frame, and `noise()` reused those bytes for something else.
Without the `noise()` call it often prints `id-99` and looks fine — which is
exactly what makes this class of bug expensive. **This is the single most
important failure mode in the module: a dangling slice is still a slice.**

### Two correct fixes

```zig
const std = @import("std");

fn heapLabel(gpa: std.mem.Allocator) ![]u8 {
    return std.fmt.allocPrint(gpa, "id-{d}", .{99});
}

fn bufLabel(buf: []u8) ![]const u8 {
    return std.fmt.bufPrint(buf, "id-{d}", .{99});
}

fn noise() u64 {
    var pad: [4096]u8 = undefined;
    for (&pad, 0..) |*b, i| b.* = @truncate(i);
    var sum: u64 = 0;
    for (pad) |b| sum += b;
    return sum;
}

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer _ = gpa_s.deinit();
    const gpa = gpa_s.allocator();

    const owned = try heapLabel(gpa);
    defer gpa.free(owned);

    var buf: [16]u8 = undefined;
    const borrowed = try bufLabel(&buf);

    _ = noise();
    std.debug.print("heap-returned   = \"{s}\"\n", .{owned});
    std.debug.print("caller's buffer = \"{s}\"\n", .{borrowed});
}
```

```
heap-returned   = "id-99"
caller's buffer = "id-99"
```

| Fix | Signature shape | Who frees |
|---|---|---|
| allocate on the heap | `fn f(gpa: Allocator) ![]u8` | the caller, with the same allocator |
| write into the caller's buffer | `fn f(buf: []u8) ![]const u8` | nobody — it's the caller's stack |

These are the only two ways a function hands bytes back. Lesson 5.6 turns that
into a naming convention.

## When the heap is actually required

Not "when the data is big" — **when the size isn't known at compile time.**

```zig
const std = @import("std");

pub fn main() !void {
    var n: usize = 0;
    n = 5;

    var fixed: [5]u8 = undefined;
    _ = &fixed;

    var dynamic: [n]u8 = undefined;
    _ = &dynamic;
}
```

**Deliberate compile error:**

```
a7.zig:10:19: error: unable to resolve comptime value
    var dynamic: [n]u8 = undefined;
                  ^
a7.zig:10:19: note: types must be comptime-known
```

`[n]u8` is not a type when `n` is a runtime value. There are no VLAs. The heap
is how you get a runtime length:

```zig
const std = @import("std");

fn readSized(gpa: std.mem.Allocator, n: usize) ![]u8 {
    const buf = try gpa.alloc(u8, n);
    for (buf, 0..) |*b, i| b.* = @intCast('a' + i % 26);
    return buf;
}

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer _ = gpa_s.deinit();
    const gpa = gpa_s.allocator();

    var n: usize = 0;
    n = 30;

    const buf = try readSized(gpa, n);
    defer gpa.free(buf);
    std.debug.print("n={d} -> \"{s}\"\n", .{ n, buf });
}
```

```
n=30 -> "abcdefghijklmnopqrstuvwxyzabcd"
```

You need the heap when:

| Reason | Example |
|---|---|
| size is runtime-known | a file of unknown length, an HTTP body |
| the value must outlive its scope | returning a parsed struct |
| the collection grows | `ArrayList`, `HashMap` |
| it's too big for the stack | a 64 MiB buffer |
| lifetime is shared or dynamic | a cache, a graph, a linked list |

Everything else should stay on the stack.

## The stack has a hard limit

8 MiB on macOS and Linux by default, and overflowing it is not a catchable
error:

```zig
const std = @import("std");

fn blowTheStack(depth: u64) u64 {
    var frame: [8192]u8 = undefined;
    frame[0] = @truncate(depth);
    if (depth == 0) return frame[0];
    return blowTheStack(depth - 1) + frame[0];
}

pub fn main() !void {
    std.debug.print("recursing with 8 KiB frames...\n", .{});
    const r = blowTheStack(10_000);
    std.debug.print("r = {d}\n", .{r});
}
```

**Deliberate crash.** 10,000 frames × 8 KiB = 80 MiB of stack:

```
recursing with 8 KiB frames...
Segmentation fault at address 0x16e1dee50
/path/a6.zig:3: 0x100d2ffd4 in blowTheStack (a6)
fn blowTheStack(depth: u64) u64 {

/path/a6.zig:7:24: 0x100d30087 in blowTheStack (a6)
    return blowTheStack(depth - 1) + frame[0];
                       ^
... (hundreds of identical frames)
```

Note what you *don't* get: no `error.StackOverflow`, no `try`, no recovery. The
process dies. A heap allocation that fails gives you `error.OutOfMemory` and a
chance to respond; a stack that runs out does not. That asymmetry is the real
argument for moving deep recursion and large buffers to the heap.

(The addresses and the frame count differ every run. The crash does not.)

---

Next: [5.2 — The Allocator Interface](./02-allocator-interface.md)
