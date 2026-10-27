# 04: eDP PHY, Panel Controls and the PS185 HDMI Bridge

This section covers the PHY of the **DP2** controller (the Sky1's
embedded-DisplayPort output), the panel power and backlight controls tied to
it, and (§4.9) the Parade PS185 DP-to-HDMI bridge that the Orange Pi 6 Plus
hangs off DP4.

Sources: the Sky1-Linux devicetree (BSD-3-Clause), the mainline pin-function
header and bridge binding, and the CIX vendor DP driver's eDP PHY and panel
code. Facts that come only from vendor code are marked *(vendor)*.

The eDP PHY is the same Cadence Torrent-family SerDes as the combo PHY
(03), but with **no USB role and no Type-C mux**. It sits behind the DP
controller's own PHY interface window, and its PMA registers are reached
**indirectly** through a configuration bus (CDB) in that window. The PMA
register names, the per-lane stride (`0x200`) and the 16-bit register width
are as in 03 §3.3–§3.4.

## 4.1 Resources

| Item | Value |
|---|---|
| PHY interface window | `0x1414C000`, 16 KiB (DP2's third `reg` entry, `dp_phy`; ACPI memory resource index 2) |
| Interface type | Offset `0xFC` bits [31:16] = **`0x4D`** (03 §3.10.1) |
| PHY reset | S5 line `DP_PHY`, index 29 (word `0x400`, bit 26, active-low), reset name `"phy_reset"` on the DP2 node |
| External PHY handle | None. DP2 has no `phys` property, so its lane count is the controller's maximum (4) |
| Reference clock | 24 MHz *(vendor: only 24 MHz supported)* |

- **[HW]** *(vendor)* Before first use, pulse `phy_reset`: assert, wait
  10–20 µs, release, wait 100–200 µs. This puts the PHY interface back in
  its power-off state. The vendor does this together with the DP controller
  reset at every host initialisation.

> OPEN QUESTION: The interface has a `REFCLK_IS_100MHz` register (`0x30`)
> that the vendor never writes. The PLL values only make sense for a
> 24 MHz reference (450 × 24 MHz = 10.8 GHz), so the reset value
> presumably means "not 100 MHz".

## 4.2 PHY interface registers (byte offsets, 32-bit)

| Offset | Name | Bits / use |
|---|---|---|
| `0x00` | `CDB_ADDR` | PMA register index for the next CDB transaction |
| `0x04` | `CDB_WDATA` | 16-bit write data |
| `0x08` | `CDB_SEL_WR` | bit 1 = select (1 for any transaction), bit 0 = 1 write / 0 read |
| `0x0C` | `CDB_READY` | RO. bit 0 = CDB idle / last transaction complete |
| `0x10` | `CDB_RDATA` | RO. Read data |
| `0x14` | `CDB_RSTN` | bit 0: 1 = CDB out of reset |
| `0x18` | `CDB_ENABLE` | Write 1 to launch the configured transaction (clears `CDB_READY`) |
| `0x30` | `REFCLK_IS_100MHz` | Reference-clock select (not written) |
| `0x40` | `PHY_CONFIG_RSTN` | bit 0: 0 = PHY held in global reset, 1 = released |
| `0x44` | `PHY_CMN_READY` | RO. bit 0 = common ready |
| `0x48` | `PHY_LANE_EN` | bits [3:0] lane enables |
| `0x4C` | `PHY_LANE_RSTN` | bits [3:0] lane reset release |
| `0x50` | `PHY_DATA_ENABLE` | bit 0 |
| `0x54` | `AUXHPD_ENABLE` | bit 0 decap, bit 1 delayed decap, bit 2 bandgap, bit 3 AUX TX/RX |
| `0x58` | `TX_ELEC_IDLE` | bits [3:0] |
| `0x60` | `LANE_PWR_STATE_REQ` | As 03 §3.10.3 |
| `0x64` | `LANE_PWR_STATE_ACK` | RO, bits [4:0] |
| `0x68` | `LANE_PLLCLK_ENABLE` | bit 0 |
| `0x6C` | `LANE_PLLCLK_EN_ACK` | RO, bit 0 |
| `0x70` | `PHY_POWERDOWN` | Not used |
| `0x78` | `PHY_TX_LINK_RATE` | Not used |
| `0xA0`–`0xAC` | `PMA_EMPHASIS_LN0..3` | Not used |
| `0xB0`–`0xBC` | `PMA_VMARGIN_LN0..3` | Not used |
| `0xC0` | `PMA_LOW_POWER_SWING` | Not used |
| `0xF4` | `TEST_RDWR_ADDRESS` / `MAX_DP_LANE_COUNT` | See question |
| `0xFC` | `ID_REV` | [31:16] = `0x4D` |

- **[HW]** Unlike the combo-PHY interface, this variant has **no
  `DP_RESET` register** at `0x5C`.

> OPEN QUESTION: The vendor header puts both `TEST_RDWR_ADDRESS` and
> `MAX_DP_LANE_COUNT` at `0xF4`. The combo variant has the lane count at
> `0xF8`. Neither is used.

> OPEN QUESTION: The per-lane `PMA_EMPHASIS`, `PMA_VMARGIN` and
> `PMA_LOW_POWER_SWING` registers suggest a direct drive-level path that
> bypasses the PMA coefficient registers (§4.6). Their encoding is unknown.

## 4.3 CDB (indirect PMA access)

**[HW]** *(vendor)* Write PMA register *index* with *value*:

1. Poll `CDB_READY` bit 0 until 1 (vendor: up to 100 × 5 µs).
2. `CDB_ADDR` = index; `CDB_WDATA` = value.
3. `CDB_SEL_WR` = `3`.
4. `CDB_ENABLE` = `1`.

Read:

1. Poll `CDB_READY` bit 0 = 1.
2. `CDB_ADDR` = index; `CDB_SEL_WR` = `2`; `CDB_ENABLE` = `1`.
3. Poll `CDB_READY` bit 0 = 1, then read `CDB_RDATA` (low 16 bits).

- **[HW]** `CDB_RSTN` must be 1 before any CDB transaction.
- **[POLICY]** *(vendor)* On a ready timeout the write is dropped and a
  read returns 0, silently.

## 4.4 Sequences

### 4.4.1 Prepare (AUX usable afterwards)

**[HW]** *(vendor)*

1. `CDB_RSTN` = 1.
2. `AUXHPD_ENABLE` = 0; `TX_ELEC_IDLE` = `0xF`.
3. `PHY_DATA_ENABLE` = 0; `PHY_LANE_RSTN` = 0; `PHY_LANE_EN` = 0;
   `PHY_CONFIG_RSTN` = 0. Wait 1 µs.
4. `AUXHPD_ENABLE` = `0x1`, wait 1 µs; `0x3`, wait 1 µs; `0x7`, wait 5 µs;
   `0xF`.

### 4.4.2 Initialise (main link up)

**[HW]** *(vendor)* With `PHY_CONFIG_RSTN` still 0:

1. Write the PMA table (§4.5) through the CDB.
2. Write `PHY_PMA_PLL_CTRL` (index `0xE003`) = `0x0001`: PLL0 runs, PLL1
   stays in reset.
3. `PHY_LANE_RSTN` = `0xF`; `PHY_LANE_EN` = `0xF`; `TX_ELEC_IDLE` = 0.
4. `PHY_CONFIG_RSTN` = 1. Poll `PHY_CMN_READY` bit 0 = 1 (500 × 1 µs).
5. `LANE_PLLCLK_ENABLE` = 1. Poll `LANE_PLLCLK_EN_ACK` = 1 (500 × 1 µs).
6. Power state A2, then A0 (03 §3.10.3; vendor budget here 50 × 1 µs, no
   200 ns tail).
7. `PHY_DATA_ENABLE` = 1.

### 4.4.3 Link-rate change

**[HW]** *(vendor)*

1. `PHY_DATA_ENABLE` = 0; enter A3.
2. `LANE_PLLCLK_ENABLE` = 0; poll ack bit 0 = 0 (100 × 1 µs).
3. CDB-write `0xE003` = 0 (PLL0 into reset). Poll CDB-read of
   `PHY_PMA_CMN_CTRL2` (`0xE001`) until bit 2 (PLL0 disabled) is set
   (100 × 1 µs).
4. Write `XCVR_DIAG_HSCLK_DIV` on **all four** lanes and
   `PLL0_INTDIV_M0`, `PLL0_FRACDIVL_M0`, `PLL0_HIGH_THR_M0`,
   `PDIAG_PLL0_CLK_SEL_M0` from §4.5.2.
5. CDB-write `0xE003` = 1. Poll `0xE001` bit 0 (PLL0 ready) (100 × 1 µs).
6. `LANE_PLLCLK_ENABLE` = 1; poll ack = 1; A2; A0; `PHY_DATA_ENABLE` = 1.

- **[HW]** Unlike the combo PHY, the eDP PHY carries DP on **PLL0**.

### 4.4.4 Reset and off

- **[HW]** *(vendor)* Full reset: `PHY_DATA_ENABLE`, `PHY_LANE_RSTN`,
  `PHY_LANE_EN`, `PHY_CONFIG_RSTN` all 0. Minimal reset:
  `LANE_PWR_STATE_REQ` = 0, `LANE_PLLCLK_ENABLE` = 0, `PHY_LANE_RSTN` = 0,
  `PHY_LANE_EN` = 0.
- **[POLICY]** *(vendor)* Link off is A3 only. PSR uses A3 / A2 → A0 as on
  the combo PHY.

## 4.5 PMA configuration *(vendor)*

### 4.5.1 Initial table

The vendor labels it "5.4 Gb/s by default". Relative to the combo PHY's
DP-only table (03 §3.7):

- **Identical:** the common calibration timers (03 §3.7.1 first table), the
  per-lane `RX_SDCAL*` / `TX_RCVDET_ST_TMR` timers on all four lanes, the
  charge-pump/filter values, `PDIAG_PLLn_CTRL_M0` = `0x1002`, the VCO
  calibration and lock-detector values for both PLLs, and the DP-lane
  settings (03 §3.7.2 rows `TX_PSC_*`, `RX_PSC_*`, `BIDI_CTRL`, `RX_REE_*`)
  on **all four lanes**.
- **Different or eDP-only:**

| Index / register | Value |
|---|---|
| `0xC00E` `PHY_PLL_CFG` | `0x0000` |
| `XCVR_DIAG_HSCLK_DIV` (all lanes) | `0x0000` |
| `XCVR_DIAG_PLLDRC_CTRL` (all lanes) | `0x0011` |
| `PLL0_DSM_DIAG_M0` / `PLL1_DSM_DIAG_M0` | `0x8004` / `0x0004` (swapped versus 03) |
| `PLL0_INTDIV_M0` | `0x01C2` (10.8 GHz) |
| `PLL0_HIGH_THR_M0` | `0x012C` |
| `PLL1_INTDIV_M0` / `PLL1_FRACDIVL_M0` / `PLL1_HIGH_THR_M0` | `0x014D` / `0x5555` / `0x00E0` |
| `PLLn_FRACDIVH_M0` | `0x0002` both |
| `PLL0_SS_CTRL1..4_M0` | `0x0001`, `0x044F`, `0x007F`, `0x0003` |
| `PLL1_SS_CTRL1..4_M0` | `0x0001`, `0x03C7`, `0x007F`, `0x0003` |

- **Not written:** `XCVR_DIAG_HSCLK_SEL`, `PLL0_FRACDIVL_M0`, any
  `CLK_SEL`, `ITRIM`, the lane map, the PIPE/USB registers, the PWRI
  overrides, the isolation PLL registers and `CMN_DIAG_GPANA_0`.

> OPEN QUESTION: `PDIAG_PLL0_CLK_SEL_M0` is not set until the first rate
> change, so the link must be rate-configured (§4.4.3) before training.
> PLL1 is programmed (333.33 × 24 MHz = 8.0 GHz) but held in reset. Its
> purpose is unknown.

### 4.5.2 Rates (PLL0)

| Code | Rate (Gb/s) | `HSCLK_DIV` | `PDIAG_PLL0_CLK_SEL_M0` | `PLL0_INTDIV_M0` | `PLL0_FRACDIVL_M0` | `PLL0_HIGH_THR_M0` | Inferred VCO / ratio |
|---|---|---|---|---|---|---|---|
| `0x06` | 1.62 | `0x0002` | `0x0F01` | `0x0195` | `0x0000` | `0x010E` | 9.72 GHz / 6 |
| `0x09` | 2.43 | `0x0001` | `0x0701` | `0x0195` | `0x0000` | `0x010E` | 9.72 GHz / 4 |
| `0x0A` | 2.70 | `0x0001` | `0x0701` | `0x01C2` | `0x0000` | `0x012C` | 10.8 GHz / 4 |
| `0x0C` | 3.24 | `0x0002` | `0x0B00` | `0x0195` | `0x0000` | `0x010E` | 9.72 GHz / 3 |
| `0x14` | 5.40 | `0x0000` | `0x0301` | `0x01C2` | `0x0000` | `0x012C` | 10.8 GHz / 2 |
| `0x1E` | 8.10 | `0x0000` | `0x0200` | `0x0151` | `0x8000` | `0x00E2` | 8.1 GHz / 1 |

- **[HW]** The code is the link rate in units of 0.27 Gb/s, the DPCD
  link-bandwidth convention. 2.43 and 3.24 Gb/s are eDP intermediate rates.
  The 1.62/2.7/5.4/8.1 rows match the combo PHY's PLL1 rows (03 §3.8.2).
- **[POLICY]** The Orion O6 and Orange Pi 6 Plus cap DP2 at 5.4 Gb/s
  (`cix,dp-max-rate` = 540000).

> OPEN QUESTION: The rate-dependent ITRIM and spread-spectrum adjustments
> that the combo PHY applies at 8.1 Gb/s (03 §3.8.2) are absent here.
> Whether eDP at HBR3 needs them is untested on any known board.

## 4.6 Voltage swing and pre-emphasis

**[HW]** *(vendor)* Same per-lane procedure as 03 §3.9 (set
`TX_DIAG_ACYA` bit 0, write `TX_TXCC_CTRL` = `0x08A4`, `DRV_DIAG_TX_DRV` =
`0x0003`, `MGNFS_MULT_000`, `CPOST_MULT_00`, clear `TX_DIAG_ACYA` bit 0),
except that `TX_DIAG_ACYA` is read-modify-written through the CDB.

Index = swing × 4 + pre-emphasis. Values (MGNFS, CPOST):

| (V, P) | DP table (used) | eDP low-swing (unused) | eDP high-swing (unused) |
|---|---|---|---|
| (0,0) | `0x2A`, `0x00` | `0x2F`, `0x00` | `0x27`, `0x00` |
| (0,1) | `0x1F`, `0x14` | `0x2A`, `0x0D` | `0x23`, `0x09` |
| (0,2) | `0x13`, `0x20` | `0x26`, `0x15` | `0x1E`, `0x10` |
| (0,3) | `0x0A`, `0x2A` | `0x22`, `0x1B` | `0x1A`, `0x15` |
| (1,0) | `0x1F`, `0x00` | `0x2B`, `0x00` | `0x23`, `0x00` |
| (1,1) | `0x13`, `0x12` | `0x27`, `0x0B` | `0x1E`, `0x08` |
| (1,2) | `0x00`, `0x1F` | `0x23`, `0x12` | `0x1A`, `0x0E` |
| (2,0) | `0x13`, `0x00` | `0x27`, `0x00` | `0x1E`, `0x00` |
| (2,1) | `0x00`, `0x13` | `0x23`, `0x09` | `0x1A`, `0x07` |
| (3,0) | `0x00`, `0x00` | `0x23`, `0x00` | `0x1A`, `0x00` |

(Cells with V + P > 3 hold filler and must not be requested.)

- **[POLICY]** *(vendor)* The vendor always uses the **DP** table, even on
  the eDP output. Its eDP-table selector is never set. The low-swing table
  is the natural choice for an eDP panel that advertises reduced swing. The
  clean room may select it.
- **[POLICY]** *(vendor)* Only the lane-0 request is applied. Lane
  selection: a 1-lane link programs **PMA lane 2 only**; 2- and 4-lane
  links program all four lanes.

> OPEN QUESTION: The DP table here differs from the combo PHY's table at
> (0,0), (0,2) and (0,3) (03 §3.9). It is unclear which is tuned for Sky1.
> The "1 lane → PMA lane 2" choice suggests that on this PHY too, DP
> lane 0 may sit on PMA lane 2. Verify the lane order before using
> 1- or 2-lane eDP.

## 4.7 eDP panel device

### 4.7.1 Identification

| Firmware | Identifier |
|---|---|
| Devicetree | compatible `"cix-edp-panel"` (no vendor prefix) |
| ACPI | `_HID` `CIXH5040` |
| Link from DP2 | DP2 property `edp-panel` = phandle/reference to the panel node |

- **[HW]** A DP controller whose node has `edp-panel` is an **eDP**
  output: connector type eDP, panel power sequencing (§4.7.3), modes from
  the panel's EDID over AUX.

### 4.7.2 Properties

| Property | Type | Meaning |
|---|---|---|
| `enable-gpios` | GPIO, optional | Panel logic power enable (eDP "VDD enable"). Driven active at probe |
| `reset-gpios` | GPIO, optional | Panel reset. Active = in reset |
| `hpd-gpios` | GPIO, optional | Panel HPD input, polled during power-up |
| `no-hpd` | bool | No HPD line. Add the HPD-absent delay to the prepare delay |
| `power-supply` | regulator | Panel supply. A missing supply is a no-op |
| `power-invert` | bool | Supply logic inverted: "panel on" = supply **disabled** |
| `backlight` | phandle | The backlight device (§4.8) |
| `prepare-delay-ms` | u32 | After power-on, before AUX/link use |
| `enable-delay-ms` | u32 | After link video starts, before backlight on |
| `disable-delay-ms` | u32 | After backlight off |
| `unprepare-delay-ms` | u32 | After power-off (minimum off time) |
| `reset-delay-ms`, `init-delay-ms` | u32 | Reset pulse width / wait after reset release |
| `display-timings` or `panel-timing` child | node, optional | Fixed mode. When present, `bpc`, `bus-format`, `width-mm`, `height-mm` are also read |
| orientation / rotation | standard panel property | Panel mounting orientation |

- **[HW]** *(vendor)* Under ACPI the same names are read as `_DSD`
  properties. Orientation comes from an integer `rotation` (0, 90, 180 or
  270; absent = unknown). `backlight` is a `_DSD` reference to the
  `CIXH5041` backlight device.

> OPEN QUESTION: The vendor DP driver resolves DP2's `edp-panel` reference
> only when it points to a devicetree node. How an ACPI DP2 device links to
> `CIXH5040` is untested in the sources.

- **[POLICY]** *(vendor)* `width-mm`/`height-mm` are only honoured when a
  timing node exists. The Orion O6 sets them without a timing node, so they
  are ignored and the size comes from EDID.

### 4.7.3 Power sequencing

**[HW]** The order follows the eDP panel power sequence. The delays are
board data. *(vendor)* flow:

1. **Power-up** (when the DP2 host is initialised, before link training):
   turn the supply on, drive `enable` active, wait `prepare-delay-ms`
   (+ HPD-absent delay if `no-hpd`). If `hpd-gpios` exists, poll it every
   1 ms for up to 2 s until asserted. If `reset-gpios` exists, assert reset,
   wait `reset-delay-ms`, release, wait `init-delay-ms`. The vendor then
   waits another 50 ms.
2. **Link up:** PHY prepare/init (§4.4), sink power-up over AUX, link
   training.
3. **Enable:** wait `enable-delay-ms`, then switch the backlight on
   (§4.8).
4. **Disable** (stream off): switch the backlight off, wait
   `disable-delay-ms`.
5. **Power-down** (host de-initialised): assert reset (if present), drive
   `enable` inactive, turn the supply off, wait `unprepare-delay-ms`.
   Power-up must not start again before that wait ends.

- **[POLICY]** Board values (Orion O6 and Orange Pi 6 Plus): prepare
  120 ms, enable 120 ms, disable 120 ms, unprepare 500 ms. 500 ms matches
  the eDP minimum power-off time.
- **[POLICY]** *(vendor)* For an eDP connector the vendor always uses the
  first (preferred) mode as the output mode, whatever mode was requested.

## 4.8 Panel control pins and PWM backlight

### 4.8.1 Dedicated eDP pads

| Pad (S0 pad controller index) | Function 0 | Use |
|---|---|---|
| 18 `DP2_DIGON` | `DP2_DIGON` | Panel logic power enable ("digital on") |
| 19 `DP2_BLON` | `DP2_BLON` | Backlight enable |
| 20 `DP2_VARY_BL` | `DP2_VARY_BL` | Backlight brightness PWM |

Pin-function encoding (public mainline header): (pad index << 8) |
function, so `0x1200`, `0x1300` and `0x1400`. Function 0 is the only
function listed for these pads.

- **[HW]** Board pin configuration: `DP2_DIGON` and `DP2_BLON` pull-down,
  drive level 4. `DP2_VARY_BL` drive level 7, no pull.
- **[HW]** The panel enable is FCH GPIO bank 3 line **16** (active high). The
  backlight enable is FCH GPIO bank 3 line **15** (active high). FCH GPIO 3
  is a Cadence GPIO block at `0x04150000`.
- **[HW]** The brightness PWM is **FCH PWM0** (`0x04110000`, compatible
  `cix,sky1-pwm`, clocks `CLK_TREE_FCH_TIMER_APB` + `CLK_TREE_FCH_TIMER_FUN`,
  FCH reset `SW_TIMER_RST_FUNC_N`), channel 0, routed to the
  `DP2_VARY_BL` pad.

> OPEN QUESTION: The pads are put in their `DP2_DIGON`/`DP2_BLON` function,
> yet software drives them as FCH GPIO 3 lines 16 and 15. The likely
> reading is that the pads' function 0 *is* the connection to those GPIO
> lines (and `DP2_VARY_BL` the connection to PWM0). It is also possible
> that the DP2 controller can drive them itself. No vendor code drives
> panel pins from the DP controller.

### 4.8.2 Backlight device

| Item | Value |
|---|---|
| Devicetree | `pwm-backlight` (standard binding) |
| ACPI | `_HID` `CIXH5041` |
| PWM | PWM0 channel 0, period 100 000 ns (**10 kHz**), normal polarity |
| Enable | `enable-gpios` = FCH GPIO 3 line 15, active high |
| Levels | 256 linear steps, 0…255; default 200 |

- **[POLICY]** Backlight on = PWM running at the set duty with the enable
  line asserted. Off = PWM stopped and enable deasserted. Brightness level k
  gives duty k/255. The order of enable versus PWM start is not set by the
  board data.
- The FCH PWM controller's register interface is not covered here.

## 4.9 Parade PS185 DP-to-HDMI bridge (Orange Pi 6 Plus)

### 4.9.1 What it is

- **[HW]** The Parade **PS185HDM** converts DisplayPort to HDMI. The public
  mainline binding lists it as a **transparent, non-programmable bridge**
  (compatible `parade,ps185hdm`): one input port (`port@0`, DP), one output
  port (`port@1`, HDMI) and an optional enable GPIO. The OS sends it no
  configuration.
- **[HW]** On the Orange Pi 6 Plus it is fed by **DP4**, whose combo PHY 3
  is fixed in DP-only mode (03 §3.13, up to 4 lanes). Its output drives an
  HDMI type-A connector.

### 4.9.2 Wiring and control

| Signal | Connection |
|---|---|
| `PS185_PDB` (power-down control) | FCH GPIO bank 0 line **31**, on pad `GPIO074` (the `I2S2_TWS` pad in GPIO function), pull-up, drive level 4 |
| DP main link + AUX + HPD | DP4 via combo PHY 3 |
| HDMI out | Type-A connector |

- **[HW]** The bridge must be powered **before** DP4 samples HPD or reads
  the sink. The community devicetree describes `PS185_PDB` as a 3.3 V fixed
  supply that is on at boot and always on. With no `enable-active-high`,
  that binding drives the line **physically low** while "on". The vendor
  devicetree described the same line as an active-low `pdb-gpios`. Both
  descriptions therefore drive the line low in normal operation.

> OPEN QUESTION: A pin named "PDB" is normally active-low power-down, so
> low would mean "powered down", which contradicts working boards. Either
> the board inverts the signal or the name is misleading. Measure the line
> on hardware, or keep it low as both firmware descriptions do.

### 4.9.3 What the DP side must do

- **[HW]** Treat the bridge as an ordinary DP sink. HPD, DPCD and EDID all
  come over DP4's HPD and AUX. (Inferred from the bridge being
  non-programmable and from the DP driver reading the sink only over AUX.)
- **[HW]** Devicetree graph: DP4's `ports` gains **`port@2`** (output) whose
  endpoint links to the bridge's `port@0`. Ports 0 and 1 of a DP controller
  remain the two DPU pipeline inputs (sky1-dp-tx 01). A DP controller with
  no `port@2` has no downstream bridge.
- **[POLICY]** Report the output to userspace as an **HDMI-A** connector,
  taken from the last element of the bridge chain (the `hdmi-connector`
  node, `type = "a"`), not as DisplayPort.
- **[POLICY]** The board caps DP4 at 5.4 Gb/s.

> OPEN QUESTION: The bridge's HDMI-side limits (maximum TMDS rate,
> HDMI 2.0 scrambling, YCbCr 4:2:0) are not described anywhere in the
> sources. A DP-to-HDMI converter normally reports them in the DPCD
> downstream-port capability fields. The DP side should read those fields
> and filter modes, but the vendor driver does not.
