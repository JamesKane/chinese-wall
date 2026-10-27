# 01: Audio Clock and Reset Unit (CRU)

One 64 KiB register block at **`0x07110000`** holds the audio subsystem's
clock muxes, dividers and gates, its block-level software resets, the DSP
control words and two routing controls. It is exposed as:

| Boot | Clock provider | Reset provider | Register access |
|---|---|---|---|
| Mainline DT | `cix,sky1-audss-cru` (one node, both roles) | same node | the node's own `reg` |
| Vendor DT | child `cix,sky1-audss-clock` | child `cix,sky1-audss-reset` | parent register-region node |
| ACPI | `CIXH6061` | `CIXH6062` | `_DSD` `audss_cru` → `CIXHA018` (00 §0.6.1) |

All registers are 32-bit and accessed as whole words. **[HW]** The block is
in the audio power domain and needs the NoC reset released (00 §0.5).

## 1.1 Register map

| Offset | Name (mnemonic) | Content | Saved across power-off (§1.5) |
|---|---|---|---|
| `0x00` | `INFO_HIFI0` | DSP: `[0]` DSP core clock enable, `[1]` debug halt-on-reset (03 §3.5) | yes |
| `0x04` | `INFO_HIFI1` | DSP alternate reset vector (03 §3.5) | no |
| `0x08` | `INFO_HIFI2` | DSP run control (03 §3.5) | no |
| `0x10` | `INFO_CLK_GATE` | Clock gates (§1.1.1) | yes |
| `0x14` | `INFO_CLK_DIV` | Clock dividers (§1.1.2) | yes |
| `0x18` | `INFO_CLK_MUX` | I2S clock source selects (§1.1.3) | yes |
| `0x54` | `AP_IRQ` | DMA-350 interrupt routing: bit n = 1 sends DMA channel n's interrupt to the AP (03 §3.2.3) | no |
| `0x5C` | `I2S_LOOP` | `[1:0]` loopback source captured by I2S2: 0 none, 1 I2S0, 2 I2S1, 3 I2S3 | no |
| `0x70` | `INFO_MCLK` | MCLK output gates, selects and dividers (§1.1.4) | yes |
| `0x78` | `SW_RST` | Block software resets (§1.4) | no |

> OPEN QUESTION: Other offsets in the block are not used by any driver.
> Reset values are not known; Linux programs every field it uses.

### 1.1.1 `INFO_CLK_GATE` (`0x10`)

**[HW]** 1 = clock running, 0 = gated. Several clock outputs share one gate
bit. Gating a bit stops **all** of them.

| Bit | Gates |
|---|---|
| 0 … 9 | I2S n (n = bit): both its register (APB) clock and its core clock |
| 10 | Watchdog: APB and functional clocks |
| 11 | Timer: APB and functional clocks |
| 12 | Mailbox MB0 (AP→DSP) APB clock |
| 13 | Mailbox MB1 (DSP→AP) APB clock |
| 14 | HDA: system clock and 48 MHz link clock |
| 15 | DMA-350 AXI clock |
| 16 | DSP shared SRAM AXI clock |

- **[HW]** An OS must reference-count each **bit**, not each clock output.
  Linux models each output as a separate gate and relies on enable counts
  of the outputs; disabling one output of a shared bit (for example the
  I2S APB clock) while the other (the I2S core clock) is still in use
  stops both.

### 1.1.2 `INFO_CLK_DIV` (`0x14`)

**[HW]** All dividers are power-of-two: field value v divides by 2^v
(v = 0 … 3 → ÷1, ÷2, ÷4, ÷8).

| Bits | Divider |
|---|---|
| `[1:0]` | Shared bus divider: applied after the fixed pre-dividers of the DSP, DSP bus, DSP peripheral bus, SRAM AXI, HDA system, DMA AXI, timer APB and all ten I2S APB clocks (§1.2) |
| `[3 + 2n : 2 + 2n]`, n = 0 … 9 | I2S n core clock divider (bits `[3:2]` for I2S0 … `[21:20]` for I2S9) |

