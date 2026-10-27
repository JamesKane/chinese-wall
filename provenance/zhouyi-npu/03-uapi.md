# Provenance: zhouyi-npu/03-uapi

- Baseline: mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e (checked only: no Zhouyi/aipu driver, no Sky1 NPU node) plus the Sky1-Linux revisions below
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Only facts (offsets, bits, values, layouts, sequences, error codes) taken; no code or comments copied. Struct offsets were computed by compiling the vendor uapi header in a scratch tool (not distributed).
- Driver tree: Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patch 0009 extracted into an empty git repo (only drivers/misc/armchina-npu/*), with patches 0103, 0132, 0139 applied on top. Paths below are relative to drivers/misc/armchina-npu/ in that tree; line numbers refer to the patched files.
- Licenses (SPDX): all driver .c/.h files GPL-2.0 (Copyright Arm Technology (China) 2023-2024; sky1/sky1.c and include/cix_sky1_soc.h Copyright Cix Technology 2024); Kconfig GPL-2.0-only; include/armchina_aipu.h GPL-2.0 WITH Linux-syscall-note (NOT MIT). Vendor DT Sky1-Linux/linux@665339c3109ba8fbce67aca29e5327e134d6b2b6 sky1.dtsi / sky1-orion-o6.dts: BSD-3-Clause. Mainline include/dt-bindings/clock/cix,sky1.h: GPL-2.0-only OR BSD-2-Clause; arch/arm64/boot/dts/cix/sky1-power.h: GPL-2.0-only (numeric IDs only). Sky1-Linux/acpi-to-dts-tools@9ddd22c9926d329a2037a800e99545d09f493b51: MIT.
- Sources read:
  - include/armchina_aipu.h 1-699 entire (GPL-2.0 WITH Linux-syscall-note): enums, struct layouts, ioctl definitions. Offsets, sizes and request codes computed by compiling the header with gcc on x86-64 LP64 (same layout rules as AArch64 for these types); request-code values printed by the scratch tool.
  - aipu.c 31-311 (open/release, ioctl dispatch incl. KILL_TIMEOUT_JOB 8-byte read, GET_DRIVER_VERSION/ALLOC_GRID_ID copy behaviour, compat path, mmap, poll), Makefile 6 (KMD_VERSION "1.0.0")
  - aipu_priv.c 13-20 (misc device name/mode), 181-288 (capability filling, REQ_IO dispatch, open check)
  - zhouyi/v3.c 277-297 (REQ_IO bound), zhouyi/v3_priv.c 13-76 (cluster capability source)
  - aipu_job_manager.c 253-317, 524-657, 783-813 (SCHEDULE_JOB semantics and errors), 1277-1307 (ABORT), 1394-1420 (KILL), 1430-1510 (QUERY_STATUS), 1521-1554 (poll), 1556-1577 (HW status), 1579-1605 (tick counter refcount), 1607-1655 (CONFIG_CLUSTERS), 1691-1747 (grid/group IDs), 667-723 (grid/group init only for v3_1)
  - aipu_mm.c 1433-1551 (REQ_BUF validation and fallback), 634-721 (allocation result fields), 723-782, 1592-1616 (free, deferred free), 1674-1732 (cache ioctls), 1764-1804 (mmap), 1806-1896 (SRAM enable/disable), 2137-2196 (ASID/GM reporting), 607-621 (unset ASID base = U64_MAX), 812-853 (gm_policy sysfs)
  - aipu_dma_buf.c 1-332 entire (export, mmap, info, attach first segment, detach)
  - aipu_common.c 14-110 (ext_registers sysfs)
  - linux-sky1 config/config.sky1 (CONFIG_COMPAT=y, 4K pages)
