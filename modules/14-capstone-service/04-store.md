# 14.4 — Store & State

The only long-lived thing in the service. Everything it holds is **gpa-owned**,
because the arena from 14.3 is gone by the time the next request reads it.

## `src/store.zig`

```zig
const std = @import("std");

pub const Item = struct {
    id: []const u8,
    title: []const u8,
    done: bool = false,
};

pub const Store = struct {
    gpa: std.mem.Allocator,
    mutex: std.Io.Mutex = .init,
    items: std.StringHashMap(Item),
    next_id: u32 = 1,

    pub const Error = error{ NotFound, OutOfMemory, Canceled };

    pub fn init(gpa: std.mem.Allocator) Store {
        return .{ .gpa = gpa, .items = .init(gpa) };
    }

    pub fn deinit(s: *Store) void {
        var it = s.items.valueIterator();
        while (it.next()) |item| {
            s.gpa.free(item.id);
            s.gpa.free(item.title);
        }
        s.items.deinit();
    }

    pub fn create(s: *Store, io: std.Io, title: []const u8, done: bool) Error!Item {
        try s.mutex.lock(io);
        defer s.mutex.unlock(io);

        const id = try std.fmt.allocPrint(s.gpa, "{d}", .{s.next_id});
        errdefer s.gpa.free(id);
        const owned = try s.gpa.dupe(u8, title);
        errdefer s.gpa.free(owned);

        const item: Item = .{ .id = id, .title = owned, .done = done };
        try s.items.put(id, item);
        s.next_id += 1;
        return item;
    }

    pub fn get(s: *Store, io: std.Io, id: []const u8) Error!Item {
        try s.mutex.lock(io);
        defer s.mutex.unlock(io);
        return s.items.get(id) orelse error.NotFound;
    }

    pub fn remove(s: *Store, io: std.Io, id: []const u8) Error!void {
        try s.mutex.lock(io);
        defer s.mutex.unlock(io);
        const kv = s.items.fetchRemove(id) orelse return error.NotFound;
        s.gpa.free(kv.value.id);
        s.gpa.free(kv.value.title);
    }

    pub fn list(s: *Store, io: std.Io, arena: std.mem.Allocator) Error![]Item {
        try s.mutex.lock(io);
        defer s.mutex.unlock(io);

        const out = try arena.alloc(Item, s.items.count());
        var i: usize = 0;
        var it = s.items.valueIterator();
        while (it.next()) |item| : (i += 1) out[i] = item.*;
        std.mem.sort(Item, out, {}, byId);
        return out;
    }

    fn byId(_: void, a: Item, b: Item) bool {
        if (a.id.len != b.id.len) return a.id.len < b.id.len;
        return std.mem.lessThan(u8, a.id, b.id);
    }
};
```

Add `main` and it runs:

```zig
pub fn main(init: std.process.Init) !void {
    const io = init.io;
    var gpa_s: std.heap.DebugAllocator(.{}) = .init;
    defer {
        const status = gpa_s.deinit();
        std.debug.print("gpa deinit: {t}\n", .{status});
    }
    const gpa = gpa_s.allocator();

    var store: Store = .init(gpa);
    defer store.deinit();

    var arena_s: std.heap.ArenaAllocator = .init(gpa);
    defer arena_s.deinit();

    _ = try store.create(io, "write resume", false);
    const second = try store.create(io, "ship the service", true);
    _ = try store.create(io, "practice interviews", false);

    for (try store.list(io, arena_s.allocator())) |item|
        std.debug.print("{s} {s} done={}\n", .{ item.id, item.title, item.done });

    std.debug.print("get 2   -> {s}\n", .{(try store.get(io, second.id)).title});
    if (store.get(io, "99")) |it| {
        std.debug.print("get 99  -> {s}\n", .{it.title});
    } else |e| std.debug.print("get 99  -> error.{t}\n", .{e});
    try store.remove(io, "1");
    std.debug.print("after remove 1, count={d}\n", .{store.items.count()});
    store.remove(io, "1") catch |e| std.debug.print("remove 1 again -> error.{t}\n", .{e});
}
```

```
1 write resume done=false
2 ship the service done=true
3 practice interviews done=false
get 2   -> ship the service
get 99  -> error.NotFound
after remove 1, count=2
remove 1 again -> error.NotFound
gpa deinit: ok
```

`gpa deinit: ok` is the point of the whole file.

## Key & value lifetimes

| Field | Allocator | Freed by |
|---|---|---|
| `items` map backing | gpa (map is **managed**: `.init(gpa)`) | `items.deinit()` |
| `Item.id` — **also the map key** | gpa, `allocPrint` | `remove`, or `deinit`'s loop |
| `Item.title` | gpa, `dupe(title)` | same |
| `title` as it arrives | **arena** (parsed out of the request body) | the arena reset — which is why `dupe` is not optional |
| the `[]Item` from `list` | **arena**, passed in by the caller | the arena reset |

