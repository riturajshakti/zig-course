# 5.2 — The Allocator Interface

`std.mem.Allocator` is **16 bytes of struct you pass around like an `int`**. Not
a global, not a trait, not a keyword.

```zig
const std = @import("std");

pub fn main() !void {
    std.debug.print("@sizeOf(std.mem.Allocator) = {d}\n", .{@sizeOf(std.mem.Allocator)});
    std.debug.print("fields: ptr ({d} bytes) + vtable ({d} bytes)\n", .{
        @sizeOf(*anyopaque),
        @sizeOf(*const std.mem.Allocator.VTable),
    });

    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer _ = gpa_s.deinit();
    const gpa: std.mem.Allocator = gpa_s.allocator();

    std.debug.print("vtable has 4 fns: ", .{});
    inline for (@typeInfo(std.mem.Allocator.VTable).@"struct".field_names) |name| {
        std.debug.print("{s} ", .{name});
    }
    std.debug.print("\n", .{});
    std.debug.print("gpa.ptr    = {*}\n", .{gpa.ptr});
    std.debug.print("gpa.vtable = {*}\n", .{gpa.vtable});
}
```

```
@sizeOf(std.mem.Allocator) = 16
fields: ptr (8 bytes) + vtable (8 bytes)
vtable has 4 fns: alloc resize remap free 
gpa.ptr    = anyopaque@16f536488
gpa.vtable = mem.Allocator.VTable@100a55ac0
```

That is the whole interface: an opaque `ptr` to some allocator's state, and a
`vtable` of four function pointers. Lesson 5.7 implements all four by hand.

## Why there is no global allocator

A function that takes `gpa: std.mem.Allocator` works with **every** allocator,
and the caller picks:

```zig
const std = @import("std");

fn sum(gpa: std.mem.Allocator, n: usize) !u64 {
    const xs = try gpa.alloc(u64, n);
    defer gpa.free(xs);
    for (xs, 0..) |*x, i| x.* = i;
    var total: u64 = 0;
    for (xs) |x| total += x;
    return total;
}

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer _ = gpa_s.deinit();

    var arena_s: std.heap.ArenaAllocator = .init(gpa_s.allocator());
    defer arena_s.deinit();

    var fba_buf: [4096]u8 = undefined;
    var fba: std.heap.FixedBufferAllocator = .init(&fba_buf);

    std.debug.print("debug  : {d}\n", .{try sum(gpa_s.allocator(), 100)});
    std.debug.print("arena  : {d}\n", .{try sum(arena_s.allocator(), 100)});
    std.debug.print("fixed  : {d}\n", .{try sum(fba.allocator(), 100)});
    std.debug.print("page   : {d}\n", .{try sum(std.heap.page_allocator, 100)});
}
```

```
debug  : 4950
arena  : 4950
fixed  : 4950
page   : 4950
```

One `sum`, four allocation strategies, zero changes to `sum`. Compare:

| Language | Who chooses the allocator |
|---|---|
| Node / Python | the runtime. You cannot. |
| Go | the runtime GC. You cannot. |
| Rust | global `#[global_allocator]`, or `A: Allocator` generics (unstable) |
| C | `malloc`, hardcoded into every library you link |
| **Zig** | **the caller, per call, at zero cost** |

This is why Zig libraries work in kernels, on microcontrollers, in WASM and in
a server all at once. A library that calls `malloc` internally has made that
choice for you forever.

**The rule: if a function allocates, it takes an `Allocator` parameter.** You
will see this in every signature in the standard library.

## The five allocation calls

```zig
const std = @import("std");

const Node = struct { value: i32, next: ?*Node };

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer _ = gpa_s.deinit();
    const gpa = gpa_s.allocator();

    const slice = try gpa.alloc(u32, 3);
    defer gpa.free(slice);
    slice[0] = 10;
    slice[1] = 20;
    slice[2] = 30;
    std.debug.print("alloc  : {any} len={d}\n", .{ slice, slice.len });

    const node = try gpa.create(Node);
    defer gpa.destroy(node);
    node.* = .{ .value = 7, .next = null };
    std.debug.print("create : value={d} next={?}\n", .{ node.value, node.next });

    const copy = try gpa.dupe(u8, "borrowed bytes");
    defer gpa.free(copy);
    std.debug.print("dupe   : \"{s}\"\n", .{copy});

    const z = try gpa.allocSentinel(u8, 3, 0);
    defer gpa.free(z);
    @memcpy(z[0..3], "abc");
    std.debug.print("sentinel: \"{s}\" z[3]={d}\n", .{ z, z[3] });

    const printed = try std.fmt.allocPrint(gpa, "{s}-{d}", .{ "id", 42 });
    defer gpa.free(printed);
    std.debug.print("allocPrint: \"{s}\"\n", .{printed});
}
```

```
alloc  : { 10, 20, 30 } len=3
create : value=7 next=null
dupe   : "borrowed bytes"
sentinel: "abc" z[3]=0
allocPrint: "id-42"
```

| Call | Returns | Freed with | Use for |
|---|---|---|---|
| `alloc(T, n)` | `[]T` | `free(slice)` | n items |
| `create(T)` | `*T` | `destroy(ptr)` | exactly one item |
| `dupe(T, src)` | `[]T` (a copy) | `free(slice)` | owning a copy of borrowed data |
| `allocSentinel(u8, n, 0)` | `[:0]u8` | `free(slice)` | C-compatible strings |
| `std.fmt.allocPrint(gpa, …)` | `[]u8` | `free(slice)` | formatted strings |

The pairings are strict: **`alloc`→`free`, `create`→`destroy`.** Mixing them is
a bug (lesson 5.8 shows what it looks like).

