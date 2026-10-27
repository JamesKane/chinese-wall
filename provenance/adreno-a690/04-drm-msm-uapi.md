# Provenance: `specs/adreno-a690/04-drm-msm-uapi.md`

- **Baseline Commits**:
  - `mainline`: `d135ab453a7400a34930bdda1ef4bdc8acfac80e`
  - `radxa/kernel`: `c00e1e17e65104c8143a8f3d559ef723ae5824d3` (`linux-7.0.11`)
  - `TC956X/TC9564_Host_Driver`: `3eca71ce0c5708c92e20f14b8e858cd4483d58f3`
- **Extracted Date**: `2026-10-27`

## Source Files Read

1. `/home/jkane/Projects/OS/linux/include/uapi/drm/msm_drm.h`
   - **Lines 1–578**: Extracted all userspace ABI constants, `MSM_PIPE_*` identifiers, `MSM_PARAM_*` parameter IDs (`0x01..0x17`), `MSM_BO_*` buffer allocation flags, `MSM_INFO_*` gem info opcodes (`0x00..0x05`), `MSM_PREP_*` flags, `MSM_SUBMIT_*` command/BO/syncobj flags, `MSM_VM_BIND_*` opcodes, ioctl numbers (`0x40..0x4e`), and exact byte layouts of all ioctl structures.

2. `/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/msm_drv.c`
   - **Lines 900–1060**: Extracted DRM driver identification (`name = "msm"`, `desc = "MSM Snapdragon"`, version `1.13.0`), `DRIVER_GEM | DRIVER_RENDER | DRIVER_SYNCOBJ | DRIVER_SYNCOBJ_TIMELINE | DRIVER_GPUVA` feature flags, and ioctl dispatch table (`msm_ioctls`).

3. `/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/adreno/adreno_gpu.c`
   - **Lines 365–533**: Extracted `adreno_get_param` and `adreno_set_param` implementations for `MSM_PARAM_GPU_ID`, `MSM_PARAM_GMEM_SIZE`, `MSM_PARAM_CHIP_ID` (`chip_id | (speedbin << 32)`), `MSM_PARAM_MAX_FREQ`, `MSM_PARAM_TIMESTAMP`, `MSM_PARAM_GMEM_BASE` (`0` on `ADRENO_6XX_GEN3`+), `MSM_PARAM_PRIORITIES`, `MSM_PARAM_PP_PGTABLE`, `MSM_PARAM_FAULTS`, `MSM_PARAM_SUSPENDS`, `MSM_PARAM_VA_START`, `MSM_PARAM_VA_SIZE`, `MSM_PARAM_HIGHEST_BANK_BIT`, `MSM_PARAM_RAYTRACING`, `MSM_PARAM_UBWC_SWIZZLE`, `MSM_PARAM_MACROTILE_MODE`, `MSM_PARAM_UCHE_TRAP_BASE`, `MSM_PARAM_HAS_PRR`, `MSM_PARAM_EN_VM_BIND`, and `MSM_PARAM_AQE`.

4. `/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/adreno/a6xx_gpu.c`
   - **Lines 2650–2720**: Extracted `ADRENO_VM_START = 0x100000000ULL` (`4 GiB`) and 48-bit SMMU user address space calculation (`va_size = (1ULL << 48) - ADRENO_VM_START`).

5. `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/gpu-display.md`
   - **Lines 142–200**: Verified Mesa Turnip (`libvulkan_freedreno.so`, `TU_DEBUG`) and Freedreno Gallium runtime verification expectations on Radxa Dragon Q8B (`chip-id = 0x06090000`, `patchid = 0x06090001`).
