# 15.2 — Memory Layout

In Go and Rust a struct's fields sit in the order you wrote them, padded.
**Zig reorders them**, and that one fact explains `extern`, `packed`, and most
ABI bugs you'll ever hit.

## Zig reorders your fields

```zig
const std = @import("std");

const S = struct { a: u8, b: u32, c: u8 };

pub fn main() !void {
    std.debug.print("@sizeOf      = {d}\n", .{@sizeOf(S)});
    std.debug.print("@alignOf     = {d}\n", .{@alignOf(S)});
    std.debug.print("offsetOf(a)  = {d}\n", .{@offsetOf(S, "a")});
    std.debug.print("offsetOf(b)  = {d}\n", .{@offsetOf(S, "b")});
    std.debug.print("offsetOf(c)  = {d}\n", .{@offsetOf(S, "c")});
    std.debug.print("sum of parts = {d}\n", .{@sizeOf(u8) + @sizeOf(u32) + @sizeOf(u8)});
    std.debug.print("padding      = {d}\n", .{@sizeOf(S) - 6});
}
```

```
@sizeOf      = 8
@alignOf     = 4
offsetOf(a)  = 4
offsetOf(b)  = 0
offsetOf(c)  = 5
sum of parts = 6
padding      = 2
```

**`b` is at offset 0, not 4.** Zig put the `u32` first so it lands on its
natural 4-byte boundary, then packed both `u8`s into the tail:

```
offset: 0   1   2   3   4   5   6   7
        [------ b ------] [a] [c] [ pad ]
```

Declaration order gives `[a][pad×3][b][c][pad×3]` = 12 bytes. Zig's order is 8.
Had you written the fields in the good order yourself, C would also give 8 —
Zig just stops you from having to think about it.

| Language | Field order | Size of `{u8, u32, u8}` |
|---|---|---|
| C / C++ | as declared | 12 |
| Rust (`repr(Rust)`) | compiler's choice | 8 |
| Go | as declared | 12 |
| **Zig** (plain `struct`) | **compiler's choice** | **8** |
| Zig `extern struct` | as declared (C ABI) | 12 |

**Never assume a layout for a plain `struct`.** It is not specified, it can
change between compiler versions, and it already differs from what you wrote.

## plain vs `packed` vs `extern`

```zig
const std = @import("std");

const Plain = struct { a: u8, b: u32, c: u8 };
const Packed = packed struct { a: u8, b: u32, c: u8 };
const Extern = extern struct { a: u8, b: u32, c: u8 };

fn row(comptime name: []const u8, comptime T: type) void {
    std.debug.print("{s:<8} size={d} align={d}  a@{d} b@{d} c@{d}\n", .{
        name,              @sizeOf(T),        @alignOf(T),
        @offsetOf(T, "a"), @offsetOf(T, "b"), @offsetOf(T, "c"),
    });
}

pub fn main() !void {
    row("plain", Plain);
    row("packed", Packed);
    row("extern", Extern);
    std.debug.print("\n@bitSizeOf(Packed) = {d}\n", .{@bitSizeOf(Packed)});
    std.debug.print("backing int        = {s}\n", .{@typeName(@typeInfo(Packed).@"struct".backing_integer.?)});
}
```

```
plain    size=8 align=4  a@4 b@0 c@5
packed   size=8 align=8  a@0 b@1 c@5
extern   size=12 align=4  a@0 b@4 c@8

@bitSizeOf(Packed) = 48
backing int        = u48
```

Three different layouts from three keywords on identical fields:

| | Order | Padding | `@sizeOf` | What it really is |
|---|---|---|---|---|
| `struct` | **compiler's** | yes, wherever it likes | 8 | "store these fields efficiently" |
| `extern struct` | **declared** | C's rules | 12 | "match the C ABI exactly" |
| `packed struct` | **declared** | **none, bit-level** | 8 | "an integer with named bit ranges" |

A `packed struct` is literally an integer: `@bitSizeOf` is 48 and
`backing_integer` is `u48`. `@sizeOf` is 8 because a `u48` needs an 8-byte
container. Fields sit at **bit** offsets with zero padding, which is why
`@offsetOf(Packed, "b")` is 1 — byte 1, bit 0.

**`@bitSizeOf` only works on `packed` structs.** On a plain or `extern` struct
it is a compile error:

```
error: no bit size available for type 'Plain'
```

Because the padding bytes aren't "bits of the value" — they're holes.

## Why `extern` is mandatory at an ABI boundary

Same 8 bytes off the wire, read two ways:

