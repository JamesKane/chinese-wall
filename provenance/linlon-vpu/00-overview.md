# Provenance: linlon-vpu/00-overview

- Baseline: vendor sources below; mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e for public binding IDs only
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Driver source: drivers/media/platform/cix/ as created by Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0010-media-cix-Add-Sky1-video-codec-VPU-driver.patch (extracted with git apply into an empty tree; paths and line numbers below are in the extracted files). Later patches touching it: 0127 (Kconfig select lines only; no facts used). 0132 and 0139 were checked and do not touch drivers/media/platform/cix.
- Licence: every driver file carries SPDX GPL-2.0-only; the header text also contains an Arm Technology (China) "confidential and proprietary" notice and an "or (at your option) any later version" GPL paragraph, which contradict each other and the SPDX tag. No file is dual-licensed or MIT. Only facts (offsets, bits, values, layouts, sequences, ABI numbers) were taken; no code or comments were copied or reworded.
- Sources read:
  - dev/mvx_dev.c 30-200 (names, module params, power-domain names, DVFS table), 200-460 (format fuse filter, load/frequency selection), 461-500 (IRQ top half), 500-735 (devfreq/perf-domain attach, governor tuning), 740-1000 (probe: resources by index, clock/reset names, shared IRQ, ACPI CRE* child scan, REPR method, autosuspend 1000 ms, core-mask check), 1041-1086 (ENPWOFF/CLKFORCE/PGCTRL), 1088-1230 (runtime suspend/resume order, 10-20 us reset pulse), 1230-1300 (OF/ACPI/PCI match tables, CIXH3010)
  - dev/mvx_hwreg.c 40-50 (hw_ncores), 160-196 (HARDWARE_ID decode), 389-410, 550-594 (core mask from STRAP_PIN2)
  - dev/mvx_hwreg.h 30-160 (fuse bits, RCSU field positions)
  - dev/mvx_scheduler.c 924-1110 (suspend wait 1000 ms, session load calculation multipliers)
  - if/mvx_if.c 60-150 (40-bit DMA mask, 2 GiB segment size); if/mvx_if.h 150-180 (SVN magic values)
  - if/mvx_firmware_cache.c 112-170 (firmware file names), 440-500 (async loading)
  - if/mvx_session.c 6636-6700 (4K variant limit, referenced)
  - Vendor DT Sky1-Linux/linux@665339c3109ba8fbce67aca29e5327e134d6b2b6 arch/arm64/boot/dts/cix/sky1.dtsi (BSD-3-Clause): 570-595 (vpu_private/vpu_protected), 2977-2998 (vpu node); sky1-orion-o6.dts 2298-2300 (status okay)
  - Mainline include/dt-bindings/clock/cix,sky1.h (GPL-2.0-only OR BSD-2-Clause) 76; include/dt-bindings/reset/cix,sky1-s5-system-control.h (GPL-2.0-only OR BSD-2-Clause) 23, 161
  - Sky1-Linux/acpi-to-dts-tools@9ddd22c9926d329a2037a800e99545d09f493b51 (MIT): docs/ACPI-to-DTS-Mapping.md 78-83, 397; acpi-to-dts.sh 82-83, 199 (CIXH3010 -> cix,sky1-vpu, 0x14230000)
  - Cross-references only: specs/sky1-soc/01-mailbox-and-scmi.md, specs/sky1-soc/02-s5-reset-and-rcsu.md
