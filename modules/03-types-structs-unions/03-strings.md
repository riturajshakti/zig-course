# 3.3 — Strings

**Zig has no `String` type.** There is no `str`, no `String`, no `StringBuilder`,
no methods on text. A string is `[]const u8` — a slice of bytes — and everything
you want to do with one is a free function in `std.mem`, `std.ascii`, `std.fmt`
or `std.unicode`.

```zig
const std = @import("std");

pub fn main() void {
    const lit = "Hello, Zig World";
    const s: []const u8 = lit;
    const z: [:0]const u8 = lit;
    const arr = lit.*;

    std.debug.print("@TypeOf(lit) = {s}\n", .{@typeName(@TypeOf(lit))});
    std.debug.print("@TypeOf(s)   = {s}\n", .{@typeName(@TypeOf(s))});
    std.debug.print("@TypeOf(z)   = {s}\n", .{@typeName(@TypeOf(z))});
    std.debug.print("@TypeOf(arr) = {s}\n", .{@typeName(@TypeOf(arr))});
    std.debug.print("s.len = {d}\n", .{s.len});
    std.debug.print("s[0] = {d} = '{c}'\n", .{ s[0], s[0] });
    std.debug.print("as bytes: {any}\n", .{s[0..5]});
}
```

```
@TypeOf(lit) = *const [16:0]u8
@TypeOf(s)   = []const u8
@TypeOf(z)   = [:0]const u8
@TypeOf(arr) = [16:0]u8
s.len = 16
s[0] = 72 = 'H'
as bytes: { 72, 101, 108, 108, 111 }
```

**A string literal is a `*const [N:0]u8`** — a pointer to a nul-terminated array
of known length, living in the binary's read-only data. It coerces to all three
of the others for free.

| Type | What it is | When |
|---|---|---|
| `*const [16:0]u8` | what a literal actually is | you rarely write this |
| `[]const u8` | ptr + len, read-only bytes | **the default — write this** |
| `[]u8` | ptr + len, writable bytes | buffers you own |
| `[:0]const u8` | plus a guaranteed trailing `0` | passing to C |
| `[16:0]u8` | the array by value | `lit.*`, copies to the stack |

### Why no `String` type?

The three reasons, in order of how much they matter:

1. **Who allocates?** A `String` must own a heap buffer, which means a hidden
   allocator. Zig's entire design is that allocation is visible in the
   signature. `[]const u8` owns nothing, so it can point at a literal, a stack
   buffer, an mmap'd file or the heap — identically and for free.
2. **It would be a lie about Unicode.** `str.length()` has to pick: bytes,
   codepoints, or grapheme clusters. Every language that picked got it wrong for
   someone. `[]const u8` just says *bytes*, and makes you choose explicitly when
   you need more.
3. **Zero-copy becomes the default.** `splitScalar` on a 100 MB log file
   allocates nothing — every token is a slice into the original buffer.

The cost is that `[]const u8` **borrows**, so everything from lesson 3.2's
dangling-slice trap applies to every string you handle.

## Searching and comparing

```zig
const std = @import("std");
const mem = std.mem;

pub fn main() void {
    const s = "Hello, Zig World";

    std.debug.print("s                        = \"{s}\"\n", .{s});
    std.debug.print("len                      = {d}\n", .{s.len});
    std.debug.print("eql(s, \"Hello, Zig World\") = {}\n", .{mem.eql(u8, s, "Hello, Zig World")});
    std.debug.print("eql(s, \"hello...\")        = {}\n", .{mem.eql(u8, s, "hello, zig world")});
    std.debug.print("eqlIgnoreCase            = {}\n", .{std.ascii.eqlIgnoreCase("ABC", "abc")});
    std.debug.print("startsWith \"Hello\"       = {}\n", .{mem.startsWith(u8, s, "Hello")});
    std.debug.print("endsWith \"World\"         = {}\n", .{mem.endsWith(u8, s, "World")});
    std.debug.print("indexOf \"Zig\"            = {?d}\n", .{mem.indexOf(u8, s, "Zig")});
    std.debug.print("indexOf \"Rust\"           = {?d}\n", .{mem.indexOf(u8, s, "Rust")});
    std.debug.print("lastIndexOf \"o\"          = {?d}\n", .{mem.lastIndexOf(u8, s, "o")});
    std.debug.print("indexOfScalar ','        = {?d}\n", .{mem.indexOfScalar(u8, s, ',')});
    std.debug.print("indexOfAny \"xyzZ\"        = {?d}\n", .{mem.indexOfAny(u8, s, "xyzZ")});
    std.debug.print("count \"l\"                = {d}\n", .{mem.count(u8, s, "l")});
    std.debug.print("containsAtLeast 2 \"l\"    = {}\n", .{mem.containsAtLeast(u8, s, 2, "l")});
    std.debug.print("order vs \"Hello\"         = {t}\n", .{mem.order(u8, s, "Hello")});
    std.debug.print("lessThan vs \"Z\"          = {}\n", .{mem.lessThan(u8, s, "Z")});
}
```

