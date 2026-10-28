# 10.6 — Shipping

Four optimize modes, one `build.zig`, five platform binaries, no container.

## The four modes

**`src/main.zig`**

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("hello\n", .{});
}
```

**`build.zig`**

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    const strip = b.option(bool, "strip", "Strip debug symbols") orelse false;

    const exe = b.addExecutable(.{
        .name = "hello",
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            .target = target,
            .optimize = optimize,
            .strip = strip,
        }),
    });
    b.installArtifact(exe);
}
```

```sh
for m in Debug ReleaseSafe ReleaseFast ReleaseSmall; do
  rm -rf zig-out .zig-cache
  zig build -Doptimize=$m >/dev/null 2>&1
  a=$(stat -f%z zig-out/bin/hello)
  rm -rf zig-out
  zig build -Doptimize=$m -Dstrip=true >/dev/null 2>&1
  b=$(stat -f%z zig-out/bin/hello)
  printf "%-13s %9s %9s\n" "$m" "$a" "$b"
done
```

```
Debug           2001640    845656
ReleaseSafe      469032    206872
ReleaseFast      395192    190184
ReleaseSmall     372088    165080
```

A hello-world on macOS arm64, bytes, unstripped then stripped:

| Mode | Size | Stripped | Safety checks | Optimization | Use for |
|---|---:|---:|---|---|---|
| `Debug` | 2,001,640 | 845,656 | **all**, with panics + stack traces | none | development |
| `ReleaseSafe` | 469,032 | 206,872 | **all** (overflow, bounds, `unreachable`) | full | **production default** |
| `ReleaseFast` | 395,192 | 190,184 | removed → UB | full | measured hot paths |
| `ReleaseSmall` | 372,088 | 165,080 | removed → UB | for size | embedded, wasm |

Debug → ReleaseSafe is a **4.3x** size cut and the single biggest win. Going
further costs you safety for 16%:

- `ReleaseSafe` keeps integer-overflow, bounds and `unreachable` checks. They
  panic with a message instead of becoming undefined behavior.
- `ReleaseFast` / `ReleaseSmall` delete them. An overflow that panicked in
  `ReleaseSafe` silently wraps; reaching `unreachable` can crash *or hang*.
  (Leak detection in `DebugAllocator` also silently stops working — see 5.3.)

**Ship `ReleaseSafe` unless a profiler told you otherwise.**

## Spelling the mode

```sh
zig build -Doptimize=ReleaseFast --summary all
```

```
Build Summary: 4/4 steps succeeded
install success
+- install app cached
   +- compile exe app fast native cached 35ms MaxRSS:33M
      +- options cached
```

```sh
zig build --release=small --summary all
```

```
Build Summary: 3/3 steps succeeded
install success
+- install hello success
   +- compile exe hello small native cached 36ms MaxRSS:36M
```

| Form | Note |
|---|---|
| `-Doptimize=ReleaseFast` | the long spelling; works |
| `-Doptimize=fast` | the short one `--help` advertises; identical |
| `--release=fast` \| `safe` \| `small` | shorthand, no `build.zig` support needed |
| `--release` (bare) | only if `build.zig` declared a preferred mode |

The summary line always reports the short name (`debug`, `safe`, `fast`,
`small`).

```sh
zig build --release
```

```
the project does not declare a preferred optimization mode. choose: --release=fast, --release=safe, or --release=small
```

(followed by `error: configure command exited with code 1:` and the full
`configurer` command line — build-root paths and cache hashes differ per
machine.)

Declare a preference in `build()` and bare `--release` works:

```zig
const optimize = b.standardOptimizeOption(.{ .preferred_optimize_mode = .ReleaseSafe });
```

```sh
zig build --release --summary all
```

```
Build Summary: 3/3 steps succeeded
install success
+- install hello success
   +- compile exe hello safe native success 5s MaxRSS:351M
```

## Stripping

Two places to ask for it:

| Where | How |
|---|---|
| `build.zig`, per module | `.strip = true` in `createModule` |
| `build.zig`, via a flag | `b.option(bool, "strip", ...)` → `-Dstrip=true` |
| CLI, `build-exe` | `-fstrip` / `-fno-strip` |

Strip removes the symbol table and debug info: no function names in crash
dumps, no `std.debug.dumpStackTrace` output worth reading. Keep an unstripped
copy of whatever you ship.

### `build-exe` and `installArtifact` disagree

```sh
for m in Debug ReleaseSafe ReleaseFast ReleaseSmall; do
  zig build-exe src/main.zig -O $m -femit-bin=/tmp/be-$m >/dev/null 2>&1
  printf "%-13s %s\n" "$m" "$(stat -f%z /tmp/be-$m)"
done
```

