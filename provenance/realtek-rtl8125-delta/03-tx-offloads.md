# Provenance: realtek-rtl8125-delta/03-tx-offloads

- **Baseline**: `mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e` (`v7.3-16755-gd135ab453a74`)
- **Cross-reference (permissive, not a source of expression)**: `../openbsd-src/sys/dev/pci/if_rge.c` rev 1.43 (ISC) — read only to determine coverage gaps (lines 162–166 PCI IDs, 248–278 HWREV matching, 318–328 capabilities, 704/815/2033/2049 EEE handling)
- **Extracted**: `2026-10-27` by Claude (Opus 5.5) in the dirty room, for James Kane
- **Methodology**: Register offsets, bit positions, magic values, ordering constraints and the firmware file ABI only, restated in tables and fresh prose. No code, comments, function/struct/enum names were carried over; Linux-internal mac-version numbers were replaced by marketing chip names.
- **Sources Read** (`drivers/net/ethernet/realtek/`, all GPL-2.0-only):
  - `r8169_main.c`: 645–720 (descriptor bits, GSO limits), 726–730 (TX descriptor layout), 2087–2091 (VLAN tag byte swap), 4846–4877 (descriptor fill, end-of-ring), 4902–5050 (UDP padding quirk 8125A/B, ETH_ZLEN padding, csum/TSO v2 fields, IPv6 TSO header prep), 5058–5067, 5077–5148 (first/last descriptor ordering, doorbell), 5180–5207 (offset limits, short-frame checksum fallback), 6940–6958 (TSO default policy, 8125A exclusion).
