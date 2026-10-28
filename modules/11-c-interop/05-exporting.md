# 11.5 — Exporting to C

The other direction, and the more valuable one commercially: a Zig library that
C, Python, Go, Node or Rust can all call, because they all speak the C ABI.
Replacing one hot module inside a large C codebase is how Zig gets adopted
incrementally.

## `export fn`

`export fn` does three things: gives the function the C calling convention,
emits its symbol with an unmangled name, and keeps it alive through dead-code
elimination.

`zigmath.zig`:

```zig
const std = @import("std");

export fn zig_add(a: c_int, b: c_int) c_int {
    return a + b;
}

export fn zig_sum(xs: [*]const c_int, n: usize) c_long {
    var total: c_long = 0;
    for (xs[0..n]) |x| total += x;
    return total;
}

pub export fn zig_scale(p: *Point, k: c_int) callconv(.c) void {
    p.x *= k;
    p.y *= k;
}

pub const Point = extern struct { x: c_int, y: c_int };

export fn zig_version() [*:0]const u8 {
    return "zigmath 1.0";
}
```

`zigmath.h` — **you write this by hand**; Zig does not generate it:

```c
#ifndef ZIGMATH_H
#define ZIGMATH_H
#include <stddef.h>

typedef struct { int x; int y; } Point;

int zig_add(int a, int b);
long zig_sum(const int *xs, size_t n);
void zig_scale(Point *p, int k);
const char *zig_version(void);

#endif
```

`app.c`:

```c
#include <stdio.h>
#include "zigmath.h"

int main(void) {
    printf("zig_add(2,3)   = %d\n", zig_add(2, 3));

    int xs[4] = { 1, 2, 3, 4 };
    printf("zig_sum        = %ld\n", zig_sum(xs, 4));

    Point p = { 3, -4 };
    zig_scale(&p, 10);
    printf("zig_scale      = (%d, %d)\n", p.x, p.y);

    printf("zig_version    = %s\n", zig_version());
    return 0;
}
```

Build the static library, then compile the C program against it with `zig cc`:

```sh
zig build-lib zigmath.zig -lc
zig cc app.c libzigmath.a -o app
./app
```

```
zig_add(2,3)   = 5
zig_sum        = 10
zig_scale      = (30, -40)
zig_version    = zigmath 1.0
```

A C `main` driving Zig code. Nothing in `app.c` knows Zig exists.

| Zig | C header | Note |
|---|---|---|
| `export fn zig_add(a: c_int, b: c_int) c_int` | `int zig_add(int, int);` | |
| `xs: [*]const c_int, n: usize` | `const int *xs, size_t n` | slices can't cross — pass ptr + len |
| `p: *Point` with `extern struct` | `Point *p` | mutates C's struct in place |
| `-> [*:0]const u8` | `const char *` | points at Zig's static data — **C must not free it** |
| `-> c_long` | `long` | not `i64` — see 11.2 |

## `export fn` vs `callconv(.c)` vs `pub`

```zig
const std = @import("std");

export fn exported(x: c_int) c_int {
    return x + 1;
}

pub fn explicit(x: c_int) callconv(.c) c_int {
    return x + 2;
}

pub fn zigcall(x: c_int) c_int {
    return x + 3;
}

comptime {
    @export(&explicit, .{ .name = "explicit_c", .linkage = .strong });
}

pub fn main() void {
    std.debug.print("exported = {d}\n", .{exported(1)});
    std.debug.print("explicit = {d}\n", .{explicit(1)});
    std.debug.print("zigcall  = {d}\n", .{zigcall(1)});
    std.debug.print("callconv(exported) = {t}\n", .{@typeInfo(@TypeOf(exported)).@"fn".attrs.@"callconv"});
    std.debug.print("callconv(explicit) = {t}\n", .{@typeInfo(@TypeOf(explicit)).@"fn".attrs.@"callconv"});
    std.debug.print("callconv(zigcall)  = {t}\n", .{@typeInfo(@TypeOf(zigcall)).@"fn".attrs.@"callconv"});
}
```

```sh
zig run conv.zig
```

```
exported = 2
explicit = 3
zigcall  = 4
callconv(exported) = aarch64_aapcs_darwin
callconv(explicit) = aarch64_aapcs_darwin
callconv(zigcall)  = auto
```

> **Reflection note:** the calling convention lives at
> `@typeInfo(F).@"fn".attrs.@"callconv"` in 0.17. Older code uses
> `.@"fn".calling_convention`, which now fails with
> `no field named 'calling_convention' in struct 'lang.Type.Fn'`.

`export fn` and `callconv(.c)` produce the **same** calling convention — the
platform-specific C one, here `aarch64_aapcs_darwin`. A plain Zig function is
`auto`: the compiler may pass arguments however it likes, and that is not
something C can call.

| Form | Calling convention | Symbol emitted | Use for |
|---|---|---|---|
| `export fn f()` | C | **yes**, named `f` | the normal way to expose a function |
| `fn f() callconv(.c)` | C | no | **callbacks you pass *to* C** |
| `@export(&f, .{ .name = "g" })` | whatever `f` has | yes, named `g` | exporting under a different name |
| `pub fn f()` | `auto` | no | Zig-only |

