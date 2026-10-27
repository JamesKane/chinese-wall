# 01: Clock and Reset Controllers

The K3 has no single clock controller. Clock generation and gating are spread
across five register blocks. Four of them also contain reset lines. All five
are plain memory-mapped registers with no firmware mediation: the operating
system programs them directly.

Clocks and resets are named here by the devicetree binding constants in
`dt-bindings/clock/spacemit,k3-clocks.h` and
`dt-bindings/reset/spacemit,k3-resets.h`. Both headers are dual-licensed
GPL-2.0-only OR BSD-2-Clause and may be used directly. The `CLK_` and
`RESET_` prefixes are dropped in the tables below.

> **[ABI] Stale header in the FreeBSD tree.** The copy of
> `spacemit,k3-clocks.h` under `sys/contrib/device-tree` was imported from an
> older Linux release. It lacks `MPMU_I2S_SYSCLK` (51),
> `MPMU_I2S_BCLK_FACTOR` (52), `MPMU_I2S1_SYSCLK_SRC` (53),
> `APMU_PCIE_PORTA_DBI`…`APMU_PCIE_PORTE_DBI` (89–93) and
> `APMU_UFS_REFCLK` (94). No other IDs changed. Refresh the header before
> building against it. The reset header and the clock-controller DT nodes are
> already current.

## 1.1 Register blocks

| Block | DT compatible | Base | Size | Clocks | Resets | DT input clocks |
|---|---|---|---|---|---|---|
| APBS (PLLs) | `spacemit,k3-pll` | `0xD4090000` | 64 KiB | yes | no | 24 MHz |
| MPMU | `spacemit,k3-syscon-mpmu` | `0xD4050000` | 64 KiB | yes | yes | 32 kHz, 1 MHz, 3 MHz, 24 MHz |
| APBC | `spacemit,k3-syscon-apbc` | `0xD4015000` | 4 KiB | yes | yes | 32 kHz, 1 MHz, 3 MHz, 24 MHz |
| APMU | `spacemit,k3-syscon-apmu` | `0xD4282800` | 1 KiB | yes | yes | 32 kHz, 1 MHz, 3 MHz, 24 MHz |
| DCIU | `spacemit,k3-syscon-dciu` | `0xD8440000` | 48 KiB | yes | yes | none |

- **[ABI]** Every block has `#clock-cells = <1>`. MPMU, APBC, APMU and DCIU
  also have `#reset-cells = <1>`. The cell is the binding ID from the headers.
  Each block has its own ID space starting at 0.
- **[ABI]** The PLL node has a `spacemit,mpmu` phandle to the MPMU node. The
  PLL lock status bits are in the MPMU block, not in APBS (§1.4.2).
- **[ABI]** The MPMU and APMU nodes declare `#power-domain-cells = <1>`, but no
  power-domain behavior exists in the baseline. Nothing in the baseline
  devicetree references these blocks as power domains.
- **[HW]** All registers are 32 bits wide. Every field update is a
  read-modify-write of the containing register. Many registers mix fields for
  several unrelated clocks and a reset, so a driver must never write a whole
  register with values that clobber its neighbours.

## 1.2 Input clocks

| Source | Rate | DT `clock-names` |
|---|---|---|
| Crystal-derived reference | 24 MHz | `vctcxo_24m` |
| Reference divider | 3 MHz | `vctcxo_3m` |
| Reference divider | 1 MHz | `vctcxo_1m` |
| Real-time oscillator | 32.768 kHz | `osc` |

- **[HW]** All eight PLLs take the 24 MHz reference.
- **[HW]** The CPU timebase (`timebase-frequency`) is also 24 MHz.
- **[HW]** Some mux inputs are described only as "external" (an off-chip
  clock). Others are "reserved". The baseline devicetree supplies neither.
  Treat both as unusable: never select those mux values.

> OPEN QUESTION: The Linux driver looks up the 32 kHz parent under a name
> that does not match the devicetree `clock-names` entry `osc`. On Linux that
> parent probably never resolves, so the affected clocks (slow UART gate, PWM
> option 1, timer option 1, RTC) would have no known rate. A new driver should
> bind the 32 kHz input by its DT name `osc`.

## 1.3 Building blocks

Every clock in the K3 is built from the elements below. A single register
often holds several elements of one clock: for example a mux field, a divider
field, a gate bit and a frequency-change bit.

### 1.3.1 Gate

- **[HW]** A gate has a bit mask. Normally the clock runs when all mask bits
  are 1 and stops when they are 0.
- **[HW]** A few gates use more than one bit. The APBC RTC function clock
  needs bits 1 and 7 both set.
- **[HW]** A few gates are inverted, where 1 means stopped. The EMAC reference
  clocks (§1.8.6) are inverted.

### 1.3.2 Fixed factor

- **[HW]** Output = parent × *mul* / *div*. No register is involved unless the
  factor is paired with a gate.

### 1.3.3 Mux

- **[HW]** A mux has a field (shift, width). The field value is the parent
  index directly, with no translation table.

### 1.3.4 Divider

- **[HW]** A divider has a field (shift, width). The divisor is *field + 1*.
  A 3-bit field divides by 1 to 8.

### 1.3.5 Frequency-change (FC) handshake

Some mux/divider clocks have an FC bit. It may sit in the same register or, for
some display clocks, in a different one.

- **[HW]** After changing the divider or mux field of such a clock, set its FC
  bit. Hardware applies the change and then clears the bit itself. Software
  must poll until the bit reads 0.
- **[POLICY]** Linux polls every 5 µs and gives up after 10 ms.
- **[POLICY]** Linux skips the FC write when a divider write would not change
  the value. It still issues FC after every mux change.

### 1.3.6 Fractional divider ("DDN")

- **[HW]** A register holds a numerator field *N* and a denominator field *D*.
  The block also has a fixed pre-divider *P*. Output = parent × *D* / (*P* × *N*).
- **[HW]** Both fields are written in one register update.
- **[POLICY]** Linux picks *N* and *D* as the best rational approximation of
  the requested rate within the field widths.

### 1.3.7 PLL

See §1.4.

## 1.4 PLLs (APBS block)

### 1.4.1 Registers

All K3 PLLs use the same three-register layout. In each PLL's `SWCR2`, bits
other than [16] and [15:8] gate that PLL's divided outputs (§1.5).

| PLL | `SWCR1` | `SWCR2` | `SWCR3` | Lock bit (MPMU `POSR` at `0x0010`) |
|---|---|---|---|---|
| PLL1 | `0x100` | `0x104` | `0x108` | bit 24 |
| PLL2 | `0x118` | `0x11C` | `0x120` | bit 25 |
| PLL3 | `0x124` | `0x128` | `0x12C` | bit 26 |
| PLL4 | `0x130` | `0x134` | `0x138` | bit 27 |
| PLL5 | `0x13C` | `0x140` | `0x144` | bit 28 |
| PLL6 | `0x148` | `0x14C` | `0x150` | bit 29 |
| PLL7 | `0x158` | `0x15C` | `0x160` | bit 30 |
| PLL8 | `0x180` | `0x184` | `0x188` | bit 31 |

