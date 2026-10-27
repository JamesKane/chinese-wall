# Provenance: 10-uapi

- Baseline: d135ab453a7400a34930bdda1ef4bdc8acfac80e
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Sources read:
  - include/uapi/drm/panthor_drm.h (all, 1-1329; MIT-licensed): ioctl IDs, struct layouts, flag values, extension rules
  - drivers/gpu/drm/panthor/panthor_drv.c (all, 1-1953): uobj copy helpers and minimum sizes (41-230), sync op handling and batch submit context (230-770), timestamp query (775-899), priority permission (901-929), DEV_QUERY (931-997), VM create/destroy (999-1034), BO create/mmap offset (1035-1104), GROUP_SUBMIT (1106-1210), group destroy/create/get_state (1212-1261), tiler heap create/destroy (1263-1334), VM_BIND async/sync (1336-1456), VM_GET_STATE, BO label, user MMIO offset, BO sync, BO query info (1458-1590), open/postclose and default MMIO offset (1592-1647), ioctl table (1649-1668), mmap dispatch (1670-1700), fdinfo (1702-1745), version history comment and driver features (1771-1806), sysfs profiling attribute (1837-1870)
  - drivers/gpu/drm/panthor/panthor_mmu.c:1073-1140, 1311-1360, 1413-1470, 1509-1720 (VM create args, user VA default, VM pool limits, destroy), 2714-2770 (exec op, unusable VM), 2975-3105 (bind op validation, async job creation), 3160-3197 (sync bind op), 3377-3380 (MMU_INFO page sizes)
  - drivers/gpu/drm/panthor/panthor_sched.c:1003-1010 (group can-run conditions), 3478-3510 (queue create checks), 3624-3880 (group create/destroy/get_state), 3884-3908 and 3089-3107 (fdinfo gathering), 3955-4040 (job create checks); constants at 94 (job timeout), 140 (priority count), 390 (queue priority max)
  - drivers/gpu/drm/panthor/panthor_fw.h:12-13 (MAX_CSGS, MAX_CS_PER_CSG), 26 (work register decode)
  - drivers/gpu/drm/panthor/panthor_fw.c:217 (unpreserved register count 4), 915-932
  - drivers/gpu/drm/panthor/panthor_hw.c:175-235, 298 (GPU_INFO field sources, shader-present nvmem override)
  - drivers/gpu/drm/panthor/panthor_device.h:116-128 (profiling mask bits)
  - drivers/gpu/drm/panthor/panthor_device.c:399-475 (MMIO mmap)
  - Documentation/gpu/panthor.rst (all; fdinfo keys, sysfs profiling node)
- Ioctl request codes were computed from the _IOWR encoding with type 'd', command base 0x40 and the struct sizes derived from the MIT header.
- No Mesa source was available locally; section 10.12 is marked unverified.
