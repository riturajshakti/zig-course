# 15.4 — Data-Oriented Design

One idea: **the layout of your data is a performance decision, and usually a
bigger one than your algorithm.** Zig gives you `std.MultiArrayList` to make it
a one-line change.

## The budget

```zig
const std = @import("std");

const Entity = struct {
    id: u64,
    x: f32,
    y: f32,
    name: [32]u8,
    hp: u16,
    active: bool,
};

pub fn main() !void {
    std.debug.print("@sizeOf(Entity) = {d}\n", .{@sizeOf(Entity)});
    std.debug.print("cache line      = 64 B -> {d} entities per line\n", .{64 / @sizeOf(Entity)});
    std.debug.print("hot fields (hp) = {d} B -> {d} per line in SoA\n\n", .{ @sizeOf(u16), 64 / @sizeOf(u16) });

    const n = 1_000_000;
    std.debug.print("1M entities AoS = {d} B ({d:.1} MB)\n", .{ n * @sizeOf(Entity), @as(f64, @floatFromInt(n * @sizeOf(Entity))) / 1024.0 / 1024.0 });
    std.debug.print("1M hp only      = {d} B ({d:.1} MB)\n", .{ n * @sizeOf(u16), @as(f64, @floatFromInt(n * @sizeOf(u16))) / 1024.0 / 1024.0 });

    var mal: std.MultiArrayList(Entity) = .empty;
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();
    defer mal.deinit(gpa);
    try mal.append(gpa, .{ .id = 1, .x = 1.0, .y = 2.0, .name = @splat('a'), .hp = 100, .active = true });
    try mal.append(gpa, .{ .id = 2, .x = 3.0, .y = 4.0, .name = @splat('b'), .hp = 50, .active = false });

    std.debug.print("\nMultiArrayList len={d}\n", .{mal.len});
    std.debug.print("  items(.hp)     = {any}\n", .{mal.items(.hp)});
    std.debug.print("  items(.active) = {any}\n", .{mal.items(.active)});
    std.debug.print("  items(.x)      = {any}\n", .{mal.items(.x)});
    std.debug.print("  get(0).id      = {d}\n", .{mal.get(0).id});
    std.debug.print("  capacityInBytes({d}) = {d}\n", .{ mal.capacity, @TypeOf(mal).capacityInBytes(mal.capacity) });
}
```

```
@sizeOf(Entity) = 56
cache line      = 64 B -> 1 entities per line
hot fields (hp) = 2 B -> 32 per line in SoA

1M entities AoS = 56000000 B (53.4 MB)
1M hp only      = 2000000 B (1.9 MB)

MultiArrayList len=2
  items(.hp)     = { 100, 50 }
  items(.active) = { true, false }
  items(.x)      = { 1, 3 }
  get(0).id      = 1
  capacityInBytes(5) = 255
```

**1 entity per cache line versus 32 `hp` values per cache line.** That ratio is
the whole lesson. A loop that only reads `hp` over an array of `Entity` pulls 56
bytes from memory to use 2 — it wastes 96% of every cache line it touches.

| | AoS (`[]Entity`) | SoA (`MultiArrayList(Entity)`) |
|---|---|---|
| in memory | `[id,x,y,name,hp,act][id,x,y,…]…` | `[id,id,…][x,x,…]…[hp,hp,…][act,…]` |
| allocations | 1 | **1** (one block, sub-sliced) |
| read one field over all rows | strided, 1 useful value per line | **sequential, 32 per line** |
| read one whole row | 1 cache line | up to 6 cache lines |
| add a field | free | free |

`MultiArrayList` makes **one** allocation and carves the parallel arrays out of
it — `capacityInBytes(5) = 255`, not six separate blocks.

## The measurement

