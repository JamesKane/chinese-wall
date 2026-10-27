# 08: CIX Sky1 Integration

This section covers the Linlon-D6 instances in the **CIX Sky1** SoC (Radxa
Orion O6, Orange Pi 6 Plus and similar boards).

Sources: the mainline Sky1 headers, plus the community Sky1-Linux
devicetree (CIX's BSD-3-Clause SoC and board files) and the CIX vendor DPU
driver it ships. Facts that come only from the vendor driver are marked
*(vendor)*. They describe what the shipping driver does on this silicon,
which is stronger evidence than the D71-model inference used in 00–07.

> **Status:** node data complete for the DPUs. The DisplayPort controller
> (`cix,sky1-dptx`) is out of scope (§8.8).

## 8.1 Instances

Sky1 has **five** identical display units, n = 0 … 4. Each has one DPU with
**two pipelines**, a separate AFBC encoder unit, and one DisplayPort
controller.

| Unit | DPU MMIO (128 KiB) | DPU IRQ (GIC SPI, level-high) | AEU MMIO (128 KiB) | AEU IRQ | DP controller node | DP IRQ |
|---|---|---|---|---|---|---|
| 0 | `0x14010000` | 316 | `0x14030000` | 317 | `0x14060000` | 332 |
| 1 | `0x14080000` | 318 | `0x140A0000` | 319 | `0x140D0000` | 333 |
| 2 | `0x140F0000` | 320 | `0x14110000` | 321 | `0x14140000` | 334 |
| 3 | `0x14160000` | 322 | `0x14180000` | 323 | `0x141B0000` | 335 |
| 4 | `0x141D0000` | 324 | `0x141F0000` | 325 | `0x14220000` | 336 |

- **[HW]** Each DPU node uses compatible `"armchina,linlon-d6"`. (The Sky1
  devicetree lists it **alone**, without the `"arm,mali-d71"` fallback the
  mainline binding requires. A mainline-conformant node would add the
  fallback.)
- **[HW]** The AEU (AFBC encoder for write-back, compatible
  `"armchina,linlon-aeu"`) is a separate device with its own register window,
  interrupt and stream ID. It shares the DPU's power domain and `aclk`.
  It is disabled on the Orion O6 and outside this spec.
- **[HW]** ACPI: the DPU's hardware ID is `CIXH5010` (vendor driver match
  table). Sky1 boards normally boot with ACPI.

## 8.2 Resources per unit n

| Resource | Value | Notes |
|---|---|---|
| `aclk` | SCMI clock `CLK_TREE_DPUn_ACLK` = 44 + n | Via the SCMI clock protocol (`0x14`) |
| `pipeline@0` `pxclk` | SCMI clock `CLK_TREE_DPn_PIXEL0` = 33 + 2n | |
| `pipeline@1` `pxclk` | SCMI clock `CLK_TREE_DPn_PIXEL1` = 34 + 2n | |
| Power domain | SCMI power domain `SKY1_PD_DPUn` = 6 + n, name `"dpu_pd"` | Trusted-firmware SMC agent, protocol `0x11` |
| Resets | `"rcsu_reset"` = `SKY1_DPUn_RCSU_RESET_N`; `"ip_reset"` = `SKY1_DPU_RESETn_N` | S5 system controller at `0x16000000` (§8.4) |
| IOMMU | `smmu_mmhub` (Arm SMMUv3 at `0x0B1B0000`), stream IDs below | |
| Unit index | `device-id = <n>` | Vendor property |

`CLK_TREE_DPU_CLK` (43) is not used by any node.

### 8.2.1 Stream IDs

**[HW]** Each unit owns a group of 6 consecutive SMMU stream IDs starting at
6n:

| Offset in group | Master |
|---|---|
| 0 | DPU, first stream |
| 1 | DPU, second stream |
| 2 | **AEU** |
| 3 | DPU, third stream |
| 4 | DPU, fourth stream |
| 5 | DPU, fifth stream |

So DPU n lists 5 IDs: 6n, 6n+1, 6n+3, 6n+4, 6n+5. The AEU uses 6n+2. This
grouping matches the firmware's ACPI IORT table (as reported by the
Sky1-Linux maintainers).

