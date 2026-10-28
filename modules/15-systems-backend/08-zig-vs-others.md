# 15.8 — Zig vs C, Go & Rust

You will be asked "why Zig?" in every interview where Zig appears on your CV.
The bad answer is a list of features. The good answer is a *trade-off* you
understand well enough to argue the other side of.

This lesson is the honest version — including where Zig loses, because an
interviewer who uses Zig knows exactly where it loses and is listening for
whether you do too.

## The one-screen comparison

| | C | Zig | Rust | Go |
|---|---|---|---|---|
| **memory management** | manual, implicit | manual, **explicit allocators** | ownership + borrow checker | GC |
| **memory safety** | none | bounds/overflow checks in safe modes; **no use-after-free protection** | **compile-time guaranteed** (safe subset) | GC-guaranteed |
| **data races** | on you | **on you** | **prevented by the compiler** | detectable at runtime (`-race`) |
| **error handling** | return codes, `errno` | **error unions, `try`, forced handling** | `Result<T,E>`, `?` | `if err != nil` |
| **null** | null pointers everywhere | `?T`, no null pointers | `Option<T>` | nil pointers |
| **hidden control flow** | no | **none by design** | `Drop`, deref coercion, operator overloading | GC pauses, `defer`, panics |
| **hidden allocation** | no | **none — nothing allocates without an allocator param** | collections allocate implicitly | everything allocates implicitly |
| **metaprogramming** | preprocessor | **`comptime` — just Zig at compile time** | macros (2 systems) + generics | generics, `go:generate` |
| **compile time (hello world)** | ~0.1 s | **~1.0 s Debug, 4.6 s ReleaseFast** | slow (10s+ for real projects) | **very fast** |
| **cross-compilation** | painful | **one flag, zero setup** | needs toolchains per target | one flag (cgo breaks it) |
| **C interop** | native | **`@cImport` — no bindings to write** | `bindgen`, `unsafe` blocks | cgo (slow, awkward) |
| **binary size (hello)** | ~50 KB | **165 KB, or 134 KB static Linux** | ~300 KB+ | ~1.5 MB |
| **ecosystem** | 50 years of libraries | **small and churning** | large, mature (crates.io) | large, mature |
| **language stability** | frozen | **pre-1.0, breaks every release** | stable since 2015 | stable since 2012 |
| **hiring market** | large | **tiny** | growing | large |
| **concurrency** | pthreads | `std.Io` async + threads (**new in 0.17, immature**) | async + Send/Sync | **goroutines + channels** |

## Where Zig genuinely wins

**1. Allocators are visible in the type system.**

This is the real headline, and it's not primarily about speed. In Zig, a
function that can allocate *says so in its signature*:

```zig
fn parse(buf: []const u8) !Message              // cannot allocate. Period.
fn parse(gpa: Allocator, buf: []const u8) !Message   // can
```

In Rust, `String::from(s)` allocates and the signature doesn't mention it. In
Go and Node, everything allocates and you have no idea where. In Zig you can
look at a module's public API and know its allocation behavior, and you can
hand a different allocator to a subsystem — an arena per request, a fixed
buffer in an interrupt handler, a failing allocator in a test — without
changing a line inside it.

The practical payoff: **the per-request arena**. One `arena.deinit()` at the end
of a handler, and that handler cannot leak, no matter what it allocated. 15.5
measured it at 1.8x slower than zero-copy versus 4.2x for `dupe`/`free`, and it
removes an entire class of bug. Rust can do this (`bumpalo`) but it's not the
default; Go cannot do it at all.

**2. `comptime` is just Zig.**

One language for runtime and compile time. `ArrayList(T)` is a function that
takes a type and returns a type. Compare: C has the preprocessor, C++ has
templates *and* `constexpr`, Rust has declarative macros *and* procedural
macros *and* generics *and* `const fn` — four systems with four syntaxes.

This genuinely reduces what you have to learn, and reflection
(`@typeInfo`, `inline for` over fields) gives you JSON serialization, ORMs and
protocol codegen without a build-time code generator.

**3. Cross-compilation that actually works.**

Verified on this machine, from macOS arm64:

```sh
zig build-exe hello.zig -O ReleaseSmall -fstrip -target x86_64-linux-musl
```

| Target | Time | Output |
|---|---|---|
| `x86_64-linux-musl` | **0.98 s** | 134,144 B, "ELF 64-bit LSB executable, x86-64, statically linked, stripped" |
| `aarch64-linux-musl` | < 1 s | 131,000 B |
| `x86_64-windows` | < 1 s | 423,936 B |

