# 00: Overview, Instances and Resources

## 0.1 What the block is

Each Sky1 display unit n (n = 0 … 4) is a chain:

    DPU n (Linlon-D6, two pipelines) → RCSU video adapter → DPTX core n → PHY → connector

- **[HW]** The DPTX core is a DisplayPort 1.4 source: up to **4 lanes**,
  link rates RBR/HBR/HBR2/HBR3 (1.62/2.7/5.4/8.1 Gbit/s per lane), 8b/10b
  coding, enhanced framing, SSC (down-spread), TPS1–TPS4, an AUX channel
  with native and I2C-over-AUX transactions, HPD detection, MST with a
  64-slot payload table, secondary-data (SDP/InfoFrame) insertion, audio
  packetisation from an I2S input, eDP PSR, a host timer, and an HDCP
  cipher block.
- **[HW]** The register file has **four "virtual source" blocks** (sources
  0–3, 01 §1.5). On Sky1 only sources 0 and 1 are wired, because the DPU has
  two pipelines (§0.5).
- **[HW]** The core reports its revision in `CORE_REVISION` (01 §1.3). The
  vendor driver only prints it and never checks it.

## 0.2 Instances

| Unit n | Node base | `dp` (core, 16 KiB) | `dsc` (16 KiB) | `dp_phy` | `dp_rcsu` | IRQ (GIC SPI, level-high) | ACPI GSIV |
|---|---|---|---|---|---|---|---|
| 0 | `0x14060000` | `0x14064000` | `0x14068000` | `0x1406FF00`, 256 B | `0x14050000`, `0x304` B | 332 | 364 |
| 1 | `0x140D0000` | `0x140D4000` | `0x140D8000` | `0x140DFF00`, 256 B | `0x140C0000`, `0x304` B | 333 | 365 |
| 2 | `0x14140000` | `0x14144000` | `0x14148000` | `0x1414C000`, 16 KiB | `0x14130000`, `0x320` B | 334 | 366 |
| 3 | `0x141B0000` | `0x141B4000` | `0x141B8000` | `0x141BFF00`, 256 B | `0x141A0000`, `0x304` B | 335 | 367 |
| 4 | `0x14220000` | `0x14224000` | `0x14228000` | `0x1422FF00`, 256 B | `0x14210000`, `0x304` B | 336 | 368 |

- **[HW]** `dp` = node + `0x4000`, `dsc` = node + `0x8000`, `dp_rcsu` =
  node − `0x10000`. All registers are 32-bit, accessed with 32-bit reads
  and writes.
- **[HW]** The `dp_phy` window identifies the PHY type: bits `[31:16]` of
  the word at `dp_phy + 0xFC` read `0x54` for the USB-C/DP combo PHY
  (units 0, 1, 3, 4) and `0x4D` for the eDP PHY (unit 2). PHY programming
  is in sky1-dp-tx 03 and 04.
- **[HW]** `dsc` is never mapped by the vendor driver *(vendor)*. Its
  contents are unknown.
- **[HW]** One interrupt line per instance; it carries every core interrupt
  source (01 §1.4).

## 0.3 Clocks

| Name | SCMI clock (protocol `0x14`) | Use |
|---|---|---|
| `apb_clk` | `CLK_TREE_DPCn_APBCLK` = 59 + n | Register interface. Also the source of the AUX 1 MHz time base (02 §2.3) |
| `vid_clk0` | `CLK_TREE_DPCn_VIDCLK0` = 49 + 2n | Video input of source 0 (DPU pipeline 0) |
| `vid_clk1` | `CLK_TREE_DPCn_VIDCLK1` = 50 + 2n | Video input of source 1 (DPU pipeline 1) |

- **[HW]** `apb_clk` must run before any register access.
- **[POLICY]** *(vendor)* The driver never sets a rate on any of these
  clocks. It enables `vid_clk k` just before stream k is enabled and
  disables it after the stream is disabled. The pixel rate itself is set by
  the DPU's pixel clock (`CLK_TREE_DPn_PIXEL0/1`, DPU spec 08 §8.2).
- **[POLICY]** *(vendor)* The driver also asks for optional `vid_clk2` and
  `vid_clk3` (sources 2 and 3); Sky1 does not provide them.

