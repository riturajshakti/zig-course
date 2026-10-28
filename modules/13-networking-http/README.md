# 13 — Networking & HTTP

0.17 moved every socket under **`std.Io.net`** and rewrote **`std.http`** on top of
the `std.Io.Reader`/`std.Io.Writer` pair from module 8. There is no framework
here: no router, no middleware, no request context. You get a listener, a
stream, a head parser, and a `respond` call — the routing is a `switch` you
write yourself.

That is the point. Everything in these six lessons is a primitive you can see
the whole of, and module 14 assembles them into a real service.

| # | Lesson | What you learn |
|---|---|---|
| 13.1 | [Addresses & Sockets](./01-addresses.md) | `IpAddress` union, `parse`/`loopback`, `Socket.Mode`, UDP, DNS via `net.HostName` |
| 13.2 | [TCP Client](./02-tcp-client.md) | `connect`, stream reader/writer + `.interface`, **flush**, line and length framing |
| 13.3 | [TCP Server](./03-tcp-server.md) | `listen`, `accept`, `ListenOptions`, connection lifecycle, concurrent handlers |
| 13.4 | [HTTP Client](./04-http-client.md) | `std.http.Client`, `fetch`, headers, POST bodies, redirects, the keep-alive trap |
| 13.5 | [HTTP Server](./05-http-server.md) | `std.http.Server.init`, `receiveHead`, `respond`, `respondStreaming`, request bodies |
| 13.6 | [Serving JSON](./06-serving-json.md) | `Stringify.valueAlloc`, `parseFromSlice`, JSON errors, arena-per-request |

Every program is self-contained: the server runs under `io.async`, the client
runs on the main task, and the server serves a fixed number of requests and
returns. Copy a block, `zig run` it, and it finishes on its own.

Ports are hardcoded (39240 upward) because **`net.Socket` has no
`localAddress()`** in 0.17 — you cannot ask the kernel which ephemeral port it
gave you, so examples bind a port they chose.

---

Next: [14 — Capstone: A JSON HTTP Service](../14-capstone-service/)
