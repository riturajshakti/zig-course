# Module 15 — Zig for Systems & Backend Work 💼

**The job-hunting module.** Everything here serves one goal: being able to write
Zig a reviewer would approve, reason about what the machine actually does, and
talk about the language like someone who uses it rather than someone who read
about it.

Lessons 15.1–15.7 are code and measurements. **15.8 and 15.9 are the interview
prep** — the honest comparison against C, Go and Rust, and real questions with
answers you could say out loud.

Every number in this module was measured on one machine (macOS, arm64, Zig
0.17.0) with the build mode stated. The *ratios* travel; the microseconds don't.

## Lessons

| # | Lesson | What you get |
|---|---|---|
| 15.1 | [Idiomatic Zig](./01-idiomatic-zig.md) | the six things a reviewer checks; `init`/`deinit` vs `create`/`destroy`, allocator-first signatures, three shapes of `errdefer` (including the one everyone forgets), errors not sentinels, named vs inferred error sets, `*std.Io.Writer` as a parameter, when `anytype` earns its keep, plus a review checklist |
| 15.2 | [Memory Layout](./02-memory-layout.md) | **Zig reorders your fields** — `offsetOf(b) == 0` proven; plain vs `packed` vs `extern` side by side; the same 8 wire bytes read correctly through `extern` and as **garbage** through a plain struct; alignment rules; struct-size golf 32 → 16 bytes, and why `packed` is *not* an optimization |
| 15.3 | [Bits & Bytes](./03-bits-and-bytes.md) | `@bitCast`/`@truncate`/`@intCast`/`@byteSwap` and when each is legal; ~12 bit operations; a real **DNS flags header** as a `packed struct(u16)` (LSB-first — the trap); the `asBytes`/`toBytes` view-vs-copy distinction; a length-prefixed wire format parsed with all four error paths exercised |
| 15.4 | [Data-Oriented Design](./04-data-oriented.md) | `std.MultiArrayList` with its full operations table; SoA vs AoS **measured at 5.8x on one hot field and ~1.0x on three of six** — the honest result; why a benchmark that reports 0 us isn't fast; `@sizeOf` budgeting numbers worth memorizing |
| 15.5 | [Zero-Copy Parsing](./05-zero-copy.md) | `split` vs `tokenize` (empty fields decide it); a log-line parser and an HTTP request parser with **zero allocations** and no allocator parameter; **measured** zero-copy vs arena vs `dupe`/`free` at 1.0 / 1.8 / 4.2x; and a parsed `"alpha"` silently becoming `"ZZZZZ"` |
| 15.6 | [Syscalls & the OS](./06-syscalls.md) | what the **thinned** `std.posix` still exposes and what moved to `std.Io`; `mmap` with 0.17's **packed-struct PROT flags**; `mincore`/`madvise`; mapping a file via `file.handle`; fds and `poll`; `uname`/`getrlimit`/`getrusage` read operationally; `strace`/`dtruss` |
| 15.7 | [Measuring Performance](./07-performance.md) | **`std.time.Timer` is gone** — timing via `std.Io.Clock`; benchmark hygiene with a real 2.62x outlier that moved the mean 9% and the median 0.3%; all four release modes measured for **size and speed** (ReleaseSmall is 3.3x *slower*); `@setRuntimeSafety(false)`; reading `--summary`; what actually matters in a service, ranked |
| 15.8 | [Zig vs C, Go & Rust](./08-zig-vs-others.md) | a one-screen comparison; six places Zig genuinely wins and **seven where it loses**; verified cross-compile numbers (134 KB static Linux binary in under a second); a full scripted answer to **"why Zig?"**, three answers that lose the room, and what to say when they push back |
| 15.9 | [Interview Prep](./09-interview-prep.md) | nine real questions with runnable answers — allocators, `defer`/`errdefer`, slices vs pointers, `comptime`, error handling → HTTP status, release modes, `std.Io`, leaks, service structure; what to build for a portfolio; the **honest state of the 0.17 ecosystem** and how to talk about the instability; 17 rapid-fire answers |

## The five things worth slowing down for

1. **Zig reorders struct fields.** `struct { a: u8, b: u32, c: u8 }` puts `b` at
   offset **0**. That's why `extern struct` exists, and 15.2 shows the same wire
   bytes decoded correctly one way and as garbage the other — with no crash and
   no warning.
2. **Layout beats cleverness, but only sometimes.** SoA was 5.8x faster on one
   hot field and *exactly break-even* on three fields of six. "SoA is faster" is
   wrong; "match the layout to the access pattern" is the answer.
3. **A parser needs no allocator.** `fn parse(buf: []const u8) !Message` can't
   allocate and can't return `OutOfMemory` — the signature proves it. The cost
   is that the result borrows the buffer, and 15.5 shows exactly how that bites.
4. **Release modes change behavior, not just speed.** Overflow panics in
   Debug/ReleaseSafe and silently wraps in ReleaseFast/ReleaseSmall, and
   `DebugAllocator`'s leak detection reports `ok` on a real leak in the build
   you ship. Catch leaks in **tests** — `std.testing.allocator` works in all
   four modes.
5. **Volunteer the weaknesses.** 15.8's strongest interview move is recommending
   Rust or Go for a specific requirement. They're testing whether you choose
   tools by trade-offs, not whether you like Zig.

## 0.17.0 specifics this module depends on

| What you'll find online | Zig 0.17.0 |
|---|---|
| `std.time.Timer.start()` | `std.Io.Clock.Timestamp.now(io, .awake)` — **not fallible**, and `.raw` to reach `toMicroseconds()` |
| `posix.open` / `close` / `write` / `socket` / `fork` | **all gone** — `std.Io.File`, `std.Io.net`, `std.process.Child` |
| `PROT.READ \| PROT.WRITE` for `mmap` | **a packed struct**: `.{ .READ = true, .WRITE = true }` |
| `std.os.linux.*` on a Mac | **won't compile** — gated on target OS; use `std.posix` for portable code |
| `errdefer \|e\| { ... }` | **doesn't exist** — plain `errdefer` only |
| `@intFromEnum` / `@enumFromInt` | compile, but `zig fmt` rewrites them to `@backingInt` / `@intCast` |
| hardcoded 4096 page size | `std.heap.pageSize()` — **16384** on Apple silicon |
| `@typeInfo(T).Struct.fields` | `.field_names` + `.field_types` (parallel arrays) |

**`@bitSizeOf` on a plain or `extern` struct is a compile error**
(`no bit size available`) — it only works on `packed` structs, because padding
bytes aren't bits of the value.

Every program is complete and runnable: copy a block into a `.zig` file and
`zig run` it. The benchmarks need `pub fn main(init: std.process.Init)` because
reading a clock is I/O in 0.17.

---

## 🎓 End of the course

Fifteen modules, from `zig version` to answering "why Zig?" in an interview.

The habit that mattered most wasn't a language feature: **read the source, run
the program.** Every number in this module was measured on the machine it was
written on, and every program was executed before being pasted — because in an
ecosystem where the docs are a version behind and search results don't compile,
that's the only thing that works.

Now go build something and measure it.

← [Module 14 — Capstone Service](../14-capstone-service/) ·
[Course index](../../README.md)
