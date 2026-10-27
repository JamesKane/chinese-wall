# Provenance: realtek-rtl8125-delta/05-phy-firmware-leds

- **Baseline**: `mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e` (`v7.3-16755-gd135ab453a74`)
- **Cross-reference (permissive, not a source of expression)**: `../openbsd-src/sys/dev/pci/if_rge.c` rev 1.43 (ISC) — read only to determine coverage gaps (lines 162–166 PCI IDs, 248–278 HWREV matching, 318–328 capabilities, 704/815/2033/2049 EEE handling)
- **Extracted**: `2026-10-27` by Claude (Opus 5.5) in the dirty room, for James Kane
- **Methodology**: Register offsets, bit positions, magic values, ordering constraints and the firmware file ABI only, restated in tables and fresh prose. No code, comments, function/struct/enum names were carried over; Linux-internal mac-version numbers were replaced by marketing chip names.
- **Sources Read** (`drivers/net/ethernet/realtek/`, all GPL-2.0-only):
  - `r8169_firmware.c` lines 1–233 and `r8169_firmware.h` lines 1–39 (container layouts, validation, opcode semantics).
  - `r8169_main.c`: 2674–2699 (post-program PHY reset wait, base restore), 2905–2928 (PHY vs MAC-MCU targets), 1434–1488 (target register mapping), 1095–1135 (LEDSEL registers, mask 0x23F).
  - `r8169_leds.c` lines 1–274 (8125 LED bit assignments, mode validity policy).
  - `r8169_phy_config.c` lines 720–731, 996–999, 1098–1162, 1360–1380 (gen 3 / CP / BP / 8126A PHY tuning order and parameters).
