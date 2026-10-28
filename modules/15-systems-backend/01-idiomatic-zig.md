# 15.1 — Idiomatic Zig

What a reviewer checks first, in order:

| # | They look at | They want to see |
|---|---|---|
| 1 | the **signature** | allocator first, ownership readable without the body |
| 2 | `init` / `deinit` | symmetric, `deinit` takes back the same allocator |
| 3 | every allocation | a `defer` or `errdefer` on the next line |
| 4 | the return type | `!T` or `?T`, never `-1` / `null` as a magic value |
| 5 | names | `init`/`deinit` for values, `create`/`destroy` for pointers |
| 6 | `anytype` | used where it buys something, not everywhere |

## 1. Naming: `init`/`deinit` vs `create`/`destroy`

```zig
const std = @import("std");

const Buffer = struct {
    bytes: []u8,
    len: usize,

    fn init(gpa: std.mem.Allocator, cap: usize) !Buffer {
        return .{ .bytes = try gpa.alloc(u8, cap), .len = 0 };
    }

    fn deinit(b: *Buffer, gpa: std.mem.Allocator) void {
        gpa.free(b.bytes);
        b.* = undefined;
    }

    fn push(b: *Buffer, byte: u8) error{NoSpace}!void {
        if (b.len == b.bytes.len) return error.NoSpace;
        b.bytes[b.len] = byte;
        b.len += 1;
    }

    fn slice(b: *const Buffer) []const u8 {
        return b.bytes[0..b.len];
    }
};

const Node = struct {
    value: u32,

    fn create(gpa: std.mem.Allocator, value: u32) !*Node {
        const n = try gpa.create(Node);
        n.* = .{ .value = value };
        return n;
    }

    fn destroy(n: *Node, gpa: std.mem.Allocator) void {
        gpa.destroy(n);
    }
};

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer std.debug.print("deinit status: {t}\n", .{dbg.deinit()});
    const gpa = dbg.allocator();

    var b = try Buffer.init(gpa, 4);
    defer b.deinit(gpa);
    for ("abcd") |c| try b.push(c);
    std.debug.print("buffer    : \"{s}\"\n", .{b.slice()});
    std.debug.print("push full : {!}\n", .{b.push('e')});

    const n = try Node.create(gpa, 42);
    defer n.destroy(gpa);
    std.debug.print("node      : {d}\n", .{n.value});
}
```

```
buffer    : "abcd"
push full : error.NoSpace
node      : 42
deinit status: ok
```

| Convention | Means | stdlib examples |
|---|---|---|
| `init` → `T` | returns a **value**; caller owns the struct | `ArrayList`, `Arena`, `Thread.Mutex` |
| `deinit(self, gpa)` | releases what `init` acquired | all 0.17 unmanaged containers |
| `create` → `*T` | returns a **heap pointer** | `Allocator.create`, `Thread.spawn`-alikes |
| `destroy(ptr, gpa)` | frees that pointer | `Allocator.destroy` |
| `.empty` / `.init` constant | no allocation at construction | `ArrayList`, `DebugAllocator` |

Two details reviewers notice:

- **`b.* = undefined;` in `deinit`.** After this, touching the struct is a
  detectable bug in Debug instead of a quiet use-after-free. Costs nothing in
  release builds.
- **`slice()` takes `*const Buffer`.** A method that doesn't mutate says so.

## 2. The allocator is the first parameter — always

```zig
fn load(gpa: std.mem.Allocator, path: []const u8) ![]u8   // yes
fn load(path: []const u8, gpa: std.mem.Allocator) ![]u8   // no
```

| Rule | Why |
|---|---|
| allocator **first** | matches the whole stdlib; grep-able |
| `io` **second** if present | `std.Io` ordering (but see the exceptions in 8.x) |
| the thing being operated on (`self`) before both when it's a method | `b.deinit(gpa)` reads left-to-right |
| **never store an allocator in a container type** | that's what "unmanaged" means in 0.17 |

A function with no allocator parameter **cannot allocate**. That's the point:
the signature is the audit.

## 3. `errdefer` discipline