```
s                        = "Hello, Zig World"
len                      = 16
eql(s, "Hello, Zig World") = true
eql(s, "hello...")        = false
eqlIgnoreCase            = true
startsWith "Hello"       = true
endsWith "World"         = true
indexOf "Zig"            = 7
indexOf "Rust"           = null
lastIndexOf "o"          = 12
indexOfScalar ','        = 5
indexOfAny "xyzZ"        = 7
count "l"                = 3
containsAtLeast 2 "l"    = true
order vs "Hello"         = gt
lessThan vs "Z"          = true
```

**`==` on strings does not compile at all** — Zig refuses rather than silently
comparing pointers like C:

```
t2.zig:5:33: error: operator == not allowed for type '[]const u8'
    std.debug.print("{}\n", .{a == b});
                              ~~^~~~
```

`std.mem.eql` always.

## Slicing and trimming

```zig
const std = @import("std");
const mem = std.mem;

pub fn main() void {
    const s = "Hello, Zig World";

    std.debug.print("s[0..5]        = \"{s}\"\n", .{s[0..5]});
    std.debug.print("s[7..]         = \"{s}\"\n", .{s[7..]});
    std.debug.print("s[s.len - 5..] = \"{s}\"\n", .{s[s.len - 5 ..]});
    std.debug.print("first char     = '{c}'\n", .{s[0]});
    std.debug.print("last char      = '{c}'\n", .{s[s.len - 1]});

    const padded = "  pad  ";
    std.debug.print("trim      = \"{s}\"\n", .{mem.trim(u8, padded, " ")});
    std.debug.print("trimStart = \"{s}\"\n", .{mem.trimStart(u8, padded, " ")});
    std.debug.print("trimEnd   = \"{s}\"\n", .{mem.trimEnd(u8, padded, " ")});
    std.debug.print("trim \\n\\t  = \"{s}\"\n", .{mem.trim(u8, "\t hi \n", " \t\n\r")});
}
```

```
s[0..5]        = "Hello"
s[7..]         = "Zig World"
s[s.len - 5..] = "World"
first char     = 'H'
last char      = 'd'
trim      = "pad"
trimStart = "pad  "
trimEnd   = "  pad"
trim \n\t  = "hi"
```

The second argument to `trim` is a **set of bytes to strip**, not a prefix
string. All slicing here allocates nothing — every result points into `s`.

> `trimLeft`/`trimRight` were renamed to **`trimStart`/`trimEnd`** in 0.17.0.

