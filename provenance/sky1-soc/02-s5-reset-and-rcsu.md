# Provenance: sky1-soc/02-s5-reset-and-rcsu

- Baseline: mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e plus the Sky1-Linux revisions below
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Only facts (offsets, bits, IDs, sequences, table formats) taken; no code or comments copied.
- Sources read:
  - mainline drivers/reset/reset-sky1.c (GPL-2.0-only; offsets/bits/polarity/delays only): 20-62, 206-260, 266-351
  - mainline include/dt-bindings/reset/cix,sky1-s5-system-control.h and cix,sky1-system-control.h (GPL-2.0-only OR BSD-2-Clause): all (indices; Annex A generated mechanically by joining these with the driver tables)
  - mainline arch/arm64/boot/dts/cix/sky1.dtsi: 510-513, 765-768 (controller sizes)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0003: vendor reset-sky1.c (identical tables; CIXHA020/21 match; MMIO regmap; delays), clk-sky1-audss.c 160-171, 720-735, 815-835 (audio RCSU remap/timeout, NoC reset pulse)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0077 (RSTL/RSNL/DLKL formats, CIXA1019)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0006 210-335, 480-500 (GPU RCSU PGCTRL 0x218, harvesting 0x304, power-up order); 0096 600-615
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0010 1295-1331, 1794-1802, 2200-2212, 2335-2362 (VPU RCSU PGCTRL/STRAP_PIN0/STRAP_PIN2 fields)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0002 1487-1489, 4272-4276 (PCIe RCSU strap/status offsets); 0004 1019 (USB RCSU reset default)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0107 commit message and 90-110 (PPRS power resource, RCSU PD_CTRL, DMRP memory repair)
  - tree/drivers/gpu/drm/cix/dptx/trilin_dptx.h 175, trilin_dptx.c 3206-3213 (dp_rcsu window index only)
  - Sky1-Linux/acpi-to-dts-tools@9ddd22c9926d329a2037a800e99545d09f493b51 docs/ACPI-to-DTS-Mapping.md (MIT): RSTL section, reset controller table, examples
