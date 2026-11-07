# HuananZhi X99-QD4 — coreboot port material

Board data collected from a stock HuananZhi X99-QD4 (LGA2011-3, Xeon E5 v3
with a client Q87 PCH) to support a native coreboot port: autoport output, a
full set of runtime logs, and the vendor blobs the Grantley DRAM init needs.

## Board

| | |
|---|---|
| Vendor / board | HuananZhi X99-QD4 V1.0 |
| CPU | Xeon E5-2630 v3 (Haswell-EP, 8C/16T), model 63 |
| Memory | 2 x 8 GiB in 4 slots |
| Northbridge | Grantley, `8086:2f00` DMI2, IIO root ports `2f02`/`2f04`/`2f08` |
| Southbridge | Intel Q87 (Lynx Point, 8 Series/C220), LPC `8086:8c4e` |
| Storage | 6-port AHCI SATA `8086:8c02` only |
| Super I/O | Nuvoton NCT5539D at `0x2e` |
| BIOS | `5.11`, release date 10/10/2022 |
| Flash | 16 MiB SPI |

The board is a hybrid: a Haswell-EP CPU and IMC pair with a client Lynx
Point PCH. There is no sSATA complex, no SPS and no second storage function
here, and the four DIMM slots are wired to the CPU IMC, not to the PCH.

## Stock image

| | |
|---|---|
| file | `X99_QD4_20221010_H819903.rom` |
| size | 16777216 bytes |
| sha256 | `ff7d29d5e4b19d6ea9cca260122a1da5cf538db91cbd7a250c1181035f872ca3` |

The image itself is not stored here; the blobs below are derived from it.

```
descriptor 00000000 - 00001000  4096 bytes
bios       00800000 - 01000000  8388608 bytes
me         00001000 - 00800000  8384512 bytes
```

## Blobs

Extracted with `util/grantley/extract_blobs.py` from a coreboot tree:

```
util/grantley/extract_blobs.py -o blobs/mainboard/huananzhi/x99-qd4 \
        X99_QD4_20221010_H819903.rom
```

| blob | source | size | sha256 |
|---|---|---|---|
| `descriptor.bin` | flash descriptor region | 4096 | `bd3a7ecd757bd7864e25a513842157bfde1df209b018686eed24db6851d09416` |
| `me.bin` | ME region, `0x1000`–`0x800000` | 8384512 | `1a3bc71d1b952c538674861d9e70dd7c8e32d5a062e74716e559474135b3942f` |
| `uncore.bin` | PE32 image of `UncoreInitPeim` | 605408 | `74c6dafebbcfb5b823f4eef98c958df35b21dd5ee91a731a114aaa14de5e4f04` |
| `intelsetup.bin` | `IntelSetup` NVAR variable | 6090 | `d35e9bc1d2d410c7b72af4dd83f656364ca4c9ed2b1597543a3c83a01eceab90` |

Layout mirrors `3rdparty/blobs`, so the directory can be copied as is; the
Grantley Kconfig defaults already look for
`3rdparty/blobs/mainboard/$(MAINBOARDDIR)/uncore.bin` and
`.../intelsetup.bin`.

## Compatibility with the coreboot Grantley code

Checked against `src/northbridge/intel/grantley` and its `uncore_runner`.

**`uncore.bin`** — the runner has no loader: it maps the CBFS file and calls it
in place. `grantley_peim_entry()` requires `MZ`, `PE\0\0`, machine `0x014c`,
optional header `0x010b`, flat sections (`VirtualAddress == PointerToRawData`),
the entry point inside an executable section and `ImageBase` equal to the
mapped address. This image satisfies all of them:

```
machine 0x014c (i386)   optional header 0x010b (PE32)   sections 2
ImageBase 0xffdcf9b8    entry RVA 0x195c0   SizeOfImage 0x93ce0 (= file size)
.text   VA 0x000220 == raw 0x000220   vsize 0x924c8   rawsize 0x924e0  exec
.reloc  VA 0x092700 == raw 0x092700   vsize 0x15cc    rawsize 0x15e0
```

`Makefile.mk` takes the CBFS position from this very header
(`util/grantley/pe_image_base.py`), so the file is placed at its `ImageBase` and
the runtime address check passes without relocation.

