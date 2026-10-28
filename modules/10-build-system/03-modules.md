# 10.3 — Modules

Zig has two separate import mechanisms and conflating them is the usual source
of confusion:

| You write | Resolves to | Declared where |
|---|---|---|
| `@import("stats.zig")` | a **file**, path relative to the importing file | nowhere — just the path |
| `@import("mathx")` | a **module**, by name | `build.zig`, via `addImport` / `.imports` |

A *module* is a compilation unit: one root source file plus its own target,
optimize mode, and import table. Files reached by path `@import` are part of
whatever module their root belongs to. Module boundaries are where Zig decides
visibility, caching and compile flags.

## A 4-file project

```
build.zig
src/main.zig        root of the exe module
src/fmtx.zig        a file inside the exe module
lib/mathx.zig       root of the `mathx` module
lib/stats.zig       a file inside the mathx module
```

**`build.zig`**

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const mathx = b.addModule("mathx", .{
        .root_source_file = b.path("lib/mathx.zig"),
        .target = target,
        .optimize = optimize,
    });

    const exe_mod = b.createModule(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    exe_mod.addImport("mathx", mathx);

    const exe = b.addExecutable(.{ .name = "calc", .root_module = exe_mod });
    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);
    const run_step = b.step("run", "Run the app");
    run_step.dependOn(&run_cmd.step);

    const t = b.addTest(.{ .root_module = exe_mod });
    const test_step = b.step("test", "Run tests");
    test_step.dependOn(&b.addRunArtifact(t).step);

    const tlib = b.addTest(.{ .root_module = mathx });
    test_step.dependOn(&b.addRunArtifact(tlib).step);
}
```

**`src/main.zig`**

```zig
const std = @import("std");
const mathx = @import("mathx");
const fmtx = @import("fmtx.zig");

pub fn main() void {
    std.debug.print("{s}\n", .{fmtx.banner});
    std.debug.print("square(9) = {d}\n", .{mathx.square(9)});
    std.debug.print("mean = {d:.2}\n", .{mathx.mean(&.{ 2, 4, 9 })});
}

test "banner" {
    try std.testing.expect(fmtx.banner.len > 0);
}
```

**`src/fmtx.zig`**

```zig
pub const banner = "== calc ==";
```

**`lib/mathx.zig`**

```zig
const std = @import("std");
const stats = @import("stats.zig");

pub const mean = stats.mean;

pub fn square(x: i64) i64 {
    return x * x;
}

test "square" {
    try std.testing.expectEqual(@as(i64, 49), square(7));
}
```

**`lib/stats.zig`**

```zig
const std = @import("std");

pub fn mean(xs: []const f64) f64 {
    var total: f64 = 0;
    for (xs) |x| total += x;
    return total / @as(f64, @floatFromInt(xs.len));
}

test "mean" {
    try std.testing.expectEqual(@as(f64, 2.0), mean(&.{ 1, 2, 3 }));
}
```

```sh
zig build run
```

```
== calc ==
square(9) = 81
mean = 5.00
```

Three import styles, one program:

| In `src/main.zig` | Kind | Why it resolves |
|---|---|---|
| `@import("std")` | module | always available |
| `@import("mathx")` | module | `exe_mod.addImport("mathx", mathx)` |
| `@import("fmtx.zig")` | file | sits next to `main.zig` |

`mean` is re-exported: `stats.zig` is *inside* the `mathx` module, so
`src/main.zig` cannot reach it directly. Only what the module's **root file**
makes `pub` is visible across a module boundary. That line —
`pub const mean = stats.mean;` — is the module's public surface.

## The two failure modes

Drop the `addImport` line:

```sh
zig build
```

```
install
+- install calc
   +- compile exe calc debug native 1 errors
src/main.zig:2:23: error: no module named 'mathx' available within module 'root'
const mathx = @import("mathx");
                      ^~~~~~~
referenced by:
    main: src/main.zig:7:44
    callMain [inlined]: /opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/start.zig:788:64
    callMainWithArgs [inlined]: /opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/start.zig:729:20
    main: /opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/start.zig:754:28
    1 reference(s) hidden; use '-freference-trace=5' to see all references
```

Note `within module 'root'` — the exe's own module is always called `root`.

Now try reaching across a module boundary by path instead:

```zig
const mathx = @import("../lib/mathx.zig");
```

```sh
zig build
```

```
install
+- install calc
   +- compile exe calc debug native 1 errors
src/main.zig:2:23: error: import of file outside module path
const mathx = @import("../lib/mathx.zig");
                      ^~~~~~~~~~~~~~~~~~
