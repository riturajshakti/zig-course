# 10.5 — Cross-Compilation

Zig's headline feature. Every Zig installation is a cross-compiler for every
target it supports — no SDK, no sysroot, no `brew install
aarch64-linux-gnu-gcc`, no Docker. One flag.

Everything below ran on a **macOS arm64 (Apple M4)** host.

**`src/main.zig`**

```zig
const std = @import("std");
const builtin = @import("builtin");

pub fn main() void {
    std.debug.print("{t}-{t}-{t}\n", .{ builtin.cpu.arch, builtin.os.tag, builtin.abi });
}
```

```sh
zig build-exe src/main.zig -femit-bin=out/native && ./out/native
```

```
aarch64-macos-none
```

Now seven foreign targets, back to back:

```sh
for t in x86_64-linux-musl x86_64-linux-gnu aarch64-linux-musl x86_64-windows \
         aarch64-macos riscv64-linux-musl wasm32-wasi; do
  zig build-exe src/main.zig -target $t -femit-bin=out/demo-$t
done
file out/demo-*
```

```
demo-aarch64-linux-musl:   ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), statically linked, with debug_info, not stripped
demo-aarch64-macos:        Mach-O 64-bit executable arm64
demo-riscv64-linux-musl:   ELF 64-bit LSB executable, UCB RISC-V, RVC, double-float ABI, version 1 (SYSV), statically linked, with debug_info, not stripped
demo-wasm32-wasi:          WebAssembly (wasm) binary module version 0x1 (MVP)
demo-x86_64-linux-gnu:     ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, with debug_info, not stripped
demo-x86_64-linux-musl:    ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, with debug_info, not stripped
demo-x86_64-windows:       PE32+ executable (console) x86-64, for MS Windows
demo-x86_64-windows.pdb:   MSVC program database ver 7.00, 4096*719 bytes
```

Seven targets, four architectures, four executable formats, zero toolchain
installs. The
Windows build also dropped a `.pdb` symbol file alongside the `.exe`.

## The target triple

```
x86_64 - linux   - musl
  │        │        │
 arch     os      abi
```

An unparseable component is rejected before anything compiles. The two
front-ends word it differently:

```sh
zig build-exe src/main.zig -target x86_64-banana
```

```
error: unable to parse target query 'x86_64-banana': UnknownOperatingSystem
```

```sh
zig build -Dtarget=x86_64-banana
```

```
unknown OS: "banana"
available operating systems:
 freestanding
 other
```

The `-Dtarget=` form goes on to print the whole valid set — that error message
is faster than the docs.

Optional version ranges attach to the OS and glibc:

| Triple | Means |
|---|---|
| `x86_64-linux-musl` | any Linux kernel, musl libc |
| `x86_64-linux-gnu.2.28` | glibc **2.28 or newer** — the ABI floor you link against |
| `aarch64-macos.13` | macOS 13+ |
| `native-linux-musl` | this machine's CPU, but Linux + musl |
| `x86_64-linux` | abi omitted → the OS default |
| `native` | everything from the host |

## `zig targets`

```sh
zig targets | wc -l
```

```
   35017
```

35,000 lines. In 0.17.0 it prints a **ZON literal**, not JSON:

```sh
zig targets | head -5
```

```
.{
    .arch = .{
        "aarch64",
        "aarch64_be",
        "alpha",
```

Top-level keys and their sizes on this install:

| Key | Count | Is |
|---|---|---|
| `.arch` | 61 | CPU architectures |
| `.os` | 48 | operating systems |
| `.abi` | 31 | ABIs |
| `.libc` | 108 | triples with a **bundled libc** — cross-compile with libc, no sysroot |
| `.glibc` | — | glibc versions you can target |
| `.cpus` | — | named CPU models per arch |
| `.cpu_features` | — | per-CPU feature flags (this is the 29,000 lines) |
| `.native` | — | what *this* machine is |

```sh
zig targets | sed -n '/^    .abi = /,/^    },/p' | tr -d ' "' | tr '\n' ' '
```