```zig
const std = @import("std");

const ZigOrder = struct { tag: u8, value: u32 };
const COrder = extern struct { tag: u8, value: u32 };

pub fn main() !void {
    const wire = [_]u8{ 0x07, 0x00, 0x00, 0x00, 0x44, 0x33, 0x22, 0x11 };
    std.debug.print("wire bytes (C struct {{u8 tag; u32 value;}}) = {x}\n\n", .{wire});

    const as_c = std.mem.bytesAsValue(COrder, &wire);
    std.debug.print("extern struct: tag={d} value=0x{x}   <- correct\n", .{ as_c.tag, as_c.value });
    std.debug.print("  offsetOf: tag@{d} value@{d}\n\n", .{ @offsetOf(COrder, "tag"), @offsetOf(COrder, "value") });

    const as_zig = std.mem.bytesAsValue(ZigOrder, &wire);
    std.debug.print("plain struct : tag={d} value=0x{x}   <- GARBAGE\n", .{ as_zig.tag, as_zig.value });
    std.debug.print("  offsetOf: tag@{d} value@{d}\n", .{ @offsetOf(ZigOrder, "tag"), @offsetOf(ZigOrder, "value") });
}
```

```
wire bytes (C struct {u8 tag; u32 value;}) = 0700000044332211

extern struct: tag=7 value=0x11223344   <- correct
  offsetOf: tag@0 value@4

plain struct : tag=68 value=0x7   <- GARBAGE
  offsetOf: tag@4 value@0
```

`tag=68` and `value=0x7` — it read `value`'s first byte as `tag` and `tag`'s
byte as `value`. **No crash, no warning, no error return.** Just wrong numbers
flowing downstream.

Use `extern struct` whenever bytes cross a boundary you don't control:

| Boundary | Need |
|---|---|
| a C function's parameter or return | `extern struct` |
| a struct written to / read from a file | `extern` or `packed` |
| a network packet or on-disk record | `packed` (bit-exact) or `extern` |
| shared memory / `mmap` with another process | `extern` |
| anything you `@ptrCast` raw bytes into | `extern` or `packed` |
| **purely internal to your Zig program** | plain `struct` — let Zig optimize |

`@cImport` generates `extern struct` for you, which is why C interop "just
works" — the keyword is doing the work.

## Alignment

```zig
const std = @import("std");

pub fn main() !void {
    inline for (.{ u8, u16, u32, u64, u128, f32, f64, bool, usize, *u8, ?*u8, ?u32, []const u8, [3]u8, [4]u32 }) |T| {
        std.debug.print("{s:<12} size={d:>2} align={d:>2}\n", .{ @typeName(T), @sizeOf(T), @alignOf(T) });
    }

    var raw: [128]u8 align(64) = @splat(0);
    std.debug.print("\nbase addr mod 64 = {d}\n", .{@intFromPtr(&raw) % 64});

    const Al = std.mem.Alignment;
    inline for (.{ 1, 7, 33 }) |off| {
        const addr = @intFromPtr(&raw) + off;
        std.debug.print("addr+{d:<2}: mod 8 = {d}  next 8-aligned = +{d}  next 64-aligned = +{d}\n", .{
            off,
            addr % 8,
            Al.@"8".forward(addr) - addr,
            Al.fromByteUnits(64).forward(addr) - addr,
        });
    }

    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();
    const cacheline = try gpa.alignedAlloc(u8, .fromByteUnits(64), 8);
    defer gpa.free(cacheline);
    std.debug.print("\nalignedAlloc type = {s}, addr mod 64 = {d}\n", .{ @typeName(@TypeOf(cacheline)), @intFromPtr(cacheline.ptr) % 64 });
    std.debug.print("page size = {d}\n", .{std.heap.pageSize()});
}
```

```
u8           size= 1 align= 1
u16          size= 2 align= 2
u32          size= 4 align= 4
u64          size= 8 align= 8
u128         size=16 align=16
f32          size= 4 align= 4
f64          size= 8 align= 8
bool         size= 1 align= 1
usize        size= 8 align= 8
*u8          size= 8 align= 8
?*u8         size= 8 align= 8
?u32         size= 8 align= 4
[]const u8   size=16 align= 8
[3]u8        size= 3 align= 1
[4]u32       size=16 align= 4

base addr mod 64 = 0
addr+1 : mod 8 = 1  next 8-aligned = +7  next 64-aligned = +63
addr+7 : mod 8 = 7  next 8-aligned = +1  next 64-aligned = +57
addr+33: mod 8 = 1  next 8-aligned = +7  next 64-aligned = +31

alignedAlloc type = []align(64) u8, addr mod 64 = 0
page size = 16384
```

The rules, all four of them:

| Rule | |
|---|---|
| a scalar's alignment == its size, capped at the pointer width | `u128` is the exception: 16 |
| a struct's alignment == its **largest field's** alignment | `extern struct {u8, u32}` → 4 |
| a struct's size is **rounded up** to its own alignment | so arrays of it stay aligned |
| an array's alignment == its element's | `[3]u8` → 1 |