> OPEN QUESTION: Both mainline and vendor drivers attach the same `[1:0]`
> field to fourteen clocks, so changing one changes all. Whether the
> hardware really has one shared divider, or the drivers simplify, is not
> proven. Linux never changes it from its reset value; leave it alone.

### 1.1.3 `INFO_CLK_MUX` (`0x18`)

**[HW]** Field `[1 + 2n : 2n]`, n = 0 … 9, selects the source of I2S n's
core clock. Encodings:

| Value | Source | Valid for |
|---|---|---|
| 0 | `audio_clk0` (48 kHz family) | all I2S |
| 1 | I2S3, I2S4: `audio_clk1`. Others: `audio_clk2` | see note |
| 2 | `audio_clk2` (44.1 kHz family) | I2S3, I2S4 |
| 3 | `audio_clk3` | I2S3, I2S4 |

- **[HW]** I2S0–2 and I2S5–9 have a two-input mux: 0 = `audio_clk0`,
  1 = `audio_clk2`. I2S3 and I2S4 have a four-input mux: 0 … 3 =
  `audio_clk0` … `audio_clk3`.
- **[POLICY]** Linux selects `audio_clk0` for sample rates that are
  multiples of 8 kHz and `audio_clk2` for multiples of 11.025 kHz.
  `audio_clk1` and `audio_clk3` are never selected (mainline omits them;
  the vendor driver lists them but its I2S driver never picks them).

### 1.1.4 `INFO_MCLK` (`0x70`)

Five external master-clock outputs MCLK0 … MCLK4 (k = 0 … 4):

| Bits | Field |
|---|---|
| `[k]` | MCLK k gate (1 = running) |
| `[5 + k]` | MCLK k source: 0 = `audio_clk0`, 1 = `audio_clk2` |
| `[12 + 3k : 10 + 3k]` | MCLK k divider code |

**[HW]** *(vendor)* Observed divider codes (from the machine driver's rate
table):

| Code | Division | Examples |
|---|---|---|
| 0 | ÷6 | 192 kHz × 256 from `audio_clk0`; 176.4 kHz × 256 from `audio_clk2` |
| 2 | ÷12 | 48 kHz × 512, 96 kHz × 256; 44.1 kHz × 512, 88.2 kHz × 256 |
| 3 | ÷18 | 32 kHz × 512, 64 kHz × 256 |

> OPEN QUESTION: Codes 1 and 4–7 are not used. A linear rule does not fit
> the three observed points; do not extrapolate. Which board pads carry
> MCLK0 … MCLK4, and which I2S each belongs to, is per board (the vendor
> machine driver takes an `mclk-idx` per link).

## 1.2 Clock tree

### 1.2.1 Parents (SCMI clocks)

| Name | SCMI ID (`CLK_TREE_*`) | Binding name (mainline) | Default rate set by Linux | Family |
|---|---|---|---|---|
| `audio_clk0` | 76 | `x8k` | 294 912 000 Hz | 48 kHz × 6144 |
| `audio_clk1` | 77 | — (vendor only) | 344 064 000 Hz | *(vendor)* |
| `audio_clk2` | 78 | `x11k` | 270 950 400 Hz | 44.1 kHz × 6144 |
| `audio_clk3` | 79 | — (vendor only) | 316 108 800 Hz | *(vendor)* |
| `audio_clk4` | 70 | `sys` | 800 000 000 Hz | bus/DSP clock |
| `audio_clk5` | 71 | `48m` | 48 000 000 Hz | HDA link, timer, watchdog |

- **[HW]** The CRU outputs only run while their parents are enabled at the
  SCMI level. **[POLICY]** Linux enables all parents for as long as the
  CRU is powered and sets the rates above at probe.

