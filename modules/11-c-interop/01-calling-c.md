# 11.1 — Calling C

## ⚠️ `@cImport` has been removed

Every C-interop tutorial you will find online opens with `@cImport` +
`@cInclude`. In 0.17.0 that code does not compile — the builtin is gone:

```zig
const std = @import("std");
const c = @cImport({
    @cInclude("string.h");
});
pub fn main() void {
    std.debug.print("{d}\n", .{c.strlen("hi")});
}
```

```sh
zig run cimp.zig
```

```
cimp.zig:2:11: error: invalid builtin function: '@cImport'
const c = @cImport({
          ^~~~~~~~
```

Not a deprecation warning. `@cImport` is not a builtin any more, so the error
arrives before Zig even looks at `@cInclude`. The compiler no longer embeds a C
parser that runs at compile time; translation is an explicit, separate step now.

| What tutorials show | 0.17.0 |
|---|---|
| `const c = @cImport({ @cInclude("x.h") });` | **gone** — declare `extern fn` yourself |
| `c.strlen(s)` | `extern fn strlen(s: [*:0]const u8) usize;` |
| rely on `@cImport` for libc | `std.c` — 300 libc functions already declared |
| `@cImport` for your own header | `zig translate-c x.h > x.zig`, then `@import("x.zig")` |
| `@cDefine("FOO", "1")` | `-DFOO=1` in `addCSourceFiles` flags (lesson 11.4) |
| `@cImport` auto-linked libc | `-lc` on the command line, `.link_libc = true` in `build.zig` |

The replacement is more typing and strictly more honest: you see the exact
signature you are calling, instead of whatever a C preprocessor guessed.

## `extern fn` + `-lc`

The primary approach now. Declare the symbol, link libc, call it.

```zig
const std = @import("std");

extern fn strlen(s: [*:0]const u8) usize;
extern fn abs(n: c_int) c_int;
extern fn printf(fmt: [*:0]const u8, ...) c_int;

pub fn main() !void {
    const s: [*:0]const u8 = "hello";
    std.debug.print("strlen={d} abs={d}\n", .{ strlen(s), abs(-5) });
    _ = printf("from C printf: %d\n", @as(c_int, 42));
}
```

```sh
zig run -lc ext.zig
```

```
strlen=5 abs=5
from C printf: 42
```

Three things in that output worth noticing:

- **`printf`'s line comes last** even though it is the last statement — it is
  also on a different buffer. `std.debug.print` writes to stderr unbuffered; C's
  `printf` writes to C's own stdout buffer, flushed at exit. Never interleave
  the two and expect ordering.
- **A variadic needs explicit types.** `@as(c_int, 42)` is not optional:
  `printf("%d\n", 42)` passes a `comptime_int` with no ABI size and fails to
  compile.
- **`"hello"` coerced to `[*:0]const u8`.** Zig string literals are `[:0]const u8`
  — already NUL-terminated — so they decay to a C string pointer for free.
  Lesson 11.3 covers the cases where they don't.

## There is no linker magic — just symbols

An `extern fn` is a promise that some symbol exists. Break the promise and the
**linker** complains, not the compiler:

```zig
const std = @import("std");
extern fn no_such_function_anywhere(x: c_int) c_int;
pub fn main() void {
    std.debug.print("{d}\n", .{no_such_function_anywhere(1)});
}
```

```sh
zig run miss.zig
```

```
error: undefined symbol: _no_such_function_anywhere
    note: referenced by /Users/raj/.cache/zig/o/b1212b7ca0ae7df1bc6ee838d028417f/miss_zcu.o:_miss.main
```

Note the leading underscore: that is the Mach-O name mangling for C symbols on
macOS. On Linux you would see `no_such_function_anywhere` bare.

**Zig does not check your declaration against the real one.** If you declare
`extern fn strlen(s: [*:0]const u8) c_int` — wrong return width — it links
happily and you read garbage in the high bits. The declaration *is* the contract;
getting it wrong is on you. That is the one real cost of losing `@cImport`.

> **macOS note:** `-lc` is often unnecessary here, because the system libc is
> always in the link. `zig run slice2c.zig` with no flags found `memcmp` and
> `sqrt` just fine. On Linux it will not. **Always pass `-lc`** so your code
> builds on both.

## `std.c` — don't hand-write what the stdlib already declares

`std.c` ships 300 `pub extern "c" fn` declarations, written by people who read
the headers:

```sh
grep -cE '^pub extern "c" fn' /opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/c.zig
```

```
300
```

