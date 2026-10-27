# 03: SPI and XSPI (Flash) Controllers

Sky1 has two general-purpose SPI controllers (Cadence SPI IP, revision
r1p6) and one serial-flash controller (Cadence "XSPI" family) in the FCH
domain. This section covers the SPI controllers in full and the flash
controller as far as the sources allow.

## 3.1 Which HID is which

**[ABI]** Driver match tables, not the public ACPI notes, decide the
HIDs:

| `_HID` | Device | Evidence |
|---|---|---|
| `CIXH2001` | General-purpose SPI (r1p6) | Matched by the Sky1-Linux SPI driver. The extracted DSDT has a `CIXH2001` device at `0x04090000`, IRQ GSIV 326, which is SPI0 |
| `CIXH2002` | Serial-flash controller | Matched by the Sky1-Linux *quad-SPI* driver (see §3.5) |
| `CIXH2011` | PWM, **not** SPI | Matched by the Sky1 PWM driver |
| `CIXH1005` | Watchdog | Matched by the Sky1 watchdog driver |

The public notes list `CIXH2011` as SPI and `CIXH2001` as watchdog. Both
are wrong. Their conversion tool therefore shows SPI0 as a "watchdog".

## 3.2 SPI instances

| Controller | Base | Size | IRQ (GIC SPI / GSIV) | SCMI clock | FCH reset index | Pin group | Pads used |
|---|---|---|---|---|---|---|---|
| SPI0 | `0x04090000` | 64 KiB | 294 / 326 | 250 (`FCH_SPI0_APB`) | 16 | `pinctrl_fch_spi0` | S5 pads `SPI1_MISO`, `SPI1_CS0`, `SPI1_CS1`, `SPI1_MOSI`, `SPI1_CLK` (function 0) |
| SPI1 | `0x040A0000` | 64 KiB | 295 / 327 | 251 (`FCH_SPI1_APB`) | 17 | `pinctrl_fch_spi1` | FCH pads `GPIO138`–`GPIO142` in function `SPI2_*` |

- **[HW]** Note the naming offset: controller SPI0 is wired to the pads
  called `SPI1_*` (in the S5 pin controller), and controller SPI1 to the
  `SPI2_*` pad function (in the FCH pin controller).
- **[HW]** Interrupts are level-high.
- **[HW]** One SCMI clock per controller feeds both the register (APB)
  clock and the bit-rate reference clock. The devicetree lists it twice,
  as `pclk` and `ref_clk`.
- **[HW]** Up to 4 chip selects per controller. Only CS0 and CS1 reach
  pads on SPI0 and SPI1.
- **[POLICY]** *(vendor)* Both reference boards leave SPI0/SPI1 disabled
  and only offer user-space SPI devices on them (O6: SPI0 CS0/CS1, SPI1
  CS0, 5 MHz).

### 3.2.1 Firmware description

- **[ABI]** ACPI: `_HID` `CIXH2001`; `_CRS` with `Memory32Fixed` and
  `Interrupt`; CLKT with the SCMI clock; RSTL `RST1` index 16/17.
- **[ABI]** Devicetree: `compatible = "cix,sky1-spi-r1p6", "cdns,spi-r1p6"`
  (mainline) or `"cdns,spi-r1p6"` (vendor); `clocks` + `clock-names =
  "ref_clk", "pclk"`; optional `num-cs` (default 4), `is-decoded-cs`,
  `resets`. The vendor devicetree also has `fifo-width = <32>`.

> OPEN QUESTION: The ACPI CLKT for SPI is not in the public examples.
> The Sky1-Linux driver asks for clocks by the names `pclk` and `ref_clk`.
> Whether CLKT provides two named entries, or one unnamed entry, is
> unknown. One clock is enough for a clean-room driver.

> OPEN QUESTION: The Linux driver asks for a reset named `spi`; the vendor
> devicetree names it `spi_reset`. The reset is optional, so Linux runs
> without it. Release reset index 16/17 explicitly.

## 3.3 SPI register map (Cadence SPI r1p6)

| Offset | Name | Access | Content |
|---|---|---|---|
| `0x00` | `CR` | RW | Configuration (§3.3.1) |
| `0x04` | `ISR` | R / W1C | Interrupt status (§3.3.2) |
| `0x08` | `IER` | WO | Write 1 = enable interrupt |
| `0x0C` | `IDR` | WO | Write 1 = disable interrupt |
| `0x10` | `IMR` | RO | 1 = interrupt enabled |
| `0x14` | `ER` | RW | Bit 0: 1 = controller enabled |
| `0x18` | `DR` | RW | Delay register (CS and inter-word delays). Not used by Linux |
| `0x1C` | `TXD` | WO | Push one word into the TX FIFO |
| `0x20` | `RXD` | RO | Pop one word from the RX FIFO |
| `0x24` | `SICR` | RW | Target-mode idle count. Not used in host mode |
| `0x28` | `THLD` | RW | TX FIFO threshold for the `TXOW` interrupt |

### 3.3.1 `CR`: configuration

