# Memory Management and Mapping

## Overview

- The main board is shipped with 128KB of Static RAM, which can be expanded by
  the user.
- At boot the ROM is overlaid at address `0x00'0000`.

## Main Memory Map

|    Address Range    | Size  | Description                |
| :-----------------: | :---: | -------------------------- |
| 0x00'0000-0x01'ffff | 128KB | Internal Work RAM          |
| 0x02'0000-0x03'ffff | 128KB | Internal Work RAM mirror   |
| 0x04'0000-0x04'ffff | 64KB  | MMIO (see MMIO Memory Map) |
| 0x05'0000-0x05'ffff | 64KB  | MMIO mirror                |
| 0x06'0000-0x06'ffff | 64KB  | MMIO mirror                |
| 0x07'0000-0x07'ffff | 64KB  | MMIO mirror                |
| 0x08'0000-0x0b'ffff | 256KB | Kernel ROM (BIOS)          |
| 0x0c'0000-0x0f'ffff | 256KB | Kernel ROM mirror          |
| 0x10'0000-0x7f'ffff |  7MB  | Reserved                   |
| 0x80'0000-0xbf'ffff |  4MB  | External Work RAM          |
| 0xc0'0000-0xff'ffff |  4MB  | Reserved                   |

> Unallocated regions should trigger /BERR on access.

> Memory Map is subject to changes.

### Internal Work RAM layout

|    Address Range    | Size | Description             |
| :-----------------: | :--: | ----------------------- |
| 0x00'0000-0x00'03ff | 1KB  | Exception Vector Table  |
| 0x00'0400-0x00'0fff | 3KB  | BIOS Memory / Boot Info |
| 0x00'1000-0x00'7fff | 28KB | Kernel Heap             |
| 0x00'8000-0x01'ffff | 96KB | User Heap               |

### MMIO Memory Map

| Start Address | End Address | Description       |
| :-----------: | :---------: | ----------------- |
|   0x04'0000   |  0x04'00ff  | VDP Registers     |
|   0x04'0100   |  0x04'01ff  | System/IO Control |
|   0x04'0200   |  0x04'7fff  | Reserved (/DTACK) |
|   0x04'8000   |  0x04'8fff  | Card MMIO 0       |
|   0x04'9000   |  0x04'9fff  | Card MMIO 1       |
|   0x04'a000   |  0x04'afff  | Card MMIO 2       |
|   0x04'b000   |  0x04'bfff  | Card MMIO 3       |
|   0x04'c000   |  0x04'ffff  | Reserved (/DTACK) |

> Each MMIO region is 256-bytes wide

#### MMIO - VDP Registers

- [Video Display Processor](docs/vdp.md)

#### MMIO - IO

All registers are accessible by the I/O Controller register map

> IO_MMIO: 0x04'0100