error: 1 compilation errors
failed command: /opt/homebrew/Cellar/zig/0.17.0/bin/zig build-exe -Odebug --dep mathx -Mroot=src/main.zig -Odebug -Mmathx=lib/mathx.zig --cache-dir .zig-cache --global-cache-dir /Users/raj/.cache/zig --build-root . --name calc --zig-lib-dir /opt/homebrew/Cellar/zig/0.17.0/lib/zig/ --listen=-

Build Summary: 0/3 steps succeeded (1 failed)
```

A module's root file defines its directory, and path `@import` cannot escape it.
Read the `failed command` line — that is the graph flattened into one compiler
invocation: `-Mroot=src/main.zig`, `-Mmathx=lib/mathx.zig`, and `--dep mathx`
wiring root to it. Every failing step prints this, and it is the best debugging
tool the build system has.

## `addModule` vs `createModule`

| | Visible to | Use for |
|---|---|---|
| `b.addModule("name", .{...})` | this package **and anything depending on it** | your library's public API |
| `b.createModule(.{...})` | this package only | the exe's root, internal helpers |

Both return `*std.Build.Module`. If you are publishing a package, the module
others `@import` must come from `addModule` — see 10.4.

## Wiring imports: two spellings

After the fact:

```zig
const exe_mod = b.createModule(.{
    .root_source_file = b.path("src/main.zig"),
    .target = target,
    .optimize = optimize,
});
exe_mod.addImport("mathx", mathx);
```

Inline at creation:

```zig
const exe_mod = b.createModule(.{
    .root_source_file = b.path("src/main.zig"),
    .target = target,
    .optimize = optimize,
    .imports = &.{
        .{ .name = "mathx", .module = mathx },
    },
});
```

Identical result. Use `addImport` when the module you need doesn't exist yet —
import tables are allowed to be cyclical, so `a.addImport("b", b)` and
`b.addImport("a", a)` together are legal.

Renaming on import is free, which is how you resolve collisions between two
packages that both named their module `json`:

```zig
.imports = &.{
    .{ .name = "m", .module = mathx },
}
```

```zig
const m = @import("m");
```

## `root_module` vs added imports

```zig
const exe = b.addExecutable(.{ .name = "calc", .root_module = exe_mod });
```

`root_module` is the one module that gets compiled as the entry point — it must
contain `pub fn main`. Everything in its `.imports` table is compiled too, but
as a dependency. Each `addExecutable`/`addTest` has exactly one root.

Which is why **one test binary tests exactly one module**. The `build.zig` above
creates two `addTest` nodes, one rooted at `exe_mod` and one at `mathx`, both
hanging off the same `test` step:

```sh
zig build test --summary all
```

```
Build Summary: 5/5 steps succeeded; 2/2 tests passed
test success
+- run test 1 pass (1 total) 425ms MaxRSS:2M
|  +- compile test debug native success 1s MaxRSS:258M
+- run test 1 pass (1 total) 840ms MaxRSS:2M
   +- compile test debug native success 1s MaxRSS:262M
```

Two binaries, two tests. But there are **three** `test` blocks in the project —
`stats.zig`'s `"mean"` test never ran.

### Tests only run in files the compiler reached

`mathx.zig` uses `stats.mean` but never forces `stats.zig` to be analyzed as a
whole. Add a bare `test` block that references it:

**`lib/mathx.zig`**

```zig
const std = @import("std");
const stats = @import("stats.zig");

pub const mean = stats.mean;

pub fn square(x: i64) i64 {
    return x * x;
}

test {
    _ = stats;
}

test "square" {
    try std.testing.expectEqual(@as(i64, 49), square(7));
}
```

```sh
zig build test --summary all
```

```
Build Summary: 5/5 steps succeeded; 4/4 tests passed
test success
+- run test 1 pass (1 total) 422ms MaxRSS:2M
|  +- compile test debug native success 1s MaxRSS:258M
+- run test 3 pass (3 total) 830ms MaxRSS:2M
   +- compile test debug native success 1s MaxRSS:265M
```

2 → 4 tests. (The bare `test {}` block counts as one.) The `_ = stats;` is a
comptime reference that drags the whole file into analysis. Put one in every
module root that has sub-files, or silently lose test coverage — this is the most
common way a Zig project ends up with tests that never run.

## The dependency graph

```
exe (addExecutable)
 └── exe_mod ──────── root, "root"
      ├── src/fmtx.zig          (file, same module)
      ├── std                   (module)
      └── mathx ──────────────  (module)
           └── lib/stats.zig    (file, same module)
```

Module edges are explicit in `build.zig`. File edges are implicit in the source.
The graph may contain cycles between modules; files inside one module may not
escape its directory.

---

Next: [10.4 — Dependencies](./04-dependencies.md)
