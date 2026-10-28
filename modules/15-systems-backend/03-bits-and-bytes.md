# 15.3 — Bits & Bytes

Reinterpreting memory with no copy and no cast operator in sight. This is the
lesson that makes a binary protocol tractable.

## Endianness and reinterpretation

```zig
const std = @import("std");

pub fn main() !void {
    const builtin = @import("builtin");
    std.debug.print("endian     = {t}\n", .{builtin.cpu.arch.endian()});
    std.debug.print("usize bits = {d}\n\n", .{@bitSizeOf(usize)});

    const v: u32 = 0x11223344;
    std.debug.print("value           = 0x{x}\n", .{v});
    std.debug.print("asBytes         = {x}   <- LSB first\n", .{std.mem.asBytes(&v).*});
    std.debug.print("byteSwap        = 0x{x}\n", .{@byteSwap(v)});
    std.debug.print("asBytes(swapped)= {x}\n\n", .{std.mem.asBytes(&@byteSwap(v)).*});

    std.debug.print("f32 1.0  bits = 0x{x}\n", .{@as(u32, @bitCast(@as(f32, 1.0)))});
    std.debug.print("f32 -2.0 bits = 0x{x}\n", .{@as(u32, @bitCast(@as(f32, -2.0)))});
    std.debug.print("f64 1.0  bits = 0x{x}\n", .{@as(u64, @bitCast(@as(f64, 1.0)))});
    std.debug.print("0x3f800000 as f32 = {d}\n\n", .{@as(f32, @bitCast(@as(u32, 0x3f800000)))});

    const wide: u32 = 0xDEADBEEF;
    std.debug.print("truncate u32->u8  = 0x{x}\n", .{@as(u8, @truncate(wide))});
    std.debug.print("truncate u32->u16 = 0x{x}\n", .{@as(u16, @truncate(wide))});
    std.debug.print("intCast  u8 ->u32 = 0x{x}\n", .{@as(u32, @intCast(@as(u8, 0xEF)))});
    std.debug.print("bitCast  i8 <-u8  = {d}\n", .{@as(i8, @bitCast(@as(u8, 0xFF)))});
}
```

```
endian     = little
usize bits = 64

value           = 0x11223344
asBytes         = 44332211   <- LSB first
byteSwap        = 0x44332211
asBytes(swapped)= 11223344

f32 1.0  bits = 0x3f800000
f32 -2.0 bits = 0xc0000000
f64 1.0  bits = 0x3ff0000000000000
0x3f800000 as f32 = 1

truncate u32->u8  = 0xef
truncate u32->u16 = 0xbeef
intCast  u8 ->u32 = 0xef
bitCast  i8 <-u8  = -1
```

**`0x11223344` in memory is `44 33 22 11`.** Little-endian: least significant
byte at the lowest address. Every x86-64 and arm64 machine you will deploy to is
little-endian; every network protocol defined before ~2000 is big-endian. That
mismatch is why `byteSwap` exists.

`@as(f32, @bitCast(0x3f800000))` printing `1` is the point of `@bitCast`: same
32 bits, different type, zero instructions.

### The four conversions, and when each is legal

| Builtin | Changes | Size must | Fails if |
|---|---|---|---|
| `@bitCast(x)` | the **type**, not one bit | **match exactly** | sizes differ (compile error) |
| `@truncate(x)` | drops high bits | shrink | never — it's defined to discard |
| `@intCast(x)` | the width, value preserved | either way | value doesn't fit (**panics** in Debug/ReleaseSafe) |
| `@byteSwap(x)` | byte order | same | never |

`@as(i8, @bitCast(@as(u8, 0xFF)))` → `-1`: the bits are identical, two's
complement reads them as negative. `@intCast` on the same value would panic.

## Bit manipulation

