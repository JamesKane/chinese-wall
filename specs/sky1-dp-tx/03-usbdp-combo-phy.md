# 03: USB-C / DisplayPort Combo PHY

This section covers the four **USB/DP combo PHYs** that carry the main-link
lanes of DisplayPort controllers DP0, DP1, DP3 and DP4 on the CIX Sky1. The
same PHY also carries USB 3 SuperSpeed traffic when the port is a USB-C
connector. DP2 uses a separate eDP PHY (04).

Sources: the community Sky1-Linux devicetree (CIX's BSD-3-Clause SoC and
board files), the CIX vendor PHY driver and the PHY-facing part of the
vendor DP driver. Almost every fact here comes only from vendor code and is
marked *(vendor)* where it matters. No mainline Linux driver or binding
for this PHY exists at the baseline.

Two register spaces are involved:

- the **PHY's own window** (§3.3, §3.4): the SerDes core (PMA) and its
  PHY-level wrapper. Programmed by whoever owns the PHY (USB and DP share it);
- a small **PHY interface window inside each DP controller** (§3.10): lane
  enables, resets, power-state handshake and AUX/HPD analog enables between
  the DP controller and the PHY.

The DP controller registers themselves are in sky1-dp-tx 01/02.

## 3.1 Instances

| PHY | MMIO (256 KiB) | `pclk` (SCMI clock) | PHY reset / APB reset (S5 index) | DP controller | USB controller | Index `n` |
|---|---|---|---|---|---|---|
| USB-C PHY 0 | `0x09030000` | `CLK_TREE_USB3C_DRD_PHY3_GATE` = 107 | 55 / 51 | DP0 | USB-C DRD 0 | 0 |
| USB-C PHY 1 | `0x090A0000` | `CLK_TREE_USB3C_0_PHY3_GATE` = 111 | 56 / 52 | DP1 | USB-C 1 | 1 |
| USB-C PHY 2 | `0x09110000` | `CLK_TREE_USB3C_1_PHY3_GATE` = 115 | 57 / 53 | DP3 | USB-C 2 | 2 |
| USB-C PHY 3 | `0x09180000` | `CLK_TREE_USB3C_2_PHY3_GATE` = 119 | 58 / 54 | DP4 | USB-C 3 | 3 |

- **[HW]** The reset indices are the S5 system-controller lines in word
  `0x408` (sky1-soc 02): `USB_DP_PHYn` (bit 4 + n) is the PHY core reset,
  `USB_DP_PHYn_PRST` (bit n) is the register-interface (APB) reset. Both are
  active-low.
- **[HW]** The PHY index `n` selects the per-PHY orientation bit (§3.5.3)
  and the per-PHY firmware handover byte (§3.12).
- **[HW]** The IP is a Cadence multi-protocol SerDes of the Torrent family.
  The vendor calls it "SD0804 ... Type-C". The PMA register names and offsets
  below match the Cadence Torrent register layout.
- The DP2 → eDP PHY mapping and the separate USB3-only PHY (`CIXH2034`,
  `0x09210000`, compatible `cix,sky1-usb3-phy`, clocks `apb_clk` + `ref_clk`)
  carry no DisplayPort and are outside this section.

## 3.2 Firmware description

### 3.2.1 Devicetree (`cix,sky1-usbdp-phy`)

| Property | Meaning |
|---|---|
| `reg` | One 256 KiB window (§3.1) |
| `resets` / `reset-names` | `"reset"` = PHY core reset, `"preset"` = APB reset |
| `clocks` / `clock-names` | `"pclk"`: APB/register clock |
| `cix,usbphy_syscon` | Phandle to the S5 system controller (`0x16000000`) holding the orientation bits |
| `orientation-switch` | Present: the PHY accepts plug-orientation updates from a Type-C controller |
| `mode-switch` | Present: the PHY accepts alt-mode (pin-assignment) updates from a Type-C controller |
| `default_conf` | u8. Initial mode: `1` USB only, `2` DP only (4 lanes), `3` USB + DP (2 + 2) |
| `svid` | `0xFF01` (the VESA DisplayPort SVID). The PHY driver ignores it |
| `port` / endpoint | Graph link to the Type-C connector's mux/orientation endpoint |
| alias `usbdpphyN` | Gives the PHY index `n` |
| child `usb-port` (`reg = 0`, `#phy-cells = 0`) | The USB3 PHY handle consumed by the USB controller |
| child `dp-port` (`reg = 1`, `#phy-cells = 0`) | The DP PHY handle, referenced by the DP controller as `phys` / `phy-names = "dp_phy"` |

- **[POLICY]** *(vendor)* With no `mode-switch` and no `default_conf`, the
  PHY assumes DP-only mode if it has a `dp-port` child and USB-only
  otherwise. A fixed DP output (no Type-C controller) must be in DP mode
  before the DP controller probes for the link, or the DP side waits forever
  for the PHY to report DP mode (§3.11.3).

### 3.2.2 ACPI (`CIXH2033`)

- **[HW]** `_HID` `CIXH2033`, one device per PHY. `_CRS`: one memory
  resource, the 256 KiB window.
- **[HW]** Clock `"pclk"` through the CLKT table (sky1-soc 01 §1.4). Resets
  `"reset"` and `"preset"` through RSTL (sky1-soc 02).
- **[HW]** `_DSD` properties: `id` (integer PHY index, used when there is no
  devicetree alias), `default_conf`, `orientation-switch`, `mode-switch`
  (as in §3.2.1).
- **[HW]** Child device nodes. A child whose name starts with `UDPP` is the DP
  PHY port. A child whose name starts with `USB` is the USB3 port. A
  consumer's `_DSD` reference names the child. For example, the DP controller
  has a `dp_phy` reference to `...UDPP`. The vendor resolves the provider as
  the child's parent device and the PHY by the child's name.
