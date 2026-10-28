# 9.3 — `testing.allocator`

`std.testing.allocator` is a leak-checking allocator wired into the test runner.
**A leak fails the test.** This is the most valuable thing in Zig's testing
story, and the way it reports failure is unlike anything in Node, Go or Python.

It exists only in a test build — outside one, `std.testing.allocator` is
`@compileError("not testing")`.

## A leak fails the test

```zig
const std = @import("std");
const testing = std.testing;

fn repeat(gpa: std.mem.Allocator, s: []const u8, n: usize) ![]u8 {
    var out: std.ArrayList(u8) = .empty;
    errdefer out.deinit(gpa);
    for (0..n) |_| try out.appendSlice(gpa, s);
    return out.toOwnedSlice(gpa);
}

test "repeat: freed" {
    const got = try repeat(testing.allocator, "ab", 3);
    defer testing.allocator.free(got);
    try testing.expectEqualStrings("ababab", got);
}

test "repeat: not freed" {
    const got = try repeat(testing.allocator, "ab", 3);
    try testing.expectEqualStrings("ababab", got);
}
```

```sh
zig test l1.zig; echo "exit=$?"
```

**The second test leaks on purpose.** The real output:

```
1/2 l1.test.repeat: freed...OK
2/2 l1.test.repeat: not freed...OK
[SafeAllocator] (err): leaked [addr: 100f64090, len: 6 (0x6) align: 1] allocated at: 
/opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/array_list.zig:751:26: 0x100e3619b in toOwnedSlice (test)
            if (gpa.remap(old_memory, self.items.len)) |new_items| {
                         ^
/path/l1.zig:8:28: 0x100e333ff in repeat (test)
    return out.toOwnedSlice(gpa);
                           ^
/path/l1.zig:18:27: 0x100e3382b in test.repeat: not freed (test)
    const got = try repeat(testing.allocator, "ab", 3);
                          ^
...
(additional stack frames may have been skipped...)

All 2 tests passed.
1 errors were logged.
1 tests leaked memory.
error: the following test command failed with exit code 1:
/Users/raj/.cache/zig/o/dabceff9962ce791c166a81fb6a13c12/test --seed=0xcf2763b2
exit=1
```

**Read that again: `OK`, then `All 2 tests passed.`, then exit code 1.**

Both assertions were correct — `repeat` returned `"ababab"` as promised. The
test ran to completion and passed. The leak is detected *after* the body
returns, so the per-test verdict is already printed. The build still fails:

| Line | Meaning |
|---|---|
| `2/2 ... OK` | the assertions passed |
| `[SafeAllocator] (err): leaked [addr: ... len: 6 ...]` | 6 bytes never freed, with the allocation's trace |
| `All 2 tests passed.` | the per-test verdicts, printed before the leak check |
| `1 errors were logged.` | a leak is reported through `std.log.err` |
| `1 tests leaked memory.` | and attributed to the test that leaked |
| `exit code 1` | **the authoritative result** |

So: **never grep for "All N tests passed" in CI — check the exit code.** The
summary line is not the verdict.

## `[SafeAllocator]`, not `DebugAllocator`

In 0.17 `std.testing.allocator` is a `std.heap.SafeAllocator`, a different type
from the `std.heap.DebugAllocator` you use in `main`. The report format differs:

| | `DebugAllocator` (in `main`) | `testing.allocator` |
|---|---|---|
| leak prefix | `error(DebugAllocator): memory address 0x... leaked:` | `[SafeAllocator] (err): leaked [addr: ... len: ... align: ...]` |
| how you check | `gpa_s.deinit()` returns `.leak` | the test runner does it for you |
| reports the size | no | **yes** — `len: 6 (0x6)` |
| double free | SEGFAULTs by default | reports it properly (below) |

The useful part — the stack trace of the allocation that leaked — is the same in
both. Don't try to match the message text between them.

## Double free is caught, with three traces

```zig
const std = @import("std");
const testing = std.testing;

test "double free" {
    const buf = try testing.allocator.alloc(u8, 8);
    testing.allocator.free(buf);
    testing.allocator.free(buf);
}
```

```sh
zig test df.zig
```

**Deliberate crash**, trimmed to the three headings:

```
1/1 df.test.double free...thread 16617132 panic: double free of [addr: 100e80010, len: 8 (0x8) align: 1]
alloc: 
/path/df.zig:5:44: 0x100d573c3 in test.double free (test)
    const buf = try testing.allocator.alloc(u8, 8);
                                           ^
...

first free: 
/path/df.zig:6:27: 0x100d5740f in test.double free (test)
    testing.allocator.free(buf);
                          ^
...

second free:
/opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/heap/SafeAllocator.zig:518:30: 0x100d4ff57 in startModify (test)
                        panic(
                             ^
...
/path/df.zig:7:27: 0x100d57423 in test.double free (test)
    testing.allocator.free(buf);
                          ^
...
error: the following test command terminated with signal ABRT:
```

**Three stack traces: where it was allocated, where it was first freed, where
you freed it again.** That is the whole investigation in one shot. Note this is
a *panic* (SIGABRT), not a failed test — the process dies immediately, so later
tests don't run.

## The shape every allocating function's test takes

Three lines, and it catches every leak class from module 5:

```zig
const std = @import("std");
const testing = std.testing;

fn shout(gpa: std.mem.Allocator, s: []const u8) ![]u8 {
    const out = try gpa.alloc(u8, s.len + 1);
    for (s, 0..) |c, i| out[i] = std.ascii.toUpper(c);
    out[s.len] = '!';
    return out;
}

test "shout does not leak" {
    const result = try shout(testing.allocator, "hello");
    defer testing.allocator.free(result);
    try testing.expectEqualStrings("HELLO!", result);
}
```

```sh
zig test sh.zig
```

```
1/1 sh.test.shout does not leak...OK
All 1 tests passed.
```

The `defer free` is what makes it a leak test. Omit it and the test leaks the
*result*, which masks a leak *inside* the function.

## `testing.failing_allocator` — the OOM path

`error.OutOfMemory` is the one error path you can't trigger with normal input.
`testing.failing_allocator` fails **every** allocation:

```zig
const std = @import("std");
const testing = std.testing;

fn build(gpa: std.mem.Allocator) ![]u8 {
    return gpa.dupe(u8, "payload");
}

test "failing_allocator always OOMs" {
    try testing.expectError(error.OutOfMemory, build(testing.failing_allocator));
}

test "fail on the 2nd allocation" {
    var fa: testing.FailingAllocator = .init(testing.allocator, .{ .fail_index = 1 });
    const gpa = fa.allocator();

    const first = try build(gpa);
    defer gpa.free(first);
    try testing.expectError(error.OutOfMemory, build(gpa));
    try testing.expectEqual(1, fa.allocations);
}

test "arena in a test: no per-alloc frees" {
    var arena_s: std.heap.ArenaAllocator = .init(testing.allocator);
    defer arena_s.deinit();
    const arena = arena_s.allocator();

    for (0..100) |i| {
        _ = try std.fmt.allocPrint(arena, "row {d}", .{i});
    }
    try testing.expect(true);
}
```

```sh
zig test fa.zig
```

```
1/3 fa.test.failing_allocator always OOMs...OK
2/3 fa.test.fail on the 2nd allocation...OK
3/3 fa.test.arena in a test: no per-alloc frees...OK
All 3 tests passed.
```

| Tool | Behavior |
|---|---|
| `testing.failing_allocator` | a ready-made instance; allocation #0 fails |
| `FailingAllocator.init(backing, .{ .fail_index = n })` | the first `n` succeed, #`n` fails |
| `.{ .resize_fail_index = n }` | same for `resize`/`remap` |
| `fa.allocations` / `.deallocations` / `.allocated_bytes` / `.freed_bytes` | counters you can assert on |

Last test: wrapping an **arena** over `testing.allocator` is the right pattern
for a test with lots of scratch allocation. `arena_s.deinit()` frees everything
in one call, and the arena's own blocks come from `testing.allocator`, so
forgetting that `deinit` still fails the test.

## `checkAllAllocationFailures` — every OOM point at once

Rather than pick a `fail_index` by hand, this runs your function once per
allocation site, failing a different one each time, and verifies that **each
failure path frees everything it had allocated**.

