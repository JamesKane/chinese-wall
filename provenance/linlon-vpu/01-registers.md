# Provenance: linlon-vpu/01-registers

- Baseline: vendor sources below
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Driver source: drivers/media/platform/cix/ as created by Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0010-media-cix-Add-Sky1-video-codec-VPU-driver.patch (extracted with git apply into an empty tree; paths and line numbers below are in the extracted files). Later patches touching it: 0127 (Kconfig select lines only; no facts used). 0132 and 0139 were checked and do not touch drivers/media/platform/cix.
- Licence: every driver file carries SPDX GPL-2.0-only; the header text also contains an Arm Technology (China) "confidential and proprietary" notice and an "or (at your option) any later version" GPL paragraph, which contradict each other and the SPDX tag. No file is dual-licensed or MIT. Only facts (offsets, bits, values, layouts, sequences, ABI numbers) were taken; no code or comments were copied or reworded.
- Sources read:
  - dev/mvx_hwreg.c 54-158 (global, per-LSID and RCSU offsets), 160-196 (HARDWARE_ID fields), 198-330 (register-dump names), 550-594
  - dev/mvx_hwreg.h 38-120 (FUSE bits, JOBQUEUE/CORELSID/CTRL field widths, ALLOC values, BUSCTRL ref shift, CLKFORCE fields, RCSU fields, terminate timeout)
  - dev/mvx_lsid.c 52-56 (default BUSATTR values), 58-130 (field helpers), 132-205 (attach sequence), 206-233 (detach), 234-275 (job-queue add), 277-291 (doorbell/soft IRQ/flush), 292-305 (terminate), 306-335 (job-queue remove), 336-359 (idle test)
  - dev/mvx_scheduler.c 150-215 (priority insertion, 0xFF obsoleted entries), 571-600 (CTRL value composition)
  - dev/mvx_dev.c 100-110 (BUSCTRL defaults), 1030-1040 (BUSCTRL write)
