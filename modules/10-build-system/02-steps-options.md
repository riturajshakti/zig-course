# 10.2 — Steps & Options

A step is a graph node. `b.step(name, desc)` creates one the user can invoke by
name; `dependOn` is the only way to make anything happen.

## Edges decide order — and only edges

**`build.zig`**

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const a = b.addSystemCommand(&.{ "echo", "A" });
    const c = b.addSystemCommand(&.{ "echo", "C" });
    const d = b.addSystemCommand(&.{ "echo", "D" });

    c.step.dependOn(&a.step);
    d.step.dependOn(&c.step);

    const chain = b.step("chain", "A then C then D");
    chain.dependOn(&d.step);

    const x = b.addSystemCommand(&.{ "echo", "X" });
    const y = b.addSystemCommand(&.{ "echo", "Y" });
    const both = b.step("both", "X and Y, no ordering");
    both.dependOn(&x.step);
    both.dependOn(&y.step);
}
```

```sh
zig build chain --summary all
```

```
A
C
D
Build Summary: 4/4 steps succeeded
chain success
+- run echo success 1ms
   +- run echo success 2ms
      +- run echo success 2ms
```

Three nested edges, so strictly `A → C → D`. Now the unordered pair, five times:

```sh
for i in 1 2 3 4 5; do echo -n "run $i: "; zig build both 2>/dev/null | tr '\n' ' '; echo; done
```

```
run 1: X Y 
run 2: Y X 
run 3: Y X 
run 4: X Y 
run 5: Y X 
```

No edge between `x` and `y`, so they run concurrently and the order flips run to
run. **Declaration order in `build()` means nothing.** If you need ordering, say
so with `dependOn`.

## `run`, with arguments

**`build.zig`**

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "greet",
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            .target = target,
            .optimize = optimize,
        }),
    });
    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());
    run_cmd.addPassthruArgs();
    const run_step = b.step("run", "Run the app");
    run_step.dependOn(&run_cmd.step);

    const unit_tests = b.addTest(.{
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            .target = target,
            .optimize = optimize,
        }),
    });
    const test_step = b.step("test", "Run unit tests");
    test_step.dependOn(&b.addRunArtifact(unit_tests).step);
}
```

**`src/main.zig`**

```zig
const std = @import("std");

fn shout(buf: []u8, name: []const u8) ![]u8 {
    return std.fmt.bufPrint(buf, "HELLO, {s}!", .{name});
}

pub fn main(init: std.process.Init) !void {
    var it = init.minimal.args.iterate();
    _ = it.next();
    var buf: [128]u8 = undefined;
    while (it.next()) |arg| {
        std.debug.print("{s}\n", .{try shout(&buf, arg)});
    }
}

test "shout" {
    var buf: [64]u8 = undefined;
    try std.testing.expectEqualStrings("HELLO, zig!", try shout(&buf, "zig"));
}
```

```sh
zig build run
```

```
```

No args, so the loop never runs. Everything after `--` is forwarded:

```sh
zig build run -- alpha beta
```

```
HELLO, alpha!
HELLO, beta!
```

> **Where tutorials will mislead you.** `b.args` is gone in 0.17.0:
>
> ```zig
> if (b.args) |args| run_cmd.addArgs(args);
> ```
> ```
> build.zig:19:11: error: no field named 'args' in struct 'Build'
>     if (b.args) |args| run_cmd.addArgs(args);
>           ^~~~
> /opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/Build.zig:1:1: note: struct declared here
> const Build = @This();
> ^~~~~
> ```
>
> Replace that whole `if` with one call: **`run_cmd.addPassthruArgs();`**

| Adding args to a `Step.Run` | Use |
|---|---|
| fixed strings | `run.addArgs(&.{ "--verbose", "-j4" })` |
| whatever the user typed after `--` | `run.addPassthruArgs()` |
| a path to another artifact | `run.addArtifactArg(other_exe)` |
| a file the graph produces | `run.addFileArg(lazy_path)` |
| a file *this* step produces | `run.addOutputFileArg("out.bin")` |

## The test step

```sh
zig build test
```

```
```

Silence means pass. Ask for detail:

```sh
zig build test --summary all
```

```
Build Summary: 3/3 steps succeeded; 1/1 tests passed
test success
+- run test 1 pass (1 total) 919ms MaxRSS:2M
   +- compile test debug native success 1s MaxRSS:267M
```

A failure is loud with no flags at all. Change the expectation to
`"hello, zig!"`:

```sh
zig build test
```

```
test
+- run test 0 pass, 1 fail (1 total)
error: 'main.test.shout' failed:
       ====== expected this output: =========
       hello, zig!␃
       
       ======== instead found this: =========
       HELLO, zig!␃
       
       ======================================
       First difference occurs on line 1:
       expected:
       hello, zig!
       ^ ('\x68')
       found:
       HELLO, zig!
       ^ ('\x48')
       /opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/testing.zig:696:9: 0x10233b5db in expectEqualStrings (test)
               return error.TestExpectedEqual;
               ^
```

