# 6.1 — comptime Basics

Zig has no macros, no template language, and no separate constexpr dialect. It
runs **ordinary Zig** at compile time, with the same syntax you already write.

```zig
const std = @import("std");

fn fib(n: u64) u64 {
    return if (n < 2) n else fib(n - 1) + fib(n - 2);
}

const compiled_in = fib(12);

pub fn main() void {
    std.debug.print("fib(12) = {d}\n", .{compiled_in});
    std.debug.print("fib(12) = {d}\n", .{fib(12)});
}
```

```
fib(12) = 144
fib(12) = 144
```

Same function, called twice. The first call ran **in the compiler** — `144` is a
constant baked into the binary. The second runs on the CPU. No `constexpr`
keyword, no second version of `fib`.

## What forces compile-time evaluation

| Context | Evaluated at |
|---|---|
| a top-level `const` initializer | compile time, always |
| an array length `[N]T` | compile time, always |
| a `comptime` parameter's argument | compile time, always |
| the `comptime` keyword on an expression | compile time, always |
| inside a `comptime { }` block | compile time, always |
| everything else | runtime, *unless the compiler folds it anyway* |

## comptime-known vs runtime

The distinction is about **values**, not types. A value is comptime-known if the
compiler can see it:

```zig
const std = @import("std");

pub fn main() void {
    const a = 10;
    var b: u32 = 10;
    b += 0;

    const arr1: [a]u8 = @splat('x');
    std.debug.print("{s}\n", .{arr1});
    std.debug.print("b = {d}\n", .{b});
}
```

```
xxxxxxxxxx
b = 10
```

`[a]u8` works because `a` is comptime-known. Swap in `b` and the array length
has nothing to be:

```zig
const std = @import("std");

pub fn main() void {
    var b: u32 = 10;
    b += 0;

    const arr: [b]u8 = @splat('x');
    std.debug.print("{s}\n", .{arr});
}
```

**Deliberate compile error:**

```
badlen.zig:7:17: error: unable to resolve comptime value
    const arr: [b]u8 = @splat('x');
                ^
badlen.zig:7:17: note: types must be comptime-known
```

`[b]u8` is a *type*, and types must be fully known before the program runs.

Same thing with the `comptime` keyword — it is a *demand*, and an unmeetable
demand is an error:

```zig
const std = @import("std");

fn double(n: u32) u32 {
    return n * 2;
}

pub fn main() void {
    var n: u32 = 21;
    n += 0;
    const d = comptime double(n);
    std.debug.print("{d}\n", .{d});
}
```

**Deliberate compile error:**

```
p3.zig:10:31: error: unable to resolve comptime value
    const d = comptime double(n);
                              ^
p3.zig:10:15: note: 'comptime' keyword forces comptime evaluation
    const d = comptime double(n);
              ^~~~~~~~~~~~~~~~~~
```

> Read `comptime x` as "I assert `x` is knowable now." The error tells you
> exactly which value wasn't.

## comptime blocks and comptime vars

A `comptime` block runs the whole thing in the compiler. Inside it, `var` is
free to mutate — the mutation just happens before your program exists.

```zig
const std = @import("std");

pub fn main() void {
    comptime var counter: u32 = 0;
    comptime counter += 1;
    comptime counter += 1;
    comptime counter += 1;
    std.debug.print("counter = {d}\n", .{counter});

    const name = comptime blk: {
        var buf: [16]u8 = undefined;
        const parts = [_][]const u8{ "zig", "-", "0.17" };
        var len: usize = 0;
        for (parts) |p| {
            @memcpy(buf[len..][0..p.len], p);
            len += p.len;
        }
        break :blk buf[0..len].*;
    };
    std.debug.print("name = {s} ({d} bytes, built at compile time)\n", .{ name, name.len });
}
```

```
counter = 3
name = zig-0.17 (8 bytes, built at compile time)
```

That second block is string building — `@memcpy`, a mutable buffer, a loop — and
it leaves **zero** code in the binary. Only the 8 bytes `zig-0.17` survive.