```zig
const std = @import("std");

const Entity = struct {
    id: u64,
    x: f32,
    y: f32,
    name: [32]u8,
    hp: u16,
    active: bool,
};

const N = 2_000_000;
const REPS = 5;

fn bench(io: std.Io, comptime label: []const u8, comptime f: fn (u64) u64) void {
    var best: u64 = std.math.maxInt(u64);
    var checksum: u64 = 0;
    for (0..REPS) |rep| {
        const t0 = std.Io.Clock.Timestamp.now(io, .awake);
        checksum = f(rep);
        const t1 = std.Io.Clock.Timestamp.now(io, .awake);
        std.mem.doNotOptimizeAway(checksum);
        best = @min(best, @as(u64, @intCast(t0.durationTo(t1).raw.toMicroseconds())));
    }
    std.debug.print("{s:<28} best of {d}: {d:>5} us   (sum={d})\n", .{ label, REPS, best, checksum });
}

var aos: []Entity = undefined;
var soa: std.MultiArrayList(Entity) = .empty;

fn sumAos(seed: u64) u64 {
    var total: u64 = seed;
    for (aos) |e| total += e.hp;
    return total;
}

fn sumSoa(seed: u64) u64 {
    var total: u64 = seed;
    for (soa.items(.hp)) |hp| total += hp;
    return total;
}

fn sumAosAll(seed: u64) u64 {
    var total: u64 = seed;
    for (aos) |e| total += e.hp + @as(u64, e.id) + @intFromBool(e.active);
    return total;
}

fn sumSoaAll(seed: u64) u64 {
    var total: u64 = seed;
    const hps = soa.items(.hp);
    const ids = soa.items(.id);
    const acts = soa.items(.active);
    for (hps, ids, acts) |hp, id, a| total += hp + id + @intFromBool(a);
    return total;
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const gpa = init.gpa;

    aos = try gpa.alloc(Entity, N);
    defer gpa.free(aos);
    try soa.setCapacity(gpa, N);
    defer soa.deinit(gpa);

    var rng = std.Random.DefaultPrng.init(0x5EED);
    const r = rng.random();
    for (aos) |*e| {
        e.* = .{ .id = r.int(u32), .x = 0, .y = 0, .name = @splat('x'), .hp = r.int(u16), .active = r.boolean() };
        soa.appendAssumeCapacity(e.*);
    }

    std.debug.print("N={d}  @sizeOf(Entity)={d}  AoS={d:.1} MB  SoA(.hp)={d:.1} MB\n\n", .{
        N,                                                        @sizeOf(Entity),
        @as(f64, @floatFromInt(N * @sizeOf(Entity))) / 1048576.0, @as(f64, @floatFromInt(N * 2)) / 1048576.0,
    });

    std.debug.print("-- one hot field (hp) --\n", .{});
    bench(io, "AoS: for (aos) |e| e.hp", sumAos);
    bench(io, "SoA: items(.hp)", sumSoa);
    std.debug.print("\n-- three fields of six --\n", .{});
    bench(io, "AoS: hp+id+active", sumAosAll);
    bench(io, "SoA: hp+id+active", sumSoaAll);
}
```

Built with **`zig run -Ofast`** (ReleaseFast), macOS arm64. Three consecutive
runs:

```
N=2000000  @sizeOf(Entity)=56  AoS=106.8 MB  SoA(.hp)=3.8 MB

-- one hot field (hp) --
AoS: for (aos) |e| e.hp      best of 5:  1062 us   (sum=65545601261)
SoA: items(.hp)              best of 5:   185 us   (sum=65545601261)

-- three fields of six --
AoS: hp+id+active            best of 5:  1557 us   (sum=4293281341130136)
SoA: hp+id+active            best of 5:  1503 us   (sum=4293281341130136)
```

| Workload | AoS | SoA | Ratio |
|---|---|---|---|
| **1 field of 6** | 1062 / 1086 / 1095 us | 185 / 184 / 184 us | **~5.8x** |
| **3 fields of 6** | 1557 / 1540 / 1559 us | 1503 / 1532 / 1486 us | **~1.0x — no win** |

Repeatable to within 3% across runs. **These numbers are from one machine** (M-series
arm64, 2M rows, ReleaseFast); the *ratio* is the transferable part, not the
microseconds, and it depends on your cache sizes and row width.

