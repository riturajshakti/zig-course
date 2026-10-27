# Module 02 — Language Basics

Six short lessons covering ordinary procedural Zig. Nothing here needs an
allocator or an `io` parameter — just `std.debug.print` and the language itself.

> **What will surprise you**, coming from Node.js, Go, Python or Rust:
>
> | You expect | Zig 0.17.0 |
> |---|---|
> | `var` you never reassign is fine | **compile error** — "consider using 'const'" |
> | an unused variable is a warning | **compile error** |
> | an inner scope may reuse a name | **shadowing is a compile error** |
> | `if (n)`, `if (list.length)` | **compile error** — conditions must be `bool` |
> | `250 + 10` on a `u8` wraps to `4` | **panics** in Debug; wraps only in `ReleaseFast` |
> | `-7 / 2` picks a rounding for you | **compile error** — say `@divTrunc` or `@divFloor` |
> | `switch` has a `default` | exhaustive by default; a new enum variant **breaks the build** |
> | `defer` runs at function exit | runs at **block** exit — including every loop iteration |
> | narrowing an int silently truncates | **compile error** — use `@intCast` or `@truncate` |

## Lessons

| # | Lesson | What you get |
|---|---|---|
| 2.1 | [Variables](./01-variables.md) | `const` vs `var`, inference and `comptime_int`, the real danger of `undefined`, no shadowing, `comptime var` |
| 2.2 | [Numbers](./02-numbers.md) | `u8`/`i32`/`u64`, arbitrary widths like `u3` and `i7`, `usize`, floats, `maxInt`/`minInt`, every literal form |
| 2.3 | [Overflow & Casting](./03-overflow-casting.md) | **overflow panics** (and wraps in release), `+%` `+\|`, `@intCast` vs `@truncate`, `@bitCast`, `@divFloor` vs `@divTrunc` |
| 2.4 | [Control Flow](./04-control-flow.md) | **no truthiness**, `if` as an expression, all four `for` forms, `break` with a value, `inline for` |
| 2.5 | [Switch](./05-switch.md) | enforced exhaustiveness, no fallthrough, ranges `1...5`, capturing with `\|x\|`, `unreachable` |
| 2.6 | [Blocks & Labels](./06-blocks-labels.md) | `blk: { break :blk v; }`, labeled `break`/`continue` through nested loops, `for ... else`, `defer` is LIFO |

Every program is complete. Copy a block into a `.zig` file and `zig run` it
as-is. Several blocks are **deliberate failures** — compile errors and panics
used as teaching devices — and each is labelled where it appears.

---

Next: [Module 03 — Types, Structs & Unions](../03-types-structs-unions/)
