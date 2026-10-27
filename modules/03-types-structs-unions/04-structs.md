# 3.4 — Structs

## Fields and defaults

```zig
const std = @import("std");

const Config = struct {
    host: []const u8,
    port: u16 = 8080,
    verbose: bool = false,
};

pub fn main() void {
    const a: Config = .{ .host = "localhost" };
    const b: Config = .{ .host = "0.0.0.0", .port = 3000 };
    const c = Config{ .host = "api", .port = 443, .verbose = true };

    std.debug.print("a = {any}\n", .{a});
    std.debug.print("b = {any}\n", .{b});
    std.debug.print("c = {any}\n", .{c});
    std.debug.print("a.port = {d}\n", .{a.port});
}
```

```
a = .{ .host = { 108, 111, 99, 97, 108, 104, 111, 115, 116 }, .port = 8080, .verbose = false }
b = .{ .host = { 48, 46, 48, 46, 48, 46, 48 }, .port = 3000, .verbose = false }
c = .{ .host = { 97, 112, 105 }, .port = 443, .verbose = true }
a.port = 8080
```

A field with a default may be omitted; one without **must** be given. This is a
deliberate error:

```zig
const std = @import("std");

const Config = struct {
    host: []const u8,
    port: u16 = 8080,
};

pub fn main() void {
    const a: Config = .{ .port = 99 };
    std.debug.print("{any}\n", .{a});
}
```

```
sc2.zig:9:24: error: missing struct field: host
    const a: Config = .{ .port = 99 };
                      ~^~~~~~~~~~~~~~
sc2.zig:3:16: note: struct declared here
const Config = struct {
```

| Form | When |
|---|---|
| `const a: Config = .{ ... }` | type is on the left → **preferred**, write `.{}` |
| `const c = Config{ ... }` | no type annotation, so name it |
| `.{}` with every field defaulted | `const d: Config = .{};` |

`{any}` dumped `host` as raw bytes because `[]const u8` *is* a byte slice — Zig
has no way to know you meant text. That is what a custom `format` fixes.

## Methods, `Self`, and `format`

Methods are just functions in the struct's namespace whose first parameter is the
struct. There is no `this`; the receiver is explicit.

```zig
const std = @import("std");

const Counter = struct {
    const Self = @This();

    count: u32 = 0,
    step: u32 = 1,

    pub fn init(step: u32) Self {
        return .{ .step = step };
    }

    pub fn value(self: Self) u32 {
        return self.count;
    }

    pub fn bump(self: *Self) void {
        self.count += self.step;
    }

    pub fn reset(self: *Self) void {
        self.count = 0;
    }

    pub fn format(self: Self, w: *std.Io.Writer) !void {
        try w.print("Counter({d} by {d})", .{ self.count, self.step });
    }
};

pub fn main() void {
    var c = Counter.init(5);
    std.debug.print("start   {f}\n", .{c});
    c.bump();
    c.bump();
    std.debug.print("bumped  {f}  value()={d}\n", .{ c, c.value() });
    c.reset();
    std.debug.print("reset   {f}\n", .{c});

    const d: Counter = .{};
    std.debug.print("default {f}\n", .{d});
}
```

```
start   Counter(0 by 5)
bumped  Counter(10 by 5)  value()=10
reset   Counter(0 by 5)
default Counter(0 by 1)
```

| Piece | Meaning |
|---|---|
| `const Self = @This();` | the enclosing type — avoids repeating the name |
| `fn init(...) Self` | plain function, called `Counter.init(5)` |
| `fn value(self: Self)` | takes a **copy**; cannot mutate |
| `fn bump(self: *Self)` | takes a pointer; **can** mutate |
| `c.bump()` | sugar for `Counter.bump(&c)` — `&` is implicit |
| `pub fn format(self, w)` | makes `{f}` work (0.17 signature: 2 params) |

`init` is a convention, not a language feature — nothing calls it for you, and
there are no constructors or destructors.

## Value vs pointer receiver

Forget the `*` and your mutation goes into a copy that is thrown away:

