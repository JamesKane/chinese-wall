# Provenance: realtek-rtl8125-delta/02-init-power-eee

- **Baseline**: `mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e` (`v7.3-16755-gd135ab453a74`)
- **Cross-reference (permissive, not a source of expression)**: `../openbsd-src/sys/dev/pci/if_rge.c` rev 1.43 (ISC) — read only to determine coverage gaps (lines 162–166 PCI IDs, 248–278 HWREV matching, 318–328 capabilities, 704/815/2033/2049 EEE handling)
- **Extracted**: `2026-10-27` by Claude (Opus 5.5) in the dirty room, for James Kane
- **Methodology**: Register offsets, bit positions, magic values, ordering constraints and the firmware file ABI only, restated in tables and fresh prose. No code, comments, function/struct/enum names were carried over; Linux-internal mac-version numbers were replaced by marketing chip names.
- **Sources Read** (`drivers/net/ethernet/realtek/`, all GPL-2.0-only):
  - `r8169_main.c`: 293–341, 501–600 (register/bit names), 1813–1829 (PMCH D3 PLL), 1945–1995 (WoL incl. OCP 0xC0B6, 0xE0C6), 2452–2482 (EEE TX idle timer 0x6048), 2786–2812 (RX config), 2835–2880 (jumbo / readrq / reset), 2976–2987 (RXDV gate), 3010–3025 (TX config, RX max size), 3079–3102 (ring base order), 3176–3256 (CSI, Gen3 0x890 quirk, ASPM latency 0x70F), 3277–3333 (CLKREQ, L2/L3, L1 exit 0xC0AC), 3334–3434 (LTR tables for 8125/8126), 3436–3500 (ASPM/CLKREQ enable/disable), 4207–4311 (8125 common table), 4335–4368 (per-chip start), 4421–4453 (8125 start, moderation table clear, tally), 4486–4525 (overall start order), 5679–5710 (TX LPI enable), 6179–6183, 6312–6327 (hw init), 6346–6368 (jumbo max 16K), 6397–6404, 6862–6875 (ASPM safe flag 0xC0B2), 6581–6600 (EEE speeds).
  - `r8169_phy_config.c`: 104–115 (8125/8126 EEE PHY bits).
