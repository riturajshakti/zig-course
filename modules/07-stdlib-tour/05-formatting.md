# 7.5 — Formatting

One format-string syntax serves `std.debug.print`, `writer.print`,
`std.fmt.allocPrint` and `std.fmt.bufPrint`. The specifier goes inside braces,
options after a colon:

```
{d}        {d:.3}       {d:>8}      {d:0>8}
 │          │            │           │
 └ what     └ precision  └ width     └ fill + align
```

A wrong specifier is a **compile** error, not a runtime surprise.

## Specifiers

| Specifier | Prints | Example output |
|---|---|---|
| `{d}` | decimal (ints and floats) | `65`, `3.14159265` |
| `{s}` | bytes as a UTF-8 string | `zig` |
| `{c}` | one byte as a character | `A` |
| `{x}` / `{X}` | lowercase / uppercase hex | `beef` / `BEEF` |
| `{b}` | binary | `101` |
| `{o}` | octal | `10` |
| `{e}` | scientific notation | `3.14159265e0` |
| `{u}` | a Unicode codepoint (`u21`) | `Z`, `😀` |
| `{any}` | Zig literal syntax, recursively | `{ 122, 105, 103 }` |
| `{t}` | an enum's or error's tag name | `debug` |
| `{f}` | calls the type's own `pub fn format` | `(3, -4)` |
| `{?d}` | unwraps an optional, else `null` | `7` / `null` |
| `{!d}` | unwraps an error union, else the error | `5` / `error.Boom` |
| `{}` | the type's default | `65`, `true` |

```zig
const std = @import("std");

const Level = enum { debug, info, warn };

pub fn main() void {
    const n: u8 = 65;
    const big: u32 = 48879;
    const pi: f64 = 3.14159265;
    const some: ?i32 = 7;
    const none: ?i32 = null;
    const ok: anyerror!i32 = 5;
    const bad: anyerror!i32 = error.Boom;
    const bytes = [_]u8{ 'z', 'i', 'g' };

    std.debug.print("{{d}}   {d}\n", .{n});
    std.debug.print("{{s}}   {s}\n", .{"hi"});
    std.debug.print("{{c}}   {c}\n", .{n});
    std.debug.print("{{x}}   {x}\n", .{big});
    std.debug.print("{{X}}   {X}\n", .{big});
    std.debug.print("{{b}}   {b}\n", .{@as(u8, 5)});
    std.debug.print("{{o}}   {o}\n", .{@as(u8, 8)});
    std.debug.print("{{any}} {any}\n", .{bytes});
    std.debug.print("{{s}}   {s}\n", .{bytes});
    std.debug.print("{{t}}   {t}\n", .{Level.debug});
    std.debug.print("{{?d}}  {?d} {?d}\n", .{ some, none });
    std.debug.print("{{!d}}  {!d} {!d}\n", .{ ok, bad });
    std.debug.print("{{u}}   {u} {u}\n", .{ @as(u21, 0x5A), @as(u21, 0x1F600) });
    std.debug.print("{{e}}   {e}\n", .{pi});
    std.debug.print("{{}}    {} {} {}\n", .{ n, true, pi });
    std.debug.print("{{d}} f  {d}\n", .{pi});
}
```

```
{d}   65
{s}   hi
{c}   A
{x}   beef
{X}   BEEF
{b}   101
{o}   10
{any} { 122, 105, 103 }
{s}   zig
{t}   debug
{?d}  7 null
{!d}  5 error.Boom
{u}   Z 😀
{e}   3.14159265e0
{}    65 true 3.14159265
{d} f  3.14159265
```

`{{` and `}}` are literal braces. `%` is **not** special in Zig format strings —
write it directly.

Three traps in that output:

- **`{d}` on an array is a compile error.** `{d}` with a `[3]u8` gives
  `error: invalid format string 'd' for type '[3]u8'`. Numeric specifiers need a
  scalar or a slice; use `{any}` for the numbers or `{s}` for the text.
