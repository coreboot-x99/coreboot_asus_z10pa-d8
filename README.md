# ASUS Z10PA-D8 — coreboot port material

Board data collected from a stock ASUS Z10PA-D8 Series (dual LGA2011-3 for
Xeon E5-2600 v3/v4, C612/Wellsburg PCH) to support a native coreboot port:
autoport output, a full set of runtime logs, and the vendor blobs the Grantley
DRAM init needs.

CPU count and DIMM fill are properties of the machine, not of the board: the
board takes one or two LGA2011-3 CPUs and any valid DDR4 RDIMM/LRDIMM fill, so
nothing here assumes the configuration the capture happened to have.

## Board

| | |
|---|---|
| Vendor / board | ASUSTeK Z10PA-D8 Series, Rev 1.xx |
| Sockets | 2 × LGA2011-3, Xeon E5-2600 v3/v4, model `0x3f`/`0x4f` |
| Memory | 8 DIMM slots, DDR4 RDIMM/LRDIMM; the capture had 4 populated |
| Northbridge | Grantley, `8086:2f00` DMI2; IIO root ports `2f02` (01.0), `2f04` (02.0), `2f08` (03.0) |
| Southbridge | Intel C612 (Wellsburg), LPC `8086:8d44` (lspci names the family "C610/X99 series") |
| Storage | 6-port AHCI SATA `8086:8d02` (1f.2) + sSATA AHCI `8086:8d62` (11.4) |
| Super I/O | Nuvoton NCT6779D at `0x2e` (id `0xc562`), GPIO base `0x500`, PM base `0x400` |
| BIOS | `3801`, release date 08/23/2019 |
| Flash | 16 MiB SPI |

The board is a server board in the strict sense: server CPU/IMC pair with the
Wellsburg PCH, both storage complexes present (SATA and sSATA), SPSR function
`8086:8d7c` on 11.0 and no client-style LPC device. **Both sockets were
populated in the capture**, so the port must initialize remote-socket memory as
well; the Grantley runner in this tree has so far only been validated on
single-socket boards.

IIO link capabilities reported by stock:

| function | `LnkCap` port | width | state on stock |
|---|---|---|---|
| `00:01.0` | Port #1 | x8 | link down (`LnkSta` x0) |
| `00:02.0` | Port #3 | x16 | 8 GT/s, x16 |
| `00:03.0` | Port #7 | x16 | link down (`LnkSta` x0) |

## Stock image

| | |
|---|---|
| file | `Z10PA-D8_3801.rom` |
| size | 16777216 bytes |
| sha256 | `5d09dd4e33d52b018b44657abf641d9939c043a817ca9af2bf73a34d5ed4c59d` |

The image itself is not stored here; the blobs below are derived from it.

```
descriptor 00000000 - 00000fff  4096 bytes
me         00001000 - 007fffff  8384512 bytes
bios       00800000 - 00ffffff  8388608 bytes
```

## Blobs

`blobs/mainboard/asus/z10pa-d8/` mirrors the `3rdparty/blobs` path, so the
directory copies in whole.

| file | size | sha256 |
|---|---|---|
| `descriptor.bin` | 4096 | `5ddfd7387c75961e98214a8bb4a0473d887db815622d6a682f355fdf293e5b4f` |
| `me.bin` | 8384512 | `8358b473773bfc1624744959e0e581dd30912ddaadcfde4cc65e3fadec2be024` |
| `uncore.bin` | 621728 | `623ce81a7ef4140af986f2667d36735fb09577549eb0b3e907ca5846cbddd59f` |
| `intelsetup.bin` | 6211 | `f1004da7b160ab3d67aa7c81cfd614288bd8bfadc4456efebe0219170845ffdc` |

### Uncore image

The blob satisfies the in-place PE32 contract the Grantley runner enforces:

| check | value |
|---|---|
| machine / optional magic | `0x014c` / `0x010b` |
| sections | `.text` (`0x220`, 600136 bytes) + `.reloc` (`0x96460`, 6208 bytes), flat |
| `SizeOfImage` | `0x97ca0`, equal to the file size |
| entry point | `0x19ae0`, inside `.text` |
| `ImageBase` | `0xffe0f3d0` → CBFS position `0x60f3d0` |