```zig
const std = @import("std");

const Box = struct {
    n: u32 = 0,

    fn bumpByValue(self: Box) void {
        var copy = self;
        copy.n += 1;
    }

    fn bumpByPointer(self: *Box) void {
        self.n += 1;
    }
};

pub fn main() void {
    var b: Box = .{};
    b.bumpByValue();
    std.debug.print("after bumpByValue:   n={d}\n", .{b.n});
    b.bumpByPointer();
    std.debug.print("after bumpByPointer: n={d}\n", .{b.n});

    const c: Box = .{};
    _ = c;
    std.debug.print("@sizeOf(Box)={d}\n", .{@sizeOf(Box)});
}
```

```
after bumpByValue:   n=0
after bumpByPointer: n=1
@sizeOf(Box)=4
```

And a `*Self` method on a `const` value is a compile error — deliberately:

```zig
const std = @import("std");

const Box = struct {
    n: u32 = 0,
    fn bump(self: *Box) void {
        self.n += 1;
    }
};

pub fn main() void {
    const b: Box = .{};
    b.bump();
    std.debug.print("{d}\n", .{b.n});
}
```

```
sc5.zig:12:6: error: expected type '*sc5.Box', found '*const sc5.Box'
    b.bump();
    ~^~~~~
sc5.zig:12:6: note: cast discards const qualifier
sc5.zig:5:19: note: parameter type declared here
    fn bump(self: *Box) void {
```

`const` means `const` all the way down. Change `const b` to `var b`.

## Structs are values

```zig
const std = @import("std");

const Point = struct { x: i32, y: i32 };

fn moveByValue(p: Point) void {
    var local = p;
    local.x = 999;
}

fn moveByPointer(p: *Point) void {
    p.x = 999;
}

pub fn main() void {
    var a: Point = .{ .x = 1, .y = 2 };
    const b = a;

    a.x = 10;
    std.debug.print("a={any} b={any}  (b was a copy)\n", .{ a, b });

    var c: Point = .{ .x = 1, .y = 2 };
    moveByValue(c);
    std.debug.print("after moveByValue:   {any}\n", .{c});
    moveByPointer(&c);
    std.debug.print("after moveByPointer: {any}\n", .{c});

    const p: Point = .{ .x = 1, .y = 2 };
    const q: Point = .{ .x = 1, .y = 2 };
    std.debug.print("std.meta.eql(p,q) = {}\n", .{std.meta.eql(p, q)});
    std.debug.print("std.meta.eql(p,a) = {}\n", .{std.meta.eql(p, a)});
}
```

```
a=.{ .x = 10, .y = 2 } b=.{ .x = 1, .y = 2 }  (b was a copy)
after moveByValue:   .{ .x = 1, .y = 2 }
after moveByPointer: .{ .x = 999, .y = 2 }
std.meta.eql(p,q) = true
std.meta.eql(p,a) = false
```

No references, no implicit sharing — unlike a JS object or a Python instance,
assignment copies. `==` is also rejected on structs:

```zig
const std = @import("std");

const Point = struct { x: i32, y: i32 };

pub fn main() void {
    const p: Point = .{ .x = 1, .y = 2 };
    const q: Point = .{ .x = 1, .y = 2 };
    std.debug.print("{}\n", .{p == q});
}
```

```
sc11.zig:8:33: error: operator == not allowed for type 'sc11.Point'
    std.debug.print("{}\n", .{p == q});
                              ~~^~~~
sc11.zig:3:15: note: struct declared here
```

`std.meta.eql` compares field by field. It is **not** deep: a `[]const u8` field
compares pointer + length, not bytes, so write your own `eql` when a struct holds
strings.

## Anonymous structs and tuples

```zig
const std = @import("std");

const Point = struct { x: i32, y: i32 };

fn length(p: Point) i32 {
    return p.x + p.y;
}

pub fn main() void {
    const anon = .{ .x = 1, .y = 2 };
    const tuple = .{ 1, "two", true };
    const typed: Point = .{ .x = 1, .y = 2 };

    std.debug.print("anon            = {any}\n", .{anon});
    std.debug.print("anon.x          = {d}\n", .{anon.x});
    std.debug.print("length(.{{...}})  = {d}\n", .{length(.{ .x = 1, .y = 2 })});
    std.debug.print("length(typed)   = {d}\n", .{length(typed)});
    std.debug.print("tuple           = {any}\n", .{tuple});
    std.debug.print("tuple[1]        = {s}\n", .{tuple[1]});
    std.debug.print("tuple.len       = {d}\n", .{tuple.len});

    inline for (tuple, 0..) |field, i| {
        std.debug.print("  [{d}] {s}\n", .{ i, @typeName(@TypeOf(field)) });
    }
}
```

