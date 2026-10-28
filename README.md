# Zig 0.17.0 — A Tour for Working Developers

Code-first tour: every lesson is runnable programs and their real output.

| | |
|---|---|
| **Zig version** | 0.17.0 |
| **Written for** | devs who know Node.js / Go / Python / some Rust, new to Zig |
| **Goal** | systems programming & backend roles in Zig |
| **Every program** | executed on 0.17.0 — outputs are real, not written from memory |

---

## ⚠️ 0.17.0 broke the old APIs

Tutorials online target 0.13–0.15 and **will not compile here**:

| You'll see online | Reality in 0.17.0 |
|---|---|
| `std.fs.File` | `std.Io.File` |
| `std.fs.cwd()` | `std.Io.Dir.cwd()` |
| `getStdOut().writer()` | `std.Io.File.stdout().writer(io, &buf)` |
| I/O "just works" | I/O takes an explicit `io: std.Io` you create in `main` |
| writes appear instantly | **buffered — you must `flush()`** |
| `ArrayList.init(alloc)` | `.empty`, then `append(alloc, x)` |
| `GeneralPurposeAllocator` | `std.heap.DebugAllocator` |
| `std.process.argsAlloc` | `init.minimal.args.iterate()` |
| `std.json.stringifyAlloc` | `std.json.Stringify.valueAlloc` |
| `Clock.monotonic` | `Clock.awake` |
| `Duration.fromMs(10)` | `Duration.fromMilliseconds(10)` |
| `std.builtin` | works — now an alias of `std.lang` |

Stuck? The stdlib source is the real documentation: `$(zig env | grep std_dir)`.

---

## How to use

Every program is complete and runnable — copy a block into a file and run it:

```sh
zig run hello.zig      # any program from a lesson
zig test mytests.zig   # blocks containing `test { ... }`
```

Modules 1–5 in order; after that, jump around. ~5 min per lesson.

---

## Modules

**96 lessons across 15 modules.** One row = one lesson. ✅ = written & verified.

### [01 — Getting Started](modules/01-getting-started/README.md) · 5 lessons
*By the end: you can run Zig and print things.*

| # | Done | Lesson | What you learn |
|---|---|---|---|
| 1.1 | ✅ | [Setup & Verify](modules/01-getting-started/01-setup.md) | `zig version`, `zig run`, `zig fmt`, project layout |
| 1.2 | ✅ | [Hello World](modules/01-getting-started/02-hello-world.md) | `main`, `!void`, `std.debug.print` |
| 1.3 | ✅ | [Printing Properly](modules/01-getting-started/03-printing.md) | `std.Io.File.stdout()`, writers, **why you must `flush()`** |
| 1.4 | ✅ | [The `io` Parameter](modules/01-getting-started/04-the-io-parameter.md) | `std.Io.Threaded`, why 0.17 passes `io` everywhere |
| 1.5 | ✅ | [Reading the Source](modules/01-getting-started/05-reading-stdlib.md) | stdlib *is* the docs; how to find any API fast |

### [02 — Language Basics](modules/02-language-basics/README.md) · 6 lessons
*By the end: you can write ordinary procedural logic.*

| # | Done | Lesson | What you learn |
|---|---|---|---|
| 2.1 | ✅ | [Variables](modules/02-language-basics/01-variables.md) | `const`, `var`, `undefined`, shadowing, type inference |
| 2.2 | ✅ | [Numbers](modules/02-language-basics/02-numbers.md) | sized ints (`u8`..`u64`, `i32`), floats, `usize` |
| 2.3 | ✅ | [Overflow & Casting](modules/02-language-basics/03-overflow-casting.md) | overflow **panics**, `+%`, `@intCast`, `@as`, `@floatFromInt` |
| 2.4 | ✅ | [Control Flow](modules/02-language-basics/04-control-flow.md) | `if` as expression, `while`, `for (0..n)`, `for` over slices |
| 2.5 | ✅ | [Switch](modules/02-language-basics/05-switch.md) | exhaustiveness, ranges, multi-case, `else` |
| 2.6 | ✅ | [Blocks & Labels](modules/02-language-basics/06-blocks-labels.md) | labeled blocks return values, labeled `break`/`continue` |

### [03 — Types, Structs & Unions](modules/03-types-structs-unions/README.md) · 7 lessons
*By the end: you can model data.*

