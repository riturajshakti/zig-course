# 9.2 — Assertions

Every matcher in `std.testing` returns an **error union**, so every assertion
needs `try`. A test fails by returning an error — there is no `assert.fail()`,
no exception, no magic.

## The whole set, passing

```zig
const std = @import("std");
const testing = std.testing;

const Point = struct { x: i32, y: i32 };

fn parse(s: []const u8) !u32 {
    return std.fmt.parseInt(u32, s, 10);
}

test "expect" {
    try testing.expect(1 + 1 == 2);
}

test "expectEqual" {
    try testing.expectEqual(5, 2 + 3);
    try testing.expectEqual(Point{ .x = 1, .y = 2 }, Point{ .x = 1, .y = 2 });
}

test "expectEqualStrings" {
    try testing.expectEqualStrings("zig", "zi" ++ "g");
}

test "expectEqualSlices" {
    const a = [_]u32{ 1, 2, 3 };
    const b = [_]u32{ 1, 2, 3 };
    try testing.expectEqualSlices(u32, &a, &b);
}

test "expectError" {
    try testing.expectError(error.InvalidCharacter, parse("12x"));
}

test "expectApproxEqAbs" {
    const third: f64 = 1.0 / 3.0;
    try testing.expectApproxEqAbs(0.333333, third, 0.0001);
}

test "expectFmt" {
    try testing.expectFmt("x=7", "x={d}", .{7});
}

test "expectEqualDeep" {
    const a: []const []const u8 = &.{ "a", "b" };
    const b: []const []const u8 = &.{ "a", "b" };
    try testing.expectEqualDeep(a, b);
}
```

```sh
zig test a1.zig
```

```
1/8 a1.test.expect...OK
2/8 a1.test.expectEqual...OK
3/8 a1.test.expectEqualStrings...OK
4/8 a1.test.expectEqualSlices...OK
5/8 a1.test.expectError...OK
6/8 a1.test.expectApproxEqAbs...OK
7/8 a1.test.expectFmt...OK
8/8 a1.test.expectEqualDeep...OK
All 8 tests passed.
```

| Matcher | Arguments | Use for |
|---|---|---|
| `expect(bool)` | one bool | anything; **no diagnostic on failure** |
| `expectEqual(expected, actual)` | scalars, enums, structs, pointers | the default |
| `expectEqualStrings(a, b)` | two `[]const u8` | text — gives a character-level diff |
| `expectEqualSlices(T, a, b)` | element type + two slices | arrays/slices — gives a hex diff |
| `expectEqualDeep(a, b)` | anything | follows pointers and slices recursively |
| `expectError(error.X, union)` | an error + an error union | the error path |
| `expectApproxEqAbs(a, b, tol)` | floats + absolute tolerance | floats |
| `expectApproxEqRel(a, b, tol)` | floats + relative tolerance | floats of unknown magnitude |
| `expectFmt(want, "fmt", args)` | expected string + format call | a type's `format` method |
| `expectStringStartsWith/EndsWith` | two `[]const u8` | partial text matches |

**Argument order is (expected, actual)** everywhere. The failure messages say
"expected X, found Y", so getting it backwards produces a correct-looking but
inverted report.

## What a failure actually prints

```zig
const std = @import("std");
const testing = std.testing;

fn slugify(s: []const u8) []const u8 {
    _ = s;
    return "my-post";
}

test "expectEqual fails" {
    try testing.expectEqual(5, 2 + 2);
}

test "expectEqualStrings fails" {
    try testing.expectEqualStrings("my-blog-post", slugify("My Blog Post"));
}

test "expect fails" {
    try testing.expect(slugify("x").len == 3);
}
```

```sh
zig test f1.zig
```

**All three failures are deliberate.** Trimmed to the interesting parts:

```
1/3 f1.test.expectEqual fails...expected 5, found 4
FAIL (TestExpectedEqual)
/opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/testing.zig:93:17: 0x105081e93 in expectEqualInner__func_1036 (test)
                return error.TestExpectedEqual;
                ^
/path/f1.zig:10:5: 0x105081f33 in test.expectEqual fails (test)
    try testing.expectEqual(5, 2 + 2);
    ^
2/3 f1.test.expectEqualStrings fails...
====== expected this output: =========
my-blog-post␃

======== instead found this: =========
my-post␃

======================================
First difference occurs on line 1:
expected:
my-blog-post
   ^ ('\x62')
found:
my-post
   ^ ('\x70')
FAIL (TestExpectedEqual)
...
3/3 f1.test.expect fails...FAIL (TestUnexpectedResult)
/opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/testing.zig:584:14: 0x10507f3bf in expect (test)
    if (!ok) return error.TestUnexpectedResult;
             ^
/path/f1.zig:18:5: 0x10507f463 in test.expect fails (test)
    try testing.expect(slugify("x").len == 3);
    ^
0 passed; 0 skipped; 3 failed.
error: the following test command failed with exit code 1:
```