```
anon            = .{ .x = 1, .y = 2 }
anon.x          = 1
length(.{...})  = 3
length(typed)   = 3
tuple           = .{ 1, { 116, 119, 111 }, true }
tuple[1]        = two
tuple.len       = 3
  [0] comptime_int
  [1] *const [3:0]u8
  [2] bool
```

- `.{ .x = 1 }` **written at the call site** coerces to any struct with matching
  fields. Bound to a `const` first, it becomes its own distinct type and no
  longer coerces — `length(anon)` would be an error.
- `.{ 1, "two", true }` with no field names is a **tuple**: a struct with fields
  named `0`, `1`, `2`. It has `.len`, indexes with `[i]`, and is heterogeneous.
- **This is what `std.debug.print`'s second argument is.** `.{ a, b }` is a
  tuple, which is why a single argument still needs the braces, and why
  `inline for` can walk it.

`inline for` (not plain `for`) is required because each element has a different
type — the loop is unrolled at compile time.

## Nested structs

```zig
const std = @import("std");

const Server = struct {
    const Limits = struct {
        max_conns: u32 = 100,
        timeout_ms: u32 = 5000,
    };

    name: []const u8,
    limits: Limits = .{},
    addr: struct { host: []const u8, port: u16 },

    pub fn describe(self: Server) void {
        std.debug.print("{s} on {s}:{d}, max {d} conns, {d}ms timeout\n", .{
            self.name,
            self.addr.host,
            self.addr.port,
            self.limits.max_conns,
            self.limits.timeout_ms,
        });
    }
};

pub fn main() void {
    var s: Server = .{
        .name = "api",
        .addr = .{ .host = "0.0.0.0", .port = 8080 },
    };
    s.describe();

    s.limits.max_conns = 10;
    s.addr.port = 9090;
    s.describe();

    const strict: Server = .{
        .name = "admin",
        .limits = .{ .max_conns = 2, .timeout_ms = 500 },
        .addr = .{ .host = "127.0.0.1", .port = 1 },
    };
    strict.describe();

    const l: Server.Limits = .{};
    std.debug.print("Server.Limits default = {any}\n", .{l});
}
```

```
api on 0.0.0.0:8080, max 100 conns, 5000ms timeout
api on 0.0.0.0:9090, max 10 conns, 5000ms timeout
admin on 127.0.0.1:1, max 2 conns, 500ms timeout
Server.Limits default = .{ .max_conns = 100, .timeout_ms = 5000 }
```

Three things at once: a **named** nested type reachable as `Server.Limits`, an
**inline anonymous** struct type as the `addr` field, and `= .{}` defaulting a
whole nested struct. Nested values are stored inline — one flat allocation, no
pointers.

## A struct as a namespace

A struct with no fields is Zig's module system. There is no `namespace` keyword
because `struct` already is one.

```zig
const std = @import("std");

const math2d = struct {
    pub const origin: Point = .{ .x = 0, .y = 0 };

    pub const Point = struct { x: f64, y: f64 };

    pub fn add(a: Point, b: Point) Point {
        return .{ .x = a.x + b.x, .y = a.y + b.y };
    }

    pub fn dot(a: Point, b: Point) f64 {
        return a.x * b.x + a.y * b.y;
    }
};

pub fn main() void {
    const p: math2d.Point = .{ .x = 3, .y = 4 };
    const q: math2d.Point = .{ .x = 1, .y = 2 };

    std.debug.print("add = {any}\n", .{math2d.add(p, q)});
    std.debug.print("dot = {d}\n", .{math2d.dot(p, q)});
    std.debug.print("origin = {any}\n", .{math2d.origin});
    std.debug.print("@sizeOf(math2d) = {d}\n", .{@sizeOf(math2d)});
}
```

```
add = .{ .x = 4, .y = 6 }
dot = 11
origin = .{ .x = 0, .y = 0 }
@sizeOf(math2d) = 0
```

