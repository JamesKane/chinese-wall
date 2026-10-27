# Provenance: realtek-rtl8125-delta/01-identification-and-access

- **Baseline**: `mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e` (`v7.3-16755-gd135ab453a74`)
- **Cross-reference (permissive, not a source of expression)**: `../openbsd-src/sys/dev/pci/if_rge.c` rev 1.43 (ISC) — read only to determine coverage gaps (lines 162–166 PCI IDs, 248–278 HWREV matching, 318–328 capabilities, 704/815/2033/2049 EEE handling)
- **Extracted**: `2026-10-27` by Claude (Opus 5.5) in the dirty room, for James Kane
- **Methodology**: Register offsets, bit positions, magic values, ordering constraints and the firmware file ABI only, restated in tables and fresh prose. No code, comments, function/struct/enum names were carried over; Linux-internal mac-version numbers were replaced by marketing chip names.
- **Sources Read** (`drivers/net/ethernet/realtek/`, all GPL-2.0-only):
  - `r8169_main.c`: 140–262 (chip table with XID masks, extended table), 263–289 (PCI IDs), 293–500 (register enums incl. 8125 block, ERIAR/OCPAR/GPHY_OCP), 946–960 (config lock), 1011–1018, 6158–6178, 6370–6395 (MAC address incl. backup at 0x19E0), 1232–1312 (MAC/PHY OCP access and modify), 1434–1490 (paged PHY mapping, MAC MCU base), 1563–1595 (PHY accessor selection), 1850–1900 (8125 IRQ status/mask), 5069–5075 (doorbell), 5481–5515 (removal check), 6184–6232 (C22/C45 mapping), 6840–6862 (XID extraction, extended XID at 0x60B0).
  - `r8169_phy_config.c`: 45–63 (two indexed-parameter mechanisms).
