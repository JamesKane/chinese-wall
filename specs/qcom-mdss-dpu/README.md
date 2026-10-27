# Qualcomm SC8280XP MDSS v8.1 / DPU1 / DisplayPort / DISP_CC / QMP Combo PHY Specification

## 1. Document Index

**[HW]** This directory contains clean-room hardware specifications for the Qualcomm SC8280XP Mobile Display Subsystem (`MDSS`), Display Processing Unit (`DPU`), DisplayPort (`DP` / `eDP`) controllers, Display Clock Controllers (`DISP_CC`), and QMP V5 5nm USB4/USB3/DP Combo PHYs, including board-specific routing on the Radxa Dragon Q8B.

| Document | Scope |
|---|---|
| `README.md` | Top-level architecture, dual-MDSS (`MDSS0` and `MDSS1`) memory map, SMMU stream mappings, interrupt hierarchy, power domains, and Dragon Q8B board routing. |
| `01-mdss-dpu1-pipeline.md` | MDSS wrapper registers, UBWC 4.0 memory compression setup, VBIF configuration, DPU v8.0/v8.1 catalog (`TOP`, `CTL`, `SSPP`, `QSEED3lite`, `CSC`, `LM`, `DSPP`, `MERGE_3D`, `PINGPONG`, `DSC 1.2`, `INTF`), interrupt banks, and atomic frame flush sequence. |
| `02-dp-controller-aux-link-tu.md` | DP/eDP controller register map (`AHB`, `AUX`, `LINK`, `P0`), AUX native and I2C-over-AUX transactions, HPD state machine, Link Training 1 and 2, Main Stream Attributes (`MVID`/`NVID`), widebus timing, and Transfer Unit (`TU`) sizing algorithm. |
| `03-dispcc-and-combo-phy.md` | `DISP_CC_0` and `DISP_CC_1` Lucid 5LPE PLLs, Root Clock Generators (`RCGs`), branch clocks, resets, GDSCs, and QMP V5 5nm Combo PHY initialization, DP rate configuration, voltage swing/pre-emphasis tables, and Type-C DP Alt-Mode pin multiplexing. |

---

## 2. Clean-Room and Permissive Register Definition Notice

**[ABI]** The mainline kernel source tree provides standalone XML hardware register definitions under `drivers/gpu/drm/msm/registers/display/` (specifically `mdss.xml`, `dpu.xml`, `dsi.xml`, `edp.xml`, and `hdmi.xml`) dual-licensed under `MIT` or `BSD-2-Clause` (`SPDX-License-Identifier: MIT OR BSD-2-Clause`). A clean-room implementation may directly consume or generate headers from those permissive XML files without GPL taint, while relying on the clean-room specifications in this directory for all functional behavior, hardware catalogs, DisplayPort controller registers, clock trees, PHY initialization tables, and programming sequences.

---

## 3. SC8280XP Dual-MDSS Architecture Overview

**[HW]** The SC8280XP SoC integrates two independent display subsystem instances (`MDSS0` at `0x0ae00000` and `MDSS1` at `0x22000000`), each paired with its own Display Clock Controller (`DISP_CC_0` at `0x0af00000` and `DISP_CC_1` at `0x22100000`).

**[HW]** Each MDSS instance contains:
- An MDSS top-level wrapper (`+0x00000`) providing hardware version identification, level-1 interrupt aggregation (`HW_INTR_STATUS`), and Universal Bandwidth Compression (`UBWC`) decoder static configuration.
- A Display Processing Unit (`DPU`, also referred to as `MDP`, at `+0x01000`) implementing 8 source pipes (`SSPP`: 4 VIG + 4 DMA), 6 Layer Mixers (`LM_0..5`), 4 Destination Scalar/Post-Processing blocks (`DSPP_0..3`), 6 Control paths (`CTL_0..5`), 3 3D Merge blocks (`MERGE_3D_0..2`), 6 Ping-Pong buffers (`PINGPONG_0..5`), 6 Display Stream Compression 1.2 encoders (`DSC_0..5`), 1 Chroma Downsampler (`CDM_0`), and 9 Timing Engine Interfaces (`INTF_0..8`, of which `INTF_0`, `INTF_3`, `INTF_4`, `INTF_5`, `INTF_6`, `INTF_7`, and `INTF_8` are active DisplayPort/eDP interfaces and `INTF_1..2` are DSI interfaces).
- Four DisplayPort / Embedded DisplayPort controllers (`DP0` at `+0x90000`, `DP1` at `+0x98000`, `DP2` at `+0x9a000`, and `DP3` at `+0xa0000`).
- A Virtual Bus Interface (`VBIF_RT` at `+0xb0000`) mediating AXI memory fetches between the DPU source pipes and the system memory interconnect via the Apps SMMU.

