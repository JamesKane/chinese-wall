# Provenance: ite-it87-superio/04-gigabyte-wmi-temperature

- **Baseline**: `mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e` (`v7.3-16755-gd135ab453a74`)
- **Extracted**: `2026-10-27` by Claude (Opus 5.5) in the dirty room, for James Kane
- **Methodology**: Functional facts only (port numbers, key bytes, register offsets, bit positions, conversion constants, sequences). Every statement was rewritten in fresh prose. No code, comments or identifiers were copied. Driver-internal feature-flag names and structure names were translated into plain capability statements.
- **Sources Read**:
  - `drivers/platform/x86/gigabyte-wmi.c`, lines 1–164: GUID, command codes (`0x1`, `0x2`, `0x4`, `0x5`, `0x125`), 32-bit argument, method instance 0, signed 8-bit result, zero = absent, six-index discovery.
  - ACPI WMI `_WDG` / `PNP0C14` mechanics are from the public Microsoft WMI-ACPI specification, not from the source.
