# 3.5 — Pointers

Zig has **four** pointer-ish types, and picking the wrong one is where the bugs
live:

| Type | Points at | Has a length | Indexable | Arithmetic |
|---|---|---|---|---|
| `*T` | exactly one `T` | n/a | no | no |
| `[*]T` | many `T`s, count unknown | **no** | yes — **unchecked** | yes |
| `[]T` | many `T`s | yes, `.len` | yes — **checked** | no |
| `*[N]T` | an array of exactly `N` | yes, comptime | yes — checked | no |

Everything below is about when to reach for which.

## `*T` and `*const T`

```zig
const std = @import("std");

pub fn main() void {
    var n: i32 = 42;
    const p: *i32 = &n;
    const cp: *const i32 = &n;

    std.debug.print("n      = {d}\n", .{n});
    std.debug.print("p.*    = {d}\n", .{p.*});
    std.debug.print("cp.*   = {d}\n", .{cp.*});

    p.* = 99;
    std.debug.print("after p.* = 99 -> n = {d}, cp.* = {d}\n", .{ n, cp.* });

    std.debug.print("@TypeOf(p)  = {s}\n", .{@typeName(@TypeOf(p))});
    std.debug.print("@TypeOf(cp) = {s}\n", .{@typeName(@TypeOf(cp))});
    std.debug.print("@sizeOf(*i32) = {d}\n", .{@sizeOf(*i32)});
    std.debug.print("p == cp ? {}\n", .{@intFromPtr(p) == @intFromPtr(cp)});
}
```

```
n      = 42
p.*    = 42
cp.*   = 42
after p.* = 99 -> n = 99, cp.* = 99
@TypeOf(p)  = *i32
@TypeOf(cp) = *const i32
@sizeOf(*i32) = 8
p == cp ? true
```

`&x` takes the address, `p.*` dereferences — **postfix**, not `*p`. Two pointers
of different constness can alias the same byte; `const` restricts this *handle*,
not the memory.

```zig
const std = @import("std");

pub fn main() void {
    var n: i32 = 42;
    const cp: *const i32 = &n;
    cp.* = 99;
    std.debug.print("{d}\n", .{n});
}
```

```
p2.zig:6:7: error: cannot assign to constant
    cp.* = 99;
    ~~^~
```

Deliberate. `&x` on a `var` gives `*T`; on a `const` it gives `*const T` and
there is no cast back.

## There is no null pointer

```zig
const std = @import("std");

pub fn main() void {
    const p: *i32 = null;
    std.debug.print("{d}\n", .{p.*});
}
```

```
p4.zig:4:21: error: expected type '*i32', found '@TypeOf(null)'
    const p: *i32 = null;
                    ^~~~
```

A `*T` **always** points at a valid `T`. There is no null dereference to check
for, no `NullPointerException`, no `if (p != NULL)`. When absence is meaningful
you say so in the type: `?*T`.

```zig
const std = @import("std");

pub fn main() void {
    var n: i32 = 7;
    var maybe: ?*i32 = null;

    std.debug.print("maybe == null : {}\n", .{maybe == null});
    if (maybe) |ptr| {
        std.debug.print("if: got {d}\n", .{ptr.*});
    } else {
        std.debug.print("if: it was null\n", .{});
    }

    maybe = &n;
    if (maybe) |ptr| std.debug.print("if: got {d}\n", .{ptr.*});
    std.debug.print(".?            : {d}\n", .{maybe.?.*});
    std.debug.print("orelse default: {d}\n", .{(maybe orelse &n).*});

    std.debug.print("@sizeOf(*i32)  = {d}\n", .{@sizeOf(*i32)});
    std.debug.print("@sizeOf(?*i32) = {d}  <- no extra byte\n", .{@sizeOf(?*i32)});
    std.debug.print("@sizeOf(?i32)  = {d}  <- needs a flag\n", .{@sizeOf(?i32)});
}
```

```
maybe == null : true
if: it was null
if: got 7
.?            : 7
orelse default: 7
@sizeOf(*i32)  = 8
@sizeOf(?*i32) = 8  <- no extra byte
@sizeOf(?i32)  = 8  <- needs a flag
```