`addTest` takes the same `.root_module` shape as `addExecutable` — it compiles a
test binary from the module's `test` blocks. `addRunArtifact` then executes it.

## `b.option` and `-D`

**`build.zig`**

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const name = b.option([]const u8, "name", "Binary name") orelse "app";
    const strip = b.option(bool, "strip", "Strip symbols") orelse false;
    const level = b.option(u8, "level", "Log level 0-3") orelse 1;

    const opts = b.addOptions();
    opts.addOption(u8, "level", level);
    opts.addOption([]const u8, "app_name", name);

    const mod = b.createModule(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
        .strip = strip,
    });
    mod.addOptions("config", opts);

    const exe = b.addExecutable(.{ .name = name, .root_module = mod });
    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);
    const run_step = b.step("run", "Run the app");
    run_step.dependOn(&run_cmd.step);

    const hello = b.addSystemCommand(&.{ "echo", "custom step ran" });
    const hello_step = b.step("hello", "Run a host command");
    hello_step.dependOn(&hello.step);

    const all = b.step("all", "install + hello");
    all.dependOn(b.getInstallStep());
    all.dependOn(&hello.step);
}
```

**`src/main.zig`**

```zig
const std = @import("std");
const config = @import("config");

pub fn main() void {
    std.debug.print("{s} level={d}\n", .{ config.app_name, config.level });
}
```

```sh
zig build run
```

```
app level=1
```

```sh
zig build run -Dname=tool -Dlevel=3
```

```
tool level=3
```

Two things happened there. `-Dname` changed the **output filename** (a build-time
decision), and `-Dlevel` was compiled *into* the program via `b.addOptions()`.
`@import("config")` is a module Zig generated for you from those values — they
are `comptime` constants, not runtime lookups.

| Step | API |
|---|---|
| read a CLI flag | `b.option(T, "name", "description") -> ?T` |
| make it reachable from source | `const o = b.addOptions(); o.addOption(T, "k", v);` |
| expose as an importable module | `mod.addOptions("config", o)` |
| read it in Zig | `const config = @import("config");` |

Supported `T` for `b.option` (from `typeToEnum` in `std/Build.zig`): `bool`,
any integer, any float, `[]const u8`, `[]const []const u8`, any `enum`,
`[]const SomeEnum`, `std.Build.LazyPath`, `[]const LazyPath`, and
`std.zig.BuildId`. Anything else is a `@compileError`.

Your options appear in the generated help automatically:

```sh
zig build --help
```

```
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
  -Dname=[string]              Binary name
  -Dstrip=[bool]               Strip symbols
  -Dlevel=[int]                Log level 0-3
```

A bad value fails the configure phase and lists what was available:

```sh
zig build -Doptimize=Bogus
```

```
error: expected -Doptimize to be of type "lang.Optimize"
info: available option: "target": The CPU architecture, OS, and ABI to build for
info: available option: "cpu": Target CPU features to add or subtract
info: available option: "ofmt": Target object format
info: available option: "dynamic-linker": Path to interpreter on the target system
info: available option: "optimize": Prioritize performance, safety, or binary size
info: available option: "name": Binary name
info: available option: "strip": Strip symbols
```

| `-D` form | Means |
|---|---|
| `-Dstrip` | bare flag on a `bool` option → `true` |
| `-Dstrip=true` / `-Dstrip=false` | explicit |
| `-Dname=tool` | scalar |
| `-Dlevel=3` | integer, range-checked against `u8` |

## Custom steps

```sh
zig build hello
```

```
custom step ran
```

```sh
zig build all --summary all
```

```
custom step ran
Build Summary: 6/6 steps succeeded
all success
+- install success
|  +- install app success
|     +- compile exe app debug native cached 34ms MaxRSS:33M
|        +- options cached
+- run echo success 1ms
```

`all` has no work of its own — it is a pure aggregator, two `dependOn` calls.
That is the idiomatic way to build a `ci` or `release` step.

```sh
zig build --list-steps
```

```
  install (default)            Copy build artifacts to prefix path
  uninstall                    Remove build artifacts from prefix path
  run                          Run the app
  hello                        Run a host command
  all                          install + hello
```

> `addSystemCommand` shells out to the host. It makes your build depend on
> what's installed on that machine, which is exactly the thing Zig otherwise
> avoids. Reach for `addRunArtifact` (a binary Zig built) when you can.

## The default step

`zig build` with no step name runs `install`. That is just a step like any
other, and you can hang more off it:

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const a = b.addSystemCommand(&.{ "echo", "A" });
    b.default_step.dependOn(&a.step);
}
```

```sh
zig build --summary all
```

```
A
Build Summary: 2/2 steps succeeded
install success
+- run echo success 1ms
```

| Accessor | Is |
|---|---|
| `b.default_step` | what bare `zig build` runs — named `install` |
| `b.getInstallStep()` | the same node, by its conventional accessor |
| `b.step(name, desc)` | a fresh top-level step |
| `b.top_level_steps` | the map behind `--list-steps` |

---

Next: [10.3 — Modules](./03-modules.md)