> OPEN QUESTION: Which of the 5 DPU streams carries which layer or the
> write-back layer is not stated. By the binding's convention (01 §1.1.1)
> they would be layers 0–3 and write-back of one pipeline, but there are two
> pipelines and only 5 IDs. The streams may be per-AXI-port rather than per
> layer. Grant all 5 the same translations.

- **[HW]** The mmhub SMMU nodes are `dma-coherent` in the Sky1-Linux tree.

### 8.2.2 ACPI attachment (`CIXH5010`)

**[HW]** Five ACPI devices with `_HID` `CIXH5010`, one per unit. Their
`_CRS` carries the same resources as the devicetree node: one
`Memory32Fixed` (the 128 KiB DPU window, §8.1) and one `Interrupt` (GSIV =
SPI + 32, so 348, 350, 352, 354, 356). (Confirmed by a devicetree generated
from the Orion O6's real ACPI tables, where these five devices appear at
the DPU addresses with the DPU interrupts, mislabelled as USB.)

The rest of the description is in `_DSD` (device-properties UUID
`daffd814-6eba-4d8c-8a91-bc9bbf4aa301`), the hierarchical data-node
extension, and the CIX tables:

| Item | Where | Notes |
|---|---|---|
| Pipelines | Data-only child nodes whose names start with `pipeline` | Each has an integer `reg` = pipeline index. Every hardware pipeline must have one |
| Output ports | Under each pipeline node: an optional `ports` node, then `port…` child nodes with integer `reg` | Port 0 is the output. Its endpoints 0 and 1 link to DP controller inputs (ACPI `_DSD` graph convention) |
| `aclk` | `CLKT` entry with connection name `"aclk"` (Sky1 SoC spec 01 §1.4) | |
| Pixel clock of pipeline k | `CLKT` entry on the DPU device whose connection name is **the pipeline node's name** | The vendor driver looks the clock up by that name |
| Resets | `RSTL` entries named `"ip_reset"` and `"rcsu_reset"` | Optional under ACPI |
| `aclk_freq_fixed`, `side_by_side_master`, `smart_aclk_freq`, `device-id`, `enabled_by_gop` | `_DSD` device properties | Same meaning as the DT properties (§8.3, §8.5, 06 §6.6) |
| `pixel_per_cycle` | `_DSD` property of the device that owns the pipeline node | 1 or 2 |
| Power | ACPI power resources / `_PS0` | No SCMI power-domain provider under ACPI (Sky1 SoC spec 01 §1.3.4) |

> OPEN QUESTION: The exact names of the pipeline data nodes (and so of the
> pixel-clock CLKT entries), and whether `aclk_freq_fixed` is a 32- or
> 64-bit integer in the DSDT (the vendor driver tries both), need a DSDT
> disassembly to confirm.

### 8.2.3 DMA and coherency rules

- **[HW]** The DPU addresses 40 bits.
- **[HW]** The DPU is a **non-snooping** master: its layer fetches read the
  system-level cache / DRAM and never snoop CPU or GPU caches. Every
  scanout buffer must be visible in memory before it is flipped to:
  - CPU-written buffers: map them write-combined (non-cacheable), or clean
    the CPU caches over the buffer before the flip.
  - GPU-rendered buffers: the GPU must use ACE-Lite with non-cacheable
    memory attributes on Sky1 (Panthor spec 11.7); otherwise GPU L2
    contents may not reach memory in time and the display shows stale
    tiles.
- **[HW]** Treat the DPU as non-coherent whatever the firmware's `_CCA`
  says. The DPU's SMMU (mmhub) is itself `dma-coherent` for its own
  page-table walks, which does not make the DPU's data reads coherent.
- **[HW]** IORT: each DPU's five stream IDs (§8.2.1) map to the mmhub SMMU.
  The firmware boot-splash range is an IORT reserved memory region that must
  stay identity-mapped until takeover (§8.5).

> OPEN QUESTION: The DPU's own `_CCA` value in the DSDT is not known. The
> GPU's is 1 and wrong (Panthor spec 11.7); assume the DPU's may be too.

- **[ABI]** *(vendor)* The DPU device exposes no render node. Exposing one
  makes compositors pick the DPU instead of the GPU for rendering.

## 8.3 Clocks