### The classic: a generated lookup table

```zig
const std = @import("std");

pub fn main() void {
    const table = comptime blk: {
        var t: [10]u64 = undefined;
        for (&t, 0..) |*slot, i| slot.* = i * i;
        break :blk t;
    };
    std.debug.print("{any}\n", .{table});
}
```

```
{ 0, 1, 4, 9, 16, 25, 36, 49, 64, 81 }
```

The loop is gone. The array is data in the binary. This replaces the build-time
codegen script you'd write in Node or Python.

## The branch quota

Compile-time evaluation is interpreted, and the compiler refuses to loop
forever. The default budget is **1000 backwards branches**:

```zig
const std = @import("std");

fn fib(n: u64) u64 {
    return if (n < 2) n else fib(n - 1) + fib(n - 2);
}

const v = fib(14);

pub fn main() void {
    std.debug.print("fib(14) = {d}\n", .{v});
}
```

**Deliberate compile error:**

```
q14b.zig:4:46: error: evaluation exceeded 1000 backwards branches
    return if (n < 2) n else fib(n - 1) + fib(n - 2);
                                          ~~~^~~~~~~
q14b.zig:4:46: note: use @setEvalBranchQuota() to raise the branch limit from 1000
q14b.zig:4:46: note: called at comptime here (5 times)
q14b.zig:4:33: note: called at comptime here (2 times)
```

The error names its own fix:

```zig
const std = @import("std");

fn fib(n: u64) u64 {
    return if (n < 2) n else fib(n - 1) + fib(n - 2);
}

const small = fib(12);
const big = blk: {
    @setEvalBranchQuota(2_000_000);
    break :blk fib(25);
};

pub fn main() void {
    std.debug.print("fib(12) = {d}  (fits default quota of 1000)\n", .{small});
    std.debug.print("fib(25) = {d}  (needed @setEvalBranchQuota)\n", .{big});
}
```

```
fib(12) = 144  (fits default quota of 1000)
fib(25) = 75025  (needed @setEvalBranchQuota)
```

`@setEvalBranchQuota` applies to the block it appears in. It is a safety valve,
not a performance knob — raising it makes **your compile** slower, which 6.6
measures.

## comptime `if` deletes code

When the condition is comptime-known, the dead branch is never compiled at all.
Proof: put a `@compileError` in the branch that shouldn't exist.

```zig
const std = @import("std");

const enable_check = false;

fn add(a: u32, b: u32) u32 {
    if (enable_check) {
        @compileError("this branch was compiled, so enable_check was true");
    }
    return a + b;
}

pub fn main() void {
    std.debug.print("{d}\n", .{add(2, 3)});
}
```

```
5
```

It compiled and ran. A `@compileError` that is never *reached* is never
*triggered* — so the branch was genuinely removed, not optimized away later.
Flip `enable_check` to `true` and you get the error.

This is how feature flags, debug logging and platform switches cost nothing:

```zig
const std = @import("std");
const builtin = @import("builtin");

const verbose = false;

fn log(comptime msg: []const u8) void {
    if (verbose) {
        std.debug.print("[log] {s}\n", .{msg});
    }
}

fn describe(comptime T: type) []const u8 {
    if (@typeInfo(T) == .int) return "integer";
    if (@typeInfo(T) == .float) return "float";
    if (@typeInfo(T) == .bool) return "bool";
    return "other";
}

pub fn main() void {
    log("you will never see this");
    std.debug.print("verbose = {}\n", .{verbose});
    std.debug.print("{s} {s} {s} {s}\n", .{ describe(u8), describe(f32), describe(bool), describe([]u8) });
    std.debug.print("built in {t} mode for {t}\n", .{ builtin.mode, builtin.cpu.arch });
}
```

```
verbose = false
integer float bool other
built in debug mode for aarch64
```

`describe` returns a `[]const u8` but contains no runtime work either — every
`if` folded, so each call is just a pointer to a string literal.

## `@compileError` — your own diagnostics