### 1.2.2 Outputs and IDs

IDs are the index numbers of the public header `cix,sky1-audss-cru.h`
(identical in the vendor header). They are the provider cell in DT and
element 0 of CLKA entries (§1.6).

| ID | Output | Parent | Mux | Divider | Gate bit |
|---|---|---|---|---|---|
| 0 | `audio_clk4_div2` | `audio_clk4` ÷ 2 (fixed) | — | — | — |
| 1 | `audio_clk4_div4` | `audio_clk4` ÷ 4 (fixed) | — | — | — |
| 2 | `audio_clk5_div2` | `audio_clk5` ÷ 2 (fixed) | — | — | — |
| 3 | DSP core | `audio_clk4` | — | DIV `[1:0]` | `INFO_HIFI0[0]` |
| 4 | DSP bus | ID 0 | — | DIV `[1:0]` | none |
| 5 | DSP peripheral bus | ID 1 | — | DIV `[1:0]` | none |
| 6 | SRAM AXI | ID 0 | — | DIV `[1:0]` | 16 |
| 7 | HDA system | ID 0 | — | DIV `[1:0]` | 14 |
| 8 | HDA 48 MHz | `audio_clk5` | — | — | 14 |
| 9 | DMA AXI | ID 0 | — | DIV `[1:0]` | 15 |
| 10 | Watchdog APB | ID 2 | — | — | 10 |
| 11 | Watchdog functional | ID 2 | — | — | 10 |
| 12 | Timer APB | ID 1 | — | DIV `[1:0]` | 11 |
| 13 | Timer functional | ID 2 | — | — | 11 |
| 14 | MB0 APB (AP→DSP) | ID 1 | — | — | 12 |
| 15 | MB1 APB (DSP→AP) | ID 1 | — | — | 13 |
| 16 + n | I2S n APB, n = 0 … 9 | ID 1 | — | DIV `[1:0]` | n |
| 26 + n | I2S n core, n = 0 … 9 | §1.1.3 | MUX `[1+2n:2n]` | DIV `[3+2n:2+2n]` | n |
| 36 + k | MCLK k, k = 0 … 4 | `audio_clk0` / `audio_clk2` | MCLK `[5+k]` | (MCLK code, §1.1.4) | MCLK `[k]` |

Total: 41 outputs (IDs 0 … 40).

Nominal rates with all programmable dividers at ÷1: DSP 800 MHz; DSP bus,
SRAM, HDA system and DMA 400 MHz; DSP peripheral bus, mailbox, timer APB and
I2S APB 200 MHz; HDA link 48 MHz; watchdog and timer 24 MHz.

- **[POLICY]** The mainline MCLK outputs have no divider; the machine
  driver writes the divider field directly.
- **[POLICY]** The vendor driver marks every output "do not cache the
  rate" and ties output gating to the CRU's own power reference, so that
  enabling any output first powers the CRU.

## 1.3 Consumers (who uses which output)

| Consumer | Clock names (vendor DT / mainline DT) | IDs |
|---|---|---|
| HDA | `sysclk`, `clk48m` / `ipg`, `per` | 7, 8 |
| DMA-350 | `axiclk` | 9 |
| I2S n | `hst` (APB), `i2s` (core); plus SCMI `audio_clk0`, `audio_clk2` (and `audio_clk1`, `audio_clk3` for I2S3/4) as mux parents | 16 + n, 26 + n |
| DSP | `clk`, `bclk`, `pbclk`, `sramclk`, `mb0clk`, `mb1clk` | 3, 4, 5, 6, 14, 15 |

## 1.4 Block software resets (`SW_RST`, `0x78`)

**[HW]** One bit per block, **active-low**: 0 = held in reset, 1 = running.
Read-modify-write; the bit reads back the current state. Index = bit
number; the same numbers are used by the public header
`cix,sky1-audss-cru.h` (reset) and the vendor header.

