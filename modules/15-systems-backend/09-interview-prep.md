# 15.9 — Interview Prep

Real questions, with answers you could actually say out loud. Where an answer
is better shown than described, the code is runnable.

The format throughout: **the short answer** (what you say first), then the depth
to go into if they follow up.

---

## Q1. "Why does every allocating function take an allocator?"

**Short answer:** So that allocation is visible in the type system and
controllable by the caller. A function without an allocator parameter cannot
allocate — you can audit a whole module's memory behavior from its signatures.
And the caller chooses the *strategy*: an arena for per-request work, a fixed
buffer where there's no heap, a failing allocator in a test, all without
changing the function.

**If they push:** the alternative is a global allocator, which gives you one
strategy for the whole program and makes "can this fail with OOM?" unanswerable
without reading every line. Zig's `std.mem.Allocator` is 16 bytes — a data
pointer and a vtable pointer — so passing it costs two registers.

**The follow-up they usually ask:** *"Which allocator would you use where?"*

| Need | Allocator | Why |
|---|---|---|
| long-lived app state | `std.heap.DebugAllocator` in dev, `smp_allocator` in release | leak detection where it's useful, speed where it matters |
| **per-request work in a server** | `std.heap.ArenaAllocator`, `deinit` when the response is written | **handlers can't leak; no per-field frees** |
| no heap at all (embedded, hot path) | `std.heap.FixedBufferAllocator` over a stack array | allocation is a pointer bump; OOM is an error you handle |
| tests | `std.testing.allocator` | **a leak fails the test, in all four build modes** |
| raw pages | `std.heap.page_allocator` | what the others are built on |

---

## Q2. "What's the difference between `defer` and `errdefer`?"

**Short answer:** `defer` always runs on scope exit; `errdefer` runs only when
the function returns an error. The decision is one question: *on success, who
owns this memory?* If the function still owns it, `defer`. If it's been handed
to the caller, `errdefer`.

| On success, the memory is… | Use |
|---|---|
| freed here (scratch, temporaries) | `defer` |
| returned to the caller | `errdefer` |
| stored in a struct the caller owns | `errdefer` |

**The real question behind it** is the multi-step constructor:

```zig
fn open(gpa: std.mem.Allocator, host: []const u8, user: []const u8) !Conn {
    const h = try gpa.dupe(u8, host);
    errdefer gpa.free(h);          // covers: `user` dupe fails, or validation fails
    const u = try gpa.dupe(u8, user);
    errdefer gpa.free(u);
    if (user.len == 0) return error.NoUser;
    return .{ .host = h, .user = u };
}
```

Each `errdefer` covers exactly the allocation above it. Delete them and the
error path leaks both strings — the single most common real leak in Zig code.

**The detail that impresses:** the gap between "allocated" and "owned by a
container."

```zig
const s = try std.fmt.allocPrint(gpa, "item-{d}", .{i});
errdefer gpa.free(s);              // <- this one
try list.append(gpa, s);
```

`append` can fail *after* `s` exists but *before* the list owns it. A
one-allocation leak that only happens under memory pressure. Most candidates
miss it.

**0.17 note:** `errdefer |e| { ... }` **does not exist** — plain `errdefer`
only. Older tutorials show the capture form.

---

## Q3. "Slices vs pointers vs arrays — what's actually different?"

**Short answer:** An array is a value with a comptime-known length. A slice is
a `{ptr, len}` pair — 16 bytes, length known at runtime. A `*T` is one address.
A `[*]T` is an address with no length at all, which is what C gives you.

```zig
const std = @import("std");

pub fn main() !void {
    const arr = [_]i32{ 10, 20, 30, 40, 50 };

    const ptr: *const [5]i32 = &arr;
    const slice: []const i32 = &arr;
    const sub = arr[1..4];
    const many: [*]const i32 = &arr;

    std.debug.print("array  {s:<18} size={d}\n", .{ @typeName(@TypeOf(arr)), @sizeOf(@TypeOf(arr)) });
    std.debug.print("&array {s:<18} size={d}  len known at comptime\n", .{ @typeName(@TypeOf(ptr)), @sizeOf(@TypeOf(ptr)) });
    std.debug.print("slice  {s:<18} size={d}  ptr+len\n", .{ @typeName(@TypeOf(slice)), @sizeOf(@TypeOf(slice)) });
    std.debug.print("a[1..4]{s:<18} size={d}  <- NOT a slice!\n", .{ @typeName(@TypeOf(sub)), @sizeOf(@TypeOf(sub)) });
    std.debug.print("many   {s:<18} size={d}  no length at all\n", .{ @typeName(@TypeOf(many)), @sizeOf(@TypeOf(many)) });

    std.debug.print("\nslice.len = {d}, slice[0] = {d}\n", .{ slice.len, slice[0] });
    var end: usize = 4;
    end += @intFromBool(arr[0] == 999);
    const real_slice = arr[1..end];
    std.debug.print("runtime bounds -> {s}, len={d}\n", .{ @typeName(@TypeOf(real_slice)), real_slice.len });
}
```

