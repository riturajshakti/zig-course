# 7.6 — JSON

Two directions, two names:

| Direction | Call |
|---|---|
| JSON text → your struct | `std.json.parseFromSlice(T, gpa, text, .{})` |
| your struct → JSON text | `std.json.Stringify.valueAlloc(gpa, v, .{})` |

> **`std.json.stringifyAlloc` is gone.** Encoding moved into a
> `std.json.Stringify` namespace in 0.17. Every tutorial you find online calls
> the old name.

## Operations table

| Operation | Code |
|---|---|
| parse to a struct | `try std.json.parseFromSlice(T, gpa, text, .{})` → `Parsed(T)` |
| the value | `parsed.value` |
| **free everything** | `parsed.deinit()` |
| parse into an arena | `try std.json.parseFromSliceLeaky(T, arena, text, .{})` → `T` |
| parse dynamically | `try std.json.parseFromSlice(std.json.Value, gpa, text, .{})` |
| `Value` → a struct | `try std.json.parseFromValue(T, gpa, value, .{})` |
| allow extra JSON fields | `.{ .ignore_unknown_fields = true }` |
| last duplicate wins | `.{ .duplicate_field_behavior = .use_last }` |
| validate only | `std.json.validate(gpa, text)` |
| encode to a new string | `try std.json.Stringify.valueAlloc(gpa, v, .{})` |
| encode to a writer | `try std.json.Stringify.value(v, .{}, writer)` |
| encode in a format string | `{f}` on `std.json.fmt(v, .{})` |
| pretty-print | `.{ .whitespace = .indent_2 }` |
| omit `null` fields | `.{ .emit_null_optional_fields = false }` |
| build by hand | `var s: std.json.Stringify = .{ .writer = w, .options = .{} };` |

## Parsing to a struct

Field names in the struct must match the JSON keys. Types are checked.

```zig
const std = @import("std");

const User = struct {
    name: []const u8,
    age: u32,
    active: bool,
};

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    const text =
        \\{"name":"ada","age":36,"active":true}
    ;

    const parsed = try std.json.parseFromSlice(User, gpa, text, .{});
    defer parsed.deinit();

    const u = parsed.value;
    std.debug.print("name   = {s}\n", .{u.name});
    std.debug.print("age    = {d}\n", .{u.age});
    std.debug.print("active = {}\n", .{u.active});
}
```

```
name   = ada
age    = 36
active = true
```

**`parseFromSlice` does not return a `User`** — it returns a
`std.json.Parsed(User)`, a wrapper whose `.value` is your `User` and which also
owns the arena holding every string inside it. That is the whole lifetime story, and the next section is
about getting it wrong.

## ⚠️ `parsed.deinit()` frees the strings too

`parsed.value.name` is a slice into memory owned by `parsed`. Returning it and
then calling `deinit` leaves a dangling slice — and it will often *look* like it
worked.

```zig
const std = @import("std");

const User = struct { name: []const u8, age: u32 };

fn borrowed(gpa: std.mem.Allocator, text: []const u8) ![]const u8 {
    const parsed = try std.json.parseFromSlice(User, gpa, text, .{});
    defer parsed.deinit();
    return parsed.value.name;
}

fn owned(gpa: std.mem.Allocator, text: []const u8) ![]const u8 {
    const parsed = try std.json.parseFromSlice(User, gpa, text, .{});
    defer parsed.deinit();
    return gpa.dupe(u8, parsed.value.name);
}

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    const text = "{\"name\":\"ada\",\"age\":36}";

    const good = try owned(gpa, text);
    defer gpa.free(good);
    std.debug.print("owned (correct): \"{s}\"\n", .{good});

    const bad = try borrowed(gpa, text);
    std.debug.print("borrowed length: {d} (points into freed memory)\n", .{bad.len});
}
```

```
owned (correct): "ada"
borrowed length: 3 (points into freed memory)
```

`borrowed` is a **use-after-free**. Printing those bytes on this machine still
prints `ada`, because the freed pages haven't been reused yet — which is exactly
what makes this bug survive code review and then fail in production under load.

Three ways to hold a parse result correctly:

| Pattern | Code |
|---|---|
| **keep `Parsed` alive as long as the data** | store `parsed`, call `deinit` when done with the values |
| **copy out what you need** | `try gpa.dupe(u8, parsed.value.name)` — you free it |
| **parse into an arena you already own** | `parseFromSliceLeaky(T, arena, …)` — returns `T` directly |