**[HW]** On the Radxa Dragon Q8B (`SC8280XP`), only `MDSS0` (`0x0ae00000`) and `DISP_CC_0` (`0x0af00000`) are wired to external display connectors:
- `MDSS0_DP0` (`0x0ae90000`, fed by `INTF_0` or `INTF_3`) routes to USB-C Port 0 via QMP V5 Combo PHY 0 (`0x088eb000`).
- `MDSS0_DP1` (`0x0ae98000`, fed by `INTF_4` or `INTF_8`) routes to USB-C Port 1 via QMP V5 Combo PHY 1 (`0x08903000`).
- `MDSS0_DP2` (`0x0ae9a000`, fed by `INTF_6` or `INTF_7`) routes via dedicated eDP/DP PHY 2 (`0x0aec2000`) to an onboard Chrontel CH7218A DisplayPort-to-HDMI bridge driving the full-size HDMI connector.
- At boot, UEFI initializes `MDSS0` with `SSPP_VIG2` -> `LM_2` -> `CTL_2` -> `INTF_6` -> `DP2` scanning out the boot framebuffer at `0xc6200000` (`1920x1080` 32-bit `XRGB8888`, stride `7680` bytes) over 4 lanes at HBR3 (`8.1 Gbps/lane`) with widebus enabled.

---

## 4. Top-Level Physical Memory Map

**[HW]** All display-related MMIO regions on SC8280XP use 32-bit little-endian register accesses unless noted otherwise (QMP PHY SerDes/PCS registers are 8-bit values aligned on 32-bit boundaries).

### 4.1 MDSS0 and DISP_CC_0 (`0x0ae00000` – `0x0af20000`)

| Sub-Block | Physical Base | Offset from `MDSS0` | Size | Description |
|---|---|---|---|---|
| `MDSS0_TOP` | `0x0ae00000` | `+0x00000` | `0x01000` | MDSS0 wrapper, Level-1 IRQ controller, UBWC decoder configuration |
| `MDSS0_DPU` (`MDP`) | `0x0ae01000` | `+0x01000` | `0x8f000` | DPU1 pipeline sub-blocks (`TOP`, `SSPP`, `CTL`, `LM`, `DSPP`, `PP`, `DSC`, `INTF`) |
| `MDSS0_DP0` | `0x0ae90000` | `+0x90000` | `0x01400` | DisplayPort Controller 0 (`AHB` `+0x0`, `AUX` `+0x200`, `LINK` `+0x400`, `P0` `+0x1000`) |
| `MDSS0_DP0_P1` | `0x0ae92000` | `+0x92000` | `0x00200` | DisplayPort Controller 0 Pixel Clock 1 domain (`MST` stream 1) |
| `MDSS0_DP1` | `0x0ae98000` | `+0x98000` | `0x01400` | DisplayPort Controller 1 (`AHB` `+0x0`, `AUX` `+0x200`, `LINK` `+0x400`, `P0` `+0x1000`) |
| `MDSS0_DP2` | `0x0ae9a000` | `+0x9a000` | `0x01400` | DisplayPort Controller 2 (`AHB` `+0x0`, `AUX` `+0x200`, `LINK` `+0x400`, `P0` `+0x1000`) |
| `MDSS0_DP3` | `0x0aea0000` | `+0xa0000` | `0x01400` | DisplayPort / eDP Controller 3 (`AHB` `+0x0`, `AUX` `+0x200`, `LINK` `+0x400`, `P0` `+0x1000`) |
| `MDSS0_VBIF_RT` | `0x0aeb0000` | `+0xb0000` | `0x01000` | Real-Time Virtual Bus Interface (`VBIF`) |
| `MDSS0_DP2_PHY` | `0x0aec2000` | `+0xc2000` | `0x00c00` | Dedicated DP2 PHY (`QSERDES_COM` `+0x0`, `TX0` `+0x200`, `TX1` `+0x600`, `DP_PHY` `+0xa00`) |
| `MDSS0_DP3_PHY` | `0x0aec5000` | `+0xc5000` | `0x00c00` | Dedicated DP3/eDP PHY (`QSERDES_COM` `+0x0`, `TX0` `+0x200`, `TX1` `+0x600`, `DP_PHY` `+0xa00`) |
| `DISP_CC_0` | `0x0af00000` | `+0x100000` | `0x20000` | Display Clock Controller 0 (`PLL0..2`, RCGs, branch clocks, `MDSS_GDSC`) |