`pub` and `export` are orthogonal: `pub` controls visibility to other *Zig*
code, `export` controls visibility to the *linker*. `export fn` with no `pub` is
fine, and common — the C world sees it, Zig importers don't.

**`callconv(.c)` without `export` is the callback case.** A function pointer you
hand to `qsort` or a C library's event loop must have the C convention but needs
no symbol of its own (lesson 11.1 shows this working).

> `callconv(.C)` with a capital C is what older code uses. Lowercase `.c` is the
> 0.17 spelling.

## Errors and optionals cannot cross

Zig's error unions and optionals-of-non-pointer have no C representation. The
compiler says so:

```zig
export fn bad_error(x: c_int) !c_int {
    if (x < 0) return error.Negative;
    return x;
}
pub fn main() void {}
```

```sh
zig run badexport.zig
```

```
badexport.zig:1:32: error: return type '!c_int' not allowed in function with calling convention 'aarch64_aapcs_darwin'
export fn bad_error(x: c_int) !c_int {
                               ^~~~~
```

So translate at the boundary, the same way you translate C error codes inward:

```zig
const std = @import("std");

const Err = enum(c_int) { ok = 0, negative = -1, overflow = -2 };

fn checkedDouble(x: c_int) !c_int {
    if (x < 0) return error.Negative;
    if (x > 1000) return error.Overflow;
    return x * 2;
}

export fn zig_double(x: c_int, out: *c_int) c_int {
    const v = checkedDouble(x) catch |e| return @intFromEnum(@as(Err, switch (e) {
        error.Negative => .negative,
        error.Overflow => .overflow,
    }));
    out.* = v;
    return @intFromEnum(Err.ok);
}

export fn zig_find(haystack: [*:0]const u8, needle: u8) isize {
    const s = std.mem.span(haystack);
    return if (std.mem.indexOfScalar(u8, s, needle)) |i| @intCast(i) else -1;
}

pub fn main() void {
    var out: c_int = 0;
    std.debug.print("zig_double(21)   rc={d} out={d}\n", .{ zig_double(21, &out), out });
    std.debug.print("zig_double(-1)   rc={d}\n", .{zig_double(-1, &out)});
    std.debug.print("zig_double(9999) rc={d}\n", .{zig_double(9999, &out)});
    std.debug.print("zig_find 'l'     = {d}\n", .{zig_find("hello", 'l')});
    std.debug.print("zig_find 'z'     = {d}\n", .{zig_find("hello", 'z')});
}
```

```sh
zig run errors.zig
```

```
zig_double(21)   rc=0 out=42
zig_double(-1)   rc=-1
zig_double(9999) rc=-2
zig_find 'l'     = 2
zig_find 'z'     = -1
```

| Zig type | C-ABI representation |
|---|---|
| `!T` | **illegal** — return a code, write `T` through an out-param |
| `?T` for non-pointer `T` | **illegal** — use a sentinel (`-1`) or an out-param + `bool` |
| `?*T`, `?[*]T` | **legal** — `null` is the zero address, so just `T*` |
| `[]T` | **illegal** — ptr + len as two parameters |
| `error{...}` | map to an `enum(c_int)` |
| `enum(c_int)` | `int` |
| `extern struct` | `struct` |
| plain `struct`, `union(enum)` | **illegal** |

`zig_find` returning `isize` with `-1` for "not found" is the C idiom; keep the
`?usize` version as a separate `pub fn` for Zig callers so neither audience pays
for the other's conventions.

## Memory across the boundary

The hardest part of a Zig library used from C is deciding who allocates. Three
designs, from best to worst:

```zig
const std = @import("std");

var gpa_state: std.heap.DebugAllocator(.{}) = .init;

export fn zig_fill(buf: [*]u8, cap: usize) usize {
    const text = "caller-provided buffer";
    if (cap < text.len) return 0;
    @memcpy(buf[0..text.len], text);
    return text.len;
}

export fn zig_alloc_greeting(name: [*:0]const u8) ?[*:0]u8 {
    const gpa = gpa_state.allocator();
    const s = std.mem.span(name);
    const out = gpa.allocSentinel(u8, s.len + 5, 0) catch return null;
    @memcpy(out[0..4], "hi, ");
    @memcpy(out[4 .. 4 + s.len], s);
    out[4 + s.len] = '!';
    return out.ptr;
}

export fn zig_free_greeting(p: ?[*:0]u8) void {
    const ptr = p orelse return;
    const gpa = gpa_state.allocator();
    gpa.free(std.mem.span(ptr));
}

pub fn main() void {
    var buf: [64]u8 = undefined;
    const n = zig_fill(&buf, buf.len);
    std.debug.print("zig_fill  -> {d} bytes: {s}\n", .{ n, buf[0..n] });
    std.debug.print("too small -> {d}\n", .{zig_fill(&buf, 4)});

    const g = zig_alloc_greeting("raj") orelse return;
    std.debug.print("allocated -> {s}\n", .{std.mem.span(g)});
    zig_free_greeting(g);

    std.debug.print("leaks: {t}\n", .{gpa_state.deinit()});
}
```

