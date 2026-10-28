# 9.1 — Writing Tests

There is no test framework, no `describe`, no import, no `_test.go` convention.
`test` is a **keyword**, test blocks live in the same file as the code they test,
and the compiler strips them out of normal builds.

## A test file

```zig
const std = @import("std");

fn add(a: i32, b: i32) i32 {
    return a + b;
}

test "add works" {
    try std.testing.expectEqual(5, add(2, 3));
}
```

This file has **no `main`**. Run it with `zig test`, not `zig run`:

```sh
zig test t1.zig
```

```
1/1 t1.test.add works...OK
All 1 tests passed.
```

> **`zig run` on a test-only file fails bizarrely.** There is no `main`, so the
> startup code can't be instantiated and you get an error from deep inside the
> standard library that has nothing to do with your code:
>
> ```
> /opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/start.zig:624:23: error: struct 'elf.AT__struct_853' has no member named 'HWCAP'
>                 elf.AT.HWCAP => at_hwcap = auxv[i].a_un.a_val,
>                 ~~~~~~^~~~~~
> ```
> If you ever see `HWCAP`, you typed `zig run` where you meant `zig test`.

## Tests sit next to the code — including private code

A `test` block is a top-level declaration like `fn` or `const`, so it can see
everything in the file, `pub` or not. There is nothing to export for testability.

```zig
const std = @import("std");

pub fn celsiusToF(c: f64) f64 {
    return c * 9.0 / 5.0 + 32.0;
}

fn clampByte(n: i32) u8 {
    if (n < 0) return 0;
    if (n > 255) return 255;
    return @intCast(n);
}

test "celsiusToF" {
    try std.testing.expectEqual(212.0, celsiusToF(100));
}

test "clampByte saturates" {
    try std.testing.expectEqual(0, clampByte(-7));
    try std.testing.expectEqual(255, clampByte(999));
    try std.testing.expectEqual(42, clampByte(42));
}
```

```sh
zig test t2.zig
```

```
1/2 t2.test.celsiusToF...OK
2/2 t2.test.clampByte saturates...OK
All 2 tests passed.
```

`clampByte` is private. In Go you'd put the test in the same package; in Node
you'd export it "for testing". Here there is no boundary to cross.

## Tests are not compiled in a normal build

Not "compiled and skipped" — **not analysed at all**. This file proves it with a
`@compileError` inside a test body:

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("main ran\n", .{});
}

test "this test is not compiled by zig run" {
    @compileError("if you see this, the test body was analysed");
}
```

```sh
zig run t3.zig
```

```
main ran
```

It builds and runs fine. Now the same file under `zig test`:

```sh
zig test t3.zig
```

```
t3.zig:8:5: error: if you see this, the test body was analysed
    @compileError("if you see this, the test body was analysed");
    ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

Consequences:

| | |
|---|---|
| Test code costs **zero** bytes in a shipped binary | no `#ifdef`, no build tags |
| A test referencing a deleted function breaks only `zig test` | your app still builds |
| Mock/fixture helpers used only by tests are free | they aren't even semantically checked |

## Naming

The reported name is `<file>.test.<your name>`, so the name is a search key.
Namespace with a prefix and `--test-filter` becomes a selector.

```zig
const std = @import("std");

fn isEven(n: u32) bool {
    return n % 2 == 0;
}

fn isOdd(n: u32) bool {
    return !isEven(n);
}

test "math: isEven" {
    try std.testing.expect(isEven(4));
}

test "math: isOdd" {
    try std.testing.expect(isOdd(5));
}

test "strings: starts with" {
    try std.testing.expect(std.mem.startsWith(u8, "zigzag", "zig"));
}
```

```sh
zig test t4.zig
```

```
1/3 t4.test.math: isEven...OK
2/3 t4.test.math: isOdd...OK
3/3 t4.test.strings: starts with...OK
All 3 tests passed.
```

```sh
zig test --test-filter "math:" t4.zig
```

```
1/2 t4.test.math: isEven...OK
2/2 t4.test.math: isOdd...OK
All 2 tests passed.
```

```sh
zig test --test-filter "isOdd" t4.zig
```

```
1/1 t4.test.math: isOdd...OK
All 1 tests passed.
```

A filter that matches nothing is **not an error**:

```sh
zig test --test-filter "nope" t4.zig; echo "exit=$?"
```

