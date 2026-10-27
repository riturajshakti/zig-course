# 2.1 — Variables

```zig
const std = @import("std");

pub fn main() void {
    const name = "zig";
    var count: u32 = 3;
    count += 1;

    std.debug.print("{s} {d}\n", .{ name, count });
}
```

```
zig 4
```

`const` is immutable, `var` is mutable. That much is familiar. What is not:
**Zig rejects a `var` you never mutate.**

## The compiler polices your choice

```zig
const std = @import("std");

pub fn main() void {
    const count: u32 = 3;
    count = 4;
    std.debug.print("{d}\n", .{count});
}
```

```
error: cannot assign to constant
    count = 4;
    ^~~~~
```

Expected. Now the other direction — a `var` that is never reassigned:

```zig
const std = @import("std");

pub fn main() void {
    var count: u32 = 3;
    std.debug.print("{d}\n", .{count});
}
```

```
error: local variable is never mutated
    var count: u32 = 3;
        ^~~~~
note: consider using 'const'
```

A **compile error**, not a lint warning. In Go this is a style preference; in
Zig `var` is a claim about the code that the compiler verifies. The default is
`const`, and you reach for `var` only when you are about to mutate.

## Unused is also an error

```zig
const std = @import("std");

pub fn main() void {
    const unused: i32 = 1;
    std.debug.print("hi\n", .{});
}
```

```
error: unused local constant
    const unused: i32 = 1;
          ^~~~~~
```

The escape hatch is `_ = x;` — discard it explicitly:

```zig
const std = @import("std");

pub fn main() void {
    const unused: i32 = 1;
    _ = unused;
    std.debug.print("hi\n", .{});
}
```

```
hi
```

You will type `_ = x;` constantly while sketching code. It is not a hack; it is
how you tell the compiler "yes, on purpose."

| Declaration | Result |
|---|---|
| `const x = 1;` never read | **error:** unused local constant |
| `var x: u8 = 1;` never written | **error:** local variable is never mutated |
| `var x: u8 = 1;` never read | **error:** unused local variable |
| `const x = 1; _ = x;` | fine |

## Type inference and what it infers

```zig
const std = @import("std");

pub fn main() void {
    const a = 42;
    const b: u8 = 42;
    const c = 4.5;
    const d: f32 = 4.5;
    const e = "zig";
    const f = true;
    const g = @as(i64, 42);

    std.debug.print("{s}\n", .{@typeName(@TypeOf(a))});
    std.debug.print("{s}\n", .{@typeName(@TypeOf(b))});
    std.debug.print("{s}\n", .{@typeName(@TypeOf(c))});
    std.debug.print("{s}\n", .{@typeName(@TypeOf(d))});
    std.debug.print("{s}\n", .{@typeName(@TypeOf(e))});
    std.debug.print("{s}\n", .{@typeName(@TypeOf(f))});
    std.debug.print("{s}\n", .{@typeName(@TypeOf(g))});
}
```

```
comptime_int
u8
comptime_float
f32
*const [3:0]u8
bool
i64
```

Three of these deserve attention:

- **`comptime_int` / `comptime_float`** — an unannotated number literal is not
  `i32`. It is an arbitrary-precision compile-time value that has no runtime
  representation at all. It becomes a concrete type the moment you store it in
  something typed, pass it to a function, or `@as` it.
- **`*const [3:0]u8`** — a string literal is a *pointer to a 3-byte,
  null-terminated array*, not a `string` type. Zig has no string type.
- **`@as(T, v)`** is the explicit-coercion builtin. It is how you pin a literal's
  type without writing a full declaration annotation.

`comptime_int` has no width, so it cannot overflow:

```zig
const std = @import("std");

pub fn main() void {
    const big = 1 << 100;
    const sum = big + 1;
    std.debug.print("{d}\n", .{sum});
    std.debug.print("{s}\n", .{@typeName(@TypeOf(sum))});
}
```

```
1267650600228229401496703205377
comptime_int
```

A 101-bit integer, computed by the compiler, printed as a constant. Try to put
that in a `u64` and you get a compile error instead of a wrong answer.

## `undefined` — Zig's sharpest edge

`undefined` means "allocate the storage, write nothing to it." It is not `null`,
not zero, not a sentinel. Reading it before you assign is **undefined behavior**.

```zig
const std = @import("std");

pub fn main() void {
    var x: i32 = undefined;
    std.debug.print("before: {d}\n", .{x});
    x = 7;
    std.debug.print("after:  {d}\n", .{x});
}
```

```
before: -1431655766
after:  7
```