| Register | Bits | Meaning |
|---|---|---|
| `SWCR1` | [31:0] | Configuration word (§1.4.4), written whole |
| `SWCR2` | [16] | PLL enable (1 = running) |
| `SWCR2` | [15:8] | Low byte of the fractional feedback (§1.4.4) |
| `SWCR3` | [31:0] | Configuration word (§1.4.4), written whole |

### 1.4.2 Enable, disable and lock

1. **[HW]** To enable a PLL, set `SWCR2` bit 16.
2. **[HW]** Wait for the PLL's lock bit in MPMU `POSR` to read 1.
   - **[POLICY]** Linux polls every 5 µs with a 3 ms timeout and reports
     failure on timeout.
3. **[HW]** To disable a PLL, clear `SWCR2` bit 16.

### 1.4.3 Changing a PLL's rate

- **[HW]** A PLL must be disabled while its configuration is rewritten. The
  sequence is:
  1. Move every consumer off the PLL.
  2. Disable the PLL.
  3. Write `SWCR1`.
  4. Write `SWCR3`.
  5. Update `SWCR2` bits [15:8] without touching the other `SWCR2` bits.
  6. Re-enable the PLL and wait for lock.
- **[HW]** A running CPU cluster clocked from PLL3/4/5/8 must therefore be
  switched to another mux parent (§1.8.1) before its PLL is reprogrammed.
- **[POLICY]** Linux treats the configuration as opaque. It supports only the
  exact register triples in §1.4.5 and reports the current rate by matching
  the registers against that table.
- **[POLICY]** At start-up, if a PLL's registers match no table entry, Linux
  disables it and programs the first table entry. A new driver should not do
  this for PLL1 or PLL2: almost every peripheral and bus clock depends on them,
  and firmware has already set them up.

### 1.4.4 Configuration word layout (inferred)

Linux never decodes the configuration words. The layout below was derived by
checking every table entry, and it reproduces all 32 entries in §1.4.5 to
within rounding. Use it to understand the values or to add new rates. The
known-good values in §1.4.5 remain authoritative.

| Field | Meaning |
|---|---|
| `SWCR1` [31:24] | Always `0x0B` in every table entry |
| `SWCR1` [23:16] | Integer feedback multiplier *I* |
| `SWCR1` [13:0] | High 14 bits of a 22-bit fraction *F* |
| `SWCR2` [15:8] | Low 8 bits of *F* |
| `SWCR3` [31:16] | `0xA045` (×1 rows) or `0xA055` (×2 rows). The two values differ only in bit 20. |
| `SWCR3` [20] | Doubler: 0 = ×1, 1 = ×2 |
| `SWCR3` [15:8], [7:0] | Two copies of a band byte that rises with output rate (`0x80`–`0x8E`) |

Output = 24 MHz × (*I* + *F* / 2²²) × (2 if `SWCR3`[20] else 1).

*(vendor)* The vendor UEFI firmware has one rate Linux lacks: 2600 MHz, with
`SWCR1` = `0x0B360AAA`, `SWCR2`[15:8] = `0xAB` and `SWCR3` = `0xA0558A8A`.
The formula predicts that entry exactly. The firmware source is
BSD-2-Clause-Patent, so the clean room may read it directly; see the README.
At boot, that firmware also turns on all of each PLL's divided outputs:
`SWCR2` bits 0–7 and bit 21.

> OPEN QUESTION: Several things here are unconfirmed. The doubler may act on
> the reference or on the output. The band byte probably selects a VCO or
> charge-pump range; its observed mapping is in the table below, but its rule
> is not known. The meaning of the constant fields is also unknown.

### 1.4.5 Supported rates

**[HW]** Fixed-rate PLLs. These must stay at these rates because peripheral
clock trees assume them.

| PLL | Rate | `SWCR1` | `SWCR2` [15:8] (shown in place) | `SWCR3` |
|---|---|---|---|---|
| PLL1 | 2457.6 MHz | `0x0B330CCC` | `0x0000CD00` | `0xA0558989` |
| PLL2 | 3000 MHz | `0x0B3E2000` | `0x00000000` | `0xA0558C8C` |
| PLL6 | 3200 MHz | `0x0B422AAA` | `0x0000AB00` | `0xA0558E8E` |
| PLL7 | 2800 MHz | `0x0B3A1555` | `0x00005500` | `0xA0558B8B` |

**[HW]** Variable PLLs (CPU clusters, §1.8.1). PLL3 and PLL4 support every
row. PLL5 and PLL8 support rows up to and including 2000 MHz.

| Rate (MHz) | `SWCR1` | `SWCR2` [15:8] | `SWCR3` |
|---|---|---|---|
| 1050 | `0x0B2B3000` | `0x00` | `0xA0458080` |
| 1100 | `0x0B2D3555` | `0x55` | `0xA0458181` |
| 1150 | `0x0B2F3AAA` | `0xAB` | `0xA0458181` |
| 1200 | `0x0B320000` | `0x00` | `0xA0458181` |
| 1250 | `0x0B340555` | `0x55` | `0xA0458181` |
| 1300 | `0x0B360AAA` | `0xAB` | `0xA0458282` |
| 1350 | `0x0B381000` | `0x00` | `0xA0458282` |
| 1400 | `0x0B3A1555` | `0x55` | `0xA0458282` |
| 1450 | `0x0B3C1AAA` | `0xAB` | `0xA0458383` |
| 1500 | `0x0B3E2000` | `0x00` | `0xA0458383` |
| 1550 | `0x0B402555` | `0x55` | `0xA0458383` |
| 1600 | `0x0B422AAA` | `0xAB` | `0xA0458484` |
| 1650 | `0x0B443000` | `0x00` | `0xA0458484` |
| 1700 | `0x0B463555` | `0x55` | `0xA0458484` |
| 1750 | `0x0B483AAA` | `0xAB` | `0xA0458585` |
| 1800 | `0x0B4B0000` | `0x00` | `0xA0458585` |
| 1850 | `0x0B4D0555` | `0x55` | `0xA0458585` |
| 1900 | `0x0B4F0AAA` | `0xAB` | `0xA0458686` |
| 1950 | `0x0B511000` | `0x00` | `0xA0458686` |
| 2000 | `0x0B292AAA` | `0xAB` | `0xA0558686` |
| 2050 | `0x0B2A2D55` | `0x55` | `0xA0558686` |
| 2100 | `0x0B2B3000` | `0x00` | `0xA0558787` |
| 2150 | `0x0B2C32AA` | `0xAB` | `0xA0558787` |
| 2200 | `0x0B2D3555` | `0x55` | `0xA0558787` |
| 2250 | `0x0B2E3800` | `0x00` | `0xA0558888` |
| 2300 | `0x0B2F3AAA` | `0xAB` | `0xA0558888` |
| 2350 | `0x0B303D55` | `0x55` | `0xA0558888` |
| 2400 | `0x0B320000` | `0x00` | `0xA0558989` |