- **[POLICY]** *(vendor)* On the Orion O6, every DPU sets
  `aclk_freq_fixed = <800000000>`: `aclk` is always requested at **800 MHz**,
  whatever the mode. This satisfies the `aclk` ≥ pixel-clock rule for every
  supported mode at one pixel per clock (§8.6).
- **[POLICY]** *(vendor)* Without the fixed rate, an optional
  "smart" table picks `aclk` from the pixel clock: ≥ 445.5 MHz → 800 MHz;
  ≥ 297 MHz → 600 MHz; ≥ 148.5 MHz → 400 MHz; ≥ 74.25 MHz → 300 MHz;
  otherwise 200 MHz.
- **[HW]** The vendor driver accepts a pixel clock whose rounded rate is
  **at least** the requested rate (not only an exact match, as in 01 §1.2).
  This resolves the concern that SCMI clock rounding would reject modes.

## 8.4 Resets and power

| Reset | S5 controller word | Bit |
|---|---|---|
| `SKY1_DPU_RESETn_N` ("ip_reset", index 19 + n) | `0x400` | 16 + n |
| `SKY1_DPUn_RCSU_RESET_N` ("rcsu_reset", index 110 + n) | `0x800` | 19 + n |

- **[HW]** Active-low: bit cleared = in reset.
- **[POLICY]** *(vendor)* The driver does **not** reset the DPU at probe. It
  relies on power-domain firmware to bring the DPU up out of reset. It
  takes both reset lines but only ever pulses `ip_reset`, when taking over
  from the boot firmware with a mode change (§8.5): assert, wait 1 ms,
  deassert, wait 1 ms. `rcsu_reset` is never toggled.
- **[HW]** Power domain 6 + n must be on before any register access
  (01 §1.6).

## 8.5 Boot-firmware (GOP) framebuffer handover

The UEFI firmware can leave DPU 0 scanning out a boot splash.

- **[HW]** Reserved memory: `0x84800000`, **8 MiB**, `no-map`, with
  identity IOMMU mappings (`iommu-addresses`) for all five DPUs. The IORT
  declares the same range as a reserved memory region, so the SMMU must
  keep it identity-mapped until the OS driver takes over. (The Sky1-Linux
  tree at one point declared 32 MiB. The IORT says 8 MiB.)
- **[HW]** Firmware handover query: SMC (SiP service) function ID
  `0xC200000F`, argument 1 = `1` (get). The result's bit 17 (`0x00020000`)
  is a handover flag. Argument 1 = `2` (set) with bit 17 in argument 2
  sets it. The vendor driver defines "set" but never calls it.
- **[POLICY]** *(vendor)* The handover logic is:
  1. Read the DT/ACPI property `enabled_by_gop` (0 or 1). Then query the
     SMC. If bit 17 is set, the driver treats the display as already taken
     over by an OS driver (it names this case "module inserted before") and
     ignores `enabled_by_gop`.
  2. While GOP-enabled: skip the soft reset during block discovery
     (03 §3.2 step 1), and mark the device runtime-active at probe so the
     power domain is not cycled.
  3. At the first display enable: compare the requested mode's active size
     and porches with the values in pipeline 0's `BS_ACTIVESIZE`,
     `BS_HINTERVALS` and `BS_VINTERVALS`. If they match, keep the running
     timing and just program the new layers. If they differ:
     a. mask interrupts;
     b. phase 1: disable pipeline-0 layers 0 and 2, clear `CU_INPUT_ID0`
        and `CU_INPUT_ID1`, disable both scalers and the merger, then write
        `GCU_CONFIG_VALID0` = 1 and poll until it reads 0;
     c. phase 2: clear `IPS_INPUT_ID0`, disable IPS and BS, set
        `GCU_CONFIG_VALID0` again and poll;
     d. gate the pixel clock, pulse `ip_reset` (§8.4), and continue with a
        normal enable (03 §3.8).
- **[HW]** This shows the firmware splash uses pipeline 0, layers 0 and 2,
  possibly both scalers and the merger.

> OPEN QUESTION: Who sets the SMC handover flag is not shown (the vendor
> driver never does). A clean-room driver can avoid depending on it: check
> whether pipeline 0's `BS_CONTROL.EN` is set at probe, and if so do the
> takeover above.

