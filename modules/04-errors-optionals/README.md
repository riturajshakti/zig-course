# Module 04 — Errors & Optionals

Zig has **no exceptions** and **no null pointers**. Failure is a value with a
type, absence is a different type, and cleanup is a line you write next to the
thing being acquired. The compiler will not let you ignore any of it.

If you are coming from somewhere else, the mapping is:

| Elsewhere | Zig |
|---|---|
| `throw` / `raise` | `return error.Name` — an ordinary return |
| `try { } catch (e) { }` | `expr catch \|e\| { }` — an expression, not a block |
| `if err != nil { return err }` (Go) | `try expr` |
| `Result<T, E>` (Rust) | `E!T`, with no `Ok(...)`/`Err(...)` wrapping |
| `Option<T>` / `null` / `nil` | `?T` — and a plain `T` can **never** be null |
| `finally` / `with` / destructors | `defer`, plus `errdefer` for failure-only cleanup |
| stack trace on an exception | **error return trace** — the path the error travelled |

## Lessons

| # | Lesson | What you get |
|---|---|---|
| 4.1 | [Error Sets](./01-error-sets.md) | `error{A,B}` as a type, `!T`, merging with `\|\|`, inferred sets, `anyerror`, `@errorName` |
| 4.2 | [`try` & `catch`](./02-try-catch.md) | `try` as shorthand, `catch` defaults, `catch \|e\| switch (e)`, `if (x) \|v\| else \|e\|`, what an escaping error prints |
| 4.3 | [`defer` & `errdefer`](./03-defer-errdefer.md) | LIFO cleanup, errdefer fires **only on error**, the alloc/errdefer-free pattern, a real leak report |
| 4.4 | [Error Traces](./04-error-traces.md) | return traces vs stack traces, `{!d}`/`{!s}`/`{t}`, a real panic, release-mode differences |
| 4.5 | [Optionals](./05-optionals.md) | `?T`, `orelse`, `.?`, `if \|v\|`, `while (it.next()) \|v\|`, why `?*T` is free, nested `??` |
| 4.6 | [Error vs Optional](./06-error-vs-optional.md) | `!T` vs `?T` vs `!?T`, `unreachable` per release mode, `assert`, API design for a service |

Every program is complete. Copy a block into a `.zig` file and `zig run` it.
Several are **deliberate failures** — compile errors, panics and leak reports
with their real output and exit codes, clearly labelled as such.

## The three ideas worth slowing down for

1. **Errors are values in the type system.** `FileErr!u32` is a single value that
   holds a `u32` or one of two errors. Widening an error set is implicit,
   narrowing is a compile error, and a `switch` over a closed set is checked for
   exhaustiveness — so adding a failure mode breaks the build until someone
   handles it.
2. **`errdefer` replaces destructors.** It runs only when the scope exits with an
   error, which makes "free what I allocated if step 4 fails, but hand it to the
   caller if it succeeds" a single line with no bookkeeping variable.
3. **`?T` exists so that `T` doesn't need to be nullable.** A `*User` is a
   guaranteed-valid pointer. `?*User` costs no extra bytes (null is address 0)
   and cannot be dereferenced without unwrapping.

---

Next: [Module 05 — Memory & Allocators](../05-memory-allocators/)
