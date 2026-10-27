# Module 03 — Types, Structs & Unions

Seven lessons on modelling data in Zig. Three of them — slices, strings and
pointers — are the ones that take real work if you are coming from a language
with a garbage collector, because Zig makes **who owns this memory** part of the
type.

Lessons 3.2 and 3.3 are built as **reference pages**: each ends with a dense
table of every operation on the type, so you can come back and look things up.

## Lessons

| # | Lesson | What you get |
|---|---|---|
| 3.1 | [Arrays](./01-arrays.md) | `[N]T` where the length *is* the type, `++`, `@splat`, `@memset`/`@memcpy`, 2D, comparing |
| 3.2 | [Slices](./02-slices.md) | `[]T` vs `[]const T`, sentinel slices, ~45 `std.mem` operations, and **the dangling-slice trap** |
| 3.3 | [Strings](./03-strings.md) | there is **no `String` type** — search, split, join, replace, case, parse, and UTF-8 bytes vs codepoints |
| 3.4 | [Structs](./04-structs.md) | defaults, methods, `Self`, anonymous structs, tuples, namespaces, and **why Zig reorders your fields** |
| 3.5 | [Pointers](./05-pointers.md) | `*T`, **no null**, `?*T` for free, `[*]T` vs `[]T`, `@ptrCast` |
| 3.6 | [Enums](./06-enums.md) | backing types, methods, switch exhaustiveness, conversions, non-exhaustive `_` |
| 3.7 | [Tagged Unions](./07-unions.md) | `union(enum)` as a real sum type, payload capture, and how it replaces Rust enums / TS discriminated unions |

## The four things worth slowing down for

1. **Arrays are values, slices are borrows.** Copying an array copies every
   element; copying a slice copies 16 bytes and shares the data. (3.1, 3.2)
2. **The dangling-slice trap.** Returning a slice into a local compiles silently
   and corrupts memory. Lesson 3.2 shows two "independent" strings landing on the
   same stack slot. There is no borrow checker to stop you.
3. **`?*T` is the same 8 bytes as `*T`.** Zig has no null pointer; optional
   pointers cost nothing and cannot be dereferenced unchecked. (3.5)
4. **`union(enum)`, never bare `union`.** Lesson 3.7 runs the same program in
   Debug and ReleaseFast: a panic in one, silently wrong bytes in the other. (3.7)

## Things 0.17.0 changed

Tutorials written for 0.13–0.15 will hand you these:

| What you'll find online | Zig 0.17.0 |
|---|---|
| `[_]u8{'-'} ** 8` | **`**` was removed** — `const d: [8]u8 = @splat('-')` |
| `std.mem.trimLeft` / `trimRight` | `std.mem.trimStart` / `trimEnd` |
| `gpa.dupeZ(u8, s)` | gone — `gpa.allocSentinel(u8, s.len, 0)` + `@memcpy` |
| `EnumSet(E).initEmpty()` | the constant `.empty` (and `.full`) |
| `pub fn format(self, fmt, opts, writer)` | `pub fn format(self: T, w: *std.Io.Writer) !void` |
| `std.mem.indexOf` | still works, now an alias of `std.mem.find` |

Every program in these lessons is complete. Copy a block into a `.zig` file and
`zig run` it as-is — including the ones labelled deliberate, which are there so
you recognize the error message when you cause it yourself.

---

Next: [Module 04 — Errors & Optionals](../04-errors-optionals/)