- **[HW]** There is no syscon reference under ACPI, so the OS does not write
  the orientation bit (§3.5.3). The vendor says firmware handles it.

> OPEN QUESTION: How Type-C orientation reaches the orientation bit under
> ACPI after a hot-plug is not explained. If firmware only sets it at boot,
> a flipped cable plugged in later would come up with swapped lanes. A
> clean-room driver should write `0x16000000 + 0x420` bit n itself when it
> can reach the S5 block.

## 3.3 Register access

- **[HW]** Each PHY register is **16 bits wide** and lives in a 32-bit slot.
  Byte address = window base + **register index × 4**. All indices in this
  section are register indices, not byte offsets. Use 32-bit accesses; the
  upper 16 bits are unused.
- **[HW]** Per-lane registers repeat with a lane stride of `0x200` indices:
  lane L register = lane-0 index + `0x200` × L, L = 0…3.

| Index range | Block |
|---|---|
| `0x0000`–`0x01FF` | PMA common (startup state machine, calibration, PLL0, PLL1) |
| `0x4000`–`0x47FF` | PMA transmitter / transceiver, per lane (`0x4000` + `0x200`·L) |
| `0x8000`–`0x87FF` | PMA receiver, per lane (`0x8000` + `0x200`·L) |
| `0xC000`–`0xD0FF` | PHY (PCS) level: lane map, PIPE/USB3 configuration, isolation |
| `0xE000`–`0xF0FF` | PMA control/status and PMA isolation |

## 3.4 Register map (registers the vendor touches)

### 3.4.1 Control and status

| Index | Name | Use |
|---|---|---|
| `0xE000` | `PHY_PMA_CMN_CTRL1` | bit 0 = common ready (read) |
| `0xE001` | `PHY_PMA_CMN_CTRL2` | Status: bit 0 PLL0 ready, 1 PLL1 ready, 2 PLL0 disabled, 3 PLL1 disabled, 4 PLL0 clock-enable ack, 5 PLL1 clock-enable ack, 6 PLL0 locked, 7 PLL1 locked |
| `0xE003` | `PHY_PMA_PLL_CTRL` (raw PLL control) | bit 0 = PLL0 reset release (1 = run), bit 1 = PLL1 reset release |
| `0xE004` | `PHY_PMA_ISO_CMN_CTRL` | Isolation (not written) |
| `0xE005` | `PHY_PMA_ISO_PLL_CTRL0` | Written `0x000B` |
| `0xE006` | `PHY_PMA_ISO_PLL_CTRL1` | Written `0x2224` |
| `0xC010` | `PHY_PMA_LANE_MAP` | PHY-lane to PMA-lane mapping (§3.5.2) |
| `0xC011` | `PHY_LANE_OFF_CTRL` | Written `0x0100` |
| `0xC018`, `0xC01A`, `0xC01B` | `PHY_PIPE_USB3_GEN2_PRE_CFG0`, `_POST_CFG0`, `_POST_CFG1` | USB 3 Gen 2 PIPE configuration |

> OPEN QUESTION: The vendor header also defines `PHY_PLL_CFG` at `0xC00E`
> and alternative USB3 Gen 2 PIPE registers at `0xC01C`–`0xC01F`, labelled
> for an "0803" PHY revision. The eDP PHY (04) uses the `0xC00E`/`0xC01C`
> layout. The combo PHY on Sky1 is written only at `0xC018`–`0xC01B`.

### 3.4.2 PMA common (PLL n: PLL0 base, PLL1 = PLL0 + `0x40`; PDIAG PLL1 = PLL0 + `0x20`)

| PLL0 | PLL1 | Name | Meaning (inferred from Cadence naming) |
|---|---|---|---|
| `0x0090` | `0x00D0` | `CMN_PLLn_INTDIV_M0` | Feedback divider, integer part |
| `0x0091` | `0x00D1` | `CMN_PLLn_FRACDIVL_M0` | Feedback divider, fraction (/65536) |
| `0x0092` | `0x00D2` | `CMN_PLLn_FRACDIVH_M0` | Fraction high / mode |
| `0x0093` | `0x00D3` | `CMN_PLLn_HIGH_THR_M0` | Lock-detect high threshold |
| `0x0094` | `0x00D4` | `CMN_PLLn_DSM_DIAG_M0` | Delta-sigma modulator diagnostics |
| `0x0098`–`0x009B` | `0x00D8`–`0x00DB` | `CMN_PLLn_SS_CTRL1..4_M0` | Spread-spectrum control |
| `0x0082`, `0x0084`–`0x0086`, `0x0088` | `0x00C2`, `0x00C4`–`0x00C6`, `0x00C8` | `VCOCAL_TCTRL`, `VCOCAL_INIT_TMR`, `VCOCAL_ITER_TMR`, `VCOCAL_REFTIM_START`, `VCOCAL_PLLCNT_START` | VCO calibration |
| `0x009C`, `0x009E`, `0x009F` | `0x00DC`, `0x00DE`, `0x00DF` | `LOCK_REFCNT_START`, `LOCK_PLLCNT_START`, `LOCK_PLLCNT_THR` | Lock detector |
| `0x01A0` | `0x01C0` | `CMN_PDIAG_PLLn_CTRL_M0` | PLL control |
| `0x01A1` | `0x01C1` | `CMN_PDIAG_PLLn_CLK_SEL_M0` | Output clock select / post-divider |
| `0x01A3` | `0x01C3` | `CMN_PDIAG_PLLn_ITRIM_M0` | Current trim |
| `0x01A4`–`0x01A6` | `0x01C4`–`0x01C6` | `CP_PADJ`, `CP_IADJ`, `FILT_PADJ` | Charge pump / loop filter |