- **[POLICY]** These rows are the Linux table. The upper limits (2.4 GHz for
  PLL3/PLL4, 2.0 GHz for PLL5/PLL8) are a reasonable guide to the clusters'
  limits. Voltage scaling is out of scope here. No OPP table or CPU supply
  exists in the baseline devicetree.

## 1.5 PLL output dividers (APBS block)

Each PLL output below is a fixed divider with a gate bit in that PLL's `SWCR2`.

| ID | Source | Divide | Gate bit (`SWCRn2`) | Rate |
|---|---|---|---|---|
| `PLL1_D2` … `PLL1_D8` (8–14) | PLL1 | 2…8 | bit (*n*−1), i.e. D2=bit1 … D8=bit7 | 1228.8, 819.2, 614.4, 491.52, 409.6, 351.09, 307.2 MHz |
| `PLL1_DX` (15) | PLL1 | variable: field [27:23] + 1 | bit 22 | 2457.6 / (1…32) MHz |
| `PLL1_D64` (16) | PLL1 | 64 | bit 31 | 38.4 MHz |
| `PLL1_D10_AUD` (17) | PLL1 | 10 | bit 21 | 245.76 MHz |
| `PLL1_D100_AUD` (18) | `PLL1_D10_AUD` | 10 (no gate) | — | 24.576 MHz |
| `PLLk_D1` … `PLLk_D8`, k = 2…8 | PLLk | 1…8 | bit (*n*−1), i.e. D1=bit0 … D8=bit7 | PLL rate / n |

The binding IDs for the PLLk_Dn outputs are listed in the header: PLL2 19–26,
PLL3 34–41, PLL4 42–49, PLL5 50–57, PLL6 58–65, PLL7 69–76, PLL8 77–84.

Ungated fixed factors derived from these outputs:

| ID | Parent | Factor | Rate |
|---|---|---|---|
| `PLL2_66` (27) | `PLL2_D5` | ÷9 | 66.67 MHz |
| `PLL2_33` (28) | `PLL2_66` | ÷2 | 33.33 MHz |
| `PLL2_50` (29) | `PLL2_D5` | ÷12 | 50 MHz |
| `PLL2_25` (30) | `PLL2_50` | ÷2 | 25 MHz |
| `PLL2_20` (31) | `PLL2_D5` | ÷30 | 20 MHz |
| `PLL2_D24_125` (32) | `PLL2_D3` | ÷8 | 125 MHz |
| `PLL2_D120_25` (33) | `PLL2_D3` | ÷40 | 25 MHz |
| `PLL6_80` (66) | `PLL6_D5` | ÷8 | 80 MHz |
| `PLL6_40` (67) | `PLL6_D5` | ÷16 | 40 MHz |
| `PLL6_20` (68) | `PLL6_D5` | ÷32 | 20 MHz |

- **[HW]** `PLL1_D8` (307.2 MHz) must never be gated. It is the root of the
  APB bus clock and most peripheral clocks.

## 1.6 MPMU clocks

### 1.6.1 Registers

| Register | Offset |
|---|---|
| `FCCR` | `0x0008` |
| `POSR` (PLL lock status) | `0x0010` |
| `SUCCR` | `0x0014` |
| `ISCCR0` | `0x0040` |
| `ISCCR1` | `0x0044` |
| `WDTPCR` | `0x0200` |
| `RIPCCR` | `0x0210` |
| `ACGR` | `0x1024` |
| `APBCSCR` | `0x1050` |
| `SUCCR_1` | `0x10B0` |
| `I2S0_SYSCLK` | `0x1100` |
| `I2S2_SYSCLK` | `0x1104` |
| `I2S3_SYSCLK` | `0x1108` |
| `I2S4_SYSCLK` | `0x110C` |
| `I2S5_SYSCLK` | `0x1110` |
| `I2S_SYSCLK_CTRL` | `0x1114` |

### 1.6.2 PLL1 branch gates (`ACGR`)

| ID | Parent | Factor | `ACGR` gate bit | Rate (MHz) |
|---|---|---|---|---|
| `MPMU_PLL1_307P2` (0) | `PLL1_D8` | 1 | 13 | 307.2 |
| `MPMU_PLL1_76P8` (1) | `MPMU_PLL1_307P2` | ÷4 | — | 76.8 |
| `MPMU_PLL1_61P44` (2) | `MPMU_PLL1_307P2` | ÷5 | — | 61.44 |
| `MPMU_PLL1_153P6` (3) | `PLL1_D8` | ÷2 | — | 153.6 |
| `MPMU_PLL1_102P4` (4) | `PLL1_D8` | ÷3 | 12 | 102.4 |
| `MPMU_PLL1_51P2` (5) | `PLL1_D8` | ÷6 | 7 | 51.2 |
| `MPMU_PLL1_51P2_AP` (6) | `PLL1_D8` | ÷6 | 11 | 51.2 |
| `MPMU_PLL1_57P6` (7) | `PLL1_D8` | ×3/16 | 8 | 57.6 |
| `MPMU_PLL1_25P6` (8) | `PLL1_D8` | ÷12 | 4 | 25.6 |
| `MPMU_PLL1_12P8` (9) | `PLL1_D8` | ÷24 | 3 | 12.8 |
| `MPMU_PLL1_12P8_WDT` (10) | `PLL1_D8` | ÷24 | 19 | 12.8 |
| `MPMU_PLL1_6P4` (11) | `PLL1_D8` | ÷48 | 2 | 6.4 |
| `MPMU_PLL1_3P2` (12) | `MPMU_PLL1_6P4` | ÷2 | — | 3.2 |
| `MPMU_PLL1_1P6` (13) | `MPMU_PLL1_6P4` | ÷4 | — | 1.6 |
| `MPMU_PLL1_0P8` (14) | `MPMU_PLL1_6P4` | ÷8 | — | 0.8 |
| `MPMU_PLL1_409P6` (15) | `PLL1_D6` | 1 | 0 | 409.6 |
| `MPMU_PLL1_204P8` (16) | `PLL1_D6` | ÷2 | 5 | 204.8 |
| `MPMU_PLL1_491` (17) | `PLL1_D5` | 1 | 21 | 491.52 |
| `MPMU_PLL1_245P76` (18) | `PLL1_D5` | ÷2 | 18 | 245.76 |
| `MPMU_PLL1_614` (19) | `PLL1_D4` | 1 | 15 | 614.4 |
| `MPMU_PLL1_47P26` (20) | `PLL1_D4` | ÷13 | 10 | 47.26 |
| `MPMU_PLL1_31P5` (21) | `PLL1_D4` | ×2/39 | 6 | 31.51 |
| `MPMU_PLL1_819` (22) | `PLL1_D3` | 1 | 14 | 819.2 |
| `MPMU_PLL1_1228` (23) | `PLL1_D2` | 1 | 16 | 1228.8 |

