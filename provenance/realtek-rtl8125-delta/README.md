# Provenance: realtek-rtl8125-delta/README

- **Baseline**: `mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e` (`v7.3-16755-gd135ab453a74`)
- **Cross-reference (permissive, not a source of expression)**: `../openbsd-src/sys/dev/pci/if_rge.c` rev 1.43 (ISC) — read only to determine coverage gaps (lines 162–166 PCI IDs, 248–278 HWREV matching, 318–328 capabilities, 704/815/2033/2049 EEE handling)
- **Extracted**: `2026-10-27` by Claude (Opus 5.5) in the dirty room, for James Kane
- **Methodology**: Register offsets, bit positions, magic values, ordering constraints and the firmware file ABI only, restated in tables and fresh prose. No code, comments, function/struct/enum names were carried over; Linux-internal mac-version numbers were replaced by marketing chip names.
- **Sources Read** (`drivers/net/ethernet/realtek/`, all GPL-2.0-only):
  - `r8169.h` lines 60–80 (mac-version list); `r8169_main.c` lines 50–175 (chip table, firmware names), 1004–1010, 1789–1792 (RSS only on 8127A), 6093–6118, 6920–6960 (feature defaults), 6417–6442 (flow-control note).
  - Git history `90d0cfb577db`, `677f8f8c4c5f`, `7c543c5d12b2`, `5df447aee076`, `36af5aac9545` (RSS / multi-IRQ series) — titles only.