**[ABI]** Under ACPI on the Radxa Dragon Q8B, device `GPU0` (`QCOM0636`) memory resource 0 maps `0x0ae00000` with a size of `0x00200000` (`2 MB`), covering both `MDSS0` (`0x0ae00000`–`0x0aecffff`) and `DISP_CC_0` (`0x0af00000`–`0x0af1ffff`) within a single contiguous BAR.

### 4.2 MDSS1 and DISP_CC_1 (`0x22000000` – `0x22120000`)

**[HW]** `MDSS1` and `DISP_CC_1` have an identical internal register offset layout to `MDSS0` and `DISP_CC_0`, shifted to base `0x22000000`:

| Sub-Block | Physical Base | Offset from `MDSS1` | Size | Description |
|---|---|---|---|---|
| `MDSS1_TOP` | `0x22000000` | `+0x00000` | `0x01000` | MDSS1 wrapper, Level-1 IRQ controller, UBWC decoder configuration |
| `MDSS1_DPU` (`MDP`) | `0x22001000` | `+0x01000` | `0x8f000` | MDSS1 DPU1 pipeline sub-blocks |
| `MDSS1_DP0` | `0x22090000` | `+0x90000` | `0x01400` | MDSS1 DisplayPort Controller 0 |
| `MDSS1_DP1` | `0x22098000` | `+0x98000` | `0x01400` | MDSS1 DisplayPort Controller 1 |
| `MDSS1_DP2` | `0x2209a000` | `+0x9a000` | `0x01400` | MDSS1 DisplayPort Controller 2 |
| `MDSS1_DP3` | `0x220a0000` | `+0xa0000` | `0x01400` | MDSS1 DisplayPort Controller 3 |
| `MDSS1_VBIF_RT` | `0x220b0000` | `+0xb0000` | `0x01000` | MDSS1 Real-Time Virtual Bus Interface |
| `DISP_CC_1` | `0x22100000` | `+0x100000` | `0x20000` | Display Clock Controller 1 |

### 4.3 External USB4/USB3/DP QMP Combo PHYs (`0x088eb000`, `0x08903000`)

**[HW]** `MDSS0_DP0` and `MDSS0_DP1` use the shared USB-C QMP V5 5nm Combo PHYs in the Periph SS address space:

| Sub-Block | Port 0 Base (`DP0`) | Port 1 Base (`DP1`) | Offset from PHY Base | Size | Description |
|---|---|---|---|---|---|
| `DP_COM` | `0x088eb000` | `0x08903000` | `+0x0000` | `0x0200` | Type-C Combo Common block (`PHY_MODE_CTRL`, `TYPEC_CTRL`, resets, powerdown) |
| `TXA` (Lanes 0/1) | `0x088eb400` | `0x08903400` | `+0x0400` | `0x0400` | Combo TX Lane A (`+0x000` Lane 0, `+0x200` Lane 1) |
| `RXA` (Lanes 0/1) | `0x088eb600` | `0x08903600` | `+0x0600` | `0x0400` | Combo RX Lane A (`+0x000` Lane 0, `+0x200` Lane 1) |
| `TXB` (Lanes 2/3) | `0x088eba00` | `0x08903a00` | `+0x0a00` | `0x0400` | Combo TX Lane B (`+0x000` Lane 2, `+0x200` Lane 3) |
| `RXB` (Lanes 2/3) | `0x088ebc00` | `0x08903c00` | `+0x0c00` | `0x0400` | Combo RX Lane B (`+0x000` Lane 2, `+0x200` Lane 3) |
| `USB3_SERDES` | `0x088ec000` | `0x08904000` | `+0x1000` | `0x0200` | USB3 QSERDES Common PLL block |
| `USB3_PCS_MISC` | `0x088ec200` | `0x08904200` | `+0x1200` | `0x0200` | USB3 PCS Misc registers |
| `USB3_PCS` | `0x088ec400` | `0x08904400` | `+0x1400` | `0x0300` | USB3 PCS control and status |
| `USB3_PCS_USB` | `0x088ec700` | `0x08904700` | `+0x1700` | `0x0300` | USB3 PCS USB3-specific registers |
| `DP_SERDES` | `0x088ed000` | `0x08905000` | `+0x2000` | `0x0200` | DisplayPort QSERDES V4/V5 Common PLL block |
| `DP_DP_PHY` | `0x088ed200` | `0x08905200` | `+0x2200` | `0x0200` | DisplayPort PHY control, AUX config, VCO div, status |

---

## 5. Interrupt Routing and Hierarchy

**[HW]** Each MDSS instance has a single shared GIC SPI interrupt line feeding into the MDSS wrapper's Level-1 interrupt status register (`HW_INTR_STATUS` at `MDSS_BASE + 0x0010`):

| Instance | GIC SPI Number | GIC GSIV (SPI + 32) | Trigger Type | Description |
|---|---|---|---|---|
| `MDSS0` | `83` | `115` (`0x73`) | Level-high | Aggregated interrupt for `MDSS0_DPU`, `DP0..3`, `DSI0..1` |
| `MDSS1` | `585` | `617` (`0x269`) | Level-high | Aggregated interrupt for `MDSS1_DPU`, `MDSS1_DP0..3`, `MDSS1_DSI0..1` |

**[HW]** The `HW_INTR_STATUS` register (`MDSS_BASE + 0x0010`) is a read-only bitmap reflecting active interrupts from child sub-blocks:

| Bit | Mask | Child Sub-Block | Target Controller / Notes |
|---|---|---|---|
| `0` | `0x00000001` | `MDP` (`DPU1`) | DPU pipeline interrupts (`INTR`, `INTR2`, `INTF_TEAR_INTR`) |
| `4` | `0x00000010` | `DSI_0` | DSI Controller 0 |
| `5` | `0x00000020` | `DSI_1` | DSI Controller 1 |
| `12` | `0x00001000` | `DP_0` | DisplayPort Controller 0 (`0x0ae90000`) |
| `13` | `0x00002000` | `DP_1` | DisplayPort Controller 1 (`0x0ae98000`) |
| `14` | `0x00004000` | `DP_2` | DisplayPort Controller 2 (`0x0ae9a000`, Dragon Q8B HDMI bridge) |
| `15` | `0x00008000` | `DP_3` | DisplayPort / eDP Controller 3 (`0x0aea0000`) |

**[HW]** Because `HW_INTR_STATUS` is a pure status aggregator with no write-to-clear latch of its own, the top-level interrupt handler must dispatch to each child controller whose bit is set in `HW_INTR_STATUS`, and the child handler must acknowledge its own interrupt status registers so the level-high line deasserts.

---

## 6. Apps SMMU (`0x15000000`) Stream IDs and DMA Constraints

