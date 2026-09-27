# Expansion Card Slots

&nbsp;&nbsp;&nbsp;&nbsp;The RBT-16 exposes two kinds of expansion connectors: four general-purpose
IO Expansion Slots, and a single External Work RAM slot. Both share the CPU's
address and data bus directly (no bus mastering, so no expansion card can ever
drive the bus).

## Slot Types

| Slot              | Qty |   Connector    | Purpose                             |
| :---------------- | :-: | :------------: | :---------------------------------- |
| IO Expansion      |  4  | 2x50 (100-pin) | Modem, UART, USB, Floppy, HDD, etc. |
| External Work RAM |  1  | 2x30 (60-pin)  | RAM Expansion (64KB-4MB)            |

> Neither connector carries `/BR`, `/BG` or `/BGACK`. Every card is essentially
> a slave device.

## External Work RAM

- Address range: `0x80'0000 - 0xbf'ffff` (4MB), see Memory Map.
- Size is reported by 3 Present-Detect pins. The slot mirrors a smaller
  module across the full 4MB window (e.g. a populated 64MB mirrors 64 times
  to fille the range).

| Group          | Signals                                |
| :------------- | :------------------------------------- |
| Data Bus       | `D0-D15`                               |
| Address Bus    | `A1-A23`                               |
| Bus Control    | `/AS`, `R/W`, `/UDS`, `/LDS`, `/DTACK` |
| Clock          | `CLK` (Fixed 12MHz)                    |
| Present-Detect | `/PD0`, `/PD1`, `/PD2`                 |
| Slot Control   | `/CS`                                  |
| Power          | `+3.3V`, `+5V`, `+12V`, `GND`          |

### Present-Detect size encoding

```asm
PD[2:0] -> Module Size
  0b000 -> Empty (no card present)
  0b001 -> 64KB
  0b010 -> 128KB
  0b011 -> 256KB
  0b100 -> 512KB
  0b101 -> 1MB
  0b110 -> 2MB
  0b111 -> 4MB
```

## IO Expansion Slot

| Group             | Signals                                | Share or Unique? |
| :---------------- | :------------------------------------- | :--------------: |
| Data Bus          | `D0-D15`                               |      Shared      |
| Address Bus       | `A1-A23`                               |      Shared      |
| Bus Control       | `/AS`, `R/W`, `/UDS`, `/LDS`, `/DTACK` |      Shared      |
| Clock/Reset       | `CLK` (Fixed 12MHz), `/RESET`          |      Shared      |
| MMIO Enable       | `/IO0`, `/IO1`, `/IO2`, `/IO3`         |      Shared      |
| Interrupt Request | `/IRQ1`, `/IRQ2`, `/IRQ3`, `/IRQ4`     |      Shared      |
| Slot Control      | `/CS`, `/IRQACK`                       | Unique, per-slot |
| SPI               | `SCK`, `MOSI`, `MISO`                  |      Shared      |
| SPI Select        | `/SS_SPI`                              | Unique, per-slot |
| Audio             | `AUDIO_MIX_IN`, `AUDIO_MIX_OUT`        | Shared (analog)  |
| Power             | `+3.3V`, `+5V`, `+12V`, `GND`          |      Shared      |

- `/CS` selects this slot's always present `4KB` control window in MMIO.
- `/IRQn` triggers a interrupt request into the central priority encoder
  (fixed priority: Slot 0 highest -> Slot 3 lowest). To trigger an interrupt,
  the line should be held low until `/IRQACK` asserts low for acknowledgement.
  The interrupts `4`, `5`, `6` and `7` are reserved by the mainboard system, the
  card cannot trigger those interrupts.
- `AUDIO_MIX_IN/OUT` is a shared analogic summing bus, a card can inject and
  extract audio through this tap.

> Every IO slot carries the same 100 pins. Each card decides which pins to use.

## IO Expansion Card Addressing Mode

An IO Expansion Card picks one of two addressing modes, self-reported to the
BIOS:

1. **Banked**(`ADDR_MODE=0`): The card listens for one or more of
   `/IO0-/IO3`, selected by an onboard jumper/DIP-switch. Each line when
   asserted, exposes a pre-decoded `4KB` MMIO window. A card wanting more
   than `4KB` window can claim multiple adjacent lines.
2. **Self-decode**(`ADDR_MODE=1`): The card decodes `A23:A1` itself for a
   custom base/size within any region of the memory (not overlapping with
   reserved spaces).