The honest conclusion, which is more useful than "SoA is faster":

- **SoA wins big when you touch a small fraction of each row.** 5.8x here, and
  that's bounded by `56/2 = 28` — you can't beat the bandwidth ratio.
- **SoA wins nothing when you touch most of the row.** Three fields of six is
  already break-even. Touch all six and SoA is *worse*, because you're now
  streaming six separate regions instead of one.
- **Neither is "the fast one."** Match the layout to the access pattern.

### Two benchmark details that matter

**`std.mem.doNotOptimizeAway(checksum)`** is load-bearing. Without it, and with
a function that ignores its argument, ReleaseFast noticed the loop was
invariant across reps and hoisted it out entirely — all four benchmarks reported
**0 us**. That's the single most common way a microbenchmark lies. The `seed`
parameter plus `doNotOptimizeAway` forces the work to actually happen.

**`best of 5`, not the mean.** The minimum is the run least disturbed by the
scheduler, other processes, and frequency scaling. Means and first-run numbers
measure your machine's mood.

## MultiArrayList operations

```zig
const std = @import("std");

const Row = struct { id: u32, score: f32, live: bool };

const ByScore = struct {
    scores: []const f32,
    pub fn lessThan(ctx: @This(), a: usize, b: usize) bool {
        return ctx.scores[a] < ctx.scores[b];
    }
};

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    var m: std.MultiArrayList(Row) = .empty;
    defer m.deinit(gpa);

    try m.append(gpa, .{ .id = 3, .score = 7.5, .live = true });
    try m.append(gpa, .{ .id = 1, .score = 2.5, .live = false });
    try m.append(gpa, .{ .id = 2, .score = 9.0, .live = true });
    std.debug.print("len={d} capacity={d}\n", .{ m.len, m.capacity });
    std.debug.print("ids    = {any}\n", .{m.items(.id)});
    std.debug.print("scores = {any}\n", .{m.items(.score)});
    std.debug.print("live   = {any}\n", .{m.items(.live)});

    const r = m.get(1);
    std.debug.print("\nget(1)       = id={d} score={d} live={}\n", .{ r.id, r.score, r.live });
    m.set(1, .{ .id = 99, .score = 0.5, .live = true });
    std.debug.print("after set(1) ids = {any}\n", .{m.items(.id)});

    m.items(.score)[0] = 1.0;
    std.debug.print("direct write scores = {any}\n", .{m.items(.score)});

    m.sort(ByScore{ .scores = m.items(.score) });
    std.debug.print("\nsorted by score:\n", .{});
    std.debug.print("  ids    = {any}\n", .{m.items(.id)});
    std.debug.print("  scores = {any}\n", .{m.items(.score)});

    m.swapRemove(0);
    std.debug.print("\nafter swapRemove(0) ids = {any} len={d}\n", .{ m.items(.id), m.len });
    const popped = m.pop();
    std.debug.print("pop() = id={d}  len now {d}\n", .{ popped.?.id, m.len });

    const s = m.slice();
    std.debug.print("\nslice().items(.id) = {any}\n", .{s.items(.id)});
    std.debug.print("one allocation holds all {d} fields: {d} bytes for cap {d}\n", .{
        @typeInfo(Row).@"struct".field_names.len,
        @TypeOf(m).capacityInBytes(m.capacity),
        m.capacity,
    });
}
```

```
len=3 capacity=33
ids    = { 3, 1, 2 }
scores = { 7.5, 2.5, 9 }
live   = { true, false, true }

get(1)       = id=1 score=2.5 live=false
after set(1) ids = { 3, 99, 2 }
direct write scores = { 1, 0.5, 9 }

sorted by score:
  ids    = { 99, 3, 2 }
  scores = { 0.5, 1, 9 }

after swapRemove(0) ids = { 2, 3 } len=2
pop() = id=3  len now 1

slice().items(.id) = { 2 }
one allocation holds all 3 fields: 297 bytes for cap 33
```