```sh
zig run alloc.zig
```

```
zig_fill  -> 22 bytes: caller-provided buffer
too small -> 0
allocated -> hi, raj!
leaks: ok
```

`leaks: ok` is the proof the paired free worked. Note `std.mem.span(ptr)` inside
`zig_free_greeting`: it recovers the length by scanning, because the pointer C
handed back carries no length. The `[:0]u8` that `span` returns is exactly what
`allocSentinel` produced, so `gpa.free` matches.

| Design | API shape | Verdict |
|---|---|---|
| **caller's buffer** | `size_t f(char *buf, size_t cap)` | **best** — no ownership question at all |
| **Zig allocates, Zig frees** | `T *f(); void f_free(T *)` | fine — **the free function is mandatory** |
| Zig allocates, C `free`s | `T *f()` | **broken** — Zig's allocator isn't `malloc`'s heap |

That third row is the bug to never write. `gpa.allocSentinel` memory passed to
C's `free()` corrupts the heap. If C must own it, allocate with
`std.c.malloc` so the heaps match — or just use design 1 or 2.

Also note the `var gpa_state` at file scope. A C caller has no place to store
your allocator, so a library either keeps global state like this or takes an
opaque context handle it created earlier — the `lib_init()` / `lib_deinit()`
pattern every C library has. Global `DebugAllocator` is fine for a demo; in a
real library, hand out a context so two users don't share one allocator.

## ABI stability

Once C code links against your library, your `extern struct` layout is a
published contract. Changing it breaks every binary already compiled against it,
with no error — just wrong field reads.

```zig
const std = @import("std");

const V1 = extern struct { x: c_int, y: c_int };
const V2 = extern struct { x: c_int, y: c_int, z: c_int };
const V3 = extern struct { x: c_int, z: c_int, y: c_int };

pub fn main() void {
    std.debug.print("V1 (shipped)   size={d} x@{d} y@{d}\n", .{ @sizeOf(V1), @offsetOf(V1, "x"), @offsetOf(V1, "y") });
    std.debug.print("V2 (appended)  size={d} x@{d} y@{d} z@{d}\n", .{ @sizeOf(V2), @offsetOf(V2, "x"), @offsetOf(V2, "y"), @offsetOf(V2, "z") });
    std.debug.print("V3 (reordered) size={d} x@{d} y@{d} z@{d}\n", .{ @sizeOf(V3), @offsetOf(V3, "x"), @offsetOf(V3, "y"), @offsetOf(V3, "z") });
}
```

```sh
zig run abi.zig
```

```
V1 (shipped)   size=8 x@0 y@4
V2 (appended)  size=12 x@0 y@4 z@8
V3 (reordered) size=12 x@0 y@8 z@4
```

- **V2 appended a field**: `x` and `y` kept their offsets. Old code reading only
  `x`/`y` still works — but anything that did `sizeof(V1)`-based allocation or
  array indexing is now wrong. Append is *safer*, not safe.
- **V3 reordered**: `y` moved from 4 to 8. Old binaries read `z` and call it
  `y`. Silent, total breakage.

| Change | Breaks old binaries? |
|---|---|
| append a field at the end | arrays and `sizeof` users, yes |
| reorder fields | **yes, silently** |
| change a field's type/width | **yes, silently** |
| rename a field | no (names aren't in the ABI) |
| add a new `export fn` | no |
| remove or rename an `export fn` | yes, link error (loud — the good kind) |
| change a function's signature | **yes, silently** |

How real C libraries cope:

| Technique | How |
|---|---|
| **opaque handles** | C only ever holds `MyThing *`; the struct is `opaque {}` on their side and your fields are free to change |
| size-in-struct | first field is `size_t struct_size`; the callee checks it |
| versioned symbols | `zig_init_v2`, keep `zig_init` working |
| accessor functions | `int thing_get_x(Thing *)` instead of `t->x` |

**The opaque-handle pattern is the right default** for anything you expect to
evolve. Export `create`/`destroy`/accessor functions, keep the struct private,
and you can restructure internals freely — which is the whole reason you wrote
the module in Zig in the first place.

## Checklist for a C-callable Zig library

| | |
|---|---|
| every exported function | `export fn`, C-ABI types only |
| every exported struct | `extern struct` |
| every error | an `enum(c_int)` return code + out-params |
| every allocation you hand out | a matching `*_free` export |
| every string you return | document whether C may free it |
| every callback you accept | `*const fn (...) callconv(.c) R` |
| anything likely to change | `opaque {}` + accessors |
| the header | hand-written, checked in, kept in sync |
| CI | one C program that calls every export, built with `zig cc` |

That last row is the one people skip. The `app.c` at the top of this lesson is
the whole test: if it compiles and runs, your header matches your exports. Make
it a `zig build` step and the compiler checks your ABI contract on every commit.

---

Module 11 complete. Next: [Module 12 — Concurrency & Async](../12-concurrency-async/)
