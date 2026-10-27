# Provenance: sky1-dp-tx/00-overview

- Baseline: Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f; Sky1-Linux/linux@665339c3109ba8fbce67aca29e5327e134d6b2b6 (DT); Sky1-Linux/acpi-to-dts-tools@9ddd22c9926d329a2037a800e99545d09f493b51; mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e (ID headers)
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- drivers/gpu/drm/cix/dptx/* is GPL-2.0. Only facts (offsets, bits, values, sequences) were taken; no code or comments copied.
- Sources read (vendor driver: linux-sky1 patches/0005 applied to an empty tree; line numbers refer to the applied files under drivers/gpu/drm/cix/dptx/):
  - trilin_dptx_cix.c 45-414 (DT compatible, ACPI HID CIXH502F, component bind, ACPI graph possible_crtcs, PM ops, shutdown)
  - trilin_dptx.c 3131-3281 (probe: resources by index under ACPI / by name under DT, IRQ 0, all vendor properties and defaults, clock and reset names, SMC GOP get), 3106-3129 (SMC get/set), 3283-3293 (remove calls SMC set), 2292-2318 (dp_phy lookup under ACPI via _DSD reference and parent device), 1774-1805 (vid_clk per stream), 1908-1932 (apb_clk), 906-929 (AUX divider from apb_clk rate, fallback 25)
  - trilin_dptx.h 45-61, 168-187 (max rate, lane count, resource indices, SMC IDs/bit)
  - trilin_phy.h 18-126 (PHY ID register 0xFC, IDs 0x4D/0x54, power states, 24 MHz reference); trilin_usbdp_phy.c 575-595, trilin_edp_phy.c 1165-1185 (PHY type detection)
  - trilin_drm.c 1011-1066 (RCSU adapter), 1130-1184 (connector type, ycbcr420 only with RCSU window), 386-413 (600 MHz single-pixel limit)
  - patches/0022 commit message and dptx hunks (port@2 bridge chain, HDMI-A connector type)
  - patches/0107 dptx hunk (codec numbered by ACPI _UID)
- Devicetree (BSD-3-Clause, (C) Cix Technology): Sky1-Linux/linux@665339c arch/arm64/boot/dts/cix/sky1.dtsi 2836-2985 (dp0-dp4: reg windows and sizes, reg-names, interrupts, clocks, resets incl. phy_reset on dp2, phys, support_d3_cmd, enabled_by_gop, cfg_adapter_port); sky1-orion-o6.dts 60-135 (dptxN_audio cards, i2s5..9, bitclock-inversion), 1100-1235 (dp0-dp4: cix,dp-max-rate 540000, support_d3_cmd "no" on dp1, edp-panel on dp2, ports to DPU pipelines)
- ID headers (GPL-2.0-only OR BSD-2-Clause), mainline d135ab45: include/dt-bindings/clock/cix,sky1.h 58-72 (DPCn VIDCLK0/1, APBCLK IDs); include/dt-bindings/reset/cix,sky1-s5-system-control.h 33-38 (DP_RESETn, DP_PHY); reset word/bit positions from specs/sky1-soc/02 Annex A
- acpi-to-dts-tools (MIT): acpi-to-dts.sh 60, 1487-1495 (CIXH502F mapping, only first MMIO + IRQ emitted); docs/ACPI-to-DTS-Mapping.md 64 (5 instances); examples/generated-o6-from-acpi.dts 640-680 (five instances at dp window addresses, SPI 332-336)
- Cross-references (not re-derived): specs/linlon-dpu/08-cix-sky1.md §8.1, 8.6, 8.8; specs/sky1-soc/01 §1.4, 02 §2.1-2.2
- Not read: any DSDT disassembly; Trilinear datasheet (not available)
