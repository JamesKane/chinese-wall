# 08: CIX Sky1 Integration

This section covers what is known about the Linlon-D6 instances in the
**CIX Sky1** SoC (Radxa Orion O6 and similar boards).

> **Status: partial.** At the source baseline, mainline has **no Sky1
> display devicetree node**: no DPU node, no DisplayPort controller node,
> and no driver for the Sky1 DisplayPort controller or PHY. The DPU driver
> accepts the Linlon-D6 compatible, and the Sky1 headers fix the clock,
> reset and power-domain IDs a DPU node would use. MMIO addresses,
> interrupts, IOMMU stream IDs and the encoder are not known.

## 8.1 Instances

The Sky1 headers give **five** display units, numbered 0–4. Each has its own
DPU, power domain, clocks, resets and DisplayPort controller ("DPC" / "DP"):

| Unit | DPU power domain | DPU `aclk` | Pixel clocks (pipeline 0 / 1) | DP controller clocks |
|---|---|---|---|---|
| 0 | `SKY1_PD_DPU0` = 6 | `CLK_TREE_DPU0_ACLK` = 44 | `CLK_TREE_DP0_PIXEL0` = 33 / `CLK_TREE_DP0_PIXEL1` = 34 | `DPC0_VIDCLK0` = 49, `DPC0_VIDCLK1` = 50, `DPC0_APBCLK` = 59 |
| 1 | `SKY1_PD_DPU1` = 7 | `CLK_TREE_DPU1_ACLK` = 45 | 35 / 36 | 51, 52, 60 |
| 2 | `SKY1_PD_DPU2` = 8 | `CLK_TREE_DPU2_ACLK` = 46 | 37 / 38 | 53, 54, 61 |
| 3 | `SKY1_PD_DPU3` = 9 | `CLK_TREE_DPU3_ACLK` = 47 | 39 / 40 | 55, 56, 62 |
| 4 | `SKY1_PD_DPU4` = 10 | `CLK_TREE_DPU4_ACLK` = 48 | 41 / 42 | 57, 58, 63 |

Also: `CLK_TREE_DPU_CLK` = 43 (one shared DPU clock, purpose unknown).

- The two pixel clocks per unit fit a DPU with **two pipelines** (one
  `pxclk` per `pipeline@N` node, 01 §1.1.2). The DP controller's two video
  clocks fit a DP controller with two input streams (for example
  multi-stream transport). Both readings come from the names only.

> OPEN QUESTION: Whether each Sky1 DPU really has two pipelines, and how
> many layers and scalers each has, must be read from the hardware
> (`GLB_CORE_INFO`, block walk; 03 §3.2).

> OPEN QUESTION: Whether a Sky1 DPU node would list `CLK_TREE_DPUn_ACLK`
> or `CLK_TREE_DPU_CLK` as `aclk` is unknown. The per-unit ACLK is the
> likely match for the binding's single `"aclk"`.

## 8.2 Clocks

- **[HW]** All IDs in §8.1 are **SCMI clock-protocol** identifiers (protocol
  `0x14`, served by the power-management processor through the mailbox SCMI
  agent; see the Panthor spec, section 11 §11.5.1, for the transport).
- **[HW]** The DPU driver sets rates on both `aclk` and `pxclk` (01 §1.2).
  On Sky1 these become SCMI `CLOCK_RATE_SET` requests, and mode validation
  depends on the firmware's rate rounding (SCMI `CLOCK_DESCRIBE_RATES`).
  The DPU's exact-rate check on `pxclk` (07 §7.5) only passes if the
  firmware can produce the exact pixel rate or reports the rounded rate
  honestly.

> OPEN QUESTION: Whether the SCMI firmware allows the OS to set the DP pixel
> clock rate freely, and with what granularity, is unknown. If it only
> offers a fixed set of rates, many modes will fail the exact-match rule,
> and a clean-room driver may need to relax it (for example by accepting a
> small tolerance, as DisplayPort sinks do).

## 8.3 Power domains

- **[HW]** `SKY1_PD_DPU0–4` (6–10) are SCMI **power-domain** protocol
  (`0x11`) IDs, served by the trusted-firmware SMC agent (as for the GPU's
  `SKY1_PD_GPU`). The matching DPU domain must be on before any DPU register
  access (01 §1.6 step 1).
- **[HW]** The DPU's IOMMU is likely in the multimedia hub, which has its
  own domains `SKY1_PD_MMHUB` = 4 and `SKY1_PD_MMHUB_SMMU` = 5.

> OPEN QUESTION: Whether a DPU node also needs `SKY1_PD_MMHUB` or
> `SKY1_PD_MMHUB_SMMU` listed, and whether the Sky1 DPUs sit behind an SMMU
> at all, is not in the source.

## 8.4 Resets

