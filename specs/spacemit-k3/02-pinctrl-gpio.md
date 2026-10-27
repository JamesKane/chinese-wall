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
| [6] | `EDGE_CLEAR` | Pad edge-detect clear / disable (see note below) |
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

> OPEN QUESTION: Bit 6 is named as a "clear" control in one source and
> as "no edge detection" in another. Whether it is a self-clearing
> status clear or a level that disables pad edge detection is not shown.
> Setting it on pads not used for wake-up is safe either way.

**[HW]** The slew-rate field overlaps the top of the drive field. When
`SLEW_EN` is set, bits [12:11] select the slew rate, so the two upper
drive-code bits double as the slew setting. The devicetree `slew-rate`
values map like this:

| `slew-rate` | Register effect |
|---|---|
| 0 | `SLEW_EN` = 0 |
| 1 | `SLEW_EN` = 1; rate follows whatever `DRIVE[3:2]` already holds |
| 2, 3, 4, 5 | `SLEW_EN` = 1 and bits [12:11] = value − 2 (slow 0, slow 1, medium, fast) |

[POLICY] When a node sets both `drive-strength` and an explicit slew rate
(2–5), Linux rejects the node unless the slew rate equals the value the
drive code already implies. A different driver may choose its own
precedence, but cannot set the two independently.

> OPEN QUESTION: Whether bits [12:11] mean "slew rate" or "drive strength"
> to the pad when `SLEW_EN` is 0. The sources only show that they are
> shared.

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

| Pin IDs | Supply | Default function for GPIO use |
|---|---|---|
| 0–41 | switchable 1.8 V / 3.3 V | 0 |
| 42–75 | fixed 1.8 V | 0 |
| 76–127 | switchable 1.8 V / 3.3 V | 0 |
| 128–131 | fixed 1.8 V | 0 |
| 132–144 | switchable 1.8 V / 3.3 V | 1 |
| 145–152 | fixed 1.8 V | 1 |

The last column is the function number that selects plain GPIO on that pad.

> OPEN QUESTION: Pins 128–152 have a GPIO function number, but the GPIO
> block (§2.4) has only 128 lines and its ranges map to pins 0–127. Which
> GPIO controller, if any, serves pads 128–152 is not shown. The manual's
> secure GPIO and audio GPIO instances (§2.4.1) are candidates.

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
APBC syscon, to locate the unlock registers. [POLICY] Without that
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
| `0x38` | `CPMASK` | RW | Interrupt mask for another processor |

**[HW]** The output-set and output-clear registers cannot be read back. To
learn the driven level, read `PLR`.

The K1 has the same registers at a 12-byte stride with banks at `0x0`,
`0x4`, `0x8`, `0x100`. K3 packs each bank's registers contiguously.

> OPEN QUESTION: `CPMASK` is undocumented in the manual and untouched by
> mainline. Presumably it routes edge interrupts to a co-processor (the
> real-time CPU). Leave it at its reset value.

The manual also lists GPIO blocks with the same layout at `0xC0889400`
(audio subsystem) and `0xF0619000` (secure domain). Neither appears in the
mainline devicetree.

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

1. **[HW]** Set the pad's `MUX` to the GPIO function number for that pad
   (§2.2.2): 0 for pins 0–131, 1 for pins 132–152.
2. Set bias and drive in the same MFPR as needed.
3. Set direction with `SDR` or `CDR`, and level with `PSR` or `PCR`.

[POLICY] When a line is claimed as GPIO, Linux writes the whole MFPR with
only the GPIO function number, which clears any bias, drive and edge
settings in that register. The UEFI firmware instead sets function 0,
drive code 3 and `EDGE_CLEAR` (bit 6), and keeps the pull it was asked
for. Either works; keeping the board's configured bias is the safer
choice.

## 2.5 Pad edge detect (wake-up)

**[HW]** The MFPR's `EDGE_RISE`, `EDGE_FALL` and `EDGE_CLEAR` bits feed a
wake-up path in the power management unit, separate from the GPIO block.
The manual's pad tables mark which pads support it. Mainline does not use
these bits.

> OPEN QUESTION: Where pad edge-detect status is read and how it routes
> to a wake-up interrupt on K3. Neither mainline nor the vendor firmware
> shows the status register.

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
