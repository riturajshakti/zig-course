# 10.1 — `build.zig`

`build.zig` is **not** a build script. It is a Zig program that *constructs a
graph of steps* and then exits. A separate runner walks that graph and decides
what actually executes.

That distinction is the whole module. Here it is, proven:

**`build.zig`**

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    std.debug.print("build() runs: declaring the graph\n", .{});
    const exe = b.addExecutable(.{
        .name = "demo",
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            .target = b.standardTargetOptions(.{}),
            .optimize = b.standardOptimizeOption(.{}),
        }),
    });
    b.installArtifact(exe);
    std.debug.print("build() returned: nothing has compiled yet\n", .{});
}
```

**`src/main.zig`**

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("the program itself\n", .{});
}
```

```sh
zig build
```

```
build() runs: declaring the graph
build() returned: nothing has compiled yet
```

Both prints land *before* a single byte is compiled. `addExecutable` compiled
nothing; it appended a node. `installArtifact` copied nothing; it appended
another node and wired it to the default step.

Run it a second time:

```sh
zig build
```

```
```

Silence. The build-graph *construction* is itself cached — `build()` did not
even run. Change an option and it runs again:

```sh
zig build -Doptimize=ReleaseFast
```

```
build() runs: declaring the graph
build() returned: nothing has compiled yet
```

| Mental model | npm / cargo / go | Zig |
|---|---|---|
| config file | declarative data (`package.json`, `Cargo.toml`) | a **program** that emits a graph |
| when it executes | parsed once | `build()` runs, returns, then the runner works |
| parallelism | tool-specific | automatic — anything without a `dependOn` edge |
| incremental unit | file / crate | **step** |

> **Where tutorials will mislead you.** Every 0.13–0.15 tutorial writes the
> flat form. It does not compile on 0.17.0:
>
> ```zig
> const exe = b.addExecutable(.{
>     .name = "demo",
>     .root_source_file = b.path("src/main.zig"),
>     .target = target,
>     .optimize = optimize,
> });
> ```
> ```
> build.zig:6:10: error: no field named 'root_source_file' in struct 'Build.ExecutableOptions'
>         .root_source_file = b.path("src/main.zig"),
>          ^~~~~~~~~~~~~~~~
> /opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/Build.zig:559:31: note: struct declared here
> pub const ExecutableOptions = struct {
>                               ^~~~~~
> ```
>
> `ExecutableOptions` now has exactly two required fields: `.name` and
> `.root_module`. Source file, target and optimize mode moved **onto the
> module**.

## The canonical `build.zig`

**`build.zig`**

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    const exe = b.addExecutable(.{
        .name = "demo",
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            .target = target,
            .optimize = optimize,
        }),
    });
    b.installArtifact(exe);
    const run_cmd = b.addRunArtifact(exe);
    const run_step = b.step("run", "Run the app");
    run_step.dependOn(&run_cmd.step);
}
```

**`src/main.zig`**

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("hello from the build system\n", .{});
}
```

```sh
zig build
```

```
```

No output on success. The result is on disk:

```sh
find . -not -path "./.zig-cache*" | sort
```

```
.
./build.zig
./src
./src/main.zig
./zig-out
./zig-out/bin
./zig-out/bin/demo
```

```sh
./zig-out/bin/demo
```

```
hello from the build system
```

```sh
zig build run
```

```
hello from the build system
```

| Call | What node it adds |
|---|---|
| `b.standardTargetOptions(.{})` | declares `-Dtarget`/`-Dcpu`/`-Dofmt`, returns a `ResolvedTarget` |
| `b.standardOptimizeOption(.{})` | declares `-Doptimize`, returns a `std.builtin.Optimize` |
| `b.createModule(.{...})` | a **private** module: source file + target + optimize |
| `b.addExecutable(.{...})` | a `Step.Compile` node — compiles into `.zig-cache/` |
| `b.installArtifact(exe)` | a `Step.InstallArtifact` node — copies cache → `zig-out/bin/` |
| `b.addRunArtifact(exe)` | a `Step.Run` node — executes the binary |
| `b.step("run", "...")` | a **top-level** step, invokable as `zig build run` |
| `run_step.dependOn(&n.step)` | an edge: `run_step` cannot run until `n` has |

