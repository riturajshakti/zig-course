# Module 12 — Concurrency & Async

Zig had an `async`/`await` keyword pair. It was **removed**, and 0.17 replaced
it with something better: concurrency is a property of the **`io` value** you
pass in, not of a function's type.

```zig
fn work(io: std.Io, ms: i64) void { ... }   // no `async`, no Future, no colour

work(io, 100);                              // blocks
var f = io.async(work, .{ io, 100 });       // doesn't
f.await(io);
```

That one shift fixes the thing that makes async painful everywhere else:

| | Node / Python | Go | Zig 0.17 |
|---|---|---|---|
| spawn | `create_task(f())` | `go f()` | `io.async(f, .{args})` |
| join | `await t` | channel / `WaitGroup` | `fut.await(io)` |
| function colouring | **yes** — `async` infects every caller | no | **no** |
| cancellation | convention (`AbortController`) / `CancelledError` | `context.Context`, by hand | `error.Canceled` **in the error set** |
| swap the execution model | rewrite | no | pass a different `io` |

The second big idea: **cancellation is in the type system.** Any function that
can block on `io` returns `std.Io.Cancelable!T`, so the compiler tells you
where a cancellation can arrive. That is why `std.Thread.Mutex` is gone and
`std.Io.Mutex.lock(io)` needs a `try` — waiting for a lock is an `io`
operation, therefore interruptible.

> **Where this module's `io` comes from.** `std.Io.Threaded` backs every
> blocking operation with a thread pool, and it is what you get from
> `std.process.Init`. `std.Io.Evented` — green threads on one OS thread, which
> is where the design is heading — **does not compile in shipped 0.17.0**. Every
> program here uses `Threaded`, and none of them would change a line when
> `Evented` lands. That is the whole argument for explicit `io`.

## Lessons

| # | Lesson | What you get |
|---|---|---|
| 12.1 | [async & await](./01-async-await.md) | `io.async` → `Future(T)`, `await(io)`, sequential vs async **measured**, why there is no `async` keyword and no function colouring, returning values and errors, fan-out arrays, `std.Io.Group` |
| 12.2 | [concurrent](./02-concurrent.md) | `io.async` may run **inline** — proved with a timer; `io.concurrent` and `error.ConcurrencyUnavailable`; concurrency vs parallelism; ~4-5x on 12 cores; `async_limit`/`concurrent_limit` turning parallel into sequential with no code change |
| 12.3 | [Cancellation](./03-cancellation.md) | `Cancelable`, cancellation points, `future.cancel(io)`, `defer`/`errdefer` on the cancel path, `io.checkCancel` for CPU loops, `Io.Timeout`, racing a deadline with `Io.Select`, why `lock(io)` can fail, cooperative shutdown |
| 12.4 | [Threads](./04-threads.md) | `spawn`/`join`/`detach`, `SpawnConfig` and the 16 MiB stack, `getCpuCount` (12 here), `setName` takes `io` and **macOS only lets a thread name itself**, passing pointers safely, and the Debug-vs-release flip in spawn cost |
| 12.5 | [Mutexes & Locks](./05-mutexes.md) | `std.Io.Mutex` (and why `std.Thread.Mutex` is gone), `tryLock`, `RwLock` with overlapping readers, `Condition` producer/consumer, `Semaphore` as a concurrency cap, a real deadlock **escaped by cancelling one task**, and why lock striping needs `align(std.atomic.cache_line)` |
| 12.6 | [Atomics](./06-atomics.md) | every `std.atomic.Value(T)` method, the orderings table with a real compile error, `@fence` **removed in 0.17**, release/acquire publishing a plain `u64`, the CAS retry loop, and a benchmark where the shared atomic is **5x slower than a mutex** and 100x slower than the same atomics on private cache lines |
| 12.7 | [Concurrent Echo Server](./07-echo-server.md) | `listen`/`accept`/`connect`, one `io.concurrent` per connection, `.interface` as a field, per-connection arenas, graceful shutdown via `Group.cancel`, and 4 × 100 ms handlers finishing in 105 ms |

Every program is complete and runnable. Copy a block, `zig run` it, get the
output shown — but see the next section first.

## Two things about the output

**Interleaving is not deterministic.** Where a program prints from several
tasks, the order changes between runs. Each lesson says so and pastes one real
run; the programs whose numbers matter print **totals**, which are stable.

**Benchmark in release mode.** `Threaded`'s dispatch is heavily affected by
optimisation: in 12.4, `Group.async` is 4x *slower* than raw threads in Debug
and 4x *faster* under `-Ofast`. Lessons that measure say which mode they used.

## The interview answer

> Zig 0.17 has no `async` keyword. You pass an `io` — a 16-byte
> vtable, like an allocator — and `io.async(fn, args)` returns a `Future(T)`
> you `await(io)`. Because async-ness lives in the value rather than the
> function's type, there is no function colouring: one function body works
> blocking or concurrent, and the caller chooses by handing it a different
> `io`. `io.async` is best-effort and may run inline, so anything that must
> coexist uses `io.concurrent`, which fails loudly instead. Cancellation is
> `error.Canceled` in the error set of every blocking `io` call, which is why
> the sync primitives moved to `std.Io` and `Mutex.lock(io)` is fallible — and
> it means `defer` is all the cleanup machinery you need, even on cancel.

---

Next: [Module 13 — Networking & HTTP](../13-networking-http/)