Other common registers written: `CMN_SSM_BIAS_TMR` `0x0022`,
`CMN_PLLSM0_PLLPRE_TMR` `0x002A`, `CMN_PLLSM0_PLLLOCK_TMR` `0x002C`,
`CMN_PLLSM1_PLLPRE_TMR` `0x0032`, `CMN_PLLSM1_PLLLOCK_TMR` `0x0034`,
`CMN_CDIAG_CDB_PWRI_OVRD` `0x0041`, `CMN_CDIAG_XCVRC_PWRI_OVRD` `0x0047`,
`CMN_BGCAL_INIT_TMR` `0x0064`, `CMN_BGCAL_ITER_TMR` `0x0065`,
`CMN_IBCAL_INIT_TMR` `0x0074`, `CMN_TXPUCAL_INIT_TMR` `0x0104`,
`CMN_TXPDCAL_INIT_TMR` `0x010C`, `CMN_RXCAL_INIT_TMR` `0x0114`,
`CMN_SD_CAL_PLLCNT_START` `0x0128`, `CMN_DIAG_GPANA_0` `0x01F0`.

### 3.4.3 Per-lane (lane-0 index; add `0x200` × L)

| Index | Name | Index | Name |
|---|---|---|---|
| `0x4040` | `TX_TXCC_CTRL` | `0x8000`–`0x8003` | `RX_PSC_A0..A3` |
| `0x404C` | `TX_TXCC_CPOST_MULT_00` | `0x8006` | `RX_PSC_CAL` |
| `0x404D` | `TX_TXCC_CPOST_MULT_01` | `0x8044`, `0x8045` | `RX_SDCAL0_INIT_TMR`, `_ITER_TMR` |
| `0x4050` | `TX_TXCC_MGNFS_MULT_000` | `0x804C`, `0x804D` | `RX_SDCAL1_INIT_TMR`, `_ITER_TMR` |
| `0x40C6` | `DRV_DIAG_TX_DRV` | `0x8080`, `0x8082` | `RX_CDRLF_CNFG`, `RX_CDRLF_CNFG3` |
| `0x40E5` | `XCVR_DIAG_PLLDRC_CTRL` | `0x8090` | `RX_SIGDET_HL_FILT_TMR` |
| `0x40E6` | `XCVR_DIAG_HSCLK_SEL` | `0x8108`, `0x8110`, `0x8118` | `RX_REE_GCSM1_CTRL`, `GCSM2_CTRL`, `PERGCSM_CTRL` |
| `0x40E7` | `XCVR_DIAG_HSCLK_DIV` | `0x8149` | `RX_REE_ATTEN_THR` |
| `0x40EA` | `XCVR_DIAG_BIDI_CTRL` | `0x8177`, `0x8178` | `RX_REE_SMGM_CTRL1`, `_CTRL2` |
| `0x40EB` | `XCVR_DIAG_PSC_OVRD` | `0x81E2`, `0x81E3` | `RX_DIAG_DFE_AMP_TUNE_2`, `_3` |
| `0x4100`–`0x4103` | `TX_PSC_A0..A3` (TX power-state configuration) | `0x81E5` | `RX_DIAG_NQST_CTRL` |
| `0x4123` | `TX_RCVDET_ST_TMR` | `0x81E8` | `RX_DIAG_SIGDET_TUNE` |
| `0x41E7` | `TX_DIAG_ACYA` | `0x81F4`, `0x81F5` | `RX_DIAG_PI_RATE`, `RX_DIAG_PI_CAP` |
| | | `0x81FF` | `RX_DIAG_ACYA` |

## 3.5 Modes and lane assignment

### 3.5.1 Modes

| Mode code | Mode | DP lanes | USB 3 | Type-C DP pin assignment / state |
|---|---|---|---|---|
| 0 | None | 0 | no | Not configured |
| 1 | USB only | 0 | yes | USB, safe state, or anything not listed |
| 2 | DP only | 4 | no | C or E |
| 3 | USB + DP | 2 | yes | D |

- **[HW]** The code is a bit mask: bit 0 = USB, bit 1 = DP.
- **[HW]** The PHY reports a usable DP lane count of 4 (DP only), 2 (USB +
  DP) or 0 to the DP controller. The controller must cap its lane count to
  this value. With USB + DP, DP can train at 1 or 2 lanes.
- **[HW]** Mode 1 and mode 3 use the **same** PHY initialisation table
  (§3.7). Mode 1 just leaves the DP lanes unused.

### 3.5.2 PHY lane to PMA lane mapping *(vendor)*

The four PMA lanes (0–3) connect to the Type-C high-speed pairs. The vendor
documents this mapping:

| PMA lane | Normal, C/E (DP 4-lane) | Normal, D (USB + DP) | Flipped, C/E | Flipped, D |
|---|---|---|---|---|
| 0 | DP lane 2 | USB TX | DP lane 1 | DP lane 1 |
| 1 | DP lane 3 | USB RX | DP lane 0 | DP lane 0 |
| 2 | DP lane 0 | DP lane 0 | DP lane 3 | USB RX |
| 3 | DP lane 1 | DP lane 1 | DP lane 2 | USB TX |

So with USB + DP, the **DP lanes are PMA 2/3 (normal) or PMA 1/0
(flipped)**. The USB TX/RX pair is PMA 0/1 (normal) or PMA 3/2 (flipped).
DP lane 0 is PMA 2 (normal) or PMA 1 (flipped).

- **[HW]** `PHY_PMA_LANE_MAP` (`0xC010`) = `0x5100` in USB + DP mode
  (**both** orientations) and `0x51D9` in DP-only mode.

