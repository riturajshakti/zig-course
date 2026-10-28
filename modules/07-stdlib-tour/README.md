# Module 07 — Standard Library Tour

Seven lessons on the parts of the standard library you reach for every day:
the two containers, the slice toolbox, sorting, formatting, JSON, and clocks.

**This module is built to be re-read, not just read.** The container lessons
(7.1, 7.2) and `std.mem` (7.3) each open with a dense table of *every* operation
on the type — create, add, read, update, remove, count, iterate, search, sort,
clear, convert, free — followed by runnable programs that show them working.
Come back to the tables; skim the prose once.

## Lessons

| # | Lesson | What you get |
|---|---|---|
| 7.1 | [ArrayList](./01-arraylist.md) | **unmanaged in 0.17** — ~35 operations, `appendAssumeCapacity`, `swapRemove` vs `orderedRemove`, `toOwnedSlice`, the string builder, and **why appending invalidates your pointers** |
| 7.2 | [HashMap](./02-hashmap.md) | **still managed** — `getOrPut`, `fetchRemove`, the three iterators, sets, `BufSet`, sorted output, and **the key-lifetime trap** |
| 7.3 | [std.mem](./03-std-mem.md) | ~45 operations: compare, search, copy direction, split vs tokenize, join/concat/replace, and raw bytes with endianness |
| 7.4 | [Sorting & Search](./04-sorting.md) | `std.mem.sort`, multi-key comparators, context instead of closures, stable vs unstable **observed**, `binarySearch` / `lowerBound` / `equalRange` |
| 7.5 | [Formatting](./05-formatting.md) | every specifier, width/precision/fill, `allocPrint` / `bufPrint` / `comptimePrint`, custom `format` for `{f}`, parsing numbers back |
| 7.6 | [JSON](./06-json.md) | parse to a struct, **`parsed.deinit()` frees your strings**, arena + leaky parsing, `Stringify` options, dynamic `std.json.Value`, round trip |
| 7.7 | [Time & Random](./07-time-random.md) | **`std.time.Timer` is gone** — clocks via `io`, durations, deadlines, seeded PRNGs, and why secrets never come from one |

## The five things worth slowing down for

1. **The containers disagree about allocators.** `ArrayList` is unmanaged
   (`.empty`, then `append(gpa, x)`); `HashMap` is managed (`.init(gpa)`, then
   `deinit()`). Mixing them up is the most common 0.17 compile error in
   container code. (7.1, 7.2)
2. **Appending invalidates every pointer into an `ArrayList`.** Hold indices, not
   `*T`. Under `DebugAllocator` a stale pointer **segfaults** rather than
   returning garbage. (7.1)
3. **A HashMap does not copy its keys.** Put a key from a reused buffer and the
   entry becomes silently unreachable — no crash, no warning. Lesson 7.2 shows it
   happening and gives the `getOrPut` + `dupe` fix. (7.2)
4. **`parsed.deinit()` frees the strings inside `parsed.value`.** Returning
   `parsed.value.name` from a function is a use-after-free that *looks* like it
   works. (7.6)
5. **`deinit` frees the container, never what the elements point at.** Every
   lesson here is leak-clean under `DebugAllocator`; the discipline is the point.

## Things 0.17.0 changed

Tutorials written for 0.13–0.15 will hand you all of these:

| What you'll find online | Zig 0.17.0 |
|---|---|
| `std.ArrayList(T).init(gpa)` | `var l: std.ArrayList(T) = .empty;` |
| `list.append(x)` / `list.deinit()` | `list.append(gpa, x)` / `list.deinit(gpa)` |
| `std.json.stringifyAlloc` | `std.json.Stringify.valueAlloc` |
| `std.time.Timer.start()` | `std.Io.Clock.Timestamp.now(io, .awake)` |
| `std.time.timestamp()` | `…now(io, .real).raw.toSeconds()` |
| `std.fmt.fmtDuration(ns)` | `{f}` on a `std.Io.Duration` |
| `std.crypto.random.bytes(buf)` | `io.random(buf)` / `try io.randomSecure(buf)` |
| `pub fn format(self, fmt, opts, writer)` | `pub fn format(self: T, w: *std.Io.Writer) !void` |
| `std.mem.trimLeft` / `trimRight` | `std.mem.trimStart` / `trimEnd` |
| `std.mem.indexOf` | works — now an alias of `std.mem.find` |
| `Clock.monotonic` | `Clock.awake` |
| `[_]u8{'-'} ** 29` | **`**` was removed** — `const l: [29]u8 = @splat('-')` |

Anything that reads a clock needs `pub fn main(init: std.process.Init)`, because
time is I/O in 0.17.

Every program is complete. Copy a block into a `.zig` file and `zig run` it
as-is — including the ones showing a trap, which are labelled so you recognize
the symptom when you cause it yourself.

---

Next: [Module 08 — Files & I/O](../08-files-io/)
