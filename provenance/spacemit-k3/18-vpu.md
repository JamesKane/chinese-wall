# Provenance: spacemit-k3/18-vpu

- Baseline: spacemit-com/linux-6.18@7d17e358c660f1046eb1f27f26eea33955813afd (vendor, secondary)
- Stub created 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Sources read so far:
  - arch/riscv/boot/dts/spacemit/k3.dtsi 2629-2639 (GPL-2.0 OR MIT): node, base, interrupt, clock/reset/power-domain references
  - drivers/media/platform/spacemit/vpu_k3/dev/mvx_dev.c 1-12: license header only (GPL-2.0-only, Arm China copyright); dirty room only
- To read when the section is written: drivers/media/platform/spacemit/vpu_k3/ (whole tree), diffed against the Sky1 sources listed in provenance/linlon-vpu/
- Gap research pass, 2026-10-27 (research subagent, spot-checked by the main session):
  - Vendor drivers/media/platform/spacemit/vpu_k3/dev/mvx_hwreg.c 150-165 (accepted products), dev/mvx_lsid.h 45 (LSID maximum), if/mvx_firmware_cache.c 111-153 (codec map, firmware path pattern, no AV1), if/mvx_firmware.c 477-485 (protocol major check). GPL-2.0-only.
  - github.com/spacemit-com/k3x-vpu-firmware, branch k3-br-v1.0.y: directory listing of lib/firmware/linlon-v52_v76-80-2/; h264dec.fwb first 16 bytes and build string (inspected, not redistributed). debian/copyright states no license.
  - Compared with specs/linlon-vpu/00-overview.md §0.1.1, §0.9.