> OPEN QUESTION: The encoding of `PHY_PMA_LANE_MAP` is not given. The value
> does not change with orientation in USB + DP mode, so the swap must
> happen in the per-lane programming (§3.7) plus the S5 orientation bit
> (§3.5.3). In DP-only mode a single table is used for both orientations,
> so the reversal of DP lane order on a flipped plug relies entirely on the
> orientation bit. Test 4-lane DP on a flipped cable.

### 3.5.3 Orientation bit (S5 system controller)

| Address | Bits | Meaning |
|---|---|---|
| `0x16000000 + 0x420` | bit n (n = PHY index 0…3) | 0 = normal plug orientation, 1 = flipped (reverse) |

- **[HW]** *(vendor)* Write this bit read-modify-write **before** taking the
  PHY out of reset, then wait at least 10 µs.
- **[HW]** The bit, the lane map and the per-lane tables must all agree. A
  change of orientation therefore needs a full PHY re-initialisation
  (§3.6).
- **[POLICY]** *(vendor)* An "orientation: none" (unplugged) update is
  ignored. The last known orientation is kept.

## 3.6 Power-up, reset and initialisation

**[HW]** *(vendor)* Bring-up sequence of the PHY core (performed once for
the shared PHY, whichever of USB or DP comes first):

1. Choose the mode (§3.5.1) and the orientation.
2. If the S5 block is reachable, set or clear bit n of S5 `0x420` per the
   orientation and wait 10 µs.
3. Assert both the PHY core reset and the APB reset.
4. Enable `pclk`.
5. Release the APB reset. Registers are now writable. The PHY core stays in
   reset.
6. Write the initialisation table for the mode and orientation (§3.7), in
   the listed order.
7. Release the PHY core reset.
8. The DP side then runs the interface sequence in §3.10.4 and polls for
   common ready / lane PLL acknowledge there.

- **[POLICY]** *(vendor)* `pclk` enable is retried up to 10 times, 5–6 ms
  apart.
- **[POLICY]** *(vendor)* The vendor does no PLL lock or ready check after
  step 7. Readiness is observed through the DP interface (§3.10). A
  clean-room driver may also poll `PHY_PMA_CMN_CTRL2` (`0xE001`) bits 0/6
  (PLL0, used by USB) and 1/7 (PLL1, used by DP).
- **[HW]** Power-down: assert the PHY core reset, assert the APB reset,
  disable `pclk`. The PHY is shared, so do this only when **both** the USB
  and the DP users have released it.
- **[POLICY]** *(vendor)* Re-initialisation rules: the PHY is not
  re-initialised when the requested mode equals the current mode, when
  going from USB-only to USB + DP (same table) or when going from DP-only to
  USB + DP.

> OPEN QUESTION: Skipping re-initialisation on DP-only → USB + DP leaves the
> DP-only lane map and lane tables in place while USB starts using PMA
> lanes 0/1 (or 2/3). This looks wrong. A clean-room driver should
> re-initialise whenever the table would change.

## 3.7 Initialisation tables *(vendor)*

The vendor gives three complete tables: DP only, USB + DP normal and USB +
DP flipped. They are restated below as a common part plus per-lane-role
parts. All values are 16-bit. "L" means the per-lane index for every lane
that has the role. The vendor writes the groups in the order given here.
Whether the order matters is unknown, but `CMN_DIAG_GPANA_0` is always
written last.

Lane roles per mode:

| Mode | DP lanes | USB TX lane | USB RX lane |
|---|---|---|---|
| DP only | 0, 1, 2, 3 | – | – |
| USB + DP, normal | 2, 3 | 0 | 1 |
| USB + DP, flipped | 0, 1 | 3 | 2 |

### 3.7.1 Common part (all modes)

| Index | Value | Index | Value |
|---|---|---|---|
| `0x0022` | `0x0018` | `0x0074` | `0x0018` |
| `0x002A` | `0x0030` | `0x0104` | `0x001D` |
| `0x002C` | `0x00F0` | `0x010C` | `0x001D` |
| `0x0032` | `0x0030` | `0x0114` | `0x02D0` |
| `0x0034` | `0x00F0` | `0x0128` | `0x0137` |
| `0x0064` | `0x0078` | `0x0065` | `0x0078` |

Per lane, **all four lanes**: `0x8044`+L = `0x0018`, `0x8045`+L = `0x0078`,
`0x804C`+L = `0x0018`, `0x804D`+L = `0x0078`, `0x4123`+L = `0x0960`.

PLLs:

| Register | PLL0 | PLL1 |
|---|---|---|
| `PDIAG_PLLn_CLK_SEL_M0` (`0x01A1` / `0x01C1`) | `0x8600` | `0x0601` (USB + DP), `0x0701` (DP only) |
| `PLLn_DSM_DIAG_M0` | `0x0004` | `0x8004` |
| `PDIAG_PLLn_ITRIM_M0` | – | `0x003F` |
| `CP_PADJ` / `CP_IADJ` / `FILT_PADJ` | `0x0B17` / `0x0E01` / `0x0D05` | same |
| `PLLn_INTDIV_M0` | `0x01A0` | `0x01C2` |
| `PLLn_FRACDIVL_M0` | `0xAAAB` | `0x0000` |
| `PLLn_FRACDIVH_M0` | `0x0002` | `0x0002` |
| `PLLn_HIGH_THR_M0` | `0x0116` | `0x012C` |
| `PDIAG_PLLn_CTRL_M0` | `0x1002` | `0x1002` |
| `PLLn_SS_CTRL1..4_M0` | `0x0001`, `0x0416`, `0x007C`, `0x0003` | `0x0001`, `0x044F`, `0x007F`, `0x0003` |
| `VCOCAL_INIT_TMR` / `ITER_TMR` | `0x00F0` / `0x0004` | same |
| `VCOCAL_REFTIM_START` / `PLLCNT_START` | `0x02F8` / `0x02F6` | same |
| `VCOCAL_TCTRL` | `0x0003` | same |
| `LOCK_REFCNT_START` / `LOCK_PLLCNT_START` / `LOCK_PLLCNT_THR` | `0x00BF` / `0x00BF` / `0x0005` | same |