**[HW]** All DPU memory reads (via `VBIF_RT`) pass through the ARM MMU-500 Apps SMMU at `0x15000000`:

| Master | Stream ID | Mask | Notes |
|---|---|---|---|
| `MDSS0` Real-Time (`VBIF_RT` XINs) | `0x0800` | `0x0402` | Used for all `MDSS0` scanout fetches (`SSPP`) |
| `MDSS1` Real-Time (`VBIF_RT` XINs) | `0x1800` | `0x0402` | Used for all `MDSS1` scanout fetches |

**[HW]** Important DMA and SMMU constraints on SC8280XP / Radxa Dragon Q8B:
1. **32-bit SSPP Address Registers**: All DPU source pipe plane address registers (`SSPP_SRC0_ADDR`..`SSPP_SRC3_ADDR`) are 32 bits wide. Whether the Apps SMMU stream `0x0800` is in identity/bypass mode or translated via a stage-1 context bank, the IOVA (or physical address in bypass mode) programmed into the SSPP must fit below `4 GiB` (`0x00000000`–`0xffffffff`).
2. **UEFI Boot Framebuffer Hand-off**: On the Radxa Dragon Q8B, UEFI configures Apps SMMU stream `0x0800` as an identity/bypass Stream Match Register (`SMR`) so that `MDSS0` can continuously fetch the EFI framebuffer at physical address `0xc6200000`. Modifying or disabling that SMR while `INTF_6` is active will immediately trigger an SMMU fault or display corruption. If a driver leaves stream `0x0800` in bypass mode, scanout buffers must be allocated as physically contiguous memory below `4 GiB` (`DMA32`).
3. **Cache Coherency**: The DPU/VBIF read path does not snoop CPU data caches (`_CCA = 0`). Scanout buffers mapped to CPU virtual memory (kernel or userspace) must either use Non-Cacheable / Write-Combining (`WC`) memory attributes or undergo explicit cache clean operations prior to page-flip flush.

---

## 7. Power Domains, Clocks, and Interconnect Overview

**[HW]** Before accessing any register in `MDSS0` (`0x0ae00000`–`0x0aecffff`), the following power and clock prerequisites must be satisfied:
1. **Global Distributed Switch Controller (`GDSC`)**: `MDSS_GDSC` at `DISP_CC_0 + 0x3000` (`0x0af03000`) must be enabled (`SW_COLLAPSE` bit `0` = `0`, `PWR_ON` bit `31` = `1`). On `MDSS0`, a secondary internal power domain `MDSS_INT2_GDSC` exists at `DISP_CC_0 + 0xa000`.
2. **GCC Display AHB & AXI Clocks**: The Global Clock Controller (`GCC` at `0x00100000`) feeds the display subsystem configuration and memory buses (`gcc_disp_ahb_clk`, `gcc_disp_hf_axi_clk`, `gcc_disp_sf_axi_clk`).
3. **DISP_CC_0 Core Clocks**:
   - `disp_cc_mdss_ahb_clk` (`CBCR` at `DISP_CC_0 + 0x2080`, fed by `ahb_clk_src` RCG at `0x2170`) clocks all MDSS/DPU/DP register interfaces.
   - `disp_cc_mdss_mdp_clk` (`CBCR` at `DISP_CC_0 + 0x2010`, fed by `mdp_clk_src` RCG at `0x20c8` from `DISP_CC_PLL0` at `0x0000`) clocks the DPU pixel processing pipeline (up to `575 MHz` at `NOM_L1` / `650 MHz` at `TURBO` on SC8280XP).
   - `disp_cc_mdss_vsync_clk` (`CBCR` at `DISP_CC_0 + 0x2024`, `19.2 MHz` `bi_tcxo`) clocks the DPU tear-check and vsync timers.
   - Per-DP-controller clocks (`dptxN_aux_clk`, `dptxN_link_clk`, `dptxN_link_intf_clk`, `dptxN_pixel0_clk`, `dptxN_usb_router_link_intf_clk`) must be enabled for each active DP port.