> OPEN QUESTION: The `apb_clk` rate is not stated anywhere. The driver
> reads it back to program the AUX divider and falls back to 25 (MHz) if the
> rate reads as 0. Whether `vid_clk0/1` are plain gates or must be set to
> the pixel rate is not shown; the vendor driver treats them as gates.

## 0.4 Resets and power

| Name | S5 reset index | Register word / bit | Instances |
|---|---|---|---|
| `dp_reset` | `SKY1_DP_RESETn_N` = 24 + n | `0x400` bit 21 + n | all |
| `dp_rcsu_reset` | `SKY1_DPn_RCSU_RESET_N` = 105 + n | `0x800` bit 14 + n | all |
| `phy_reset` | `SKY1_DP_PHY_RST_N` = 29 | `0x400` bit 26 | unit 2 only (eDP PHY) |

- **[HW]** All resets are active-low lines of the S5 reset controller
  (Sky1 SoC spec 02 §2.1).
- **[POLICY]** *(vendor)* The driver pulses `dp_reset` (assert, 10–20 µs,
  deassert) and, where present, `phy_reset` the same way, every time it
  initialises the host side (02 §2.1). It never touches `dp_rcsu_reset`
  (it does not even request it).
- **[HW]** The devicetree gives the DP nodes **no power domain**.

> OPEN QUESTION: Whether the DP controller sits in the DPU's power domain
> (`SKY1_PD_DPUn`) is unknown. The vendor driver links the DPU device as a
> runtime-PM consumer of the DP device, so in practice the DP block is only
> touched while the DPU is powered. A clean-room driver should power the
> DPU domain first.

## 0.5 Video inputs from the DPU

- **[HW]** Input port k of DP n is fed by DPU n pipeline k (k = 0, 1). In
  the devicetree these are `ports/port@0` and `port@1` with endpoints to
  the DPU pipeline outputs (DPU spec 08 §8.6). Pipeline k drives DPTX
  **source k**.
- **[HW]** SST uses source 0 only. MST can carry both sources as two
  streams on one link (02 §2.9).
- **[HW]** Between the DPU and the core, a per-pipeline **video adapter**
  in the `dp_rcsu` window must be told the pixel encoding (RGB/YCbCr 4:4:4,
  4:2:2, 4:2:0) and whether the DPU sends 1 or 2 pixels per clock
  (01 §1.10, 02 §2.6.5).
- **[HW]** The DPU timing generator produces the timing; the DPTX only
  needs the same timing written into its MSA registers (02 §2.6). There is
  no hardware link from the DPU's timing registers to the DPTX's.
- **[HW]** *(vendor)* A single pixel per clock carries pixel clocks up to
  **600 MHz**. Above that (and below 1200 MHz) the DPU and the DPTX both
  switch to 2 pixels per clock (DPU spec 08 §8.6).
- **[HW]** A third devicetree port, `port@2`, can describe a downstream
  bridge (for example a DP-to-HDMI converter on the Orange Pi 6 Plus) whose
  last element decides the connector type (05 §5.1).

## 0.6 PHY relationship

- **[HW]** Units 0, 1, 3, 4 use the four USB-C/DP combo PHYs `usbc0` …
  `usbc3` (in that order). The PHY is shared with USB 3; a Type-C
  controller decides how many lanes DP gets (2 or 4) and switches the PHY
  into DP mode. The DPTX must wait for that before training (02 §2.2).
  Details: sky1-dp-tx 03.
- **[HW]** Unit 2 has a dedicated eDP/DP PHY in its 16 KiB `dp_phy` window
  and an optional eDP panel (`edp-panel` reference, backlight, panel power
  pins). Details: sky1-dp-tx 04.
- **[HW]** What the controller needs from either PHY: power-on with a
  **24 MHz** reference, a link-rate/lane-count setting, a per-lane voltage
  swing and pre-emphasis setting (levels 0–3), power states A0 (active),
  A2 and A3 (powered down), and lane-ready status visible in the core's
  `PHY_STATUS` register (01 §1.6).

## 0.7 Devicetree node (`"cix,sky1-dptx"`)

