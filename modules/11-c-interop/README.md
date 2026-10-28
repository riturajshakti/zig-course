# Module 11 — C Interop

Wrapping C libraries is a large share of real Zig work. Most of what a backend
or systems job needs already exists as a C library — zlib, SQLite, libcurl,
OpenSSL, a vendor SDK — and the job is to put a safe Zig surface on it. The
other direction pays too: an `export fn` module dropped into an existing C
codebase is how Zig gets adopted one file at a time.

Zig is unusually good at this. It ships a C compiler, cross-compiles C, and
needs no `Makefile`, no FFI runtime, and no binding generator.

## ⚠️ Read this before anything else

**`@cImport` and `@cInclude` were removed.** Every C-interop tutorial you will
find online opens with them, and in 0.17.0 they produce
`error: invalid builtin function: '@cImport'` before the compiler even looks at
the rest. Lesson 11.1 shows the error and the migration table. The replacement
is `extern fn` declarations you write yourself, `std.c` for libc, and
`zig translate-c` run as an explicit one-time step.

## Lessons

| # | Lesson | What you get |
|---|---|---|
| 11.1 | [Calling C](./01-calling-c.md) | **`@cImport` is gone** — the real error and migration path, `extern fn` + `-lc`, `std.c`'s 300 libc declarations, variadics, `qsort` callbacks, `zig translate-c` output read line by line, `zig cc` |
| 11.2 | [C Types](./02-c-types.md) | the `c_*` types measured here (`@bitSizeOf(c_int)` = 32), **Zig reorders struct fields** and `extern struct` doesn't, `@sizeOf`/`@offsetOf` checked from both sides, enums, `packed struct` bitflags, `[*c]T` vs `[*]T` vs `[]T` vs `?*T`, `opaque {}`, `@ptrCast`/`@alignCast` |
| 11.3 | [C Strings](./03-c-strings.md) | `[*:0]const u8` vs `[]const u8`, `std.mem.span` / `sliceTo`, **`dupeZ` is gone** → `allocSentinel`, `bufPrintSentinel`, who frees a returned string, and **embedded NULs truncating silently** |
| 11.4 | [Linking Libraries](./04-linking.md) | `.c` files alongside Zig in one command, `-lc`/`-lm`, the **0.17 `createModule` + `root_module`** form, `addCSourceFiles`, `addIncludePath`, `-D` defines, `linkSystemLibrary` against real zlib, static vs dynamic, static-musl cross-compiles |
| 11.5 | [Exporting to C](./05-exporting.md) | `export fn` vs `callconv(.c)`, a C `main` calling Zig via `zig cc`, **`!T` and `?T` cannot cross** — map to codes and out-params, who allocates, hand-written headers, and ABI breakage shown with offsets |

## The six things worth slowing down for

1. **`@cImport` is removed.** Not deprecated — the builtin does not exist. This
   invalidates essentially every C-interop tutorial written before 0.17. (11.1)
2. **Your `extern fn` declaration is unchecked.** Nothing compares it to the
   real C header. Get the width wrong and it links fine, then reads garbage.
   This is the one real cost of losing `@cImport`, and why 11.2 is a table of
   measured sizes rather than remembered ones. (11.1, 11.2)
3. **Zig reorders struct fields.** `struct { a: u8, b: u32, c: u8 }` puts `b` at
   offset **0**. `extern struct` is mandatory at any C boundary, and passing a
   plain struct by value is at least a compile error. By pointer, it is not. (11.2)
4. **`c_long` is 64 bits here and 32 on Windows.** Writing `i64` for a C `long`
   is the most common cross-platform interop bug. (11.2, and live in 11.4's
   zlib wrapper)
5. **A Zig slice can hold a NUL; a C string cannot.** Converting one truncates
   **silently** — three bytes in, one byte out, no error. One `indexOfScalar`
   turns it into a handled error. (11.3)
6. **Never free C's memory with a Zig allocator, or vice versa.** Different
   heaps. If you hand allocated memory to C, you must also export the function
   that frees it. (11.3, 11.5)

## Things 0.17.0 changed

| What you'll find online | Zig 0.17.0 |
|---|---|
| `@cImport({ @cInclude("x.h") })` | **removed** — `extern fn` by hand, or `zig translate-c` |
| `@cInclude`, `@cDefine`, `@cUndef` | removed with it; `-D` goes in build flags |
| `gpa.dupeZ(u8, s)` | `allocSentinel(u8, s.len, 0)` + `@memcpy` |
| `std.fmt.allocPrintZ` | `std.fmt.allocPrintSentinel(gpa, fmt, args, 0)` |
| `std.fmt.bufPrintZ` | `std.fmt.bufPrintSentinel(&buf, fmt, args, 0)` |
| `callconv(.C)` | `callconv(.c)` — lowercase |
| `b.addExecutable(.{ .root_source_file = …, .target = … })` | `b.createModule(.{…})` then `.root_module = mod` |
| `exe.linkLibC()` | `.link_libc = true` on the module |
| `exe.addCSourceFile(…)` / `exe.addIncludePath(…)` | `mod.addCSourceFiles(…)` / `mod.addIncludePath(…)` |
| `exe.defineCMacro("X","1")` | `"-DX=1"` in `.flags` |
| `@typeInfo(F).@"fn".calling_convention` | `@typeInfo(F).@"fn".attrs.@"callconv"` |
| `@intFromEnum` / `@enumFromInt` | still compile, but `zig fmt` rewrites to `@backingInt` / `@intCast` |

## A note on build times

The **first** build that links libc takes well over two minutes — Zig is
compiling its libc shim from source. Everything after that is cached and takes
under a second. Don't assume something has hung.

Every program here is complete and was run on Zig 0.17.0 before being pasted,
including the ones that fail on purpose — those are labelled, and their error
output is real, so you recognize the message when you cause it yourself.
Multi-file examples show every file in full, with the exact command.

---

Next: [Module 12 — Concurrency & Async](../12-concurrency-async/)
