# 11.4 — Linking Libraries

Zig is a C compiler as well as a Zig compiler. There is no `Makefile`, no
`CMakeLists.txt` and no separate build of the C half — you hand `.c` files to
the same command that builds your Zig.

## Compiling C alongside Zig

Three files, one command.

`mylib.h`:

```c
int add_ints(int a, int b);
```

`mylib.c`:

```c
#include "mylib.h"

int add_ints(int a, int b) {
    return a + b;
}
```

`main.zig`:

```zig
const std = @import("std");

extern fn add_ints(a: c_int, b: c_int) c_int;

pub fn main() void {
    std.debug.print("add_ints(3,4)={d}\n", .{add_ints(3, 4)});
}
```

```sh
zig build-exe main.zig mylib.c -lc
./main
```

```
add_ints(3,4)=7
```

Note what is **not** there: no `#include "mylib.h"` on the Zig side. Zig never
reads the header — you wrote the signature in the `extern fn` instead. The
header exists for the C compiler and for human readers.

> **The first libc-linked build is slow** — well over two minutes, because Zig
> compiles its libc shim from source. Every build after that hits the cache and
> takes under a second. Don't conclude something is broken on the first run.

| Command | Produces |
|---|---|
| `zig build-exe main.zig a.c b.c -lc` | an executable |
| `zig run main.zig a.c -lc` | build and run in one step |
| `zig test main.zig a.c -lc` | run the `test` blocks against C |
| `zig build-lib main.zig a.c -lc` | a static `.a` (11.5) |
| `zig build-lib -dynamic …` | a `.so` / `.dylib` |

## `-lc`, `-lm`, and friends

```zig
const std = @import("std");

extern fn memcmp(a: *const anyopaque, b: *const anyopaque, n: usize) c_int;
extern fn sqrt(x: f64) f64;

pub fn main() void {
    const a = "zig rocks";
    const b = "zig rolls";
    std.debug.print("memcmp     = {d}\n", .{memcmp(a, b, a.len)});
    std.debug.print("memcmp(4)  = {d}\n", .{memcmp(a, b, 4)});
    std.debug.print("sqrt(2)    = {d:.6}\n", .{sqrt(2.0)});
}
```

```sh
zig run -lc slice2c.zig
```

```
memcmp     = -9
memcmp(4)  = 0
sqrt(2)    = 1.414214
```

On this machine the same program links **with no flags at all**:

```sh
zig run slice2c.zig
```

```
memcmp     = -9
memcmp(4)  = 0
sqrt(2)    = 1.414214
```

That is a macOS-specific accident: the system libc is always in the link, and
there is no separate libm — the math functions live in libSystem. **Do not rely
on it.** On Linux the same program fails at link time without `-lc`, and `sqrt`
needs `-lm` as well. Always pass the flags you logically depend on:

| Flag | Library | Needed on |
|---|---|---|
| `-lc` | libc | **always pass it**; required on Linux, implicit on macOS |
| `-lm` | libm (math) | Linux; a no-op on macOS |
| `-lpthread` | threads | older Linux; now folded into glibc |
| `-ldl` | `dlopen` family | Linux |
| `-lz`, `-lsqlite3`, … | a named system library | wherever it's installed |
| `-L/path` | extra library search dir | |
| `-I/path` | extra header search dir (for your `.c` files) | |
| `-DNAME=val` | a preprocessor define | |

`-lm` alone works fine here too:

```zig
const std = @import("std");
extern fn cbrt(x: f64) f64;
pub fn main() void {
    std.debug.print("cbrt(27) = {d}\n", .{cbrt(27.0)});
}
```

```sh
zig run -lm m.zig
```

```
cbrt(27) = 3
```

## `build.zig` — the 0.17 module form

Command-line flags stop scaling at the second `.c` file. The build system is
where real projects live, and **its API changed in 0.17**: you build a
`Module` first, then hand it to `addExecutable` as `root_module`. The old form
(passing `root_source_file`/`target`/`optimize` directly to `addExecutable`) is
gone, and C options now live on the *module*, not the compile step.

```
bld/
├── build.zig
├── csrc/
│   ├── geom.c
│   └── geom.h
└── src/
    └── main.zig
```

`csrc/geom.h`:

```c
#ifndef GEOM_H
#define GEOM_H

typedef struct { double x, y; } Vec2;

double vec2_len(Vec2 v);
int geom_scale_factor(void);

#endif
```

`csrc/geom.c`:

```c
#include <math.h>
#include "geom.h"

double vec2_len(Vec2 v) {
    return sqrt(v.x * v.x + v.y * v.y);
}

int geom_scale_factor(void) {
#ifdef FAST_MODE
    return 2;
#else
    return 1;
#endif
}
```

`src/main.zig`:

```zig
const std = @import("std");

const Vec2 = extern struct { x: f64, y: f64 };

extern fn vec2_len(v: Vec2) f64;
extern fn geom_scale_factor() c_int;
extern fn ceil(x: f64) f64;

pub fn main() void {
    const v: Vec2 = .{ .x = 3, .y = 4 };
    std.debug.print("vec2_len({d},{d})   = {d}\n", .{ v.x, v.y, vec2_len(v) });
    std.debug.print("geom_scale_factor  = {d}\n", .{geom_scale_factor()});
    std.debug.print("libm ceil(2.1)     = {d}\n", .{ceil(2.1)});
}
```

