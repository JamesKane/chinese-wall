# Provenance: linlon-vpu/README

- Baseline: no mainline code (mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e was searched with git grep for linlon-v*, mali-v5*, mali-v6*, mali-mve, amvx: no driver, binding or DT node exists). Public mainline header used: include/dt-bindings/clock/cix,sky1.h (GPL-2.0-only OR BSD-2-Clause) line 76 (CLK_TREE_VPU_APBCLK = 67).
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Driver source: drivers/media/platform/cix/ as created by Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0010-media-cix-Add-Sky1-video-codec-VPU-driver.patch (extracted with git apply into an empty tree; paths and line numbers below are in the extracted files). Later patches touching it: 0127 (Kconfig select lines only; no facts used). 0132 and 0139 were checked and do not touch drivers/media/platform/cix.
- Licence: every driver file carries SPDX GPL-2.0-only; the header text also contains an Arm Technology (China) "confidential and proprietary" notice and an "or (at your option) any later version" GPL paragraph, which contradict each other and the SPDX tag. No file is dual-licensed or MIT. Only facts (offsets, bits, values, layouts, sequences, ABI numbers) were taken; no code or comments were copied or reworded.
- Sources read:
  - Summary of 00-04; family naming from dev/mvx_hwreg.c 160-196, if/mvx_if.h 150-180, Kconfig
