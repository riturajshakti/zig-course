# Module 08 — Files & I/O

Lessons 1.3 and 1.4 gave you a stdout writer and the `io` parameter. This module
is the rest of the story: files, directories, readers, and the CLI programs that
tie them together.

0.17 rebuilt this layer from scratch — the release is nicknamed **"Writergate"**.
`std.fs` became `std.Io.Dir`/`std.Io.File`, every operation takes `io`, every
writer takes a buffer you own, and nothing reaches the disk until you `flush()`.
None of the file-I/O code you will find online compiles.

| Old (0.13–0.15) | 0.17.0 |
|---|---|
| `std.fs.cwd()` | `std.Io.Dir.cwd()` |
| `std.fs.File` | `std.Io.File` |
| `dir.makePath(p)` | `dir.createDirPath(io, p)` |
| `file.writer()` | `file.writer(io, &buf)` → `&w.interface` |
| `file.reader()` | `file.reader(io, &buf)` → `&r.interface` |
| `dir.readFileAlloc(gpa, p, max)` | `dir.readFileAlloc(io, p, gpa, .limited(max))` |
| `std.process.argsAlloc(gpa)` | `init.minimal.args.iterate()` |
| writes appear immediately | buffered — **`flush()` or lose everything** |

## Lessons

| # | Lesson | What you get |
|---|---|---|
| 8.1 | [The std.Io Model](./01-io-model.md) | `io` is 16 bytes and a vtable, why it is a parameter, `Threaded` vs `Evented`, passing it down, what it abstracts, and `io.async` running two 100 ms sleeps in 105 ms |
| 8.2 | [Readers & Writers](./02-readers-writers.md) | the `.interface` field, `*std.Io.Writer` as the portable parameter, buffer size measured against real drain counts, `Writer.fixed`, `Writer.Allocating`, positional vs streaming |
| 8.3 | [Reading Files](./03-reading-files.md) | `readFileAlloc` with `.limited()` (and why the off-by-one bites), line-by-line with `takeDelimiterInclusive`, `peek`/`take`/`discard`, EOF, and four real open errors |
| 8.4 | [Writing Files](./04-writing-files.md) | `writeFile`, `createFile` + writer + flush, a 0-byte file when you forget, truncate, exclusive, append, atomic replace |
| 8.5 | [Directories](./05-directories.md) | `createDirPath`, `iterate()`, `walk()`, `statFile`, delete vs deleteTree, existence checks, and all of `std.fs.path` |
| 8.6 | [CLI Programs](./06-cli-programs.md) | args via `iterate()`/`toSlice`, a real flag parser, **stdin without the hang**, `environ_map`, `main() u8` exit codes, stdout-vs-stderr discipline |

Every program is complete and runnable — copy a block into a `.zig` file and
`zig run` it as-is. Programs that fail on purpose are labelled, and the error
text on these pages is the real compiler or runtime output. Anything a lesson
writes goes to `/tmp/`, so nothing lands in your project.

## Two traps that will cost you an afternoon

**Nothing is written until you flush.** Not a warning, not an error — a 0-byte
file and exit code 0. Lesson 8.4 shows it happening.

**`writer()` is positional; `writerStreaming()` appends.** A positional writer
starts at offset 0 and overwrites, which matters even for stdout: when your
program inherits a terminal or a redirect that something already wrote to, a
positional stdout writer clobbers it. Use `writerStreaming` for stdout and
stderr in CLI programs. Lessons 8.2 and 8.6 demonstrate both halves.

---

Next: [Module 09 — Testing & Debugging](../09-testing-debugging/)