```
Debug         2001624
ReleaseSafe   469024
ReleaseFast   395184
ReleaseSmall  165088
```

Compare to the install column above: `Debug`, `ReleaseSafe` and `ReleaseFast`
match within a few bytes, but `ReleaseSmall` is **165,088 via `build-exe`
vs 372,088 via `installArtifact`**. Adding `-fstrip` to the `build-exe` call
changes nothing at `ReleaseSmall` (165,088 either way) — `ReleaseSmall` already
implies stripping there, while `installArtifact` keeps the symbol table unless
you ask.

**Don't compare a `build-exe` size to a `zig-out/bin` size.** Set `.strip`
explicitly and the two agree.

## Static binaries

`musl` + no dynamic linking = one file that runs on any Linux:

```sh
zig build -Doptimize=ReleaseSmall -Dstrip=true -Dtarget=x86_64-linux-musl
file zig-out/bin/hello
```

```
zig-out/bin/hello: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, stripped
```

```sh
stat -f%z zig-out/bin/hello
```

```
134144
```

131 KB, `statically linked`, `stripped`, built on macOS. Copy it to any
x86_64 Linux box — Alpine, CentOS 7, a `FROM scratch` container, a bare
initramfs — and run it. No `ldd` surprises, no glibc version floor, nothing to
install.

| Linkage | `file` says | Deploy needs |
|---|---|---|
| static (musl) | `statically linked` | nothing |
| dynamic (gnu) | `interpreter /lib64/ld-linux-x86-64.so.2` | matching glibc on target |
| dynamic (musl) | `interpreter /lib/ld-musl-x86_64.so.1` | musl on target |

Verify before you ship:

```sh
file zig-out/bin/hello        # format, arch, static/dynamic, stripped
```

On the target machine, `ldd` should say "not a dynamic executable" for a static
build.

## Every target in one step

**`build.zig`**

```zig
const std = @import("std");

const targets: []const std.Target.Query = &.{
    .{ .cpu_arch = .x86_64, .os_tag = .linux, .abi = .musl },
    .{ .cpu_arch = .aarch64, .os_tag = .linux, .abi = .musl },
    .{ .cpu_arch = .x86_64, .os_tag = .macos },
    .{ .cpu_arch = .aarch64, .os_tag = .macos },
    .{ .cpu_arch = .x86_64, .os_tag = .windows },
};

pub fn build(b: *std.Build) void {
    const release = b.step("release", "Build every shipping target");

    for (targets) |t| {
        const resolved = b.resolveTargetQuery(t);
        const exe = b.addExecutable(.{
            .name = "hello",
            .root_module = b.createModule(.{
                .root_source_file = b.path("src/main.zig"),
                .target = resolved,
                .optimize = .ReleaseSafe,
                .strip = true,
            }),
        });
        const install = b.addInstallArtifact(exe, .{
            .dest_dir = .{ .override = .{ .custom = t.zigTriple(b.allocator) catch @panic("OOM") } },
        });
        release.dependOn(&install.step);
    }

    const native = b.addExecutable(.{
        .name = "hello",
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            .target = b.standardTargetOptions(.{}),
            .optimize = b.standardOptimizeOption(.{}),
        }),
    });
    b.installArtifact(native);
    const run_step = b.step("run", "Run natively");
    run_step.dependOn(&b.addRunArtifact(native).step);
}
```

**`src/main.zig`**

```zig
const std = @import("std");
const builtin = @import("builtin");

pub fn main() void {
    std.debug.print("hello from {t}-{t}\n", .{ builtin.cpu.arch, builtin.os.tag });
}
```

```sh
zig build release --summary all
```

```
Build Summary: 11/11 steps succeeded
release success
+- install hello cached
|  +- compile exe hello safe x86_64-linux-musl cached 54ms MaxRSS:33M
+- install hello cached
|  +- compile exe hello safe aarch64-linux-musl cached 54ms MaxRSS:33M
+- install hello cached
|  +- compile exe hello safe x86_64-macos cached 54ms MaxRSS:33M
+- install hello cached
|  +- compile exe hello safe aarch64-macos cached 54ms MaxRSS:33M
+- install hello cached
   +- compile exe hello safe x86_64-windows cached 54ms MaxRSS:33M
```

Five siblings under `release`, no edges between them, so **all five compile in
parallel**.

```sh
find zig-out -type f | sort | xargs stat -f "%z %N" | sed 's|zig-out/||'
```

```
207656 aarch64-linux-musl/hello
206872 aarch64-macos/hello
298224 x86_64-linux-musl/hello
232069 x86_64-macos/hello
547840 x86_64-windows/hello.exe
```

```sh
for f in $(find zig-out -type f | sort); do echo "$f: $(file -b $f | cut -c1-50)"; done
```