```zig
const std = @import("std");
const c = std.c;

pub fn main() !void {
    std.debug.print("abs(-9)      = {d}\n", .{c.abs(-9)});
    std.debug.print("getenv(HOME) = {s}\n", .{std.mem.span(c.getenv("HOME") orelse return)});

    const f = c.fopen("/tmp/zc11_stdc.txt", "w") orelse return error.OpenFailed;
    const msg = "written by libc fwrite\n";
    _ = c.fwrite(msg, 1, msg.len, f);
    _ = c.fclose(f);

    var buf: [64]u8 = undefined;
    const g = c.fopen("/tmp/zc11_stdc.txt", "r") orelse return error.OpenFailed;
    const n = c.fread(&buf, 1, buf.len, g);
    _ = c.fclose(g);
    std.debug.print("read back    = {s}", .{buf[0..n]});

    const mem = c.malloc(16) orelse return error.OutOfMemory;
    defer c.free(mem);
    const bytes: [*]u8 = @ptrCast(mem);
    bytes[0] = 7;
    std.debug.print("malloc'd[0]  = {d}\n", .{bytes[0]});
}
```

```sh
zig run -lc stdc.zig
```

```
abs(-9)      = 9
getenv(HOME) = /Users/raj
read back    = written by libc fwrite
malloc'd[0]  = 7
```

Look at the signatures the stdlib chose — they are a style guide for your own
externs:

```zig
pub extern "c" fn fopen(noalias filename: [*:0]const u8, noalias modes: [*:0]const u8) ?*FILE;
pub extern "c" fn fclose(stream: *FILE) c_int;
pub extern "c" fn fwrite(noalias ptr: [*]const u8, size_of_type: usize, item_count: usize, noalias stream: *FILE) usize;
pub extern "c" fn getenv(name: [*:0]const u8) ?[*:0]u8;
pub extern "c" fn malloc(usize) ?*anyopaque;
pub extern "c" fn free(?*anyopaque) void;
```

| Choice | Why it matters |
|---|---|
| `?*FILE`, `?[*:0]u8`, `?*anyopaque` | C returns NULL on failure → **`?`**, so `orelse` is forced on you |
| `*FILE` on `fclose` | non-null required; passing NULL is UB in C, so the type forbids it |
| `[*:0]const u8` not `[]const u8` | a C string is a pointer, not a pointer+length |
| `noalias` | the `restrict` keyword — lets Zig optimize around it |

Check before you write an extern:

```zig
const std = @import("std");
pub fn main() void {
    inline for (.{ "fopen", "malloc", "qsort", "dlopen", "pthread_create" }) |name| {
        std.debug.print("{s:<16} {}\n", .{ name, @hasDecl(std.c, name) });
    }
}
```

```
fopen            true
malloc           true
qsort            false
dlopen           true
pthread_create   true
```

`qsort` is **not** there — so that one you declare yourself, as the callback
example below does. `@hasDecl` is the one-line check that tells you which.

## Variadics

C variadics are supported in `extern` declarations with `...`, but every
argument must have a concrete ABI type:

```zig
const std = @import("std");

extern fn printf(fmt: [*:0]const u8, ...) c_int;
extern fn snprintf(buf: [*]u8, n: usize, fmt: [*:0]const u8, ...) c_int;

pub fn main() void {
    _ = printf("int %d, double %f, string %s\n", @as(c_int, 7), @as(f64, 2.5), "zig");

    var buf: [64]u8 = undefined;
    const n = snprintf(&buf, buf.len, "%s-%03d", "id", @as(c_int, 42));
    std.debug.print("snprintf wrote {d} bytes: {s}\n", .{ n, buf[0..@intCast(n)] });
}
```

```sh
zig run -lc varargs.zig
```

```
snprintf wrote 6 bytes: id-042
int 7, double 2.500000, string zig
```

| Zig side | C format | Note |
|---|---|---|
| `@as(c_int, n)` | `%d` | bare `42` won't compile — no ABI size |
| `@as(f64, x)` | `%f` | C promotes `float` to `double` in varargs |
| `"literal"` or `[*:0]const u8` | `%s` | a `[]const u8` **slice** won't compile |
| `@as(usize, n)` | `%zu` | |

`snprintf` returns the length it *wanted* to write, so `@intCast(n)` can exceed
your buffer on truncation — clamp it in real code.

**You cannot write a Zig function that *accepts* C varargs.** Only `extern`
declarations may use `...`. If a C library hands you a `va_list`, you need a
small C shim that calls the `v`-suffixed variant (`vsnprintf`) for you.

## Calling C through a function pointer

`qsort` is the classic test of whether your callback ABI is right:

```zig
const std = @import("std");

extern fn qsort(base: *anyopaque, n: usize, size: usize, cmp: *const fn (*const anyopaque, *const anyopaque) callconv(.c) c_int) void;

fn cmpInts(a: *const anyopaque, b: *const anyopaque) callconv(.c) c_int {
    const x: *const c_int = @ptrCast(@alignCast(a));
    const y: *const c_int = @ptrCast(@alignCast(b));
    return x.* - y.*;
}

pub fn main() void {
    var xs = [_]c_int{ 42, 7, 19, 3, 25 };
    std.debug.print("before = {any}\n", .{xs});
    qsort(&xs, xs.len, @sizeOf(c_int), &cmpInts);
    std.debug.print("after  = {any}\n", .{xs});
}
```

```sh
zig run -lc qs.zig
```