PHY level: `0xC010` = lane map (§3.5.2); `0xC011` = `0x0100`; `0xC018` =
`0x0A0A`; `0xC01A` = `0x1000`; `0xC01B` = `0x0010`; `0x0041` = `0x8200`;
`0x0047` = `0x8200`; `0xE005` = `0x000B`; `0xE006` = `0x2224`; finally
`0x01F0` = `0x0100`.

### 3.7.2 DP lanes

| Register (per lane) | Value |
|---|---|
| `XCVR_DIAG_HSCLK_SEL` `0x40E6`+L | `0x0001` |
| `XCVR_DIAG_HSCLK_DIV` `0x40E7`+L | `0x0001` (DP only), `0x0000` (USB + DP) |
| `XCVR_DIAG_PLLDRC_CTRL` `0x40E5`+L | `0x0019` |
| `TX_PSC_A0` / `A2` / `A3` (`0x4100`/`0x4102`/`0x4103`+L) | `0x00FB` / `0x04AA` / `0x04AA` |
| `RX_PSC_A0`, `A2`, `A3`, `RX_PSC_CAL` | `0x0000` |
| `XCVR_DIAG_BIDI_CTRL` `0x40EA`+L | `0x000F` |
| `RX_REE_GCSM1_CTRL`, `GCSM2_CTRL`, `PERGCSM_CTRL` | `0x0000` |

`TX_PSC_A1` is not written for DP lanes (A1 is unused by DP, §3.10.3).

### 3.7.3 USB lanes (USB + DP modes only)

| Register | USB TX lane | USB RX lane |
|---|---|---|
| `XCVR_DIAG_PLLDRC_CTRL` | `0x0041` | `0x0041` |
| `TX_PSC_A0` / `A1` / `A2` / `A3` | `0x02FF` / `0x06AF` / `0x06AE` / `0x06AE` | – |
| `TX_TXCC_CTRL` | `0x2A82` | – |
| `TX_TXCC_CPOST_MULT_01` | `0x0014` | – |
| `TX_TXCC_MGNFS_MULT_000` | `0x0007` | – |
| `RX_DIAG_ACYA` `0x81FF`+L | `0x0002` | – |
| `XCVR_DIAG_BIDI_CTRL` | `0x000F` | `0x00F0` |
| `RX_PSC_A0` / `A1` / `A2` / `A3` | – | `0x0D1D` / `0x0D1D` / `0x0D00` / `0x0500` |
| `RX_SIGDET_HL_FILT_TMR` | – | `0x0013` |
| `RX_REE_GCSM1_CTRL` | – | `0x0000` |
| `RX_REE_ATTEN_THR` | – | `0x0C02` |
| `RX_REE_SMGM_CTRL1` / `CTRL2` | – | `0x0330` / `0x0300` |
| `XCVR_DIAG_PSC_OVRD` | – | `0x0003` |
| `RX_DIAG_SIGDET_TUNE` | – | `0x1004` |
| `RX_DIAG_NQST_CTRL` | – | `0x00F9` |
| `RX_DIAG_DFE_AMP_TUNE_2` / `_3` | – | `0x0C01` / `0x0002` |
| `RX_DIAG_PI_CAP` / `PI_RATE` | – | `0x0000` / `0x0031` |
| `RX_CDRLF_CNFG` / `CNFG3` | – | `0x018C` / `0x0003` |

`HSCLK_SEL` and `HSCLK_DIV` are not written for USB lanes (reset defaults).

> OPEN QUESTION: `RX_DIAG_ACYA` = `0x0002` is written on the USB **TX** lane
> (PMA 0 normal, PMA 3 flipped), not the RX lane. This may be deliberate
> (a bidirectional-lane setting) or a slip. The two `XCVR_DIAG_BIDI_CTRL`
> values (`0x000F` = TX, `0x00F0` = RX) suggest a direction field.

> OPEN QUESTION: The meanings of `CMN_DIAG_GPANA_0` = `0x0100`, the
> isolation PLL values `0x000B`/`0x2224`, `PHY_LANE_OFF_CTRL` = `0x0100`
> and the PWRI override value `0x8200` are not explained. The vendor marks
> `GPANA_0` and the BIDI writes as late additions needed for USB speed.

## 3.8 PLLs and DP link rate

### 3.8.1 PLL roles

- **[HW]** *(vendor)* **PLL0 is the USB 3 PLL** and **PLL1 is the DP PLL**.
  The reference clock is 24 MHz. (The DP-side driver accepts only a 24 MHz
  reference.)
- The VCO frequencies are inferred, assuming VCO = 24 MHz × (`INTDIV` +
  `FRACDIVL`/65536). PLL0 = 416.667 × 24 MHz = **10.0 GHz** (USB 3 Gen 2).
  PLL1 at reset table = 450 × 24 MHz = **10.8 GHz**.
- Inferred: `HIGH_THR` ≈ ⅔ × `INTDIV` in every entry (`0x12C` = 300 for 450,
  `0x10E` = 270 for 405, `0xE2` ≈ 225 for 337.5).

### 3.8.2 Per-rate values (PLL1 and DP lanes)

The rate is chosen by the DPCD link-bandwidth code:

| DPCD code | Rate (Gb/s/lane) | `XCVR_DIAG_HSCLK_DIV` | `PDIAG_PLL1_CLK_SEL_M0` | `PLL1_INTDIV_M0` | `PLL1_FRACDIVL_M0` | `PLL1_HIGH_THR_M0` | Inferred VCO |
|---|---|---|---|---|---|---|---|
| `0x06` | 1.62 | `0x0002` | `0x0F01` | `0x0195` | `0x0000` | `0x010E` | 9.72 GHz |
| `0x0A` | 2.70 | `0x0001` | `0x0701` | `0x01C2` | `0x0000` | `0x012C` | 10.8 GHz |
| `0x14` | 5.40 | `0x0000` | `0x0301` | `0x01C2` | `0x0000` | `0x012C` | 10.8 GHz |
| `0x1E` | 8.10 | `0x0000` | `0x0200` | `0x0151` | `0x8000` | `0x00E2` | 8.1 GHz |

- **[HW]** Other codes are rejected. eDP intermediate rates are not
  supported on this PHY.
- **[HW]** *(vendor)* Rate-dependent extras, written before the table above:

| Register | 8.1 Gb/s | Other rates |
|---|---|---|
| `PDIAG_PLL1_ITRIM_M0` `0x01C3` | `0x007F` | `0x003F` |
| `PLL1_SS_CTRL2_M0` `0x00D9` | `0x0430` | `0x044F` |
| `PLL1_SS_CTRL3_M0` `0x00DA` | `0x0062`, then `0x0004` (see question) | `0x007F` |
| `PLL1_SS_CTRL4_M0` `0x00DB` | not written | `0x0003` |

> OPEN QUESTION: For 8.1 Gb/s the vendor writes `SS_CTRL3` twice (`0x0062`
> then `0x0004`) and never writes `SS_CTRL4`. The second write is almost
> certainly meant for `SS_CTRL4` (`0x0004`, by analogy with `0x0003` at
> other rates). As written, `SS_CTRL3` ends at `0x0004`. Test both.

> OPEN QUESTION: Whether `SS_CTRL1` = `0x0001` enables spread-spectrum
> clocking is unknown. If it does, the DP controller must advertise
> down-spread in the link configuration (sky1-dp-tx 02).

> OPEN QUESTION: In the initial tables `PDIAG_PLL1_CLK_SEL_M0` is `0x0601`
> (USB + DP) or `0x0701` (DP only). `0x0701` matches the 2.7 Gb/s entry.
> `0x0601` matches no rate. Rate setting always overwrites it, so a link
> must be configured (§3.8.3) before training. The vendor header claims
> "8.1 Gb/s by default", which the values do not support.

### 3.8.3 Rate change sequence

**[HW]** *(vendor)*, run while the DP link is idle:

1. On the DP interface (§3.10.5): clear data enable, enter A3, disable the
   lane PLL clock and wait for its acknowledge to clear.
2. Clear bit 1 (PLL1 run) of `PHY_PMA_PLL_CTRL` (`0xE003`). Wait 10 µs.
3. Write the rate-dependent extras (§3.8.2).
4. Write `XCVR_DIAG_HSCLK_DIV` on the DP lanes in use: 1 lane → DP lane 0
   only (PMA 2 normal, PMA 1 flipped); 2 lanes → PMA 2, 3 (normal) or
   PMA 0, 1 (flipped); 4 lanes → all four.
5. Write `PLL1_INTDIV_M0`, `PLL1_FRACDIVL_M0`, `PLL1_HIGH_THR_M0`,
   `PDIAG_PLL1_CLK_SEL_M0`.
6. Set bit 1 of `0xE003`. Wait 10 µs.
7. On the DP interface: enable the lane PLL clock, wait for the
   acknowledge, step A2 → A0, set data enable.
8. The DP controller then waits for its PHY status lane-ready bits
   (sky1-dp-tx 02).

> OPEN QUESTION: The vendor does not poll PLL1 ready/locked
> (`0xE001` bits 1 and 7) after step 6. It relies on the interface
> acknowledge in step 7. Polling bit 7 would be safer.

## 3.9 Voltage swing and pre-emphasis

**[HW]** *(vendor)* For each DP lane in use (lane selection as in §3.8.3
step 4):

1. Set bit 0 of `TX_DIAG_ACYA` (`0x41E7`+L).
2. Write `TX_TXCC_CTRL` = `0x08A4`, `DRV_DIAG_TX_DRV` = `0x0003`,
   `TX_TXCC_MGNFS_MULT_000` and `TX_TXCC_CPOST_MULT_00` from the table.
3. Clear bit 0 of `TX_DIAG_ACYA`.

| Swing | Pre-emph | `MGNFS_MULT_000` | `CPOST_MULT_00` |
|---|---|---|---|
| 0 | 0 | `0x0028` | `0x0000` |
| 0 | 1 | `0x001F` | `0x0014` |
| 0 | 2 | `0x0012` | `0x0020` |
| 0 | 3 | `0x0000` | `0x0027` |
| 1 | 0 | `0x001F` | `0x0000` |
| 1 | 1 | `0x0013` | `0x0012` |
| 1 | 2 | `0x0000` | `0x001F` |
| 2 | 0 | `0x0013` | `0x0000` |
| 2 | 1 | `0x0000` | `0x0013` |
| 3 | 0 | `0x0000` | `0x0000` |

- **[HW]** Swing + pre-emphasis > 3 is rejected (the table holds filler
  entries for those cells: (1,3) = (1,2); (2,2),(2,3) = (2,1); (3,x) =
  (3,0)).
- **[HW]** Inferred: `MGNFS_MULT_000` is the main-cursor margin reduction
  (0 = full swing) and `CPOST_MULT_00` is the post-cursor (de-emphasis)
  weight. Bit 0 of `TX_DIAG_ACYA` holds the coefficients while they are
  updated.
