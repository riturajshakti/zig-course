# 3.1 — Arrays

**The length is part of the type.** `[3]i32` and `[4]i32` are different types,
as unrelated as `i32` and `bool`. That one fact drives everything else.

## Creating

```zig
const std = @import("std");

pub fn main() void {
    const a: [3]i32 = .{ 1, 2, 3 };
    const b = [_]i32{ 10, 20, 30, 40 };
    var c: [4]u8 = undefined;
    c = .{ 'z', 'i', 'g', '!' };

    std.debug.print("a={any} a.len={d}\n", .{ a, a.len });
    std.debug.print("b={any} b.len={d}\n", .{ b, b.len });
    std.debug.print("c={s}\n", .{c});
    std.debug.print("@TypeOf(a)={s}\n", .{@typeName(@TypeOf(a))});
    std.debug.print("@TypeOf(b)={s}\n", .{@typeName(@TypeOf(b))});
}
```

```
a={ 1, 2, 3 } a.len=3
b={ 10, 20, 30, 40 } b.len=4
c=zig!
@TypeOf(a)=[3]i32
@TypeOf(b)=[4]i32
```

| Form | Meaning |
|---|---|
| `[3]i32` | the type: exactly 3 `i32`s, 12 bytes, no heap |
| `const a: [3]i32 = .{1,2,3}` | type on the left, so the literal can be anonymous |
| `[_]i32{10,20,30,40}` | `_` means "count them for me" → `[4]i32` |
| `var c: [4]u8 = undefined` | uninitialized; reading before writing is UB |
| `a.len` | a **compile-time** constant, not a stored field |

`.len` is baked into the type, so this is free at runtime — the compiler already
knows it.

## Length mismatches are compile errors

Deliberately broken:

```zig
const std = @import("std");

fn sum3(xs: [3]i32) i32 {
    return xs[0] + xs[1] + xs[2];
}

pub fn main() void {
    const three = [_]i32{ 1, 2, 3 };
    const four = [_]i32{ 1, 2, 3, 4 };
    std.debug.print("sum3(three)={d}\n", .{sum3(three)});
    std.debug.print("sum3(four)={d}\n", .{sum3(four)});
}
```

```
a2.zig:11:48: error: expected type '[3]i32', found '[4]i32'
    std.debug.print("sum3(four)={d}\n", .{sum3(four)});
                                               ^~~~
a2.zig:11:48: note: destination has length 3
a2.zig:11:48: note: source has length 4
```

A function that should accept *any* length takes a **slice** (`[]const i32`)
instead — that is lesson 3.2, and it is the reason slices exist.

## `++` concatenates, `@splat` repeats

```zig
const std = @import("std");

pub fn main() void {
    const zeros: [5]i32 = @splat(0);
    const dashes: [8]u8 = @splat('-');
    const joined = [_]i32{ 1, 2 } ++ [_]i32{ 3, 4, 5 };
    const greet = "hi " ++ "there";

    std.debug.print("zeros={any}\n", .{zeros});
    std.debug.print("dashes={s}\n", .{dashes});
    std.debug.print("joined={any} len={d}\n", .{ joined, joined.len });
    std.debug.print("greet={s}\n", .{greet});
}
```

```
zeros={ 0, 0, 0, 0, 0 }
dashes=--------
joined={ 1, 2, 3, 4, 5 } len=5
greet=hi there
```

> **`**` no longer exists.** Every tutorial you will find writes
> `[_]u8{'-'} ** 8` for a repeated array. Zig 0.17.0 removed the `**` operator
> entirely — it is not even a token any more, so the parser sees two separate
> `*`s and says something baffling:
>
> ```
> t.zig:3:26: error: binary operator '*' has whitespace on one side, but not the other
>     const z1 = [_]i32{0} ** 5;
>                          ^
> ```
>
> The replacement is **`@splat`**, which takes the element value and infers the
> count from the destination type. `++` was *not* removed.

`@splat` nests, which is how you fill a 2D array:

```zig
const std = @import("std");

pub fn main() void {
    const flat: [4]u8 = @splat('x');
    const grid: [2][3]u8 = @splat(@splat('.'));
    const pairs: [3][2]u8 = @splat("ab".*);

    std.debug.print("flat={s}\n", .{flat});
    std.debug.print("grid={any}\n", .{grid});
    std.debug.print("pairs as bytes={s}\n", .{@as([]const u8, @ptrCast(&pairs))});
}
```

```
flat=xxxx
grid={ { 46, 46, 46 }, { 46, 46, 46 } }
pairs as bytes=ababab
```

`"ab".*` dereferences the string literal pointer to get a `[2]u8` value — see
lesson 3.5.

## Iterating

```zig
const std = @import("std");

pub fn main() void {
    const a = [_]i32{ 5, 3, 9, 1, 7 };

    for (a) |v| std.debug.print("{d} ", .{v});
    std.debug.print("\n", .{});

    for (a, 0..) |v, i| std.debug.print("[{d}]={d} ", .{ i, v });
    std.debug.print("\n", .{});

    var doubled: [5]i32 = undefined;
    for (&doubled, a) |*dst, v| dst.* = v * 2;
    std.debug.print("doubled={any}\n", .{doubled});

    var inplace = a;
    for (&inplace) |*p| p.* += 100;
    std.debug.print("inplace={any} original={any}\n", .{ inplace, a });
}
```

```
5 3 9 1 7 
[0]=5 [1]=3 [2]=9 [3]=1 [4]=7 
doubled={ 10, 6, 18, 2, 14 }
inplace={ 105, 103, 109, 101, 107 } original={ 5, 3, 9, 1, 7 }
```

| Loop | Gives you |
|---|---|
| `for (a) \|v\|` | each element **by value** (a copy) |
| `for (a, 0..) \|v, i\|` | value + index |
| `for (&a) \|*p\|` | a pointer, so `p.* = ...` mutates in place |
| `for (&dst, src) \|*d, s\|` | two arrays in lockstep; lengths must match |

**`var inplace = a` copied the whole array.** Arrays are values in Zig, not
references like a JS array or a Go slice. Assignment, passing to a function and
returning all copy every element. `original` is untouched above.

## Filling and copying: `@memset` / `@memcpy`

```zig
const std = @import("std");

pub fn main() void {
    var buf: [6]u8 = undefined;
    @memset(&buf, '.');
    std.debug.print("after memset: {s}\n", .{buf});

    @memcpy(buf[0..3], "abc");
    std.debug.print("after memcpy: {s}\n", .{buf});

    var nums: [4]i32 = undefined;
    @memset(&nums, -1);
    @memcpy(&nums, &[_]i32{ 7, 8, 9, 10 });
    std.debug.print("nums={any}\n", .{nums});
}
```

```
after memset: ......
after memcpy: abc...
nums={ 7, 8, 9, 10 }
```

Both are builtins taking **pointers or slices**, not arrays. `@memcpy` requires
the lengths to match exactly — that is why `buf[0..3]` is sliced to 3 bytes for
a 3-byte source.

## 2D arrays

```zig
const std = @import("std");

pub fn main() void {
    var grid: [3][4]u8 = @splat(@splat('.'));
    grid[1][2] = '#';
    grid[0][0] = 'A';

    for (grid) |row| std.debug.print("{s}\n", .{row});

    std.debug.print("rows={d} cols={d}\n", .{ grid.len, grid[0].len });

    const fixed = [2][3]i32{
        .{ 1, 2, 3 },
        .{ 4, 5, 6 },
    };
    std.debug.print("fixed={any}\n", .{fixed});
    std.debug.print("fixed[1][2]={d}\n", .{fixed[1][2]});
}
```

```
A...
..#.
....
rows=3 cols=4
fixed={ { 1, 2, 3 }, { 4, 5, 6 } }
fixed[1][2]=6
```

