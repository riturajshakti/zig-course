# 10.4 — Dependencies

There is no registry and no `zig add`. A dependency is a URL plus a content
hash, recorded in `build.zig.zon`, and `build.zig` turns that into modules.

| | npm | cargo | Zig |
|---|---|---|---|
| manifest | `package.json` | `Cargo.toml` | `build.zig.zon` |
| lockfile | `package-lock.json` | `Cargo.lock` | **none** — the hash *is* the lock |
| registry | npmjs.com | crates.io | none — any URL |
| store | `node_modules/` | `~/.cargo/registry` | `~/.cache/zig/p` |
| add a dep | `npm i x` | `cargo add x` | `zig fetch --save <url>` |

Two files, two jobs:

- **`build.zig.zon`** — *what* to fetch and what it must hash to.
- **`build.zig`** — *how* to use it: `b.dependency(...)` then `.module(...)`.

## `build.zig.zon`

```sh
zig init
```

Stripped of its (extensive, worth reading) comments:

```zig
.{
    .name = .pinit,
    .version = "0.0.0",
    .fingerprint = 0xd8c02cbaf60c99ae,
    .minimum_zig_version = "0.17.0",
    .dependencies = .{},
    .paths = .{
        "build.zig",
        "build.zig.zon",
        "src",
    },
}
```

| Field | Is | Notes |
|---|---|---|
| `.name` | an **enum literal** — `.pinit`, not `"pinit"` | the default key consumers use |
| `.version` | semver string | reserved for future dedup |
| `.fingerprint` | `u64`, generated once, then **never changed** | globally unique package id |
| `.minimum_zig_version` | semver string | optional |
| `.dependencies` | a struct of named deps | each is `url`+`hash`, or `path` |
| `.paths` | files included in *your* hash | `""` means the whole build root |

> **Not a string.** `.name = "mypkg"` was the 0.13 spelling. On 0.17.0 it is an
> enum literal: `.name = .mypkg`.

### `.fingerprint` is validated

You cannot make one up. Its low 32 bits are derived from `.name`:

```zig
.{
    .name = .pdep,
    .version = "0.1.0",
    .fingerprint = 0x1a2b3c4d5e6f7081,
    .minimum_zig_version = "0.17.0",
    .dependencies = .{},
    .paths = .{ "build.zig", "build.zig.zon", "src" },
}
```

```sh
zig build
```

```
build.zig.zon:1:2: error: invalid fingerprint: 0x1a2b3c4d5e6f7081; if this is a new or forked package, use this value: 0x4d2af2a20a1f3f65
.{
 ^
```

Paste the suggested value and it builds. The high 32 bits are random, so the
suggestion differs every time you ask — which is the point: when you fork a
package, delete the field, build once, and take the new value, so your fork
cannot impersonate upstream.

## `zig fetch --save`

```sh
zig fetch --save git+https://github.com/Hejsil/zig-clap
```

```
info: resolved to commit 05faf3905e8548f5cc269a8836e154065e70128d
```

`build.zig.zon` is now:

```zig
.{
    .name = .pdep,
    .version = "0.1.0",
    .fingerprint = 0x4d2af2a20a1f3f65,
    .minimum_zig_version = "0.17.0",
    .dependencies = .{
        .clap = .{
            .url = "git+https://github.com/Hejsil/zig-clap#05faf3905e8548f5cc269a8836e154065e70128d",
            .hash = "clap-0.12.0-oBajB4XpAQCF1WRKnD5KOKPsnwwT_NNdAJ7Ww3irYJac",
        },
    },
    .paths = .{ "build.zig", "build.zig.zon", "src" },
}
```

`zig fetch` did four things: downloaded the tarball, computed its hash, stored it
in the global cache, and **rewrote your manifest** — pinning the floating branch
to a commit SHA, and keying it under `clap` (the upstream package's `.name`, not
the repo name `zig-clap`).

Without `--save` it fetches and prints the hash on stdout, changing nothing:

```sh
zig fetch git+https://github.com/Hejsil/zig-clap#05faf3905e8548f5cc269a8836e154065e70128d
```

```
clap-0.12.0-oBajB4XpAQCF1WRKnD5KOKPsnwwT_NNdAJ7Ww3irYJac
```

```sh
zig fetch --help
```