**`@sizeOf(math2d) == 0`** — it is a pure compile-time namespace with no runtime
footprint. Note `origin` references `Point` before it is declared: declaration
order does not matter inside a container.

This is also literally what `@import` gives you. **Every Zig file is a struct**,
so `std` is a struct value and `std.mem` is a field of it. Declarations are
private unless marked `pub`.

## Layout: `@sizeOf`, `@offsetOf`, field ordering

**Zig reorders your fields.** This is the fact that surprises people coming from
C.

```zig
const std = @import("std");

const Auto = struct { a: u8, b: u32, c: u8 };
const Ext = extern struct { a: u8, b: u32, c: u8 };
const Packed = packed struct { a: u8, b: u32, c: u8 };

fn report(comptime T: type, label: []const u8) void {
    std.debug.print("{s:<8} size={d:<3} align={d:<3} offsets a={d} b={d} c={d}\n", .{
        label,
        @sizeOf(T),
        @alignOf(T),
        @offsetOf(T, "a"),
        @offsetOf(T, "b"),
        @offsetOf(T, "c"),
    });
}

pub fn main() void {
    report(Auto, "auto");
    report(Ext, "extern");
    report(Packed, "packed");
    std.debug.print("bitSizeOf(Packed) = {d}\n", .{@bitSizeOf(Packed)});
}
```

```
auto     size=8   align=4   offsets a=4 b=0 c=5
extern   size=12  align=4   offsets a=0 b=4 c=8
packed   size=8   align=8   offsets a=0 b=1 c=5
bitSizeOf(Packed) = 48
```

Read those offsets:

- **`auto`** put `b` at offset **0** and the two `u8`s after it. Same fields,
  **8 bytes instead of 12** — it packed the small fields into what would have
  been padding. The declaration order is a suggestion.
- **`extern`** keeps your exact order (`a=0, b=4, c=8`) with C's padding rules.
  12 bytes. This is required at any ABI boundary.
- **`packed`** removes padding between fields — `b` sits at byte offset 1,
  unaligned. 48 bits of real data. It is backed by an integer, so you can
  `@bitCast` it.

| Layout | Order | Padding | Use for |
|---|---|---|---|
| `struct` (default) | compiler's choice | inserted and exploited | **everything internal** |
| `extern struct` | exactly as declared | C ABI rules | C interop, mmap'd files, syscalls |
| `packed struct` | as declared, bit-packed | none | wire protocols, bitfields, registers |

**Never assume field order or offsets for a plain `struct`.** The moment bytes
cross a process, file or language boundary, say `extern` or `packed` and the
compiler will hold the layout still. (Module 15 goes deeper.)

## Operations table

| Operation | Code |
|---|---|
| declare | `const T = struct { a: i32, b: bool = false };` |
| self-reference | `const Self = @This();` |
| instantiate, typed context | `const x: T = .{ .a = 1 };` |
| instantiate, named | `const x = T{ .a = 1 };` |
| all defaults | `const x: T = .{};` |
| read a field | `x.a` |
| write a field | `x.a = 2;` (needs `var`) |
| value method | `fn f(self: T) ...` — read-only |
| mutating method | `fn f(self: *T) ...` |
| call | `x.f()` — `&x` is implicit |
| static function | `fn init(...) T` → `T.init(...)` |
| custom printing | `pub fn format(self: T, w: *std.Io.Writer) !void` → `{f}` |
| debug dump | `{any}` — needs no method |
| copy | `const y = x;` — element-wise |
| compare | `std.meta.eql(a, b)` (shallow; `==` is an error) |
| anonymous literal | `f(.{ .a = 1 })` — coerces at the call site |
| tuple | `.{ 1, "two", true }` → `.len`, `[i]` |
| walk a tuple | `inline for (tup) \|v\| ...` |
| nested type | `const Inner = struct {...};` → `T.Inner` |
| namespace | a field-less `struct` — `@sizeOf` is 0 |
| size / alignment | `@sizeOf(T)` / `@alignOf(T)` |
| field offset | `@offsetOf(T, "b")` |
| size in bits | `@bitSizeOf(T)` |
| field names | `@typeInfo(T).@"struct".field_names` |
| C-compatible layout | `extern struct { ... }` |
| bit-packed layout | `packed struct { ... }` |

---

Next: [3.5 — Pointers](./05-pointers.md)