## Case and classification — `std.ascii`

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const gpa = init.gpa;

    const up = try std.ascii.allocUpperString(gpa, "shout");
    defer gpa.free(up);
    const lo = try std.ascii.allocLowerString(gpa, "WHISPER");
    defer gpa.free(lo);

    var buf: [16]u8 = undefined;
    const in_place = std.ascii.upperString(&buf, "stack only");

    std.debug.print("allocUpperString = \"{s}\"\n", .{up});
    std.debug.print("allocLowerString = \"{s}\"\n", .{lo});
    std.debug.print("upperString      = \"{s}\"\n", .{in_place});
    std.debug.print("toUpper('a')     = '{c}'\n", .{std.ascii.toUpper('a')});
    std.debug.print("toLower('Z')     = '{c}'\n", .{std.ascii.toLower('Z')});
    std.debug.print("eqlIgnoreCase    = {}\n", .{std.ascii.eqlIgnoreCase("ABC", "abc")});
    std.debug.print("isDigit('7')     = {}\n", .{std.ascii.isDigit('7')});
    std.debug.print("isAlphabetic('7')= {}\n", .{std.ascii.isAlphabetic('7')});
    std.debug.print("isWhitespace(' ')= {}\n", .{std.ascii.isWhitespace(' ')});
}
```

```
allocUpperString = "SHOUT"
allocLowerString = "whisper"
upperString      = "STACK ONLY"
toUpper('a')     = 'A'
toLower('Z')     = 'z'
eqlIgnoreCase    = true
isDigit('7')     = true
isAlphabetic('7')= false
isWhitespace(' ')= true
```

The namespace is called `ascii` as a warning, not a coincidence — see the UTF-8
section. `upperString(&buf, src)` needs `buf.len >= src.len`; `allocUpperString`
returns memory **you** free.

## Splitting

```zig
const std = @import("std");
const mem = std.mem;

pub fn main() void {
    const csv = "a,b,,c";

    std.debug.print("splitScalar on ',':\n", .{});
    var it = mem.splitScalar(u8, csv, ',');
    while (it.next()) |p| std.debug.print("  [{s}]\n", .{p});

    std.debug.print("tokenizeScalar on ',':\n", .{});
    var tk = mem.tokenizeScalar(u8, csv, ',');
    while (tk.next()) |p| std.debug.print("  [{s}]\n", .{p});

    std.debug.print("splitSequence on \"::\":\n", .{});
    var sq = mem.splitSequence(u8, "x::y::z", "::");
    while (sq.next()) |p| std.debug.print("  [{s}]\n", .{p});

    std.debug.print("tokenizeAny on \" \\t,\":\n", .{});
    var ta = mem.tokenizeAny(u8, "one two\tthree,four", " \t,");
    while (ta.next()) |p| std.debug.print("  [{s}]\n", .{p});

    var kv = mem.splitScalar(u8, "name=ada", '=');
    const key = kv.next().?;
    const val = kv.rest();
    std.debug.print("key=[{s}] val=[{s}]\n", .{ key, val });
}
```

```
splitScalar on ',':
  [a]
  [b]
  []
  [c]
tokenizeScalar on ',':
  [a]
  [b]
  [c]
splitSequence on "::":
  [x]
  [y]
  [z]
tokenizeAny on " \t,":
  [one]
  [two]
  [three]
  [four]
key=[name] val=[ada]
```

| Function | Delimiter | Empty fields |
|---|---|---|
| `splitScalar(u8, s, ',')` | one byte | **kept** — CSV wants this |
| `splitSequence(u8, s, "::")` | a substring | kept |
| `splitAny(u8, s, " ,;")` | any byte in a set | kept |
| `tokenizeScalar(u8, s, ',')` | one byte | **skipped** |
| `tokenizeAny(u8, s, " \t\n")` | any byte in a set | skipped — whitespace wants this |

`.rest()` gives everything not yet consumed, which is how you split *once* on
the first `=`. Every piece is a slice into the input — **zero allocations**.

## Building

```zig
const std = @import("std");
const mem = std.mem;