**`?*i32` costs the same 8 bytes as `*i32`.** Address 0 can never be a valid
pointer, so Zig uses it as the `null` tag — the same trick Rust's
`Option<&T>` uses. `?i32` by contrast needs a separate flag, hence padding to 8.

So you get C's representation with Rust's guarantee: the compiler will not let
you dereference it until you have unwrapped it with `if (p) |ptr|`, `.?` or
`orelse`. Module 04 covers optionals properly.

## `[*]T` vs `[]T` — the one to get right

```zig
const std = @import("std");

pub fn main() void {
    var arr = [_]i32{ 10, 20, 30, 40, 50 };

    const slice: []i32 = &arr;
    const many: [*]i32 = &arr;
    const arrptr: *[5]i32 = &arr;

    std.debug.print("slice  {s:<10} len={d}  val[2]={d}\n", .{ @typeName(@TypeOf(slice)), slice.len, slice[2] });
    std.debug.print("many   {s:<10} len=??  val[2]={d}\n", .{ @typeName(@TypeOf(many)), many[2] });
    std.debug.print("arrptr {s:<10} len={d}  val[2]={d}\n", .{ @typeName(@TypeOf(arrptr)), arrptr.len, arrptr[2] });

    std.debug.print("\n@sizeOf([]i32)    = {d}\n", .{@sizeOf([]i32)});
    std.debug.print("@sizeOf([*]i32)   = {d}\n", .{@sizeOf([*]i32)});
    std.debug.print("@sizeOf(*[5]i32)  = {d}\n", .{@sizeOf(*[5]i32)});

    std.debug.print("\nmany[1..3]        = {any}\n", .{many[1..3]});
    std.debug.print("many + arithmetic = {d}\n", .{(many + 2)[0]});
    std.debug.print("arrptr.*          = {any}\n", .{arrptr.*});
    std.debug.print("slice.ptr         = {s}\n", .{@typeName(@TypeOf(slice.ptr))});
}
```

```
slice  []i32      len=5  val[2]=30
many   [*]i32     len=??  val[2]=30
arrptr *[5]i32    len=5  val[2]=30

@sizeOf([]i32)    = 16
@sizeOf([*]i32)   = 8
@sizeOf(*[5]i32)  = 8

many[1..3]        = { 20, 30 }
many + arithmetic = 30
arrptr.*          = { 10, 20, 30, 40, 50 }
slice.ptr         = [*]i32
```

`[]T` is 16 bytes because it carries the length. `[*]T` is 8 — it is literally
C's `T*`, including pointer arithmetic (`many + 2`).

**`[*]T` has no `.len`:**

```zig
const std = @import("std");

pub fn main() void {
    var arr = [_]i32{ 10, 20, 30 };
    const many: [*]i32 = &arr;
    std.debug.print("len={d}\n", .{many.len});
}
```

```
p6.zig:6:40: error: type '[*]i32' does not support field access
    std.debug.print("len={d}\n", .{many.len});
                                   ~~~~^~~~
```

**And no bounds check.** This is the difference that matters:

```zig
const std = @import("std");

pub fn main() void {
    var arr = [_]i32{ 10, 20, 30 };
    const many: [*]i32 = &arr;
    const slice: []i32 = &arr;
    var i: usize = 9;
    _ = &i;

    std.debug.print("many[9]  = {d}   <- read past the end, no complaint\n", .{many[i]});
    std.debug.print("slice[9] = ", .{});
    std.debug.print("{d}\n", .{slice[i]});
}
```

```
many[9]  = 1   <- read past the end, no complaint
slice[9] = thread 16458164 panic: index out of bounds: index 9, len 3
p7.zig:12:37: 0x1050e9283 in main (p7)
    std.debug.print("{d}\n", .{slice[i]});
                                    ^
```

Deliberate. `many[9]` read six elements past the array and returned whatever
happened to be on the stack (the `1` is arbitrary — it is not your data). The
slice panicked instead.
(The thread id and the `0x...` addresses differ on every run — only the
message and the source location are stable.)