- **[POLICY]** *(vendor)* The vendor applies the **lane 0** request to all
  lanes. Per-lane values are possible, since the registers are per lane.

## 3.10 DP-side PHY interface window

Each DP controller has a second register window for its PHY interface. For
DP0, DP1, DP3 and DP4 it is 256 bytes (`dp_phy`, the third `reg` entry) at
the end of the controller's 64 KiB unit:

| Controller | Window |
|---|---|
| DP0 | `0x1406FF00` |
| DP1 | `0x140DFF00` |
| DP3 | `0x141BFF00` |
| DP4 | `0x1422FF00` |

Under ACPI it is memory resource index 2 of the DP device (sky1-dp-tx 01).

### 3.10.1 Identification

- **[HW]** Offset `0xFC`, bits [31:16] = interface type: **`0x54`** = USB-C
  combo PHY interface (this section), **`0x4D`** = eDP PHY interface (04).
  Software must pick the sequence by reading this field.

### 3.10.2 Registers (byte offsets, 32-bit)

| Offset | Name | Bits / use |
|---|---|---|
| `0x10` | `CDB_RSTN` | Defined, not used on this variant |
| `0x20` | `CDB_TIMEOUT_COUNT` | Defined, not used |
| `0x24` | `CDB_TIMEOUT` | Defined, not used |
| `0x40` | `PHY_CONFIG_RSTN` | bit 0: 0 = hold PHY interface in reset, 1 = release |
| `0x44` | `PHY_CMN_READY` | RO. bit 0 = PHY common ready |
| `0x48` | `PHY_LANE_EN` | bits [3:0]: per-lane enable |
| `0x4C` | `PHY_LANE_RSTN` | bits [3:0]: per-lane reset release (1 = out of reset) |
| `0x50` | `PHY_DATA_ENABLE` | bit 0: pass link data to the PHY |
| `0x54` | `AUXHPD_ENABLE` | bit 0 decap enable; bit 1 delayed decap enable; bit 2 bandgap enable; bit 3 AUX TX/RX enable |
| `0x58` | `TX_ELEC_IDLE` | bits [3:0]: 1 = lane held electrically idle |
| `0x5C` | `DP_RESET` | bit 0: 1 = DP part of the PHY interface in reset |
| `0x60` | `LANE_PWR_STATE_REQ` | Power-state request (§3.10.3). 0 = no request |
| `0x64` | `LANE_PWR_STATE_ACK` | RO. bits [4:0] = acknowledged state |
| `0x68` | `LANE_PLLCLK_ENABLE` | bit 0: enable lane PLL clocks |
| `0x6C` | `LANE_PLLCLK_EN_ACK` | RO. bit 0 = lane PLL clock enable acknowledged |
| `0x70` | `PHY_POWERDOWN` | Defined, not used |
| `0x78` | `PHY_TX_LINK_RATE` | Defined, not used |
| `0xA0` | `TYPEC_CONN_ORIENTATION` | Defined, not used by the vendor |
| `0xF4` | `TEST_RDWR_ADDRESS` | Test |
| `0xF8` | `MAX_LANE_COUNT` | Defined, not used |
| `0xFC` | `ID_REV` | [31:16] type (§3.10.1) |

> OPEN QUESTION: `TYPEC_CONN_ORIENTATION` (`0xA0`) suggests the interface
> can also be told the plug orientation. The vendor relies on the S5 bit
> instead. Its encoding is unknown.

### 3.10.3 Power states

| Value | State | Notes |
|---|---|---|
| `0x0` | No request / idle | Written after each handshake |
| `0x1` | A0 | TX active |
| `0x2` | A1 | Not used by DP |
| `0x4` | A2 | Power-down. Configured the same as A3 for DP |
| `0x8` | A3 | Deepest. Entered for rate change, PSR and link off |

**[HW]** *(vendor)* Handshake: write the state to `0x60`, poll `0x64`
bits [4:0] until they equal the state, then write `0` to `0x60`, then wait
200 ns. To come back up from A3, step A2 → A0.

- **[POLICY]** *(vendor)* Poll budget 200 × 1 µs. A timeout is logged and
  ignored.

### 3.10.4 Interface bring-up

**[HW]** *(vendor)* "Prepare" (needed before any AUX transaction):

1. `AUXHPD_ENABLE` = 0; `TX_ELEC_IDLE` = `0xF`.
2. `PHY_DATA_ENABLE` = 0; `PHY_LANE_RSTN` = 0; `PHY_LANE_EN` = 0;
   `PHY_CONFIG_RSTN` = 0; `DP_RESET` = 1. Wait 1 µs.
3. `AUXHPD_ENABLE` = `0x1`, wait 1 µs; = `0x3`, wait 1 µs; = `0x7`, wait
   5 µs; = `0xF`.
4. `DP_RESET` = 0.

"Init" (link power-up, after the PHY core is up per §3.6):

1. Make sure the combo PHY is powered and in a DP-capable mode (§3.6).
2. `PHY_LANE_RSTN` = `0xF`; `PHY_LANE_EN` = `0xF`; `TX_ELEC_IDLE` = 0;
   `PHY_CONFIG_RSTN` = 1.
3. Poll `PHY_CMN_READY` bit 0 = 1 (vendor budget 500 × 1 µs).
4. `LANE_PLLCLK_ENABLE` = 1. Poll `LANE_PLLCLK_EN_ACK` bit 0 = 1 (500 × 1 µs).
5. Power state A2, then A0 (§3.10.3).
6. `PHY_DATA_ENABLE` = 1.
7. Set the link rate (§3.8.3) and voltages (§3.9) during link training.

