# 4.5 — Optionals

`?T` is a type that holds a `T` or `null`. The important part is the inverse:
**`T` can never be null.** Nullability is spelled out in the type, so a plain
`*User` is a guaranteed-valid pointer and the compiler enforces it.

## The five operations

```zig
const std = @import("std");

pub fn main() void {
    const some: ?i32 = 7;
    const none: ?i32 = null;

    std.debug.print("some        = {?d}\n", .{some});
    std.debug.print("none        = {?d}\n", .{none});
    std.debug.print("some.?      = {d}\n", .{some.?});
    std.debug.print("none orelse = {d}\n", .{none orelse -1});
    std.debug.print("some == null= {}\n", .{some == null});
    std.debug.print("none == null= {}\n", .{none == null});
    std.debug.print("type        = {s}\n", .{@typeName(@TypeOf(some))});
    std.debug.print("sizeOf ?i32 = {d}, i32 = {d}\n", .{ @sizeOf(?i32), @sizeOf(i32) });
    std.debug.print("sizeOf ?*i32= {d}, *i32= {d}\n", .{ @sizeOf(?*i32), @sizeOf(*i32) });
}
```

```
some        = 7
none        = null
some.?      = 7
none orelse = -1
some == null= false
none == null= true
type        = ?i32
sizeOf ?i32 = 8, i32 = 4
sizeOf ?*i32= 8, *i32= 8
```

| Form | Does |
|---|---|
| `?T` | the type |
| `null` | the empty value |
| `x.?` | unwrap, **panic if null** |
| `x orelse d` | unwrap, or `d` |
| `x == null` | test without unwrapping |
| `if (x) \|v\|` | unwrap into a scope |
| `while (x) \|v\|` | loop while non-null |

Look at the last two lines. `?i32` costs 4 extra bytes for the "is it there" flag.
`?*i32` costs **nothing** — a pointer can't legally be address 0, so `null` is
encoded as 0. Same for slices and `?*T` in general. This is why optional pointers
are free in Zig and why there is no separate "nullable pointer" concept.

## `if (x) |v|` — the capture form

```zig
const std = @import("std");

fn find(hay: []const u32, needle: u32) ?usize {
    for (hay, 0..) |v, i| if (v == needle) return i;
    return null;
}

pub fn main() void {
    const xs = [_]u32{ 5, 9, 13 };

    if (find(&xs, 9)) |i| {
        std.debug.print("found at {d}\n", .{i});
    } else {
        std.debug.print("not found\n", .{});
    }

    if (find(&xs, 99)) |i| {
        std.debug.print("found at {d}\n", .{i});
    } else {
        std.debug.print("not found\n", .{});
    }

    const idx = find(&xs, 13) orelse 0;
    std.debug.print("idx = {d}\n", .{idx});

    const label = if (find(&xs, 5)) |i| i else std.math.maxInt(usize);
    std.debug.print("label = {d}\n", .{label});
}
```

```
found at 1
not found
idx = 2
label = 0
```

`|i|` is `usize`, not `?usize` — inside the block it is already unwrapped. This is
Rust's `if let`, with no `Option` enum and no `match`.

Note `find` returns `?usize` and **not** an error. "Not present" is not a failure;
see [4.6](./06-error-vs-optional.md).

## `while (x) |v|` — the iterator protocol

Zig has no `Iterator` interface. An iterator is any type with a `next()` that
returns `?T`, and `while` unwraps it:

```zig
const std = @import("std");

const Counter = struct {
    n: u32 = 0,
    limit: u32,
    fn next(c: *Counter) ?u32 {
        if (c.n >= c.limit) return null;
        c.n += 1;
        return c.n * c.n;
    }
};

pub fn main() void {
    var c: Counter = .{ .limit = 4 };
    while (c.next()) |sq| std.debug.print("{d} ", .{sq});
    std.debug.print("\n", .{});

    var it = std.mem.splitScalar(u8, "ada,bob,cy", ',');
    while (it.next()) |name| std.debug.print("[{s}]", .{name});
    std.debug.print("\n", .{});

    var tok = std.mem.tokenizeAny(u8, "  a  b   c ", " ");
    while (tok.next()) |t| std.debug.print("<{s}>", .{t});
    std.debug.print("\n", .{});
}
```