```zig
const std = @import("std");

const Conn = struct {
    host: []u8,
    user: []u8,
    id: u32,

    fn open(gpa: std.mem.Allocator, host: []const u8, user: []const u8) !Conn {
        const h = try gpa.dupe(u8, host);
        errdefer gpa.free(h);
        const u = try gpa.dupe(u8, user);
        errdefer gpa.free(u);
        if (user.len == 0) return error.NoUser;
        return .{ .host = h, .user = u, .id = 1 };
    }

    fn close(c: *Conn, gpa: std.mem.Allocator) void {
        gpa.free(c.host);
        gpa.free(c.user);
        c.* = undefined;
    }
};

fn collect(gpa: std.mem.Allocator, n: u32) ![][]u8 {
    var list: std.ArrayList([]u8) = .empty;
    errdefer {
        for (list.items) |s| gpa.free(s);
        list.deinit(gpa);
    }
    for (0..n) |i| {
        const s = try std.fmt.allocPrint(gpa, "item-{d}", .{i});
        errdefer gpa.free(s);
        try list.append(gpa, s);
    }
    if (n > 3) return error.TooMany;
    return list.toOwnedSlice(gpa);
}

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer std.debug.print("deinit status: {t}\n", .{dbg.deinit()});
    const gpa = dbg.allocator();

    var c = try Conn.open(gpa, "db.local", "raj");
    defer c.close(gpa);
    std.debug.print("conn ok   : {s}@{s}\n", .{ c.user, c.host });
    std.debug.print("conn fail : {!}\n", .{Conn.open(gpa, "db.local", "")});

    const ok = try collect(gpa, 3);
    defer {
        for (ok) |s| gpa.free(s);
        gpa.free(ok);
    }
    std.debug.print("collect(3): {d} items, first=\"{s}\"\n", .{ ok.len, ok[0] });
    if (collect(gpa, 9)) |_| {} else |e| std.debug.print("collect(9): {t}\n", .{e});
}
```

```
conn ok   : raj@db.local
conn fail : error.NoUser
collect(3): 3 items, first="item-0"
collect(9): TooMany
deinit status: ok
```

**`deinit status: ok` after two deliberate failures** is the whole point. Three
different `errdefer` shapes, each covering a different scope:

| Shape | Covers | Note |
|---|---|---|
| `errdefer gpa.free(h);` | the one allocation above it | one per allocation, immediately after |
| `errdefer { for (list.items) \|s\| gpa.free(s); list.deinit(gpa); }` | the container **and** its contents | the function-wide net |
| `errdefer gpa.free(s);` inside the loop | the gap between `allocPrint` and `append` | without it, a failing `append` leaks `s` |

That inner `errdefer` is the one people forget. `try list.append(gpa, s)` can
fail *after* `s` exists but *before* the list owns it — a one-allocation leak
only under memory pressure, which is exactly when you can't afford it.

**`errdefer |e| { ... }` does not exist in 0.17.** Plain `errdefer` only; older
tutorials show the capture form.

## 4. Return errors, not sentinel values

```zig
const std = @import("std");

fn findSentinel(haystack: []const u8, needle: u8) i64 {
    for (haystack, 0..) |c, i| {
        if (c == needle) return @intCast(i);
    }
    return -1;
}

fn findOptional(haystack: []const u8, needle: u8) ?usize {
    return std.mem.indexOfScalar(u8, haystack, needle);
}

const ParseError = error{ Empty, BadDigit, Overflow };

fn parseU8(text: []const u8) ParseError!u8 {
    if (text.len == 0) return error.Empty;
    var acc: u16 = 0;
    for (text) |c| {
        if (c < '0' or c > '9') return error.BadDigit;
        acc = acc * 10 + (c - '0');
        if (acc > 255) return error.Overflow;
    }
    return @intCast(acc);
}

pub fn main() !void {
    std.debug.print("sentinel  'x' in \"abc\" -> {d}\n", .{findSentinel("abc", 'x')});
    std.debug.print("optional  'x' in \"abc\" -> {?d}\n", .{findOptional("abc", 'x')});
    std.debug.print("optional  'b' in \"abc\" -> {?d}\n", .{findOptional("abc", 'b')});

    for ([_][]const u8{ "7", "255", "256", "1x", "" }) |t| {
        std.debug.print("parseU8(\"{s}\") -> {!d}\n", .{ t, parseU8(t) });
    }
}
```