> **Use `[]T`.** `[*]T` exists for C interop and for the inside of allocator
> implementations. If you are writing it in application code, you almost
> certainly want a slice, and you can always build one: `ptr[0..len]`.

## Pointer to array: `*[N]T`

```zig
const std = @import("std");

fn takesSlice(s: []const i32) usize {
    return s.len;
}

fn takesMany(p: [*]const i32, n: usize) i32 {
    return p[n];
}

pub fn main() void {
    var arr = [_]i32{ 1, 2, 3, 4 };
    const ap: *[4]i32 = &arr;

    std.debug.print("takesSlice(&arr)   = {d}\n", .{takesSlice(&arr)});
    std.debug.print("takesSlice(ap)     = {d}\n", .{takesSlice(ap)});
    std.debug.print("takesSlice(ap[1..])= {d}\n", .{takesSlice(ap[1..])});

    const sl: []i32 = ap;
    std.debug.print("*[4]i32 -> []i32   = {any}\n", .{sl});

    const mp: [*]i32 = sl.ptr;
    std.debug.print("takesMany(mp, 3)   = {d}\n", .{takesMany(mp, 3)});

    const rebuilt: []i32 = mp[0..sl.len];
    std.debug.print("[*]T + len -> []T  = {any}\n", .{rebuilt});
}
```

```
takesSlice(&arr)   = 4
takesSlice(ap)     = 4
takesSlice(ap[1..])= 3
*[4]i32 -> []i32   = { 1, 2, 3, 4 }
takesMany(mp, 3)   = 4
[*]T + len -> []T  = { 1, 2, 3, 4 }
```

`*[N]T` is the best of both: 8 bytes like a raw pointer, but the length lives in
the type so it is bounds-checked *and* coerces to `[]T` for free. It is what
`&arr` actually produces.

| Conversion | How |
|---|---|
| `*[N]T` → `[]T` | automatic |
| `*[N]T` → `[*]T` | automatic |
| `[]T` → `[*]T` | `s.ptr` |
| `[*]T` → `[]T` | `p[0..len]` — **you supply the length** |
| `[]T` → `*[N]T` | `s[0..N]` with comptime `N` |
| `*T` → `*[1]T` | automatic |

## Passing by value vs by pointer

```zig
const std = @import("std");

const Big = struct { data: [256]u8, id: u32 };

fn readByValue(b: Big) u32 {
    return b.id;
}

fn readByPointer(b: *const Big) u32 {
    return b.id;
}

fn writeByPointer(b: *Big) void {
    b.id = 99;
}

pub fn main() void {
    var b: Big = .{ .data = @splat(0), .id = 1 };

    std.debug.print("@sizeOf(Big)       = {d} bytes\n", .{@sizeOf(Big)});
    std.debug.print("@sizeOf(*const Big) = {d} bytes\n", .{@sizeOf(*const Big)});
    std.debug.print("readByValue        = {d}\n", .{readByValue(b)});
    std.debug.print("readByPointer      = {d}\n", .{readByPointer(&b)});
    writeByPointer(&b);
    std.debug.print("after write        = {d}\n", .{b.id});
}
```

```
@sizeOf(Big)       = 260 bytes
@sizeOf(*const Big) = 8 bytes
readByValue        = 1
readByPointer      = 1
after write        = 99
```

| Pass | Signature | Semantics |
|---|---|---|
| by value | `fn f(x: T)` | callee gets a copy; cannot affect the caller |
| read-only pointer | `fn f(x: *const T)` | no copy, no mutation — the default for large types |
| mutable pointer | `fn f(x: *T)` | **this is how you return a second value** |

Unlike C++, you never write `&` in the *declaration* to mean "reference". The
`*` is visible at the call site too (`f(&b)`), so a reader can always see which
calls can mutate their arguments.

**Do not reach for a pointer just to avoid a copy.** Zig passes struct parameters
by value *semantically*, but is free to pass a hidden pointer when the type is
large and the callee does not mutate — so `fn f(b: Big)` is usually not a real
260-byte memcpy. Use `*const T` when you need a stable address or the type is
genuinely huge; use a value otherwise, because it cannot alias or dangle.

