# ASUS Z10PA-D8 — coreboot port material

Board data collected from a stock ASUS Z10PA-D8 Series (dual LGA2011-3, Xeon
E5 v3, C610/Wellsburg PCH) to support a native coreboot port: autoport output,
a full set of runtime logs, and the vendor blobs the Grantley DRAM init needs.

## Board

| | |
|---|---|
| Vendor / board | ASUSTeK Z10PA-D8 Series, Rev 1.xx |
| CPU | 2 × Xeon E5-2673 v3 (Haswell-EP, 12C/24T each, 48 threads), model 63 |
| Memory | 2 × 16 GiB DDR4-2133 ECC RDIMM (`DIMM_A1`, `NODE 1`) |
| Northbridge | Grantley, `8086:2f00` DMI2; IIO root ports `2f02` (01.0), `2f04` (02.0), `2f08` (03.0) |
| Southbridge | Intel C612 (Wellsburg), LPC `8086:8d44` (lspci names the family "C610/X99 series") |
| Storage | 6-port AHCI SATA `8086:8d02` (1f.2) + sSATA AHCI `8086:8d62` (11.4) |
| Super I/O | Nuvoton NCT6779D at `0x2e` (id `0xc562`), GPIO base `0x500`, PM base `0x400` |
| BIOS | `3801`, release date 08/23/2019 |
| Flash | 16 MiB SPI |

The board is a server board in the strict sense: server CPU/IMC pair with the
Wellsburg PCH, both storage complexes present (SATA and sSATA), SPSR function
`8086:8d7c` on 11.0 and no client-style LPC device. **Two sockets are
populated**, so a port has to initialize remote-socket memory as well; the
Grantley runner in this tree has so far only been validated on single-socket
boards.

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

## Logs

`logs/` holds the runtime capture this material is based on: `acpidump.log`,
`cpuinfo.log`, `dmidecode.log`, `ectool.log`, `input_bustypes.log`,
`inteltool.log`, `ioports.log`, `lspci.log` (with config space dumps),
`superiotool.log`.

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