`b.path("src/main.zig")` is required — a bare `"src/main.zig"` string won't
coerce. It returns a `LazyPath`, which is how the graph refers to files that may
not exist yet.

## Where output lands

```
.zig-cache/   compiler output, content-addressed — everything builds here first
zig-out/bin/  what installArtifact copied — this is your deliverable
```

Both are generated; both belong in `.gitignore`. Nothing lands in `zig-out`
unless some step you declared installs it there.

Drop `installArtifact` and you get a build with nothing to install:

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const exe = b.addExecutable(.{
        .name = "demo",
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            .target = b.standardTargetOptions(.{}),
            .optimize = b.standardOptimizeOption(.{}),
        }),
    });
    const run_step = b.step("run", "Run the app");
    run_step.dependOn(&b.addRunArtifact(exe).step);
}
```

```sh
zig build && ls zig-out 2>&1
```

```
ls: zig-out: No such file or directory
```

```sh
zig build run
```

```
hello from the build system
```

`zig build run` still works — the compile node is reachable from `run`, so it
runs. `zig build` (the default `install` step) has no dependencies, so it does
nothing at all.

## Seeing the graph

```sh
zig build --summary all
```

```
Build Summary: 3/3 steps succeeded
install success
+- install demo success
   +- compile exe demo debug native success 1s MaxRSS:239M
```

The indentation **is** the dependency graph. Run it again:

```sh
zig build --summary all
```

```
Build Summary: 3/3 steps succeeded
install success
+- install demo cached
   +- compile exe demo debug native cached 35ms MaxRSS:33M
```

`success` → `cached`. Per step, not per project.

```sh
zig build run --summary all
```

```
hello from the build system
Build Summary: 3/3 steps succeeded
run success
+- run exe demo success 436ms
   +- compile exe demo debug native cached 35ms MaxRSS:33M
```

Note what is **absent**: `install`. `run` depends on the *compile* node, not on
the install node, so `zig build run` executes the binary straight out of
`.zig-cache/`. Add the edge yourself if you want it installed first:

```zig
run_cmd.step.dependOn(b.getInstallStep());
```

## `zig build --help`

This is generated from your graph, not hardcoded. It is the fastest way to see
what a project you just cloned supports:

```sh
zig build --help
```

```
Usage: /opt/homebrew/Cellar/zig/0.17.0/bin/zig build [steps] [options]

Steps:
  install (default)            Copy build artifacts to prefix path
  uninstall                    Remove build artifacts from prefix path
  run                          Run the app

Project-Specific Options:
  -Dtarget=[string]            The CPU architecture, OS, and ABI to build for
  -Dcpu=[string]               Target CPU features to add or subtract
  -Dofmt=[string]              Target object format
  -Ddynamic-linker=[string]    Path to interpreter on the target system
  -Doptimize=[enum]            Prioritize performance, safety, or binary size
                                 Supported Values:
                                   debug
                                   safe
                                   fast
                                   small
```

Those four `-D` flags came from the two `standard*Options` calls. `run` came
from your `b.step("run", ...)`. Just the step names:

```sh
zig build --list-steps
```

```
  install (default)            Copy build artifacts to prefix path
  uninstall                    Remove build artifacts from prefix path
  run                          Run the app
```

| Flag | Does |
|---|---|
| `zig build` | runs the default step (`install`) |
| `zig build <step>` | runs that top-level step instead |
| `-h`, `--help` | the generated menu |
| `-l`, `--list-steps` | step names only |
| `--summary all` | print the graph with per-step timing and cache status |
| `-p <dir>`, `--prefix <dir>` | install somewhere other than `zig-out` |
| `--verbose` | echo every command before running it |

## A starting skeleton

`zig init` writes a heavily commented four-file project you can read as
reference:

```sh
mkdir myproj && cd myproj && zig init
```

```
info: created build.zig
info: created build.zig.zon
info: created src/main.zig
info: created src/root.zig
info: see `zig build --help` for a menu of options
```

It is already up to date for 0.17.0 — when in doubt about an API, `zig init` in
a temp directory and read what it generated.

---

Next: [10.2 — Steps & Options](./02-steps-options.md)
