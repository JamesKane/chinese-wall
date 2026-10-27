# 02: Pin Controller and GPIO

**Status: draft.** The pad register layout, pin-to-register mapping and
GPIO register file agree across mainline, the SpacemiT K3 user manual and
the SpacemiT UEFI firmware.

Two blocks are covered here:

| Block | Compatible | Base | Window | Clocks (`clock-names`) | Interrupt |
|---|---|---|---|---|---|
| Pin controller (MFPR, "AIB") | `spacemit,k3-pinctrl` | `0xD401E000` | `0x1000` | APBC AIB: `func` (gate) and `bus` (bus gate) | none |
| GPIO | `spacemit,k3-gpio` | `0xD4019000` | `0x100` | APBC GPIO: `core` (gate) and `bus` (bus gate) | APLIC source 58, one line for all banks |

Both clocks come from the APBC (01 §1.7, register `0x03C` for AIB and
`0x008` for GPIO). Both are fixed 24 MHz with no mux. **[HW]** Enable both
clocks of a block before touching its registers.

The user manual (section 3, "Pinout", and section 16.8, "GPIO") documents both
blocks. It is the primary reference for the per-pin function lists
(functions 0–7 for every pad).

## 2.1 Pin numbering

**[ABI]** Devicetree pin IDs run from 0 to 152. They are not the same as the
pad register index for the last 21 pins:

| Pin IDs | Pad names | Register offset from `0xD401E000` |
|---|---|---|
| 0–127 | `GPIO_00` … `GPIO_127` | `pin × 4` (`0x000`–`0x1FC`) |
| 128–130 | `PWR_SCL`, `PWR_SDA`, `VCXO_EN` | `pin × 4` (`0x200`–`0x208`) |
| 131–152 | see §2.1.1 | `(pin + 2) × 4` (`0x214`–`0x268`) |

**[HW]** Offsets `0x20C` and `0x210` have no pad. Do not write them.

*(vendor)* The vendor kernel also knows eleven eMMC pads (`EMMC_D0`–`D7`,
`DS`, `CLK`, `CMD`) as pin IDs 153–163, at `(pin + 2) × 4` (`0x26C`–`0x294`).
For those pins it does not use the pad's mux to make a GPIO; it sets bit 20
of APMU register `0x104` instead. Neither the user manual's pad table nor
mainline lists these pads.

> OPEN QUESTION: Whether pads 153–163 exist at those offsets on all K3
> parts, and what APMU `0x104` bit 20 does beyond the vendor's GPIO use.

### 2.1.1 Named pads

| Pin ID | Pad | Offset | Pin ID | Pad | Offset |
|---|---|---|---|---|---|
| 128 | `PWR_SCL` | `0x200` | 141 | `QSPI_DAT3` | `0x23C` |
| 129 | `PWR_SDA` | `0x204` | 142 | `QSPI_CS0` | `0x240` |
| 130 | `VCXO_EN` | `0x208` | 143 | `QSPI_CS1` | `0x244` |
| 131 | `PMIC_INT_N` | `0x214` | 144 | `QSPI_CLK` | `0x248` |
| 132 | `MMC1_DAT3` | `0x218` | 145 | `PRI_TDI` | `0x24C` |
| 133 | `MMC1_DAT2` | `0x21C` | 146 | `PRI_TMS` | `0x250` |
| 134 | `MMC1_DAT1` | `0x220` | 147 | `PRI_TCK` | `0x254` |
| 135 | `MMC1_DAT0` | `0x224` | 148 | `PRI_TDO` | `0x258` |
| 136 | `MMC1_CMD` | `0x228` | 149 | `PWR_SSP_SCLK` | `0x25C` |
| 137 | `MMC1_CLK` | `0x22C` | 150 | `PWR_SSP_FRM` | `0x260` |
| 138 | `QSPI_DAT0` | `0x230` | 151 | `PWR_SSP_TXD` | `0x264` |
| 139 | `QSPI_DAT1` | `0x234` | 152 | `PWR_SSP_RXD` | `0x268` |
| 140 | `QSPI_DAT2` | `0x238` | | | |

*(vendor)* The UEFI firmware numbers pads by register index (offset ÷ 4),
so its pad numbers for pins 131 and up are two higher than the devicetree
pin IDs. Both the mainline and vendor devicetrees use the pin IDs above.

### 2.1.2 Devicetree `pinmux` cells