```zig
const std = @import("std");

fn bytesOf(comptime T: type) usize {
    if (@typeInfo(T) != .int) @compileError("bytesOf needs an integer type, got " ++ @typeName(T));
    return @sizeOf(T);
}

pub fn main() void {
    std.debug.print("u32 -> {d}\n", .{bytesOf(u32)});
    std.debug.print("f64 -> {d}\n", .{bytesOf(f64)});
}
```

**Deliberate compile error:**

```
ce.zig:4:31: error: bytesOf needs an integer type, got f64
    if (@typeInfo(T) != .int) @compileError("bytesOf needs an integer type, got " ++ @typeName(T));
                              ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
referenced by:
    main: ce.zig:10:46
```

Note `++` concatenating with `@typeName(T)` — building the message is itself
comptime string work. `referenced by:` points at the offending call. This is
Zig's equivalent of a trait bound failure, and 6.3 uses it for constraints.

## `@compileLog` — printf debugging the compiler

```zig
const std = @import("std");

fn sizeTag(comptime T: type) []const u8 {
    @compileLog("inspecting ", @typeName(T), @sizeOf(T));
    return @typeName(T);
}

pub fn main() void {
    std.debug.print("{s} {s}\n", .{ sizeTag(u8), sizeTag(u64) });
}
```

**Deliberate compile error** — `@compileLog` always fails the build, by design:

```
cl.zig:4:5: error: found compile log statement
    @compileLog("inspecting ", @typeName(T), @sizeOf(T));
    ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
cl.zig:4:5: note: also here

Compile Log Output:
@as(*const [11:0]u8, "inspecting "), @as(*const [3:0]u8, "u64"), @as(comptime_int, 8)
@as(*const [11:0]u8, "inspecting "), @as(*const [2:0]u8, "u8"), @as(comptime_int, 1)
```

| Detail | Why it matters |
|---|---|
| it's an **error**, not a warning | you can never ship a `@compileLog` |
| values print with full types | `@as(comptime_int, 8)` — exactly what the compiler sees |
| one line per instantiation | two lines here = `sizeTag` was compiled twice |

That last row is the whole of 6.6 in miniature: one generic function, two types,
two separate compiled copies.

## `std.debug.assert` at comptime

The same `assert` you use at runtime becomes a compile-time check when its
condition is comptime-known.

```zig
const std = @import("std");

fn Buffer(comptime size: usize) type {
    std.debug.assert(size > 0);
    std.debug.assert(size % 8 == 0);
    return struct { bytes: [size]u8 = undefined };
}

pub fn main() void {
    const B = Buffer(64);
    std.debug.print("{s} is {d} bytes\n", .{ @typeName(B), @sizeOf(B) });
}
```

```
as.Buffer(64) is 64 bytes
```

Pass a size that violates it:

```zig
const std = @import("std");

fn Buffer(comptime size: usize) type {
    std.debug.assert(size % 8 == 0);
    return struct { bytes: [size]u8 = undefined };
}

pub fn main() void {
    const B = Buffer(10);
    std.debug.print("{d}\n", .{@sizeOf(B)});
}
```

**Deliberate compile error:**

```
/opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/debug.zig:442:14: error: reached unreachable code
    if (!ok) unreachable; // assertion failure
             ^~~~~~~~~~~
asf.zig:4:21: note: called at comptime here
    std.debug.assert(size % 8 == 0);
    ~~~~~~~~~~~~~~~~^~~~~~~~~~~~~~~
asf.zig:9:21: note: called at comptime here
    const B = Buffer(10);
              ~~~~~~^~~~
```

A runtime panic became a **build failure**. The trace reads bottom-up: your
call, then the assert, then the `unreachable` inside it.

`assert` is blunt, though — the message is `reached unreachable code`, pointing
into the stdlib. For an API other people call, `@compileError` is kinder:

| Want | Use |
|---|---|
| an internal invariant, terse | `std.debug.assert(cond)` |
| a message naming the problem and the type | `if (!cond) @compileError("...")` |

---

Next: [6.2 — Types Are Values](./02-types-are-values.md)