```
array  [5]i32             size=20
&array *const [5]i32      size=8  len known at comptime
slice  []const i32        size=16  ptr+len
a[1..4]*const [3]i32      size=8  <- NOT a slice!
many   [*]const i32       size=8  no length at all

slice.len = 5, slice[0] = 10
runtime bounds -> []const i32, len=3
```

**The one that catches people:** `arr[1..4]` with **comptime-known** bounds is
`*const [3]i32`, a pointer-to-array — not a slice. With runtime bounds it's
`[]const i32`. Same syntax, two different types, and it shows up in confusing
`@TypeOf` output and error messages.

**Why it matters in an interview:** "a slice carries its length, so bounds
checks are possible and a C-style buffer-overflow is caught" is the whole
safety argument for slices, and it's why `[*]T` exists only for C interop.

---

## Q4. "What is `comptime` and when do you use it?"

**Short answer:** `comptime` runs ordinary Zig at compile time. Generics are
just functions that take a `type` and return a `type` — there's no separate
template language. Use it for generic containers, lookup tables you'd otherwise
generate with a script, and reflection-driven code like serialization.

```zig
const std = @import("std");

fn Queue(comptime T: type, comptime cap: usize) type {
    return struct {
        items: [cap]T = undefined,
        head: usize = 0,
        len: usize = 0,

        const Self = @This();

        fn push(q: *Self, v: T) error{Full}!void {
            if (q.len == cap) return error.Full;
            q.items[(q.head + q.len) % cap] = v;
            q.len += 1;
        }

        fn pop(q: *Self) ?T {
            if (q.len == 0) return null;
            const v = q.items[q.head];
            q.head = (q.head + 1) % cap;
            q.len -= 1;
            return v;
        }
    };
}

pub fn main() !void {
    var q: Queue(u32, 3) = .{};
    try q.push(1);
    try q.push(2);
    try q.push(3);
    std.debug.print("push x3, push again -> {!}\n", .{q.push(4)});
    std.debug.print("pop {?d} {?d} {?d} {?d}\n", .{ q.pop(), q.pop(), q.pop(), q.pop() });
    std.debug.print("Queue(u32,3) @sizeOf = {d}, Queue(u64,8) @sizeOf = {d} (no heap, no allocator)\n", .{
        @sizeOf(Queue(u32, 3)), @sizeOf(Queue(u64, 8)),
    });
    std.debug.print("distinct types? {}\n", .{Queue(u32, 3) != Queue(u32, 4)});

    comptime {
        var sum: u32 = 0;
        for (1..101) |i| sum += @intCast(i);
        if (sum != 5050) @compileError("math broke");
    }
    const table = comptime blk: {
        var t: [8]u32 = undefined;
        for (&t, 0..) |*e, i| e.* = @intCast(i * i);
        break :blk t;
    };
    std.debug.print("comptime table = {any}  (baked into the binary)\n", .{table});
}
```

```
push x3, push again -> error.Full
pop 1 2 3 null
Queue(u32,3) @sizeOf = 32, Queue(u64,8) @sizeOf = 80 (no heap, no allocator)
distinct types? true
comptime table = { 0, 1, 4, 9, 16, 25, 36, 49 }  (baked into the binary)
```

Three things to point at:

- **`Queue(u32, 3)` is a function call that returns a type.** The capacity is a
  comptime parameter, so the storage is inline — no allocator, no heap,
  `@sizeOf` is 32 bytes. `Queue(u32, 3) != Queue(u32, 4)` is `true`: each set of
  arguments produces a **genuinely distinct type**.
- **The `comptime` block with `@compileError`** is a compile-time assertion. If
  the arithmetic were wrong the build would fail, not the test suite.
- **`comptime blk: { ... }`** computes the table during compilation; the binary
  contains the finished array.