Three rows worth stopping on:

- **`?*u8` is 8 bytes, same as `*u8`.** Null-pointer optimization: `null` is
  address 0. But **`?u32` is 8, not 4** — there's no spare bit pattern in a
  `u32`, so the tag needs its own byte and padding follows. "No null pointers"
  is free; optional integers are not.
- **`[]const u8` is 16 bytes** — pointer plus length. Passing slices by value is
  cheap but not free; it's two registers.
- **`[4]u32` is align 4, not 16.** Element alignment, not total size. Only
  `align(N)` or `alignedAlloc` buys you more.

`std.mem.Alignment` is an enum, not a number. **`alignedAlloc` takes
`?std.mem.Alignment`** — `.fromByteUnits(64)` or `.@"64"`; passing `64` won't
compile. The result type carries it: `[]align(64) u8`.

## Struct-size golf

The one optimization that reliably pays off in a backend service: make your hot
record smaller.

```zig
const std = @import("std");

const V1 = extern struct { id: u64, kind: u8, flag: bool, score: f64, count: u32 };
const V2 = extern struct { id: u64, score: f64, count: u32, kind: u8, flag: bool };
const V3 = struct { id: u64, score: f64, count: u32, kind: u8, flag: bool };
const V4 = struct { id: u32, score: f32, count: u32, kind: u8, flag: bool };
const V5 = packed struct { id: u32, score: f32, count: u24, kind: u7, flag: bool };

fn row(comptime name: []const u8, comptime T: type) void {
    std.debug.print("{s} size={d:>3} align={d}  1M rows = {d:>9} B\n", .{ name, @sizeOf(T), @alignOf(T), @sizeOf(T) * 1_000_000 });
}

pub fn main() !void {
    row("V1 extern, bad order ", V1);
    row("V2 extern, hand-sorted", V2);
    row("V3 plain, Zig sorts it", V3);
    row("V4 narrower types     ", V4);
    row("V5 packed bitfields   ", V5);
    std.debug.print("\nV5 @bitSizeOf = {d} bits (backing {s}); @sizeOf rounds up to its alignment\n", .{ @bitSizeOf(V5), @typeName(@typeInfo(V5).@"struct".backing_integer.?) });
}
```

```
V1 extern, bad order  size= 32 align=8  1M rows =  32000000 B
V2 extern, hand-sorted size= 24 align=8  1M rows =  24000000 B
V3 plain, Zig sorts it size= 24 align=8  1M rows =  24000000 B
V4 narrower types      size= 16 align=4  1M rows =  16000000 B
V5 packed bitfields    size= 16 align=16  1M rows =  16000000 B

V5 @bitSizeOf = 96 bits (backing u96); @sizeOf rounds up to its alignment
```

**32 → 16 bytes: 16 MB saved per million rows**, and twice as many records per
cache line. The steps, in order of payoff:

| Step | V1→V2 | Gain | Note |
|---|---|---|---|
| sort fields big→small (in `extern`) | 32 → 24 | 25% | mandatory if `extern`; free if plain |
| **let Zig do it** (drop `extern`) | — | same as V2 | V3 == V2 without you thinking |
| **narrow the types** | 24 → 16 | 33% | `u64`→`u32` id, `f64`→`f32` score |
| `packed` bitfields | 16 → 16 | **0** | 96 bits, but align 16 rounds it back up |

The lesson in V5: **`packed` is not an optimization.** It removed 32 bits of
padding and then `@sizeOf` rounded the `u96` back up to 16 anyway. `packed`
exists for *bit-exact* layouts — protocol headers, hardware registers — not for
shrinking records. For that, narrow your field types.

And for a plain `struct`, field order costs you nothing: **V3 is already
optimal.** Reordering by hand only matters once you've written `extern`.

## Inspecting any type

| Builtin | Gives | Works on |
|---|---|---|
| `@sizeOf(T)` | bytes, incl. padding | everything |
| `@alignOf(T)` | required address multiple | everything |
| `@bitSizeOf(T)` | bits, no padding | ints, floats, bools, **`packed` only** |
| `@offsetOf(T, "f")` | byte offset of a field | structs (any kind) |
| `@bitOffsetOf(T, "f")` | bit offset | `packed struct` |
| `@typeInfo(T).@"struct".backing_integer` | `?type` | `packed struct` → the int |
| `@typeInfo(T).@"struct".layout` | `.auto` / `.@"extern"` / `.@"packed"` | structs |

---

Next: [15.3 — Bits & Bytes](./03-bits-and-bytes.md)