### 1.6.3 Bus, UART, watchdog and IPC clocks

| ID | Kind | Register | Fields | Parents / notes |
|---|---|---|---|---|
| `MPMU_APB` (24) | mux | `APBCSCR` | [1:0] | 0 = `MPMU_PLL1_25P6`, 1 = `MPMU_PLL1_51P2`, 2 = `MPMU_PLL1_25P6`, 3 = `MPMU_PLL1_102P4`. This is the APB bus clock and the parent of every APBC bus gate. |
| `MPMU_SLOW_UART` (25) | gate | `ACGR` | bit 1 | Parent: 32 kHz input |
| `MPMU_SLOW_UART1` (26) | fractional | `SUCCR` | *N* = [28:16], *D* = [12:0], *P* = 2 | Parent `MPMU_PLL1_153P6`. Intended for ≈14.7456 MHz UART baud bases. |
| `MPMU_SLOW_UART2` (27) | fractional | `SUCCR_1` | *N* = [28:16], *D* = [12:0], *P* = 2 | Parent `MPMU_PLL1_614`. Intended for ≈48 MHz. |
| `MPMU_WDT` (28) | gate | `WDTPCR` | bit 1 | Parent `MPMU_PLL1_25P6` |
| `MPMU_WDT_BUS` (29) | gate | `WDTPCR` | bit 0 | Parent `MPMU_APB` |
| `MPMU_RIPC` (30) | gate | `RIPCCR` | bit 0 | Parent `MPMU_APB` |

- **[POLICY]** Linux never gates `MPMU_SLOW_UART` automatically, even when it
  appears unused.

### 1.6.4 Audio clocks

| ID | Kind | Register | Fields | Parents / notes |
|---|---|---|---|---|
| `MPMU_I2S_153P6` (31) | factor | — | ÷2 | Parent `MPMU_PLL1_307P2` |
| `MPMU_I2S_153P6_BASE` (32) | mux | `FCCR` | [29] | 0 = `MPMU_I2S_153P6`, 1 = `MPMU_PLL1_307P2` |
| `MPMU_I2S_SYSCLK_SRC` (33) | mux + gate | `ISCCR1` | mux [30], gate bit 31 | 0 = `MPMU_PLL1_25P6`, 1 = `MPMU_I2S_153P6_BASE` |
| `MPMU_I2S_SYSCLK` (51) | fractional | `ISCCR1` | *N* = [14:0], *D* = [26:15], *P* = 1 | Parent `MPMU_I2S_SYSCLK_SRC` |
| `MPMU_I2S_BCLK_FACTOR` (52) | factor | — | ÷2 | Parent `MPMU_I2S_SYSCLK` |
| `MPMU_I2S_BCLK` (35) | divider + gate | `ISCCR1` | div [28:27], gate bit 29 | Parent `MPMU_I2S_BCLK_FACTOR`. In hardware this divider always includes the extra ÷2, so the overall rate is sysclk / (2 × (field+1)). |
| `MPMU_I2S1_SYSCLK_SRC` (53) | mux + gate | `ISCCR0` | mux [30], gate bit 31 | Same parents as `MPMU_I2S_SYSCLK_SRC` |
| `MPMU_I2S1_SYSCLK` (34) | fractional | `ISCCR0` | *N* = [14:0], *D* = [26:15], *P* = 1 | Parent `MPMU_I2S1_SYSCLK_SRC` |
| `MPMU_I2Sn_SYSCLK_SEL`, n = 0,2,3,4,5 (36–40) | mux | `I2S_SYSCLK_CTRL` | n=0: [1:0]; 2: [5:4]; 3: [13:12]; 4: [17:16]; 5: [21:20] | 0 = `MPMU_PLL1_614`, 1 = 24 MHz, 2 = `PLL2_D5`, 3 = 24 MHz |
| `MPMU_I2Sn_SYSCLK_DIV`, n = 0,2,3,4,5 (41–45) | fractional | `I2Sn_SYSCLK` | *N* = [15:0], *D* = [31:16], *P* = 2 | Parent `MPMU_I2Sn_SYSCLK_SEL` |
| `MPMU_I2S0_SYSCLK` (46) | gate | `I2S_SYSCLK_CTRL` | bit 2 | Parent `MPMU_I2S0_SYSCLK_DIV` |
| `MPMU_I2S2_SYSCLK` (47) | mux + gate | `I2S_SYSCLK_CTRL` | mux [8], gate bit 6 | 0 = `MPMU_I2S_SYSCLK`, 1 = `MPMU_I2S2_SYSCLK_DIV` |
| `MPMU_I2S3_SYSCLK` (48) | gate | `I2S_SYSCLK_CTRL` | bit 14 | Parent `MPMU_I2S3_SYSCLK_DIV` |
| `MPMU_I2S4_SYSCLK` (49) | gate | `I2S_SYSCLK_CTRL` | bit 18 | Parent `MPMU_I2S4_SYSCLK_DIV` |
| `MPMU_I2S5_SYSCLK` (50) | gate | `I2S_SYSCLK_CTRL` | bit 22 | Parent `MPMU_I2S5_SYSCLK_DIV` |

## 1.7 APBC clocks

### 1.7.1 Per-peripheral register layout

**[HW]** Each APBC peripheral has one 32-bit register with this layout:

| Bits | Meaning |
|---|---|
| 0 | Bus (APB) clock enable. Parent is `MPMU_APB`. |
| 1 | Functional clock enable |
| 2 | Reset: 1 = held in reset, 0 = released (§1.10.2) |
| 3 | SSP and SSPA only: selects the source for functional-mux option 7 (0 = 1 MHz reference, 1 = `MPMU_I2S_BCLK`) |
| [6:4] | Functional clock mux, when the peripheral has one |
| 7 | RTC only: second functional-enable bit, set together with bit 1 |

**[HW]** If an SSP or SSPA peripheral should run from `MPMU_I2S_BCLK`, bit 3
must be set to 1 as well as selecting mux option 7.

### 1.7.2 Register offsets

