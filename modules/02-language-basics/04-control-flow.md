# 2.4 — Control Flow

## There is no truthiness

```zig
const std = @import("std");

pub fn main() void {
    const n: i32 = 5;
    if (n) {
        std.debug.print("truthy\n", .{});
    }
}
```

```
error: expected type 'bool', found 'i32'
    if (n) {
        ^
```

Not a warning — the program does not build. Same for a length:

```zig
const std = @import("std");

pub fn main() void {
    const name = "zig";
    if (name.len) {
        std.debug.print("non-empty\n", .{});
    }
}
```

```
error: expected type 'bool', found 'usize'
    if (name.len) {
        ~~~~^~~~
```

A condition must be a `bool`. Nothing else converts — not `0`, not `""`, not
`null`, not an empty array. Write the comparison:

```zig
const std = @import("std");

pub fn main() void {
    const n: i32 = 5;
    const name = "zig";
    const maybe: ?i32 = null;

    if (n != 0) std.debug.print("n is non-zero\n", .{});
    if (name.len > 0) std.debug.print("name is non-empty\n", .{});
    if (maybe == null) std.debug.print("maybe is null\n", .{});
}
```

```
n is non-zero
name is non-empty
maybe is null
```

After enough JavaScript this feels like bureaucracy for about a day, and then
you notice you have stopped wondering whether `0`, `""` and `[]` are falsy in
the language you are currently in.

## `if` is an expression

```zig
const std = @import("std");

pub fn main() void {
    const n: i32 = 17;

    const label = if (n < 10) "small" else if (n < 100) "medium" else "large";
    const abs = if (n < 0) -n else n;
    const clamped: i32 = if (n > 10) 10 else n;

    std.debug.print("{s} {d} {d}\n", .{ label, abs, clamped });
}
```

```
medium 17 10
```

There is no ternary operator because there is no need for one — `if/else` *is*
the ternary. The `else` is mandatory when you use the value: without it the
expression would have no value on the false path.

## `while`

```zig
const std = @import("std");

pub fn main() void {
    var i: usize = 0;
    while (i < 3) {
        std.debug.print("plain {d}\n", .{i});
        i += 1;
    }

    var j: usize = 0;
    while (j < 3) : (j += 1) {
        std.debug.print("cont-expr {d}\n", .{j});
    }

    var k: usize = 3;
    while (k > 0) : (k -= 1) std.debug.print("down {d}\n", .{k});
}
```

```
plain 0
plain 1
plain 2
cont-expr 0
cont-expr 1
cont-expr 2
down 3
down 2
down 1
```

The `: (j += 1)` part is the **continue expression**. It runs after every
iteration *including* iterations ended by `continue`, which is the bug that
hand-written increment-at-the-bottom loops keep producing. Zig has no `for (;;)`
C-style loop; this is its replacement.

## `for` over ranges and slices

```zig
const std = @import("std");

pub fn main() void {
    for (0..4) |i| std.debug.print("range {d}\n", .{i});

    const items = [_][]const u8{ "a", "b", "c" };

    for (items) |s| std.debug.print("item {s}\n", .{s});

    for (items, 0..) |s, i| std.debug.print("{d}: {s}\n", .{ i, s });

    const nums = [_]i32{ 10, 20, 30 };
    for (items, nums) |s, n| std.debug.print("{s}={d}\n", .{ s, n });
}
```

```
range 0
range 1
range 2
range 3
item a
item b
item c
0: a
1: b
2: c
a=10
b=20
c=30
```

Four forms of the same keyword:

| Form | Iterates |
|---|---|
| `for (0..n) \|i\|` | the range `0` to `n-1`; `i` is a `usize` |
| `for (slice) \|item\|` | each element, **by value** |
| `for (slice, 0..) \|item, i\|` | element and index together |
| `for (a, b) \|x, y\|` | two sequences in lockstep |

`0..` in the multi-object form is a range with no end — it just counts along
with whatever it is paired with. And a length mismatch is caught at **compile
time**, not with a silent short iteration:

```zig
const std = @import("std");

pub fn main() void {
    const a = [_]i32{ 1, 2, 3 };
    const b = [_]i32{ 1, 2 };
    for (a, b) |x, y| std.debug.print("{d}{d}\n", .{ x, y });
}
```

```
error: non-matching for loop lengths
    for (a, b) |x, y| std.debug.print("{d}{d}\n", .{ x, y });
    ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
note: length 3 here
note: length 2 here
```

## By value vs by pointer

```zig
const std = @import("std");

pub fn main() void {
    var nums = [_]i32{ 1, 2, 3 };

    for (nums) |n| std.debug.print("copy {d}\n", .{n});

    for (&nums) |*n| n.* *= 10;

    for (nums) |n| std.debug.print("after {d}\n", .{n});

    const slice = nums[1..];
    for (slice) |n| std.debug.print("slice {d}\n", .{n});
}
```

```
copy 1
copy 2
copy 3
after 10
after 20
after 30
slice 20
slice 30
```

`|n|` gives you a copy — assigning to it would be pointless, and Zig makes the
captured value `const` so you cannot try. To mutate, iterate over the address
with `for (&nums)` and capture a pointer with `|*n|`, then write through it with
`n.*`. This is the same `.*` dereference you will see on every pointer in
module 03.

## `break` and `continue`

```zig
const std = @import("std");

pub fn main() void {
    var sum: i32 = 0;
    for (1..11) |i| {
        const n: i32 = @intCast(i);
        if (@mod(n, 2) == 0) continue;
        if (n > 7) break;
        sum += n;
        std.debug.print("adding {d}, sum now {d}\n", .{ n, sum });
    }
    std.debug.print("final {d}\n", .{sum});
}
```

```
adding 1, sum now 1
adding 3, sum now 4
adding 5, sum now 9
adding 7, sum now 16
final 16
```

`for (1..11)` yields `usize`, so `@intCast` converts it before the signed
arithmetic — the explicit-cast rule from lesson 2.3 applies inside loops too.

## `break` with a value

A loop is an expression as well. `break value` makes the loop evaluate to that
value, and the `else` clause supplies the result when the loop finished without
breaking:

```zig
const std = @import("std");

pub fn main() void {
    var i: usize = 0;
    const found = while (i < 10) : (i += 1) {
        if (i * i > 20) break i;
    } else 999;

    std.debug.print("found {d}\n", .{found});

    const nums = [_]i32{ 2, 4, 6 };
    const odd = for (nums) |n| {
        if (@mod(n, 2) == 1) break n;
    } else -1;

    std.debug.print("first odd {d}\n", .{odd});
}
```

```
found 5
first odd -1
```

`found` is `5` because `5 * 5 > 20` broke out with `i`. `first odd` is `-1`
because the loop ran out of elements without ever breaking, so the `else`
branch produced the result. This "search, with a default" pattern replaces
the `found` boolean flag you would write in Go. Lesson 2.6 goes further with it.

## `inline for`

```zig
const std = @import("std");

const Shape = enum { circle, square, triangle };

pub fn main() void {
    inline for (.{ Shape.circle, Shape.square, Shape.triangle }) |s| {
        std.debug.print("{t} has {d} sides\n", .{ s, comptime sides(s) });
    }

    var total: usize = 0;
    inline for (0..3) |i| total += i * i;
    std.debug.print("sum of squares = {d}\n", .{total});
}

fn sides(s: Shape) usize {
    return switch (s) {
        .circle => 0,
        .square => 4,
        .triangle => 3,
    };
}
```

```
circle has 0 sides
square has 4 sides
triangle has 3 sides
sum of squares = 5
```

`inline for` is **unrolled by the compiler**: there is no loop in the binary,
just three copies of the body with `s` substituted. That is what lets `sides(s)`
be evaluated at compile time here, and it is the only way to iterate over things
whose *type* changes per iteration — a tuple of mixed types, or a struct's
fields. Module 06 uses it heavily for reflection.

Use plain `for` for data. Use `inline for` when the iteration must vanish at
compile time; unrolling a large range bloats the binary.

---

Next: [2.5 — Switch](./05-switch.md)