`build.zig`:

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const mod = b.createModule(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
        .link_libc = true,
    });

    mod.addIncludePath(b.path("csrc"));
    mod.addCSourceFiles(.{
        .files = &.{"csrc/geom.c"},
        .flags = &.{ "-std=c11", "-DFAST_MODE=1" },
    });

    const exe = b.addExecutable(.{
        .name = "geomdemo",
        .root_module = mod,
    });

    b.installArtifact(exe);

    const run = b.addRunArtifact(exe);
    const run_step = b.step("run", "Run the demo");
    run_step.dependOn(&run.step);
}
```

```sh
zig build run
```

```
vec2_len(3,4)   = 5
geom_scale_factor  = 2
libm ceil(2.1)     = 3
```

`geom_scale_factor` returned **2**, which proves `-DFAST_MODE=1` reached the C
compiler — the `#else` branch would have returned 1.

| Old (0.13–0.15 tutorials) | 0.17.0 |
|---|---|
| `b.addExecutable(.{ .root_source_file = …, .target = …, .optimize = … })` | `b.createModule(.{ … })` then `.root_module = mod` |
| `exe.linkLibC()` | `.link_libc = true` in `createModule` |
| `exe.addCSourceFile(.{ .file = … })` | `mod.addCSourceFiles(.{ .files = &.{…} })` |
| `exe.addIncludePath(…)` | `mod.addIncludePath(…)` |
| `exe.linkSystemLibrary("z")` | `mod.linkSystemLibrary("z", .{})` |
| `exe.defineCMacro("X", "1")` | `"-DX=1"` in `.flags` |

The rule: **anything about *how code is compiled* belongs on the module;
anything about *what artifact is produced* belongs on the compile step.** C
sources, include paths, libc and system libraries are all compilation concerns.

### The C options, in detail

| Call | Does |
|---|---|
| `mod.addCSourceFiles(.{ .files = &.{"a.c","b.c"}, .flags = &.{"-std=c11"} })` | compile these with these flags |
| `mod.addCSourceFiles(.{ .root = b.path("csrc"), .files = &.{"a.c"} })` | paths relative to `root` |
| `mod.addIncludePath(b.path("include"))` | `-I` for your own headers |
| `mod.addSystemIncludePath(.{ .cwd_relative = "/opt/homebrew/include" })` | `-isystem`, suppresses warnings from it |
| `mod.addLibraryPath(.{ .cwd_relative = "/opt/homebrew/lib" })` | `-L` |
| `mod.linkSystemLibrary("z", .{})` | `-lz`, resolved at build time |
| `.link_libc = true` | `-lc` |
| `.link_libcpp = true` | link libc++ for `.cpp` sources |

`.flags` is where defines go: `"-DVERSION=2"`, `"-DNDEBUG"`, `"-DLOG(x)="`.
Pass `-std=` explicitly — don't inherit whatever the default happens to be.

## `linkSystemLibrary` — a real system library

`linkSystemLibrary` resolves against the libraries actually installed on the
machine. zlib ships with macOS, so this builds with no setup:

`main.zig`:

```zig
const std = @import("std");

extern fn compress2(dest: [*]u8, dest_len: *c_ulong, src: [*]const u8, src_len: c_ulong, level: c_int) c_int;
extern fn uncompress(dest: [*]u8, dest_len: *c_ulong, src: [*]const u8, src_len: c_ulong) c_int;

pub fn main() !void {
    const src = "the quick brown fox jumps over the lazy dog, repeatedly and at length";
    var comp: [256]u8 = undefined;
    var comp_len: c_ulong = comp.len;
    if (compress2(&comp, &comp_len, src, src.len, 9) != 0) return error.CompressFailed;

    var out: [256]u8 = undefined;
    var out_len: c_ulong = out.len;
    if (uncompress(&out, &out_len, &comp, comp_len) != 0) return error.DecompressFailed;

    std.debug.print("original     = {d} bytes\n", .{src.len});
    std.debug.print("compressed   = {d} bytes\n", .{comp_len});
    std.debug.print("round trip   = {d} bytes, equal={}\n", .{ out_len, std.mem.eql(u8, src, out[0..out_len]) });
}
```

