# Expansion Card Slots

&nbsp;&nbsp;&nbsp;&nbsp;The RBT-16 exposes two kinds of expansion connectors: four general-purpose
IO Expansion Slots, and a single Expansion RAM slot. Both share the CPU's
address and data bus directly.

> Note: Bus mastering can be added in the future.

## Slot Types

| Slot          | Qty |   Connector    | Purpose                             |
| :------------ | :-: | :------------: | :---------------------------------- |
| IO Expansion  |  4  | 2x50 (100-pin) | Modem, UART, USB, Floppy, HDD, etc. |
| Expansion RAM |  1  | 2x30 (60-pin)  | RAM Expansion (up until 4MB)        |

## Expansion RAM

| Signals                | Direction          | Function                                               |
| :--------------------- | :----------------- | :----------------------------------------------------- |
| `A1-A23`               | Mainboard -> Card  | 24-bit CPU address bus.                                |
| `D0-D15`               | Mainboard <-> Card | 16-bit CPU data bus.                                   |
| `/AS`                  | Mainboard -> Card  | Address Strobe. Indicate a valid address on bus.       |
| `R/W`                  | Mainboard -> Card  | Read/Write direction. High = read, low = write.        |
| `/UDS`                 | Mainboard -> Card  | Upper Data Strobe.                                     |
| `/LDS`                 | Mainboard -> Card  | Low Data Strobe.                                       |
| `/DTACK`               | Mainboard <- Card  | Data Transfer Acknowledgment.                          |
| `/BERR`                | Mainboard <- Card  | Bus Error.                                             |
| `CLK`                  | Mainboard -> Card  | Fixed 12MHz system clock.                              |
| `/PD`                  | Mainboard <- Card  | Present Detect. Pulled-up when no physical connection. |
| `/CS`                  | Mainboard -> Card  | Chip Select. Only asserted within 4MB window.          |
| `+3.3V`, `+5V`, `+12V` | Power              | Mainboard power supply.                                |
| `AGND`, `GND`          | Power              | Ground reference.                                      |

## IO Expansion Slot

| Signals                | Direction          | Function                                               |
| :--------------------- | :----------------- | :----------------------------------------------------- |
| `A1-A23`               | Mainboard -> Card  | 24-bit CPU address bus.                                |
| `D0-D15`               | Mainboard <-> Card | 16-bit CPU data bus.                                   |
| `/AS`                  | Mainboard -> Card  | Address Strobe. Indicate a valid address on bus.       |
| `R/W`                  | Mainboard -> Card  | Read/Write direction. High = read, low = write.        |
| `/UDS`                 | Mainboard -> Card  | Upper Data Strobe.                                     |
| `/LDS`                 | Mainboard -> Card  | Low Data Strobe.                                       |
| `/DTACK`               | Mainboard <- Card  | Data Transfer Acknowledgment.                          |
| `/BERR`                | Mainboard <- Card  | Bus Error. Pull-up.                                    |
| `CLK`                  | Mainboard -> Card  | Fixed 12MHz system clock.                              |
| `/IRQ1-/IRQ7`          | Mainboard <- Card  | Interrupt Request (level 1 -> level 7).                |
| `/IRQACK`              | Mainboard -> Card  | Interrupt Request Acknowledgment.                      |
| `/PD`                  | Mainboard <- Card  | Present Detect. Pulled-up when no physical connection. |
| `/CS`                  | Mainboard -> Card  | Chip Select. Only asserted within 4KB control window.  |
| `/RESET`               | Mainboard -> Card  | Resets the expansion card.                             |
| `AUDIO_MIX_IN`         | Mainboard -> Card  | Mixed Analog audio input.                              |
| `AUDIO_MIX_OUT`        | Mainboard <- Card  | Mixed Analog audio output.                             |
| `+3.3V`, `+5V`, `+12V` | Power              | Mainboard power supply.                                |
| `AGND`, `GND`          | Power              | Ground reference.                                      |

---

- `/IRQx` triggers an interrupt request into the central priority encoder,
  (fixed priority: Slot 0 highest -> Slot 7 lowest). To trigger an interrupt
  the line should be held low until `/IRQACK` asserts low for acknowledgment.
  Is dedicated for each slot.
- `/IRQACK` is asserted when the CPU do the acknowledgment cycle:
  (`FC0-FC2=111`) and `/AS=0`. A latch preserves which slot won in the
  priority encoder, then it is used to drive the `/IRQACK` back to the
  corresponding slot.
- `AUDIO_MIX_IN/OUT` is a shared analog summing bus, a card can inject and
  extract audio through this tap. Is daisy-chained from slot 0 to slot 3.
- When `/BERR` and `/DTACK` are not asserted, two chained `74HC161` 4-bit
  binary counters are enabled, and after 64 cycles, `/BERR` is asserted
  automatically. The slot must respond with `/DTACK` or `/BERR` before the
  counter finishes.

> Every IO slot carries the same 100 pins. Each card decides which pins to use.

---

> rbt-docs © 2026 by aCube is licensed under CC BY-SA 4.0.<br>
> See https://creativecommons.org/licenses/by-sa/4.0/