| Offset | Peripheral | Offset | Peripheral |
|---|---|---|---|
| `0x000` | UART0 | `0x070` | UART4 |
| `0x004` | UART2 | `0x074` | UART5 |
| `0x008` | GPIO | `0x078` | UART6 |
| `0x00C` | PWM0 | `0x07C` | SSP3 (SPI3) |
| `0x010` | PWM1 | `0x080` | SSPA0 (I2S0) |
| `0x014` | PWM2 | `0x084` | SSPA1 (I2S1) |
| `0x018` | PWM3 | `0x088` | SSPA2 (I2S2) |
| `0x01C` | IR1 | `0x08C` | SSPA3 (I2S3) |
| `0x020` | TWSI8 (I2C8) | `0x090` | IPC AP→RCPU |
| `0x024` | UART3 | `0x094` | UART7 |
| `0x028` | RTC | `0x098` | UART8 |
| `0x02C` | TWSI0 | `0x09C` | UART9 |
| `0x030` | TWSI1 | `0x0A0` | CAN0 |
| `0x034` | TIMERS0 | `0x0A4` | CAN1 |
| `0x038` | TWSI2 | `0x0A8`–`0x0E4` | PWM4 … PWM19 (4-byte stride) |
| `0x03C` | AIB | `0x11C`–`0x130` | TIMERS2 … TIMERS7 (4-byte stride) |
| `0x040` | TWSI4 | `0x148` | CAN2 |
| `0x044` | TIMERS1 | `0x14C` | CAN3 |
| `0x048` | ONEWIRE | `0x150` | CAN4 |
| `0x04C` | TWSI5 | `0x154` | UART10 |
| `0x058` | DRO | `0x158` | SSP0 (SPI0) |
| `0x05C` | IR0 | `0x15C` | SSP1 (SPI1) |
| `0x060` | TWSI6 | `0x160` | SSPA4 (I2S4) |
| `0x064` | counter clock select (not driven) | `0x164` | SSPA5 (I2S5) |
| `0x06C` | TSEN (thermal sensor) | | |

Offsets `0x050` and `0x054` are not clocks: they take the unlock keys for the
pin controller's I/O power-domain registers (02 §2.3).

There is no UART1, SSP2, TWSI3 or TWSI7 in this block. Those instances belong
to other (secure or real-time) domains not described in the baseline.

### 1.7.3 Functional clock sources

| Peripherals | Mux [6:4] options | Gate | Bus gate |
|---|---|---|---|
| UART0, 2–10 | 0 = `MPMU_PLL1_57P6`, 1 = `MPMU_SLOW_UART1`, 2 = `MPMU_SLOW_UART2` | bit 1 | bit 0 |
| GPIO | none; 24 MHz | bit 1 | bit 0 |
| PWM0–19 | 0 = `MPMU_PLL1_12P8`, 1 = 32 kHz | bit 1 | bit 0 |
| SSP0, SSP1, SSP3 (SPI) | 0 = `MPMU_PLL1_6P4`, 1 = `_12P8`, 2 = `_25P6`, 3 = `_51P2`, 4 = `_3P2`, 5 = `_1P6`, 6 = `_0P8`, 7 = bit-3 select | bit 1 | bit 0 |
| SSPA0–5 (I2S) | same as SSP | bit 1 | bit 0 |
| RTC | none; 32 kHz | bits 1 **and** 7 | bit 0 |
| TWSI0, 1, 2, 4, 5, 6, 8 (I2C) | 0 = `MPMU_PLL1_31P5`, 1 = `MPMU_PLL1_51P2`, 2 = `MPMU_PLL1_61P44` | bit 1 | bit 0 |
| TIMERS0–7 | 0 = `MPMU_PLL1_12P8`, 1 = 32 kHz, 2 = `MPMU_PLL1_6P4`, 3 = 3 MHz, 4 = 1 MHz | bit 1 | bit 0 |
| AIB, ONEWIRE | none; 24 MHz | bit 1 | bit 0 |
| DRO, IR0, IR1 | none; `MPMU_APB` | bit 1 | — |
| TSEN, IPC AP→RCPU | none; `MPMU_APB` | bit 1 | bit 0 |
| CAN0–4 | 0 = `PLL6_20`, 1 = `PLL6_40`, 2 = `PLL6_80` | bit 1 | bit 0 |

**[ABI]** APBC binding IDs follow the header. Each peripheral has a functional
ID and a `_BUS` ID. SPI and I2S also have `_I2S_BCLK` / `_BCLK` IDs for the
bit-3 selector.

## 1.8 APMU clocks

Clocks marked "FC" use the handshake in §1.3.5. Unless the table gives a
separate FC register, the FC bit is in the same register as the clock's
fields.

### 1.8.1 System and CPU clocks

| ID | Register | Mux | Div | FC bit | Gate | Parents (index order) |
|---|---|---|---|---|---|---|
| `APMU_AXICLK` (0) | `0x388` | [0] | [2:1] | 4 | — | `MPMU_PLL1_307P2`, `MPMU_PLL1_409P6` |
| `APMU_CCI550` (1) | `0x300` | [2:0] | [9:8] | 12 | — | `MPMU_PLL1_245P76`, `MPMU_PLL1_409P6`, `MPMU_PLL1_614`, `MPMU_PLL1_819`, `PLL7_D3`, `PLL2_D3`, `MPMU_PLL1_1228`, `PLL7_D2` |
| `APMU_CPU_C0_CORE` (2) | `0x38C` | [2:0] | [5:3] | 12 | — | `MPMU_PLL1_819`, `MPMU_PLL1_491`, `MPMU_PLL1_614`, `PLL2_D3`, `PLL3_D2`, `MPMU_PLL1_1228`, `PLL2_D2`, `PLL3_D1` |
| `APMU_CPU_C1_CORE` (3) | `0x390` | [2:0] | [5:3] | 12 | — | as C0, with PLL4 in place of PLL3 |
| `APMU_CPU_C2_CORE` (4) | `0x394` | [2:0] | [5:3] | 12 | — | as C0, with PLL5 in place of PLL3 |
| `APMU_CPU_C3_CORE` (5) | `0x208` | [2:0] | [5:3] | 12 | — | as C0, with PLL8 in place of PLL3 |
| `APMU_TOP_DCLK` (43) | `0x158` | [4:2] | [7:5] | 8 | bit 1 | `MPMU_PLL1_307P2`, `MPMU_PLL1_409P6`, `PLL3_D4`, `PLL6_D5`, `PLL7_D4`, `PLL6_D4`, `PLL7_D3`, `PLL6_D3` |
| `APMU_MC_AHB` (42) | `0x0E8` | — | — | — | bit 1 | `APMU_AXICLK` |
| `APMU_DMA` (37) | `0x064` | — | — | — | bit 3 | `APMU_AXICLK` |
| `APMU_DTC` (40) | `0x0AC` | — | — | — | bit 3 | `APMU_AXICLK` |
| `APMU_RCPU` (46) | `0x14C` | [9:7] | [6:4] | 15 | bit 12 | `PLL1_D10_AUD`, `MPMU_PLL1_307P2`, `MPMU_PLL1_491`, `MPMU_PLL1_409P6` |
| `APMU_D1P_*` (10–16) | `0x040` | — | — | — | bits 31, 30, 29, 28, 27, 26, 22 | Plain gates of `MPMU_PLL1_1228`, `_819`, `_614`, `_491`, `_409P6`, `_307P2`, `_245P76` |

