# 2.6 — Blocks & Labels

## A block can produce a value

```zig
const std = @import("std");

pub fn main() void {
    const x = blk: {
        var acc: i32 = 0;
        for (1..5) |i| acc += @intCast(i);
        break :blk acc * 2;
    };

    std.debug.print("x = {d}\n", .{x});
}
```

```
x = 20
```

Label a block, `break :label value` out of it, and the block *is* an expression.
This replaces the IIFE you would write in JavaScript and the
`var x T; { ... x = ... }` dance you would write in Go — and crucially, `x` stays
`const`.

The scratch variable `acc` disappears with the block. That is the real payoff:
multi-step initialization without leaking its intermediates into the enclosing
scope.

## Every control structure is an expression

```zig
const std = @import("std");

pub fn main() void {
    const n: i32 = 42;

    const a = if (n > 0) n else -n;

    const b = switch (n) {
        0 => "zero",
        else => "nonzero",
    };

    const c = blk: {
        if (n < 0) break :blk "negative";
        if (n == 0) break :blk "zero";
        break :blk "positive";
    };

    std.debug.print("{d} {s} {s}\n", .{ a, b, c });
}
```

```
42 nonzero positive
```

| Need | Use |
|---|---|
| one condition, two values | `if (c) a else b` |
| match one value against many | `switch` |
| several statements, early exits, loops | labeled block + `break :blk` |

A labeled block gives you **early return from an expression**, which `if` and
`switch` cannot do. That `c` reads like a function with guard clauses, but it is
inline and produces a `const`.

## Blocks at file scope: compile-time computed data

```zig
const std = @import("std");

const squares = blk: {
    var t: [8]u32 = undefined;
    for (&t, 0..) |*slot, i| slot.* = @intCast(i * i);
    break :blk t;
};

pub fn main() void {
    std.debug.print("{any}\n", .{squares});

    const runtime_sum = sum: {
        var acc: u32 = 0;
        for (squares) |s| acc += s;
        break :sum acc;
    };
    std.debug.print("sum = {d}\n", .{runtime_sum});
}
```

```
{ 0, 1, 4, 9, 16, 25, 36, 49 }
sum = 140
```

The first block runs **at compile time** — file-scope initializers always do —
so `squares` is a lookup table baked into the binary, written as a loop rather
than typed out by hand. No `build.zig` codegen step, no macros.

You do not write `comptime` on it. In fact you cannot:

```
error: redundant comptime keyword in already comptime scope
const squares = comptime blk: {
                ^~~~~~~~
```

The second block is identical syntax running at runtime. Same construct, two
phases — the distinction is *where* it appears, not a keyword.

## Labeled loops: `break` out of nested loops

```zig
const std = @import("std");

pub fn main() void {
    const grid = [_][3]i32{
        .{ 1, 2, 3 },
        .{ 4, 5, 6 },
        .{ 7, 8, 9 },
    };

    outer: for (grid, 0..) |row, r| {
        for (row, 0..) |cell, c| {
            if (cell == 5) {
                std.debug.print("found 5 at ({d},{d})\n", .{ r, c });
                break :outer;
            }
            std.debug.print("checked {d}\n", .{cell});
        }
    }
}
```

```
checked 1
checked 2
checked 3
checked 4
found 5 at (1,1)
```

`break :outer` exits both loops. Without the label, `break` only leaves the
inner one and you need the `found` flag that every nested-search function in C
and Python carries around.

## Labeled `continue`

```zig
const std = @import("std");

pub fn main() void {
    const grid = [_][3]i32{
        .{ 1, 3, 5 },
        .{ 4, 5, 6 },
        .{ 7, 9, 11 },
    };

    rows: for (grid, 0..) |row, r| {
        for (row) |cell| {
            if (@mod(cell, 2) == 0) {
                std.debug.print("row {d}: even found, skip\n", .{r});
                continue :rows;
            }
        }
        std.debug.print("row {d}: all odd\n", .{r});
    }
}
```

```
row 0: all odd
row 1: even found, skip
row 2: all odd
```

`continue :rows` abandons the inner loop and advances the **outer** one. The
statement after the inner loop (`all odd`) is only reached when the inner loop
completed without jumping — making it the natural place for "nothing matched"
logic.

## `while ... else` and `for ... else`

The `else` clause on a loop runs when the loop **finished normally** — i.e. the
condition went false or the sequence ran out — and is skipped when a `break`
ended it. Combine that with `break value` and you get search-with-default:

```zig
const std = @import("std");

pub fn main() void {
    const nums = [_]i32{ 2, 4, 6, 8 };

    const first_odd = for (nums) |n| {
        if (@mod(n, 2) == 1) break n;
    } else blk: {
        std.debug.print("no odd number in the list\n", .{});
        break :blk -1;
    };
    std.debug.print("first_odd = {d}\n", .{first_odd});

    var i: usize = 0;
    const big = while (i < nums.len) : (i += 1) {
        if (nums[i] > 5) break nums[i];
    } else 0;
    std.debug.print("first > 5 = {d}\n", .{big});
}
```

```
no odd number in the list
first_odd = -1
first > 5 = 6
```

The `else` is itself a labeled block here, so the "not found" path can log and
*then* produce a value. Reading the two together:

| Outcome | `break` taken | `else` runs | Result |
|---|---|---|---|
| found a match | yes | no | the broken-with value |
| sequence exhausted | no | yes | the `else` value |

## `defer` runs in reverse

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("start\n", .{});
    defer std.debug.print("defer 1\n", .{});
    defer std.debug.print("defer 2\n", .{});
    defer std.debug.print("defer 3\n", .{});
    std.debug.print("end of main body\n", .{});
}
```

```
start
end of main body
defer 3
defer 2
defer 1
```

**LIFO**, like a stack. That ordering is not a quirk — it is what makes
acquire/release pairs correct. If you open a file then allocate a buffer, you
must free the buffer before closing the file, and writing the `defer` next to
each acquisition gives you that order for free.

## `defer` is scoped to its block, not the function

This is where Go programmers get caught: Go's `defer` runs at *function* exit,
Zig's runs at **block** exit.

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("enter main\n", .{});
    {
        defer std.debug.print("  leaving inner block\n", .{});
        std.debug.print("  inside inner block\n", .{});
    }
    std.debug.print("after inner block\n", .{});

    for (0..3) |i| {
        defer std.debug.print("  iteration {d} done\n", .{i});
        std.debug.print("iteration {d}\n", .{i});
    }
    std.debug.print("exit main\n", .{});
}
```

```
enter main
  inside inner block
  leaving inner block
after inner block
iteration 0
  iteration 0 done
iteration 1
  iteration 1 done
iteration 2
  iteration 2 done
exit main
```

A `defer` inside a loop body runs **every iteration**. In Go that same code
would queue three deferred calls and run them all when the function returns —
the classic "defer in a loop leaks file handles" bug. Zig's version is what you
almost always want.

## `defer` runs on the error path too

```zig
const std = @import("std");

fn work(fail: bool) !i32 {
    std.debug.print("  acquire\n", .{});
    defer std.debug.print("  release\n", .{});

    if (fail) return error.Boom;
    return 7;
}

pub fn main() void {
    std.debug.print("ok path:\n", .{});
    std.debug.print("  got {!d}\n", .{work(false)});
    std.debug.print("fail path:\n", .{});
    std.debug.print("  got {!d}\n", .{work(true)});
}
```

```
ok path:
  acquire
  release
  got 7
fail path:
  acquire
  release
  got error.Boom
```

`release` printed on both paths. Any exit from the scope — falling off the end,
`return`, `break`, or returning an error — runs the `defer`. There is no path
that skips it, which is why `defer` is the only cleanup mechanism Zig needs and
why you will see it on the line *after* every allocation in module 05.

Its sibling `errdefer` runs **only** on the error path, for the case where
success means handing ownership to the caller. Module 04 covers it.

## Putting it together

```zig
const std = @import("std");

const Found = struct { row: usize, sum: u32 };

pub fn main() void {
    const grid = [_][4]u8{
        .{ 3, 9, 7, 4 },
        .{ 2, 8, 6, 4 },
        .{ 1, 1, 1, 1 },
    };

    const report: ?Found = find: {
        defer std.debug.print("(search finished)\n", .{});

        for (grid, 0..) |row, r| {
            var sum: u32 = 0;
            for (row) |cell| sum += cell;
            std.debug.print("row {d} sum {d}\n", .{ r, sum });
            if (@mod(sum, 2) == 0) break :find .{ .row = r, .sum = sum };
        } else break :find null;
    };

    if (report) |f| {
        std.debug.print("first even-sum row: {d} (sum {d})\n", .{ f.row, f.sum });
    } else {
        std.debug.print("no even-sum row\n", .{});
    }
}
```

```
row 0 sum 23
row 1 sum 20
(search finished)
first even-sum row: 1 (sum 20)
```

Make every row odd-summed and the `else` arm takes over:

```zig
const std = @import("std");

const Found = struct { row: usize, sum: u32 };

pub fn main() void {
    const grid = [_][4]u8{
        .{ 3, 9, 7, 4 },
        .{ 1, 1, 1, 2 },
    };

    const report: ?Found = find: {
        defer std.debug.print("(search finished)\n", .{});

        for (grid, 0..) |row, r| {
            var sum: u32 = 0;
            for (row) |cell| sum += cell;
            std.debug.print("row {d} sum {d}\n", .{ r, sum });
            if (@mod(sum, 2) == 0) break :find .{ .row = r, .sum = sum };
        } else break :find null;
    };

    if (report) |f| {
        std.debug.print("first even-sum row: {d} (sum {d})\n", .{ f.row, f.sum });
    } else {
        std.debug.print("no even-sum row\n", .{});
    }
}
```

```
row 0 sum 23
row 1 sum 5
(search finished)
no even-sum row
```

One `const`, one labeled block, a `for ... else` for the not-found case, a
`defer` that fires however the block exits, and no mutable state outside.

Two details worth keeping:

- The block's type is annotated `?Found`. Had both arms used anonymous
  `.{ ... }` literals, each would be a *distinct* anonymous struct type and the
  block would not compile: `error: incompatible types`. Naming the type (or
  annotating the destination) is what makes the two `break` arms agree.
- `defer` fired before the `if` ran, because the block is its scope — the search
  cleaned up the instant it produced a value.

---

Module 02 done. Next: [Module 03 — Types, Structs & Unions](../03-types-structs-unions/)