**The honest caveat to volunteer:** `comptime` has costs. Every distinct
instantiation generates separate code — `Queue(u32,3)` and `Queue(u32,4)` are
different types with duplicated machine code — and heavy `comptime` makes
compile times and compiler memory blow up. `MaxRSS` in `zig build --summary`
is where you notice.

---

## Q5. "How does error handling work, and how do you map it to HTTP?"

**Short answer:** Errors are values in an error union (`E!T`). `try` propagates,
`catch` handles, and an unhandled error is a compile error. Error sets are
`switch`-able, so mapping them to status codes is exhaustive and the compiler
tells you when you add a new one.

```zig
const std = @import("std");

const StoreError = error{ NotFound, Conflict, Invalid };
const Status = enum(u16) { ok = 200, bad_request = 400, not_found = 404, conflict = 409, internal = 500 };

fn toStatus(e: anyerror) Status {
    return switch (e) {
        error.NotFound => .not_found,
        error.Conflict => .conflict,
        error.Invalid => .bad_request,
        error.OutOfMemory => .internal,
        else => .internal,
    };
}

fn fetch(id: u32) StoreError!u32 {
    return switch (id) {
        0 => error.Invalid,
        7 => error.NotFound,
        9 => error.Conflict,
        else => id * 100,
    };
}

fn handle(arena: std.mem.Allocator, id: u32) ![]const u8 {
    const value = fetch(id) catch |e| {
        return std.fmt.allocPrint(arena, "{{\"error\":\"{t}\",\"status\":{d}}}", .{ e, @backingInt(toStatus(e)) });
    };
    return std.fmt.allocPrint(arena, "{{\"value\":{d},\"status\":200}}", .{value});
}

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer std.debug.print("\ngpa deinit: {t}\n", .{dbg.deinit()});
    const gpa = dbg.allocator();

    for ([_]u32{ 3, 0, 7, 9 }) |id| {
        var arena: std.heap.ArenaAllocator = .init(gpa);
        defer arena.deinit();
        std.debug.print("GET /items/{d} -> {s}\n", .{ id, try handle(arena.allocator(), id) });
    }
}
```

```
GET /items/3 -> {"value":300,"status":200}
GET /items/0 -> {"error":"Invalid","status":400}
GET /items/7 -> {"error":"NotFound","status":404}
GET /items/9 -> {"error":"Conflict","status":409}

gpa deinit: ok
```

This is the whole backend error-handling pattern in one program:

| | |
|---|---|
| **a named error set** (`StoreError`) | so `toStatus` can be exhaustive and a new variant breaks the build |
| **an arena per request**, `defer arena.deinit()` | the response strings need no individual frees; `gpa deinit: ok` proves it |
| **errors formatted with `{t}`** | prints the bare name (`NotFound`), not `error.NotFound` |
| `else => .internal` | the catch-all, because `anyerror` is an open set |

**Named vs inferred error sets** is the depth question. `fn f() !T` infers the
set from the body, which means it *grows silently* as you add code — and a
caller's exhaustive `switch` breaks in a future commit. Name the set on anything
public that callers will `switch` on; inferred is fine for internal helpers and
`main`.

**What `try` is not:** it isn't an exception. There's no unwinding, no stack
capture cost in release, no catch-anywhere. It's `if (result) |v| v else |e| return e`.

---

## Q6. "What changes between build modes?"

**Short answer:** Speed, size, and *behavior*. Integer overflow panics in Debug
and ReleaseSafe and silently wraps in ReleaseFast and ReleaseSmall. Bounds
checks disappear. And `DebugAllocator`'s leak detection silently turns off.

| | Debug / ReleaseSafe | ReleaseFast / ReleaseSmall |
|---|---|---|
| integer overflow | **panics** | **silently wraps** |
| out-of-bounds index | **panics** | undefined behavior |
| `unreachable` reached | panics | UB — can **hang** instead of crashing |
| wrong field of a bare `union` | **panics** | returns garbage |
| `DebugAllocator` leak detection | works | **silently reports `ok`** |
| `std.testing.allocator` | works | **works** (all four modes) |

Measured on this machine (hello world, install path, unstripped / `-fstrip`):

| Mode | Size | Stripped |
|---|---|---|
| Debug | 2,001,512 | 845,656 |
| ReleaseSafe | 468,912 | 206,880 |
| ReleaseFast | 395,072 | 190,192 |
| ReleaseSmall | 165,088 | 165,088 |

And on a bounds-check-heavy loop (30M random indexes, best of 5):