**[ABI]** Each `pinmux` entry is one 32-bit cell: bits [31:16] hold the pin
ID and bits [15:0] the function number (0–7). The board devicetree builds
these values with a helper defined in the dual-licensed `k3-pinctrl.dtsi`.
The other properties in a pin-configuration node are the generic
pin-configuration properties, decoded as described in §2.3.

## 2.2 Pad register (MFPR)

One 32-bit register per pad. **[HW]** Field layout on K3:

| Bits | Field | Meaning |
|---|---|---|
| [2:0] | `MUX` | Function select, 0–7. See the manual's per-pad function tables. |
| [3] | `SPU` | Strong pull-up. Applies only when the pull-up is selected. |
| [4] | `EDGE_RISE` | Pad edge detect on rising edge (wake-up logic, §2.5) |
| [5] | `EDGE_FALL` | Pad edge detect on falling edge |
| [6] | `EDGE_CLEAR` | Pad edge-detect hold-off: while 1, the pad's latched edge is cleared and stays cleared (§2.5) |
| [7] | `SLEW_EN` | Output slew-rate control enable |
| [8] | `ST` | Input Schmitt trigger enable (K3 has one bit; K1 had two) |
| [12:9] | `DRIVE` | Drive strength code 0–15 (§2.2.1) |
| [13] | `PULL_DOWN` | Pull-down select |
| [14] | `PULL_UP` | Pull-up select |
| [15] | `PULL_EN` | Pull resistor enable |
| [31:16] | — | Reserved, write 0 |

**[HW]** Pull settings use bits [15:13] together:

| [15:13] | Bias |
|---|---|
| `000` | none |
| `101` | pull-down |
| `110` | pull-up (add `SPU` for strong pull-up) |

**[HW]** Bit 6 is a level, not a self-clearing pulse. *(vendor)* The vendor
kernel sets it to clear and hold off a pad's wake-up edge and clears it to
arm the pad. At start-up it sets bit 6 and clears bits 4–5 on every pad.
Set bit 6 on every pad that is not used for wake-up.

The sources disagree on slew rate. The UEFI firmware and the vendor kernel
define only the enable (bit 7) and treat [12:9] purely as drive strength;
the vendor kernel rejects a `slew-rate` property outright. Only mainline
Linux reads bits [12:11] as a slew-rate selector, overlapping the top of
the drive field. The user manual gives no MFPR field description. **[POLICY]**
Mainline maps the devicetree `slew-rate` values like this:

| `slew-rate` | Register effect |
|---|---|
| 0 | `SLEW_EN` = 0 |
| 1 | `SLEW_EN` = 1; rate follows whatever `DRIVE[3:2]` already holds |
| 2, 3, 4, 5 | `SLEW_EN` = 1 and bits [12:11] = value − 2 (slow 0, slow 1, medium, fast) |

[POLICY] When a node sets both `drive-strength` and an explicit slew rate
(2–5), Linux rejects the node unless the slew rate equals the value the
drive code already implies. A different driver may choose its own
precedence, but cannot set the two independently.

> OPEN QUESTION: Whether bits [12:11] select a slew rate at all, or are
> only the upper drive-strength bits. A safe driver sets `SLEW_EN` only
> when a board asks for it, and takes [12:9] from the drive table.

### 2.2.1 Drive strength

**[HW]** The drive code in [12:9] means a different current depending on
the pad supply voltage. Typical output-high current in mA:

| Code | 1.8 V | 3.3 V | Code | 1.8 V | 3.3 V |
|---|---|---|---|---|---|
| 0 | 2 | 3 | 8 | 21 | 25 |
| 1 | 4 | 5 | 9 | 23 | 27 |
| 2 | 6 | 7 | 10 | 25 | 29 |
| 3 | 7 | 9 | 11 | 26 | 31 |
| 4 | 9 | 11 | 12 | 28 | 33 |
| 5 | 11 | 13 | 13 | 30 | 35 |
| 6 | 13 | 15 | 14 | 31 | 37 |
| 7 | 14 | 17 | 15 | 33 | 38 |

**[ABI]** The devicetree `drive-strength` property is in mA and must be one
of the values in the column for the pad's voltage (the binding lists
exactly these). [POLICY] Linux converts a requested mA by picking the
smallest code whose current is at least the request, saturating at code 15.

*(vendor)* The UEFI firmware's default drive code for a pad it turns into
a GPIO is 3.

### 2.2.2 Pad supply type

**[HW]** Each pad has a fixed supply class:

| Pin IDs | Supply | GPIO function |
|---|---|---|
| 0–41 | switchable 1.8 V / 3.3 V | 0: GPIO line = pin ID |
| 42–75 | fixed 1.8 V | 0: GPIO line = pin ID |
| 76–127 | switchable 1.8 V / 3.3 V | 0: GPIO line = pin ID |
| 128–131 | fixed 1.8 V | none |
| 132–144 | switchable 1.8 V / 3.3 V | 1: an alternate pad for a lower GPIO line (table below) |
| 145–152 | fixed 1.8 V | 1: an alternate pad for a lower GPIO line (table below) |

**[HW]** Pads 128–131 (`PWR_SCL`, `PWR_SDA`, `VCXO_EN`, `PMIC_INT_N`) cannot
be GPIOs, per the user manual's pad function tables. Function 1 on
`PWR_SCL`/`PWR_SDA` is the real-time domain's power I2C. Both Linux
drivers list function 0 as "GPIO" for these pads, which is wrong; never
offer them as GPIO.

**[HW]** On pads 132–152, function 1 routes a line of the main GPIO block
(§2.4) to that pad as a second place it can appear:

| Pads (pin IDs) | GPIO lines |
|---|---|
| `QSPI_DAT0`–`DAT3` (138–141) | 3, 2, 1, 0 |
| `QSPI_CS0`, `QSPI_CS1`, `QSPI_CLK` (142–144) | 5, 6, 4 |
| `MMC1_DAT3`–`DAT0`, `MMC1_CMD`, `MMC1_CLK` (132–137) | 93–98 |
| `PWR_SSP_TXD`, `RXD`, `SCLK`, `FRM` (151, 152, 149, 150) | 120, 121, 122, 123 |
| `PRI_TDI`, `TMS`, `TCK`, `TDO` (145–148) | 124–127 |

The devicetree `gpio-ranges` only map GPIO line *n* to pin *n*. Using an
alternate pad means muxing it explicitly in a pin group.

> OPEN QUESTION: What happens when both the primary pad and an alternate
> pad select the same GPIO line (which input wins, and whether both drive).

Many pads also have an `R.GPIO[n]` function (usually function 3). That is
the real-time domain's GPIO block (§2.4.1), not the main one.

For "switchable" pads, the voltage the board actually supplies is given in
the devicetree with `power-source` (1800 or 3300). The pad's I/O
power-domain register must match it (§2.3).

## 2.3 I/O power-domain registers

**[HW]** A set of protected registers at pin-controller offset `0x800`
tells each switchable pad group whether its supply is 1.8 V or 3.3 V:

| Register (offset from `0xD401E000`) | Pads (pin IDs) |
|---|---|
| `0x804` | 0–20 |
| `0x80C` | 21–41 |
| `0x820` | 76–98 |
| `0x810` | 99–127 |
| `0x81C` | 132–137 (MMC1) |
| `0x82C` | 138–144 (QSPI) |

| Bit | Meaning |
|---|---|
| 2 | 1 = pad group runs at 1.8 V; 0 = 3.3 V |
| others | reserved, write 0 |

**[HW]** These registers are locked. Every single access, read or write,
must be preceded by an unlock:

1. Write `0x0000BABA` to APBC offset `0x050` (`0xD4015050`).
2. Write `0x0000EB10` to APBC offset `0x054` (`0xD4015054`).
3. Perform exactly one access to one I/O power-domain register.

The unlock allows one access and then relocks by itself. To check a write,
unlock again before reading it back. *(vendor)* The UEFI firmware does
exactly this and treats a mismatch on read-back as an error.

**[HW]** A wrong setting can damage the pads: telling a 3.3 V-supplied
group that it is at 1.8 V is the dangerous case. Program the domain only
from the board's `power-source` value, and only for groups that the board
description covers.

**[ABI]** The pin controller node carries `spacemit,apbc`, a phandle to the
APBC syscon, to locate the unlock registers. *(vendor)* The vendor
devicetree instead gives the pin controller a second `reg` window,
`0xD401E800` size `0x34`, for the power-domain registers, and passes the
unlock offset `0x50` as an argument to the `spacemit,apbc` phandle. [POLICY] Without that
property Linux skips voltage programming and leaves whatever the firmware
set.

[POLICY] Linux updates the domain register only while applying a drive
strength to a switchable pad, and it rejects a drive strength on a
switchable pad whose node has no `power-source`. A clean-room driver may
apply `power-source` independently of `drive-strength`.

*(vendor)* The vendor devicetree also describes voltage domains in a
separate node (`spacemit,pinctrl-voltage-domain`), which the UEFI firmware
applies once at start-up. Mainline does not use it.