Sizes and bases for comparison with the two boards already supported:

| board | size | `ImageBase` | `IntelSetup` |
|---|---|---|---|
| HuananZhi X99-QD4 | 605408 | `0xffdcf9b8` | 6090 |
| Machinist X99-B9 | 766368 | `0xffdce8e0` | 6090 |
| ASUS Z10PA-D8 | 621728 | `0xffe0f3d0` | **6211** |

**The setup struct does not match the existing boards.** `IntelSetup` is 6211
bytes here against 6090 on the QD4/B9 images, so the QD4/B9 `intelsetup.bin`
must not be reused, and any board-side setup offsets have to be re-derived from
this image rather than copied.

### What the image asks of the firmware

Derived from the disassembly of this dump (IDA, image base `0xffe0f3d0`):

* `IntelSetup` is read through the PEI read-only variable PPI with a declared
  buffer of **6212 bytes** at all five call sites (`0xffea1e2e`, `0xffea02d5`,
  `0xffea04ac`, `0xffea0aae`, `0xffe3223f`); the payload stored here is 6211
  bytes, i.e. the struct plus the trailing zero the variable drops. Serving the
  stored size is what the module expects, so the runner needs no change.
* The trained-configuration export is published as GUID
  `1de25879-6e2a-4d72-a768-288ccb9fa719` (`0xffe0f640`) in **136219 bytes**,
  written as `0xF000`-byte chunks with a shorter tail (`0xffea3442`). That is
  byte-for-byte the same contract the QD4 and B9 images use, so `RW_MRC_CACHE`
  handling carries over unchanged. The source sits at host offset 141732
  (`0x229a4`) instead of the 141632 used by the other two builds; the offset is
  internal to the module and does not affect the firmware side.

### Serial debug output

`IntelSetup` offset **0x2ee** is `Serial Debug Message Level`, and it is part of
the 6212-byte `IntelSetup` varstore the setup forms bind to (varstore 1 in that
formset; the question sits at `0x1ca4f8` with prompt/help string ids 5066/5067
and options 0 `Disable`, 1 `Minimum`, 2 `Normal`, 3 `Maximum`, default
`Disable`). On this image the byte reads 1, i.e. the board was left at
`Minimum`.

That offset is only the setup field, not an output path: this dump is still a
release build, with all 858 `DebugPrint` call sites resolving to a one-byte
`ret` at `0xffe52e6d` and no UART port I/O in the image. Setting the level
therefore records the setting the vendor code would consult, but no vendor
messages appear on COM1 from this blob - which is also why the QD4/B9 captures
show none despite the B9 entry setting `0x2d6`.

## Logs

`logs/` holds the runtime capture this material is based on: `acpidump.log`,
`cpuinfo.log`, `dmesg.log`, `dmidecode.log`, `ectool.log`,
`input_bustypes.log`, `inteltool.log`, `ioports.log`, `lspci.log` (with config
space dumps), `superiotool.log`.

`dmesg.log` is a boot of the stock firmware (BIOS 3801, kernel 7.2.15). Its
CPU count, DIMM fill and the resulting NUMA/IOAPIC/DMAR layout are that
machine's configuration, not board properties - the log is there for the
firmware behaviour (DRHD placement, GSI routing, ACPI tables), not as a
specification of what a port must expect. In that boot the firmware published
two DMAR units, `0xfbffc000` (flags 0) and `0xc7ffc000` (flags 1); the port
publishes `0xfbffc000`, the base the other X99 boards use, and does not
reproduce the second unit yet.

## Autoport

`autoport/` is raw autoport output. It is material, not a buildable board
directory: vendor symbols are still empty and `board_info.txt` still carries
autoport's `FIXME` lines. Two things in it are wrong for this board and must be
corrected by hand in a port:

* the root-port slot comments are autoport's guesses, not the widths in the
  `LnkCap` table above;
* the PCH blocks describe a Lynx Point client function set; Wellsburg's sSATA
  complex (11.4) and SPSR function (11.0) are both present and both visible on
  stock, so a port must not take the second storage complex off the bus.

## SHA256SUMS

`SHA256SUMS` covers every file shipped here.