```zig
const std = @import("std");
const testing = std.testing;

fn pair(gpa: std.mem.Allocator, a: []const u8, b: []const u8) !void {
    const x = try gpa.dupe(u8, a);
    defer gpa.free(x);
    const y = try gpa.dupe(u8, b);
    defer gpa.free(y);
}

fn leakyPair(gpa: std.mem.Allocator, a: []const u8, b: []const u8) !void {
    const x = try gpa.dupe(u8, a);
    const y = try gpa.dupe(u8, b);
    gpa.free(x);
    gpa.free(y);
}

test "pair survives every OOM point" {
    try testing.checkAllAllocationFailures(testing.allocator, pair, .{ "hello", "world" });
}

test "leakyPair does not" {
    try testing.checkAllAllocationFailures(testing.allocator, leakyPair, .{ "hello", "world" });
}
```

```sh
zig test c1.zig
```

**The second failure is deliberate:**

```
1/2 c1.test.pair survives every OOM point...OK
2/2 c1.test.leakyPair does not...
fail_index: 1/2
allocated bytes: 5
freed bytes: 0
allocations: 1
deallocations: 0
allocation that was made to fail: 
/opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/mem/Allocator.zig:481:40: 0x1004d6a53 in dupe__func_117 (test)
    const new_buf = try allocator.alloc(T, m.len);
                                       ^
/path/c1.zig:13:27: 0x1005ef43f in leakyPair (test)
    const y = try gpa.dupe(u8, b);
                          ^
...
1 passed; 0 skipped; 1 failed.
1 errors were logged.
1 tests leaked memory.
error: the following test command failed with exit code 1:
```

Both functions are correct on the happy path and both pass an ordinary leak
test. `leakyPair` leaks only when the **second** `dupe` OOMs: `x` is already
allocated and the `try` returns past both `free`s. `fail_index: 1/2` names the
allocation it sabotaged, `allocated bytes: 5 / freed bytes: 0` is the leak.

Requirements, enforced at compile time:

| Requirement | Else |
|---|---|
| returns `!void` | `error: Return type must be !void` |
| first parameter is `std.mem.Allocator` | `error: The provided function must have an mem.Allocator as its first argument` |
| extra args passed as a tuple matching the rest of the signature | type error |
| allocation count is **deterministic** across runs | `error.NondeterministicMemoryUsage` |

Other errors it can return:

- `error.SwallowedOutOfMemoryError` — your function caught `OutOfMemory` and
  returned success anyway.
- `error.MemoryLeakDetected` — an OOM path leaked (the case above).

**Use it on anything with more than one allocation that returns an error
union.** `defer`/`errdefer` discipline from module 5 is exactly what it checks,
and it checks it on paths you can't otherwise reach.

## Leak checking survives release modes — unlike `DebugAllocator`

`DebugAllocator`'s leak detection silently disappears in `-Ofast`/`-Osmall`
(lesson 9.5). `testing.allocator` does **not**:

```zig
const std = @import("std");
const builtin = @import("builtin");

test "leaks on purpose" {
    const buf = try std.testing.allocator.alloc(u8, 64);
    _ = buf;
    try std.testing.expect(builtin.mode == builtin.mode);
}
```

```sh
for m in debug safe fast small; do zig test -O$m lt.zig; done
```

All four fail, **deliberately**. Only the trace quality degrades:

| `-O` | Verdict | Trace |
|---|---|---|
| `debug` | exit 1, `1 tests leaked memory.` | full source lines |
| `safe` | exit 1, `1 tests leaked memory.` | `(empty stack trace)` |
| `fast` | exit 1, `1 tests leaked memory.` | `(empty stack trace)` |
| `small` | exit 1, `1 tests leaked memory.` | `Cannot print stack trace: stack tracing is disabled` |

So `zig test -Ofast` still catches leaks — but run tests in Debug when you want
the trace that tells you *where*.

## Checklist

| Situation | Do |
|---|---|
| a function allocates and returns memory | test with `testing.allocator` + `defer free` |
| a function allocates internally and frees before returning | test with `testing.allocator`; nothing else needed |
| more than one allocation + an error path | add `checkAllAllocationFailures` |
| you need to see `error.OutOfMemory` handled | `testing.failing_allocator` |
| a specific allocation must fail | `FailingAllocator.init(..., .{ .fail_index = n })` |
| the test itself needs lots of scratch memory | arena over `testing.allocator`, `defer arena_s.deinit()` |
| checking the result in CI | the **exit code** — never the summary line |

---

Next: [9.4 — Debugging](./04-debugging.md)