```zig
const std = @import("std");

pub fn main() !void {
    const x: u16 = 0b0000_1011_0100_0000;
    std.debug.print("x            = 0b{b:0>16} ({d})\n", .{ x, x });
    std.debug.print("popCount     = {d}\n", .{@popCount(x)});
    std.debug.print("clz          = {d}\n", .{@clz(x)});
    std.debug.print("ctz          = {d}\n", .{@ctz(x)});
    std.debug.print("bitReverse   = 0b{b:0>16}\n", .{@bitReverse(x)});
    std.debug.print("byteSwap     = 0b{b:0>16}\n\n", .{@byteSwap(x)});

    std.debug.print("set bit 0    = 0b{b:0>16}\n", .{x | (@as(u16, 1) << 0)});
    std.debug.print("clear bit 6  = 0b{b:0>16}\n", .{x & ~(@as(u16, 1) << 6)});
    std.debug.print("toggle bit 6 = 0b{b:0>16}\n", .{x ^ (@as(u16, 1) << 6)});
    std.debug.print("test bit 6   = {}\n", .{x & (@as(u16, 1) << 6) != 0});
    std.debug.print("test bit 5   = {}\n\n", .{x & (@as(u16, 1) << 5) != 0});

    std.debug.print("shl 2        = 0b{b:0>16}\n", .{x << 2});
    std.debug.print("shr 2        = 0b{b:0>16}\n", .{x >> 2});
    std.debug.print("low 8 bits   = 0b{b:0>8}\n", .{@as(u8, @truncate(x))});
    std.debug.print("high 8 bits  = 0b{b:0>8}\n", .{@as(u8, @truncate(x >> 8))});

    const ov = @shlWithOverflow(x, 6);
    std.debug.print("\nshlWithOverflow(x,6) = 0b{b:0>16} overflow={d}\n", .{ ov[0], ov[1] });
    std.debug.print("rotl 4  = 0b{b:0>16}\n", .{std.math.rotl(u16, x, 4)});
    std.debug.print("rotr 4  = 0b{b:0>16}\n", .{std.math.rotr(u16, x, 4)});
}
```

```
x            = 0b0000101101000000 (2880)
popCount     = 4
clz          = 4
ctz          = 6
bitReverse   = 0b0000001011010000
byteSwap     = 0b0100000000001011

set bit 0    = 0b0000101101000001
clear bit 6  = 0b0000101100000000
toggle bit 6 = 0b0000101100000000
test bit 6   = true
test bit 5   = false

shl 2        = 0b0010110100000000
shr 2        = 0b0000001011010000
low 8 bits   = 0b01000000
high 8 bits  = 0b00001011

shlWithOverflow(x,6) = 0b1101000000000000 overflow=1
rotl 4  = 0b1011010000000000
rotr 4  = 0b0000000010110100
```

| Need | Zig |
|---|---|
| count set bits | `@popCount(x)` |
| leading / trailing zeros | `@clz(x)` / `@ctz(x)` |
| reverse bits / bytes | `@bitReverse(x)` / `@byteSwap(x)` |
| set bit n | `x \| (@as(T, 1) << n)` |
| clear bit n | `x & ~(@as(T, 1) << n)` |
| toggle bit n | `x ^ (@as(T, 1) << n)` |
| test bit n | `x & (@as(T, 1) << n) != 0` |
| shift, detect loss | `@shlWithOverflow(x, n)` → `.{ value, overflow_bit }` |
| rotate | `std.math.rotl` / `rotr` |
| is a power of two | `@popCount(x) == 1` |
| round up to power of two | `std.math.ceilPowerOfTwo(T, x)` |

Two Zig-specific traps:

- **`@as(u16, 1) << n`, not `1 << n`.** A bare `1` is `comptime_int`; shifting
  it gives a `comptime_int`, which then won't combine with a `u16` the way you
  expect. Annotate the literal.
- **`x << 2` can overflow and panic** in Debug/ReleaseSafe if bits fall off the
  top. `@shlWithOverflow` returns a tuple instead, and `<<|` saturates.

## A `packed struct` is a protocol header

Real DNS header flags — 16 bits, nine fields, no padding:

```zig
const std = @import("std");

const Opcode = enum(u4) { query = 0, iquery = 1, status = 2, notify = 4, update = 5 };

const DnsFlags = packed struct(u16) {
    rcode: u4,
    cd: bool,
    ad: bool,
    z: bool,
    ra: bool,
    rd: bool,
    tc: bool,
    aa: bool,
    opcode: Opcode,
    qr: bool,
};

pub fn main() !void {
    std.debug.print("bitSizeOf = {d}  sizeOf = {d}\n", .{ @bitSizeOf(DnsFlags), @sizeOf(DnsFlags) });
    inline for (.{ "rcode", "opcode", "qr", "rd" }) |f| {
        std.debug.print("  bitOffsetOf({s:<6}) = {d:>2}\n", .{ f, @bitOffsetOf(DnsFlags, f) });
    }

    const raw: u16 = 0x8180;
    const f: DnsFlags = @bitCast(raw);
    std.debug.print("\nraw 0x{x} = 0b{b:0>16}\n", .{ raw, raw });
    std.debug.print("  qr={} opcode={t} aa={} tc={} rd={} ra={} rcode={d}\n", .{ f.qr, f.opcode, f.aa, f.tc, f.rd, f.ra, f.rcode });

    const reply: DnsFlags = .{
        .qr = true,
        .opcode = .query,
        .aa = true,
        .tc = false,
        .rd = true,
        .ra = true,
        .z = false,
        .ad = false,
        .cd = false,
        .rcode = 3,
    };
    const out: u16 = @bitCast(reply);
    std.debug.print("\nbuilt NXDOMAIN reply = 0x{x} = 0b{b:0>16}\n", .{ out, out });
    std.debug.print("round trip = {}\n", .{@as(u16, @bitCast(@as(DnsFlags, @bitCast(out)))) == out});
}
```

```
bitSizeOf = 16  sizeOf = 2
  bitOffsetOf(rcode ) =  0
  bitOffsetOf(opcode) = 11
  bitOffsetOf(qr    ) = 15
  bitOffsetOf(rd    ) =  8

raw 0x8180 = 0b1000000110000000
  qr=true opcode=query aa=false tc=false rd=true ra=true rcode=0

built NXDOMAIN reply = 0x8583 = 0b1000010110000011
round trip = true
```

Compare the C you'd otherwise write:

```c
uint8_t  qr     = (raw >> 15) & 1;
uint8_t  opcode = (raw >> 11) & 0xF;
uint8_t  rcode  =  raw        & 0xF;
```

Three shift-and-mask expressions with hand-maintained constants, versus
`f.qr`. Every one of those magic numbers is a bug waiting for a spec revision.

Four things to get right:

| | |
|---|---|
| **`packed struct(u16)`** | the explicit backing type makes a wrong total a **compile error** |
| **field order is LSB-first** | `rcode` is declared first and sits at bit 0; `qr` is declared last at bit 15 |
| **`enum(u4)` fields work** | `opcode` reads as `.query`, printed with `{t}` |
| **`bool` is exactly 1 bit here** | not 1 byte — that's the difference from a plain struct |

LSB-first is the one that catches people: protocol specs draw bit 15 on the
*left*, so you declare the fields **bottom-up relative to the spec diagram**.

## Byte-level helpers