## `@ptrCast` — reinterpreting bytes

```zig
const std = @import("std");

pub fn main() void {
    const n: u32 = 0x41424344;
    const n_bytes: *const [4]u8 = @ptrCast(&n);

    std.debug.print("u32 0x41424344 -> bytes = {any}\n", .{n_bytes.*});
    std.debug.print("                 as text = {s}\n", .{n_bytes.*});
    std.debug.print("std.mem.asBytes          = {any}\n", .{std.mem.asBytes(&n).*});

    const aligned: [2]u32 align(4) = .{ 1, 2 };
    const back: *const [8]u8 = @ptrCast(&aligned);
    std.debug.print("[2]u32{{1,2}} as bytes    = {any}\n", .{back.*});

    const round: *const [2]u32 = @alignCast(@ptrCast(back));
    std.debug.print("back to [2]u32           = {any}\n", .{round.*});
}
```

```
u32 0x41424344 -> bytes = { 68, 67, 66, 65 }
                 as text = DCBA
std.mem.asBytes          = { 68, 67, 66, 65 }
[2]u32{1,2} as bytes    = { 1, 0, 0, 0, 2, 0, 0, 0 }
back to [2]u32           = { 1, 2 }
```

`0x41424344` is `"ABCD"` in hex, and the bytes came out **`DCBA`** — this machine
is little-endian, least significant byte first.

Casting *up* in alignment is refused unless you assert it:

```zig
const std = @import("std");

pub fn main() void {
    const bytes = [_]u8{ 0x01, 0x00, 0x00, 0x00, 0x02, 0x00, 0x00, 0x00 };
    const as_u32: *const [2]u32 = @ptrCast(&bytes);
    std.debug.print("{any}\n", .{as_u32.*});
}
```

```
p9b.zig:5:35: error: @ptrCast increases pointer alignment
    const as_u32: *const [2]u32 = @ptrCast(&bytes);
                                  ^~~~~~~~~~~~~~~~
p9b.zig:5:44: note: '*const [8]u8' has alignment '1'
p9b.zig:5:35: note: '*const [2]u32' has alignment '4'
p9b.zig:5:35: note: use @alignCast to assert pointer alignment
```

Deliberate, and the error tells you the fix. `@alignCast` is an **assertion**,
not a conversion: it panics in Debug if the address is not actually aligned.

`@ptrCast` is the escape hatch for binary protocols, memory-mapped files and C
interop. Prefer `std.mem.asBytes`, `std.mem.bytesAsValue` or `@bitCast` when they
fit — module 15 covers all of this.

## Operations table

| Operation | Code |
|---|---|
| take an address | `&x` |
| dereference | `p.*` |
| read a field through | `p.field` — no `->` |
| call a method through | `p.method()` |
| declare mutable | `const p: *T = &x;` (needs `var x`) |
| declare read-only | `const p: *const T = &x;` |
| optional pointer | `?*T`, unwrapped with `if (p) \|q\|` / `.?` / `orelse` |
| null test | `p == null` |
| many-item pointer | `[*]T` — no `.len`, supports `p + n`, **unchecked** |
| pointer to array | `*[N]T` — `&arr`, bounds-checked, coerces to `[]T` |
| slice from `[*]T` | `p[0..len]` |
| pointer from a slice | `s.ptr` |
| slice a pointer-to-array | `ap[1..]` |
| address as an integer | `@intFromPtr(p)` |
| integer as an address | `@ptrFromInt(addr)` |
| reinterpret | `@ptrCast(p)` + `@alignCast` when alignment grows |
| bytes of a value | `std.mem.asBytes(&x)` → `*[N]u8` |
| value from bytes | `std.mem.bytesAsValue(T, bytes)` |
| alignment of a type | `@alignOf(T)` |
| explicit alignment | `var x: T align(16) = ...;` |
| heap single value | `try gpa.create(T)` / `gpa.destroy(p)` |

Everything from the dangling-slice trap in lesson 3.2 applies here, more sharply:
a `*T` into a dead stack frame compiles fine and is just as broken.

---

Next: [3.6 — Enums](./06-enums.md)