| Mode | Time | vs ReleaseFast |
|---|---|---|
| Debug | 92,305 us | ~18x slower |
| ReleaseSafe | 8,021 us | ~1.6x slower |
| ReleaseFast | 5,325 us | — |
| ReleaseSmall | 16,391 us | **~3.3x slower** |

**What to recommend:** "**ReleaseSafe by default for a service.**" Keeping bounds
and overflow checks for ~1.6x on a check-heavy loop — and *zero* cost on code
without checks — is a good trade in anything networked. Where a profile says a
specific function can't afford it, `@setRuntimeSafety(false)` turns checks off
**per function** instead of switching the whole program to ReleaseFast.

Two facts that show you've actually used this: **ReleaseSmall is not "fast but
smaller"** — it's 3.3x slower here. And **never quote a Debug benchmark**;
Debug being ~18x slower is why half the "Zig is slow" posts exist.

---

## Q7. "What's `std.Io` and why does 0.17 pass `io` everywhere?"

**Short answer:** `std.Io` is an interface — like `Allocator`, but for I/O. The
caller provides the implementation, so the same function body can run on a
thread pool, on an event loop, or against a mock in a test. It makes I/O
capability visible in the signature exactly the way `Allocator` makes
allocation visible.

```zig
pub fn main(init: std.process.Init) !void {
    const io = init.io;              // std.Io
    const gpa = init.gpa;            // std.mem.Allocator
    const arena = init.arena.allocator();  // init.arena is *ArenaAllocator
}
```

| Consequence | Detail |
|---|---|
| **writers are buffered** | `out.print(...)` then **`try out.flush()`** or you see nothing |
| **a function that does I/O says so** | it takes `io: std.Io` — same audit story as allocators |
| **timing needs `io`** | `std.time.Timer` is gone; `std.Io.Clock.Timestamp.now(io, .awake)` |
| **testable** | `std.Io.Writer.fixed(&buf)` needs no `io` at all |
| **one `write` syscall, not a thousand** | the buffering is the point; `strace -c` shows it |

**The honest part:** this is new and unfinished. `std.Io.Threaded` works;
**`std.Io.Evented` does not compile in shipped 0.17** (its vtable sets a field
`Io.VTable` no longer has). So "the portable async story" is real in design and
partial in practice. Saying that is better than pretending otherwise.

**The flush trap is worth volunteering**, because everyone hits it: writes are
buffered, so a program that prints and exits without `flush()` prints nothing.
`std.debug.print` is the exception — unbuffered, straight to stderr, no `io`.

---

## Q8. "Walk me through a leak you'd catch, and how."

This is a favorite because it's hands-on. The answer has three parts:

**1. Where leaks come from.** Five recurring shapes:

| Shape | Fix |
|---|---|
| an early `return` past an allocation | `errdefer` after each allocation |
| `deinit` frees the container but not its elements | loop freeing elements **first**, then `deinit` |
| a `HashMap` key that the map doesn't own | `getOrPut` + `dupe` the key |
| `toOwnedSlice` followed by `deinit` | after `toOwnedSlice` the list owns nothing — use `errdefer`, not `defer` |
| the gap between `allocPrint` and `append` | `errdefer gpa.free(s)` inside the loop |

**2. How you catch it.** `DebugAllocator` prints the **allocating line**:

```
error(DebugAllocator): memory address 0x104580000 leaked:
/path/f.zig:8:31: 0x104436baf in init (f)
        const n = try gpa.dupe(u8, name);
```

It names line 8 — the exact `dupe` whose cleanup is missing. For a double free,
`DebugAllocator(.{ .never_unmap = true, .retain_metadata = true })` gives three
stack traces (alloc site, first free, second free) instead of a segfault.

**3. The trap to mention.** Leak detection **silently disappears in
ReleaseFast/ReleaseSmall** — `deinit()` returns `.ok` on a real leak, because
`safety` defaults to `std.debug.runtime_safety`. So leak checking belongs in
**tests**, where `std.testing.allocator` keeps working in all four modes and a
leak *fails the test*.

That last point is the one that reads as production experience rather than
tutorial knowledge.

---

## Q9. "How would you structure a Zig HTTP service?"

**Short answer, as a sketch:**

```
main           — std.process.Init: io, gpa; read config; build the server
server loop    — accept; per connection: arena = ArenaAllocator.init(gpa)
handler        — fn handle(arena: Allocator, io: Io, req: Request) !Response
                 defer arena.deinit()  <- the handler cannot leak
store          — long-lived state on the gpa, behind a Mutex
errors         — named error set per layer; one errorToStatus switch at the edge
parsing        — zero-copy: slices into the read buffer, no allocator parameter
tests          — std.testing.allocator so a leak fails the build
release        — ReleaseSafe; ReleaseSmall + musl static for the container
```