| # | Done | Lesson | What you learn |
|---|---|---|---|
| 3.1 | ✅ | [Arrays](modules/03-types-structs-unions/01-arrays.md) | fixed size, `[N]T`, `@splat` fill, `++` concat, `.len` |
| 3.2 | ✅ | [Slices](modules/03-types-structs-unions/02-slices.md) | slice/copy/compare/search/reverse/split + **the #1 dangling-pointer trap** |
| 3.3 | ✅ | [Strings](modules/03-types-structs-unions/03-strings.md) | no `String` type — concat, compare, split, trim, case, find, replace, parse, UTF-8 |
| 3.4 | ✅ | [Structs](modules/03-types-structs-unions/04-structs.md) | fields, defaults, methods, `.{}` literals, `Self` |
| 3.5 | ✅ | [Pointers](modules/03-types-structs-unions/05-pointers.md) | `*T`, `*const T`, `&x`, `.*`, `[*]T` vs `[]T` |
| 3.6 | ✅ | [Enums](modules/03-types-structs-unions/06-enums.md) | values, methods, `@tagName`, `@intFromEnum` |
| 3.7 | ✅ | [Tagged Unions](modules/03-types-structs-unions/07-unions.md) | `union(enum)`, `switch` capture — Zig's sum type |

### [04 — Errors & Optionals](modules/04-errors-optionals/README.md) · 6 lessons
*By the end: you handle failure the Zig way — no exceptions, no null pointers.*

| # | Done | Lesson | What you learn |
|---|---|---|---|
| 4.1 | ✅ | [Error Sets](modules/04-errors-optionals/01-error-sets.md) | `error{...}`, `!T`, inferred sets, merging with `||` |
| 4.2 | ✅ | [try & catch](modules/04-errors-optionals/02-try-catch.md) | `try`, `catch` with default, `catch |e|`, propagation |
| 4.3 | ✅ | [defer & errdefer](modules/04-errors-optionals/03-defer-errdefer.md) | cleanup that always runs; **cleanup only on failure** |
| 4.4 | ✅ | [Error Traces](modules/04-errors-optionals/04-error-traces.md) | return traces, `{!d}` formatting, debugging failures |
| 4.5 | ✅ | [Optionals](modules/04-errors-optionals/05-optionals.md) | `?T`, `orelse`, `if (x) |v|`, `.?`, optional pointers |
| 4.6 | ✅ | [Error vs Optional](modules/04-errors-optionals/06-error-vs-optional.md) | which to use when; `unreachable` and asserts |

### [05 — Memory & Allocators](modules/05-memory-allocators/README.md) · 8 lessons ⭐
*Matters most for interviews; the one Node/Python/Go never taught you. By the end: you can answer "who frees this?"*

| # | Done | Lesson | What you learn |
|---|---|---|---|
| 5.1 | ✅ | [Stack vs Heap](modules/05-memory-allocators/01-stack-vs-heap.md) | what lives where, why Zig makes you choose |
| 5.2 | ✅ | [The Allocator Interface](modules/05-memory-allocators/02-allocator-interface.md) | `std.mem.Allocator`, `alloc`/`free`/`create`/`destroy` |
| 5.3 | ✅ | [DebugAllocator](modules/05-memory-allocators/03-debug-allocator.md) | leak detection — **watch a real leak get caught** |
| 5.4 | ✅ | [Arena](modules/05-memory-allocators/04-arena.md) | allocate freely, free once — the server pattern |
| 5.5 | ✅ | [FixedBuffer](modules/05-memory-allocators/05-fixed-buffer.md) | zero heap allocation, stack-only workloads |
| 5.6 | ✅ | [Ownership Rules](modules/05-memory-allocators/06-ownership.md) | who allocates, who frees, documenting it in signatures |
| 5.7 | ✅ | [Write an Allocator](modules/05-memory-allocators/07-write-your-own.md) | build a pool allocator from scratch, implement the vtable |
| 5.8 | ✅ | [Common Leaks](modules/05-memory-allocators/08-common-leaks.md) | the 5 mistakes that leak, and how each is caught |

### [06 — Comptime & Generics](modules/06-comptime-generics/README.md) · 6 lessons ⭐
*Zig's real differentiator. By the end: you can write generic, zero-cost abstractions.*