The Sky1 S5 system controller (`cix,sky1-s5-system-control`, MMIO
`0x16000000`, 4 KiB) has these display reset lines:

| Identifier | Index | Register offset | Bit |
|---|---|---|---|
| `SKY1_DPU_RESET0_N` … `SKY1_DPU_RESET4_N` | 19–23 | `0x400` (group 0, S0 domain word 0) | 16–20 |
| `SKY1_DP_RESET0_N` … `SKY1_DP_RESET4_N` | 24–28 | `0x400` | 21–25 |
| `SKY1_DP_PHY_RST_N` | 29 | `0x400` | 26 |
| `SKY1_USB_DP_PHY0_PRST_N` … `PHY3_PRST_N` | 51–54 | `0x408` (group 1, USB PHYs) | 0–3 |
| `SKY1_USB_DP_PHY0_RST_N` … `PHY3_RST_N` | 55–58 | `0x408` | 4–7 |
| `SKY1_DP0_RCSU_RESET_N` … `SKY1_DP4_RCSU_RESET_N` | 105–109 | `0x800` (group 0, RCSU word) | 14–18 |
| `SKY1_DPU0_RCSU_RESET_N` … `SKY1_DPU4_RCSU_RESET_N` | 110–114 | `0x800` | 19–23 |

- **[HW]** All lines are active-low: clearing the bit holds the block in
  reset, setting it releases reset. A read of 0 means "in reset".
- **[POLICY]** The Sky1 reset driver waits 50–100 µs after each assert and
  each deassert.
- **[HW]** The DPU binding has **no** `resets` property, and the DPU driver
  uses only the internal soft reset (`GCU_CONTROL.SRST`, 03 §3.2). On Sky1
  the external DPU reset must therefore already be released, by firmware or
  by power-domain sequencing, before the driver probes.

> OPEN QUESTION: Whether firmware releases `SKY1_DPUn_RESET_N` and
> `SKY1_DPUn_RCSU_RESET_N` when power domain 6 + n turns on is unknown. If
> it does not, a clean-room driver must deassert them itself (DPU reset
> first, then RCSU, order unconfirmed) after the power domain is on and
> before the first register read. "RCSU" is not expanded in the source.

> OPEN QUESTION: The four "USB_DP" PHYs are combined USB/DP PHYs (USB-C
> alternate mode). Which DPU unit drives which PHY or connector is not
> stated.

## 8.5 Encoders and PHYs

- The DPU drives a parallel pixel bus into an encoder (01 §1.3). On Sky1
  that encoder is the matching DP controller (one per unit). Its driver,
  register map and binding are **not** in mainline at the baseline.
- The Sky1 pin-function header lists pins for an embedded-panel power and backlight
  interface on DP unit 2 (`DP2_DIGON`, `DP2_BLON`, `DP2_VARY_BL`), which
  suggests unit 2 can drive an eDP panel.

> OPEN QUESTION: Everything about the DP controller (identity, registers,
> link training, HPD, audio, MST) is outside this spec. It needs its own
> extraction once a source exists.

## 8.6 Devicetree status and expected node shape

- **[HW]** A Sky1 DPU node would use `compatible = "armchina,linlon-d6",
  "arm,mali-d71"` (01 §1.1.1). No Sky1-specific DPU compatible exists.
- Expected resources per unit n, from §8.1–8.4: `clocks = <&scmi_clk
  CLK_TREE_DPUn_ACLK>` named `"aclk"`; `power-domains = <&<SMC SCMI
  power-domain provider> SKY1_PD_DPUn>`; per pipeline k, `clocks =
  <&scmi_clk CLK_TREE_DPn_PIXELk>` named `"pxclk"`; a graph port to DP
  controller n.

> OPEN QUESTION: MMIO base and size, interrupt numbers and trigger type,
> and `iommus` stream IDs of the five DPUs are not in the baseline source.

> OPEN QUESTION: Sky1 boards often boot with ACPI. The DPU driver is
> devicetree-only. No ACPI identifier for the DPU appears in the source.

## 8.7 Bring-up checklist

1. Get the DPU node data (§8.6) from a source the clean room may use.
2. Turn on SCMI power domain 6 + n (§8.3).
3. Make sure the DPU reset lines are released (§8.4).
4. Enable `aclk` through SCMI clocks. Read `GLB_CORE_ID` and expect
   product `0x0060` (03 §3.1).
5. Soft-reset and walk the blocks (03 §3.2). Log the pipeline, layer and
   scaler counts, PERIPH presence (expect none on a D6), line sizes and
   capability bits. Refuse unknown block types with a clear message rather
   than failing silently.
6. Start with one pipeline, one linear `XRGB8888` layer, no scaling, a
   standard mode, and check `CVAL` and `PL0` interrupts arrive (03 §3.4–3.5).
7. Bring up the DP controller separately. The DPU only supplies pixels and
   timing.
