# Provenance: sky1-audio/00-overview

- Baseline: mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e plus Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f (patches) and Sky1-Linux/linux@665339c3109ba8fbce67aca29e5327e134d6b2b6 (vendor DT)
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Only facts (offsets, bits, IDs, addresses, sequences, table formats) taken; no code or comments copied.
- Sources read:
  - mainline arch/arm64/boot/dts/cix/sky1.dtsi (BSD-3-Clause): 1-20 (includes), 545-605 (mailboxes, audss_cru node)
  - mainline Documentation/devicetree/bindings/soc/cix/cix,sky1-audss-cru.yaml (GPL-2.0-only OR BSD-2-Clause): all
  - mainline sound/hda/controllers/cix-ipbloq.c (GPL-2.0): 22 (DMA offset constant)
  - mainline drivers/clk/cix/clk-sky1-audss.c (GPL-2.0-only): 963-1120 (probe order: NoC reset vs parent clocks; no RCSU writes)
  - Sky1-Linux/linux@665339c3109ba8fbce67aca29e5327e134d6b2b6 arch/arm64/boot/dts/cix/sky1.dtsi (BSD-3-Clause): 596-631 (reserved memory), 736, 1127-1144 (fch_cru, fch_dmac), 1221-1262 (hifi5, audss_dmac), 1264-1443 (i2s0-9), 1445-1465 (hda_bus dma-ranges), 1467-1511 (mailboxes, audss_cru), 1534-1537, 1556-1559, 3199-3210, 3856-3859 (all syscon nodes, for the CIXHA018 count)
  - Sky1-Linux/linux@665339c3109ba8fbce67aca29e5327e134d6b2b6 arch/arm64/boot/dts/cix/sky1-orion-o6.dts (BSD-3-Clause): 15-140 (sound card), 954-1082 (audio node enables, dmas, pin masks, HDA)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches 0062 (CIXHA018 syscon driver: lines 1-105), 0079 (audss_cru _DSD reference: lines 1-27, 140-160, 193-234), 0080 (lines 1-20; SCMI PD skipped under ACPI), 0094 (lines 1-23, 46-128: SError on unpowered audio registers), 0102 (cix,dsp-ctrl reference: lines 100-160), 0107 (lines 1-50 bring-up rationale, 95-130 D0/PPRS and dependency release, 131-270 DMA defer/ACPI DMA, 308-355 hard-coded reset lookups, 356-480 CIXH6011 instance topology), 0114 (lines 1-30, arm,remote-ctrl/_DSD maps), 0134 (lines 1-20, missing I2S on Orange Pi 6 Plus)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches 0083 (lines 1-25, 100-185: RSVL on CIXA1019)
  - Sky1-Linux/acpi-to-dts-tools@9ddd22c9926d329a2037a800e99545d09f493b51 docs/ACPI-to-DTS-Mapping.md (MIT): 55-125 (HID table and counts, noted as unreliable), 215-232 (audio RST0 31 "noc"), 385-400 (address ranges)
  - Cross-referenced (not re-extracted): specs/sky1-soc/01, 02; specs/sky1-dp-tx/02 §2.8, 05 §5.4
