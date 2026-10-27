# 4.3 — `defer` & `errdefer`

| Keyword | Runs when scope exits | On success | On error return |
|---|---|---|---|
| `defer` | always | yes | yes |
| `errdefer` | only if the scope exits by returning an error | **no** | yes |

That second row is the whole lesson. Zig has no destructors, no `finally`, no
`with`, no RAII — cleanup is a statement you write next to the thing being
acquired.

## `defer` is LIFO

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("start\n", .{});
    defer std.debug.print("defer 1\n", .{});
    defer std.debug.print("defer 2\n", .{});
    defer std.debug.print("defer 3\n", .{});
    std.debug.print("end of body\n", .{});
}
```

```
start
end of body
defer 3
defer 2
defer 1
```

Reverse registration order — which is exactly what you want, because resource #3
was probably built out of #2 and #1.

## Scope, not function

```zig
const std = @import("std");

pub fn main() void {
    for (0..3) |i| {
        defer std.debug.print("  leave {d}\n", .{i});
        std.debug.print("enter {d}\n", .{i});
    }
    {
        defer std.debug.print("inner block done\n", .{});
        std.debug.print("in inner block\n", .{});
    }
    std.debug.print("after\n", .{});
}
```

```
enter 0
  leave 0
enter 1
  leave 1
enter 2
  leave 2
in inner block
inner block done
after
```

A `defer` inside a loop body fires **every iteration**, not at the end of the
function. Go programmers: this is the opposite of Go's function-scoped `defer`,
and it is the behaviour you actually wanted.

## `errdefer` fires only on error

```zig
const std = @import("std");

fn run(fail: bool) !void {
    std.debug.print("  open\n", .{});
    defer std.debug.print("  defer: always\n", .{});
    errdefer std.debug.print("  errdefer: only on error\n", .{});

    if (fail) return error.Boom;
    std.debug.print("  work done\n", .{});
}

pub fn main() void {
    std.debug.print("run(false):\n", .{});
    run(false) catch {};
    std.debug.print("run(true):\n", .{});
    run(true) catch |e| std.debug.print("  caught {t}\n", .{e});
}
```

```
run(false):
  open
  work done
  defer: always
run(true):
  open
  errdefer: only on error
  defer: always
  caught Boom
```

On the error path both fire, still LIFO (the `errdefer` was registered later, so
it runs first). On the success path only the `defer` does.

## The pattern this exists for

Half-built objects. You've allocated three things, the fourth step fails, and
you must unwind exactly the first three — no more, no less.

```zig
const std = @import("std");

const User = struct {
    name: []u8,
    email: []u8,

    fn create(gpa: std.mem.Allocator, name: []const u8, email: []const u8) !*User {
        const u = try gpa.create(User);
        errdefer gpa.destroy(u);

        u.name = try gpa.dupe(u8, name);
        errdefer gpa.free(u.name);

        if (std.mem.indexOfScalar(u8, email, '@') == null) return error.BadEmail;

        u.email = try gpa.dupe(u8, email);
        return u;
    }

    fn destroy(u: *User, gpa: std.mem.Allocator) void {
        gpa.free(u.email);
        gpa.free(u.name);
        gpa.destroy(u);
    }
};

pub fn main() void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer std.debug.print("leak check: {t}\n", .{dbg.deinit()});
    const gpa = dbg.allocator();

    if (User.create(gpa, "ada", "ada@example.com")) |u| {
        std.debug.print("ok: {s} <{s}>\n", .{ u.name, u.email });
        u.destroy(gpa);
    } else |e| std.debug.print("err {t}\n", .{e});

    if (User.create(gpa, "bob", "not-an-email")) |u| {
        std.debug.print("ok: {s}\n", .{u.name});
        u.destroy(gpa);
    } else |e| std.debug.print("err {t}\n", .{e});
}
```

```
ok: ada <ada@example.com>
err BadEmail
leak check: ok
```

**`leak check: ok` is the proof.** `DebugAllocator.deinit()` returns an enum that
is `ok` or `leak`; printing it with `{t}` makes the test visible. The `bob` call
allocated a `User` and duped `"bob"`, then failed — both were released by the two
`errdefer`s, and the allocator agrees nothing was lost.

Note the success path does **not** free anything: `return u` is a success, so the
`errdefer`s are discarded and ownership transfers to the caller. One keyword
covers both "clean up on failure" and "hand over on success".

### Without the errdefers: a real leak

Same program, two lines removed:

```zig
const std = @import("std");