- **`{!}` on an error union of a slice is a compile error** — write `{!s}`.
- **`{t}` replaced `@tagName` in format strings.** It works on enums, errors, and
  on `@import("builtin").cpu.arch`.

## Width, precision, alignment, fill

```zig
const std = @import("std");

pub fn main() void {
    const pi: f64 = 3.14159265;

    std.debug.print("[{d:.0}] [{d:.2}] [{d:.5}]\n", .{ pi, pi, pi });
    std.debug.print("[{d:8}]\n", .{42});
    std.debug.print("[{d:>8}] right\n", .{42});
    std.debug.print("[{d:<8}] left\n", .{42});
    std.debug.print("[{d:^8}] center\n", .{42});
    std.debug.print("[{d:0>8}] zero fill\n", .{42});
    std.debug.print("[{d:*<8}] star fill\n", .{42});
    std.debug.print("[{d:.<8}] dot fill\n", .{42});
    std.debug.print("[{s:>10}]\n", .{"zig"});
    std.debug.print("[{s:_^11}]\n", .{"zig"});
    std.debug.print("[{d:>9.3}] width+precision\n", .{pi});
    std.debug.print("[{x:0>4}] hex padded\n", .{@as(u16, 255)});
    std.debug.print("[{b:0>8}] binary byte\n", .{@as(u8, 5)});
    std.debug.print("[{d:>4}]\n[{d:>4}]\n[{d:>4}]\n", .{ 1, 22, 333 });
}
```

```
[3] [3.14] [3.14159]
[      42]
[      42] right
[42      ] left
[   42   ] center
[00000042] zero fill
[42******] star fill
[42......] dot fill
[       zig]
[____zig____]
[    3.142] width+precision
[00ff] hex padded
[00000101] binary byte
[   1]
[  22]
[ 333]
```

| Syntax | Meaning |
|---|---|
| `{d:.3}` | precision — 3 digits after the point |
| `{d:8}` | minimum width 8 |
| `{d:>8}` `{d:<8}` `{d:^8}` | right / left / center within the width |
| `{d:0>8}` | the character **before** the arrow is the fill |
| `{d:>9.3}` | width and precision together, width first |

The fill character goes before the alignment arrow: `0>` means "pad with zeros,
right-aligned". Note `{d:*<8}` gives `42******` — fill works with any alignment.

## Where the output goes

| Function | Target | Allocates | Fails if |
|---|---|---|---|
| `std.debug.print(fmt, args)` | stderr, unbuffered | no | never (ignores errors) |
| `writer.print(fmt, args)` | any `*std.Io.Writer` | no | the writer fails |
| `std.fmt.allocPrint(gpa, …)` | a new `[]u8` | **yes** | OOM |
| `std.fmt.allocPrintSentinel(gpa, …, 0)` | a new `[:0]u8` | **yes** | OOM |
| `std.fmt.bufPrint(&buf, …)` | your stack buffer | no | `error.NoSpaceLeft` |
| `std.fmt.comptimePrint(…)` | a comptime string | no | at compile time |
| `std.fmt.count(fmt, args)` | just the byte count | no | never |

```zig
const std = @import("std");

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    const heap = try std.fmt.allocPrint(gpa, "{s} v{d}.{d}", .{ "zig", 0, 17 });
    defer gpa.free(heap);
    std.debug.print("allocPrint     \"{s}\" ({d} bytes)\n", .{ heap, heap.len });

    const heapz = try std.fmt.allocPrintSentinel(gpa, "id={d}", .{7}, 0);
    defer gpa.free(heapz);
    std.debug.print("allocPrintSent \"{s}\" type={s}\n", .{ heapz, @typeName(@TypeOf(heapz)) });

    var buf: [32]u8 = undefined;
    const stack = try std.fmt.bufPrint(&buf, "{d} + {d} = {d}", .{ 2, 3, 5 });
    std.debug.print("bufPrint       \"{s}\"\n", .{stack});

    var tiny: [4]u8 = undefined;
    const r = std.fmt.bufPrint(&tiny, "way too long", .{});
    std.debug.print("bufPrint small {!s}\n", .{r});

    const ct = std.fmt.comptimePrint("built at compile time: {d}", .{6 * 7});
    std.debug.print("comptimePrint  \"{s}\" type={s}\n", .{ ct, @typeName(@TypeOf(ct)) });

    std.debug.print("fmt.count      {d}\n", .{std.fmt.count("{s}-{d}", .{ "ab", 123 })});

    var out: std.Io.Writer = .fixed(&buf);
    try out.print("to a Writer: {d}", .{99});
    std.debug.print("Writer.fixed   \"{s}\"\n", .{out.buffered()});
}
```

