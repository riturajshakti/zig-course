# 8.5 — Directories

`std.Io.Dir` is a file descriptor with methods. `std.fs.path` is pure string
manipulation and needs no `io` at all.

> **The rename that breaks every tutorial:** `makePath` is now
> **`createDirPath`**, and `mkdir` is `createDir`. Both take `io`.

## Build a tree

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const cwd = std.Io.Dir.cwd();
    const root = "/tmp/zig-course-tree";

    cwd.deleteTree(io, root) catch {};
    try cwd.createDirPath(io, root ++ "/src/util");
    try cwd.createDirPath(io, root ++ "/docs");

    try cwd.writeFile(io, .{ .sub_path = root ++ "/README.md", .data = "# demo\n" });
    try cwd.writeFile(io, .{ .sub_path = root ++ "/src/main.zig", .data = "pub fn main() void {}\n" });
    try cwd.writeFile(io, .{ .sub_path = root ++ "/src/util/str.zig", .data = "// str\n" });
    try cwd.writeFile(io, .{ .sub_path = root ++ "/docs/guide.md", .data = "guide\n" });

    var dir = try cwd.openDir(io, root, .{ .iterate = true });
    defer dir.close(io);

    std.debug.print("-- iterate (one level) --\n", .{});
    var it = dir.iterate();
    while (try it.next(io)) |entry| {
        std.debug.print("{t:<10} {s}\n", .{ entry.kind, entry.name });
    }
}
```

```
-- iterate (one level) --
directory  docs
file       README.md
directory  src
```

Three things to notice:

- **`.{ .iterate = true }` is mandatory to iterate.** Without it, iterating is
  documented illegal behavior, not an error you can catch. `cwd()` itself is not
  opened for iteration — open it explicitly if you need to walk it.
- **Order is undefined.** It is whatever the filesystem returns. Sort if order
  matters.
- **`it.next(io)`** takes `io`; `dir.iterate()` does not.
- `deleteTree(io, root) catch {}` first makes the program idempotent.

| Create | Does |
|---|---|
| `createDir(io, p, .default_dir)` | one level; `error.PathAlreadyExists` if it exists |
| `createDirPath(io, p)` | every missing parent; **succeeds if it already exists** |
| `createDirPathStatus(io, p, perms)` | same, returns `.existed` or `.created` |
| `createDirPathOpen(io, p, .{})` | creates and returns the open `Dir` |

## `walk()` — recursive

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    var dir = try std.Io.Dir.cwd().openDir(io, "/tmp/zig-course-tree", .{ .iterate = true });
    defer dir.close(io);

    var walker = try dir.walk(init.gpa);
    defer walker.deinit();

    var files: usize = 0;
    var bytes: u64 = 0;
    while (try walker.next(io)) |entry| {
        std.debug.print("{d} {t:<10} {s}\n", .{ entry.depth(), entry.kind, entry.path });
        if (entry.kind == .file) {
            files += 1;
            const st = try entry.dir.statFile(io, entry.basename, .{});
            bytes += st.size;
        }
    }
    std.debug.print("{d} files, {d} bytes\n", .{ files, bytes });
}
```

```
1 directory  docs
2 file       docs/guide.md
1 file       README.md
1 directory  src
2 directory  src/util
3 file       src/util/str.zig
2 file       src/main.zig
4 files, 42 bytes
```

| `iterate()` | `walk()` |
|---|---|
| one level | the whole subtree |
| no allocator | needs an allocator (a stack of open dirs) |
| `Entry{ name, kind, inode }` | `Entry{ dir, basename, path, kind }` + `depth()` |
| nothing to clean up | `walker.deinit()` |

`entry.path` and `entry.basename` are **invalidated by the next `next()`** —
`dupe` them if you are collecting results. And `entry.dir` is the containing
directory, so `entry.dir.statFile(io, entry.basename, .{})` avoids rebuilding
the path and avoids `error.NameTooLong` on deep trees.

### Filtering

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const gpa = init.gpa;
    var dir = try std.Io.Dir.cwd().openDir(io, "/tmp/zig-course-tree", .{ .iterate = true });
    defer dir.close(io);

    var found: std.ArrayList([]u8) = .empty;
    defer {
        for (found.items) |p| gpa.free(p);
        found.deinit(gpa);
    }

    var walker = try dir.walk(gpa);
    defer walker.deinit();
    while (try walker.next(io)) |entry| {
        if (entry.kind != .file) continue;
        if (!std.mem.eql(u8, std.fs.path.extension(entry.path), ".md")) continue;
        try found.append(gpa, try gpa.dupe(u8, entry.path));
    }

    std.mem.sort([]u8, found.items, {}, struct {
        fn lt(_: void, a: []u8, b: []u8) bool {
            return std.mem.lessThan(u8, a, b);
        }
    }.lt);

    for (found.items) |p| std.debug.print("{s}\n", .{p});
}
```

```
README.md
docs/guide.md
```

The `dupe` is why the list owns its strings; the sort is why the output is
stable across runs.

## `statFile` and existence

```zig
const std = @import("std");