`[3][4]u8` reads right-to-left: 3 rows of `[4]u8`. It is one flat 12-byte block,
not an array of pointers.

## Array of structs

```zig
const std = @import("std");

const User = struct {
    name: []const u8,
    age: u8,
};

pub fn main() void {
    const users = [_]User{
        .{ .name = "ada", .age = 36 },
        .{ .name = "linus", .age = 54 },
        .{ .name = "ken", .age = 81 },
    };

    for (users) |u| std.debug.print("{s} is {d}\n", .{ u.name, u.age });

    var total: u32 = 0;
    for (users) |u| total += u.age;
    std.debug.print("count={d} total_age={d}\n", .{ users.len, total });

    var copy = users;
    copy[0].age = 99;
    std.debug.print("copy[0].age={d} users[0].age={d}\n", .{ copy[0].age, users[0].age });
}
```

```
ada is 36
linus is 54
ken is 81
count=3 total_age=171
copy[0].age=99 users[0].age=36
```

Same copy semantics, now visibly: mutating `copy` left `users` alone.

## Comparing

`==` does **not** work on arrays:

```zig
const std = @import("std");

pub fn main() void {
    const a = [_]i32{ 1, 2, 3 };
    const b = [_]i32{ 1, 2, 3 };
    std.debug.print("{}\n", .{a == b});
}
```

```
t.zig:6:33: error: operator == not allowed for type '[3]i32'
    std.debug.print("{}\n", .{a == b});
                              ~~^~~~
```

Use `std.mem`, passing `&array` so it coerces to a slice:

```zig
const std = @import("std");

pub fn main() void {
    const a = [_]i32{ 1, 2, 3 };
    const b = [_]i32{ 1, 2, 3 };
    const c = [_]i32{ 1, 2, 4 };

    std.debug.print("eql(a,b)      = {}\n", .{std.mem.eql(i32, &a, &b)});
    std.debug.print("eql(a,c)      = {}\n", .{std.mem.eql(i32, &a, &c)});
    std.debug.print("order(a,c)    = {t}\n", .{std.mem.order(i32, &a, &c)});
    std.debug.print("lessThan(a,c) = {}\n", .{std.mem.lessThan(i32, &a, &c)});
    std.debug.print("eql on bytes  = {}\n", .{std.mem.eql(u8, "abc", "abc")});
}
```

```
eql(a,b)      = true
eql(a,c)      = false
order(a,c)    = lt
lessThan(a,c) = true
eql on bytes  = true
```

## Operations table

| Operation | Code |
|---|---|
| create, explicit length | `const a: [3]i32 = .{1,2,3};` |
| create, inferred length | `const a = [_]i32{1,2,3};` |
| uninitialized | `var a: [3]i32 = undefined;` |
| fill with one value | `const a: [8]u8 = @splat('-');` |
| 2D fill | `var g: [3][4]u8 = @splat(@splat('.'));` |
| length | `a.len` (comptime known) |
| index | `a[0]`, `a[a.len - 1]` |
| last element | `a[a.len - 1]` |
| concatenate | `a ++ b` (comptime only) |
| copy | `var b = a;` — a real element-wise copy |
| set every byte | `@memset(&a, 0)` |
| copy into | `@memcpy(a[0..n], src)` |
| as a slice | `&a` or `a[0..]` |
| iterate | `for (a) \|v\|` |
| iterate with index | `for (a, 0..) \|v, i\|` |
| mutate in place | `for (&a) \|*p\| p.* += 1;` |
| compare | `std.mem.eql(T, &a, &b)` |
| order | `std.mem.order(T, &a, &b)` → `.lt`/`.eq`/`.gt` |
| element type | `@typeInfo(@TypeOf(a)).array.child` |
| byte size | `@sizeOf(@TypeOf(a))` |

Everything in `std.mem` takes slices, so in practice you write `&a` constantly.
That is the hand-off to the next lesson.

---

Next: [3.2 — Slices](./02-slices.md)