- **[POLICY]** *(vendor)* All four lanes are enabled at the interface even
  when fewer are trained. The vendor's per-lane-count enable is disabled.

### 3.10.5 Rate-change bracket and shutdown

- **[HW]** *(vendor)* Before reprogramming PLL1: `PHY_DATA_ENABLE` = 0; A3;
  `LANE_PLLCLK_ENABLE` = 0; poll ack bit 0 = 0 (100 × 1 µs). After:
  `LANE_PLLCLK_ENABLE` = 1; poll ack = 1 (100 × 1 µs); A2; A0;
  `PHY_DATA_ENABLE` = 1.
- **[HW]** *(vendor)* Full interface reset: `PHY_DATA_ENABLE` = 0,
  `PHY_LANE_RSTN` = 0, `PHY_LANE_EN` = 0, `LANE_PWR_STATE_REQ` = 0,
  `LANE_PLLCLK_ENABLE` = 0, `DP_RESET` = 1. Minimal reset: just the first
  three.
- **[POLICY]** *(vendor)* Link off: A3, then release the combo PHY (§3.6
  power-down if USB is not using it). PSR entry without keeping the main
  link: A3. PSR exit: A2 → A0.

## 3.11 Type-C / PD controller interaction

### 3.11.1 What the PHY needs

| Input | Source | Effect |
|---|---|---|
| Orientation: normal / reverse / none | PD controller | Selects tables and S5 bit (§3.5). "None" keeps the old value |
| Mode / pin assignment | PD controller | C or E → DP only; D → USB + DP; USB, safe or other → USB only |
| HPD | PD controller (alt-mode HPD) | Consumed by the DP controller, not the PHY |

- **[POLICY]** *(vendor)* Orientation and mode updates are only recorded.
  The PHY is reprogrammed at the next power-up by USB or DP (§3.6). A
  clean-room driver that keeps the PHY powered must re-run §3.6 when
  either input changes.

### 3.11.2 RTS5453 (Realtek USB-PD controller)

- **[HW]** I²C address `0x30` (port 0) / `0x31` (second port), at 100 kHz,
  on FCH I²C1 (Orion O6) or FCH I²C7 (Orange Pi 6 Plus). Compatible
  `realtek,rts5453h`, ACPI `_HID` `CIXH200D`. Level-low interrupt on an S5
  GPIO (pin 8 on O6, pin 7 on Orange Pi 6 Plus), shared by both ports.
- **[HW]** *(vendor)* Status: I²C register `0x5F`, 6 bytes. Facts the PHY
  side uses:

| Byte | Bit | Meaning |
|---|---|---|
| 1 | 1 | Orientation: 1 = reverse |
| 1 | 4 | USB 2 connected |
| 1 | 5 | USB 3 connected |
| 1 | 7 | Data role: 1 = device |
| 2 | 0 | DP alt mode connected |
| 2 | 4 | Debug accessory |
| 2 | 6 | HPD IRQ |
| 2 | 7 | HPD level |

  Register `0x50` is the control/acknowledge register (bit 2 interrupt ack,
  bit 5 HPD-IRQ ack).
- **[POLICY]** *(vendor)* Mapping: USB3 + DP → pin assignment D (USB + DP);
  DP without USB3 → E (DP only); USB3 only → USB; neither → safe (and
  orientation "none"). The PD controller advertises pin assignments C, D
  and E.

### 3.11.3 Lane count and readiness seen by the DP controller

- **[HW]** On HPD, the DP controller must wait until the PHY is in a
  DP-capable mode before reading the sink. *(vendor)* It polls up to 100 ×
  100 ms, then proceeds anyway.
- **[HW]** After rate setting, the controller waits for its own PHY-status
  lane-ready bits (sky1-dp-tx 02).

## 3.12 Boot-firmware handover

- **[POLICY]** *(vendor, devicetree boot only)* UEFI leaves one byte per
  PHY at physical address **`0x83E05000` + n** (4 bytes):

| Byte value | Firmware state | Vendor action at probe |
|---|---|---|
| 0 | Not used by firmware | Assert both resets |
| 1 | USB host + DP (2 + 2) | Leave running. Skip the next **two** PHY initialisations (one each from USB and DP) and just enable `pclk` |
| 2 | USB device | Assert both resets |
| 3 | USB 2.0 + 4-lane DP | Leave running. No skip |
| other | – | Assert both resets |

- **[POLICY]** *(vendor)* Under ACPI the byte is ignored. Both resets are
  asserted at probe, and the PHY is fully initialised by its first user. An
  earlier community revision kept the firmware state instead. That broke
  once the USB controller began resetting itself at probe.

> OPEN QUESTION: Byte value 3 says the firmware left 4-lane DP active, but
> the PHY's own mode starts at `default_conf`. On a board whose
> `default_conf` is 3 this leaves the recorded mode and the hardware out of
> step.

## 3.13 Board configurations

| Board | PHY 0 (DP0) | PHY 1 (DP1) | PHY 2 (DP3) | PHY 3 (DP4) |
|---|---|---|---|---|
| Radxa Orion O6 | USB-C via RTS5453 @`0x30`, mode 3 default, switches | DP only (mode 2), feeds the board's HDMI path | USB-C via RTS5453 @`0x31`, mode 3 default | DP only (default from §3.2.1 rule) |
| Orion O6N | – | – | – | DP only (mode 2) |
| Orange Pi 6 Plus | USB-C, mode 3 default, switches | USB-C, mode 3 default, switches | DP only (mode 2) | DP only (mode 2), feeds the PS185 HDMI bridge (04 §4.9) |

- **[POLICY]** Boards set `cix,dp-max-rate` = 540000 (HBR2) on most DP
  outputs. The PHY itself supports up to HBR3.