```zig
const std = @import("std");

const Pt = extern struct { x: u16, y: u16 };

pub fn main() !void {
    const p: Pt = .{ .x = 0x0102, .y = 0x0304 };

    std.debug.print("asBytes(&p)          = {x}  (type {s})\n", .{ std.mem.asBytes(&p).*, @typeName(@TypeOf(std.mem.asBytes(&p))) });
    std.debug.print("toBytes(p)           = {x}  (a copy, [4]u8)\n", .{std.mem.toBytes(p)});

    const raw = [_]u8{ 0x02, 0x01, 0x04, 0x03 };
    const back = std.mem.bytesToValue(Pt, &raw);
    std.debug.print("bytesToValue         = x=0x{x} y=0x{x}  (copy)\n", .{ back.x, back.y });
    const view = std.mem.bytesAsValue(Pt, &raw);
    std.debug.print("bytesAsValue         = x=0x{x} y=0x{x}  (view, aliases raw: {})\n\n", .{ view.x, view.y, @intFromPtr(view) == @intFromPtr(&raw) });

    const pts = [_]Pt{ .{ .x = 1, .y = 2 }, .{ .x = 3, .y = 4 } };
    const bytes = std.mem.sliceAsBytes(&pts);
    std.debug.print("sliceAsBytes({d} Pt)   = {d} bytes: {x}\n", .{ pts.len, bytes.len, bytes });
    const re = std.mem.bytesAsSlice(Pt, bytes);
    std.debug.print("bytesAsSlice back    = {d} Pt, [1].y = {d}\n\n", .{ re.len, re[1].y });

    const be = [_]u8{ 0x11, 0x22, 0x33, 0x44 };
    std.debug.print("readInt u32 .big     = 0x{x}\n", .{std.mem.readInt(u32, &be, .big)});
    std.debug.print("readInt u32 .little  = 0x{x}\n", .{std.mem.readInt(u32, &be, .little)});
    var out: [4]u8 = undefined;
    std.mem.writeInt(u32, &out, 0x11223344, .big);
    std.debug.print("writeInt .big        = {x}\n", .{out});
    std.mem.writeInt(u32, &out, 0x11223344, .little);
    std.debug.print("writeInt .little     = {x}\n", .{out});
}
```

```
asBytes(&p)          = 02010403  (type *align(2) const [4]u8)
toBytes(p)           = 02010403  (a copy, [4]u8)
bytesToValue         = x=0x102 y=0x304  (copy)
bytesAsValue         = x=0x102 y=0x304  (view, aliases raw: true)

sliceAsBytes(2 Pt)   = 8 bytes: 0100020003000400
bytesAsSlice back    = 2 Pt, [1].y = 4

readInt u32 .big     = 0x11223344
readInt u32 .little  = 0x44332211
writeInt .big        = 11223344
writeInt .little     = 44332211
```

| Function | Direction | Copies? | Returns |
|---|---|---|---|
| `std.mem.asBytes(&v)` | value → bytes | **no** | `*const [N]u8` pointing at `v` |
| `std.mem.toBytes(v)` | value → bytes | **yes** | `[N]u8` |
| `std.mem.bytesAsValue(T, b)` | bytes → value | **no** | `*const T` aliasing `b` |
| `std.mem.bytesToValue(T, b)` | bytes → value | **yes** | `T` |
| `std.mem.sliceAsBytes(s)` | `[]T` → `[]u8` | **no** | flat byte view |
| `std.mem.bytesAsSlice(T, b)` | `[]u8` → `[]T` | **no** | typed view |
| `std.mem.readInt(T, b, .big)` | bytes → int, **explicit endian** | yes | `T` |
| `std.mem.writeInt(T, &b, v, .big)` | int → bytes, explicit endian | — | writes in place |
| `std.mem.bigToNative(T, x)` | swap if needed | — | `T` |

**The `As`/`To` naming is the ownership tell**: `bytesAsValue` is a *view* (the
pointer equality above proves it), `bytesToValue` is a *copy*. A view's lifetime
is the buffer's lifetime — return one from a function and you have a dangling
pointer.

`readInt`/`writeInt` are the safe default for protocol work: the endianness is
in the call, not implied by the host. They also don't care about alignment,
unlike `bytesAsValue`.

## Parsing a real wire format

Length-prefixed frames, magic bytes, big-endian fields, zero allocations:

```zig
const std = @import("std");

const Header = extern struct {
    magic: [4]u8,
    version: u16,
    kind: u16,
    payload_len: u32,
};

const ParseError = error{ Short, BadMagic, BadVersion, TruncatedPayload };

const Message = struct { kind: u16, payload: []const u8 };

fn parse(buf: []const u8) ParseError!Message {
    if (buf.len < @sizeOf(Header)) return error.Short;
    const h = std.mem.bytesAsValue(Header, buf[0..@sizeOf(Header)]);
    if (!std.mem.eql(u8, &h.magic, "ZMSG")) return error.BadMagic;

    const version = std.mem.bigToNative(u16, h.version);
    const kind = std.mem.bigToNative(u16, h.kind);
    const len = std.mem.bigToNative(u32, h.payload_len);
    if (version != 1) return error.BadVersion;

    const start = @sizeOf(Header);
    if (buf.len < start + len) return error.TruncatedPayload;
    return .{ .kind = kind, .payload = buf[start..][0..len] };
}

fn encode(out: []u8, kind: u16, payload: []const u8) []const u8 {
    const h: Header = .{
        .magic = "ZMSG".*,
        .version = std.mem.nativeToBig(u16, 1),
        .kind = std.mem.nativeToBig(u16, kind),
        .payload_len = std.mem.nativeToBig(u32, @intCast(payload.len)),
    };
    @memcpy(out[0..@sizeOf(Header)], std.mem.asBytes(&h));
    @memcpy(out[@sizeOf(Header)..][0..payload.len], payload);
    return out[0 .. @sizeOf(Header) + payload.len];
}

fn show(label: []const u8, result: ParseError!Message) void {
    if (result) |m| {
        std.debug.print("parse({s}) -> kind={d}\n", .{ label, m.kind });
    } else |e| {
        std.debug.print("parse({s}) -> {t}\n", .{ label, e });
    }
}

pub fn main() !void {
    std.debug.print("@sizeOf(Header) = {d}  offsets: magic@{d} version@{d} kind@{d} len@{d}\n\n", .{
        @sizeOf(Header),           @offsetOf(Header, "magic"),       @offsetOf(Header, "version"),
        @offsetOf(Header, "kind"), @offsetOf(Header, "payload_len"),
    });

    var buf: [64]u8 = undefined;
    const frame = encode(&buf, 7, "hello zig");
    std.debug.print("encoded {d} bytes: {x}\n", .{ frame.len, frame });

    const msg = try parse(frame);
    std.debug.print("parsed  kind={d} payload=\"{s}\" ({d} B)\n", .{ msg.kind, msg.payload, msg.payload.len });
    std.debug.print("payload aliases frame? {}\n\n", .{msg.payload.ptr == frame.ptr + @sizeOf(Header)});

    var bad = buf;
    bad[0] = 'X';
    var badv = buf;
    badv[5] = 9;
    show("frame[0..4]  ", parse(frame[0..4]));
    show("frame[0..14] ", parse(frame[0..14]));
    show("bad magic    ", parse(bad[0..frame.len]));
    show("version 9    ", parse(badv[0..frame.len]));
}
```

```
@sizeOf(Header) = 12  offsets: magic@0 version@4 kind@6 len@8

encoded 21 bytes: 5a4d5347000100070000000968656c6c6f207a6967
parsed  kind=7 payload="hello zig" (9 B)
payload aliases frame? true

parse(frame[0..4]  ) -> Short
parse(frame[0..14] ) -> TruncatedPayload
parse(bad magic    ) -> BadMagic
parse(version 9    ) -> BadVersion
```

Read the hex: `5a4d5347` = `ZMSG`, `0001` = version 1 big-endian, `0007` =
kind 7, `00000009` = 9-byte payload, then `68656c6c6f207a6967` = `hello zig`.

The three things a reviewer checks in a parser:

| | In this code |
|---|---|
| **length checked before every read** | `buf.len < @sizeOf(Header)`, then `buf.len < start + len` |
| **endianness explicit** | `bigToNative` on all three numeric fields |
| **no allocation** | `msg.payload` aliases `frame` — the `true` above proves it |

`@sizeOf(Header)` is 12, not 10, because the `u32` forces 4-byte alignment and
`extern` won't reorder. If the spec says 10 bytes on the wire, the header must
be `packed struct` instead — and that is the real reason to reach for `packed`.

**The payload's lifetime is the buffer's lifetime.** `parse` returns a view, so
`Message` is only valid while `frame` is. That's the subject of 15.5.

---

Next: [15.4 — Data-Oriented Design](./04-data-oriented.md)
