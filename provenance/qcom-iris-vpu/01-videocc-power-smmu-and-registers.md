# Provenance: `specs/qcom-iris-vpu/01-videocc-power-smmu-and-registers.md`

## Baseline Commits and Empirical Sources
- **Mainline Linux:** `d135ab453a7400a34930bdda1ef4bdc8acfac80e` (`/home/jkane/Projects/OS/linux`)
- **Radxa Linux Kernel (`linux-7.0.11`):** `radxa/kernel@c00e1e17e65104c8143a8f3d559ef723ae5824d3` (`/home/jkane/.gemini/antigravity-cli/brain/2761ccec-ec33-4e2c-9581-a942f4d0f19e/scratch/radxa-kernel`)
- **Empirical Board Documentation (AbyssBSD):** `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/`
- **Extraction Date:** `2026-10-27`

## Source Files Read
1. `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/vpu.md` (lines 1–194) and `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/lessons.md` (lines 1–134)
   - Extracted the `MMCX` unclocked register read immediate SoC reset hazard, RPMh ARC addresses `mx.lvl = 0x3001c` and `mmcx.lvl = 0x30020`, `GCC_VIDEO_AXI0_CLK` at `GCC + 0x28010`, and empirical 533 MHz bring-up sequence (`L = 0x53`, `ALPHA = 0x4800`).
2. `/home/jkane/Projects/OS/linux/drivers/clk/qcom/videocc-sm8350.c` (lines 1–589)
   - Extracted `video_pll0` (`0x42c`) and `video_pll1` (`0x7d0`) Lucid 5LPE PLL configurations, `ftbl_video_cc_mvs0_clk_src_sc8280xp` frequency table (`720, 1014, 1098, 1332, 1599, 1680 MHz`), RCG offsets (`0xb94`, `0xbd4`, and SC8280XP overrides `0xf14` XO, `0xf38` Sleep), divider offsets (`0xc54`, `0xd54`), CBCR/BCR offsets (`0xbf4`, `0xc34` with 400 µs reset delay, `0xd14`, `0xd34`, `0xe54`, `0xe58`, `0xf34`, `0xf58`), and GDSC definitions (`mvs0c_gdsc` at `0xbf8`, `mvs0_gdsc` at `0xd18`).
3. `/home/jkane/Projects/OS/linux/drivers/clk/qcom/clk-alpha-pll.h` (lines 1–265) and `/home/jkane/Projects/OS/linux/drivers/clk/qcom/clk-alpha-pll.c` (lines 100–2200)
   - Extracted `CLK_ALPHA_PLL_TYPE_TRION` / `LUCID` register offsets (`+0x00` to `+0x40`) and `clk_alpha_pll_lucid_5lpe_ops` / `clk_trion_pll_configure` bit definitions and lock/latch polling sequences.
4. `/home/jkane/Projects/OS/linux/drivers/clk/qcom/gdsc.c` (lines 1–450)
   - Extracted GDSC register bits (`SW_COLLAPSE`, `HW_CONTROL`, `RETAIN_FF_ENABLE`, `PWR_ON` / `CFG_GDSCR` polling).
5. `/home/jkane/Projects/OS/linux/drivers/firmware/qcom/qcom_scm.c` (lines 1200–1600)
   - Extracted `qcom_scm_pas_init_image`, `qcom_scm_pas_mem_setup`, `qcom_scm_pas_auth_and_reset`, and `qcom_scm_mem_protect_video_var` (`QCOM_SCM_SVC_MP = 0x0c`, `QCOM_SCM_MP_VIDEO_VAR = 0x08`).
6. `/home/jkane/.gemini/antigravity-cli/brain/2761ccec-ec33-4e2c-9581-a942f4d0f19e/scratch/radxa-kernel/drivers/media/platform/qcom/iris/iris_vpu_register_defines.h` (lines 1–79), `iris_vpu_common.c` (lines 1–508), `iris_vpu2.c` (lines 1–49), `iris_firmware.c` (lines 1–338), `iris_platform_vpu2.c` (lines 1–168)
   - Extracted complete VPU 2.0 MMIO register offsets, bit masks, `CP_START = 0`, `CP_SIZE = 0x25800000`, `CP_NONPIXEL_START = 0x01000000`, `CP_NONPIXEL_SIZE = 0x24800000`, DMA mask `0xe0000000 - 1`, and exact boot, hardware mode switch, watchdog check, and power-off sequences.