pub fn main(init: std.process.Init) !void {
    const gpa = init.gpa;

    const joined = try mem.join(gpa, "-", &.{ "a", "b", "c" });
    defer gpa.free(joined);

    const cat = try mem.concat(gpa, u8, &.{ "foo", "bar" });
    defer gpa.free(cat);

    const replaced = try mem.replaceOwned(u8, gpa, "a.b.c", ".", "/");
    defer gpa.free(replaced);

    const printed = try std.fmt.allocPrint(gpa, "{s} v{d}.{d}", .{ "zig", 0, 17 });
    defer gpa.free(printed);

    var buf: [64]u8 = undefined;
    const stacked = try std.fmt.bufPrint(&buf, "{d} items @ {d:.2}", .{ 3, 1.5 });

    const comptime_cat = "foo" ++ "bar";

    std.debug.print("join        = \"{s}\"\n", .{joined});
    std.debug.print("concat      = \"{s}\"\n", .{cat});
    std.debug.print("replaceOwned= \"{s}\"\n", .{replaced});
    std.debug.print("allocPrint  = \"{s}\"\n", .{printed});
    std.debug.print("bufPrint    = \"{s}\"\n", .{stacked});
    std.debug.print("++ (comptime)= \"{s}\"\n", .{comptime_cat});

    var inplace: [5]u8 = "a.b.c".*;
    const n = mem.replace(u8, "a.b.c", ".", "/", &inplace);
    std.debug.print("replace     = \"{s}\" ({d} replacements)\n", .{ inplace, n });
}
```

```
join        = "a-b-c"
concat      = "foobar"
replaceOwned= "a/b/c"
allocPrint  = "zig v0.17"
bufPrint    = "3 items @ 1.50"
++ (comptime)= "foobar"
replace     = "a/b/c" (2 replacements)
```

| Want | Use | Allocates |
|---|---|---|
| join with a separator | `mem.join(gpa, "-", &.{...})` | yes, you free |
| raw concatenation | `mem.concat(gpa, u8, &.{...})` | yes, you free |
| concat two literals | `"a" ++ "b"` | no — **comptime only** |
| replace, new string | `mem.replaceOwned(u8, gpa, s, from, to)` | yes, you free |
| replace into a buffer | `mem.replace(u8, s, from, to, &out)` → count | no |
| format, heap | `std.fmt.allocPrint(gpa, fmt, args)` | yes, you free |
| format, stack | `std.fmt.bufPrint(&buf, fmt, args)` | no — `error.NoSpaceLeft` if small |
| incremental appends | `std.ArrayList(u8)` + `.appendSlice` | yes (module 07) |

`++` works only on comptime-known values. For runtime strings it is
`concat`/`join`/`allocPrint` — all of which take an allocator, all of which hand
you memory to free.

## Parsing numbers

```zig
const std = @import("std");

pub fn main() !void {
    std.debug.print("parseInt i32 \"-42\"   = {d}\n", .{try std.fmt.parseInt(i32, "-42", 10)});
    std.debug.print("parseInt u8  \"ff\" 16 = {d}\n", .{try std.fmt.parseInt(u8, "ff", 16)});
    std.debug.print("parseInt u8  \"1010\" 2= {d}\n", .{try std.fmt.parseInt(u8, "1010", 2)});
    std.debug.print("parseInt auto \"0x1f\"  = {d}\n", .{try std.fmt.parseInt(i32, "0x1f", 0)});
    std.debug.print("parseFloat f64 \"3.5\" = {d}\n", .{try std.fmt.parseFloat(f64, "3.5")});
    std.debug.print("parseFloat f64 \"1e3\" = {d}\n", .{try std.fmt.parseFloat(f64, "1e3")});

    const bad = std.fmt.parseInt(i32, "12x", 10);
    std.debug.print("parseInt \"12x\"        = {!d}\n", .{bad});
    const over = std.fmt.parseInt(u8, "999", 10);
    std.debug.print("parseInt u8 \"999\"     = {!d}\n", .{over});

    var buf: [32]u8 = undefined;
    std.debug.print("int -> string         = \"{s}\"\n", .{try std.fmt.bufPrint(&buf, "{d}", .{-42})});
}
```

```
parseInt i32 "-42"   = -42
parseInt u8  "ff" 16 = 255
parseInt u8  "1010" 2= 10
parseInt auto "0x1f"  = 31
parseFloat f64 "3.5" = 3.5
parseFloat f64 "1e3" = 1000
parseInt "12x"        = error.InvalidCharacter
parseInt u8 "999"     = error.Overflow
int -> string         = "-42"
```

Base `0` auto-detects `0x`, `0o`, `0b`. Parsing returns an **error union**, never
a sentinel value like JS's `NaN` — and `error.Overflow` is checked against the
target type, so `parseInt(u8, "999")` fails rather than wrapping.

## ⚠️ UTF-8: bytes are not characters

`.len` counts **bytes**. Zig never pretends otherwise.

```zig
const std = @import("std");

