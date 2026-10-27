# Provenance: 02-register-map

- Baseline: d135ab453a7400a34930bdda1ef4bdc8acfac80e
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Sources read:
  - drivers/gpu/drm/panthor/panthor_gpu_regs.h (all)
  - drivers/gpu/drm/panthor/panthor_mmu_regs.h (all)
  - drivers/gpu/drm/panthor/panthor_pwr_regs.h (all)
  - drivers/gpu/drm/panthor/panthor_fw_regs.h (all)
  - drivers/gpu/drm/panthor/panthor_device.h:425-570 (exception codes, 32/64-bit access, INT_* offsets)
  - drivers/gpu/drm/panthor/panthor_hw.c (all; product table, arch 10-13 vs 14 split, feature reads)
  - drivers/gpu/drm/panthor/panthor_mmu.c:590-605, 650-665, 785-795, 1858-1880 (fault IRQ bits, INA_BITS)
  - Documentation/devicetree/bindings/gpu/arm,mali-valhall-csf.yaml (RK3588 base address/size)
  - drivers/gpu/drm/panthor/panthor_mmu.c:1896 (AS_FAULTSTATUS bit 10)