**`intelsetup.bin`** — 6090 bytes, the RC `SYSTEM_CONFIGURATION` as the board
last stored it in the `IntelSetup` variable. The runner keeps no copy and no
interpretation: `grantley_pei_set_intelsetup()` stores the pointer and
`GetVariable` hands the bytes back to the PEIM unchanged, so the size has to be
exactly what the PEIM expects — it is the same 6090 bytes the port already
runs with. Note that the PEIM's `PlatformInfo` payload, not this blob, is what
`grantley_pei_set_board_identity()` patches (board id at `0x46`, policy
revision at `0x1e`).

**`descriptor.bin` / `me.bin`** — full regions, used as flash layout reference
and for ME neutralisation; nothing in the Grantley code parses them.

## Logs

Collected on the board running its stock firmware, CachyOS, kernel
`7.2.8-1-cachyos`.

| file | content |
|---|---|
| `acpidump.log` | full ACPI table set |
| `lspci.log` | `lspci -xxx` tree: IIO stacks, Q87 functions, GPU |
| `dmidecode.log` | SMBIOS, DIMM population |
| `cpuinfo.log` | per-CPU MSR-ish view, turbo limits |
| `dmesg.txt` | boot log: IIO, AER, MCE, thermal |
| `inteltool.log` | chipset register dump (RCBA, GPIO, PM) |
| `superiotool.log` | NCT5539D Super I/O |
| `ectool.log` | EC access |
| `ioports.log` | `/proc/ioports` |
| `input_bustypes.log` | input device bus types |
| `codec#2` | HDA codec verbs (ALC897) |
| `pin_hwC0D2` | GPIO pin state |

## Autoport

`autoport/` holds the unedited autoport output for this board
(`devicetree.cb`, `gpio.c`, `hda_verb.c`, `romstage.c`, `bootblock.c`, `dsdt.asl`,
`acpi/*`, `Kconfig*`, `Makefile.mk`, `board_info.txt`). It is raw material, not
a buildable board directory: the vendor symbol is still empty
(`BOARD_HUANANZHI_`) and `board_info.txt` carries autoport's own FIXME.

### `gpio.c` is complete, not truncated

The old-style `pch_gpio_map` that autoport emits for Lynx Point only carries
the bits the hardware actually defines:

| block | set1 | set2 | set3 |
|---|---|---|---|
| `mode` | 32 | 32 | 12 |
| `direction` | 15 | 13 | 2 |
| `level` | 8 | 8 | 0 |
| `reset` / `invert` / `blink` | 0 | 0 | 0 |

`mode` covers every pad. `direction` is emitted only for pads in GPIO mode
(`constraint = GPIO_USE_SEL`, `0x9908f83d` / `0x02fc501d` / `0x00000300`) and
`level` only for outputs (`GP_IO_SEL & GP_LVL`); a native-mode pad has no
direction to record. The `reset`, `invert` and `blink` blocks are empty
because this board leaves `GP_RST_SEL1/2/3` at zero and shows no inverted or
blinking pads, so their constraint masks are zero.

### `hda_verb.c` needed an autoport fix

Unpatched autoport locates the card by probing `pin_hwC<card>D0`
(`util/autoport/log_reader.go`). This machine's ALC897 is **codec index 2**, so
the collected set has `pin_hwC0D2` and no `pin_hwC0D0`; the probe fails, the
function returns an empty list and `cim_verb_data` is written empty even though
`codec#2` (vendor `0x10ec0897`, subsystem `0x10ec0000`) and fourteen pin
defaults were present in the log set.

`autoport` now accepts any `pin_hwC<card>D<n>` file as the card marker, and
this directory was regenerated with that version: `autoport/hda_verb.c` holds
one subvendor plus fourteen `AZALIA_PIN_CFG` entries for codec 2. Every other
file came out byte-identical, which is what confirms the fix only affected the
codec lookup.

## Port notes

- The Super I/O LDNs have to sit under `chip superio/common` /
  `device pnp 2e.0` for the ACPI SSDT generator. Declared directly under the
  LPC bridge they are skipped with
  `Parent of device PNP: 002e.x is not a PNP device` and no ACPI devices for
  the Super I/O are emitted.
- A discrete TPM module fits the J9A1 header, so the port enables
  `MEMORY_MAPPED_TPM` and declares the TIS device at `0c31.0`.
