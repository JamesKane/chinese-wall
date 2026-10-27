# Provenance: zhouyi-npu/02-operation

- Baseline: mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e (checked only: no Zhouyi/aipu driver, no Sky1 NPU node) plus the Sky1-Linux revisions below
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Only facts (offsets, bits, values, layouts, sequences, error codes) taken; no code or comments copied. Struct offsets were computed by compiling the vendor uapi header in a scratch tool (not distributed).
- Driver tree: Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patch 0009 extracted into an empty git repo (only drivers/misc/armchina-npu/*), with patches 0103, 0132, 0139 applied on top. Paths below are relative to drivers/misc/armchina-npu/ in that tree; line numbers refer to the patched files.
- Licenses (SPDX): all driver .c/.h files GPL-2.0 (Copyright Arm Technology (China) 2023-2024; sky1/sky1.c and include/cix_sky1_soc.h Copyright Cix Technology 2024); Kconfig GPL-2.0-only; include/armchina_aipu.h GPL-2.0 WITH Linux-syscall-note (NOT MIT). Vendor DT Sky1-Linux/linux@665339c3109ba8fbce67aca29e5327e134d6b2b6 sky1.dtsi / sky1-orion-o6.dts: BSD-3-Clause. Mainline include/dt-bindings/clock/cix,sky1.h: GPL-2.0-only OR BSD-2-Clause; arch/arm64/boot/dts/cix/sky1-power.h: GPL-2.0-only (numeric IDs only). Sky1-Linux/acpi-to-dts-tools@9ddd22c9926d329a2037a800e99545d09f493b51: MIT.
- Sources read:
  - config.h 9-16 (50 us reset delay, max 32 reserved regions)
  - zhouyi/v3_priv.c 78-217 (probe order: reg/irq, global soft reset, build info, presence, per-partition init)
  - zhouyi/v3.c 14-523 entire (init, command pool create/destroy/abort/dispatch, interrupt handling, soft reset, GM size)
  - aipu_tcb.h 1-139 entire, aipu_tcb.c 7-63 (TCB layout, flag bits, group = 4 TCBs, ASID control bits)
  - aipu_job_manager.c 253-317 (v3 validation), 365-393 (TEC interrupt enabling), 433-522 (v3 scheduling, queues, single-group init-TCB skip, linking), 524-616 (job creation, hold TCB allocation, failure state), 618-657 (deferred trigger), 815-838 (printf signal), 951-1010 (job end/abnormal tests), 1018-1160 (upper half: signals, abort, coredump), 1162-1275 (bottom half: unlink, destroy, wake), 1277-1385 (abort, cancel on close), 1607-1689 (config clusters, suspend, resume)
  - aipu_mm.c 91-267 (hold TCB buffer: size 128+1024, stub bytes at offset 256, field edits, linking), 298-325, 1982-2097 (link/unlink), 867-991 (IOVA reservation of top 1 GiB per 4 GiB, 32 GiB limit, region count by RAM size), 993-1168 (reserved regions, ASID base calculation), 1170-1296 (init), 1383-1431 (4 GiB / top-1 GiB validity check), 1618-1732 (cache maintenance over whole region), 1764-1804 (uncached mmap), 2115-2196 (ASID base/size, GM sizes)
  - aipu.c 31-58 (runtime PM on open/close), 395-440 (suspend/resume)
  - sky1/sky1.c 347-523 (probe power-up order, runtime suspend/resume of CRE children)