```
Usage: zig fetch [options] <url>
Usage: zig fetch [options] <path>

    Copy a package into the global cache and print its hash.
    <url> must point to one of the following:
      - A git+http / git+https server for the package
      - A tarball file (with or without compression) containing
        package source
      - A git bundle file containing package source

Examples:

  zig fetch --save git+https://example.com/andrewrk/fun-example-tool.git
  zig fetch --save https://example.com/andrewrk/fun-example-tool/archive/refs/heads/master.tar.gz

Options:
  -h, --help                    Print this help and exit
  --cache-dir [path]            Override path to local cache directory
  --pkg-dir [path]              Override path to local package directory
  --debug-hash                  Print verbose hash information to stdout
  --debug-log [scope]           Enable printing debug/info log messages for scope
  --save                        Add the fetched package to build.zig.zon
  --save=[name]                 Add the fetched package to build.zig.zon as name
  --save-exact                  Add the fetched package to build.zig.zon, storing the URL verbatim
  --save-exact=[name]           Add the fetched package to build.zig.zon, storing the URL verbatim
```

| Variant | Effect |
|---|---|
| `zig fetch <url>` | fetch, print hash, don't touch the manifest |
| `zig fetch --save <url>` | also record it, resolving branches → commit SHA |
| `zig fetch --save=cli <url>` | record it under the key `cli` instead of the package's own name |
| `zig fetch --save-exact <url>` | record the URL **verbatim** — keeps `#main`, doesn't pin |
| `zig fetch --debug-hash <url>` | print the per-file hash breakdown |

`--save` needs a `build.zig.zon` to write to. From a bare directory:

```sh
zig fetch --save https://example.com/pkg.tar.gz
```

```
info: initialize build.zig template file with "zig init"
info: see "zig --help" for more options
error: no build.zig file found, in the current directory or any parent directories
```

A tarball URL works the same way:

```sh
zig fetch --save https://github.com/karlseguin/http.zig/archive/refs/heads/master.tar.gz
```

```zig
.dependencies = .{
    .httpz = .{
        .url = "https://github.com/karlseguin/http.zig/archive/refs/heads/master.tar.gz",
        .hash = "httpz-0.0.0-PNVzrEg0CQD-n7jqECTtPVfTxog5lp6bOCc0fQggnJ5t",
    },
},
```

A branch tarball has no commit to pin to, so the URL stays floating — the
`hash` is what protects you.

## Hashes

```
clap-0.12.0-oBajB4XpAQCF1WRKnD5KOKPsnwwT_NNdAJ7Ww3irYJac
└┬─┘ └─┬──┘ └──────────────────┬────────────────────────┘
name  version       base64 of the content hash
```

Computed over the *unpacked* directory, filtered by the dependency's own
`.paths`. It is the source of truth; `url` is just a mirror hint. Lie about it:

```zig
.hash = "clap-0.12.0-AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA",
```

```sh
zig build
```

```
./build.zig.zon:9:21: error: hash mismatch: manifest declares clap-0.12.0-AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA but the fetched package has clap-0.12.0-oBajB4XpAQCF1WRKnD5KOKPsnwwT_NNdAJ7Ww3irYJac
            .hash = "clap-0.12.0-AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA",
                    ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

This is why there is no lockfile: the manifest already pins exact content. When
you bump a dependency's URL, **delete its `hash`** — keeping the old one asserts
you expect the old bytes at the new address.

### Where packages land

```sh
ls ~/.cache/zig/p
```

```
clap-0.12.0-oBajB4XpAQCF1WRKnD5KOKPsnwwT_NNdAJ7Ww3irYJac.tar.gz
httpz-0.0.0-PNVzrEg0CQD-n7jqECTtPVfTxog5lp6bOCc0fQggnJ5t.tar.gz
```

Global, hash-named, shared across every project on the machine, and **stored as
tarballs** on 0.17.0 (older versions unpacked them into directories). There is
no per-project `node_modules`.

```sh
zig build --fetch
```

Fetches everything recursively and exits. After that `zig build` needs no
network. `--system <pkgdir>` disables fetching entirely — that is the flag
distro packagers use.

## Using it: `b.dependency` → `.module`

**`build.zig`**

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const clap_dep = b.dependency("clap", .{ .target = target, .optimize = optimize });

    const exe = b.addExecutable(.{
        .name = "pdep",
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            .target = target,
            .optimize = optimize,
            .imports = &.{
                .{ .name = "clap", .module = clap_dep.module("clap") },
            },
        }),
    });
    b.installArtifact(exe);

    const run_step = b.step("run", "Run the app");
    run_step.dependOn(&b.addRunArtifact(exe).step);
}
```

