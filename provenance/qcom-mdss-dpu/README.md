# Provenance: `specs/qcom-mdss-dpu/README.md`

- **Baseline Commits**:
  - `mainline`: `d135ab453a7400a34930bdda1ef4bdc8acfac80e` (`/home/jkane/Projects/OS/linux`)
  - `radxa/kernel`: `c00e1e17e65104c8143a8f3d559ef723ae5824d3` (`linux-7.0.11`)
  - `TC956X/TC9564_Host_Driver`: `3eca71ce0c5708c92e20f14b8e858cd4483d58f3`
- **Extracted Date**: `2026-10-27`

## Source Files and Line Ranges Read

1. **`/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/registers/display/mdss.xml` (lines 1–39)**
   - Extracted dual-licensing (`MIT OR BSD-2-Clause`) of the display register XML files (`mdss.xml`, `dpu.xml`, `dsi.xml`, `edp.xml`, `hdmi.xml`) and top-level `MDSS` register offsets (`HW_VERSION` `0x00000`, `HW_INTR_STATUS` `0x00010`, `UBWC_DEC_HW_VERSION` `0x00058`, `UBWC_STATIC` `0x00144`, `UBWC_CTRL_2` `0x00150`, `UBWC_PREDICTION_MODE` `0x00154`).
2. **`/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/msm_mdss.c` (lines 1–450)**
   - Extracted `HW_INTR_STATUS` bit assignments (`MDP` bit 0, `DSI_0` bit 4, `DSI_1` bit 5, `DP_0` bit 12, `DP_1` bit 13, `DP_2` bit 14, `DP_3` bit 15) and MDSS wrapper IRQ domain dispatch behavior.
3. **`/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/disp/dpu1/catalog/dpu_8_0_sc8280xp.h` (lines 1–428)**
   - Extracted SC8280XP DPU 8.0 sub-block counts (`SSPP` VIG0..3 & DMA0..3, `LM_0..5`, `DSPP_0..3`, `CTL_0..5`, `MERGE_3D_0..2`, `PINGPONG_0..5`, `DSC_0..5`, `INTF_0..8`), max clock rate (`650 MHz`), and INTF-to-DP controller pairings.
4. **`/home/jkane/Projects/OS/linux/drivers/clk/qcom/dispcc-sc8280xp.c` (lines 1–3203)**
   - Extracted `DISP_CC_0` (`0x0af00000`) and `DISP_CC_1` (`0x22100000`) physical base addresses, `MDSS_GDSC` (`0x3000`) and `MDSS_INT2_GDSC` (`0xa000`), and core clock offsets (`disp_cc_mdss_ahb_clk` `0x2080`, `disp_cc_mdss_mdp_clk` `0x2010`, `disp_cc_mdss_vsync_clk` `0x2024`).
5. **`/home/jkane/Projects/OS/linux/drivers/phy/qualcomm/phy-qcom-qmp-combo.c` (lines 2679–2692, 2925–2966)**
   - Extracted `qmp_combo_offsets_v5` sub-block offsets (`com` `+0x0000`, `txa` `+0x0400`, `rxa` `+0x0600`, `txb` `+0x0a00`, `rxb` `+0x0c00`, `usb3_serdes` `+0x1000`, `usb3_pcs_misc` `+0x1200`, `usb3_pcs` `+0x1400`, `usb3_pcs_usb` `+0x1700`, `dp_serdes` `+0x2000`, `dp_dp_phy` `+0x2200`) for SC8280XP QMP V5 5nm Combo PHYs (`0x088eb000` and `0x08903000`).
6. **`/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/README.md` (lines 1–336)**
   - Extracted empirical board facts for the Radxa Dragon Q8B (`MDSS0` at `0x0ae00000`, `DP2` -> Chrontel CH7218A HDMI bridge, UEFI framebuffer at `0xc6200000`).
7. **`/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/gpu-display.md` (lines 1–298)**
   - Extracted Dragon Q8B ACPI `GPU0` (`QCOM0636`) memory resource 0 (`0x0ae00000`, size `2 MB`), GSIV `115` (SPI `83`), UEFI boot pipeline (`SSPP_VIG2` -> `LM2` -> `CTL2` -> `INTF6` -> `DP2`), 32-bit SSPP address constraint (`DMA32`), Apps SMMU (`0x15000000`) bypass behavior, and write-combining (`WC`) cache coherency requirement.
8. **`/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/usb.md` (lines 38–108)**
   - Extracted USB-C Port 0 (`0x088eb000` -> `DP0` `0x0ae90000`) and Port 1 (`0x08903000` -> `DP1` `0x0ae98000`) QMP Combo PHY routing and ADSP PMIC GLINK DP Alt-Mode notifications.