## 2.4 GPIO controller

**[HW]** 128 GPIO lines in four banks of 32. Bank *n*, bit *b* is GPIO
`32n + b`, which is pad pin ID `32n + b` (the devicetree `gpio-ranges`
maps bank *n* to pins `32n` … `32n + 31`).

| Bank | Offset from `0xD4019000` |
|---|---|
| 0 | `0x000` |
| 1 | `0x040` |
| 2 | `0x080` |
| 3 | `0x100` |

Bank 3 is not at `0x0C0`.

### 2.4.1 Per-bank registers

Offsets from the bank base. All registers are 32 bits, one bit per line.

| Offset | Name | Access | Function |
|---|---|---|---|
| `0x00` | `PLR` | RO | Current pad level, whatever the direction |
| `0x04` | `PDR` | RW | Direction: 1 = output, 0 = input. Reset value 0. |
| `0x08` | `PSR` | WO | Write 1 to drive the line high (if it is an output) |
| `0x0C` | `PCR` | WO | Write 1 to drive the line low (if it is an output) |
| `0x10` | `RER` | RW | Rising-edge detect enable |
| `0x14` | `FER` | RW | Falling-edge detect enable |
| `0x18` | `EDR` | R/W1C | Edge-detect status. Write 1 to clear. |
| `0x1C` | `SDR` | WO | Write 1 to set the `PDR` bit (make output) |
| `0x20` | `CDR` | WO | Write 1 to clear the `PDR` bit (make input) |
| `0x24` | `SRER` | WO | Write 1 to set the `RER` bit |
| `0x28` | `CRER` | WO | Write 1 to clear the `RER` bit |
| `0x2C` | `SFER` | WO | Write 1 to set the `FER` bit |
| `0x30` | `CFER` | WO | Write 1 to clear the `FER` bit |
| `0x34` | `APMASK` | RW | Interrupt mask for the application processor: 1 = edge detect on this line raises the interrupt, 0 = masked. Reset value 0. |
| `0x38` | `CPMASK` | RW | Same format as `APMASK` (1 = not masked, reset 0), for a second interrupt output |

**[HW]** The output-set and output-clear registers cannot be read back. To
learn the driven level, read `PLR`.

The K1 has the same registers at a 12-byte stride with banks at `0x0`,
`0x4`, `0x8`, `0x100`. K3 packs each bank's registers contiguously.

No Linux driver or the UEFI firmware writes `CPMASK`. The user manual's
real-time CPU interrupt table has "GPIO" and "GPIO edge" sources (93, 94).
`CPMASK` very likely gates the GPIO interrupt to that CPU. **[POLICY]**
Leave it at its reset value of 0.

Other GPIO blocks with the same register layout, none of them in the
mainline or vendor devicetrees:

| Block | Base | Edge block | Interrupt |
|---|---|---|---|
| Real-time domain GPIO (`R.GPIO` pad functions) | `0xC0889400` | `0xC0889800` | real-time domain interrupt controller |
| Secure GPIO | `0xF0619000` | — | probably APLIC 59 |

### 2.4.2 Interrupts

- **[HW]** Interrupts are edge-only. An edge sets the line's `EDR` bit when
  the matching `RER`/`FER` bit is set. Setting both catches both edges.
  The `EDR` bit raises the controller interrupt only if the line's
  `APMASK` bit is 1.
- **[HW]** All four banks share one interrupt line (APLIC 58, level-high).
  On that interrupt, read `EDR` in every bank, write the value back to
  clear it, and dispatch the lines that are also set in `APMASK`.
- **[HW]** Edge detection by this block works only when the pad's `MUX`
  selects the GPIO function. It is separate from the pad edge-detect bits
  in the MFPR (§2.5).
- **[ABI]** `#interrupt-cells` and `#gpio-cells` are both 3: bank, line
  within the bank, flags. Level trigger types are invalid.

> OPEN QUESTION: The user manual says GPIO lines 0 and 1 each have their
> own interrupt. The K3 interrupt table lists no such sources, so this
> text is probably carried over from an older chip. Assume the single
> shared line.

**[POLICY]** Linux's sequence, given as a model:

| Event | Linux action |
|---|---|
| Probe, per bank | Write 0 to `APMASK`, `RER` and `FER`; write all-ones to `CRER` and `CFER` |
| Set trigger type | Record the wanted edges; write the line's bit to `SRER`/`CRER` and `SFER`/`CFER` to match |
| Mask | Clear the bit in a software copy of `APMASK` and write the copy; also clear the line's enabled edges via `CRER`/`CFER` |
| Unmask | Re-enable the recorded edges via `SRER`/`SFER`; set the bit in the software copy and write it to `APMASK` |
| Acknowledge | Write the line's bit to `EDR` |

