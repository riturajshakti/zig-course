# 6.4 — Generic Containers

Everything from 6.2 and 6.3 combined: a type function returning a struct that
owns memory, with methods. This is how every container in the standard library
is written.

## `Stack(T)` from scratch

No `ArrayList` underneath — raw `alloc`/`realloc`/`free`, so you can see all of it:

```zig
const std = @import("std");

fn Stack(comptime T: type) type {
    return struct {
        const Self = @This();

        items: []T = &.{},
        len: usize = 0,

        fn push(s: *Self, gpa: std.mem.Allocator, v: T) !void {
            if (s.len == s.items.len) {
                const new_cap = if (s.items.len == 0) 4 else s.items.len * 2;
                s.items = try gpa.realloc(s.items, new_cap);
            }
            s.items[s.len] = v;
            s.len += 1;
        }

        fn pop(s: *Self) ?T {
            if (s.len == 0) return null;
            s.len -= 1;
            return s.items[s.len];
        }

        fn peek(s: Self) ?T {
            return if (s.len == 0) null else s.items[s.len - 1];
        }

        fn deinit(s: *Self, gpa: std.mem.Allocator) void {
            gpa.free(s.items);
            s.* = .{};
        }
    };
}

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer _ = gpa_s.deinit();
    const gpa = gpa_s.allocator();

    var s: Stack(i32) = .{};
    defer s.deinit(gpa);

    for ([_]i32{ 10, 20, 30 }) |v| try s.push(gpa, v);
    std.debug.print("len={d} cap={d} peek={?d}\n", .{ s.len, s.items.len, s.peek() });
    while (s.pop()) |v| std.debug.print("pop {d}\n", .{v});
    std.debug.print("empty pop = {?d}\n", .{s.pop()});

    var words: Stack([]const u8) = .{};
    defer words.deinit(gpa);
    try words.push(gpa, "zig");
    try words.push(gpa, "comptime");
    std.debug.print("peek={?s}\n", .{words.peek()});
}
```

```
len=3 cap=4 peek=30
pop 30
pop 20
pop 10
empty pop = null
peek=comptime
```

`Stack(i32)` and `Stack([]const u8)` are separate types from one definition, and
`DebugAllocator` reported no leak — so `deinit` is correct for both.

### The five conventions in that code

| Convention | Why |
|---|---|
| `const Self = @This()` | the struct has no name to refer to itself by |
| `items: []T = &.{}` | field defaults make `.{}` a valid empty value — no `init` needed |
| `s: *Self` on mutators, `s: Self` on readers | a reader that takes `*Self` can't be called on a `const` |
| `gpa` passed **per call**, not stored | the 0.17 unmanaged style; the container is smaller and can be `comptime` |
| `s.* = .{}` in `deinit` | leaves a reusable empty value instead of dangling pointers |

> **Methods are just functions in the struct's namespace.** `s.push(gpa, v)` is
> sugar for `Stack(i32).push(&s, gpa, v)`. There is no `self` keyword — the
> first parameter's name is up to you, and `s.` only works because its type
> matches.

## `Queue(T)` built on `ArrayList(T)`

Composing a generic container over another one — the usual case in real code:

```zig
const std = @import("std");

fn Queue(comptime T: type) type {
    return struct {
        const Self = @This();

        items: std.ArrayList(T) = .empty,
        head: usize = 0,

        fn enqueue(q: *Self, gpa: std.mem.Allocator, v: T) !void {
            try q.items.append(gpa, v);
        }

        fn dequeue(q: *Self) ?T {
            if (q.head == q.items.items.len) return null;
            const v = q.items.items[q.head];
            q.head += 1;
            return v;
        }

        fn count(q: Self) usize {
            return q.items.items.len - q.head;
        }

        fn deinit(q: *Self, gpa: std.mem.Allocator) void {
            q.items.deinit(gpa);
            q.* = .{};
        }
    };
}

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer _ = gpa_s.deinit();
    const gpa = gpa_s.allocator();

    var q: Queue([]const u8) = .{};
    defer q.deinit(gpa);

    try q.enqueue(gpa, "first");
    try q.enqueue(gpa, "second");
    try q.enqueue(gpa, "third");
    std.debug.print("count={d}\n", .{q.count()});
    std.debug.print("{?s} {?s}\n", .{ q.dequeue(), q.dequeue() });
    std.debug.print("count={d} then {?s} then {?s}\n", .{ q.count(), q.dequeue(), q.dequeue() });
}
```

```
count=3
first second
count=1 then third then null
```

`q.items.items` reads oddly but is exactly right: our field `items` is an
`ArrayList(T)`, whose own slice field is also called `items`.

> `.empty` as a default value is the 0.17 `ArrayList` initializer. There is no
> `ArrayList(T).init(gpa)` any more — the list holds no allocator, so an empty
> one is a compile-time constant and `Queue(T){}` needs no code to construct.

## `std.ArrayList(T)` is literally this

Not "similar to" — the same construction. From the standard library:

```zig
pub fn ArrayList(comptime T: type) type {
    return array_list.Aligned(T, null);
}
```

and `Aligned` is a type function returning a struct that opens with:

```zig
pub fn Aligned(comptime T: type, comptime alignment: ?mem.Alignment) type {
    return struct {
        const Self = @This();
        items: Slice,
        capacity: usize,
        ...
```

`comptime T: type`, `return struct`, `const Self = @This()`, a slice plus a
length, allocator per call. You just wrote `ArrayList`'s skeleton twice.

Worth knowing: `Aligned` starts by **normalizing** its arguments —