fn exists(io: std.Io, path: []const u8) bool {
    std.Io.Dir.cwd().access(io, path, .{}) catch return false;
    return true;
}

fn kindOf(io: std.Io, path: []const u8) ?std.Io.File.Kind {
    const st = std.Io.Dir.cwd().statFile(io, path, .{}) catch return null;
    return st.kind;
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const paths = [_][]const u8{
        "/tmp/zig-course-tree/README.md",
        "/tmp/zig-course-tree/src",
        "/tmp/zig-course-tree/nope",
    };
    for (paths) |p| {
        std.debug.print("{s:<34} exists={} kind={?t}\n", .{ p, exists(io, p), kindOf(io, p) });
    }

    const st = try std.Io.Dir.cwd().statFile(io, "/tmp/zig-course-tree/README.md", .{});
    std.debug.print("\nsize={d} kind={t} nlink={d} block_size={d}\n", .{
        st.size, st.kind, st.nlink, st.block_size,
    });
}
```

```
/tmp/zig-course-tree/README.md     exists=true kind=file
/tmp/zig-course-tree/src           exists=true kind=directory
/tmp/zig-course-tree/nope          exists=false kind=null

size=7 kind=file nlink=1 block_size=4096
```

`Stat` also carries `inode`, `mtime`, `ctime`, `atime` and `permissions`. Those
differ on every machine and every run, so they are left out of the printout
above — `st.mtime.nanoseconds` is an `i96` of nanoseconds since the Unix epoch,
and `st.atime` is `?Io.Timestamp` because some filesystems refuse to report it.

| Check | Call |
|---|---|
| does the path exist | `access(io, p, .{})` — error means no |
| is it readable / writable | `access(io, p, .{ .read = true })` etc. |
| file or directory | `statFile(io, p, .{}).kind` |
| size in bytes | `statFile(io, p, .{}).size` |
| don't follow symlinks | `statFile(io, p, .{ .follow_symlinks = false })` |

**Do not check-then-act.** `exists()` followed by `openFile()` is a race: the
file can vanish in between. Just call `openFile` and handle
`error.FileNotFound`. Use `access` only for reporting, as above.

## Deleting

```zig
const std = @import("std");

fn count(io: std.Io, root: []const u8, gpa: std.mem.Allocator) !usize {
    var dir = std.Io.Dir.cwd().openDir(io, root, .{ .iterate = true }) catch return 0;
    defer dir.close(io);
    var walker = try dir.walk(gpa);
    defer walker.deinit();
    var n: usize = 0;
    while (try walker.next(io)) |_| n += 1;
    return n;
}

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const gpa = init.gpa;
    const cwd = std.Io.Dir.cwd();
    const root = "/tmp/zig-course-del";

    cwd.deleteTree(io, root) catch {};
    try cwd.createDirPath(io, root ++ "/a/b");
    try cwd.writeFile(io, .{ .sub_path = root ++ "/a/b/leaf.txt", .data = "x" });
    try cwd.writeFile(io, .{ .sub_path = root ++ "/top.txt", .data = "y" });
    std.debug.print("built:            {d} entries\n", .{try count(io, root, gpa)});

    try cwd.deleteFile(io, root ++ "/top.txt");
    std.debug.print("after deleteFile: {d} entries\n", .{try count(io, root, gpa)});

    std.debug.print("deleteDir on non-empty -> {any}\n", .{cwd.deleteDir(io, root ++ "/a")});
    std.debug.print("deleteFile on missing  -> {any}\n", .{cwd.deleteFile(io, root ++ "/gone.txt")});

    try cwd.deleteTree(io, root);
    std.debug.print("after deleteTree: {d} entries, exists={}\n", .{
        try count(io, root, gpa),
        blk: {
            cwd.access(io, root, .{}) catch break :blk false;
            break :blk true;
        },
    });
}
```

```
built:            4 entries
after deleteFile: 3 entries
deleteDir on non-empty -> error.DirNotEmpty
deleteFile on missing  -> error.FileNotFound
after deleteTree: 0 entries, exists=false
```

| Call | Scope | On missing |
|---|---|---|
| `deleteFile(io, p)` | one file | `error.FileNotFound` |
| `deleteDir(io, p)` | one **empty** dir | `error.FileNotFound`; non-empty → `error.DirNotEmpty` |
| `deleteTree(io, p)` | everything below, recursively | **succeeds** — missing is not an error |

`deleteTree` is `rm -rf`. It does not complain about a missing path, which is
exactly why `deleteTree(io, root) catch {}` is the right way to start an
idempotent demo — and exactly why you should never build its argument from
untrusted input.

## Other operations

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const cwd = std.Io.Dir.cwd();
    const root = "/tmp/zig-course-ops";

    cwd.deleteTree(io, root) catch {};
    try cwd.createDirPath(io, root);
    try cwd.writeFile(io, .{ .sub_path = root ++ "/original.txt", .data = "payload\n" });

    try cwd.copyFile(root ++ "/original.txt", cwd, root ++ "/copy.txt", io, .{});
    try cwd.rename(root ++ "/copy.txt", cwd, root ++ "/renamed.txt", io);
    try cwd.symLink(io, "original.txt", root ++ "/link.txt", .{});

    var dir = try cwd.openDir(io, root, .{ .iterate = true });
    defer dir.close(io);
    var it = dir.iterate();
    while (try it.next(io)) |e| std.debug.print("{t:<10} {s}\n", .{ e.kind, e.name });

    var lbuf: [256]u8 = undefined;
    const n = try cwd.readLink(io, root ++ "/link.txt", &lbuf);
    std.debug.print("\nlink.txt -> {s}\n", .{lbuf[0..n]});

    const direct = try cwd.statFile(io, root ++ "/link.txt", .{ .follow_symlinks = false });
    const followed = try cwd.statFile(io, root ++ "/link.txt", .{ .follow_symlinks = true });
    std.debug.print("as symlink: {t}, followed: {t}\n", .{ direct.kind, followed.kind });
}
```

