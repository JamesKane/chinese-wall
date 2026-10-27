# Provenance: 08-heap

- Baseline: d135ab453a7400a34930bdda1ef4bdc8acfac80e
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Sources read:
  - drivers/gpu/drm/panthor/panthor_heap.c (all, 1-632): context size and stride, chunk header, initial chain linking, grow and return policy, pool sizing (128 contexts), pool lifetime and accounting
  - drivers/gpu/drm/panthor/panthor_heap.h (all)
  - drivers/gpu/drm/panthor/panthor_sched.c:1495-1650 (tiler OOM worker, counter validation, CS heap register write-back, acknowledge and doorbell, return-chunk path, deferred acknowledge in IRQ path)
  - drivers/gpu/drm/panthor/panthor_sched.c:2878-2890 (firmware writes heap context on CSG suspend; VM teardown ordering)
  - drivers/gpu/drm/panthor/panthor_sched.c:4158-4175 (dedicated allocation workqueue rationale; firmware fallback behavior on allocation failure)
  - drivers/gpu/drm/panthor/panthor_fw.h:36-125 (CS input/output interface layout: heap_start/heap_end at input 0x20/0x28; heap_vt_start/vt_end/frag_end/heap_address at output 0xC0/0xC4/0xCC/0xD0; CS_TILER_OOM bit 26)
  - drivers/gpu/drm/panthor/panthor_mmu.c:1108-1140 (kernel auto-VA alignment 2 MiB / 4 KiB), 1619-1650 (VM destroy destroys heap pool), 2128-2176 (lazy heap pool creation, heap size accounting)
  - drivers/gpu/drm/panthor/panthor_gpu_regs.h:19 (L2 line size decode)
  - drivers/gpu/drm/panthor/panthor_drv.c:1263-1334 (heap create/destroy ioctls, handle encoding)
  - Documentation/gpu/panthor.rst (statement that internal BOs are pinned; used only for the OPEN QUESTION in 8.6)
- Byte offsets in CS interface tables were computed from the structure field order and types; they were not stated as literals in the source.