```zig
if (alignment) |a| {
    if (a.toByteUnits() == @alignOf(T)) {
        return Aligned(T, null);
    }
}
```

An explicit alignment equal to the natural one re-calls itself with `null`, so
`Aligned(u64, .@"8")` and `ArrayList(u64)` land on the **identical cached type**
instead of two incompatible ones. Canonicalize comptime arguments when several
spellings mean the same thing — it is the main gotcha of type caching.

## Nesting

Since `Stack(T)` is a type and `T` is any type, `Stack(Stack(u8))` just works:

```zig
const std = @import("std");

fn Stack(comptime T: type) type {
    return struct {
        const Self = @This();
        items: std.ArrayList(T) = .empty,

        fn push(s: *Self, gpa: std.mem.Allocator, v: T) !void {
            try s.items.append(gpa, v);
        }
        fn pop(s: *Self) ?T {
            return s.items.pop();
        }
        fn len(s: Self) usize {
            return s.items.items.len;
        }
        fn deinit(s: *Self, gpa: std.mem.Allocator) void {
            s.items.deinit(gpa);
        }
    };
}

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer _ = gpa_s.deinit();
    const gpa = gpa_s.allocator();

    var outer: Stack(Stack(u8)) = .{};
    defer {
        while (outer.pop()) |inner| {
            var m = inner;
            m.deinit(gpa);
        }
        outer.deinit(gpa);
    }

    for (0..3) |i| {
        var inner: Stack(u8) = .{};
        try inner.push(gpa, @intCast(i * 10));
        try inner.push(gpa, @intCast(i * 10 + 1));
        try outer.push(gpa, inner);
    }

    std.debug.print("outer.len = {d}\n", .{outer.len()});
    std.debug.print("{s}\n", .{@typeName(Stack(Stack(u8)))});
    for (outer.items.items, 0..) |inner, i| {
        std.debug.print("  inner[{d}] = {any}\n", .{ i, inner.items.items });
    }
}
```

```
outer.len = 3
c3.Stack(c3.Stack(u8))
  inner[0] = { 0, 1 }
  inner[1] = { 10, 11 }
  inner[2] = { 20, 21 }
```

`c3.Stack(c3.Stack(u8))` — the type name nests because it is built from the
call. Note what nesting costs: **a container of containers must free each
element**, which is why the `defer` block pops and deinits every inner stack
before deinitializing the outer one. `Stack(T)` cannot know whether `T` owns
memory, so ownership of the elements stays with you.

> `var m = inner;` is needed because `deinit` takes `*Self` and `inner` is a
> loop capture (a `const`). Copying the small struct and deiniting the copy is
> fine here — it owns the same heap pointer.

## Non-type comptime parameters

Comptime parameters aren't limited to types. A `usize` works, and gives you a
fixed-capacity container with no allocator at all:

```zig
const std = @import("std");

fn RingBuffer(comptime T: type, comptime capacity: usize) type {
    if (capacity == 0) @compileError("RingBuffer needs capacity > 0");
    return struct {
        const Self = @This();

        buf: [capacity]T = undefined,
        head: usize = 0,
        len: usize = 0,

        fn push(r: *Self, v: T) bool {
            if (r.len == capacity) return false;
            r.buf[(r.head + r.len) % capacity] = v;
            r.len += 1;
            return true;
        }

        fn pop(r: *Self) ?T {
            if (r.len == 0) return null;
            const v = r.buf[r.head];
            r.head = (r.head + 1) % capacity;
            r.len -= 1;
            return v;
        }
    };
}

pub fn main() void {
    var r: RingBuffer(u8, 3) = .{};
    std.debug.print("pushes: {} {} {} {}\n", .{ r.push(1), r.push(2), r.push(3), r.push(4) });
    std.debug.print("pop {?d}, push 9 -> {}\n", .{ r.pop(), r.push(9) });
    while (r.pop()) |v| std.debug.print("got {d}\n", .{v});

    std.debug.print("RingBuffer(u8,3)  = {d} bytes\n", .{@sizeOf(RingBuffer(u8, 3))});
    std.debug.print("RingBuffer(u64,8) = {d} bytes\n", .{@sizeOf(RingBuffer(u64, 8))});
    std.debug.print("same type as RingBuffer(u8,4)? {}\n", .{RingBuffer(u8, 3) == RingBuffer(u8, 4)});
}
```

```
pushes: true true true false
pop 1, push 9 -> true
got 2
got 3
got 9
RingBuffer(u8,3)  = 24 bytes
RingBuffer(u64,8) = 80 bytes
same type as RingBuffer(u8,4)? false
```

The capacity is in the **type**, so the buffer is inline, `@sizeOf` is exact, and
there is no heap and no failure path. `RingBuffer(u8, 3)` and `RingBuffer(u8, 4)`
are different types — capacity mismatches become compile errors instead of bugs.

> `% capacity` with a comptime-known capacity lets the compiler replace the
> division; a power-of-two capacity becomes a bitmask. The same expression with
> a runtime capacity would be a real `div` instruction.

## Checklist

| Decision | Default |
|---|---|
| allocator | parameter on each method, not a field |
| construction | field defaults + `.{}`, so no `init` is needed |
| self parameter | `*Self` to mutate, `Self` to read |
| cleanup | `deinit(self, gpa)`, ending with `self.* = .{}` |
| element cleanup | the caller's job — document it |
| fixed capacity | a `comptime capacity: usize` parameter, no allocator |
| canonical arguments | normalize equivalent spellings before `return struct` |

---

Next: [6.5 — Reflection](./05-reflection.md)