```
zig-out/aarch64-linux-musl/hello: ELF 64-bit LSB executable, ARM aarch64, version 1 
zig-out/aarch64-macos/hello: Mach-O 64-bit executable arm64
zig-out/x86_64-linux-musl/hello: ELF 64-bit LSB executable, x86-64, version 1 (SYSV
zig-out/x86_64-macos/hello: Mach-O 64-bit executable x86_64
zig-out/x86_64-windows/hello.exe: PE32+ executable (console) x86-64, for MS Windows
```

```sh
zig build run
```

```
hello from aarch64-macos
```

Pieces worth noting:

| Piece | Does |
|---|---|
| `std.Target.Query` | a *partial* target — omitted fields default to the host |
| `b.resolveTargetQuery(t)` | query → `ResolvedTarget`, same type `standardTargetOptions` returns |
| `t.zigTriple(b.allocator)` | `"x86_64-linux-musl"` — the canonical triple string |
| `b.addInstallArtifact(exe, .{...})` | an install node you can configure, instead of `installArtifact`'s defaults |
| `.dest_dir = .{ .override = .{ .custom = "..." } }` | install under `zig-out/<that>/` instead of `zig-out/bin/` |

Without the `dest_dir` override, all five would install to `zig-out/bin/hello`
and clobber each other. `.exe` appears automatically for the Windows target —
the install step knows the target's conventions.

## `--summary all`

| Column | Means |
|---|---|
| `success` | the step ran this time |
| `cached` | unchanged inputs, result reused |
| `(reused)` | the same node already appeared elsewhere in this run |
| `failure` / `transitive failure` | it failed / a dependency failed |
| `1s MaxRSS:239M` | wall time and peak memory of that step |

```sh
zig build --summary all     # first run
```

```
Build Summary: 3/3 steps succeeded
install success
+- install demo success
   +- compile exe demo debug native success 1s MaxRSS:239M
```

```sh
zig build --summary all     # second run
```

```
Build Summary: 3/3 steps succeeded
install success
+- install demo cached
   +- compile exe demo debug native cached 35ms MaxRSS:35M
```

`--summary all` is how you find the step that is actually slow.

| `--summary` | Prints |
|---|---|
| `all` | the whole tree |
| `new` | omit cached steps |
| `failures` | only failed steps — the default for a one-shot build |
| `line` | just the one-line total — the default under `--watch` |
| `none` | nothing |

Two more flags worth knowing for a release build:

| Flag | Does |
|---|---|
| `-j<N>` | cap concurrent steps (default: all cores) |
| `--watch` | rebuild on file change, staying resident |
| `--test-timeout 30s` | kill a hung test instead of blocking CI |
| `--maxrss <bytes>` | cap memory; `--skip-oom-steps` skips rather than fails |

## Docker-free deploys

The usual reason a Go or Rust service ships in a container is the build
environment, not the runtime. A static Zig binary removes the reason:

```sh
zig build -Doptimize=ReleaseSafe -Dstrip=true -Dtarget=x86_64-linux-musl
scp zig-out/bin/hello server:/usr/local/bin/
```

Done. No base image, no `apt-get`, no multi-stage Dockerfile, no CI runner
matching the target OS. **The macOS laptop is the Linux build machine.**

If you still want a container, the image is the binary:

```dockerfile
FROM scratch
COPY zig-out/bin/hello /hello
ENTRYPOINT ["/hello"]
```

A ~200 KB image with no shell, no package manager and no CVE surface, because
there is nothing in it but your program.

A release step for CI — a top-level step can `dependOn` another top-level step:

```zig
const test_step = b.step("test", "Run tests");
test_step.dependOn(&b.addRunArtifact(b.addTest(.{ .root_module = native_mod })).step);

const ci = b.step("ci", "test + release artifacts");
ci.dependOn(test_step);
ci.dependOn(release);
```

```sh
zig build ci --summary all
```

```
Build Summary: 15/15 steps succeeded
ci success
+- test success
|  +- run test success 423ms MaxRSS:2M
|     +- compile test debug native success 1s MaxRSS:257M
+- release success
   +- install hello success
   |  +- compile exe hello safe x86_64-linux-musl cached 70ms MaxRSS:33M
   +- install hello success
   |  +- compile exe hello safe aarch64-linux-musl cached 71ms MaxRSS:33M
   +- install hello success
   |  +- compile exe hello safe x86_64-macos cached 73ms MaxRSS:33M
   +- install hello success
   |  +- compile exe hello safe aarch64-macos cached 74ms MaxRSS:33M
   +- install hello success
      +- compile exe hello safe x86_64-windows cached 71ms MaxRSS:33M
```

One command, any host, every target, 15 steps, most of them in parallel. That
is the whole pitch.

---

Next: [Module 11 — C Interop](../11-c-interop/)