The decisions worth defending:

| Decision | Reason |
|---|---|
| **arena per request, `gpa` for app state** | two lifetimes, two allocators; handlers never free individually |
| **handlers take `Allocator`, not the arena type** | testable with `testing.allocator` / `FixedBufferAllocator` |
| **parsing takes no allocator** | the signature proves it can't allocate or OOM (15.5) |
| **fixed header array, not an `ArrayList`** | `error.TooManyHeaders` is a free DoS defence |
| **one `errorToStatus` at the edge** | exhaustive `switch` on a named set; the compiler catches new variants |
| **shared state behind a `Mutex`** | Zig has no `Send`/`Sync`, so this is on you — say that |
| **ReleaseSafe in production** | ~1.6x worst case to keep memory-safety checks in a networked service |

---

## What to build for a portfolio

Three projects, in increasing order of what they prove. One done well beats
three half-finished.

| Project | Proves | Scope |
|---|---|---|
| **1. A binary format parser/writer** (PNG, WAV, ELF headers, a protocol frame) | `packed`/`extern` layout, endianness, zero-copy, length-checked parsing, error sets | a weekend; **start here** |
| **2. A JSON HTTP service** with arena-per-request, a mutexed store, tests | the whole backend pattern; the thing a hiring manager can read in 10 minutes | 1–2 weeks (module 14 is this) |
| **3. A measured data-structure or allocator** — a pool allocator, an SoA store, a hash map — with a benchmark harness and real numbers | you measure instead of guessing; the rarest signal | 1 week |

What makes any of them good:

| Do | Why it matters to a reviewer |
|---|---|
| **a README with real numbers you measured** | "5.8x on a one-field scan over 2M rows, ReleaseFast, best of 5" — and say the machine |
| **tests using `std.testing.allocator`** | leak-clean under test is a verifiable claim |
| **state the Zig version in the README** | shows you know 0.17 broke things; saves the reviewer a failed build |
| **`build.zig` with a `test` step** | `zig build test` has to just work |
| **error sets that are named and mapped** | the thing most Zig code online gets lazy about |
| **a cross-compile line in the README** | `-target x86_64-linux-musl` → 134 KB static binary is a memorable detail |
| **a "known limitations" section** | calibration, again |

| Don't | |
|---|---|
| a toy "learn Zig" exercise repo | shows a course, not judgement |
| benchmarks without a mode or a machine | reads as marketing |
| an incomplete rewrite of something big | reviewers read finished code |
| vendored dependencies you haven't read | you'll be asked about them |

---

## The honest state of the 0.17 ecosystem

Say this accurately and you'll be more credible than most candidates.

**The language is pre-1.0 and breaks every release.** Not "might break" —
*does*. Between 0.13 and 0.17:

| Changed | From → To |
|---|---|
| all file I/O | `std.fs.File` → **`std.Io.File`**, and every writer takes `io` + a buffer |
| writes | became **buffered — you must `flush()`** |
| `ArrayList` | managed → **unmanaged** (`.empty`, `append(gpa, x)`, `deinit(gpa)`) |
| the GPA | `GeneralPurposeAllocator` → **`std.heap.DebugAllocator`** |
| timing | `std.time.Timer` → **deleted**; clocks come from `io` |
| args | `std.process.argsAlloc` → `init.minimal.args.iterate()` |
| reflection | `@typeInfo(T).Struct.fields` → **parallel arrays** (`field_names` + `field_types`) |
| array repetition | **`**` removed from the language** — use `@splat` |
| `@Type` | removed → `@Int(.unsigned, bits)` |
| custom formatting | `format(self, fmt, opts, w)` → `format(self, w: *std.Io.Writer)` |
| `std.posix` | ~thinned to 54 fns; `open`/`close`/`write`/`socket`/`fork` **all gone** |

**The practical consequences, in order of how much they'll cost you:**

1. **Search results are wrong.** Stack Overflow, blog posts, YouTube, and LLM
   completions are all 0.13–0.15. You will write code that looks right and
   doesn't compile. **The stdlib source is the documentation** —
   `$(zig env | awk -F'"' '/std_dir/{print $2}')` — and grepping it is a skill
   worth naming in an interview.