```
allocPrint     "zig v0.17" (9 bytes)
allocPrintSent "id=7" type=[:0]u8
bufPrint       "2 + 3 = 5"
bufPrint small error.NoSpaceLeft
comptimePrint  "built at compile time: 42" type=*const [25:0]u8
fmt.count      6
Writer.fixed   "to a Writer: 99"
```

Worth noting:

- **`comptimePrint` returns `*const [25:0]u8`** — a string *literal*, computed
  during compilation and baked into the binary. No allocator, no buffer, no
  runtime cost. It only accepts comptime-known arguments.
- **`std.Io.Writer.fixed(&buf)` returns a `Writer` directly** — no `.interface`
  field, unlike a `File.Writer`. `out.buffered()` reads back what was written.
  It's the easiest way to unit-test code that takes a `*std.Io.Writer`.
- **`fmt.count` is how `comptimePrint` sizes its array**, and it's useful for
  sizing an allocation before you make it.

## Custom `format`

Give a type `pub fn format(self: T, w: *std.Io.Writer) !void` and `{f}` calls it.

> **The signature changed in 0.17.** It is now **two** parameters. Older
> tutorials show `pub fn format(self, comptime fmt: []const u8, options: std.fmt.FormatOptions, writer: anytype)`
> — four parameters, which no longer compiles.

```zig
const std = @import("std");

const Point = struct {
    x: i32,
    y: i32,
    pub fn format(p: Point, w: *std.Io.Writer) !void {
        try w.print("({d}, {d})", .{ p.x, p.y });
    }
};

const Money = struct {
    cents: i64,
    pub fn format(m: Money, w: *std.Io.Writer) !void {
        const sign = if (m.cents < 0) "-" else "";
        const abs = @abs(m.cents);
        try w.print("{s}${d}.{d:0>2}", .{ sign, abs / 100, abs % 100 });
    }
};

const Tags = struct {
    items: []const []const u8,
    pub fn format(t: Tags, w: *std.Io.Writer) !void {
        try w.writeAll("[");
        for (t.items, 0..) |x, i| {
            if (i > 0) try w.writeAll(", ");
            try w.writeAll(x);
        }
        try w.writeAll("]");
    }
};

pub fn main() !void {
    var dbg: std.heap.DebugAllocator(.{}) = .init;
    defer _ = dbg.deinit();
    const gpa = dbg.allocator();

    const p = Point{ .x = 3, .y = -4 };
    std.debug.print("{{f}}   {f}\n", .{p});
    std.debug.print("{{any}} {any}\n", .{p});

    std.debug.print("{f}\n", .{Money{ .cents = 12345 }});
    std.debug.print("{f}\n", .{Money{ .cents = -705 }});
    std.debug.print("{f}\n", .{Money{ .cents = 9 }});
    std.debug.print("{f}\n", .{Tags{ .items = &.{ "zig", "fast", "small" } }});

    const s = try std.fmt.allocPrint(gpa, "point is {f}", .{p});
    defer gpa.free(s);
    std.debug.print("in allocPrint: {s}\n", .{s});

    var buf: [32]u8 = undefined;
    std.debug.print("in bufPrint:   {s}\n", .{try std.fmt.bufPrint(&buf, "{f}", .{p})});
}
```

```
{f}   (3, -4)
{any} .{ .x = 3, .y = -4 }
$123.45
-$7.05
$0.09
[zig, fast, small]
in allocPrint: point is (3, -4)
in bufPrint:   (3, -4)
```

