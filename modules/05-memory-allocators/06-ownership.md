# 5.6 — Ownership Rules

Zig has no borrow checker. **The signature is the contract**, and the
convention is strong enough that experienced Zig programmers read ownership
straight off a function declaration. Learn the four shapes and you can answer
"who frees this?" for any function in the standard library.

## The four shapes

```zig
const std = @import("std");

fn borrows(s: []const u8) usize {
    return s.len;
}

fn returnsOwned(gpa: std.mem.Allocator, s: []const u8) ![]u8 {
    return std.ascii.allocUpperString(gpa, s);
}

fn writesIntoYours(out: []u8, s: []const u8) []const u8 {
    @memcpy(out[0..s.len], s);
    return out[0..s.len];
}

fn takesOwnership(gpa: std.mem.Allocator, owned: []u8) void {
    std.debug.print("  consumed \"{s}\", freeing it\n", .{owned});
    gpa.free(owned);
}

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer {
        const status = gpa_s.deinit();
        std.debug.print("deinit status: {t}\n", .{status});
    }
    const gpa = gpa_s.allocator();

    std.debug.print("borrows            : {d}\n", .{borrows("hello")});

    const up = try returnsOwned(gpa, "hello");
    defer gpa.free(up);
    std.debug.print("returnsOwned       : \"{s}\" (caller frees)\n", .{up});

    var buf: [16]u8 = undefined;
    std.debug.print("writesIntoYours    : \"{s}\" (nobody frees)\n", .{writesIntoYours(&buf, "hello")});

    const gift = try gpa.dupe(u8, "transferred");
    std.debug.print("takesOwnership     :\n", .{});
    takesOwnership(gpa, gift);
}
```

```
borrows            : 5
returnsOwned       : "HELLO" (caller frees)
writesIntoYours    : "hello" (nobody frees)
takesOwnership     :
  consumed "transferred", freeing it
deinit status: ok
```

| Shape | Reads as | Who frees |
|---|---|---|
| `fn f(s: []const u8)` | **borrowed** — valid only during the call | nobody; caller keeps it |
| `fn f(gpa: Allocator) ![]u8` | **returns owned** memory | the **caller**, with the same allocator |
| `fn f(out: []u8) []const u8` | writes into the caller's buffer | nobody |
| `fn f(gpa: Allocator, owned: []u8)` | **takes ownership** | the **callee** |

The reliable tells:

- **An `Allocator` parameter plus an allocating return type** = the caller owns
  the result. `std.fmt.allocPrint`, `gpa.dupe`, `std.mem.join`, every
  `*Alloc`-suffixed stdlib function.
- **`[]const u8` in, no allocator** = borrowed. If you want to keep it past the
  call, `dupe` it.
- **`out: []u8`** = the caller supplies storage. `std.fmt.bufPrint`.
- **Ownership transfer is the ambiguous one** — `fn f(gpa, owned: []u8)` looks
  the same as borrowing. This is the one case where you must document it:

```zig
/// Takes ownership of `owned`; frees it with `gpa`.
fn takesOwnership(gpa: std.mem.Allocator, owned: []u8) void
```

The stdlib's naming conventions for this: `toOwnedSlice`, `fromOwnedSlice`,
`*Alloc`, `*Owned`. Follow them and you rarely need the doc comment.

**The iron rule: free with the same allocator that allocated.** Which is why
every `deinit` takes the allocator back as a parameter in 0.17's unmanaged
containers.

## `toOwnedSlice` — handing a container's buffer away

```zig
const std = @import("std");

fn collectEven(gpa: std.mem.Allocator, limit: u32) ![]u32 {
    var list: std.ArrayList(u32) = .empty;
    errdefer list.deinit(gpa);

    var i: u32 = 0;
    while (i < limit) : (i += 1) {
        if (i % 2 == 0) try list.append(gpa, i);
    }
    return list.toOwnedSlice(gpa);
}

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer {
        const status = gpa_s.deinit();
        std.debug.print("deinit status: {t}\n", .{status});
    }
    const gpa = gpa_s.allocator();

    const evens = try collectEven(gpa, 10);
    defer gpa.free(evens);
    std.debug.print("evens = {any}\n", .{evens});
}
```

```
evens = { 0, 2, 4, 6, 8 }
deinit status: ok
```