| # | Done | Lesson | What you learn |
|---|---|---|---|
| 6.1 | ✅ | [comptime Basics](modules/06-comptime-generics/01-comptime-basics.md) | compile-time execution, `comptime` vars, asserts |
| 6.2 | ✅ | [Types Are Values](modules/06-comptime-generics/02-types-are-values.md) | `type` as a value — generics are just functions |
| 6.3 | ✅ | [Generic Functions](modules/06-comptime-generics/03-generic-functions.md) | `comptime T: type`, `anytype`, duck typing |
| 6.4 | ✅ | [Generic Containers](modules/06-comptime-generics/04-generic-containers.md) | write your own `Stack(T)`, like `ArrayList` does |
| 6.5 | ✅ | [Reflection](modules/06-comptime-generics/05-reflection.md) | `@typeInfo`, `inline for` over fields, auto-serialization |
| 6.6 | ✅ | [comptime Costs](modules/06-comptime-generics/06-costs.md) | code bloat, compile times, when *not* to be generic |

### [07 — Standard Library Tour](modules/07-stdlib-tour/README.md) · 7 lessons
*Everyday data work. Container lessons cover every operation on each type — built to come back to.*

| # | Done | Lesson | What you learn |
|---|---|---|---|
| 7.1 | ✅ | [ArrayList](modules/07-stdlib-tour/01-arraylist.md) | **unmanaged in 0.17** — append, insert, remove, pop, iterate, sort, clear, toOwnedSlice |
| 7.2 | ✅ | [HashMap](modules/07-stdlib-tour/02-hashmap.md) | put, get, remove, contains, getOrPut, iterate, count, keys/values, clear |
| 7.3 | ✅ | [std.mem](modules/07-stdlib-tour/03-std-mem.md) | `eql`, `splitScalar`, `tokenizeScalar`, `join`, `indexOf` |
| 7.4 | ✅ | [Sorting & Search](modules/07-stdlib-tour/04-sorting.md) | `std.sort.pdq`, custom comparators, binary search |
| 7.5 | ✅ | [Formatting](modules/07-stdlib-tour/05-formatting.md) | format specifiers, `allocPrint`, `bufPrint`, custom `format` |
| 7.6 | ✅ | [JSON](modules/07-stdlib-tour/06-json.md) | parse to a struct, stringify, dynamic values |
| 7.7 | ✅ | [Time & Random](modules/07-stdlib-tour/07-time-random.md) | timestamps, measuring duration, PRNG seeding |

### [08 — Files & I/O](modules/08-files-io/README.md) · 6 lessons
*By the end: you can write real CLI tools.*

| # | Done | Lesson | What you learn |
|---|---|---|---|
| 8.1 | ✅ | [The std.Io Model](modules/08-files-io/01-io-model.md) | why `io` is explicit, `Threaded` vs `Evented` |
| 8.2 | ✅ | [Readers & Writers](modules/08-files-io/02-readers-writers.md) | the `.interface` pattern, buffering, **`flush()`** |
| 8.3 | ✅ | [Reading Files](modules/08-files-io/03-reading-files.md) | `readFileAlloc` with `.limited()`, line-by-line streaming |
| 8.4 | ✅ | [Writing Files](modules/08-files-io/04-writing-files.md) | `writeFile`, create/append, atomic replace |
| 8.5 | ✅ | [Directories](modules/08-files-io/05-directories.md) | `std.Io.Dir`, walking trees, stat, paths |
| 8.6 | ✅ | [CLI Programs](modules/08-files-io/06-cli-programs.md) | args, stdin, env vars, exit codes |

### [09 — Testing & Debugging](modules/09-testing-debugging/README.md) · 5 lessons
*By the end: your code is verifiable.*

| # | Done | Lesson | What you learn |
|---|---|---|---|
| 9.1 | ✅ | [Writing Tests](modules/09-testing-debugging/01-writing-tests.md) | `test {}` blocks, `zig test`, naming |
| 9.2 | ✅ | [Assertions](modules/09-testing-debugging/02-assertions.md) | `expectEqual`, `expectEqualStrings`, `expectError` |
| 9.3 | ✅ | [testing.allocator](modules/09-testing-debugging/03-testing-allocator.md) | **a leak fails the test** — free memory-safety checking |
| 9.4 | ✅ | [Debugging](modules/09-testing-debugging/04-debugging.md) | `std.log` levels, panics, stack traces, `lldb` |
| 9.5 | ✅ | [Release Modes](modules/09-testing-debugging/05-release-modes.md) | how Debug/ReleaseSafe/ReleaseFast change *behavior* |

