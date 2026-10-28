# Module 09 — Testing & Debugging

`test` is a keyword, not a library. Tests live in the same file as the code,
they see private declarations, and they are not even *analysed* in a normal
build. There is nothing to install and nothing to configure.

Three things here are genuinely different from Node, Go and Python:

1. **`std.testing.allocator` fails a test on a memory leak** — and reports it
   in the strangest way in the language: it prints `All N tests passed.` and
   then exits 1. Lesson 9.3 is the most important page in this module.
2. **Release modes change behavior, not just speed.** The same source panics on
   an integer overflow in `-Odebug`/`-Osafe` and silently wraps in
   `-Ofast`/`-Osmall`. Leak detection quietly stops working too.
3. **Every assertion returns an error union.** A test fails by returning an
   error, so `errdefer` is your diagnostic hook and `try` goes in front of
   every matcher.

> **0.17.0 changes.** The optimize modes are lowercase: `-Odebug`, `-Osafe`,
> `-Ofast`, `-Osmall`, and `builtin.mode` prints `debug`/`safe`/`fast`/`small`.
> `std.testing.allocator` is a `SafeAllocator`, not a `DebugAllocator`, so its
> leak report format differs. `std.builtin` is now `std.lang`, which surfaces in
> stack-trace type errors. `errdefer |e|` does not exist.

## Lessons

| # | Lesson | What you get |
|---|---|---|
| 9.1 | [Writing Tests](./01-writing-tests.md) | `test "name" { }`, `zig test`, why `zig run` on a test file gives a `HWCAP` error, tests seeing private fns, proof that test bodies aren't compiled in a normal build, `--test-filter`, `test { _ = @import(..) }`, `refAllDecls`, `zig build test` |
| 9.2 | [Assertions](./02-assertions.md) | all ten matchers passing, then **what each one prints when it fails**, `expectEqual` on slices comparing pointers, `expectEqualDeep` naming the bad field, `error.SkipZigTest`, `errdefer` for failure context |
| 9.3 | [testing.allocator](./03-testing-allocator.md) | **a leak fails the test** — the real report, "All 2 tests passed" with exit code 1, double-free with three stack traces, `failing_allocator`, `checkAllAllocationFailures` finding a leak on an OOM path, arenas in tests |
| 9.4 | [Debugging](./04-debugging.md) | `std.log` levels and scopes, a custom `logFn` emitting JSON, `std.debug.print` vs log, reading a panic trace line by line, `assert` vs returning an error, `dumpErrorReturnTrace` (and the two incompatible `StackTrace` types), `lldb` on a Zig binary and on a test binary |
| 9.5 | [Release Modes](./05-release-modes.md) | the four modes, overflow panic vs wrap, `unreachable` as a licence to miscompile, **leak detection silently disappearing**, bare-union `@sizeOf` and `undefined` differing by mode, measured speed and binary sizes, `@setRuntimeSafety`, which mode to ship |

Every block is complete and runnable as-is. **Test files have no `main` and must
be run with `zig test`, not `zig run`** — each page states the command. The
failures, panics and leaks on these pages are **deliberate**, labelled, and the
output is real; reading them is the skill the module teaches.

## The interview answer

> Tests live beside the code in `test` blocks and can see private declarations,
> and they cost nothing in a release binary because the compiler doesn't analyse
> them. Every allocating function gets a test using `std.testing.allocator`,
> which fails the test on a leak — and functions with multiple allocations get
> `checkAllAllocationFailures`, which exercises every `error.OutOfMemory` path
> and catches the error-path leaks nothing else reaches. CI checks the exit code,
> not the summary line, because a leaking run prints "All tests passed" and still
> exits 1. We ship `-Osafe`: an integer overflow should crash rather than wrap,
> and `-Ofast` turns both of those into undefined behaviour. Hot paths get
> `@setRuntimeSafety(false)` on the measured function instead of turning safety
> off program-wide.

---

Next: [Module 10 — Build System](../10-build-system/)