**No Docker, no cross toolchain, no sysroot, no `apt install gcc-x86-64-linux-gnu`.**
Zig ships libc sources for every target and builds what it needs. A 134 KB
static binary with no dependencies means your container is `FROM scratch`.

Rust needs `rustup target add` plus a linker for each target, and anything
touching C needs a real cross toolchain. Go is good here until you need cgo, and
then it isn't. This is the feature that makes people adopt Zig *as a C compiler*
for non-Zig projects.

**4. C interop with no binding layer.**

```zig
const c = @cImport({ @cInclude("sqlite3.h"); });
```

That's it — real types, real functions, no `bindgen`, no `.h` translation step,
no `unsafe` block, no FFI marshalling. `zig cc` is also a drop-in C compiler, so
you can adopt Zig one file at a time inside an existing C project.

**5. No hidden control flow and no hidden allocation.**

No constructors, no destructors, no operator overloading, no exceptions, no
`Drop`, no GC. If you read `a + b` it is an add. If you read `foo()` the only
things that happen are in `foo`. For reasoning about a hot path or an embedded
target, this is worth a lot — and it's the main reason people describe Zig as
"readable" relative to modern C++ or heavy Rust.

**6. Error handling that is hard to ignore.**

`try` propagates, error unions are part of the type, an unhandled error is a
compile error, and `errdefer` exists specifically for the "allocated, then
failed" case. Compared to Go's `if err != nil` boilerplate that you can silently
drop with `_`, Zig's is both terser and stricter. Compared to exceptions, there
is no invisible unwinding path.

## Where Zig genuinely loses

Say these parts out loud in an interview. It's the strongest signal you can give.

**1. No borrow checker — use-after-free is on you.**

Zig's safe modes catch bounds errors and integer overflow. They **do not catch
use-after-free, double-free, or a dangling slice into a reused buffer.** 15.5
showed a parsed `"alpha"` silently becoming `"ZZZZZ"` with no crash and no
warning. `DebugAllocator` catches many of these in development — and 15.7 showed
that same leak detection silently returning `ok` in ReleaseFast.

**Rust is categorically safer here, and it is not close.** If the requirement is
"this memory bug class cannot ship," Rust is the right answer and you should say
so.

**2. No data-race protection.**

Rust's `Send`/`Sync` make a data race a compile error. Zig gives you `Mutex`,
`RwLock` and atomics, and trusts you. Go at least has a race detector you can
turn on in CI; Zig's story is "write it correctly."

**3. Pre-1.0, and it breaks every release.**

This course exists because 0.17 redesigned the standard library. `std.fs.File`
became `std.Io.File`; every writer needs an explicit `io` and a `flush`;
`ArrayList` became unmanaged; `std.time.Timer` was deleted; `**` was removed
from the language; `@Type` was replaced. **Code written for 0.13 does not
compile on 0.17**, and that was 4 releases, not 10 years.

The downstream effect is worse than the language churn: **every tutorial,
Stack Overflow answer and LLM completion you find is for an older version.**
You cannot learn Zig from search results right now, which is a real cost on a
team.

**4. The ecosystem is small and churning.**

There is no `tokio`, no `axum`, no `sqlx`, no `serde` — no library with that
level of maturity and maintenance guarantee. `std.http` exists but is minimal.
Most packages you find support one Zig version and go stale when it changes.
The package manager (`build.zig.zon`) is young and its hash/fetch semantics have
changed more than once.

For a backend service you will write more yourself than you would in Go or Rust,
and you will be the one who ports your dependencies forward.

**5. The concurrency story is brand new.**

`std.Io`'s `async`/`concurrent` landed recently and `std.Io.Evented` does not
even compile in shipped 0.17 — you use `Threaded`. Compare Go, where goroutines
and channels have been the main selling point since 2012 and the scheduler is
battle-tested at enormous scale. **If the job is "write a highly concurrent
network service this quarter," Go is the pragmatic answer.**

**6. Compile times are mid, not fast.**

Measured here: hello world is **~1.05 s in Debug and ~4.6 s in ReleaseFast**,
from a warm compiler cache. That's much better than Rust and much worse than Go.
Zig's self-hosted backend is improving this fast, but don't claim "fast
compiles" — Go people will know better.

**7. The job market is tiny.**