const User = struct {
    name: []u8,
    email: []u8,

    fn create(gpa: std.mem.Allocator, name: []const u8, email: []const u8) !*User {
        const u = try gpa.create(User);

        u.name = try gpa.dupe(u8, name);

        if (std.mem.indexOfScalar(u8, email, '@') == null) return error.BadEmail;

        u.email = try gpa.dupe(u8, email);
        return u;
    }

    fn destroy(u: *User, gpa: std.mem.Allocator) void {
        gpa.free(u.email);
        gpa.free(u.name);
        gpa.destroy(u);
    }
};

pub fn main() void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer std.debug.print("leak check: {t}\n", .{dbg.deinit()});
    const gpa = dbg.allocator();

    if (User.create(gpa, "ada", "ada@example.com")) |u| {
        std.debug.print("ok: {s} <{s}>\n", .{ u.name, u.email });
        u.destroy(gpa);
    } else |e| std.debug.print("err {t}\n", .{e});

    if (User.create(gpa, "bob", "not-an-email")) |u| {
        std.debug.print("ok: {s}\n", .{u.name});
        u.destroy(gpa);
    } else |e| std.debug.print("err {t}\n", .{e});
}
```

**Deliberate failure.** Trimmed real output:

```
ok: ada <ada@example.com>
err BadEmail
error(DebugAllocator): memory address 0x104560000 leaked:
/opt/.../std/mem/Allocator.zig:481:40: 0x1042d10ef in dupe__func_117
    const new_buf = try allocator.alloc(T, m.len);
                                       ^
/tmp/leak.zig:11:30: 0x1043c38d7 in create (leak)
        u.name = try gpa.dupe(u8, name);
                             ^
/tmp/leak.zig:37:20: 0x1043c3bf7 in main (leak)
    if (User.create(gpa, "bob", "not-an-email")) |u| {
                   ^

error(DebugAllocator): memory address 0x104540000 leaked:
/tmp/leak.zig:8:33: 0x1043c387f in create (leak)
        const u = try gpa.create(User);
                                ^
...

leak check: leak
```

Two leaks, each with the **allocation site**, and `leak check: leak`. Exit code
is still 0 — a leak is a diagnostic, not a crash, so it is on you to look.
(Addresses and paths differ every run.)

## Transferring or wiping a whole arena

```zig
const std = @import("std");

fn transfer(gpa: std.mem.Allocator, fail_at: u8) ![]u8 {
    var arena_state: std.heap.ArenaAllocator = .init(gpa);
    errdefer arena_state.deinit();
    const arena = arena_state.allocator();

    for (0..5) |i| {
        _ = try arena.alloc(u8, 64);
        if (i == fail_at) return error.Aborted;
    }
    const out = try gpa.dupe(u8, "result");
    arena_state.deinit();
    return out;
}

pub fn main() void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer std.debug.print("leak check: {t}\n", .{dbg.deinit()});
    const gpa = dbg.allocator();

    if (transfer(gpa, 99)) |s| {
        std.debug.print("ok: {s}\n", .{s});
        gpa.free(s);
    } else |e| std.debug.print("err {t}\n", .{e});

    if (transfer(gpa, 2)) |s| {
        std.debug.print("ok: {s}\n", .{s});
        gpa.free(s);
    } else |e| std.debug.print("err {t} (arena wiped by errdefer)\n", .{e});
}
```

```
ok: result
err Aborted (arena wiped by errdefer)
leak check: ok
```

Five scratch allocations, abandoned mid-loop, all reclaimed by one `errdefer`.
This is the shape of a per-request handler in a server: scratch arena, error
wipes it, success copies out the one thing the caller keeps.

## Multiple errdefers unwind partially

Only the `errdefer`s that were **reached** fire:

```zig
const std = @import("std");