| Bits | Name | Meaning |
|---|---|---|
| 0 | `MSTREN` | 1 = host (master) mode |
| 1 | `CPOL` | Clock polarity: 1 = SCLK idles high |
| 2 | `CPHA` | Clock phase: 1 = sample on the second edge |
| 5:3 | `BAUD_DIV` | SCLK = reference clock / 2^(`BAUD_DIV` + 1). Valid 1–7 (÷4 … ÷256) |
| 9 | `PERI_SEL` | 1 = chip-select field is a binary number for an external 3-to-8 decoder; 0 = one line per chip select |
| 13:10 | `CS` | Chip select. Direct mode: one-hot **active-low** (`1110` = CS0, `1101` = CS1, …, `1111` = none). Decoded mode: binary value |
| 14 | `SSFORCE` | 1 = chip select is driven from the `CS` field by software (manual CS) |
| 15 | `MANSTRTEN` | 1 = transfers start only on `MANSTRT`. Linux leaves it 0, so writing `TXD` starts the transfer |
| 16 | `MANSTRT` | Manual start trigger |

**[HW]** Host default: `MSTREN` | `CS` = `1111` | `SSFORCE` | `BAUD_DIV` = 1.

### 3.3.2 Interrupt bits

| Bit | Name | Meaning |
|---|---|---|
| 1 | `MODF` | Mode fault (another host drove the chip-select input) |
| 2 | `TXOW` | TX FIFO level below `THLD` |
| 3 | `TXFULL` | TX FIFO full |
| 4 | `RXNEMPTY` | RX FIFO not empty |
| 0, 5, 6 | – | Defined by the IP (all-interrupts mask is `0x7F`); not used by Linux |

> OPEN QUESTION: Bits 0, 5 and 6 are probably RX overflow, RX full and TX
> underflow, as in other users of this IP. The sources only prove that
> they exist.

### 3.3.3 FIFO depth

**[HW]** Probe it: write `0xFFFF` to `THLD` and read back. The value read
is (depth − 1); unimplemented high bits read as 0. Then set `THLD` back to
1.

## 3.4 SPI operation

### 3.4.1 Initialisation

1. **[HW]** Enable clock, release reset.
2. **[HW]** Probe the FIFO depth (§3.3.3).
3. **[HW]** `ER` = 0; `IDR` = `0x7F`; read `RXD` while `ISR.RXNEMPTY` = 1;
   `ISR` = `0x7F`; `CR` = host default (§3.3.1), with `PERI_SEL` if the
   board uses a decoder; `ER` = 1.
4. **[HW]** Redo step 3 after any loss of power or clock to the block.

### 3.4.2 Per message and per transfer

1. **[HW]** Mode change (CPOL/CPHA): disable (`ER` = 0), write `CR`,
   re-enable. Only when the value changes.
2. **[HW]** Rate: choose the smallest `BAUD_DIV` (1–7) for which reference
   / 2^(`BAUD_DIV` + 1) does not exceed the requested rate. **[POLICY]**
   Linux uses reference / 4 as the controller maximum.
3. **[HW]** Chip select: write the `CS` field (§3.3.1). Clear the field to
   `1111` to deselect. **[POLICY]** GPIO chip selects are also allowed
   and are handled outside the controller.
4. **[HW]** Data: SPI is full duplex. Every word pushed to `TXD` returns
   one word in `RXD`. Write zeros when there is nothing to send; read and
   drop words when there is nothing to receive.

### 3.4.3 Interrupt-driven transfer

1. **[HW]** Push up to FIFO-depth words into `TXD`.
2. **[HW]** Enable `TXOW` and `MODF`.
3. **[HW]** On `TXOW`: the number of words that left the FIFO equals the
   number of words owed to `RXD`. Read that many, then push the same
   number of new TX words. When fewer than half a FIFO of words remain,
   set `THLD` = 1 so the last interrupt comes when the FIFO is empty.
4. **[HW]** When no TX words remain: wait about 10 µs for the last word to
   finish shifting, read the remaining RX words, mask `TXOW`/`MODF`, and
   finish.
5. **[HW]** On `MODF`: mask the interrupts and fail the transfer.
6. **[HW]** If `ISR.TXFULL` is set while pushing, wait about 10 µs before
   the next write. The controller can drop a word written while it is
   busy.
7. **[POLICY]** At the end of a message, if no chip select is active,
   `ER` = 0, and `THLD` is put back to 1.

### 3.4.4 Word size (Sky1-specific)

- **[HW]** The Sky1 SPI accepts 8-, 16- and 32-bit words: each `TXD`
  write and `RXD` read carries one word of that size. The generic r1p6
  IP supports only 8-bit words.
- **[POLICY]** *(vendor)* Linux enables 16/32-bit words only when the
  devicetree compatible `cix,sky1-spi-r1p6` is present. Under ACPI
  (`CIXH2001`) it runs in 8-bit mode only.

> OPEN QUESTION: How the controller is told the word width is not in the
> sources. The driver only changes how many bytes it packs per FIFO
> access. The vendor property `fifo-width = <32>` suggests a 32-bit FIFO.
> Test with 8-bit words first.