```
All 0 tests passed.
exit=0
```

| Form | Reported as | Notes |
|---|---|---|
| `test "add works" { }` | `t1.test.add works` | spaces and `:` are fine |
| `test { }` | `root.test_0` | unnamed; numbered in source order |
| `test add { }` | `dt.decltest.add` | a *decltest*: `add` is an identifier in scope, not a string |

The filter matches the **full** reported name, so filtering on a file name works
too — `--test-filter "t4."` runs everything in `t4.zig`.

## Multiple files: tests must be reachable

`zig test root.zig` compiles only what `root.zig` references. A separate file's
tests are invisible until something pulls it in.

`math.zig`:

```zig
const std = @import("std");

pub fn double(n: i32) i32 {
    return n * 2;
}

test "double" {
    try std.testing.expectEqual(8, double(4));
}
```

`root.zig`:

```zig
const std = @import("std");
const math = @import("math.zig");

pub fn triple(n: i32) i32 {
    return n * 3;
}

test {
    _ = math;
}

test "triple" {
    try std.testing.expectEqual(9, triple(3));
}
```

```sh
zig test root.zig
```

```
1/3 root.test_0...OK
2/3 root.test.triple...OK
3/3 math.test.double...OK
All 3 tests passed.
```

Delete the `test { _ = math; }` block and `math.zig`'s test vanishes silently —
the same run reports `All 1 tests passed`. **A silently-not-run test is worse
than a failing one**, so the `_ = @import(...)` roll-up is a real habit, not
boilerplate.

## `refAllDecls` for tests inside types

A `test` inside a `struct` has the same problem one level down: the struct is
referenced, but its declarations may not be.

```zig
const std = @import("std");

pub const Parser = struct {
    pub fn parse(s: []const u8) !u32 {
        return std.fmt.parseInt(u32, s, 10);
    }

    test "Parser.parse" {
        try std.testing.expectEqual(42, try Parser.parse("42"));
    }
};

test {
    std.testing.refAllDecls(@This());
}
```

```sh
zig test refall.zig
```

```
1/2 refall.test_0...OK
2/2 refall.Parser.test.Parser.parse...OK
All 2 tests passed.
```

Remove the `refAllDecls` block:

```
1/1 refall2.test.top level...OK
All 1 tests passed.
```

`Parser.parse` was never run. `refAllDecls(@This())` touches every declaration
in the current file so the analyzer walks into each one.

| Need | Write |
|---|---|
| another file's tests | `test { _ = @import("other.zig"); }` |
| this file's nested-type tests | `test { std.testing.refAllDecls(@This()); }` |
| both | put both in one `test { }` block |

`refAllDecls` compiles to nothing outside a test build — its first line is
`if (!builtin.is_test) return;`.

## `zig build test`

For a project, a `test` step wires this into `zig build`:

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const mod = b.createModule(.{
        .root_source_file = b.path("src/root.zig"),
        .target = target,
        .optimize = optimize,
    });

    const unit_tests = b.addTest(.{ .root_module = mod });
    const run_tests = b.addRunArtifact(unit_tests);

    const test_step = b.step("test", "Run unit tests");
    test_step.dependOn(&run_tests.step);
}
```

```sh
zig build test --summary all
```

```
Build Summary: 3/3 steps succeeded; 1/1 tests passed
test success
+- run test 1 pass (1 total) 487ms MaxRSS:2M
   +- compile test debug native success 1s MaxRSS:257M
```

Note `addTest` takes a **`root_module`** in 0.17, not the `root_source_file`/
`target`/`optimize` triple older tutorials pass directly. Module 10 covers
`build.zig` properly; `zig test file.zig` is enough for everything here.

## Command reference

| Command | Does |
|---|---|
| `zig test f.zig` | compile tests in `f.zig` and run them |
| `zig test --test-filter "x" f.zig` | only tests whose full name contains `x` |
| `zig test -Osafe f.zig` | run the tests in ReleaseSafe (see 9.5) |
| `zig test --test-no-exec -femit-bin=t f.zig` | build the test binary without running it (for `lldb`) |
| `zig build test` | run the `test` step from `build.zig` |
| `zig run f.zig` | **ignores** every `test` block |

---

Next: [9.2 — Assertions](./02-assertions.md)