- **`{any}` ignores your `format` method** and dumps Zig literal syntax, leading
  dot included. `{f}` is the only way to call it, and `{any}` stays available as
  the debug view — which is why `{any}` needs no `format` method at all.
- **The writer is a plain `*std.Io.Writer`**, so inside `format` you have
  `print`, `writeAll`, `writeByte`, `splatByteAll` — see the `Tags` example
  joining with commas.
- Returning `!void` means a `format` method can fail; the error propagates out of
  whatever `print` call invoked it.

### ⚠️ Width and alignment do not apply to `{f}`

```zig
const std = @import("std");

const M = struct {
    c: u64,
    pub fn format(m: M, w: *std.Io.Writer) !void {
        try w.print("${d}", .{m.c});
    }
};

pub fn main() !void {
    std.debug.print("[{f:>10}]\n", .{M{ .c = 5 }});
    std.debug.print("[{f:*>10}]\n", .{M{ .c = 5 }});
    var buf: [16]u8 = undefined;
    const s = try std.fmt.bufPrint(&buf, "{f}", .{M{ .c = 5 }});
    std.debug.print("[{s:>10}] via bufPrint then {{s:>10}}\n", .{s});
}
```

```
[$5]
[$5]
[        $5] via bufPrint then {s:>10}
```

The width is **silently ignored** — your `format` method writes straight to the
writer and never sees the options. To align a custom type, render it to a buffer
first and then apply `{s:>N}` to the result.

## Parsing: strings → numbers

```zig
const std = @import("std");

pub fn main() !void {
    std.debug.print("parseInt \"-42\" 10   = {d}\n", .{try std.fmt.parseInt(i32, "-42", 10)});
    std.debug.print("parseInt \"ff\" 16    = {d}\n", .{try std.fmt.parseInt(u8, "ff", 16)});
    std.debug.print("parseInt \"1010\" 2   = {d}\n", .{try std.fmt.parseInt(u8, "1010", 2)});
    std.debug.print("parseInt \"0x1f\" 0   = {d}\n", .{try std.fmt.parseInt(i32, "0x1f", 0)});
    std.debug.print("parseInt \"1_000\" 10 = {d}\n", .{try std.fmt.parseInt(u32, "1_000", 10)});
    std.debug.print("parseInt \"12x\"      = {!d}\n", .{std.fmt.parseInt(i32, "12x", 10)});
    std.debug.print("parseInt u8 \"999\"   = {!d}\n", .{std.fmt.parseInt(u8, "999", 10)});
    std.debug.print("parseInt \"\"         = {!d}\n", .{std.fmt.parseInt(i32, "", 10)});
    std.debug.print("parseUnsigned \"-1\"  = {!d}\n", .{std.fmt.parseUnsigned(u8, "-1", 10)});
    std.debug.print("parseFloat \"3.5\"    = {d}\n", .{try std.fmt.parseFloat(f64, "3.5")});
    std.debug.print("parseFloat \"1e3\"    = {d}\n", .{try std.fmt.parseFloat(f64, "1e3")});
    std.debug.print("parseFloat \"-0.25\"  = {d}\n", .{try std.fmt.parseFloat(f32, "-0.25")});
    std.debug.print("parseFloat \"abc\"    = {!d}\n", .{std.fmt.parseFloat(f64, "abc")});
    std.debug.print("parseIntSizeSuffix  = {d}\n", .{try std.fmt.parseIntSizeSuffix("10K", 10)});
}
```

```
parseInt "-42" 10   = -42
parseInt "ff" 16    = 255
parseInt "1010" 2   = 10
parseInt "0x1f" 0   = 31
parseInt "1_000" 10 = 1000
parseInt "12x"      = error.InvalidCharacter
parseInt u8 "999"   = error.Overflow
parseInt ""         = error.InvalidCharacter
parseUnsigned "-1"  = error.InvalidCharacter
parseFloat "3.5"    = 3.5
parseFloat "1e3"    = 1000
parseFloat "-0.25"  = -0.25
parseFloat "abc"    = error.InvalidCharacter
parseIntSizeSuffix  = 10000
```