That `-1431655766` is `0xAAAAAAAA` read as a signed integer. In Debug builds Zig
fills undefined memory with the repeating byte `0xAA` precisely so that garbage
looks obviously wrong in a debugger. **That is a debugging aid, not a guarantee.**
The same program optimized:

```sh
zig run -O ReleaseFast undef.zig
```

```
before: 0
after:  7
```

Same source, different value. This is what undefined behavior looks like in
practice: the program is not wrong *today*, it is wrong *whenever the optimizer
changes its mind*.

So why does `undefined` exist? Because zeroing a buffer you are about to
overwrite is waste, and Zig refuses to do it behind your back:

```zig
const std = @import("std");

pub fn main() void {
    var buf: [16]u8 = undefined;
    const written = std.fmt.bufPrint(&buf, "{d}-{d}", .{ 7, 42 }) catch unreachable;
    std.debug.print("{s} ({d} of {d} bytes)\n", .{ written, written.len, buf.len });
}
```

```
7-42 (4 of 16 bytes)
```

The other twelve bytes are still garbage, and that is correct — nobody reads
them. The rule: `undefined` is for storage you will fill before you read.

| Expression | Means |
|---|---|
| `var x: i32 = undefined;` | storage reserved, contents are garbage |
| `var x: i32 = 0;` | storage reserved, contents are zero |
| `var x: ?i32 = null;` | an optional that is currently empty |
| `const x: i32 = undefined;` | legal but useless — you can never assign it |

## Shadowing is illegal

In JavaScript and Go an inner scope may reuse an outer name. Zig refuses:

```zig
const std = @import("std");

pub fn main() void {
    const outer: i32 = 1;
    {
        const outer: i32 = 2;
        std.debug.print("{d}\n", .{outer});
    }
    std.debug.print("{d}\n", .{outer});
}
```

```
error: local constant 'outer' shadows local constant from outer scope
        const outer: i32 = 2;
              ^~~~~
note: previous declaration here
    const outer: i32 = 1;
          ^~~~~
```

No `let`-vs-`var` subtleties, no "which `outer` is this?" A name in scope means
one thing. Pick a different name:

```zig
const std = @import("std");

pub fn main() void {
    const outer: i32 = 1;
    {
        const inner: i32 = 2;
        std.debug.print("{d}\n", .{inner});
    }
    std.debug.print("{d}\n", .{outer});
}
```

```
2
1
```

Container-level (file-scope) declarations are the one exception: a local *may*
share a name with a file-scope declaration, because the two are unambiguous.

## File scope

```zig
const std = @import("std");

const greeting = "hello";
const max_items: u32 = 100;
var counter: u32 = 0;

pub fn main() void {
    counter += 1;
    std.debug.print("{s} {d} {d}\n", .{ greeting, max_items, counter });
}
```

```
hello 100 1
```

File-scope declarations are order-independent (`main` can use `greeting`
declared above or below it) and are **not** subject to the unused check — a
`pub const` is part of your module's API whether this file uses it or not.

A file-scope `var` is a global with static storage. It works, and it is exactly
as unfriendly to concurrency as a global in any other language.

## `comptime var` — a variable that exists only during compilation

```zig
const std = @import("std");

pub fn main() void {
    comptime var n: usize = 0;
    inline for (0..4) |i| n += i;

    const fixed: [n]u8 = @splat('#');
    std.debug.print("n = {d}, {s}\n", .{ n, fixed });
    std.debug.print("type of n: {s}\n", .{@typeName(@TypeOf(n))});
}
```

```
n = 6, ######
type of n: usize
```

`n` is a real `usize` that the compiler mutates while compiling, then burns into
the binary as the constant `6`. That is why `[n]u8` is legal: array lengths must
be known at compile time, and `n` is.

Swap `comptime var` for a plain `var` and the array length breaks:

```zig
const std = @import("std");

pub fn main() void {
    var n: usize = 16;
    n += 1;
    const buf: [n]u8 = undefined;
    std.debug.print("{d}\n", .{buf.len});
}
```

```
error: unable to resolve comptime value
    const buf: [n]u8 = undefined;
                ^
note: types must be comptime-known
```

**A type is a value in Zig, and values that form types must be comptime-known.**
That single rule explains most "unable to resolve comptime value" errors you will
hit. Module 06 builds generics out of it.

| Form | Known at | Lives in the binary as |
|---|---|---|
| `const x = 1;` | compile time | an inlined constant |
| `var x: u8 = 1;` | runtime | stack storage |
| `comptime var x: u8 = 1;` | compile time, mutable while compiling | an inlined constant |
| `const x: u8 = undefined;` | compile time | nothing useful |

---

Next: [2.2 — Numbers](./02-numbers.md)