```
1 4 9 16 
[ada][bob][cy]
<a><b><c>
```

That is the entire convention — stdlib iterators (`splitScalar`, `tokenizeAny`,
hash map iterators, dir walkers) all return `?T` from `next()`, so they all work
with `while (it.next()) |v|`. Write a `next()` returning `?T` and yours does too.

## Optional pointers: there are no null pointers

```zig
const std = @import("std");

const Node = struct {
    value: u32,
    next: ?*Node = null,
};

pub fn main() void {
    var c: Node = .{ .value = 3 };
    var b: Node = .{ .value = 2, .next = &c };
    var a: Node = .{ .value = 1, .next = &b };

    var cur: ?*Node = &a;
    while (cur) |n| {
        std.debug.print("{d} -> ", .{n.value});
        cur = n.next;
    }
    std.debug.print("null\n", .{});

    const p: ?*Node = null;
    std.debug.print("p == null: {}\n", .{p == null});
    if (p) |n| std.debug.print("{d}\n", .{n.value}) else std.debug.print("nothing to deref\n", .{});
}
```

```
1 -> 2 -> 3 -> null
p == null: true
nothing to deref
```

The linked list is the classic case: `next: ?*Node`. The `while (cur) |n|` loop
gives you a `*Node` that **cannot** be null, so `n.value` needs no check.

This is the C/Java/Go bug class that Zig deletes:

| Language | A pointer/reference |
|---|---|
| C | `Node*` may be `NULL`; dereferencing is undefined behaviour |
| Java | any object ref may be null; `NullPointerException` at runtime |
| Go | `*Node` may be nil; nil deref panics |
| Zig | **`*Node` is never null.** Only `?*Node` can be, and you cannot deref it without unwrapping |

You never write a null check on a `*T`, because the type says it cannot happen.
And `?*T` has no space overhead, so there's no performance reason to cheat.

## `.?` panics — deliberately

```zig
const std = @import("std");

pub fn main() void {
    const none: ?i32 = null;
    std.debug.print("about to unwrap null...\n", .{});
    std.debug.print("{d}\n", .{none.?});
}
```

**Deliberate failure.** Exit code **134** (SIGABRT):

```
about to unwrap null...
thread 16430436 panic: attempt to use null value
/tmp/unwrap.zig:6:36: 0x10490382f in main (unwrap)
    std.debug.print("{d}\n", .{none.?});
                                   ^
/opt/.../std/start.zig:788:64: 0x1049037bf in callMain (unwrap)
    if (fn_info.param_types.len == 0) return wrapMain(root.main());
                                                               ^
???:?:?: 0x180407dff in start (/usr/lib/dyld)
```

`attempt to use null value` — not an error you can catch, a **panic**. Use `.?`
only when null is impossible by construction; otherwise `orelse` or `if |v|`.

## `orelse unreachable`

```zig
const std = @import("std");

fn mustFind(hay: []const u8, c: u8) usize {
    return std.mem.indexOfScalar(u8, hay, c) orelse unreachable;
}

pub fn main() void {
    std.debug.print("index of 'g' = {d}\n", .{mustFind("zig", 'g')});

    var buf: [4]u8 = undefined;
    const maybe_slice: ?[]u8 = buf[0..2];
    const s = maybe_slice orelse unreachable;
    s[0] = 'o';
    s[1] = 'k';
    std.debug.print("s = {s}\n", .{s});

    const n: ?u32 = 3;
    const doubled = (n orelse unreachable) * 2;
    std.debug.print("doubled = {d}\n", .{doubled});
}
```

```
index of 'g' = 2
s = ok
doubled = 6
```