const Conn = struct {
    id: u32,
    fn close(c: Conn) void {
        std.debug.print("  close conn {d}\n", .{c.id});
    }
};

fn openConn(id: u32) !Conn {
    std.debug.print("  open conn {d}\n", .{id});
    if (id == 3) return error.Refused;
    return .{ .id = id };
}

fn openThree() !void {
    const a = try openConn(1);
    errdefer a.close();
    const b = try openConn(2);
    errdefer b.close();
    const c = try openConn(3);
    errdefer c.close();

    std.debug.print("  all three up\n", .{});
    a.close();
    b.close();
    c.close();
}

pub fn main() void {
    openThree() catch |e| std.debug.print("  failed: {t}\n", .{e});
}
```

```
  open conn 1
  open conn 2
  open conn 3
  close conn 2
  close conn 1
  failed: Refused
```

Conn 3 never opened, so its `errdefer` was never registered — 2 and 1 close, in
reverse. There is no bookkeeping variable, no `if (c != null)`: the control flow
*is* the bookkeeping.

## `errdefer` is scoped too

```zig
const std = @import("std");

fn scoped(fail: bool) !void {
    {
        errdefer std.debug.print("  inner errdefer\n", .{});
        std.debug.print("  inner block ok\n", .{});
    }
    errdefer std.debug.print("  outer errdefer\n", .{});
    if (fail) return error.Late;
}

pub fn main() void {
    std.debug.print("scoped(true):\n", .{});
    scoped(true) catch {};
}
```

```
scoped(true):
  inner block ok
  outer errdefer
```

The inner `errdefer` expired when its block ended successfully. Registering
cleanup in a narrow block that then succeeds means the cleanup is gone — put the
`errdefer` in the scope that owns the resource.

> **`errdefer |e| { ... }` does not exist in 0.17.0.** Tutorials and older Zig
> show capturing the error value:
>
> ```zig
> const std = @import("std");
>
> fn step(n: u8) !void {
>     errdefer |e| {
>         std.debug.print("rolling back after {t}\n", .{e});
>     }
>     if (n == 1) return error.DiskFull;
> }
>
> pub fn main() void {
>     step(1) catch {};
> }
> ```
>
> **Deliberate failure — exit code 1:**
>
> ```
> capture.zig:4:14: error: expected block or expression, found '|'
>     errdefer |e| {
>              ^
> ```
>
> `errdefer` takes no capture here. If the cleanup needs to know *which* error
> occurred, use `catch |e| switch (e)` at the call site, or an `if (x) |v| else |e|`.

## Why this replaces `finally` and RAII

| Language | Mechanism | Cost |
|---|---|---|
| Java / Python | `try/finally`, `with` | cleanup is far from acquisition; nesting grows |
| C++ / Rust | destructors / `Drop` | implicit; a hidden call at every `}`; needs move semantics to transfer ownership |
| Go | `defer` | function-scoped only; no error-only variant |
| Zig | `defer` / `errdefer` | **one line, right under the acquisition, visible in the source** |

The tradeoff is honest: Zig will not clean up for you. Forget the line and you
leak — which is why `DebugAllocator` reporting leaks with allocation sites is
part of the design, not an add-on.

Rules of thumb:

- `defer` immediately after you acquire something you will always release.
- `errdefer` immediately after you acquire something you will **hand to the
  caller on success**.
- `defer x.deinit()` right under `var x = ...init` — same line pair, every time.
- Put `defer out.flush() catch {}` under every writer.

---

Next: [4.4 — Error Traces](./04-error-traces.md)