```
.abi=.{ none, gnu, gnuabin32, gnuabi64, gnueabi, gnueabihf, gnuf32, gnusf, gnux32, eabi, eabihf, abin32, x32, ilp32, android, androideabi, musl, muslabin32, muslabi64, musleabi, musleabihf, muslf32, muslsf, muslx32, msvc, itanium, simulator, ohos, ohoseabi, call0, }
```

```sh
zig targets | sed -n '/^    .os = /,/^    },/p' | tr -d ' "' | tr '\n' ' '
```

```
.os=.{ freestanding, other, contiki, fuchsia, hermit, managarm, haiku, hurd, illumos, linux, plan9, rtems, serenity, dragonfly, freebsd, netbsd, openbsd, driverkit, ios, maccatalyst, macos, tvos, visionos, watchos, windows, uefi, 3ds, wiiu, switch, gba, psx, ps3, ps4, ps5, psp, vita, emscripten, wasi, amdhsa, amdpal, cuda, mesa3d, nvcl, opencl, opengl, vulkan, tios, ashetos, }
```

Pipe it to a file first — `zig targets | grep -q` kills the pipe and you get
`error: WriteFailed`:

```sh
zig targets > /tmp/tg.zon
grep -c '"riscv64"' /tmp/tg.zon
```

```
1
```

```sh
sed -n '/^    .libc = /,/^    },/p' /tmp/tg.zon | grep -c "aarch64-linux-musl"
```

```
1
```

A hit in `.libc` means Zig can build that triple **with libc**, no sysroot. The
glibc versions you can pin against:

```sh
sed -n '/^    .glibc = /,/^    },/p' /tmp/tg.zon | sed -n '2,6p' | tr -d ' "'
```

```
2.0.0,
2.1.0,
2.1.1,
2.1.2,
2.1.3,
```

`.native` is the useful bit when you want to know what `native` resolves to:

```sh
zig targets | sed -n '/^    .native = /,+6p'
```

```
    .native = .{
        .triple = "aarch64-macos.26.5.2...26.5.2-none",
        .cpu = .{
            .arch = "aarch64",
            .name = "apple_m4",
            .features = .{
                "aes",
```

## Why no toolchain install

Cross-compiling C normally needs a cross `gcc`, a cross `ld`, and the target's
headers and libc. Zig ships all four:

| Normally | Zig ships |
|---|---|
| cross compiler | LLVM, which is a cross-compiler by nature |
| cross linker | `zig ld` — its own linker, for ELF/Mach-O/PE/wasm |
| libc headers | the headers for every supported libc, in `lib/zig/libc/` |
| libc binary | musl, glibc stubs, mingw-w64 — **built from source on demand** |
| `libgcc` | `compiler_rt`, compiled from Zig source for the target |

The `.libc` list above — 108 triples — is the set it can do this for.

```sh
ls /opt/homebrew/Cellar/zig/0.17.0/lib/zig | head
```

The pieces are in your Zig install. That is why the download is ~50 MB and why
the first cross-build to a new target is slow (it compiles libc) and every one
after is cached.

## musl vs gnu

The programs above used `std.debug.print`, which goes straight to syscalls — no
libc at all, hence "statically linked" even for `-gnu`. Add `-lc` and the
difference appears:

```sh
zig build-exe src/main.zig -target x86_64-linux-gnu -lc -femit-bin=out/gnu-libc
file out/gnu-libc
```

```
out/gnu-libc: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, with debug_info, not stripped
```

```sh
zig build-exe src/main.zig -target x86_64-linux-musl -lc -femit-bin=out/musl-libc
file out/musl-libc
```

```
out/musl-libc: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, with debug_info, not stripped
```

| | `-gnu` + `-lc` | `-musl` + `-lc` |
|---|---|---|
| linkage | **dynamically** linked | **statically** linked |
| needs on target | `ld-linux-x86-64.so.2` + matching glibc | nothing |
| runs on | glibc ≥ the version you targeted | any Linux kernel, any distro |
| Alpine / scratch / distroless | no | yes |
| glibc-only extensions | available | some missing |