pub fn main() !void {
    const s = "héllo";

    std.debug.print("s              = \"{s}\"\n", .{s});
    std.debug.print("s.len (BYTES)  = {d}\n", .{s.len});
    std.debug.print("codepoints     = {d}\n", .{try std.unicode.utf8CountCodepoints(s)});
    std.debug.print("raw bytes      = {any}\n", .{s});
    std.debug.print("s[1] alone     = {d} (half of 'é')\n", .{s[1]});
    std.debug.print("validateSlice  = {}\n", .{std.unicode.utf8ValidateSlice(s)});

    std.debug.print("\nby byte:\n", .{});
    for (s, 0..) |b, i| std.debug.print("  [{d}] 0x{x:0>2}\n", .{ i, b });

    std.debug.print("\nby codepoint:\n", .{});
    const view = try std.unicode.Utf8View.init(s);
    var it = view.iterator();
    while (it.nextCodepointSlice()) |cp| {
        std.debug.print("  \"{s}\" = {d} bytes, U+{X:0>4}\n", .{ cp, cp.len, try std.unicode.utf8Decode(cp) });
    }
}
```

```
s              = "héllo"
s.len (BYTES)  = 6
codepoints     = 5
raw bytes      = { 104, 195, 169, 108, 108, 111 }
s[1] alone     = 195 (half of 'é')
validateSlice  = true

by byte:
  [0] 0x68
  [1] 0xc3
  [2] 0xa9
  [3] 0x6c
  [4] 0x6c
  [5] 0x6f

by codepoint:
  "h" = 1 bytes, U+0068
  "é" = 2 bytes, U+00E9
  "l" = 1 bytes, U+006C
  "l" = 1 bytes, U+006C
  "o" = 1 bytes, U+006F
```

**6 bytes, 5 codepoints.** `é` is `0xC3 0xA9`. `s[1]` is half a character and
means nothing on its own.

Two ways to get this wrong:

```zig
const std = @import("std");