Three things to read out of that:

1. **A failure is an error return trace.** It shows the path from the `return
   error.X` inside `std.testing` back out to your `try` — same mechanism as any
   other Zig error. The last frame is always your assertion.
2. **`expect` tells you nothing** beyond `TestUnexpectedResult`. It saw only a
   `bool`; the values were gone before it was called. Prefer a typed matcher.
3. The summary line changes shape: `All N tests passed` becomes
   `N passed; N skipped; N failed`, and the process exits 1.

(The hex addresses and stdlib paths differ on every run — ASLR. Match on the
source lines, not the numbers.)

## `expectEqualSlices` and `expectError` diffs

```zig
const std = @import("std");
const testing = std.testing;

fn parse(s: []const u8) !u32 {
    return std.fmt.parseInt(u32, s, 10);
}

test "expectError: wrong error" {
    try testing.expectError(error.Overflow, parse("12x"));
}

test "expectError: no error at all" {
    try testing.expectError(error.Overflow, parse("12"));
}

test "expectEqualSlices fails" {
    const want = [_]u8{ 1, 2, 3 };
    const got = [_]u8{ 1, 9, 3 };
    try testing.expectEqualSlices(u8, &want, &got);
}
```

```sh
zig test f2.zig
```

**Deliberate failures**, stack traces elided:

```
1/3 f2.test.expectError: wrong error...expected error.Overflow, found error.InvalidCharacter
FAIL (TestUnexpectedError)
2/3 f2.test.expectError: no error at all...expected error.Overflow, found 12
FAIL (TestExpectedError)
3/3 f2.test.expectEqualSlices fails...slices differ. first difference occurs at index 1 (0x1)

============ expected this output: =============  len: 3 (0x3)

01 02 03                                          ...

============= instead found this: ==============  len: 3 (0x3)

01 09 03                                          .␉.

================================================

FAIL (TestExpectedEqual)
0 passed; 0 skipped; 3 failed.
```

| Error returned | Means |
|---|---|
| `TestUnexpectedError` | an error came back, but the wrong one |
| `TestExpectedError` | the call **succeeded** — it printed the value it got instead (`found 12`) |
| `TestExpectedEqual` | the values differ |
| `TestUnexpectedResult` | `expect` got `false` |

`expectError` for a wrong error also prints the **full error return trace of the
function under test** — in this case every `fmt.zig` frame down to
`charToDigit`. That's free root-cause information you don't get from any other
matcher.

## `expectEqual` on slices compares pointers

The one trap worth memorising.

```zig
const std = @import("std");
const testing = std.testing;

const Config = struct { name: []const u8, port: u16 };

test "expectEqualDeep fails" {
    const want = Config{ .name = "api", .port = 8080 };
    const got = Config{ .name = "api", .port = 3000 };
    try testing.expectEqualDeep(want, got);
}

test "expectEqual on slices compares pointers" {
    const a: []const u8 = "zig";
    var buf = [_]u8{ 'z', 'i', 'g' };
    const b: []const u8 = &buf;
    try testing.expectEqual(a, b);
}
```

```sh
zig test d1.zig
```

**Deliberate failures:**

```
1/2 d1.test.expectEqualDeep fails...expected 8080, found 3000
Field port incorrect. expected 8080, found 3000
FAIL (TestExpectedEqual)
2/2 d1.test.expectEqual on slices compares pointers...expected slice ptr u8@1009b4ecc, found u8@16f5b5ae4
FAIL (TestExpectedEqual)
0 passed; 0 skipped; 2 failed.
```

Both slices contain `"zig"`. `expectEqual` compared the **pointers** and
reported an address mismatch. `expectEqualDeep` names the offending field
(`Field port incorrect`) instead of dumping two whole structs.

| Comparing | Use | Not |
|---|---|---|
| `[]const u8` of text | `expectEqualStrings` | `expectEqual` |
| `[]T` of anything else | `expectEqualSlices(T, ..)` | `expectEqual` |
| a struct of scalars | `expectEqual` | — |
| a struct **containing** a slice | `expectEqualDeep` | `expectEqual` |
| nested slices-of-slices | `expectEqualDeep` | `expectEqualSlices` |