```zig
const std = @import("std");

const User = struct { name: []const u8, age: u32 };

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    var arena: std.heap.ArenaAllocator = .init(gpa);
    defer arena.deinit();
    const a = arena.allocator();

    const u = try std.json.parseFromSliceLeaky(User, a, "{\"name\":\"grace\",\"age\":45}", .{});
    std.debug.print("leaky + arena: {s} {d}\n", .{ u.name, u.age });

    var buf: [128]u8 = undefined;
    var w: std.Io.Writer = .fixed(&buf);
    try std.json.Stringify.value(u, .{}, &w);
    std.debug.print("to a Writer:   {s}\n", .{w.buffered()});

    var buf2: [128]u8 = undefined;
    var w2: std.Io.Writer = .fixed(&buf2);
    var s: std.json.Stringify = .{ .writer = &w2, .options = .{} };
    try s.beginObject();
    try s.objectField("manual");
    try s.write(true);
    try s.objectField("items");
    try s.beginArray();
    try s.write(1);
    try s.write("two");
    try s.endArray();
    try s.endObject();
    std.debug.print("streaming API: {s}\n", .{w2.buffered()});
}
```

```
leaky + arena: grace 45
to a Writer:   {"name":"grace","age":45}
streaming API: {"manual":true,"items":[1,"two"]}
```

**"Leaky" is not a warning — it's the name of the contract.** The function
allocates and never frees; you hand it an arena and drop the whole arena later.
This is *the* pattern for a server handling a request: arena per request, parse
leaky, reply, `arena.deinit()`.

## Nested structs, arrays, optionals, defaults

```zig
const std = @import("std");

const Addr = struct {
    city: []const u8,
    zip: []const u8,
};

const Person = struct {
    name: []const u8,
    age: u32,
    email: ?[]const u8 = null,
    tags: []const []const u8 = &.{},
    addr: Addr,
    score: f64 = 0,
};

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    const text =
        \\{
        \\  "name": "ada",
        \\  "age": 36,
        \\  "tags": ["math", "code"],
        \\  "addr": { "city": "London", "zip": "NW1" },
        \\  "score": 99.5
        \\}
    ;

    const parsed = try std.json.parseFromSlice(Person, gpa, text, .{});
    defer parsed.deinit();
    const p = parsed.value;

    std.debug.print("name    = {s}\n", .{p.name});
    std.debug.print("age     = {d}\n", .{p.age});
    std.debug.print("email   = {?s} (absent -> default)\n", .{p.email});
    std.debug.print("score   = {d}\n", .{p.score});
    std.debug.print("city    = {s}\n", .{p.addr.city});
    std.debug.print("zip     = {s}\n", .{p.addr.zip});
    std.debug.print("tags    = {d} items:", .{p.tags.len});
    for (p.tags) |t| std.debug.print(" {s}", .{t});
    std.debug.print("\n", .{});
}
```

```
name    = ada
age     = 36
email   = null (absent -> default)
score   = 99.5
city    = London
zip     = NW1
tags    = 2 items: math code
```

| JSON | Zig field type |
|---|---|
| `"text"` | `[]const u8` |
| `42` | `u32`, `i64`, … (range-checked) |
| `1.5` | `f64` / `f32` |
| `true` | `bool` |
| `null` **or absent** | `?T` |
| `["a","b"]` | `[]const []const u8` |
| `[{…},{…}]` | `[]const Item` |
| `{…}` | a nested struct |
| may be absent | give the field a **default** |

**A field with a default is optional in the JSON; one without is required.** `?T`
accepts an explicit `null`; `?T = null` also accepts the key being absent
entirely. That distinction is the whole schema-validation story.

## When the JSON doesn't match

```zig
const std = @import("std");

const Small = struct { name: []const u8 };

fn try_(gpa: std.mem.Allocator, label: []const u8, text: []const u8, opts: std.json.ParseOptions) void {
    const r = std.json.parseFromSlice(Small, gpa, text, opts);
    if (r) |parsed| {
        defer parsed.deinit();
        std.debug.print("{s:<34} ok -> {s}\n", .{ label, parsed.value.name });
    } else |e| {
        std.debug.print("{s:<34} {t}\n", .{ label, e });
    }
}

pub fn main() void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    try_(gpa, "extra field, default opts", "{\"name\":\"a\",\"extra\":1}", .{});
    try_(gpa, "extra field, ignore_unknown", "{\"name\":\"a\",\"extra\":1}", .{ .ignore_unknown_fields = true });
    try_(gpa, "missing required field", "{}", .{});
    try_(gpa, "wrong type", "{\"name\":42}", .{});
    try_(gpa, "malformed json", "{name:", .{});
    try_(gpa, "duplicate field (default)", "{\"name\":\"a\",\"name\":\"b\"}", .{});
    try_(gpa, "duplicate, .use_last", "{\"name\":\"a\",\"name\":\"b\"}", .{ .duplicate_field_behavior = .use_last });
    try_(gpa, "duplicate, .@\"error\"", "{\"name\":\"a\",\"name\":\"b\"}", .{ .duplicate_field_behavior = .@"error" });
}
```