| Property | Meaning | O6 value |
|---|---|---|
| `reg` / `reg-names` | `"dp"`, `"dsc"`, `"dp_phy"`, `"dp_rcsu"` (§0.2) | |
| `interrupts` | §0.2 | |
| `clocks` / `clock-names` | `"apb_clk"`, `"vid_clk0"`, `"vid_clk1"` | |
| `resets` / `reset-names` | `"dp_rcsu_reset"`, `"dp_reset"`, (unit 2) `"phy_reset"` | |
| `phys` / `phy-names` | `"dp_phy"` → USB-C combo PHY (units 0, 1, 3, 4) | |
| `cix,dp-max-rate` | Maximum link rate, kHz of link symbol clock (162000 / 270000 / 540000 / 810000). Default 810000 | 540000 (HBR2) on all units |
| `cix,dp-lane-number` | Maximum lanes. Default 4 | absent |
| `cix,dp-max-stream` | Number of sources used. Default 2 | absent |
| `cix,aux-clock-divider` | Force the AUX divider (01 §1.4.1). Default: derived from `apb_clk` | absent |
| `cix,delay_after_hpd` | Milliseconds to wait after an HPD rising edge before probing. Default 0 | absent |
| `cix,dp-psr-default-on` | PSR enable (0/1). Default 1 | absent |
| `cix,dp-fasttrain-default-on` | Boolean: allow fast (AUX-less) re-training. Default off | absent |
| `cix,dp-mst-default-on` | Boolean: allow MST. Default off | absent |
| `pixel_per_cycle` | u8: force 1 or 2 pixels per clock. Default automatic | absent |
| `support_d3_cmd` | String `"yes"`/`"no"`: send DPCD D3 on power-down. Default yes | `"no"` on DP 1 |
| `enabled_by_gop` | 0/1: firmware left this link running (02 §2.12) | 0 |
| `cfg_adapter_port` | `2` selects the unit-2 layout of the RCSU video adapter (01 §1.10) | 2 on unit 2 |
| `edp-panel` | Reference to an eDP panel node (unit 2) | `panel_edp0` on DP 2 |
| `#sound-dai-cells` | 0. The node is an audio codec DAI (02 §2.8) | |

All properties except `reg`, `interrupts`, `clocks`, `resets` and `phys`
are vendor properties **[POLICY]**; a clean-room driver may choose its own
configuration source but should accept these for compatibility.

## 0.8 ACPI attachment (`CIXH502F`)

- **[HW]** Five ACPI devices with `_HID` `CIXH502F` (the firmware-derived
  devicetree of the Orion O6 shows them at the five `dp` window addresses
  with SPIs 332–336). They are distinct from the DPUs (`CIXH5010`).
- **[HW]** *(vendor)* `_CRS` resources are taken **by index**, not by name:

| `_CRS` item | Meaning |
|---|---|
| Memory 0 | `dp` core window |
| Memory 1 | `dsc` window (unused) |
| Memory 2 | `dp_phy` window |
| Memory 3 | `dp_rcsu` window (optional; without it YCbCr 4:2:0 and 2-pixel mode cannot be configured) |
| Interrupt 0 | Core interrupt (GSIV = SPI + 32) |

- **[HW]** *(vendor)* Everything else comes from `_DSD`, `CLKT` and `RSTL`:

| Item | Where |
|---|---|
| Clocks | `CLKT` entries named `"apb_clk"`, `"vid_clk0"`, `"vid_clk1"` (Sky1 SoC spec 01 §1.4) |
| Resets | `RSTL` entries named `"dp_reset"` and, unit 2, `"phy_reset"`. Missing entries are tolerated |
| Device properties | The `_DSD` properties of §0.7, same names and meanings |
| PHY | A `_DSD` reference property `"dp_phy"` that points to a data-only child node of the PHY device; the PHY is found on that node's parent device by the node's name |
| Video inputs | `_DSD` graph (`ports`/`port@k`/endpoints) linking to the DPU pipeline nodes, as for the DPU (DPU spec 08 §8.2.2) |
| Audio codec number | `_UID` of the device; the audio codec instance is numbered by it (05 §5.4) |

- **[POLICY]** *(vendor)* Under ACPI the `edp-panel` reference is looked up
  but only devicetree panel nodes are understood, so the eDP panel path is
  effectively devicetree-only.

> OPEN QUESTION: The exact `_DSD` content of the five `CIXH502F` devices
> (which vendor properties are present, the `dp_phy` reference target and
> the CLKT/RSTL names) needs a DSDT disassembly to confirm. The firmware-
> derived devicetree only records the first memory window and the
> interrupt.
