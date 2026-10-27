# Provenance: zhouyi-npu/01-registers

- Baseline: mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e (checked only: no Zhouyi/aipu driver, no Sky1 NPU node) plus the Sky1-Linux revisions below
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Only facts (offsets, bits, values, layouts, sequences, error codes) taken; no code or comments copied. Struct offsets were computed by compiling the vendor uapi header in a scratch tool (not distributed).
- Driver tree: Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patch 0009 extracted into an empty git repo (only drivers/misc/armchina-npu/*), with patches 0103, 0132, 0139 applied on top. Paths below are relative to drivers/misc/armchina-npu/ in that tree; line numbers refer to the patched files.
- Licenses (SPDX): all driver .c/.h files GPL-2.0 (Copyright Arm Technology (China) 2023-2024; sky1/sky1.c and include/cix_sky1_soc.h Copyright Cix Technology 2024); Kconfig GPL-2.0-only; include/armchina_aipu.h GPL-2.0 WITH Linux-syscall-note (NOT MIT). Vendor DT Sky1-Linux/linux@665339c3109ba8fbce67aca29e5327e134d6b2b6 sky1.dtsi / sky1-orion-o6.dts: BSD-3-Clause. Mainline include/dt-bindings/clock/cix,sky1.h: GPL-2.0-only OR BSD-2-Clause; arch/arm64/boot/dts/cix/sky1-power.h: GPL-2.0-only (numeric IDs only). Sky1-Linux/acpi-to-dts-tools@9ddd22c9926d329a2037a800e99545d09f493b51: MIT.
- Sources read:
  - zhouyi/v3.h 1-427 entire (cluster, TSM, command-pool, debug register offsets and bit fields)
  - zhouyi/zhouyi.h 13-56 (revision IDs, soft-reset bits, QoS values), 59-80 (v1/v2 host control map, noted as not applicable)
  - zhouyi/zhouyi.c 107-137 (revision to ISA mapping), 164-174 (soft reset readback value)
  - zhouyi/v3.c 14-68 (cluster control and interrupt-enable values), 75-179 (pool command sequences and fail-bit clear), 248-251 (idle test), 277-297 (REQ_IO bound, tick counter values), 299-309 (init), 311-376 (interrupt register reads and W1C acknowledges), 489-494 (GM size code)
  - zhouyi/v3_priv.c 44-61 (debug page select around GM control read), 147-152 (build info use)
  - aipu_io.c 57-82 (32-bit access, bounds)
  - sky1/sky1.c entire (confirmed no MMIO access in Sky1 layer)
  - vendor DT sky1.dtsi (grep for rcsu nodes: no NPU RCSU)