### 3.4.5 Power management

**[POLICY]** Linux gates the clock after 3 s idle and re-runs §3.4.1 step
3 on system resume.

## 3.5 Serial-flash controller (XSPI)

### 3.5.1 Instance

| Item | Value |
|---|---|
| Register window | `0x04180000`, 64 KiB |
| Direct data window | `0x00010000`, 64 MiB (bus address `0x00010000`–`0x0400FFFF`) |
| IRQ | GIC SPI 302 / GSIV 334, level-high |
| Clocks (SCMI) | APB 252 (`FCH_XSPI_APB`), MAC 241 (`FCH_XSPI_MACLK`), function 240 (`FCH_XSPI_FUNC`) |
| Resets (FCH) | index 28 "system" and index 27 "register" (sky1-soc 02 §2.1.2) |
| Pins | `pinctrl_fch_xspi`: S5 pads `SE_QSPI_CLK`, `SE_QSPI_CS_L`, `SE_QSPI_DATA0`–`3` in function 1 (`QSPI_*`) |
| ACPI `_HID` | `CIXH2002` |

- **[HW]** *(vendor)* The vendor devicetree defines 200 MHz fixed clocks
  named for the XSPI, but its XSPI node uses the SCMI clocks above.
- **[POLICY]** *(vendor)* Both reference boards keep the XSPI disabled.
  The board devicetrees describe one JEDEC SPI-NOR on CS0 with 4-wire read
  at 25 MHz. That flash holds the boot firmware. **Leave this controller to
  firmware** unless there is a reason to use it. Writing the flash can
  brick the board.

### 3.5.2 Which IP is it

> OPEN QUESTION: The sources disagree on the controller type, and the
> two candidates have unrelated register maps:
> - The vendor devicetree uses `compatible = "cdns,xspi-nor"` (the Cadence
>   XSPI controller, with command, STIG and slave-DMA modes) and gives the
>   regions the names `io` and `sdma`. The public binding for that IP
>   needs a third `aux` region, which the vendor node does not have.
> - The Sky1-Linux ACPI support binds `CIXH2002` to the Cadence
>   **quad-SPI** (Octal-SPI) driver in its generic mode with direct
>   (memory-mapped) access turned off.
>
> No source shows either driver working on Sky1. A clean-room driver
> should identify the block at run time before using it (below), and
> should not use it at all if neither signature matches.

**[HW]** Identification: the Cadence XSPI controller has a version
register at `io` + `0xF00`. Bits [31:16] read `0x6522` on that IP; bits
[7:0] give the revision. The Cadence quad-SPI controller has its
configuration register at `0x00` and no such signature.

### 3.5.3 Cadence XSPI registers (only if identified as XSPI)

**[HW]** These are facts about the Cadence XSPI IP as used on other SoCs,
not verified on Sky1. They are enough for identification and a minimal
command-mode driver:

| Offset (`io`) | Name | Content |
|---|---|---|
| `0x0000`–`0x0014` | `CMD_REG_0`…`5` | Command words for one STIG (software-generated) command. Writing `CMD_REG_0` last issues it |
| `0x0044` | `CMD_STATUS` | Bit 15 completed, bit 14 failed, bit 3 DQS error, bit 2 CRC error, bit 1 bus error, bit 0 invalid sequence |
| `0x0100` | `CTRL_STATUS` | Bit 16 initialisation complete, bit 9 legacy mode, bit 8 initialisation failed, bit 7 controller busy |
| `0x0110` | `INTR_STATUS` | Bit 23 STIG done, bit 22 slave-DMA error, bit 21 slave-DMA trigger, bit 20 command ignored, bit 16 controller idle |
| `0x0114` | `INTR_ENABLE` | Bit 31 global enable; bits 23/22/21 as in `INTR_STATUS` |
| `0x0230` | `CTRL_CONFIG` | Bits [6:5] work mode: 0 direct, 1 STIG, 3 auto-command |
| `0x0240` | `SDMA_SIZE` | Byte count of the pending slave-DMA transfer |
| `0x0244` | `SDMA_TRD_INFO` | Bit 8: direction of the pending slave-DMA transfer, 0 = host reads the window, 1 = host writes it |
| `0x0F00` | `CTRL_VERSION` | [31:16] `0x6522`, [7:0] revision |
| `0x0F04` | `CTRL_FEATURES` | [25:24] number of banks (chip selects), bit 21 DMA data width, [3:0] number of threads |

- **[HW]** In STIG mode the data phase goes through the slave-DMA window
  (`sdma`, §3.5.1): when `INTR_STATUS` shows the slave-DMA trigger, the
  host reads or writes `SDMA_SIZE` bytes at the window, in the direction
  given by `SDMA_TRD_INFO`.
- **[HW]** Wait for `CTRL_STATUS` bit 16 after reset before issuing
  commands. If bit 8 is set, the controller's own initialisation failed.

> OPEN QUESTION: PHY/DLL tuning for the XSPI usually lives in the `aux`
> region, which Sky1 does not describe. At 25 MHz single-data-rate it may
> not be needed, but this is untested.