`orelse unreachable` and `.?` do the same thing in Debug/ReleaseSafe (panic), but
`unreachable` documents the claim at the call site and in ReleaseFast becomes
undefined behaviour rather than a check. More in [4.6](./06-error-vs-optional.md).

## Nested optionals `??`

`??T` is a real, distinct type — "no answer" vs "the answer is nothing":

```zig
const std = @import("std");

pub fn main() void {
    const nested: ??i32 = 5;
    const outer_null: ??i32 = null;
    const inner_null: ??i32 = @as(?i32, null);

    std.debug.print("nested.?        = {?d}\n", .{nested.?});
    std.debug.print("nested.?.?      = {d}\n", .{nested.?.?});
    std.debug.print("outer_null null?  outer={}\n", .{outer_null == null});
    std.debug.print("inner_null null?  outer={} inner={}\n", .{ inner_null == null, inner_null.? == null });

    const a = (nested orelse null) orelse -1;
    const b = (outer_null orelse null) orelse -1;
    const c = (inner_null orelse null) orelse -1;
    std.debug.print("collapsed: {d} {d} {d}\n", .{ a, b, c });
}
```

```
nested.?        = 5
nested.?.?      = 5
outer_null null?  outer=true
inner_null null?  outer=false inner=true
collapsed: 5 -1 -1
```

`nested.?` gives `?i32`; a second `.?` gives `i32`. A cache lookup returning
`??T` genuinely distinguishes "cache miss" (outer null) from "cached, and the
answer is no value" (inner null). Rare, but when you need it, it's in the type —
not a sentinel value. `orelse null` collapses one level.

## Optional struct fields

```zig
const std = @import("std");

const Config = struct {
    host: []const u8,
    port: u16 = 8080,
    tls_cert: ?[]const u8 = null,
    max_conns: ?u32 = null,
};

fn describe(c: Config) void {
    std.debug.print("{s}:{d} tls={?s} max={?d}\n", .{ c.host, c.port, c.tls_cert, c.max_conns });
}

pub fn main() void {
    describe(.{ .host = "localhost" });
    describe(.{ .host = "api.internal", .tls_cert = "/etc/cert.pem", .max_conns = 512 });

    const c: Config = .{ .host = "x", .max_conns = null };
    const effective = c.max_conns orelse 128;
    std.debug.print("effective max_conns = {d}\n", .{effective});

    if (c.tls_cert) |path| {
        std.debug.print("loading {s}\n", .{path});
    } else {
        std.debug.print("plaintext\n", .{});
    }
}
```

```
localhost:8080 tls=null max=null
api.internal:8080 tls=/etc/cert.pem max=512
effective max_conns = 128
plaintext
```

`?T = null` as a field default is the idiomatic "unset, and the caller may not
care" — distinct from `port: u16 = 8080`, which is "unset means this value".
`orelse` at the point of use turns an unset option into a concrete default, and
`{?s}` / `{?d}` print optionals without unwrapping, which is ideal for config
dumps and logs.

| Field style | Means |
|---|---|
| `host: []const u8` | required, caller must supply it |
| `port: u16 = 8080` | optional **with** a baked-in default |
| `tls_cert: ?[]const u8 = null` | genuinely absent; behaviour differs when unset |
| `max_conns: ?u32 = null` | absent; resolve with `orelse` where it's used |

## Cheat sheet

| Expression | Result |
|---|---|
| `const x: ?i32 = 7;` | holds 7 |
| `const x: ?i32 = null;` | empty |
| `x.?` | `i32`, panics on null |
| `x orelse 0` | `i32` |
| `x orelse return` | bail out of the function |
| `x orelse unreachable` | assert non-null |
| `x == null` / `!= null` | test |
| `if (x) \|v\| ... else ...` | branch with unwrap |
| `while (it.next()) \|v\|` | iterate |
| `{?d}` `{?s}` | print the payload or `null` |
| `@sizeOf(?*T) == @sizeOf(*T)` | optional pointers are free |

---

Next: [4.6 — Error vs Optional](./06-error-vs-optional.md)
