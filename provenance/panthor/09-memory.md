# Provenance: 09-memory

- Baseline: d135ab453a7400a34930bdda1ef4bdc8acfac80e
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Sources read:
  - drivers/gpu/drm/panthor/panthor_gem.c (all, 1-1768): THP mount, WC vs cached selection (89-117), reclaim states and LRU handling (119-205), backing/pin/dev-map/vmap (207-450), free path, dma-buf ops and begin/end CPU access (450-575), export restriction (563-581), self-import (584-598), mmap and fault handling (742-960), object creation and non-movable allocation (1003-1044), sg-table import (1046-1070), create-with-handle, labels (1072-1140), BO sync (1145-1240), kernel BO create/destroy (1242-1360), dummy BO (1364-1373), shrinker count/evict/scan (1375-1580), debugfs (1582-1768)
  - drivers/gpu/drm/panthor/panthor_gem.h (all): label max length, reclaim state definitions, kernel BO structure
  - drivers/gpu/drm/panthor/panthor_drv.c:1035-1104 (BO create and mmap-offset ioctls), 1478-1590 (label, user MMIO offset, BO sync, BO query info), 1670-1700 (mmap dispatch by offset), 1702-1745 (fdinfo), 1808-1814 (transparent_hugepage module parameter)
  - drivers/gpu/drm/panthor/panthor_device.c:190-212 (dummy flush-ID page value 1), 399-475 (MMIO mmap constraints and fault-time backing selection), 535-590 (MMIO mapping teardown on resume/suspend)
  - drivers/gpu/drm/panthor/panthor_gpu.c:485-510 (coherency detection and ACE-Lite requirement)
  - drivers/gpu/drm/panthor/panthor_mmu.c:1073-1090 (bind flags to protection), 1318-1410 (map op pins temporarily, exclusive-VM check), 2178-2218 (MAIR to MEMATTR shareability rule for coherent/non-coherent), 2466-2700 (eviction of GPU mappings, restore on validate), 3263-3352 (submit-time validate, per-VM reclaim LRU)
  - drivers/gpu/drm/panthor/panthor_fw.c:490-540 (queue interface 8 KiB, suspend buffers), 640-680 (firmware section BOs)
  - drivers/gpu/drm/panthor/panthor_sched.c:330-345 (group syncobj layout), 713-723 (profiling slot layout), 3496-3575 (ring buffer and profiling BOs), 3660-3725 (group syncobj BO, suspend buffers), 3436-3461 (profiling slot count)
  - include/uapi/drm/panthor_drm.h:48-73 (user MMIO offsets), 761-812 (BO flags, exclusive VM), 1178-1271 (BO sync, query info)
  - Documentation/gpu/panthor.rst (fdinfo memory keys)
