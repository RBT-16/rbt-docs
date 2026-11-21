# Memory Management and Mapping

## Overview

- The mainboard is shipped with 256KB of Static RAM, which can be expanded by
  the user.
- At boot the entire ROM is overlaid at address `0x00'0000`.

## Main Memory Map

|     Address Range     |   Size   | Description             |
| :-------------------: | :------: | :---------------------- |
| `0x00'0000-0x03'ffff` | `256KB ` | SYSRAM (System RAM)     |
| `0x04'0000-0x0d'ffff` | `640KB ` | Reserved                |
| `0x0e'0000-0x0e'ffff` | `64KB  ` | MMIO                    |
| `0x0f'0000-0x0f'ffff` | `64KB  ` | MMIO (mirror)           |
| `0x10'0000-0x4f'ffff` | `4MB   ` | EXRAM (Expansion RAM)   |
| `0x50'0000-0x7b'ffff` | `2.75MB` | Reserved                |
| `0x7c'0000-0x7f'ffff` | `256KB ` | Firmware ROM            |
| `0x80'0000-0xff'ffff` | `8MB   ` | Expansion Address Space |

> Reserved regions must trigger /BERR on access.

> Memory Map is subject to changes.

### System RAM Layout

|     Address Range     |  Size   | Description             |
| :-------------------: | :-----: | :---------------------- |
| `0x00'0000-0x00'03ff` | `1KB  ` | Exception Vector Table  |
| `0x00'0400-0x00'0fff` | `3KB  ` | BIOS Memory / Boot Info |
| `0x00'1000-0x00'ffff` | `60KB ` | Kernel Heap             |
| `0x01'0000-0x03'ffff` | `192KB` | User Heap               |

### MMIO Memory Map

|     Address Range     |  Size  | Description       |
| :-------------------: | :----: | :---------------- |
| `0x0e'0000-0x0e'00ff` | `256B  ` | VDP Registers     |
| `0x0e'0100-0x0e'01ff` | `256B  ` | System Control |
| `0x0e'0200-0x0e'7fff` | `31.5KB` | (Unassigned)      |
| `0x0e'8000-0x0e'8fff` | `4KB   ` | Expansion Slot 0  |
| `0x0e'9000-0x0e'9fff` | `4KB   ` | Expansion Slot 1  |
| `0x0e'a000-0x0e'afff` | `4KB   ` | Expansion Slot 2  |
| `0x0e'b000-0x0e'bfff` | `4KB   ` | Expansion Slot 3  |
| `0x0e'c000-0x0e'ffff` | `16KB  ` | (Unassigned) |

> Unassigned regions assert /DTACK on access. Write does nothing and reads can
> return garbage data.

#### MMIO - VDP Registers

- [Video Display Processor](docs/vdp.md)

#### MMIO - System Control

> SYS_MMIO: 0x0e'0100

```asm
;=====================
; System and Firmware
;=====================
SYS_MMIO + 0x00 -> SYS_FLAGS | RW
  F  E  D  C  B  A  9  8  7  6  5  4  3  2  1  0
  .  .  .  .  .  .  .  .  .  .  .  .  .  .  .  O

  [0:0] O - OVERLAY -> When clear, overlay ROM on the first 256KB of SYSRAM
```

---

> rbt-docs © 2026 by aCube is licensed under CC BY-SA 4.0.<br>
> See https://creativecommons.org/licenses/by-sa/4.0/