Three things are happening, and all three matter:

1. **`toOwnedSlice` transfers the buffer out of the list** and leaves the list
   empty. There is no copy and no second allocation.
2. **After `toOwnedSlice`, you must not `deinit` the list** — it no longer owns
   anything. That's why cleanup is `errdefer`, not `defer`.
3. The caller frees with `gpa.free(evens)`, not `list.deinit`.

Swap that `errdefer` for a `defer` and you free the buffer you just returned —
a use-after-free in the caller. The `errdefer`/`toOwnedSlice` pair is the
idiomatic "build a collection, return a slice" function, and it's a common
interview question.

## `defer` vs `errdefer`

```zig
const std = @import("std");

fn trace(fail: bool) !void {
    std.debug.print("  enter (fail={})\n", .{fail});
    defer std.debug.print("  defer A\n", .{});
    errdefer std.debug.print("  errdefer B\n", .{});
    defer std.debug.print("  defer C\n", .{});
    errdefer std.debug.print("  errdefer D\n", .{});
    if (fail) return error.Boom;
    std.debug.print("  success path\n", .{});
}

pub fn main() !void {
    std.debug.print("success:\n", .{});
    try trace(false);
    std.debug.print("failure:\n", .{});
    trace(true) catch |e| std.debug.print("  caught {t}\n", .{e});
}
```

```
success:
  enter (fail=false)
  success path
  defer C
  defer A
failure:
  enter (fail=true)
  errdefer D
  defer C
  errdefer B
  defer A
  caught Boom
```

**Reverse declaration order, both kinds interleaved.** `errdefer` runs *only* on
the error path, `defer` always.

| | Runs on success | Runs on error | Use for |
|---|---|---|---|
| `defer` | yes | yes | memory the *function* owns for its own duration |
| `errdefer` | no | yes | memory the function is **about to hand to the caller** |

The decision is purely "who owns this when the function returns successfully?"

| If, on success, the memory is… | Use |
|---|---|
| freed here (scratch, temporaries) | `defer` |
| returned to the caller | `errdefer` |
| stored in a struct the caller owns | `errdefer` |

## Multi-step construction: the real reason `errdefer` exists

A constructor that allocates twice and can fail *after* the first allocation:

```zig
const std = @import("std");

const User = struct {
    name: []u8,
    email: []u8,

    fn init(gpa: std.mem.Allocator, name: []const u8, email: []const u8) !User {
        const n = try gpa.dupe(u8, name);
        errdefer gpa.free(n);
        const e = try gpa.dupe(u8, email);
        errdefer gpa.free(e);
        if (std.mem.indexOfScalar(u8, email, '@') == null) return error.InvalidEmail;
        return .{ .name = n, .email = e };
    }

    fn deinit(u: *User, gpa: std.mem.Allocator) void {
        gpa.free(u.name);
        gpa.free(u.email);
    }
};

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer {
        const status = gpa_s.deinit();
        std.debug.print("deinit status: {t}\n", .{status});
    }
    const gpa = gpa_s.allocator();

    var ok = try User.init(gpa, "raj", "raj@example.com");
    defer ok.deinit(gpa);
    std.debug.print("ok   : {s} <{s}>\n", .{ ok.name, ok.email });

    const bad = User.init(gpa, "nobody", "not-an-email");
    std.debug.print("bad  : {!}\n", .{bad});
}
```

```
ok   : raj <raj@example.com>
bad  : error.InvalidEmail
deinit status: ok
```

**`deinit status: ok`** — the successful `User` was freed by its `defer`, and the
failed one freed *both* of its duped strings via the two `errdefer`s before
returning the error. Each `errdefer` covers exactly the allocation above it.

### The same code without `errdefer`

```zig
const std = @import("std");

const User = struct {
    name: []u8,
    email: []u8,

    fn init(gpa: std.mem.Allocator, name: []const u8, email: []const u8) !User {
        const n = try gpa.dupe(u8, name);
        const e = try gpa.dupe(u8, email);
        if (std.mem.indexOfScalar(u8, email, '@') == null) return error.InvalidEmail;
        return .{ .name = n, .email = e };
    }
};

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer {
        const status = gpa_s.deinit();
        std.debug.print("deinit status: {t}\n", .{status});
    }
    const gpa = gpa_s.allocator();

    const bad = User.init(gpa, "nobody", "not-an-email");
    std.debug.print("bad  : {!}\n", .{bad});
}
```