## `expectApproxEqAbs` needs typed floats

A `comptime_float` is rejected outright:

```zig
const std = @import("std");
const testing = std.testing;

test "approx on comptime_float" {
    try testing.expectApproxEqAbs(0.333333, 1.0 / 3.0, 0.0001);
}
```

```sh
zig test aa.zig
```

**Deliberate compile error:**

```
/opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/testing.zig:270:28: error: Cannot approximately compare two comptime_float values
        .comptime_float => @compileError("Cannot approximately compare two comptime_float values"),
                           ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
referenced by:
    expectApproxEqAbs [inlined]: /opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/testing.zig:260:34
    test.approx on comptime_float: aa.zig:5:34
```

Bind the value to an `f64`/`f32` first. The reasoning: a `comptime_float` is
arbitrary precision, so there is nothing to be approximate about.

## `expectFmt` tests a `format` method

Faster than allocating and comparing — it formats straight into a comparison.

```zig
const std = @import("std");
const testing = std.testing;

const Duration = struct {
    ms: u64,
    pub fn format(d: Duration, w: *std.Io.Writer) !void {
        if (d.ms < 1000) return w.print("{d}ms", .{d.ms});
        try w.print("{d:.2}s", .{@as(f64, @floatFromInt(d.ms)) / 1000.0});
    }
};

test "Duration formats" {
    try testing.expectFmt("250ms", "{f}", .{Duration{ .ms = 250 }});
    try testing.expectFmt("1.50s", "{f}", .{Duration{ .ms = 1500 }});
    try testing.expectFmt("took 1.50s!", "took {f}!", .{Duration{ .ms = 1500 }});
}
```

```sh
zig test ef.zig
```

```
1/1 ef.test.Duration formats...OK
All 1 tests passed.
```

(Width and alignment have no effect on `{f}`: `"[{f:>8}]"` does **not** pad a
custom `format` method's output, it just prints `[250ms]`. Padding applies to
the built-in specifiers only.)

## Skipping: `error.SkipZigTest`

`SkipZigTest` is a magic error the test runner reports as `SKIP` instead of
`FAIL`. There is no `test.skip` attribute — you return it from the body.

```zig
const std = @import("std");
const builtin = @import("builtin");
const testing = std.testing;

test "runs everywhere" {
    try testing.expect(true);
}

test "linux only" {
    if (builtin.os.tag != .linux) return error.SkipZigTest;
    try testing.expect(false);
}

test "skipped unconditionally" {
    return error.SkipZigTest;
}
```

```sh
zig test s1.zig; echo "exit=$?"
```

```
1/3 s1.test.runs everywhere...OK
2/3 s1.test.linux only...SKIP
3/3 s1.test.skipped unconditionally...SKIP
1 passed; 2 skipped; 0 failed.
exit=0
```

Note that `linux only` contains `expect(false)` and still passed the run — the
early `return` happened first. **Skips do not affect the exit code.** Common
guards: `builtin.os.tag`, `builtin.cpu.arch`, `builtin.mode`.

## `errdefer` for failure diagnostics

A test body is an ordinary function returning `!void`, so `errdefer` fires
exactly when the test fails. That is the Zig answer to a custom assertion
message.

```zig
const std = @import("std");
const testing = std.testing;

test "errdefer dumps context on failure" {
    var list: std.ArrayList(u32) = .empty;
    defer list.deinit(testing.allocator);

    try list.append(testing.allocator, 1);
    try list.append(testing.allocator, 2);
    errdefer std.debug.print("list was: {any}\n", .{list.items});

    try testing.expectEqual(3, list.items.len);
}
```

```sh
zig test e1.zig
```

**Deliberate failure:**

```
1/1 e1.test.errdefer dumps context on failure...expected 3, found 2
list was: { 1, 2 }
FAIL (TestExpectedEqual)
...
0 passed; 0 skipped; 1 failed.
error: the following test command failed with exit code 1:
```

`list was: { 1, 2 }` is the context the bare matcher could never print. Place
the `errdefer` *after* the setup you want to dump; it only covers what follows
it.

Note `errdefer` takes **no capture** in 0.17 — `errdefer |e| { ... }` is
`error: expected block or expression, found '|'`. Use plain `errdefer`.

---

Next: [9.3 — `testing.allocator`](./03-testing-allocator.md)