### [10 — Build System & Cross-Compilation](modules/10-build-system/README.md) · 6 lessons
*By the end: you can ship a binary.*

| # | Done | Lesson | What you learn |
|---|---|---|---|
| 10.1 | ✅ | [build.zig](modules/10-build-system/01-build-zig.md) | the build graph, `addExecutable`, `zig build` |
| 10.2 | ✅ | [Steps & Options](modules/10-build-system/02-steps-options.md) | `zig build run`/`test`, custom steps, `-D` options |
| 10.3 | ✅ | [Modules](modules/10-build-system/03-modules.md) | splitting code into files and modules, imports |
| 10.4 | ✅ | [Dependencies](modules/10-build-system/04-dependencies.md) | `build.zig.zon`, fetching packages, hashes |
| 10.5 | ✅ | [Cross-Compilation](modules/10-build-system/05-cross-compilation.md) | **Mac → Linux in one flag** — Zig's headline feature |
| 10.6 | ✅ | [Shipping](modules/10-build-system/06-shipping.md) | static binaries, size tuning, Docker-free deploys |

### [11 — C Interop](modules/11-c-interop/README.md) · 5 lessons
*By the end: you can use the entire C ecosystem.*

| # | Done | Lesson | What you learn |
|---|---|---|---|
| 11.1 | ✅ | [Calling C](modules/11-c-interop/01-calling-c.md) | **`@cImport` is gone** — `extern fn` + `-lc`, `std.c`, variadics, `translate-c` |
| 11.2 | ✅ | [C Types](modules/11-c-interop/02-c-types.md) | `c_int`, pointers, `[*c]T`, structs across the line |
| 11.3 | ✅ | [C Strings](modules/11-c-interop/03-c-strings.md) | `[:0]const u8` vs `[]const u8`, `span`, conversions |
| 11.4 | ✅ | [Linking Libraries](modules/11-c-interop/04-linking.md) | system libs in `build.zig`, compiling C sources |
| 11.5 | ✅ | [Exporting to C](modules/11-c-interop/05-exporting.md) | `export fn`, building a C-callable static library |

### [12 — Concurrency & Async](modules/12-concurrency-async/README.md) · 7 lessons
*By the end: you can write concurrent servers.*

| # | Done | Lesson | What you learn |
|---|---|---|---|
| 12.1 | ✅ | [async & await](modules/12-concurrency-async/01-async-await.md) | `io.async`, `await`, how 0.17's model works |
| 12.2 | ✅ | [concurrent](modules/12-concurrency-async/02-concurrent.md) | `io.concurrent`, true parallelism vs concurrency |
| 12.3 | ✅ | [Cancellation](modules/12-concurrency-async/03-cancellation.md) | `Cancelable`, timeouts, cooperative shutdown |
| 12.4 | ✅ | [Threads](modules/12-concurrency-async/04-threads.md) | `std.Thread`, spawn & join, thread-per-task |
| 12.5 | ✅ | [Mutexes & Locks](modules/12-concurrency-async/05-mutexes.md) | `Mutex`, `RwLock`, protecting shared state |
| 12.6 | ✅ | [Atomics](modules/12-concurrency-async/06-atomics.md) | atomic counters, orderings, lock-free basics |
| 12.7 | ✅ | [Concurrent Echo Server](modules/12-concurrency-async/07-echo-server.md) | a TCP server handling many clients at once |

### [13 — Networking & HTTP](modules/13-networking-http/README.md) · 6 lessons
*By the end: you can speak HTTP.*

| # | Done | Lesson | What you learn |
|---|---|---|---|
| 13.1 | ✅ | [Addresses & Sockets](modules/13-networking-http/01-addresses.md) | `std.Io.net`, `IpAddress`, DNS resolution |
| 13.2 | ✅ | [TCP Client](modules/13-networking-http/02-tcp-client.md) | `connect`, stream reader/writer, framing |
| 13.3 | ✅ | [TCP Server](modules/13-networking-http/03-tcp-server.md) | `listen`, `accept`, connection lifecycle |
| 13.4 | ✅ | [HTTP Client](modules/13-networking-http/04-http-client.md) | `std.http.Client`, GET/POST, headers, JSON bodies |
| 13.5 | ✅ | [HTTP Server](modules/13-networking-http/05-http-server.md) | `std.http.Server`, requests, responses, status codes |
| 13.6 | ✅ | [Serving JSON](modules/13-networking-http/06-serving-json.md) | content types, encoding structs, error responses |