## 8.6 Output wiring and high pixel rates

- **[HW]** On the Orion O6, every DPU pipeline k drives **input port k** of
  DP controller n: pipeline 0 → DP stream 0, pipeline 1 → DP stream 1. Each
  pipeline has a single endpoint, so neither is dual-link. The DP
  controller's two video clocks (`DPCn_VIDCLK0/1`) match the two streams
  (multi-stream DP, or two halves of one wide stream, §8.7).
- **[HW]** Board use: DP 0, 1, 3 and 4 go through the four combined USB-C/DP
  PHYs (`usbc0`, `usbc1`, `usbc2`, `usbc3` DP PHYs). DP 2 has its own PHY
  window and can drive an eDP panel (DP2 panel power/backlight pins).
- **[HW]** *(vendor)* A single pipeline handles pixel clocks up to
  **600 MHz**. For 600 MHz < pixel clock < 1200 MHz the driver switches to
  **2 pixels per clock with side-by-side composition** (§8.7). Modes at or
  above 1200 MHz are rejected.

## 8.7 Linlon-D6 features used on Sky1 beyond the D71 model

These come from the vendor driver. They are described in register terms in
02, 04 and 06. Summary:

| Feature | Where | Summary |
|---|---|---|
| Different LPU AXI field layout | 02 §2.5.4 | Outstanding caps are 8 bits; cap-B enable moved to bit 22 |
| Pixels per clock | 02 §2.15.1, 06 §6.4.4 | `BS_CONTROL[22:21]` = 1 selects 2 pixels per clock |
| Side-by-side | 06 §6.6 | Each pipeline composes half of the screen; the master IPS merges the slave IPS output |
| Per-layer colour pipeline | 04 §4.9 | Degamma table, 3×3 matrix and gamma table per layer, selected from shared table blocks |
| Display degamma | 06 §6.2.3 | `IPS_CONTROL.IT` |
| RGB→YUV matrices | 04 §4.8, 06 §6.3 | Programmed explicitly (resolves the open questions in 02 §2.7 and §2.13) |
| Config-valid readback | 03 §3.4 | `GCU_CONFIG_VALID0/1` bit 0 reads 1 until latched |
| Fixed block offsets | 02 §2.17 | Pipeline-0 block addresses used by the firmware handover |

- **[HW]** *(vendor)* Known product IDs in this family: `0x0080`
  Linlon-D8, `0x0060` Linlon-D6, `0x0020` Linlon-D2. (The vendor driver
  still decodes `GLB_CORE_ID` minor/status with the old wrong widths,
  02 §2.3.)

## 8.8 Out of scope: the DisplayPort controller

For reference only (BSD-3-Clause devicetree facts). Each DP controller node
(`cix,sky1-dptx`) has register windows `"dp"` (16 KiB at node + `0x4000`),
`"dsc"` (16 KiB at + `0x8000`), `"dp_phy"`, and `"dp_rcsu"` (at node −
`0x10000`), clocks `apb_clk` = `DPCn_APBCLK`, `vid_clk0/1` =
`DPCn_VIDCLK0/1`, resets `SKY1_DPn_RCSU_RESET_N` and `SKY1_DP_RESETn_N`, and
a property `cix,dp-max-rate` (540000 = HBR2 on the O6). The vendor driver
is a Trilinear DPTX driver with MST, HDCP 2.x, audio and PSR. It needs its
own extraction.

## 8.9 Bring-up checklist

1. Power domain 6 + n on (SCMI SMC agent). `aclk` = 800 MHz via SCMI.
2. Do **not** soft-reset if the firmware splash is live on DPU 0 (§8.5).
   Otherwise soft-reset and walk the blocks (03 §3.2). Expect product
   `0x0060`, two pipelines, no PERIPH block.
3. Map stream IDs 6n, 6n+1, 6n+3, 6n+4, 6n+5. Keep `0x84800000` + 8 MiB
   identity-mapped until the first modeset completes.
4. Program the AXI controls (02 §2.5.4) or leave reset values.
5. Start with one pipeline, one linear `XRGB8888` layer at ≤ 600 MHz pixel
   clock, check that `CVAL` and `PL0` interrupts arrive.
6. Bring up the DP controller separately.