**Deliberate leak** — two `errdefer` lines deleted, two leaks:

```
bad  : error.InvalidEmail
error(DebugAllocator): memory address 0x104580000 leaked: 
...
/path/f4.zig:8:31: 0x104436baf in init (f4)
        const n = try gpa.dupe(u8, name);
                              ^
/path/f4.zig:23:26: 0x1044335fb in main (f4)
    const bad = User.init(gpa, "nobody", "not-an-email");
                         ^
...
error(DebugAllocator): memory address 0x1045a0000 leaked: 
...
/path/f4.zig:9:31: 0x104436c07 in init (f4)
        const e = try gpa.dupe(u8, email);
                              ^
```

Two reports, naming line 8 and line 9 — the exact two `dupe` calls whose
cleanup is missing. **Every early `return` past an allocation needs an
`errdefer`.** This is the most common real-world leak in Zig code, and the
DebugAllocator points straight at it.

## Containers own their contents

If a struct holds allocated memory, its `deinit` must free **both** the
container and each element:

```zig
const std = @import("std");

const Registry = struct {
    gpa: std.mem.Allocator,
    names: std.ArrayList([]u8),

    fn init(gpa: std.mem.Allocator) Registry {
        return .{ .gpa = gpa, .names = .empty };
    }

    fn deinit(r: *Registry) void {
        for (r.names.items) |n| r.gpa.free(n);
        r.names.deinit(r.gpa);
    }

    fn add(r: *Registry, name: []const u8) !void {
        const owned = try r.gpa.dupe(u8, name);
        errdefer r.gpa.free(owned);
        try r.names.append(r.gpa, owned);
    }
};

pub fn main() !void {
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer {
        const status = gpa_s.deinit();
        std.debug.print("deinit status: {t}\n", .{status});
    }
    const gpa = gpa_s.allocator();

    var reg = Registry.init(gpa);
    defer reg.deinit();

    var stack_name: [8]u8 = undefined;
    @memcpy(&stack_name, "ephemer!");

    try reg.add("alpha");
    try reg.add(&stack_name);
    @memset(&stack_name, 'X');

    for (reg.names.items) |n| std.debug.print("registered: {s}\n", .{n});
}
```

```
registered: alpha
registered: ephemer!
deinit status: ok
```

Two separate lessons in that output.

**`"ephemer!"` survived being overwritten with `X`s.** `add` took a borrowed
`[]const u8` and `dupe`d it, so the registry owns a copy. Had it stored the
slice directly, the loop would print `XXXXXXXX` — a dangling slice into a stack
buffer, exactly the bug from lesson 5.1. **A borrowed slice stored in a
longer-lived struct must be duped.**

**`deinit` frees the elements first, then the list.** Freeing `r.names` first
would lose the pointers. `ArrayList.deinit` releases only its own backing array;
it has no idea its elements are owned pointers. The same applies to `HashMap`:
if you own the keys or the values, free them in a loop before `deinit`.

### Store the allocator, or pass it?

| Approach | Example | Trade-off |
|---|---|---|
| store it in the struct | `Registry` above: `deinit(r)` | can't use the wrong allocator; +16 bytes per instance |
| pass it per call | 0.17's `ArrayList`: `deinit(gpa)` | smaller struct; caller must remember which allocator |

0.17 moved the stdlib containers to the second style — that's what "unmanaged"
means, and it's why `list.append(gpa, x)` takes the allocator every time. For
your own application types, storing it is usually friendlier.

## Ownership checklist

Before you commit an allocating function, answer these:

| Question | If unclear |
|---|---|
| Does it return allocated memory? | say so in a doc comment and name it `…Alloc` or `toOwned…` |
| Does every error path free what it allocated? | add `errdefer` after each allocation |
| Does it store a borrowed slice anywhere? | `dupe` it |
| Does the `deinit` free elements *and* the container? | loop first, then `deinit` |
| Is the result freed with the same allocator? | take the allocator as a parameter, don't guess |

---

Next: [5.7 — Write an Allocator](./07-write-your-own.md)