**`src/main.zig`**

```zig
const std = @import("std");
const clap = @import("clap");

pub fn main() void {
    std.debug.print("clap module imported OK: {any}\n", .{@TypeOf(clap.parseParamsComptime)});
}
```

**`build.zig.zon`**

```zig
.{
    .name = .pdep,
    .version = "0.1.0",
    .fingerprint = 0x4d2af2a20a1f3f65,
    .minimum_zig_version = "0.17.0",
    .dependencies = .{
        .clap = .{
            .url = "git+https://github.com/Hejsil/zig-clap#05faf3905e8548f5cc269a8836e154065e70128d",
            .hash = "clap-0.12.0-oBajB4XpAQCF1WRKnD5KOKPsnwwT_NNdAJ7Ww3irYJac",
        },
    },
    .paths = .{ "build.zig", "build.zig.zon", "src" },
}
```

```sh
zig build run
```

```
clap module imported OK: fn (comptime []const u8) anytype
```

Three names, three different things:

| Name | Where it comes from |
|---|---|
| `b.dependency("clap", ...)` | the key in **your** `.dependencies` |
| `.module("clap")` | a name the dependency passed to its own `b.addModule` |
| `@import("clap")` | the `.name` **you** chose in `.imports` |

They match here by convention, not by rule. Rename any of them freely.

The second argument to `b.dependency` is the dependency's **own** `-D` options.
`.target` and `.optimize` get forwarded so the dependency compiles for the same
target as your code; add anything else its `build.zig` declares:

```zig
const dep = b.dependency("somelib", .{
    .target = target,
    .optimize = optimize,
    .enable_tls = true,
});
```

### Two errors worth recognizing

Asking for a key that isn't in the manifest:

```sh
zig build
```

```
info: all dependencies used by build.zig must be declared in corresponding build.zig.zon
panic: no dependency named nope
/opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/Build.zig:1907:35: 0x1029a1e9f in findPkgHashOrFatal (configurer)
    if (b.pkg_hash.len == 0) panic("no dependency named {s}", .{name});
                                  ^
```

Asking a real dependency for a module it doesn't expose:

```sh
zig build
```

```
panic: unable to find module "nonexistent"
/opt/homebrew/Cellar/zig/0.17.0/lib/zig/std/Build.zig:1876:18: 0x104609a0f in module (configurer)
            panic("unable to find module {q}", .{name});
                 ^
```

Both are `panic`s from the configure phase, with a stack trace into
`std/Build.zig` — the build runner **is** a Zig program, and a mistake in
`build()` crashes it like any other. The hex addresses are ASLR-randomized, so
yours will differ from the ones above. When you hit the
second one, read the dependency's `build.zig` and look for its `addModule` calls
— that is its public API.

## Vendoring

`.path` replaces `url`+`hash` with a directory in your repo. No network, no
hash, no cache. Four files:

**`build.zig.zon`**

```zig
.{
    .name = .pvendor,
    .version = "0.1.0",
    .fingerprint = 0x35e49d001df58c1a,
    .minimum_zig_version = "0.17.0",
    .dependencies = .{
        .greet = .{ .path = "vendor/greet" },
    },
    .paths = .{ "build.zig", "build.zig.zon", "src", "vendor" },
}
```

**`build.zig`**

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const greet = b.dependency("greet", .{});

    const exe = b.addExecutable(.{
        .name = "pvendor",
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            .target = target,
            .optimize = optimize,
            .imports = &.{.{ .name = "greet", .module = greet.module("greet") }},
        }),
    });
    b.installArtifact(exe);
    const run_step = b.step("run", "Run");
    run_step.dependOn(&b.addRunArtifact(exe).step);
}
```

**`src/main.zig`**

```zig
const std = @import("std");
const greet = @import("greet");

pub fn main() void {
    std.debug.print("{s}\n", .{greet.hello("zig")});
}
```

**`vendor/greet/build.zig.zon`**

```zig
.{
    .name = .greet,
    .version = "1.0.0",
    .fingerprint = 0x2df8175b4ba2c403,
    .minimum_zig_version = "0.17.0",
    .paths = .{ "build.zig", "build.zig.zon", "src" },
}
```

**`vendor/greet/build.zig`**

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    _ = b.addModule("greet", .{
        .root_source_file = b.path("src/greet.zig"),
    });
}
```

**`vendor/greet/src/greet.zig`**

```zig
pub fn hello(name: []const u8) []const u8 {
    _ = name;
    return "hi";
}
```