```
sentinel  'x' in "abc" -> -1
optional  'x' in "abc" -> null
optional  'b' in "abc" -> 1
parseU8("7") -> 7
parseU8("255") -> 255
parseU8("256") -> error.Overflow
parseU8("1x") -> error.BadDigit
parseU8("") -> error.Empty
```

`findSentinel` returns `i64` so it has room for `-1`, and the caller can ignore
the check. `findOptional` returns `?usize` — the compiler makes you unwrap it.

| Situation | Return | Not |
|---|---|---|
| "not found" is normal | `?T` | `-1`, `0`, `null` pointer |
| the operation failed | `E!T` | a bool plus an out-param |
| several distinct failures | a **named error set** | one generic `error.Failed` |
| cannot fail | `T` | `!T` "just in case" |

**Name the error set explicitly** (`ParseError!u8`, not `!u8`) on any function
a caller will `switch` on. An inferred error set grows silently as you add code,
so a caller's exhaustive `switch` breaks in a future commit. Inferred sets are
fine for internal helpers and `main`.

## 5. Take a writer, not a buffer

```zig
const std = @import("std");

fn writeReport(w: *std.Io.Writer, rows: []const u32) !void {
    for (rows) |r| try w.print("row {d}\n", .{r});
    try w.print("total {d}\n", .{rows.len});
}

fn sumAny(items: anytype) @TypeOf(items[0]) {
    var acc: @TypeOf(items[0]) = 0;
    for (items) |v| acc += v;
    return acc;
}

fn sumConcrete(comptime T: type, items: []const T) T {
    var acc: T = 0;
    for (items) |v| acc += v;
    return acc;
}

pub fn main() !void {
    var buf: [256]u8 = undefined;
    var w = std.Io.Writer.fixed(&buf);
    try writeReport(&w, &.{ 10, 20 });
    std.debug.print("{s}", .{w.buffered()});

    std.debug.print("sumAny array    = {d}\n", .{sumAny([_]u32{ 1, 2, 3 })});
    std.debug.print("sumAny slice    = {d}\n", .{sumAny(@as([]const u8, &.{ 4, 5 }))});
    std.debug.print("sumConcrete i32 = {d}\n", .{sumConcrete(i32, &.{ -1, 7 })});
}
```

```
row 10
row 20
total 2
sumAny array    = 6
sumAny slice    = 9
sumConcrete i32 = 6
```

`*std.Io.Writer` is the single most portable parameter type in 0.17. The same
`writeReport` works against a file, a socket, stdout, or — as here — a stack
array with no `io` at all. `std.Io.Writer.fixed(&buf)` returns a `Writer`
**directly** (no `.interface` field, unlike `File.Writer`), which makes it the
clean way to unit-test anything that writes.

### When to use `anytype`

| Use `anytype` | Use `comptime T: type` | Use a concrete type |
|---|---|---|
| duck-typed input (anything indexable, any writer-like) | the type appears in the **return** type or several params | you know the type |
| a format-args tuple | you want the error message to name `T` | **the default** |

`sumAny` accepts an array *or* a slice, which is genuinely convenient.
`sumConcrete(i32, …)` gives a better error message and a readable signature.
`anytype` moves type checking into the body, so the error surfaces at
instantiation with a stack of `referenced by:` lines — pay that cost only when
it buys flexibility you need.

## The review checklist

| Check | Smell if violated |
|---|---|
| allocator is parameter #1 | inconsistent with every stdlib call |
| every `init` has a matching `deinit` | leak, or an awkward manual teardown |
| `deinit` takes the allocator | can free with the wrong one |
| each allocation has `defer`/`errdefer` on the next line | leak on the error path |
| allocations between "created" and "owned" have an inner `errdefer` | leak under memory pressure only |
| public error sets are named | caller's `switch` breaks silently later |
| no sentinel return values | missed error checks |
| no stored allocators in containers | wasted bytes, wrong-allocator bugs |
| `*std.Io.Writer` instead of `[]u8 out` | untestable, inflexible |
| `anytype` only where it earns its keep | bad errors, slow compiles |
| a borrowed slice stored in a longer-lived struct is `dupe`d | dangling slice |

---

Next: [15.2 — Memory Layout](./02-memory-layout.md)
