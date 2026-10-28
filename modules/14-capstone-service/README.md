# Module 14 — Capstone: A JSON HTTP Service 🎯

One project, seven steps, nothing thrown away. By 14.7 you have a complete
in-memory item store that speaks JSON over HTTP, has unit **and** end-to-end
tests, reads its config from the environment, shuts down gracefully, and builds
to a 256 KB static binary.

This is the module you point an interviewer at. It is also where every earlier
module cashes out: the arena from 5.4, the writer interface from 8.2, the error
sets from 4.1, `StringHashMap` from 7.2, `std.json` from 7.6, `testing.allocator`
from 9.3, and `build.zig` from module 10.

**The API, in full:**

| Method | Path | Success | Failure |
|---|---|---|---|
| `GET` | `/health` | `200 {"status":"ok"}` | — |
| `GET` | `/items` | `200 [{...}, ...]` | — |
| `GET` | `/items/:id` | `200 {...}` | `404 {"error":"not_found"}` |
| `POST` | `/items` | `201 {...}` | `400 {"error":"bad_request"}` |
| `DELETE` | `/items/:id` | `204` (no body) | `404 {"error":"not_found"}` |

Any other path is `404`; a known path with the wrong method is
`405 {"error":"method_not_allowed"}`.

> **0.17.0 rewrote I/O, networking and `http`.** Every HTTP-server tutorial you
> can find online targets 0.13–0.15 and will not compile. The four that bite:
>
> | Tutorials say | Zig 0.17.0 |
> |---|---|
> | `std.net.Address.listen` | `std.Io.net.IpAddress` → `.listen(io, .{})` |
> | `http.Server.init(conn, &buf)` | `.init(&stream_reader.interface, &stream_writer.interface)` |
> | `std.Thread.Mutex` | `std.Io.Mutex` — `lock(io)` **can fail** (`error.Canceled`) |
> | `std.json.stringifyAlloc` | `std.json.Stringify.valueAlloc` |

## Lessons

| # | Lesson | What you get |
|---|---|---|
| 14.1 | [Design & Layout](./01-design.md) | the API spec, a one-file server that answers 3 requests and exits, the `build.zig` + `src/` layout, module boundaries, the request lifecycle, what we deliberately don't build |
| 14.2 | [Routing](./02-routing.md) | `route()` as a pure function returning a tagged union, path params as slices into the target, stripping the query string, 404 vs 405, 10 unit tests with no socket |
| 14.3 | [Per-Request Arena](./03-request-arena.md) | arena-per-request measured at **~290x** faster than freeing individually, `reset(.retain_capacity)` holding capacity flat across requests, and the one thing that must outlive the request |
| 14.4 | [Store & State](./04-store.md) | `StringHashMap(Item)` behind `std.Io.Mutex`, id generation, **why the key is gpa-owned and never arena-owned**, `errdefer` in a three-allocation constructor, the full CRUD surface |
| 14.5 | [Error Handling](./05-errors.md) | one `classify()` from `anyerror` → status + code, error sets per layer, six real responses including a `500` that leaks nothing, `errdefer` mid-request |
| 14.6 | [Config & Shutdown](./06-config-shutdown.md) | `init.environ_map` + `init.minimal.args`, validation at startup with `std.process.fatal`, `task.cancel(io)` to unblock a parked `accept`, `deinit` ordering, a verified `SIGTERM` handler |
| 14.7 | [Tests & Release](./07-tests-release.md) | handler tests with no network, an end-to-end test over loopback, `zig build test` → **15/15**, measured sizes for all four modes, a static Linux cross-build, and **the complete final service** |

Every file is shown in full with its path. 14.7 is the whole thing in one
place — copy those eight files into a directory and `zig build run` works.

## The interview answer

> It's a `std.http.Server` on a `std.Io.net` listener. Each request gets an
> arena that's reset when the response is written, so handlers allocate freely
> and never free — the parsed JSON, the response body, the formatted strings all
> die together. The store is the only long-lived state: it's gpa-owned behind a
> `std.Io.Mutex`, and it dupes anything it keeps, because an arena-owned key
> would dangle the moment the request ended. Handlers return Zig errors, and one
> `classify` function maps the error set onto a status and a stable JSON error
> code — unmatched errors become a logged `500` with `{"error":"internal"}`, so
> an internal error name can't reach a client. Shutdown sets an atomic flag and
> cancels the accept task, which makes the parked `accept` return
> `error.Canceled` after the in-flight request finishes.

---

Next: [Module 15 — Zig for Systems & Backend Work](../15-systems-backend/)