```sh
zig build run
```

```
hi
```

Note what the vendored package needs: its **own** `build.zig.zon` (with its own
valid fingerprint) and a `build.zig` whose only job is `addModule`. A directory
of `.zig` files is not a package.

| | Fetching (`url`+`hash`) | Vendoring (`path`) |
|---|---|---|
| network at build time | first build only | never |
| upgrade | `zig fetch --save <new url>` | copy the new tree in |
| audit the diff | read a tarball in the global cache | it's in your git history |
| repo size | tiny | full dependency source |
| supply-chain risk | pinned by hash | pinned by your repo |
| multiple projects | shared cache entry | duplicated per repo |

Zig makes vendoring unusually painless because the dependency's `build.zig`
works identically either way. Vendor when you need an air-gapped or fully
reproducible build; fetch otherwise.

## `.lazy` — fetch only if actually used

Mark a dependency `.lazy = true` and it is not downloaded unless some step the
user asked for needs it. This is how you ship an optional 200 MB test corpus
without making every consumer download it.

**`build.zig.zon`**

```zig
.{
    .name = .plazy,
    .version = "0.1.0",
    .fingerprint = 0xfb1ca69a54e8b0c9,
    .minimum_zig_version = "0.17.0",
    .dependencies = .{
        .clap = .{
            .url = "git+https://github.com/Hejsil/zig-clap#05faf3905e8548f5cc269a8836e154065e70128d",
            .hash = "clap-0.12.0-oBajB4XpAQCF1WRKnD5KOKPsnwwT_NNdAJ7Ww3irYJac",
            .lazy = true,
        },
    },
    .paths = .{ "build.zig", "build.zig.zon", "src" },
}
```

**`build.zig`** — note `!void`:

```zig
const std = @import("std");

pub fn build(b: *std.Build) !void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "plazy",
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            .target = target,
            .optimize = optimize,
        }),
    });
    b.installArtifact(exe);

    const extra_step = b.step("extra", "Build the CLI variant, fetching clap on demand");
    const dep = try b.dependencyLazy("clap", .{ .target = target, .optimize = optimize });
    const cli = b.addExecutable(.{
        .name = "plazy-cli",
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            .target = target,
            .optimize = optimize,
            .imports = &.{.{ .name = "clap", .module = dep.module("clap") }},
        }),
    });
    extra_step.dependOn(&b.addInstallArtifact(cli, .{}).step);
}
```

**`src/main.zig`**

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("main built, lazy dep untouched\n", .{});
}
```

```sh
zig build && ls zig-out/bin
```

```
plazy
```

```sh
zig build extra --summary all && ls zig-out/bin
```

```
Build Summary: 3/3 steps succeeded
extra success
+- install plazy-cli success
   +- compile exe plazy-cli debug native success 1s MaxRSS:239M

plazy
plazy-cli
```

`dependencyLazy` returns `error.LazyDependencyNeeded` when the package isn't
fetched yet. Propagate it with `try` — that is why `build` is `!void`. The
runner catches it, fetches everything that turned out to be needed, recompiles
`build.zig`, and runs `build()` again. The `try` is not error handling; it is
how you tell the runner "stop configuring, go fetch".

> `b.lazyDependency(name, args) ?*Dependency` (returning null instead) still
> exists but is marked **Deprecated in favor of `dependencyLazy`** in
> `std/Build.zig`. Older tutorials use the `if (b.lazyDependency(...)) |dep|`
> form.

| `Dependency` method | Returns |
|---|---|
| `.module("name")` | `*Module` — one of its `addModule` modules |
| `.artifact("name")` | `*Step.Compile` — one of its executables/libraries |
| `.path("sub/path")` | `LazyPath` into its source tree |
| `.namedLazyPath("name")` | `LazyPath` it exposed via `addNamedLazyPath` |
| `.namedWriteFiles("name")` | `*Step.WriteFile` it exposed |

## `.paths` is a security boundary

`.paths` decides what goes into your package's hash *and* what survives on a
consumer's disk. Files you omit do not exist downstream:

```zig
.paths = .{
    "build.zig",
    "build.zig.zon",
    "src",
    "LICENSE",
},
```

List source needed for compilation plus licenses. Omit tests, CI config, docs,
benchmarks and assets — every byte you include is a byte every consumer fetches
and hashes. `""` includes the entire build root, which is almost never what you
want.

---

Next: [10.5 — Cross-Compilation](./05-cross-compilation.md)