A handful of companies use Zig in production (Bun, TigerBeetle, Uber for build
tooling, some game and embedded shops). If your goal is *a systems job*, Zig is
a differentiator and a conversation starter, not a job board category. Being
strong in Zig *and* competent in C or Rust is a much more employable position
than Zig alone — and the skills transfer almost completely.

## Zig vs each one, in a sentence

| | Pick Zig over it when | Pick it over Zig when |
|---|---|---|
| **C** | you want C's model with error unions, `defer`, slices with lengths, `comptime` instead of macros, bounds checks, and a build system — essentially always, for new code | you need a frozen language, certified toolchains, or 40 years of libraries |
| **Rust** | you want explicit allocation, far simpler semantics, better C interop, and faster builds; or the borrow checker is fighting your data structure | memory safety or data-race freedom is a hard requirement; or you need a mature async ecosystem |
| **Go** | you need control over memory and layout, no GC pauses, small static binaries, or you're writing something latency-critical | you need to ship a concurrent service fast with a mature ecosystem and hire for it easily |
| **C++** | you want C++'s power without constructors, destructors, overloading, templates-as-a-second-language, and 40 years of defaults | you need the existing C++ ecosystem or the template metaprogramming depth |

## Answering "why Zig?" in an interview

The structure that works: **a concrete trade-off, a real example, and the
counter-argument.**

> "Zig's distinguishing feature is that allocation is explicit — a function's
> signature tells you whether it can allocate, and you can hand a subsystem a
> different allocator without touching its code. In practice that means an arena
> per request: the handler allocates freely, I call `deinit` once when the
> response is written, and that handler structurally cannot leak. I measured the
> arena at about 1.8x the cost of true zero-copy parsing, versus 4.2x for
> `dupe`/`free` per field, so it's also the pragmatic middle.
>
> I also use Zig as a C toolchain. Cross-compiling from my Mac to a static
> `x86_64-linux-musl` binary is one flag and under a second, and the output is
> 134 KB with no dependencies — so the container is `FROM scratch`. No Docker
> build, no cross toolchain.
>
> The honest counter-argument: Zig has no borrow checker, so use-after-free is
> still on me — `DebugAllocator` catches a lot of it in development, but that
> checking silently turns off in ReleaseFast, which I think is a real trap. And
> it's pre-1.0; 0.17 redesigned the whole I/O layer, so most of what you find
> online doesn't compile. If the requirement were 'this class of memory bug
> cannot ship,' I'd argue for Rust. If it were 'a highly concurrent service by
> next quarter,' I'd argue for Go."

Why this works:

| Element | Signal |
|---|---|
| names **one** differentiator, not five | you've thought about what matters |
| a concrete pattern (arena per request) | you've written real code, not a tutorial |
| numbers you measured yourself | you measure instead of repeating marketing |
| an operational detail (static binary, `FROM scratch`) | you think about shipping |
| volunteers the weaknesses | you're calibrated, not a fan |
| **recommends a competitor for a specific requirement** | the strongest signal in the whole answer |

### Three answers that lose the room

| Answer | Why it fails |
|---|---|
| "Zig is faster than C." | It isn't, meaningfully — same backend, same codegen. The interviewer stops believing the rest. |
| "Zig is memory safe." | It isn't. Anyone who uses Zig knows about use-after-free, and now they think you don't. |
| "Rust is too complicated / the borrow checker gets in the way." | Reads as "I couldn't learn Rust." Say what Zig gives you *instead*, not what Rust costs. |

### If they push back

**"Why not just use Rust?"** — "For a service where memory-bug classes must be
eliminated, I'd use Rust. I reach for Zig when I want explicit control of
*where* allocation happens, or heavy C interop, and I'm accepting responsibility
for lifetimes in exchange. It's a real trade, not a free win."

**"Isn't Zig too immature for production?"** — "For most teams, yes, and the
version churn is the concrete reason: 0.17 moved all of file I/O. The companies
shipping it — Bun, TigerBeetle — pin a version and own their dependencies. I'd
only propose it where we're already writing C or C++, where the interop and
cross-compilation pay for the instability."

**"What would you miss most coming from Go?"** — "Goroutines and the ecosystem.
`std.Io`'s async is new — `Evented` doesn't even compile in shipped 0.17 — and
there's no `tokio` or `axum` equivalent. For a concurrent service on a deadline,
Go is the better engineering call."

**The meta-point:** they are not testing whether you love Zig. They are testing
whether you choose tools by their trade-offs. An engineer who can recommend
*against* their favorite language is the one you can trust with an architecture
decision.

---

Next: [15.9 — Interview Prep](./09-interview-prep.md)