Two things to stare at.

**`try s.items.put(id, item)` stores `id` as both key and value field.** One
allocation, one `free` per item. `StringHashMap` does not copy keys, so that
slice *is* the key — if it were the arena-owned `parsed.value.title`'s sibling,
14.3's "3 puts, 2 entries" failure is what you'd get.

**`list` takes the arena, `create` takes nothing.** `list` returns a snapshot
that only has to survive until the response is written, so it goes in the
request's arena. That is also what makes it safe to drop the lock before
`respond` runs: the caller holds copies of the structs, not pointers into the
map, so a later `create` cannot rehash under it. The `[]const u8` fields still
point at gpa memory the map owns — fine here, because nothing deletes while one
request is being served. The moment you serve concurrently, `list` must `dupe`
the strings into the arena too.

## `std.Io.Mutex`, not `std.Thread.Mutex`

`std.Thread.Mutex` **does not exist** in 0.17. The replacement takes `io` and
its `lock` is a cancelation point:

| Call | Signature | Note |
|---|---|---|
| `var m: std.Io.Mutex = .init;` | — | a field default, no `init(gpa)` |
| `try m.lock(io)` | `Cancelable!void` | **fallible** — a shutdown request makes it return `error.Canceled` |
| `m.lockUncancelable(io)` | `void` | when you must not be interrupted mid-critical-section |
| `m.tryLock()` | `bool` | no `io` |
| `m.unlock(io)` | `void` | takes `io` to wake a waiter |

That `try` is why `Store.Error` has a third member:

```zig
pub const Error = error{ NotFound, OutOfMemory, Canceled };
```

Leave `Canceled` out and you get a very clear compile error:

```
error: expected type 'error{NotFound,OutOfMemory}!store.Item', found 'error{Canceled}'
note: 'error.Canceled' not a member of destination error set
```

The alternative is an inferred error set (`!Item`), which compiles immediately
but stops documenting what callers must handle — and 14.5 maps this exact set
onto HTTP statuses, so it is worth writing out.

## `errdefer` in `create`

```zig
const id = try std.fmt.allocPrint(s.gpa, "{d}", .{s.next_id});
errdefer s.gpa.free(id);
const owned = try s.gpa.dupe(u8, title);
errdefer s.gpa.free(owned);
const item: Item = .{ .id = id, .title = owned, .done = done };
try s.items.put(id, item);
```

Three fallible steps, each able to fail after the previous one succeeded:

| If this fails | Already allocated | `errdefer`s that run |
|---|---|---|
| `allocPrint` | nothing | none |
| `dupe` | `id` | `free(id)` |
| `put` | `id`, `owned` | `free(owned)`, then `free(id)` |

LIFO, so the order is automatic. On success neither runs and both are owned by
the map. Note `errdefer` in 0.17 has **no capture form** — `errdefer |e| {...}`
is a syntax error, plain `errdefer` only.

## Sorting ids that are numbers in a string

`list` sorts so the response order is stable. Plain `lessThan` is wrong:

```zig
const std = @import("std");

fn plain(_: void, a: []const u8, b: []const u8) bool {
    return std.mem.lessThan(u8, a, b);
}

fn lenThenBytes(_: void, a: []const u8, b: []const u8) bool {
    if (a.len != b.len) return a.len < b.len;
    return std.mem.lessThan(u8, a, b);
}

pub fn main() !void {
    const src = [_][]const u8{ "1", "10", "2", "3", "11", "20" };

    var a = src;
    std.mem.sort([]const u8, &a, {}, plain);
    std.debug.print("lessThan only : ", .{});
    for (a) |s| std.debug.print("{s} ", .{s});

    var b = src;
    std.mem.sort([]const u8, &b, {}, lenThenBytes);
    std.debug.print("\nlen then bytes: ", .{});
    for (b) |s| std.debug.print("{s} ", .{s});
    std.debug.print("\n", .{});
}
```

```
lessThan only : 1 10 11 2 20 3 
len then bytes: 1 2 3 10 11 20 
```

Length-first works because these ids are unpadded decimals. It is a stopgap: a
real service either keeps `id: u32` in the struct and formats it only at the
JSON boundary, or uses ids whose lexical order *is* their order (ULIDs,
zero-padded counters).

## Id generation

`next_id: u32` behind the same mutex, formatted to a string once at creation.

| Property | This store | Why it's fine here |
|---|---|---|
| monotonic | yes | ids never collide, even after deletes |
| reused after delete | **no** — `remove` doesn't decrement | deleting `1` then creating gives `2` |
| guessable | yes | no authorization in this service (14.1) |
| survives restart | no | in-memory by design |

Swapping in a UUID is a two-line change — `io.random(&bytes)` then hex-encode
into the gpa — and nothing else in the service has to know, because `id` is
already an opaque `[]const u8` everywhere.

---

Next: [14.5 — Error Handling](./05-errors.md)