```
extra field, default opts          UnknownField
extra field, ignore_unknown        ok -> a
missing required field             MissingField
wrong type                         UnexpectedToken
malformed json                     SyntaxError
duplicate field (default)          DuplicateField
duplicate, .use_last               ok -> b
duplicate, .@"error"               DuplicateField
```

**By default an unknown field is an error.** That is strict compared with most
languages, and it is deliberate: silently dropping a field you didn't model is
how you ship a bug. When you're consuming somebody else's API — which usually
adds fields over time — set `.ignore_unknown_fields = true` explicitly.

| Error | Cause |
|---|---|
| `UnknownField` | a JSON key your struct doesn't declare |
| `MissingField` | a required field (no default) absent |
| `UnexpectedToken` | right shape, wrong type |
| `SyntaxError` | not valid JSON |
| `DuplicateField` | the same key twice |
| `Overflow` | number too big for the field's integer type |

## Encoding

```zig
const std = @import("std");

const Cfg = struct {
    host: []const u8,
    port: u16,
    debug: bool,
    retries: ?u8,
    tags: []const []const u8,
};

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    const cfg = Cfg{
        .host = "localhost",
        .port = 8080,
        .debug = true,
        .retries = null,
        .tags = &.{ "a", "b" },
    };

    const min = try std.json.Stringify.valueAlloc(gpa, cfg, .{});
    defer gpa.free(min);
    std.debug.print("minified:\n{s}\n\n", .{min});

    const pretty = try std.json.Stringify.valueAlloc(gpa, cfg, .{ .whitespace = .indent_2 });
    defer gpa.free(pretty);
    std.debug.print("indent_2:\n{s}\n\n", .{pretty});

    const skip = try std.json.Stringify.valueAlloc(gpa, cfg, .{ .emit_null_optional_fields = false });
    defer gpa.free(skip);
    std.debug.print("no null fields:\n{s}\n\n", .{skip});

    const arr = try std.json.Stringify.valueAlloc(gpa, [_]u32{ 1, 2, 3 }, .{});
    defer gpa.free(arr);
    std.debug.print("array:   {s}\n", .{arr});

    const num = try std.json.Stringify.valueAlloc(gpa, @as(f64, 1.5), .{});
    defer gpa.free(num);
    std.debug.print("number:  {s}\n", .{num});

    const str = try std.json.Stringify.valueAlloc(gpa, "he said \"hi\"", .{});
    defer gpa.free(str);
    std.debug.print("escaped: {s}\n", .{str});

    std.debug.print("via fmt: {f}\n", .{std.json.fmt(cfg, .{})});
}
```

```
minified:
{"host":"localhost","port":8080,"debug":true,"retries":null,"tags":["a","b"]}

indent_2:
{
  "host": "localhost",
  "port": 8080,
  "debug": true,
  "retries": null,
  "tags": [
    "a",
    "b"
  ]
}

no null fields:
{"host":"localhost","port":8080,"debug":true,"tags":["a","b"]}

array:   [1,2,3]
number:  1.5
escaped: "he said \"hi\""
via fmt: {"host":"localhost","port":8080,"debug":true,"retries":null,"tags":["a","b"]}
```

Encoding needs no schema — it reflects over the type at compile time. Any struct,
array, slice, optional, enum or number works, and strings are escaped for you.

Useful `Stringify.Options`:

| Option | Effect |
|---|---|
| `.whitespace = .minified` | default — no spaces |
| `.whitespace = .indent_2` | pretty-printed (also `.indent_4`, `.indent_tab`) |
| `.emit_null_optional_fields = false` | drop `null` fields instead of emitting them |
| `.escape_unicode = true` | `\uXXXX` instead of raw UTF-8 |
| `.emit_strings_as_arrays = true` | `[]u8` as numbers, not a string |

**`std.json.fmt(v, .{})` + `{f}` allocates nothing** — it encodes straight into
whatever writer the format string is going to. Prefer it when the JSON is headed
for stdout, a socket or a log line rather than into a variable.

## Dynamic JSON: `std.json.Value`

When you don't know the shape, parse to `std.json.Value` — a tagged union.