2. **Libraries go stale fast.** Most packages support one Zig version. There is
   no `tokio`, no `serde`, no `sqlx` equivalent. Expect to write more yourself
   and to port your dependencies forward when you upgrade.
3. **The package manager is young.** `build.zig.zon` hash and fetch semantics
   have changed more than once.
4. **Some things in the stdlib don't work yet.** `std.Io.Evented` doesn't
   compile in shipped 0.17. `std.http` is minimal.

**How to talk about the instability** — this is a judgement question, so answer
it like one:

> "Pre-1.0 churn is the main reason I wouldn't propose Zig for most teams. 0.17
> moved all of file I/O, so nothing you find online compiles — you learn from
> the stdlib source instead, which is actually fine once you accept it, but it's
> a real onboarding cost.
>
> The teams shipping Zig in production — Bun, TigerBeetle — handle it the same
> way: pin an exact version, own your dependencies, and treat an upgrade as a
> scheduled project rather than a `bump`. That works, and it's the only thing
> that does right now.
>
> Where I'd actually argue for Zig is a codebase that's already C or C++, where
> `zig cc`, `@cImport` and one-flag cross-compilation pay for the instability
> immediately. For a greenfield service with a deadline, I'd say Go or Rust and
> mean it."

| What they're listening for | Not |
|---|---|
| you know *which* things broke, specifically | "it's still early days" |
| you have a mitigation (pin, vendor, read source) | "hopefully 1.0 soon" |
| you'd recommend against it in the wrong context | enthusiasm as a substitute for judgement |
| you treat the stdlib as the docs | waiting for better tutorials |

---

## Rapid-fire

Things you should be able to answer in one sentence.

| Q | A |
|---|---|
| `?T` vs `!T`? | `?T` for "absent is normal", `!T` for "the operation failed". |
| Why is `@sizeOf(?*T) == @sizeOf(*T)`? | Null-pointer optimization — `null` is address 0. `?u32` is 8, though. |
| Why does Zig reorder struct fields? | To minimize padding. Use `extern` when the layout must match C. |
| `extern struct` vs `packed struct`? | `extern` = C's layout and padding; `packed` = bit-exact, no padding, backed by an integer. |
| When is `@bitSizeOf` legal? | Ints, floats, bools, and `packed` structs — not plain or `extern` structs. |
| `@bitCast` vs `@intCast` vs `@truncate`? | Same bits / same value (panics if it won't fit) / drop the high bits. |
| Is `/` legal on signed ints? | **No** — `@divTrunc`/`@divFloor`/`@divCeil`, because Zig won't pick a rounding convention for you. |
| `union` vs `union(enum)`? | Always `union(enum)`. A bare union reads garbage in ReleaseFast where Debug panics. |
| Why does appending to an `ArrayList` invalidate pointers? | It may reallocate and copy. Hold indices, not `*T`. |
| Does a `HashMap` copy its keys? | **No.** A key from a reused buffer silently becomes unreachable. |
| `std.mem.bytesAsValue` vs `bytesToValue`? | View (aliases the buffer) vs copy. The `As`/`To` naming is the tell. |
| Does `split` or `tokenize` keep empty fields? | `split` keeps them, `tokenize` collapses them. |
| Page size? | **Don't hardcode 4096** — `std.heap.pageSize()`. It's 16384 on Apple silicon. |
| How do you unit-test writer code? | `std.Io.Writer.fixed(&buf)` — returns a `Writer` directly, needs no `io`. |
| How big is a slice? | 16 bytes: pointer + length. Same as `std.mem.Allocator`. |
| `expectEqual` on two `[]const u8`? | Compares **pointers** — use `expectEqualStrings`/`expectEqualSlices`. |
| Cross-compile to a static Linux binary? | `-target x86_64-linux-musl -O ReleaseSmall -fstrip` — ~134 KB, under a second, no toolchain. |

---

## The last thing

The single most valuable habit this course has tried to build is not a Zig
feature. It's this: **when you don't know, read the source and run the program.**

Every number in these nine lessons was measured on one machine, and every
program was executed before it was pasted. That's not diligence for its own
sake — it's the only thing that works in an ecosystem where the documentation is
a version behind and the answers online don't compile. An engineer who reaches
for `grep` in the stdlib and a 20-line benchmark instead of a search engine will
out-perform one who knows more Zig trivia, and that comes across in an
interview within about five minutes.

Go build the binary format parser. Measure something. Write down the number and
the machine you measured it on.

---

That's the end of the course. [Back to the module index](./README.md) ·
[Course index](../../README.md)