Disabling the edge enables while masked means edges that happen while the
line is masked are lost, not latched. A different driver may keep the edge
enables on and rely on `APMASK` alone.

### 2.4.3 Using a pad as GPIO

1. **[HW]** Set the pad's `MUX` to function 0 (pins 0–127 carry GPIO line
   = pin ID). Function 1 on pins 132–152 routes a GPIO line to an
   alternate pad (§2.2.2).
2. Set bias and drive in the same MFPR as needed.
3. Set direction with `SDR` or `CDR`, and level with `PSR` or `PCR`.

[POLICY] When a line is claimed as GPIO, mainline Linux writes the whole
MFPR with only the GPIO function number, which clears any bias, drive and
edge settings in that register. The vendor kernel changes only `MUX`. The
UEFI firmware sets function 0, drive code 3 and `EDGE_CLEAR` (bit 6), and
keeps the pull it was asked for. Changing only `MUX` is the safer choice.

## 2.5 Pad edge detect (wake-up)

**[HW]** Each pad can latch an edge independently of the GPIO block, for
wake-up. The user manual's pad tables mark which pads support it.

| Item | Value |
|---|---|
| Edge enables | MFPR bits 4 (rising) and 5 (falling) |
| Clear / hold-off | MFPR bit 6 (§2.2): set to clear the latch and hold it clear, clear to arm |
| Status | "GPIO Edge" block at `0xD4019800` (0x800 window). One read-only 32-bit word per 32 pins at `+4 × (pin ÷ 32)`, bit `pin mod 32`. |
| Interrupt | APLIC source 60, level-high. It stays asserted while any armed pad has a latched edge. |

**[HW]** Writing to the status word does not clear it. The only way to drop
the interrupt is to set bit 6 on each pad whose status bit is set. A pad
that latches an edge with nobody handling it holds the interrupt
asserted, so the handler must clear every set status bit, including ones
it does not expect.

Sequence to use a pad as a wake source:

1. Set bit 6, then set bit 4 and/or 5 for the wanted edges.
2. Clear bit 6 to arm the pad.
3. On interrupt 60, read the status words. For each set bit, handle the
   pad, then set its bit 6 (and clear it again to re-arm).

*(vendor)* Mainline does not use any of this. The vendor devicetree makes
the pin controller an interrupt controller (`#interrupt-cells` = 2, parent
interrupt 60) and points it at the edge block with a `spacemit,gpio-edge`
syscon phandle covering only 16 bytes. Its boards use the pin controller as
interrupt parent for wake-up pins. These properties are not in the mainline
binding.

> OPEN QUESTION: Whether status bits above pin 130 follow the pin ID (as
> the vendor kernel assumes) or the pad register index. The vendor kernel
> also reads six status words while mapping only four.

## 2.6 What the devicetree gives the driver

| Property | Node | Meaning |
|---|---|---|
| `reg` | pinctrl | MFPR window, including the I/O power-domain registers at `+0x800` |
| `clocks`, `clock-names` (`func`, `bus`) | pinctrl | APBC AIB clocks |
| `spacemit,apbc` | pinctrl | APBC syscon for the unlock keys (§2.3) |
| `pinmux` | pin group | §2.1.2 |
| `bias-disable`, `bias-pull-down`, `bias-pull-up` (0 = normal, 1 = strong) | pin group | §2.2 |
| `drive-strength` | pin group | mA, §2.2.1 |
| `input-schmitt` | pin group | Bit 0 of the value goes to `ST`: 1 or 3 = on, 0 or 2 = off. The binding allows 0–3 because K1 has a 2-bit field. |
| `slew-rate` | pin group | §2.2 |
| `power-source` | pin group | 1800 or 3300, for switchable pads |
| `gpio-ranges` | gpio | four entries, bank *n* ↔ pins `32n`…`32n+31` |
| `clocks`, `clock-names` (`core`, `bus`) | gpio | APBC GPIO clocks |

## 2.7 FreeBSD notes

FreeBSD has no SpacemiT pin controller or GPIO driver for K1 or K3. The
GPIO driver needed is small. The pin controller is a single-register-per-pad
design that fits FreeBSD's FDT pinctrl framework. It needs only the
`pinmux` cell decoding, the drive tables, and the locked voltage-domain
access.
