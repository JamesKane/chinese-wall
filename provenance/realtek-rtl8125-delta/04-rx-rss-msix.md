# Provenance: realtek-rtl8125-delta/04-rx-rss-msix

- **Baseline**: `mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e` (`v7.3-16755-gd135ab453a74`)
- **Cross-reference (permissive, not a source of expression)**: `../openbsd-src/sys/dev/pci/if_rge.c` rev 1.43 (ISC) — read only to determine coverage gaps (lines 162–166 PCI IDs, 248–278 HWREV matching, 318–328 capabilities, 704/815/2033/2049 EEE handling)
- **Extracted**: `2026-10-27` by Claude (Opus 5.5) in the dirty room, for James Kane
- **Methodology**: Register offsets, bit positions, magic values, ordering constraints and the firmware file ABI only, restated in tables and fresh prose. No code, comments, function/struct/enum names were carried over; Linux-internal mac-version numbers were replaced by marketing chip names.
- **Sources Read** (`drivers/net/ethernet/realtek/`, all GPL-2.0-only):
  - `r8169_main.c`: 81–96, 470–499, 516–523, 626–643, 690–712, 732–747 (RSS registers, queue registers, vector-map registers, RSS descriptor layout and bits), 1789–1792, 2093–2110, 2882–2902 (RSS enabled on 8127A only, 128-entry table, 8 queues), 3027–3102 (key/RETA/control programming, per-queue ring bases), 4212–4231, 4263–4268 (queue count, descriptor-type bit, vector-mapping enable), 4550–4570 (descriptor return incl. RSS address rewrite), 5279–5370, 5372–5480 (hash validity, checksum and error decoding), 5543–5628 (per-vector mask/ack, vector numbering RX 0–7/TX 8/link 29), 6079–6156 (IRQ mask bits, 30–32 vector requirement, fallback).
