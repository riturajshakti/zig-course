# Module 10 — Build System & Cross-Compilation

Two genuinely new ideas here, and they're worth the time:

**`build.zig` is a program, not a config file.** It runs, builds a *graph of
steps*, and exits — then a separate runner walks that graph and decides what
executes, in what order, and in parallel. Nothing compiles while `build()` is
running. `npm`/`cargo`/`go build` all parse declarative data; Zig executes code
that emits a DAG. Once that clicks, custom steps, build-time codegen and
multi-target releases stop being tricks.

**Cross-compilation is one flag.** `zig build-exe -target x86_64-linux-musl`
produces a static Linux binary from a macOS laptop with nothing installed. Zig
ships the linker, the libc headers and the libc sources for 108 target triples,
so there is no SDK, no sysroot, no `brew install aarch64-...-gcc`, and no
Docker-just-to-build.

> **0.17.0 changed `build.zig`'s shape.** Every tutorial you find online will
> fail to compile. The three that will bite you:
>
> | Tutorials say | Zig 0.17.0 |
> |---|---|
> | `addExecutable(.{ .name, .root_source_file, .target, .optimize })` | `addExecutable(.{ .name, .root_module = b.createModule(.{...}) })` |
> | `if (b.args) \|args\| run.addArgs(args)` | `run.addPassthruArgs()` |
> | `.name = "mypkg"` in `build.zig.zon` | `.name = .mypkg` — an **enum literal** |
>
> Also new: `.fingerprint` in `build.zig.zon` is validated against `.name`,
> `zig targets` prints ZON rather than JSON, and the build graph's
> *construction* is cached (so `build()` won't even re-run on an unchanged
> second `zig build`).

## Lessons

| # | Lesson | What you get |
|---|---|---|
| 10.1 | [build.zig](./01-build-zig.md) | the declare-then-run graph model **proved** with prints, `addExecutable` + `createModule`, `installArtifact`, `.zig-cache` vs `zig-out/bin`, `--summary all` as a graph viewer, the generated `--help` |
| 10.2 | [Steps & Options](./02-steps-options.md) | `dependOn` is the only ordering (independent steps genuinely race), `addPassthruArgs`, a `test` step and a real failure report, `b.option` + `-Dname=value`, `b.addOptions` → `@import("config")`, aggregator steps, `b.default_step` |
| 10.3 | [Modules](./03-modules.md) | file `@import` vs module `@import`, `addModule` vs `createModule`, `addImport` and renaming, `root_module`, both failure messages, and **why a test block in a sub-file silently never runs** |
| 10.4 | [Dependencies](./04-dependencies.md) | `build.zig.zon` field by field, the validated `.fingerprint`, `zig fetch --save` against a real repo, how hashes lock without a lockfile, `b.dependency` → `.module`, vendoring with `.path`, `.lazy` + `dependencyLazy` |
| 10.5 | [Cross-Compilation](./05-cross-compilation.md) | **7 targets from one macOS host**, target triples, a slice of the 35,000-line `zig targets`, why no toolchain install is needed, musl vs gnu (static vs dynamic, measured), `-Dtarget=`, `resolveTargetQuery`, comptime `builtin` probes |
| 10.6 | [Shipping](./06-shipping.md) | the 4 modes with **measured sizes** (stripped and not), `--release=` shorthand, where `build-exe` and `installArtifact` disagree on stripping, a 131 KB static Linux binary, five targets in one `build.zig` loop, `--summary` modes, `FROM scratch` |

Every `build.zig` and every program is complete. Copy the files into a
directory, run the command shown, and you get the output shown.

## The interview answer

> `build.zig` declares a graph: `createModule` describes a compilation unit,
> `addExecutable` makes a node from it, `installArtifact` adds the node that
> copies the result to `zig-out`, and `dependOn` is the only thing that creates
> ordering — everything else runs in parallel. Dependencies are a URL plus a
> content hash in `build.zig.zon`, so the hash *is* the lockfile. For shipping
> I build `ReleaseSafe` against `-linux-musl`, which gives one stripped static
> binary that runs on any distro, cross-compiled from whatever machine CI
> happens to be — no container needed to build or to run.

---

Next: [Module 11 — C Interop](../11-c-interop/)
