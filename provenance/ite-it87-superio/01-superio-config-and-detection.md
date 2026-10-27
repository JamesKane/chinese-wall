# Provenance: ite-it87-superio/01-superio-config-and-detection

- **Baseline**: `mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e` (`v7.3-16755-gd135ab453a74`)
- **Extracted**: `2026-10-27` by Claude (Opus 5.5) in the dirty room, for James Kane
- **Methodology**: Functional facts only (port numbers, key bytes, register offsets, bit positions, conversion constants, sequences). Every statement was rewritten in fresh prose. No code, comments or identifiers were copied. Driver-internal feature-flag names and structure names were translated into plain capability statements.
- **Sources Read**:
  - `drivers/hwmon/it87.c`:
    - Lines 72–140: index/data port pair, entry key bytes (`0x87 0x01 0x55 0x55/0xAA`), 16-bit MSB-first read, LDN select, exit sequence.
    - Lines 141–182: device IDs (`0x8689`, `0x8695`), LDN 4 activation/base/special-config registers, LDN 7 GPIO/pin-select/beep registers.
    - Lines 323–332: always-in-configuration-mode behavior and the reason it must not be exited.
    - Lines 555–575: IT8689E / IT87952E capability rows (fan/PWM/temp counts, SMBus mask `BIT(1)|BIT(2)` on IT8689E).
    - Lines 875–907: SMBus interlock bracket on LDN 4 `0xF3`.
    - Lines 2854–2984: probe order, `0xFFFF` retry, activation and base checks, revision nibble.
    - Lines 3210–3255: IT8689E pin-mux checks.
    - Lines 3320–3428: generic pin-mux path (used for IT87952E), PINX2 bits 0/1.
    - Lines 3433–3448: SMBus bitmap capture, conditional exit.
    - Lines 3893–3941: ACPI resource-conflict policy and EC port window (base+5, 2 ports).
    - Lines 3982–4035: dual-address probe order and duplicate-base suppression.
    - Lines 3688–3711: revision-based adjustments (older chips only).