```
file       original.txt
sym_link   link.txt
file       renamed.txt

link.txt -> original.txt
as symlink: sym_link, followed: file
```

| Call | Does |
|---|---|
| `copyFile(src, dst_dir, dst, io, .{})` | copy; `io` comes **after** the paths |
| `rename(old, dst_dir, new, io)` | rename or move; `io` **last**; atomic on one filesystem |
| `symLink(io, target, link, .{})` | the target is **not** resolved or checked |
| `hardLink(io, ...)` | another name for the same inode |
| `readLink(io, p, &buf)` | where a symlink points, as a byte count |
| `realPathFileAlloc(io, p, gpa)` | fully resolved absolute path |
| `updateFile(io, src, dst_dir, dst, .{})` | copy only if size/mtime differ; returns `.stale`/`.fresh` |
| `setPermissions`, `setTimestampsNow` | metadata |

**`io` is not always the first parameter here.** `copyFile` and `rename` take it
*after* the paths, while `symLink`, `statFile`, `openDir` and almost everything
else take it second. Passing it first to `copyFile` gives
`error: expected type '[]const u8', found 'Io'`. When a call will not compile,
read the signature rather than guessing.

## `std.fs.path` — strings only

```zig
const std = @import("std");
const path = std.fs.path;

pub fn main(init: std.process.Init) !void {
    const arena = init.arena.allocator();
    const p = "/tmp/zig-course-tree/src/util/str.zig";

    std.debug.print("basename   {s}\n", .{path.basename(p)});
    std.debug.print("dirname    {?s}\n", .{path.dirname(p)});
    std.debug.print("extension  {s}\n", .{path.extension(p)});
    std.debug.print("stem       {s}\n", .{path.stem(p)});
    std.debug.print("isAbsolute {}\n", .{path.isAbsolute(p)});
    std.debug.print("sep        '{c}'\n", .{path.sep});

    const joined = try path.join(arena, &.{ "/tmp", "zig-course-tree", "docs", "guide.md" });
    std.debug.print("join       {s}\n", .{joined});

    const resolved = try path.resolveAlloc(arena, &.{ "/tmp/zig-course-tree", "src/../docs", "./guide.md" });
    std.debug.print("resolve    {s}\n", .{resolved});

    std.debug.print("fmtJoin    {f}\n", .{path.fmtJoin(&.{ "a", "b", "c.txt" })});

    var it = path.componentIterator(p);
    while (it.next()) |c| std.debug.print("  component {s}\n", .{c.name});
}
```

```
basename   str.zig
dirname    /tmp/zig-course-tree/src/util
extension  .zig
stem       str
isAbsolute true
sep        '/'
join       /tmp/zig-course-tree/docs/guide.md
resolve    /tmp/zig-course-tree/docs/guide.md
fmtJoin    a/b/c.txt
  component tmp
  component zig-course-tree
  component src
  component util
  component str.zig
```

| Function | Returns | Allocates |
|---|---|---|
| `basename(p)` | `[]const u8` — last component | no |
| `dirname(p)` | `?[]const u8` — **optional**, null at the root | no |
| `extension(p)` | `.zig`, **with** the dot; `""` if none | no |
| `stem(p)` | basename minus extension | no |
| `isAbsolute(p)` | `bool` | no |
| `join(gpa, parts)` | one path, single separators | yes |
| `fmtJoin(parts)` | a `{f}` formatter — joins with no allocator | **no** |
| `resolveAlloc(gpa, parts)` | absolute, with `.` and `..` collapsed | yes |
| `relativeAlloc(gpa, cwd, from, to)` | a relative path between two | yes |
| `componentIterator(p)` | walks components; **not** an error union | no |

`basename`/`dirname`/`extension`/`stem` are slices **into your input** — no
allocation, and they die when the input does.

Three traps: `dirname` is optional, `extension` includes the dot, and
`componentIterator` returns the iterator directly — `catch unreachable` on it is
a compile error (`expected error union type`).

---

Next: [8.6 — CLI Programs](./06-cli-programs.md)