| Call | Notes |
|---|---|
| `parseInt(T, s, 10)` | base 10 |
| `parseInt(T, s, 0)` | **auto-detect** `0x`, `0o`, `0b` |
| `parseUnsigned(T, s, 10)` | rejects a `-` sign outright |
| `parseFloat(T, s)` | no base; handles `1e3`, `inf`, `nan` |
| `parseIntSizeSuffix(s, 10)` | `"10K"` → `10000` |

Two things Zig does that JS and Python don't:

- **Errors, never sentinels.** There is no `NaN`-on-failure and no silent
  truncation — `"12x"` is `error.InvalidCharacter`, full stop.
- **Overflow is checked against the target type.** `parseInt(u8, "999")` is
  `error.Overflow`, not `255` and not a wrapped `231`.

Underscores are allowed as digit separators (`"1_000"`), matching Zig's own
integer literals.

## Durations

`std.fmt.fmtDuration` is **gone** in 0.17. Durations are a type now —
`std.Io.Duration` — and it has its own `format` method, so you print it with
`{f}`.

```zig
const std = @import("std");

pub fn main() void {
    const d1: std.Io.Duration = .fromMilliseconds(1500);
    const d2: std.Io.Duration = .fromNanoseconds(1234);
    const d3: std.Io.Duration = .fromSeconds(3725);
    const d4: std.Io.Duration = .fromMicroseconds(250);
    std.debug.print("1500ms -> {f}\n", .{d1});
    std.debug.print("1234ns -> {f}\n", .{d2});
    std.debug.print("3725s  -> {f}\n", .{d3});
    std.debug.print("250us  -> {f}\n", .{d4});
    std.debug.print("d1.toMilliseconds()={d}\n", .{d1.toMilliseconds()});
}
```

```
1500ms -> 1.5s
1234ns -> 1.234us
3725s  -> 1h2m5s
250us  -> 250us
d1.toMilliseconds()=1500
```

It picks a sensible unit on its own. Lesson 7.7 uses this to report timings.

## Putting it together: an aligned table

```zig
const std = @import("std");

const Row = struct { name: []const u8, qty: u32, price_cents: u64 };

fn money(buf: []u8, cents: u64) ![]u8 {
    return std.fmt.bufPrint(buf, "${d}.{d:0>2}", .{ cents / 100, cents % 100 });
}

pub fn main() !void {
    const rows = [_]Row{
        .{ .name = "widget", .qty = 3, .price_cents = 499 },
        .{ .name = "gizmo", .qty = 12, .price_cents = 12050 },
        .{ .name = "doohickey", .qty = 1, .price_cents = 7 },
    };

    const line: [29]u8 = @splat('-');
    var buf: [32]u8 = undefined;

    std.debug.print("{s:<12}{s:>5}{s:>12}\n", .{ "ITEM", "QTY", "TOTAL" });
    std.debug.print("{s}\n", .{line});

    var grand: u64 = 0;
    for (rows) |r| {
        const total = r.price_cents * r.qty;
        grand += total;
        std.debug.print("{s:<12}{d:>5}{s:>12}\n", .{ r.name, r.qty, try money(&buf, total) });
    }
    std.debug.print("{s}\n", .{line});
    std.debug.print("{s:<12}{s:>5}{s:>12}\n", .{ "GRAND", "", try money(&buf, grand) });
}
```

```
ITEM          QTY       TOTAL
-----------------------------
widget          3      $14.97
gizmo          12    $1446.00
doohickey       1       $0.07
-----------------------------
GRAND                $1461.04
```

Two techniques here: money is formatted into a scratch buffer so `{s:>12}` can
right-align it (per the `{f}` limitation above), and the rule is
`const line: [29]u8 = @splat('-')` — **`**` for array repetition was removed in
0.17**, so `[_]u8{'-'} ** 29` no longer compiles.

---

Next: [7.6 — JSON](./06-json.md)