**For shipping, `-musl` is the answer.** A musl static binary runs on Alpine,
Ubuntu, CentOS 7 and a `FROM scratch` container, with no "GLIBC_2.34 not found"
at 3am. Choose `-gnu` only when you must link a glibc-only system library.

Zig defaults to static for musl, but you can force it either way:

```sh
zig build-exe src/main.zig -target x86_64-linux-musl -lc -fPIE -dynamic -femit-bin=out/d1
file out/d1
```

```
out/d1: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib/ld-musl-x86_64.so.1, with debug_info, not stripped
```

```sh
zig build-exe src/main.zig -target x86_64-linux-gnu.2.17 -lc -femit-bin=out/d2
file out/d2
```

```
out/d2: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, with debug_info, not stripped
```

That second one links against a glibc **2.17** floor — CentOS 7 vintage — which
is the trick for shipping one dynamic binary to ancient distros.

## Through `build.zig`

`standardTargetOptions` already wired `-Dtarget=` for you.

**`build.zig`**

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "whereami",
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            .target = target,
            .optimize = optimize,
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
const builtin = @import("builtin");

pub fn main() void {
    std.debug.print("built for {t}-{t}-{t}\n", .{ builtin.cpu.arch, builtin.os.tag, builtin.abi });
}
```

```sh
zig build run
```

```
built for aarch64-macos-none
```

```sh
zig build -Dtarget=x86_64-linux-musl && file zig-out/bin/whereami
```

```
zig-out/bin/whereami: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, with debug_info, not stripped
```

```sh
zig build -Dtarget=x86_64-windows && file zig-out/bin/whereami.exe
```

```
zig-out/bin/whereami.exe: PE32+ executable (console) x86-64, for MS Windows
```

> **`zig-out/` is not cleaned between targets.** After those two commands the
> directory holds `whereami` (Linux ELF), `whereami.exe` and `whereami.pdb`.
> `rm -rf zig-out` between targets, or install each into its own subdirectory —
> see 10.6.

### `-Dtarget` vs `-target`

| | `zig build-exe -target T` | `zig build -Dtarget=T` |
|---|---|---|
| who parses it | the compiler | your `standardTargetOptions` call |
| available when | always | only if `build.zig` calls `standardTargetOptions` |
| affects | that one compilation | every module wired to that `target` value |

### Hardcoding a target in `build.zig`

For firmware or a fixed deploy target, skip the user option:

```zig
const target = b.resolveTargetQuery(.{
    .cpu_arch = .x86_64,
    .os_tag = .linux,
    .abi = .musl,
});
```

`resolveTargetQuery` takes a `std.Target.Query` struct and returns the same
`ResolvedTarget` that `standardTargetOptions` does. Any field you leave out
defaults to the host's.

## `-mcpu`: the portability trap

A triple names the architecture; `-mcpu` names the *model*, and that decides
which instructions the compiler may emit.

```sh
zig build-exe src/main.zig -target x86_64-linux-musl -mcpu=nosuchcpu
```

```
info: available CPUs for architecture 'x86_64':
 alderlake
 amdfam10
 arrowlake
```

| `-mcpu` | Emits | Runs on |
|---|---|---|
| omitted | `determined_by_arch_os`: **`native` for a native build, `baseline` when cross-compiling** | see below |
| `baseline` | the architecture's guaranteed minimum | **anything** with that arch |
| `native` | everything **this** machine has | machines like this one |
| `znver3`, `skylake`, `apple_m1`, … | that model's full feature set | that model and newer |
| `baseline+aes+avx2` | baseline **plus** named features | CPUs with those features |
| `znver3-avx512f` | that model **minus** named features | — |

```sh
zig build-exe src/main.zig -target x86_64-linux-musl -mcpu=baseline
zig build-exe src/main.zig -target x86_64-linux-musl -mcpu=baseline+aes+avx2
zig build-exe src/main.zig -target x86_64-linux-musl -mcpu=znver3-avx512f
```

All three succeed silently. Feature names are exact — `-avx512` is not one, and
guessing gets you the full list:

```sh
zig build-exe src/main.zig -target x86_64-linux-musl -mcpu=znver3-avx512
```

```
info: available CPU features for architecture 'x86_64':
 16bit_mode: 16-bit mode (i8086)
 32bit_mode: 32-bit mode (80386)