| Index / bit | Block |
|---|---|
| 0 … 9 | I2S0 … I2S9 |
| 10 | Watchdog |
| 11 | Timer |
| 12 | Mailbox MB0 (AP→DSP) |
| 13 | Mailbox MB1 (DSP→AP) |
| 14 | HDA |
| 15 | DMA-350 |

- **[HW]** The DSP core reset and the audio NoC reset are **not** here; they
  are S5 resets 30 and 31 (`sky1-soc` Annex A).
- **[POLICY]** Mainline waits 50 µs after every assert and every release.
  The vendor driver has no wait on plain assert/release and waits 10–20 ms
  after each edge of a full pulse. Consumers add their own delay (I2S and
  DSP 1–2 µs, HDA 10–20 µs in the vendor driver).
- **[HW]** A block's clocks should run while its reset is pulsed, so the
  reset is seen synchronously.

## 1.5 Power-off and resume

**[HW]** The CRU loses its contents when the audio domain is off. Before
power-off, save `0x00`, `0x10`, `0x14`, `0x18` and `0x70`. After power-on,
in this order:

1. Enable the six SCMI parent clocks.
2. Release the NoC reset (mainline asserts it before disabling the parents
   at suspend and releases it here).
3. *(vendor)* Rewrite audio RCSU `0x34` = `0x20000000`.
4. Restore the five saved words.

- **[HW]** `SW_RST` is not restored. It comes back with blocks in reset (or
  in its reset state); each consumer pulses its own reset on resume.
- **[HW]** `AP_IRQ` (`0x54`) is not restored by the CRU. The DMA-350 driver
  sets channel bits again when channels are allocated.
- **[POLICY]** Linux runtime-suspends the CRU when no output is in use and
  uses the same path for system sleep.

## 1.6 ACPI clock lookup table `CLKA`

**[HW]** The `CIXH6061` device has a named object **`CLKA`** that evaluates
to a package of entries. Each entry is a package of at least 3 elements:

| Element | Type | Meaning |
|---|---|---|
| 0 | Integer | CRU output ID (§1.2.2, 0 … 40) |
| 1 | String | Connection name the consumer uses; empty = unnamed |
| 2 | Reference | Consumer device (HDA, DMA, I2S n, DSP, …) |

- **[HW]** It is the CRU's equivalent of the SoC-level `CLKT` table
  (`sky1-soc` §1.4.1): same element order, but element 0 indexes the CRU
  outputs instead of SCMI clocks.
- **[HW]** Skip entries whose ID is out of range or whose element 2 is not a
  device reference.
- **[HW]** Connection names can be longer than 16 characters elsewhere in
  CIX tables (for example `fch_timer_func_clk`, 18 characters). Allow at
  least 31.
- **[POLICY]** Sky1-Linux registers each entry as a (consumer, name) →
  clock lookup after all 41 outputs exist, then releases the `_DEP`
  dependents of `CIXH6061`.

> OPEN QUESTION: The connection-name strings in `CLKA` are not visible.
> The community DMA driver asks for an unnamed clock; the vendor I2S, HDA
> and DSP drivers ask for the vendor-DT names in §1.3. A clean driver
> should accept either the vendor names or a positional (unnamed) entry.

## 1.7 Probe sequence (summary)

**[HW]** Under ACPI, for `CIXH6061`:

1. Resolve `audss_cru` to the `CIXHA018` register window.
2. Transition to D0 (runs `PPRS._ON`).
3. Enable `audio_clk0` … `audio_clk5` (from `CLKT`), set the §1.2.1 rates.
4. Pulse S5 reset 31; write audio RCSU `0x34` and `0x1000`.
5. Create the 41 outputs; publish `CLKA` lookups; satisfy dependents.

For `CIXH6062`: resolve `audss_cru`, publish 16 resets, satisfy
dependents. **[HW]** It must not touch `SW_RST` before `CIXH6061` has
powered the domain.
