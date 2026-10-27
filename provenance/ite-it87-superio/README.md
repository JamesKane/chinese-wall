# Provenance: ite-it87-superio/README

- **Baseline**: `mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e` (`v7.3-16755-gd135ab453a74`)
- **Extracted**: `2026-10-27` by Claude (Opus 5.5) in the dirty room, for James Kane
- **Methodology**: Functional facts only (port numbers, key bytes, register offsets, bit positions, conversion constants, sequences). Every statement was rewritten in fresh prose. No code, comments or identifiers were copied. Driver-internal feature-flag names and structure names were translated into plain capability statements.
- **Sources Read**:
  - `drivers/hwmon/it87.c` (GPL-2.0-or-later), whole file, lines 1–4064: overview, chip list, per-chip capability table.
  - `drivers/platform/x86/gigabyte-wmi.c` (GPL-2.0-or-later), lines 1–164.
  - IT8696E absence confirmed by searching for `8696` across `drivers/` (no match in it87.c).
