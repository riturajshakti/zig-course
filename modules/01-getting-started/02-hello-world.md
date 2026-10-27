# 1.2 — Hello World

## The shortest real program

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("Hello, world!\n", .{});
}
```

```sh
zig run hello.zig
```

```
Hello, world!
```

Three things to notice:

- `main` must be `pub`.
- The second argument to `print` is **always** a tuple, even when empty: `.{}`.
  Zig has no varargs; the format arguments are one anonymous struct.
- `std.debug.print` needs no allocator, no `io`, and no flush. It is the only
  printing API in 0.17 that is this easy — which is why these first lessons use it.

## `std.debug.print` writes to stderr, not stdout

This surprises people, so prove it:

```sh
zig run hello.zig 1>/dev/null     # stdout discarded
```

```
Hello, world!
```

```sh
zig run hello.zig 2>/dev/null     # stderr discarded
```

```
```

Nothing. The name is honest — it is a *debug* printer. Lesson 1.3 covers real
stdout. For logs, progress and quick experiments, stderr is the right place
anyway.

## `void` vs `!void`

Both of these compile and print the same thing:

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("Hello, world!\n", .{});
}
```

```zig
const std = @import("std");

pub fn main() !void {
    std.debug.print("Hello, world!\n", .{});
}
```

```
Hello, world!
```

So what is the `!` for?

### `!void` is an error union type

`!void` means *"either nothing, or an error"*. The `!` with nothing on its left
is an **inferred error set**: the compiler collects every error your function
body can actually return and builds the set for you. Written out by hand the
same type would be `error{Negative, OutOfMemory, ...}!void`.

This is Zig's whole error story and it is worth internalizing now:

- Errors are **values with types**, not exceptions. There is no unwinding, no
  `throw`, no stack-based catch.
- A function's signature tells you whether it can fail. `fn f() void` *cannot*
  fail — not "probably won't", cannot.
- You cannot ignore a possible error. The compiler forces you to either
  propagate it with `try` or handle it with `catch`.

Closest analogues from languages you know: Go's `(T, error)` return pair, or
Rust's `Result<T, E>`. Unlike Go you cannot silently drop it, and unlike Rust
the error set is usually inferred instead of named.

### `try` propagates

```zig
const std = @import("std");

fn risky(n: i32) !i32 {
    if (n < 0) return error.Negative;
    return n * 2;
}

pub fn main() !void {
    const ok = try risky(21);
    std.debug.print("ok = {d}\n", .{ok});
    const bad = try risky(-1);
    std.debug.print("never printed: {d}\n", .{bad});
}
```

```
ok = 42
error: Negative
/path/to/risky.zig:4:16: 0x1030c75cf in risky (risky)
    if (n < 0) return error.Negative;
               ^
/path/to/risky.zig:11:17: 0x1030c77c7 in main (risky)
    const bad = try risky(-1);
                ^
```

Exit code `1`. `error.Negative` escaped `main`, so Zig's default root handler
printed it along with an **error return trace** — the chain of `return`/`try`
sites the error travelled through, not a stack trace. (Your paths and hex
addresses will differ from the above; addresses are randomized per run.)

`try risky(-1)` is exactly sugar for:

```
risky(-1) catch |err| return err;
```

That is the *only* reason `main` needs `!void`: so a `try` inside it has
somewhere to return to. A `main` with no `try` and no `return error.X` does not
need the `!`.

### `catch` handles

```zig
const std = @import("std");

fn risky(n: i32) !i32 {
    if (n < 0) return error.Negative;
    return n * 2;
}

pub fn main() void {
    const v = risky(-1) catch |err| {
        std.debug.print("handled: {t}\n", .{err});
        return;
    };
    std.debug.print("v = {d}\n", .{v});
}
```

```
handled: Negative
```

Exit code `0`. Note `main` is back to plain `void` — the error never escapes, so
the `!` is not needed. `{t}` prints an error's name.

## The four `main` signatures

```zig
const std = @import("std");

fn risky(n: i32) !i32 {
    if (n < 0) return error.Negative;
    return n * 2;
}

pub fn main() u8 {
    const v = risky(-1) catch {
        std.debug.print("failing with code 3\n", .{});
        return 3;
    };
    std.debug.print("v = {d}\n", .{v});
    return 0;
}
```

```
failing with code 3
```

Exit code `3`.

| Signature | Use when |
|---|---|
| `pub fn main() void` | no `try`, no explicit exit code |
| `pub fn main() !void` | you use `try`; an escaping error prints a trace and exits `1` |
| `pub fn main() u8` | you want to control the process exit code |
| `pub fn main(init: std.process.Init) !void` | **you need I/O, allocation or args** — lesson 1.4 |

That last one is the 0.17 workhorse and the one most tutorials have never heard
of. You need it the moment you want to print to real stdout, which is next.

---

Next: [1.3 — Printing Properly](./03-printing.md)
