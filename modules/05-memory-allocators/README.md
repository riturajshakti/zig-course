# Module 05 — Memory & Allocators ⭐

**The most important module in this course.** Node, Python and Go manage memory
for you; Rust manages it with a borrow checker. Zig does neither — you pass an
allocator explicitly and you say who frees what. It is also the #1 thing Zig
interviews ask about.

The good news: Zig gives you a tool that catches your mistakes by **printing the
exact line that leaked**. Most of this module is spent writing programs that
fail on purpose so you learn to read those reports.

> **0.17.0 renames.** `GeneralPurposeAllocator` is now
> `std.heap.DebugAllocator`. The allocator vtable has **four** functions
> (`alloc`, `resize`, `remap`, `free`) — `remap` is new. `ArrayList` is
> unmanaged: `.empty`, `append(gpa, x)`, `deinit(gpa)`. `std.time.Timer` is
> gone. Anything you read online targeting 0.13–0.15 will not compile.

## Lessons

| # | Lesson | What you get |
|---|---|---|
| 5.1 | [Stack vs Heap](./01-stack-vs-heap.md) | what lives where, returning a pointer to a local (two versions — one the compiler catches, one it doesn't), struct by value vs pointer, a real stack overflow |
| 5.2 | [The Allocator Interface](./02-allocator-interface.md) | `std.mem.Allocator` is 16 bytes you pass around, `alloc`/`free`/`create`/`destroy`/`dupe`/`realloc`/`alignedAlloc`, why there's no global allocator, the `error.OutOfMemory` contract |
| 5.3 | [DebugAllocator](./03-debug-allocator.md) | `.init`, `defer _ = gpa.deinit()`, **a real leak report naming the line**, double-free detection with three stack traces, why leak detection vanishes in `ReleaseFast` |
| 5.4 | [Arena](./04-arena.md) | allocate freely and free once, `reset` modes, **the per-request server pattern**, ~400x faster, and the memory-growth trap |
| 5.5 | [FixedBuffer](./05-fixed-buffer.md) | zero heap allocation, stack-backed, a real `error.OutOfMemory`, caller-provides-buffer APIs, `page_allocator` vs `smp_allocator` |
| 5.6 | [Ownership Rules](./06-ownership.md) | the four signature shapes, `toOwnedSlice`, `defer` vs `errdefer`, multi-step constructors, containers that own their contents |
| 5.7 | [Write an Allocator](./07-write-your-own.md) | implement the vtable yourself — a counting wrapper, a stats profiler, a bump allocator, a pool allocator with slot reuse |
| 5.8 | [Common Leaks](./08-common-leaks.md) | the six recurring mistakes, each as a leaking program plus its fix, and how `std.testing.allocator` fails a test on a leak |

Every program is complete and runnable. Copy a block into a `.zig` file and
`zig run` it as-is. **The failing ones are labelled** — the leak reports,
panics and crashes on these pages are the real output of deliberately broken
programs, and reading them is the skill.

## The interview answer

By the end of 5.8 you should be able to say this without hesitating:

> Long-lived state uses a general-purpose allocator and is freed explicitly,
> with `defer` next to each allocation and `errdefer` where memory is handed to
> the caller. Per-request work comes from an arena that's reset when the
> response is written, so handlers never free anything individually and can't
> leak. In development and tests the allocator is a `DebugAllocator`, so a leak
> is reported with the line that allocated it; in tests it's
> `std.testing.allocator`, so a leak fails the build.

---

Next: [Module 06 — Comptime & Generics](../06-comptime-generics/)