```zig
const std = @import("std");

fn describe(v: std.json.Value, depth: usize) void {
    const pad: [16]u8 = @splat(' ');
    const ind = pad[0..@min(depth * 2, 16)];
    switch (v) {
        .null => std.debug.print("{s}null\n", .{ind}),
        .bool => |b| std.debug.print("{s}bool {}\n", .{ ind, b }),
        .integer => |i| std.debug.print("{s}integer {d}\n", .{ ind, i }),
        .float => |f| std.debug.print("{s}float {d}\n", .{ ind, f }),
        .number_string => |s| std.debug.print("{s}number_string {s}\n", .{ ind, s }),
        .string => |s| std.debug.print("{s}string \"{s}\"\n", .{ ind, s }),
        .array => |a| {
            std.debug.print("{s}array ({d})\n", .{ ind, a.items.len });
            for (a.items) |x| describe(x, depth + 1);
        },
        .object => |o| {
            std.debug.print("{s}object ({d})\n", .{ ind, o.count() });
            var it = o.iterator();
            while (it.next()) |e| {
                std.debug.print("{s}  key \"{s}\":\n", .{ ind, e.key_ptr.* });
                describe(e.value_ptr.*, depth + 2);
            }
        },
    }
}

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    const text =
        \\{"id":7,"name":"ada","pi":3.5,"ok":true,"none":null,"list":[1,"two"]}
    ;

    const parsed = try std.json.parseFromSlice(std.json.Value, gpa, text, .{});
    defer parsed.deinit();

    describe(parsed.value, 0);

    const root = parsed.value.object;
    std.debug.print("\ndirect access:\n", .{});
    std.debug.print("  id       = {d}\n", .{root.get("id").?.integer});
    std.debug.print("  name     = {s}\n", .{root.get("name").?.string});
    std.debug.print("  missing  = {}\n", .{root.get("nope") == null});
    std.debug.print("  list[1]  = {s}\n", .{root.get("list").?.array.items[1].string});
}
```

```
object (6)
  key "id":
    integer 7
  key "name":
    string "ada"
  key "pi":
    float 3.5
  key "ok":
    bool true
  key "none":
    null
  key "list":
    array (2)
      integer 1
      string "two"

direct access:
  id       = 7
  name     = ada
  missing  = true
  list[1]  = two
```

The eight variants of `std.json.Value`:

| Variant | Payload |
|---|---|
| `.null` | — |
| `.bool` | `bool` |
| `.integer` | `i64` |
| `.float` | `f64` |
| `.number_string` | `[]const u8` — a number too large for `i64`/`f64` |
| `.string` | `[]const u8` |
| `.array` | `std.json.Array` (an `ArrayList(Value)`) — use `.items` |
| `.object` | `std.json.ObjectMap` (a `StringArrayHashMap(Value)`) — `.get`, `.iterator` |

`.object` is an **ArrayHashMap**, so iteration is in document order — unlike the
plain `HashMap` of lesson 7.2.

> `root.get("id").?.integer` asserts twice: that the key exists and that it's an
> integer. Both panic if wrong. Use `if (root.get("id")) |v| switch (v) { … }`
> on untrusted input.

**Prefer a struct over `Value` whenever you know the shape.** The struct version
is faster (no hash map), smaller, and validates the types for you. `Value` is for
genuinely dynamic data — a config file with user-defined keys, or a pass-through
proxy.

## Round trip

```zig
const std = @import("std");

const Item = struct { sku: []const u8, qty: u32 };
const Order = struct {
    id: u64,
    customer: []const u8,
    items: []const Item,
    note: ?[]const u8 = null,
};

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    const original =
        \\{"id":1001,"customer":"ada","items":[{"sku":"A1","qty":2},{"sku":"B2","qty":1}]}
    ;

    const parsed = try std.json.parseFromSlice(Order, gpa, original, .{});
    defer parsed.deinit();
    const o = parsed.value;

    std.debug.print("order {d} for {s}\n", .{ o.id, o.customer });
    var total: u32 = 0;
    for (o.items) |it| {
        total += it.qty;
        std.debug.print("  {s} x{d}\n", .{ it.sku, it.qty });
    }
    std.debug.print("total qty = {d}\n", .{total});

    const back = try std.json.Stringify.valueAlloc(gpa, o, .{ .emit_null_optional_fields = false });
    defer gpa.free(back);
    std.debug.print("re-encoded:\n{s}\n", .{back});
    std.debug.print("identical to input: {}\n", .{std.mem.eql(u8, original, back)});
}
```

```
order 1001 for ada
  A1 x2
  B2 x1
total qty = 3
re-encoded:
{"id":1001,"customer":"ada","items":[{"sku":"A1","qty":2},{"sku":"B2","qty":1}]}
identical to input: true
```

Byte-identical, because field order follows the struct declaration and
`emit_null_optional_fields = false` dropped the absent `note`. Don't *rely* on
byte equality in general — key order and number formatting are not guaranteed to
survive a round trip — but it's a good sanity check that your struct models the
document exactly.

---

Next: [7.7 — Time & Random](./07-time-random.md)
