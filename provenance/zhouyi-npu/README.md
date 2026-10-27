# Provenance: zhouyi-npu/README

- Baseline: mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e (checked only: no Zhouyi/aipu driver, no Sky1 NPU node) plus the Sky1-Linux revisions below
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Only facts (offsets, bits, values, layouts, sequences, error codes) taken; no code or comments copied. Struct offsets were computed by compiling the vendor uapi header in a scratch tool (not distributed).
- Driver tree: Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patch 0009 extracted into an empty git repo (only drivers/misc/armchina-npu/*), with patches 0103, 0132, 0139 applied on top. Paths below are relative to drivers/misc/armchina-npu/ in that tree; line numbers refer to the patched files.
- Licenses (SPDX): all driver .c/.h files GPL-2.0 (Copyright Arm Technology (China) 2023-2024; sky1/sky1.c and include/cix_sky1_soc.h Copyright Cix Technology 2024); Kconfig GPL-2.0-only; include/armchina_aipu.h GPL-2.0 WITH Linux-syscall-note (NOT MIT). Vendor DT Sky1-Linux/linux@665339c3109ba8fbce67aca29e5327e134d6b2b6 sky1.dtsi / sky1-orion-o6.dts: BSD-3-Clause. Mainline include/dt-bindings/clock/cix,sky1.h: GPL-2.0-only OR BSD-2-Clause; arch/arm64/boot/dts/cix/sky1-power.h: GPL-2.0-only (numeric IDs only). Sky1-Linux/acpi-to-dts-tools@9ddd22c9926d329a2037a800e99545d09f493b51: MIT.
- Sources read:
  - sky1/sky1.c 545-549 (ACPI match table: CIXH4000 only), 525-540 (OF compatibles)
  - grep of all Sky1-Linux patches for CIXH IDs (no CIXH4010, no CIXHA002 match anywhere)
  - acpi-to-dts-tools: README.md 121-133 (HID table, CIXH4000 listed as GPU), docs/ACPI-to-DTS-Mapping.md 78-86, acpi-to-dts.sh 80, 120-127 (CIXH4010 -> cix,sky1-isp; CIXHA002 -> cix,sky1-aipu), 190-205 (address table 14260000 aipu, 14340000 isp), 1542-1558 (generator emitting the CIXH4000 node as a Mali GPU), examples/generated-o6-from-acpi.dts 693-698
  - vendor DT sky1.dtsi 3015-3022 (armcb ISP at 0x14340000)
  - linux-sky1 config/config.sky1 (ARMCHINA_NPU=m, ARCH_V3=y only, SOC_SKY1=y), config/config-policy.ini 90-92
  - linux-sky1 patches/0013 line 60 (30 TOPS claim)