| Operation | Code | Note |
|---|---|---|
| create | `var m: std.MultiArrayList(T) = .empty;` | |
| create with room | `var m: … = try .initCapacity(gpa, n);` | |
| reserve | `try m.setCapacity(gpa, n)` / `ensureTotalCapacity` | |
| append | `try m.append(gpa, row)` | **splits** the struct across arrays |
| append, no alloc | `m.appendAssumeCapacity(row)` | |
| **one field, all rows** | `m.items(.hp)` → `[]u16` | **the reason to use this type** |
| whole row | `m.get(i)` → `T` | **reassembles** it — copies from N arrays |
| write whole row | `m.set(i, row)` | scatters to N arrays |
| write one field | `m.items(.hp)[i] = x` | the cheap path |
| length | `m.len` (a field, not `items.len`) | |
| remove fast | `m.swapRemove(i)` → `void` | returns nothing, unlike `ArrayList` |
| remove, keep order | `m.orderedRemove(i)` | O(n) per field |
| remove last | `m.pop()` → `?T` | |
| sort | `m.sort(ctx)` with `pub fn lessThan(ctx, a: usize, b: usize) bool` | **index**-based |
| detach the arrays | `m.toOwnedSlice()` → `Slice` | |
| cheap field views | `const s = m.slice(); s.items(.hp)` | hoist out of loops |
| bytes used | `@TypeOf(m).capacityInBytes(m.capacity)` | |
| free | `m.deinit(gpa)` | one `free` |

Four differences from `ArrayList` that will bite you:

- **`m.len` is a field, not `m.items.len`.** There is no single `items` slice.
- **`swapRemove` returns `void`**, not the removed element. `pop()` still
  returns `?T`.
- **The sort comparator takes indices**, not values — it has to permute every
  array in lockstep. Your context carries the slice you're sorting by, and
  **`lessThan` must be `pub`** or you get
  `error: 'lessThan' is not marked 'pub'` from inside `multi_array_list.zig`.
- **`m.items(.f)` recomputes the sub-slice** each call. Hoist it:
  `const s = m.slice();` then `s.items(.f)` in the loop.

## When layout beats cleverness

```
cost of a memory access, roughly, on a modern core:
  L1 hit        ~1 ns
  L2 hit        ~4 ns
  L3 hit        ~15 ns
  main memory   ~80 ns    <- 80x an L1 hit
```

A loop that misses cache on every iteration spends ~98% of its time waiting.
No algorithmic constant-factor tweak competes with removing the misses — which
is why the 5.8x above came from *zero* changes to the arithmetic.

| Situation | Reach for |
|---|---|
| iterate over one or two fields of many | **SoA** (`MultiArrayList`) |
| iterate over whole records | **AoS** (`[]T` / `ArrayList`) |
| record is ≤ a cache line and you use most of it | AoS, don't bother |
| filter then process (`active == true`) | SoA for the filter pass, or **sort/partition** so the hot rows are contiguous |
| random access by key | the layout barely matters; the hash and the pointer chase dominate |
| millions of rows, memory-bound | **shrink the row first** (15.2), then consider SoA |

The ordering of that last row is deliberate. Narrowing `u64`→`u32` on two
fields (15.2: 32→16 bytes) is a smaller code change than restructuring to SoA
and helps *every* access pattern, including the whole-row ones. Do that first.

### `@sizeOf` budgeting in a service

Numbers worth carrying in your head for a backend interview:

| Thing | Size |
|---|---|
| a slice (`[]const u8`) | **16 B** — ptr + len |
| `std.mem.Allocator` | **16 B** — ptr + vtable ptr |
| a cache line | 64 B |
| a page (this machine, arm64 macOS) | **16384 B** (x86-64 Linux: 4096) |
| `std.ArrayList(T)` | 24 B — ptr + len + capacity |
| `?*T` | 8 B (no tag — null is address 0) |

So an in-memory cache of 10M 56-byte rows is 560 MB before overhead, and the
same data at 16 bytes per row is 160 MB. That is the kind of arithmetic you
should be able to do out loud.

---

Next: [15.5 — Zero-Copy Parsing](./05-zero-copy.md)