`dupe` is the one to internalize early. A `[]const u8` parameter is *borrowed* —
it may be a stack buffer or a string literal. If you intend to store it past the
function call, `dupe` it:

```zig
name: []u8 = try gpa.dupe(u8, name_param)
```

## Growing: `realloc`, `resize`, `remap`

```zig
const std = @import("std");

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer _ = gpa_s.deinit();
    const gpa = gpa_s.allocator();

    var buf = try gpa.alloc(u8, 4);
    @memcpy(buf, "abcd");
    std.debug.print("before : len={d} ptr={*} \"{s}\"\n", .{ buf.len, buf.ptr, buf });

    buf = try gpa.realloc(buf, 16);
    @memcpy(buf[4..8], "efgh");
    std.debug.print("grown  : len={d} ptr={*} \"{s}\"\n", .{ buf.len, buf.ptr, buf[0..8] });

    buf = try gpa.realloc(buf, 2);
    std.debug.print("shrunk : len={d} ptr={*} \"{s}\"\n", .{ buf.len, buf.ptr, buf });

    gpa.free(buf);

    var grow = try gpa.alloc(u8, 8);
    defer gpa.free(grow);
    const ok = gpa.resize(grow, 6);
    std.debug.print("resize(8->6) in place: {}\n", .{ok});
    if (ok) grow = grow[0..6];

    const huge = gpa.resize(grow, 1 << 20);
    std.debug.print("resize(6->1MiB) in place: {}\n", .{huge});
}
```

```
before : len=4 ptr=u8@102e80000 "abcd"
grown  : len=16 ptr=u8@102ea0000 "abcdefgh"
shrunk : len=2 ptr=u8@102ec0000 "ab"
resize(8->6) in place: true
resize(6->1MiB) in place: false
```

| Call | Returns | May move the pointer | On failure |
|---|---|---|---|
| `realloc(slice, n)` | `![]T` | **yes** | `error.OutOfMemory`, old slice untouched |
| `resize(slice, n)` | `bool` | never | `false` |
| `remap(slice, n)` | `?[]T` | yes | `null` |

**`realloc` can move your data.** Every pointer into the old slice is dangling
after it returns, which is exactly why `ArrayList.append` invalidates pointers
to its items. Always reassign: `buf = try gpa.realloc(buf, n)`.

`resize` is the "only if free" variant — it returns `false` rather than copy, so
you can take a fast path and fall back. `remap` is new in 0.17: a `realloc` that
reports failure as `null` instead of an error.

## Alignment

`alignedAlloc` takes a `std.mem.Alignment`, **not a number** — a 0.17 change
that breaks old code:

```zig
const std = @import("std");

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer _ = gpa_s.deinit();
    const gpa = gpa_s.allocator();

    const natural = try gpa.alloc(u32, 4);
    defer gpa.free(natural);

    const aligned = try gpa.alignedAlloc(u8, .fromByteUnits(64), 100);
    defer gpa.free(aligned);

    const cacheline = try gpa.alignedAlloc(u8, .@"64", 8);
    defer gpa.free(cacheline);

    std.debug.print("u32 natural align = {d}, addr % 4  = {d}\n", .{ @alignOf(u32), @intFromPtr(natural.ptr) % 4 });
    std.debug.print("alignedAlloc(64)  : addr % 64 = {d} type={s}\n", .{ @intFromPtr(aligned.ptr) % 64, @typeName(@TypeOf(aligned)) });
    std.debug.print("enum literal .@\"64\": addr % 64 = {d}\n", .{@intFromPtr(cacheline.ptr) % 64});
}
```

```
u32 natural align = 4, addr % 4  = 0
alignedAlloc(64)  : addr % 64 = 0 type=[]align(64) u8
enum literal .@"64": addr % 64 = 0
```

The alignment is in the **type**: `[]align(64) u8`. Both spellings work —
`.fromByteUnits(64)` and the enum literal `.@"64"`. Pass `null` to mean
"naturally aligned", which is what plain `alloc` does.

You need this for SIMD, DMA buffers, cache-line-aligned structs to avoid false
sharing, and `mmap`-style page alignment. Rarely otherwise.

## The `error.OutOfMemory` contract

Every allocating call returns `error{OutOfMemory}!T`. That's the entire error
set — one member:

```zig
const std = @import("std");

pub fn main() !void {
    var small: [64]u8 = undefined;
    var fba: std.heap.FixedBufferAllocator = .init(&small);
    const gpa = fba.allocator();

    const a = try gpa.alloc(u8, 32);
    std.debug.print("got {d} bytes\n", .{a.len});

    const b = gpa.alloc(u8, 1024) catch |err| {
        std.debug.print("second alloc failed: {t}\n", .{err});
        std.debug.print("error set: {s}\n", .{@typeName(std.mem.Allocator.Error)});
        return;
    };
    std.debug.print("unreachable: {d}\n", .{b.len});
}
```

**Deliberate failure** — 1024 bytes requested from a 64-byte buffer:

```
got 32 bytes
second alloc failed: OutOfMemory
error set: error{OutOfMemory}
```

The contract, which matters in interviews:

| Guarantee | Meaning |
|---|---|
| returns `error.OutOfMemory` or valid memory | no `null` checks, no half-states |
| on failure, nothing was allocated | you don't need to clean up a failed `alloc` |
| on failed `realloc`, the old slice is still valid and still yours | you must still free it |
| contents are `undefined`, not zeroed | use `@memset` if you need zeros |
| `free` never fails | so it is safe in `defer` |

Because `free` cannot fail, `defer gpa.free(x)` is always correct and the next
three lessons lean on it constantly.

---

Next: [5.3 — DebugAllocator](./03-debug-allocator.md)