pub fn main() void {
    const s = "héllo";
    const bad = s[0..2];
    const good = s[0..3];

    std.debug.print("s[0..2] = \"{s}\"  valid utf8: {}\n", .{ bad, std.unicode.utf8ValidateSlice(bad) });
    std.debug.print("s[0..3] = \"{s}\" valid utf8: {}\n", .{ good, std.unicode.utf8ValidateSlice(good) });
    std.debug.print("ascii.toUpper on \"héllo\": ", .{});
    for (s) |b| std.debug.print("{c}", .{std.ascii.toUpper(b)});
    std.debug.print("\n", .{});
}
```

```
s[0..2] = "h�"  valid utf8: false
s[0..3] = "hé" valid utf8: true
ascii.toUpper on "héllo": HéLLO
```

(That `�` is your terminal rendering a lone `0xC3` lead byte with no continuation — the raw output really is the single byte `0xC3`, not valid text.)

1. **`s[0..2]` split `é` in half** — valid Zig, invalid UTF-8, no panic. You
   get mojibake downstream.
2. **`std.ascii.toUpper` left `é` lowercase.** It is byte-wise ASCII: it has no
   idea `0xC3 0xA9` is one letter. There is no `É` in the output.

| Question | Answer |
|---|---|
| byte length | `s.len` — free |
| codepoint count | `try std.unicode.utf8CountCodepoints(s)` — O(n) |
| grapheme count ("🇮🇳" = 1?) | **not in the stdlib** — needs a library |
| is it valid UTF-8 | `std.unicode.utf8ValidateSlice(s)` |
| iterate codepoints | `(try Utf8View.init(s)).iterator()` |
| decode one | `std.unicode.utf8Decode(bytes)` → `u21` |
| bytes in this codepoint | `try std.unicode.utf8ByteSequenceLength(s[i])` |
| encode a `u21` | `std.unicode.utf8Encode(cp, &buf)` |
| uppercase, non-ASCII | **not in the stdlib** — `std.ascii` is ASCII-only |

**The practical rule:** treat text as opaque bytes — compare, search, split,
store, send. Only decode codepoints when you must count or render characters,
and reach for a library when you need real Unicode (casing, normalization,
graphemes). This is the same discipline Go uses, and for the same reason.

## Operations table

| Operation | Code |
|---|---|
| declare | `const s: []const u8 = "hi";` |
| writable buffer | `var buf: [32]u8 = undefined;` |
| literal → array copy | `const a = "hi".*;` → `[2:0]u8` |
| length in bytes | `s.len` |
| byte at | `s[i]` (a `u8`, not a character) |
| first / last | `s[0]` / `s[s.len - 1]` |
| substring | `s[a..b]`, `s[a..]`, `s[0 .. s.len - 1]` |
| equality | `std.mem.eql(u8, a, b)` |
| equality, ignore case | `std.ascii.eqlIgnoreCase(a, b)` |
| ordering | `std.mem.order(u8, a, b)` → `.lt`/`.eq`/`.gt` |
| sortable comparator | `std.mem.lessThan(u8, a, b)` |
| prefix / suffix | `std.mem.startsWith` / `endsWith(u8, s, x)` |
| find substring | `std.mem.indexOf(u8, s, sub)` → `?usize` |
| find byte | `std.mem.indexOfScalar(u8, s, c)` → `?usize` |
| find from end | `std.mem.lastIndexOf(u8, s, sub)` |
| find any of a set | `std.mem.indexOfAny(u8, s, set)` |
| contains | `std.mem.indexOf(...) != null` |
| occurrences | `std.mem.count(u8, s, sub)` |
| at least n | `std.mem.containsAtLeast(u8, s, n, sub)` |
| trim both ends | `std.mem.trim(u8, s, " \t\n\r")` |
| trim one end | `std.mem.trimStart` / `trimEnd` |
| split, keep empties | `std.mem.splitScalar(u8, s, ',')` |
| split on a substring | `std.mem.splitSequence(u8, s, "::")` |
| split, skip empties | `std.mem.tokenizeScalar(u8, s, ',')` |
| split on a set | `std.mem.tokenizeAny(u8, s, " \t\n")` |
| remaining input | `it.rest()` |
| join | `try std.mem.join(gpa, "-", &.{...})` |
| concat (runtime) | `try std.mem.concat(gpa, u8, &.{...})` |
| concat (comptime) | `"a" ++ "b"` |
| replace → new string | `try std.mem.replaceOwned(u8, gpa, s, from, to)` |
| replace → buffer | `std.mem.replace(u8, s, from, to, &out)` → count |
| uppercase / lowercase | `try std.ascii.allocUpperString(gpa, s)` / `allocLower…` |
| case into a buffer | `std.ascii.upperString(&buf, s)` |
| one byte's case | `std.ascii.toUpper(c)` / `toLower(c)` |
| classify a byte | `std.ascii.isDigit` / `isAlphabetic` / `isWhitespace` / `isUpper` |
| string → int | `try std.fmt.parseInt(i32, s, 10)` (base `0` auto-detects) |
| string → float | `try std.fmt.parseFloat(f64, s)` |
| int → string (heap) | `try std.fmt.allocPrint(gpa, "{d}", .{n})` |
| int → string (stack) | `try std.fmt.bufPrint(&buf, "{d}", .{n})` |
| iterate bytes | `for (s) \|b\|` |
| iterate with index | `for (s, 0..) \|b, i\|` |
| iterate codepoints | `var it = (try std.unicode.Utf8View.init(s)).iterator();` |
| codepoint count | `try std.unicode.utf8CountCodepoints(s)` |
| validate UTF-8 | `std.unicode.utf8ValidateSlice(s)` |
| heap copy | `try gpa.dupe(u8, s)` → **you free it** |
| nul-terminated copy | `try gpa.allocSentinel(u8, s.len, 0)` + `@memcpy` |
| from a C string | `std.mem.span(c_ptr)` |
| truncate at a byte | `std.mem.sliceTo(s, 0)` |
| free | `gpa.free(s)` |

Any row taking `gpa` returns memory **you** must free. Everything else returns a
slice into existing memory — fast, and subject to the dangling-slice trap.

---

Next: [3.4 — Structs](./04-structs.md)