`build.zig`:

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const mod = b.createModule(.{
        .root_source_file = b.path("main.zig"),
        .target = b.standardTargetOptions(.{}),
        .optimize = b.standardOptimizeOption(.{}),
        .link_libc = true,
    });
    mod.linkSystemLibrary("z", .{});

    const exe = b.addExecutable(.{ .name = "zdemo", .root_module = mod });
    b.installArtifact(exe);

    const run = b.addRunArtifact(exe);
    b.step("run", "Run it").dependOn(&run.step);
}
```

```sh
zig build run
```

```
original     = 69 bytes
compressed   = 69 bytes
round trip   = 69 bytes, equal=true
```

A working wrap of a real shared library in about twenty lines. Two things in
that output are worth a second look:

- **69 bytes "compressed" to 69 bytes.** zlib's header plus checksum costs more
  than this short string saves. The round trip is byte-identical, so the wrap is
  correct — compression just has a floor. Never test a codec wrapper on a
  53-byte input and conclude it works *well*.
- **`c_ulong` for the length in/out parameters.** zlib's API uses `uLongf`,
  which is `unsigned long` — 64 bits here, **32 on Windows**. `*u64` would link
  and then corrupt the stack on win64. This is the `c_long` trap from 11.2
  appearing in live code.

Note `comp_len` is `var` and passed as `*c_ulong`: zlib reads the capacity and
writes back the actual length. That in/out convention is everywhere in C.

## Static vs dynamic

```sh
zig build-lib mylib.zig          # libmylib.a  — static
zig build-lib -dynamic mylib.zig # libmylib.dylib / .so — dynamic
```

| | Static (`.a` / `.lib`) | Dynamic (`.so` / `.dylib` / `.dll`) |
|---|---|---|
| Linked | at build time, copied into the binary | at load time, by the OS |
| Binary size | larger | smaller |
| Deployment | one file | the `.so` must be present and found |
| Upgrades | rebuild | replace the library |
| `linkSystemLibrary` picks | whichever it finds (`.{ .preferred_link_mode = .static }` to force) | |
| Cross-compiling | **easy** | needs the target's libraries |
| Zig's default for libc | dynamic on macOS; static musl when you target `-musl` | |

For a self-contained backend binary, Zig's headline trick is static musl:

Taking the `main.zig` + `mylib.c` pair from the top of this lesson:

```sh
zig build-exe main.zig mylib.c -lc -target x86_64-linux-musl -femit-bin=linuxprog
file linuxprog
```

```
linuxprog: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, with debug_info, not stripped
```

An x86-64 Linux ELF, **statically linked**, built on an aarch64 Mac — with no
cross-toolchain installed, and the C file cross-compiled too, because Zig ships
musl's source and clang and builds both for you. A single file you can drop into
a scratch container. This is the capability that gets Zig into production
backends, and it is why `-target` is worth learning:

| Target | Gives you |
|---|---|
| `x86_64-linux-musl` | fully static Linux binary, no libc dependency |
| `x86_64-linux-gnu` | dynamic glibc; pin the version with `-gnu.2.28` |
| `aarch64-linux-musl` | static ARM Linux (Graviton, Pi) |
| `x86_64-windows-gnu` | a `.exe` |
| `native` | this machine (the default) |

Cross-compiling **C** works the same way — `zig cc -target x86_64-linux-musl f.c`
— which is why some projects adopt `zig cc` without writing any Zig at all.

Add it to `build.zig` with `b.standardTargetOptions(.{})`, which both examples
above already have, then:

```sh
zig build -Dtarget=x86_64-linux-musl -Doptimize=ReleaseFast
```

## A wrapping checklist

| Step | Do |
|---|---|
| 1 | `zig translate-c the_header.h > raw.zig` and **read** it (11.1) |
| 2 | copy the handful of decls you need into your own file, replacing `[*c]` (11.2) |
| 3 | add the C sources or `linkSystemLibrary` to `build.zig` |
| 4 | write a test asserting `@sizeOf`/`@offsetOf` match C's `sizeof`/`offsetof` (11.2) |
| 5 | wrap each call in a Zig function returning `!T` instead of an error code |
| 6 | decide and document who frees what (11.3) |
| 7 | run the test suite under `std.testing.allocator` so leaks fail the build |

Step 5 is what makes a wrapper worth having. `compress2` returning `c_int` is
C's design; your users should see an error set and a slice:

```zig
const std = @import("std");

extern fn compress2(dest: [*]u8, dest_len: *c_ulong, src: [*]const u8, src_len: c_ulong, level: c_int) c_int;

fn compress(dest: []u8, src: []const u8, level: c_int) ![]u8 {
    var n: c_ulong = dest.len;
    return switch (compress2(dest.ptr, &n, src.ptr, src.len, level)) {
        0 => dest[0..n],
        -5 => error.BufferTooSmall,
        -4 => error.OutOfMemory,
        else => error.CompressFailed,
    };
}

pub fn main() !void {
    var big: [256]u8 = undefined;
    const ok = try compress(&big, "hello hello hello hello hello hello", 9);
    std.debug.print("ok   -> {d} bytes\n", .{ok.len});

    var tiny: [4]u8 = undefined;
    std.debug.print("tiny -> {!d}\n", .{if (compress(&tiny, "xyzabcdefghijklmnop", 9)) |s| s.len else |e| e});
}
```

```sh
zig build-exe wrap.zig -lc -lz
./wrap
```

```
ok   -> 17 bytes
tiny -> error.BufferTooSmall
```

Error codes convert to an error set once, at the boundary, instead of being
checked at every call site forever — and the caller gets a `[]u8` of the right
length instead of an in/out `c_ulong` they have to remember to read back.

---

Next: [11.5 — Exporting to C](./05-exporting.md)