```asm
;==================
; SNES Controllers
;==================
IO_MMIO + 0x00 -> SNES0_DATA_L | RO
IO_MMIO + 0x01 -> SNES0_DATA_H | RO
IO_MMIO + 0x02 -> SNES1_DATA_L | RO
IO_MMIO + 0x03 -> SNES1_DATA_H | RO
IO_MMIO + 0x04 -> SNES2_DATA_L | RO
IO_MMIO + 0x05 -> SNES2_DATA_H | RO
IO_MMIO + 0x06 -> SNES3_DATA_L | RO
IO_MMIO + 0x07 -> SNES3_DATA_H | RO

LO_BYTE:
	7  6  5  4  3  2  1  0
	r  l  d  u  S  s  y  b

	[0:0] b -> B button
	[1:1] y -> Y button
	[2:2] s -> Select
	[3:3] S -> Start
	[4:4] u -> Up-pad
	[5:5] d -> Down-pad
	[6:6] l -> Left-pad
	[7:7] r -> Right-pad

HI_BYTE:
	7  6  5  4  3  2  1  0
	.  .  .  .  R  L  x  a

	[0:0] a -> A button
	[1:1] x -> X button
	[2:2] L -> Left shoulder
	[3:3] R -> Right shoulder

;=========================
; PS/2 Keyboard and Mouse
;=========================
IO_MMIO + 0x08 -> PS2K_STATUS | RO
IO_MMIO + 0x0a -> PS2M_STATUS | RO
	7  6  5  4  3  2  1  0
	E  R  A  B  T  P  O  D

	[0:0] D - DATA_READY -> Set if Scancode is ready in PS2K_DATA
	[1:1] O - OVERRUN -> Set if Buffer overflowed (Cleared after PS2K_DATA read)
	[2:2] P - PARITY_ERR -> Reflects parity bit from last received data byte
	[3:3] T - TIMEOUT -> Set if no response from keyboard
	[4:4] B - CMD_BUSY -> Set if command is being processed, new commands ignored
	[5:5] A - CMD_ACK -> Set if last command was acknowledged (0xfa)
	[6:6] R - CMD_RESEND -> Set if last command requested resend (0xfe)
	[7:7] E - ERROR -> Set at unrecoverable error

IO_MMIO + 0x09 -> PS2K_DATA | R/W
IO_MMIO + 0x0b -> PS2M_DATA | R/W
	7  6  5  4  3  2  1  0
	d  d  d  d  d  d  d  d

	[7:0] d - DATA/COMMAND -> Data or command

; If read: Returns the next byte from device. For keyboard: scancode. For Mouse: one byte
; of 3-byte packed. Reading clears DATA_READY.
; If write: Sends a command byte to the device. If the command expects a data byte, write
; it immediately after. The firmware automatically retries on resend (up to 3 times).


;===========
; Interrupt
;===========
IO_MMIO + 0x0c -> IRQ_ENABLE | R/W
	7  6  5  4  3  2  1  0
	.  .  M  K  S  S  S  S

	[3:0] S - SNESx_ENABLE -> Bitfield of which SNES controller is enabled
to send IRQs (see table below)
	[4:4] K - PS2K_IRQ_E -> If set, enables keyboard to send IRQs
	[5:5] M - PS2M_IRQ_E -> If set, enables mouse to send IRQs


IO_MMIO + 0x0d -> IRQ_FLAGS | R/W (write 1 to clear)
	7  6  5  4  3  2  1  0
	.  .  M  K  S  S  S  S

	[3:0] S - SNESx_INT -> Bitfield of which SNES controller sent by
current IRQ (see table below)
	[4:4] K - PS2K_INT -> Set if current IRQ was sent by the keyboard
	[5:5] M - PS2M_INT -> Set if current IRQ was sent by the mouse

; SNES controller bits:
; Bit 0 -> SNES controller on port 0
; Bit 1 -> SNES controller on port 1
; Bit 2 -> SNES controller on port 2
; Bit 3 -> SNES controller on port 3


;==========
; Firmware
;==========
IO_MMIO + 0x0e -> IO_FEATURES | RO
	7  6  5  4  3  2  1  0
	.  .  M  K  S  S  S  S

	[3:0] S -> SNESx_PRN -> Bit field of which SNES controller is
detected or connected
	[4:4] K -> PS2K_PRN -> Set if PS/2 keyboard is present
	[5:5] M -> PS2M_PRN -> Set if PS/2 mouse is present

IO_MMIO + 0x0f -> VERSION | RO
	7  6  5  4  3  2  1  0
	M  M  M  M  m  m  m  m

	[3:0] m -> MINOR -> Minor version number
	[7:4] M -> MAJOR -> Major version number

;====================
; Slot Configuration
;====================
IO_MMIO + 0x10|0x20|0x30|0x40 -> SLOTx_ID0 | RO ; 0x52 ('R')
IO_MMIO + 0x11|0x21|0x31|0x41 -> SLOTx_ID1 | RO ; 0x42 ('B')
IO_MMIO + 0x12|0x22|0x32|0x42 -> SLOTx_ID2 | RO ; 0x54 ('T')
IO_MMIO + 0x13|0x23|0x33|0x43 -> SLOTx_ID3 | RO ; 0x0a ('\n')
  7  6  5  4  3  2  1  0
  i  i  i  i  i  i  i  i

  [31:0] i - ID -> ASCII text "RBT\n"

IO_MMIO + 0x14|0x24|0x34|0x44 -> SLOTx_DEVICE | RO
  7  6  5  4  3  2  1  0
  N  N  N  N  N  N  N  N

  [7:0] N - NAME -> Device name/identifier

IO_MMIO + 0x15|0x25|0x35|0x45 -> SLOTx_FLAGS | R/W
  7  6  5  4  3  2  1  0
  A  .  .  .  x  x  x  x

  [7:7] A - ADDR_MODE -> 0=Banked, 1=Self-Decode
  [3:0] x - OX_MASK -> Bitmask of assigned /IO0-/I03 lines

IO_MMIO + 0x16|0x26|0x36|0x46 -> SLOTx_STATUS | R/W
  7  6  5  4  3  2  1  0
  .  .  .  .  .  .  I  i

  [1:1] I - IRQ_PENDING -> If 1, card is pending interrupt request.
  [0:0] i - IRQ_ACK -> Used it acknowledge an interrupt request. [W1C]
```

---

> rbt-docs © 2026 by aCube is licensed under CC BY-SA 4.0.<br>
> See https://creativecommons.org/licenses/by-sa/4.0/