- **[HW]** *(vendor)* Domains C0 and C1 clock the two four-core X100 clusters,
  which are the eight harts in the mainline devicetree. Domains C2 and C3
  clock two four-core clusters of A100 AI cores (harts 8–15). The mainline
  devicetree does not describe the A100 cores. The public K3 user manual
  gives each A100 cluster a 1.5 MiB scratchpad, which matches the "C2/C3 TCM
  pipe" clocks in §1.9.
- **[POLICY]** An OS that runs only on the X100 harts should leave C2, C3,
  PLL5 and PLL8 as firmware configured them.

> OPEN QUESTION: It is not yet confirmed that C0 serves mainline `cluster0`
> (harts 0–3) and C1 serves `cluster1` (harts 4–7). The vendor devicetree
> attaches both C0 and C1 to every X100 hart. Check on hardware before doing
> per-cluster frequency scaling.

### 1.8.2 Storage

| ID | Register | Mux | Div | FC bit | Gate | Parents |
|---|---|---|---|---|---|---|
| `APMU_SDH_AXI` (26) | `0x054` | — | — | — | bit 3 | `APMU_AXICLK` |
| `APMU_SDH0` (27) | `0x054` | [7:5] | [10:8] | 11 | bit 4 | 0 `MPMU_PLL1_409P6`, 1 `MPMU_PLL1_614`, 2 `PLL2_D8`, 3 `PLL2_D5`, 4–5 reserved, 6 `PLL1_DX` |
| `APMU_SDH1` (28) | `0x058` | [7:5] | [10:8] | 11 | bit 4 | as SDH0 |
| `APMU_SDH2` (29) | `0x0E0` | [7:5] | [10:8] | 11 | bit 4 | as SDH0, but index 3 = `MPMU_PLL1_819` |
| `APMU_QSPI` (35) | `0x060` | [8:6] | [11:9] | 12 | bit 4 | 0 `MPMU_PLL1_409P6`, 1 `PLL2_D8`, 2 `MPMU_PLL1_307P2`, 3 `MPMU_PLL1_245P76`, 4 reserved, 5 `PLL1_DX`, 6 `MPMU_PLL1_491`, 7 reserved |
| `APMU_QSPI_BUS` (36) | `0x060` | — | — | — | bit 3 | `APMU_AXICLK` |
| `APMU_UFS_ACLK` (53) | `0x268` | [4:2] | [7:5] | 8 | bit 1 | `MPMU_PLL1_491`, `MPMU_PLL1_409P6`, `PLL2_D6`, `PLL2_D5` |
| `APMU_UFS_REFCLK` (94) | — | — | ÷2 | — | — | `PLL1_D64` (gives 19.2 MHz) |

### 1.8.3 USB

All USB clocks are gates of `APMU_AXICLK` in `USB_CLK_RES_CTRL` (`0x05C`):

| ID | Bit |
|---|---|
| `APMU_USB2_BUS` (30) | 1 |
| `APMU_USB3_PORTA_BUS` (31) | 4 |
| `APMU_USB3_PORTB_BUS` (32) | 8 |
| `APMU_USB3_PORTC_BUS` (33) | 12 |
| `APMU_USB3_PORTD_BUS` (34) | 16 |

### 1.8.4 PCIe

Five ports, A–E. Each has its own register:

| Port | Register |
|---|---|
| A | `0x1F0` |
| B | `0x1D0` |
| C | `0x1C8` |
| D | `0x1E0` |
| E | `0x1E8` |

| Clock | Bit | Parent | IDs (A…E) |
|---|---|---|---|
| Master (`_MSTE`) | 2 | `PLL2_D6` (500 MHz) | 56, 58, 60, 62, 64 |
| Slave (`_SLV`) | 1 | `PLL2_D6` | 57, 59, 61, 63, 65 |
| DBI | 0 | `APMU_AXICLK` | 89–93 |

### 1.8.5 Ethernet

There are three MACs:

| MAC | Register |
|---|---|
| EMAC0 | `0x3E4` |
| EMAC1 | `0x3EC` |
| EMAC2 | `0x248` |

| Clock | Bits | Parent | Notes |
|---|---|---|---|
| `_BUS` | 0 | `APMU_AXICLK` | |
| `_REF` | 14, **inverted** (1 = off) | `PLL2_D120_25` (25 MHz) | |
| `_1588` | mux [15] | 0 = 24 MHz, 1 = `PLL2_D24_125` (125 MHz) | PTP clock |
| `_RGMII_TX` | 8 | `PLL2_D24_125` (125 MHz) | RGMII 1 Gb/s transmit clock |

IDs: EMAC0 66–69, EMAC1 70–73, EMAC2 74–77, in the order bus, ref, 1588,
rgmii_tx.

### 1.8.6 Display, camera, GPU, VPU and miscellaneous

These clocks matter only for blocks that have no driver at this baseline. They
are listed for completeness.