### [14 — Capstone: A JSON HTTP Service](modules/14-capstone-service/README.md) · 7 lessons 🎯
*One real project, end to end. By the end: you have something to show an interviewer.*

| # | Done | Lesson | What you learn |
|---|---|---|---|
| 14.1 | ✅ | [Design & Layout](modules/14-capstone-service/01-design.md) | project structure, module boundaries, the API spec |
| 14.2 | ✅ | [Routing](modules/14-capstone-service/02-routing.md) | method + path dispatch, params, 404s |
| 14.3 | ✅ | [Per-Request Arena](modules/14-capstone-service/03-request-arena.md) | **the idiomatic server pattern** — arena per request |
| 14.4 | ✅ | [Store & State](modules/14-capstone-service/04-store.md) | shared state behind a mutex, CRUD handlers |
| 14.5 | ✅ | [Error Handling](modules/14-capstone-service/05-errors.md) | mapping Zig errors → HTTP status + JSON bodies |
| 14.6 | ✅ | [Config & Shutdown](modules/14-capstone-service/06-config-shutdown.md) | env config, signals, graceful shutdown |
| 14.7 | ✅ | [Tests & Release](modules/14-capstone-service/07-tests-release.md) | handler tests, load check, `ReleaseFast` build |

### [15 — Zig for Systems & Backend Work](modules/15-systems-backend/README.md) · 9 lessons 💼
*The job-hunting module. By the end: you can talk about Zig like someone who uses it.*

| # | Done | Lesson | What you learn |
|---|---|---|---|
| 15.1 | ✅ | [Idiomatic Zig](modules/15-systems-backend/01-idiomatic-zig.md) | what reviewers look for; naming & API conventions |
| 15.2 | ✅ | [Memory Layout](modules/15-systems-backend/02-memory-layout.md) | `@sizeOf`/`@alignOf`/`@offsetOf`, padding, **field reordering**, `packed` vs `extern` |
| 15.3 | ✅ | [Bits & Bytes](modules/15-systems-backend/03-bits-and-bytes.md) | `@bitCast`, `asBytes`, endianness, bit manipulation, binary protocols |
| 15.4 | ✅ | [Data-Oriented Design](modules/15-systems-backend/04-data-oriented.md) | `MultiArrayList`, SoA vs AoS, cache lines, why layout beats cleverness |
| 15.5 | ✅ | [Zero-Copy Parsing](modules/15-systems-backend/05-zero-copy.md) | slices into buffers, parsing with zero allocation |
| 15.6 | ✅ | [Syscalls & the OS](modules/15-systems-backend/06-syscalls.md) | `std.posix`, `mmap`, `uname`, rlimits, fds, what the OS actually gives you |
| 15.7 | ✅ | [Measuring Performance](modules/15-systems-backend/07-performance.md) | timing, benchmarking honestly, release modes, reading the machine |
| 15.8 | ✅ | [Zig vs C, Go & Rust](modules/15-systems-backend/08-zig-vs-others.md) | where Zig wins & loses; **answering "why Zig?"** |
| 15.9 | ✅ | [Interview Prep](modules/15-systems-backend/09-interview-prep.md) | real questions + answers, ecosystem state, what to build |

---

## Reference

Written last, once the modules they summarize exist.

| File | Contents |
|---|---|
| `reference/allocators.md` | which allocator when |
| `reference/migration-0.17.md` | old API → new API |
| `reference/gotchas.md` | flush, dangling slices, overflow |
| `reference/further-reading.md` | where to go next |

## Progress

- [ ] 01 Getting Started
- [ ] 02 Language Basics
- [ ] 03 Types, Structs & Unions
- [ ] 04 Errors & Optionals
- [ ] 05 Memory & Allocators
- [ ] 06 Comptime & Generics
- [ ] 07 Standard Library Tour
- [ ] 08 Files & I/O
- [ ] 09 Testing & Debugging
- [ ] 10 Build System & Cross-Compilation
- [ ] 11 C Interop
- [ ] 12 Concurrency & Async
- [ ] 13 Networking & HTTP
- [ ] 14 Capstone Service
- [ ] 15 Systems & Backend Work