```

That default is the subtle part. **Cross-compiling is already safe** — a
`-target x86_64-linux-musl` build gets `baseline`. A **native** build gets
`native`, which is wrong for anything you distribute: build on a Zen 4 box and
it can die with an illegal instruction on a customer's older CPU.

So for a native release artifact, say `baseline` explicitly. Set it in
`build.zig` with the query:

```zig
const target = b.resolveTargetQuery(.{
    .cpu_arch = .x86_64,
    .os_tag = .linux,
    .abi = .musl,
    .cpu_model = .baseline,
});
```

## Running a foreign binary

`zig build run` can't execute what the host can't load:

```sh
zig build run -Dtarget=x86_64-linux-musl
```

```
run
+- run exe whereami failure
error: the host system (aarch64-macos.26.5.2...26.5.2-none) is unable to execute binaries from the target (x86_64-linux.5.10...7.2-musl)
failed command: ./.zig-cache/o/8ea80396facb65583dcdb2c08285b15b/whereami
```

```sh
./zig-out/bin/whereami
```

```
zsh: exec format error: ./zig-out/bin/whereami
```

The build system can bridge this if you have an emulator installed:

| Flag | Needs | Runs |
|---|---|---|
| `-fqemu` | QEMU | foreign-arch Linux binaries |
| `-fwine` | Wine | Windows binaries on Linux |
| `-fwasmtime` | wasmtime | WASI binaries |
| `-frosetta` | Rosetta 2 | x86_64 binaries on arm64 macOS |
| `-fdarling` | Darling | macOS binaries on Linux |

```sh
zig build --help
```

```
  -fdarling,  -fno-darling     Integration with system-installed Darling to
                               execute macOS programs on Linux hosts
                               (default: no)
  -fqemu,     -fno-qemu        Integration with system-installed QEMU to execute
                               foreign-architecture programs on Linux hosts
                               (default: no)
  -frosetta,  -fno-rosetta     Rely on Rosetta to execute x86_64 programs on
                               ARM64 macOS hosts. (default: no)
  -fwasmtime, -fno-wasmtime    Integration with system-installed wasmtime to
                               execute WASI binaries. (default: no)
  -fwine,     -fno-wine        Integration with system-installed Wine to execute
                               Windows programs on Linux hosts. (default: no)
```

All default to `no`, so `zig build test -Dtarget=...` for a foreign target
*compiles* the tests but cannot run them unless you opt in.

## Querying the target in source

```zig
const std = @import("std");
const builtin = @import("builtin");

pub fn main() void {
    std.debug.print("{t}-{t}-{t}\n", .{ builtin.cpu.arch, builtin.os.tag, builtin.abi });
    std.debug.print("ptr bits: {d}\n", .{@bitSizeOf(usize)});
    std.debug.print("endian:   {t}\n", .{builtin.cpu.arch.endian()});

    if (builtin.os.tag == .windows) {
        std.debug.print("windows path separator\n", .{});
    } else {
        std.debug.print("posix path separator\n", .{});
    }
}
```

```sh
zig run src/probe.zig
```

```
aarch64-macos-none
ptr bits: 64
endian:   little
posix path separator
```

`builtin` is resolved at **comptime** for the target being compiled, so that
`if` is eliminated entirely — the dead branch is not in the binary. This is how
you write cross-platform code without `#ifdef`.

| Query | Gives |
|---|---|
| `builtin.cpu.arch` | `.aarch64`, `.x86_64`, … |
| `builtin.os.tag` | `.macos`, `.linux`, `.windows`, … |
| `builtin.abi` | `.musl`, `.gnu`, `.msvc`, `.none`, … |
| `builtin.cpu.arch.endian()` | `.little` / `.big` |
| `builtin.target.ptrBitWidth()` | `64` |
| `builtin.mode` | `.Debug` … `.ReleaseSmall` — `{t}` prints it lowercase (`debug`) |
| `builtin.link_libc` | `bool` — whether libc is linked |

---

Next: [10.6 — Shipping](./06-shipping.md)