| ID | Register | Mux | Div | FC bit (reg) | Gate | Parents |
|---|---|---|---|---|---|---|
| `APMU_GPU` (41) | `0x0CC` | [20:18] | [14:12] | 15 | bit 4 | `MPMU_PLL1_614`, `MPMU_PLL1_491`, `MPMU_PLL1_819`, `MPMU_PLL1_409P6`, `MPMU_PLL1_1228`, `PLL2_D3`, `PLL2_D4`, `PLL2_D5` |
| `APMU_VPU` (39) | `0x0A4` | [12:10] | [15:13] | 21 | bit 3 | as GPU |
| `APMU_V2D` (17) | `0x044` | [13:12] | [11:9] | 28 | bit 8 | `MPMU_PLL1_491`, `PLL2_D4`, `MPMU_PLL1_307P2`, `MPMU_PLL1_614` |
| `APMU_DSI_ESC` (18) | `0x044` | [1:0] | — | — | bit 2 | `MPMU_PLL1_51P2_AP`, `MPMU_PLL1_47P26`, `MPMU_PLL1_25P6`, `MPMU_PLL1_76P8` |
| `APMU_LCD_HCLK` (19) | `0x044` | — | — | — | bit 5 | `APMU_AXICLK` |
| `APMU_LCD_DSC` (20) | `0x04C` | [31:29] | [27:25] | 26 (`0x044`) | bit 14 | `MPMU_PLL1_614`, `_491`, `_245P76`, `PLL7_D5`, `PLL2_D7`, `MPMU_PLL1_409P6`, `_51P2_AP`, `PLL2_D8` |
| `APMU_LCD_PXCLK` (21) | `0x04C` | [23:21] | [19:17] | 30 (`0x044`) | bit 16 | as LCD_DSC, except index 5 = `PLL2_D4` |
| `APMU_LCD_MCLK` (22) | `0x04C` | [7:5] | [4:1] | 29 (`0x044`) | bit 0 | `MPMU_PLL1_409P6`, `_491`, `_614`, `_307P2` |
| `APMU_DSI4LN2_DSI_ESC` (47) | `0x26C` | [0] | — | — | bit 2 | as DSI_ESC; only indices 0–1 are reachable |
| `APMU_DSI4LN2_LCD_DSC` (48) | `0x270` | [31:29] | [27:25] | 26 (`0x26C`) | bit 14 | `MPMU_PLL1_614`, `_491`, `PLL7_D5`, `PLL6_D6`, `PLL2_D7`, `MPMU_PLL1_409P6`, `_51P2_AP` |
| `APMU_DSI4LN2_LCD_PXCLK` (49) | `0x270` | [23:21] | [19:17] | 30 (`0x26C`) | bit 16 | `MPMU_PLL1_614`, `_491`, `PLL7_D5`, `PLL6_D6`, `PLL2_D7`, `PLL2_D4`, `MPMU_PLL1_51P2_AP`, `PLL2_D8` |
| `APMU_DSI4LN2_LCD_MCLK` (50) | `0x270` | [7:5] | [4:1] | 29 (`0x26C`) | bit 0 | as LCD_MCLK |
| `APMU_DSI4LN2_DPU_ACLK` (51) | `0x274` | [7:5] | [4:2] | 30 | bit 1 | `MPMU_PLL1_409P6`, `_491`, `_614`, `_307P2`, `PLL2_D4` |
| `APMU_DPU_ACLK` (52) | `0x274` | [22:20] | [19:17] | 31 | bit 16 | as DSI4LN2_DPU_ACLK |
| `APMU_EDP0_PXCLK` (54) | `0x23C` | [2] | — | — | bit 1 | 0 = `APMU_LCD_PXCLK`, 1 = external |
| `APMU_EDP1_PXCLK` (55) | `0x23C` | [18] | — | — | bit 17 | 0 = `APMU_DSI4LN2_LCD_PXCLK`, 1 = external |
| `APMU_CSI` (8) | `0x024` | [18:16] | [22:20] | 15 | bit 4 | `MPMU_PLL1_491`, `_409P6`, `_614`, `_819`, `PLL2_D2`, `PLL2_D3`, `PLL2_D4`, `MPMU_PLL1_1228` |
| `APMU_CCIC2PHY` (6) | `0x024` | [7] | — | — | bit 5 | `MPMU_PLL1_102P4`, `MPMU_PLL1_51P2_AP` |
| `APMU_CCIC3PHY` (7) | `0x024` | [31] | — | — | bit 30 | as CCIC2PHY |
| `APMU_ISP_BUS` (9) | `0x038` | [22:21] | [20:18] | 23 | bit 17 | `MPMU_PLL1_409P6`, `_491`, `_614`, `_245P76` |
| `APMU_CCIC_4X` (23) | `0x050` | [24:23] | [20:18] | 15 | bit 4 | as CSI (only indices 0–3 reachable) |
| `APMU_CCIC1PHY` (24) | `0x050` | [7] | — | — | bit 5 | as CCIC2PHY |
| `APMU_SC2_HCLK` (25) | `0x050` | [9:8] | [12:10] | 16 | bit 3 | `MPMU_PLL1_307P2`, `_614`, `_491`, `PLL2_D4` |
| `APMU_AES_WTM` (38) | `0x068` | [6] | — | — | bit 5 | `MPMU_PLL1_204P8`, `MPMU_PLL1_102P4` |
| `APMU_UCIE` (44) | `0x11C` | [6:4] | — | — | bit 0 | `MPMU_PLL1_307P2`, `_409P6`, `PLL3_D4`, `PLL6_D5`, `PLL7_D4`, `PLL6_D4` |
| `APMU_UCIE_SBCLK` (45) | `0x11C` | — | — | — | bit 8 | `APMU_AXICLK` |
| `APMU_ESPI_SCLK_SRC` (78) | `0x240` | [6:4] | — | — | — | `PLL2_20`, `PLL2_25`, `PLL2_33`, `PLL2_50`, `PLL2_66` |
| `APMU_ESPI_SCLK` (79) | `0x240` | [7] | — | — | bit 3 | 0 = external, 1 = `APMU_ESPI_SCLK_SRC` |
| `APMU_ESPI_MCLK` (80) | `0x240` | — | — | — | bit 1 | `APMU_AXICLK` |
| `APMU_CAM_SRC1…4` (81–84) | — | — | fixed | — | — | `MPMU_PLL1_409P6` ÷15; `PLL2_D5` ÷25; `PLL2_D6` ÷20; `MPMU_PLL1_409P6` ÷16 (27.3, 24, 25, 25.6 MHz) |
| `APMU_ISIM_VCLK0…3` (85–88) | `0x3F8` | [2:1], [10:9], [18:17], [26:25] | [6:3], [14:11], [22:19], [30:27] | — | bits 0, 8, 16, 24 | `APMU_CAM_SRC1` … `APMU_CAM_SRC4` (camera sensor master clocks) |

> OPEN QUESTION: The DSI4LN2 escape clock lists four parents, but its mux
> field is one bit wide. Either the field is really two bits wide (bits [1:0])
> or only two parents exist. Display is out of scope at this baseline.

## 1.9 DCIU clocks

| ID | Offset | Gate bit | Parent | Notes |
|---|---|---|---|---|
| `DCIU_HDMA` (0) | `0x234` | 0 | `APMU_AXICLK` | Must stay on (§1.11) |
| `DCIU_DMA350` (1) | `0x238` | 0 | `APMU_AXICLK` | |
| `DCIU_C2_TCM_PIPE` (2) | `0x244` | 0 | `APMU_AXICLK` | |
| `DCIU_C3_TCM_PIPE` (3) | `0x248` | 0 | `APMU_AXICLK` | |

## 1.10 Resets

### 1.10.1 Model

- **[HW]** Each reset line is a bit in one of the clock registers above.
- **[HW]** Lines have two polarities. "Assert-high" lines are in reset when
  the bit is 1. "Deassert-high" (active-low) lines are in reset when the bit
  is 0.
- **[HW]** Asserting or deasserting a line is a single read-modify-write of
  that one bit.
- **[HW]** No hardware sequencing is required beyond what the consuming device
  needs. Clocks should normally be running before a reset is released.

### 1.10.2 MPMU and APBC

- **MPMU (assert-high):**
  - `MPMU_WDT` (0): `WDTPCR` bit 2
  - `MPMU_RIPC` (1): `RIPCCR` bit 2
- **APBC (assert-high):** Every APBC peripheral's reset is bit 2 of its own
  register (§1.7.2). The binding ID → register mapping follows the peripheral
  name in the header. For example, `APBC_UART0` (0) → `0x000` and
  `APBC_TWSI8` (41) → `0x020`. `IPC_AP2AUD` (62) → `0x090`.

### 1.10.3 APMU

Unless a row says otherwise, lines are deassert-high: bit = 1 releases reset.