```
before = { 42, 7, 19, 3, 25 }
after  = { 3, 7, 19, 25, 42 }
```

**`callconv(.c)` on the callback is mandatory.** Without it the function uses
Zig's own calling convention and C will call it with arguments in the wrong
places — a crash or silent nonsense, with no diagnostic. Lesson 11.5 goes
further on this.

## `zig translate-c` — the explicit replacement for `@cImport`

`@cImport` was `translate-c` running invisibly at compile time. The tool is
still here; now you run it yourself and keep the output.

`point.h`:

```c
#define MAX_POINTS 16

typedef struct {
    int x;
    int y;
} Point;

enum Color { COLOR_RED = 0, COLOR_GREEN = 1, COLOR_BLUE = 4 };

int point_sum(Point p);
const char *color_name(enum Color c);
```

```sh
zig translate-c point.h > point.zig
wc -l point.zig
```

```
     438 point.zig
```

438 lines for a 10-line header — almost all of it predefined macros
(`__STDC__`, `__VERSION__`, and friends). The part you care about is at the top:

```zig
const __root = @This();
pub const __builtin = @import("std").zig.c_translation.builtins;
pub const __helpers = @import("std").zig.c_translation.helpers;
pub const Point = extern struct {
    x: c_int = 0,
    y: c_int = 0,
    pub const point_sum = __root.point_sum;
    pub const sum = __root.point_sum;
};
pub const COLOR_RED: c_int = 0;
pub const COLOR_GREEN: c_int = 1;
pub const COLOR_BLUE: c_int = 4;
pub const enum_Color = c_uint;
pub extern fn point_sum(p: Point) c_int;
pub extern fn color_name(c: enum_Color) [*c]const u8;
```

and the bottom:

```zig
pub const MAX_POINTS = @as(c_int, 16);
pub const Color = enum_Color;
```

Read that output closely — it tells you a lot about how C maps onto Zig:

| In the header | What `translate-c` produced | Why |
|---|---|---|
| `typedef struct { int x, y; }` | `extern struct { x: c_int = 0, … }` | C layout, never Zig's reordering (11.2) |
| `enum Color` | `enum_Color = c_uint` — a **plain integer alias** | C enums aren't exhaustive; values beyond the list are legal |
| `COLOR_BLUE = 4` | `pub const COLOR_BLUE: c_int = 4;` | loose constants, not enum members |
| `const char *` | `[*c]const u8` | it can't know whether NULL is valid (11.2) |
| `#define MAX_POINTS 16` | `pub const MAX_POINTS = @as(c_int, 16);` | object-like macros survive |
| `int point_sum(Point)` | also `Point.sum` and `Point.point_sum` | a method shim, since the first arg is `Point` |

The function-like macros in your header will **not** all survive — anything the
translator can't express becomes a `@compileError` stub you have to replace by
hand.

**Treat `translate-c` as a one-time generator, not a build step.** Run it, read
the output, and copy the handful of declarations you actually need into your own
`.zig` file — with real types instead of `[*c]`. That is the whole workflow that
replaced `@cImport`, and the hand-written result is better than what `@cImport`
gave you.

## `zig cc` and `zig c++`

Zig also ships as a drop-in C/C++ compiler:

```sh
zig cc --version
```

```
Homebrew clang version 22.1.8
Target: aarch64-apple-macosx26.5.2-unknown
Thread model: posix
```

```sh
zig c++ --version
```

```
Homebrew clang version 22.1.8
Target: aarch64-apple-macosx26.5.2-unknown
```

> On this machine `zig` comes from Homebrew, which builds it against the system
> clang rather than bundling its own — so `zig cc` reports Homebrew's clang.
> A Zig downloaded from ziglang.org bundles clang and prints its own version.
> Either way the interface is the same: `zig cc` accepts clang flags and can
> cross-compile with `-target`.

Where it earns its place:

| Command | Use |
|---|---|
| `zig cc file.c -o prog` | drop-in `cc` replacement |
| `zig cc -target x86_64-linux-gnu f.c` | cross-compile C from macOS, no toolchain to install |
| `CC="zig cc" ./configure` | build an autotools project with it |
| `zig build-exe main.zig f.c -lc` | **the Zig way** — same build, one command (11.4) |

You will use `zig cc` in lesson 11.5, to compile a C program that calls *into*
Zig.

## Which approach when

| Situation | Approach |
|---|---|
| a handful of libc calls | **`std.c`** — already declared, correctly |
| libc function not in `std.c` | `extern fn` + `-lc` |
| your own `.c` files | `extern fn` + `zig build-exe main.zig f.c -lc` (11.4) |
| a big third-party header | `zig translate-c h.h`, read it, copy what you need |
| system shared library | `extern fn` + `linkSystemLibrary` in `build.zig` (11.4) |
| C calling your Zig | `export fn` (11.5) |
| C varargs you must *accept* | a C shim calling `vsnprintf`-style functions |

---

Next: [11.2 — C Types](./02-c-types.md)
