# Module 01 — Getting Started

Five short lessons to get you from `zig version` to writing real Zig on **0.17.0**.

> **Read this first.** Zig 0.17.0 redesigned the standard library. Almost every
> tutorial, blog post and StackOverflow answer you will find online was written
> for 0.13–0.15 and **will not compile**. The biggest renames:
>
> | What tutorials say | Zig 0.17.0 |
> |---|---|
> | `std.fs.File` | `std.Io.File` |
> | `std.io.getStdOut()` | `std.Io.File.stdout()` |
> | `std.fs.cwd()` | `std.Io.Dir.cwd()` |
> | `file.writer()` | `file.writer(io, &buf)` |
> | writes appear immediately | **buffered — you must `flush()`** |
> | `pub fn main() !void` | that still works, but `pub fn main(init: std.process.Init) !void` is now preferred |
>
> Note the capital `I` in `std.Io`. Lowercase `std.io` no longer exists at all.

## Lessons

| # | Lesson | What you get |
|---|---|---|
| 1.1 | [Setup & Verify](./01-setup.md) | `zig version`, `zig run`, `zig fmt`, `zig test`, and where the stdlib lives on your disk |
| 1.2 | [Hello World](./02-hello-world.md) | `pub fn main() !void`, what `!void` actually means, `std.debug.print` |
| 1.3 | [Printing Properly](./03-printing.md) | real stdout writers, **the flush trap**, stdout vs stderr, format specifiers |
| 1.4 | [The `io` Parameter](./04-the-io-parameter.md) | `std.process.Init`, `std.Io.Threaded`, and *why* 0.17 made `io` explicit |
| 1.5 | [Reading the Source](./05-reading-stdlib.md) | the stdlib is the documentation — grep recipes to answer any API question |

Every program in these lessons is complete. Copy a block into a `.zig` file and
`zig run` it as-is.

---

Next: [Module 02 — Language Basics](../02-language-basics/)
