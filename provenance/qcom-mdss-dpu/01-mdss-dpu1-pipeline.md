# Provenance: `specs/qcom-mdss-dpu/01-mdss-dpu1-pipeline.md`

- **Baseline Commits**:
  - `mainline`: `d135ab453a7400a34930bdda1ef4bdc8acfac80e` (`/home/jkane/Projects/OS/linux`)
  - `radxa/kernel`: `c00e1e17e65104c8143a8f3d559ef723ae5824d3` (`linux-7.0.11`)
  - `TC956X/TC9564_Host_Driver`: `3eca71ce0c5708c92e20f14b8e858cd4483d58f3`
- **Extracted Date**: `2026-10-27`

## Source Files and Line Ranges Read

1. **`/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/msm_mdss.c` (lines 1–450)**
   - Extracted `msm_mdss_6x_setup_ubwc()` sequence (`REG_MDSS_UBWC_STATIC` at `0x0144`, `REG_MDSS_UBWC_CTRL_2` at `0x0150` = `2`, `REG_MDSS_UBWC_PREDICTION_MODE` at `0x0154` = `1`) and `REG_MDSS_HW_INTR_STATUS` (`0x0010`).
2. **`/home/jkane/Projects/OS/linux/drivers/soc/qcom/ubwc_config.c` (lines 1–237) & `/home/jkane/Projects/OS/linux/include/linux/soc/qcom/ubwc.h` (lines 1–134)**
   - Extracted `sc8280xp_data` UBWC 4.0 parameters (`ubwc_enc_version = UBWC_4_0`, `ubwc_swizzle = 6`, `bank_spread = true`, `highest_bank_bit = 16`, `macrotile_mode = true`) and SMEM DDR_INFO HBB override behavior.
3. **`/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/disp/dpu1/dpu_hw_vbif.c` (lines 1–246) & `dpu_hw_catalog.c` (lines 650–795)**
   - Extracted `VBIF_RT` (`0xb0000`) register offsets (`0x0000`..`0x0590`), `sdm845_vbif` AMEMTYPE configuration (`0x160`, `0x164` = `3` for XINs `0..13`), and 8-level RT/NRT QoS remap tables at `0x0550` and `0x0590`.
4. **`/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/disp/dpu1/catalog/dpu_8_0_sc8280xp.h` (lines 1–428) & `dpu_hw_catalog.c` (lines 1–795)**
   - Extracted SC8280XP DPU 8.0 hardware catalog (`sc8280xp_dpu_caps`, `sc8280xp_mdp`, `sc8280xp_ctl`, `sc8280xp_sspp`, `sc8280xp_lm`, `sc8280xp_dspp`, `sc8280xp_merge_3d`, `sc8280xp_pp`, `sc8280xp_dsc`, `sc8280xp_intf`, `sc8280xp_perf_data`), sub-block offsets, QoS/danger/safe/creq LUT values, and `clk_ctrls` positions.
5. **`/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/disp/dpu1/dpu_hw_top.c` (lines 1–313) & `dpu_hw_top.h` (lines 1–165)**
   - Extracted `MDP_TOP` registers (`SPLIT_DISPLAY_EN` `0x2ec`, `SPLIT_DISPLAY_UPPER_PIPE_CTRL` `0x2f0`, `SPLIT_DISPLAY_LOWER_PIPE_CTRL` `0x304`, `DANGER_STATUS` `0x360`, `SAFE_STATUS` `0x364`, `MDP_DP_PHY_INTF_SEL` `0x460`).
6. **`/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/disp/dpu1/dpu_hw_ctl.c` (lines 1–800)**
   - Extracted `CTL` register offsets (`0x000`..`0x150`), `ctl_blend_config` stage bit positions across `CTL_LAYER`, `CTL_LAYER_EXT`, `CTL_LAYER_EXT2`, `CTL_LAYER_EXT4`, `fetch_tbl` (`CTL_FETCH_PIPE_ACTIVE` `0x0fc`), and `CTL_FLUSH` (`0x018`) bit definitions.
7. **`/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/disp/dpu1/dpu_hw_sspp.c` (lines 1–758) & `dpu_formats.c` (lines 1–365)**
   - Extracted `SSPP` register offsets (`0x000`..`0x180`), `SSPP_SRC_FORMAT` (`0x030`), `SSPP_SRC_UNPACK_PATTERN` (`0x034`), `SSPP_SRC_OP_MODE` (`0x038`), `SSPP_MULTIRECT_OPMODE` (`0x170`), QoS/CDP setup, and UBWC plane address ordering (`SRC0`..`SRC3`).
8. **`/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/disp/dpu1/dpu_hw_util.c` (lines 1–600)**
   - Extracted `QSEED3lite` scaler (`+0xa00`, `0x00`..`0x380`) and 10-bit `CSC` (`+0x1a00`, `0x00`..`0x44`) register maps and coefficient encodings.
9. **`/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/disp/dpu1/dpu_hw_lm.c` (lines 1–300)**
   - Extracted Layer Mixer (`LM`) register offsets (`0x000`..`0x124`), 11 blend stage offsets (`0x20`..`0x110`), and `LM_BLEND0_OP` / `LM_BLEND0_CONST_ALPHA` bitfields.
10. **`/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/disp/dpu1/dpu_hw_merge3d.c` (lines 1–68), `dpu_hw_pingpong.c` (lines 1–333), & `dpu_hw_dsc_1_2.c` (lines 1–397)**
    - Extracted `MERGE_3D` (`0x000`, `0x004`), `PINGPONG` dither (`+0x0e0`..`+0x0f4`), and `DSC 1.2` (`DSC_CMN_MAIN_CNF` `0x00`, `enc` `+0x100`/`+0x200`, `ctl` `+0xf00`/`+0xf80`) register layouts.
11. **`/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/disp/dpu1/dpu_hw_intf.c` (lines 1–600)**
    - Extracted `INTF` register map (`0x000`..`0x2b4`), widebus (`DATABUS_WIDEN` bit 0, `DATA_HCTL_EN` bit 4 in `INTF_CONFIG2` at `0x234`) timing division by 2, DP horizontal timing adjustment, programmable fetch (`0x170`), and `INTF_MUX` (`0x25c`).
12. **`/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/disp/dpu1/dpu_hw_interrupts.c` (lines 1–600) & `dpu_hw_interrupts.h` (lines 1–76)**
    - Extracted `dpu_intr_set_7xxx` register bank offsets and `MDP_SSPP_TOP0_INTR` (`0x010`..`0x018`) VSYNC/underrun bit indices for `INTF_0..8`.
13. **`/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/gpu-display.md` (lines 23–120)**
    - Extracted empirical Dragon Q8B flip and mode-set flush sequence (`CTL_INTF_FLUSH` bit 6, `CTL_FLUSH` bits 2, 8, 17, 31) and `UBWC_STATIC` empirical value (`0x1e`).