| ID | Register | Bit | ID | Register | Bit |
|---|---|---|---|---|---|
| `APMU_CSI` (0) | `0x024` | 1 | `APMU_DTC` (28) | `0x0AC` | 0 |
| `APMU_CCIC2PHY` (1) | `0x024` | 2 | `APMU_GPU` (29) | `0x0CC` | 1 |
| `APMU_CCIC3PHY` (2) | `0x024` | 29 | `APMU_MC` (31) | `0x0E8` | 0 |
| `APMU_ISP_CIBUS` (3) | `0x038` | 16 | `APMU_DSI4LN2_ESCCLK` (53) | `0x26C` | 3 |
| `APMU_DSI_ESC` (4) | `0x044` | 3 | `APMU_DSI4LN2_LCD_SW` (54) | `0x26C` | 4 |
| `APMU_LCD` (5) | `0x044` | 4 | `APMU_DSI4LN2_LCD_MCLK` (55) | `0x270` | 9 |
| `APMU_V2D` (6) | `0x044` | 27 | `APMU_DSI4LN2_LCD_DSCCLK` (56) | `0x270` | 15 |
| `APMU_LCD_MCLK` (7) | `0x04C` | 9 | `APMU_DSI4LN2_DPU_ACLK` (57) | `0x274` | 0 |
| `APMU_LCD_DSCCLK` (8) | `0x04C` | 15 | `APMU_DPU_ACLK` (58) | `0x274` | 15 |
| `APMU_SC2_HCLK` (9) | `0x050` | 0 | `APMU_UFS_ACLK` (59) | `0x268` | 0 |
| `APMU_CCIC_4X` (10) | `0x050` | 1 | `APMU_EDP0` (60) | `0x23C` | 0 |
| `APMU_CCIC1_PHY` (11) | `0x050` | 2 | `APMU_EDP1` (61) | `0x23C` | 16 |
| `APMU_SDH_AXI` (12) | `0x054` | 0 | `APMU_EMAC0` (67) | `0x3E4` | 1 |
| `APMU_SDH0` (13) | `0x054` | 1 | `APMU_EMAC1` (68) | `0x3EC` | 1 |
| `APMU_SDH1` (14) | `0x058` | 1 | `APMU_EMAC2` (69) | `0x248` | 1 |
| `APMU_SDH2` (15) | `0x0E0` | 1 | `APMU_ESPI_MCLK` (70) | `0x240` | 0 |
| `APMU_QSPI` (21) | `0x060` | 1 | `APMU_ESPI_SCLK` (71) | `0x240` | 2 |
| `APMU_QSPI_BUS` (22) | `0x060` | 0 | `APMU_RCPU_AUDIO_SYS` (78) | `0x14C` | 0 |
| `APMU_DMA` (23) | `0x064` | 0 | `APMU_RCPU_MCU_CORE` (79) | `0x14C` | 2 |
| `APMU_AES_WTM` (24) | `0x068` | 4 | `APMU_RCPU_AUDIO_APMU` (80) | `0x14C` | 3 |
| `APMU_MCB_DCLK` (25) | `0x06C` | 0 | | | |
| `APMU_MCB_ACLK` (26) | `0x06C` | 1 | | | |
| `APMU_VPU` (27) | `0x0A4` | 0 | | | |

USB resets, all in `0x05C` and all deassert-high:

| Port | AHB | VCC | PHY |
|---|---|---|---|
| USB2 | `APMU_USB2_AHB` (16) bit 0 | `APMU_USB2_VCC` (17) bit 2 | `APMU_USB2_PHY` (18) bit 3 |
| USB3 A | (19) bit 5 | (20) bit 6 | (51) bit 7 |
| USB3 B | (52) bit 9 | (62) bit 10 | (63) bit 11 |
| USB3 C | (64) bit 13 | (65) bit 14 | (66) bit 15 |
| USB3 D | (72) bit 17 | (73) bit 18 | (74) bit 19 |

PCIe resets are deassert-high, in each port's register (§1.8.4):

| Line | Bit | IDs (A…E) |
|---|---|---|
| DBI | 3 | 81, 84, 87, 90, 93 |
| Slave | 4 | 82, 85, 88, 91, 94 |
| Master | 5 | 83, 86, 89, 92, 95 |

CPU and die-to-die resets are **assert-high**. All are in `PMU_CC2_AP`
(`0x100`), except the UCIe lines:

| ID | Bit |
|---|---|
| `APMU_CPUn_POP`, n = 0…3 (32, 34, 36, 38) | 0, 3, 6, 9 |
| `APMU_CPUn_SW`, n = 0…3 (33, 35, 37, 39) | 1, 4, 7, 10 |
| `APMU_C0_MPSUB_SW` (40) | 12 |
| `APMU_CPUn_POP`, n = 4…7 (41, 43, 45, 47) | 16, 19, 22, 25 |
| `APMU_CPUn_SW`, n = 4…7 (42, 44, 46, 48) | 17, 20, 23, 26 |
| `APMU_C1_MPSUB_SW` (49) | 28 |
| `APMU_MPSUB_DBG` (50) | 29 |
| `APMU_UCIE_IP` (75) | `0x11C` bit 1 |
| `APMU_UCIE_HOT` (76) | `0x11C` bit 2 |
| `APMU_UCIE_MON` (77) | `0x11C` bit 3 |

- **[POLICY]** An OS normally leaves the CPU reset lines alone. Hart start and
  stop go through the SBI HSM extension.

> OPEN QUESTION: Binding ID `APMU_ALZO` (30) exists, but nothing describes its
> register or bit. Treat it as unimplemented.

### 1.10.4 DCIU

All DCIU lines are deassert-high on bit 0 of their register:

| ID | Offset |
|---|---|
| `DCIU_HDMA` (0) | `0x22C` |
| `DCIU_DMA350` (1) | `0x230` |
| `DCIU_DMA350_0` (2) | `0x204` |
| `DCIU_DMA350_1` (3) | `0x208` |
| `DCIU_AXIDMA0`…`7` (4–11) | `0x20C` … `0x228` (4-byte stride) |

## 1.11 Clocks that must stay on

**[HW]** Gating or disturbing any of these stops the CPUs, the interconnect,
or the bus that every peripheral uses:

- `PLL1`, `PLL1_D8`, and the PLL feeding each active CPU domain
- `MPMU_APB`, which has no gate but must keep a valid parent
- `APMU_CCI550` (cache-coherent interconnect)
- `APMU_CPU_C0_CORE` … `APMU_CPU_C3_CORE`
- `APMU_TOP_DCLK`
- `DCIU_HDMA`

**[POLICY]** Linux also leaves gated clocks alone at boot if firmware left
them running and a driver has not yet claimed them. It disables unused clocks
only late in boot. A new driver should not gate anything it does not own
during early bring-up.

## 1.12 Other clock domains not covered

The baseline names further register groups whose base addresses and
devicetree nodes are not given, so no driver at this baseline uses them:

- a real-time CPU subsystem with its own UART, I2C, SPI, PWM, CAN, timer and
  audio clocks
- a secure APB block with UART1, SSP2, TWSI3, an RTC, timers and GPIO

A clean-room driver does not need them to bring up the CoM260.
