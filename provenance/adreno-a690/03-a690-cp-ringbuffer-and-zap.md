# Provenance: `specs/adreno-a690/03-a690-cp-ringbuffer-and-zap.md`

- **Baseline Commits**:
  - `mainline`: `d135ab453a7400a34930bdda1ef4bdc8acfac80e`
  - `radxa/kernel`: `c00e1e17e65104c8143a8f3d559ef723ae5824d3` (`linux-7.0.11`)
  - `TC956X/TC9564_Host_Driver`: `3eca71ce0c5708c92e20f14b8e858cd4483d58f3`
- **Extracted Date**: `2026-10-27`

## Source Files Read

1. `/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/adreno/a6xx_catalog.c`
   - **Lines 452–505**: Extracted `a690_hwcg` table of 50 RBBM clock gating register pairs (`RBBM_CLOCK_CNTL_SP0` through `RBBM_CLOCK_HYST_FCHE`).
   - **Lines 615–679**: Extracted `a690_protect_regs` array and `a690_protect` structure (`count = 34`, `count_max = 48`, plus final infinite-range entry at index 47).
   - **Lines 1171–1190**: Extracted Adreno 690 GPU catalog entry (`chip_ids = 0x06090000`, `family = ADRENO_6XX_GEN4`, `fw[ADRENO_FW_SQE] = "a660_sqe.fw"`, `fw[ADRENO_FW_GMU] = "a660_gmu.bin"`, `quirks = ADRENO_QUIRK_HAS_CACHED_COHERENT | ADRENO_QUIRK_HAS_HW_APRIV`, `gmem = 4 MiB`, `zap_shader = "a690_zap.mdt"`, `gmu_cgc_mode = 0x00020200`, `prim_fifo_threshold = 0x00800200`, `uche_trap_base = 0x1fffffffff000`).

2. `/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/adreno/a6xx_gpu.c`
   - **Lines 1–350**: Extracted `a6xx_submit` command stream sequence (`CP_SET_MARKER`, `a6xx_set_pagetable` with `CP_SMMU_TABLE_UPDATE`, `PC_CCU_INVALIDATE_*`, `CP_SET_PSEUDO_REG`, `CP_INDIRECT_BUFFER`, `CACHE_FLUSH_TS`, `CP_CONTEXT_SWITCH_YIELD`, and `a6xx_flush`).
   - **Lines 351–650**: Extracted `a6xx_set_hwcg`, `a6xx_set_cp_protect` (`CP_PROTECT_CNTL` and `CP_PROTECT(0..47)`), and `a6xx_calc_ubwc_config` / `a6xx_set_ubwc_config` for UBWC 4.0 and `highest_bank_bit = 16` on SC8280XP.
   - **Lines 651–780**: Extracted `a6xx_zap_shader_init` PAS authentication flow (`PAS ID 13`, `qcom_scm_pas_init_image`, `qcom_mdt_load`, `qcom_scm_pas_mem_setup`, `qcom_scm_pas_auth_and_reset`, and `CP_SET_SECURE_MODE` packet `0x00000000`).
   - **Lines 1220–1605**: Extracted `hw_init` full 3D initialization sequence (GBIF unhalt, 12 `ADDR_MODE_CNTL` registers, UCHE/ROQ/APRIV/QoS/bicubic weights setup, `CP_CHICKEN_DBG = 0x00028801`, `UCHE_CMDQ_CONFIG = 0x90`, `qcom_scm_set_gpu_smmu_aperture(0)`, `CP_SQE_INSTR_BASE`, `CP_RB_*`, and `CP_ME_INIT`).
   - **Lines 1606–2150**: Extracted `a6xx_llc_activate` (`LLCC_GPU = 11`, `LLCC_GPUHTW = 12`, `GBIF_SCACHE_CNTL0/1`) and `RBBM_INT_0_MASK` (`A6XX_INT_MASK = 0x03d0e342`).

3. `/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/adreno/a6xx_gpu.h`
   - **Lines 1–367**: Extracted `a6xx_gpu` state, `shadow_bo` indexing (`shadow_iova + ring->id * 4`), `CP_PROTECT` encoding macros (`A6XX_PROTECT_NORDWR`, `A6XX_PROTECT_RDONLY`), and `a6xx_fenced_write` (`GMU_AHB_FENCE_STATUS`, `FENCE_STATUS_WRITEDROPPED0_MASK` Bit 0, `FENCE_STATUS_WRITEDROPPED1_MASK` Bit 1).

4. `/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/adreno/a6xx_preempt.c` & `a6xx_preempt.h`
   - **Lines 1–403 (`a6xx_preempt.c`), Lines 1–83 (`a6xx_preempt.h`)**: Extracted `a6xx_preempt_record` (`magic = 0xae399d6e`, size `0x418000` bytes) and `a7xx_cp_smmu_info` (`magic = 0x241350d5`, `asid = 0xdecafbad`) layouts, `RB_CONTEXT_SWITCH_GMEM_SAVE_RESTORE_ENABLE`, and `CP_CONTEXT_SWITCH_*` trigger sequence.

5. `/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/adreno/adreno_gpu.c`, `adreno_gpu.h`, `msm_gpu.h`, & `msm_ringbuffer.h`
   - **Lines 1–650 (`adreno_gpu.h`), Lines 1–145 (`msm_ringbuffer.h`)**: Extracted `PKT4`/`PKT7` packet headers, `odd_parity_bit`, `MSM_GPU_RINGBUFFER_SZ` (`32 KiB`), `MSM_GPU_RB_CNTL_DEFAULT` (`0x0000020c`), and `msm_rbmemptrs` layout (`2080` bytes).

6. `/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/registers/adreno/a6xx.xml`, `adreno_pm4.xml`, & `adreno_common.xml`
   - **All lines**: Verified MIT/BSD-2-Clause license headers and exact register offsets (`0x0800..0xf810`) and PM4 Type-7 opcode encodings (`0x10..0x6b`).

7. `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/gpu-display.md`
   - **Lines 142–200**: Verified Radxa Dragon Q8B Adreno 690 empirical GMEM size notes (`1768 KB` across 4 slices vs `4 MiB` catalog entry), `qcom/a660_sqe.fw` firmware name, and `qcom/sc8280xp/LENOVO/21BX/qcdxkmsuc8280.mbn` Zap shader path at `0x8bf00000`.
