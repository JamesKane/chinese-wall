# Provenance: ite-it87-superio/03-fan-pwm-control

- **Baseline**: `mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e` (`v7.3-16755-gd135ab453a74`)
- **Extracted**: `2026-10-27` by Claude (Opus 5.5) in the dirty room, for James Kane
- **Methodology**: Functional facts only (port numbers, key bytes, register offsets, bit positions, conversion constants, sequences). Every statement was rewritten in fresh prose. No code, comments or identifiers were copied. Driver-internal feature-flag names and structure names were translated into plain capability statements.
- **Sources Read**:
  - `drivers/hwmon/it87.c`:
    - Lines 254–282: PWM control, duty and automatic-block base addresses.
    - Lines 779–850: temperature-source field mask/shift and channel 4–6 offset interpretation.
    - Lines 852–873: PWM base-clock table and datasheet discrepancy note (750 kHz vs 51 kHz).
    - Lines 932–1000: automatic block layout (off/start/full temps, start PWM, slope, hysteresis).
    - Lines 1460–1475, 1534–1556: mode decoding and frequency-select field locations.
    - Lines 1650–1848: trip-point ordering check, mode switching via `0x13`/`0x14`/control bit 7, duty write refusal in auto mode, frequency-select writes (`0x14`, `0x55` bits 6:4).
    - Lines 1850–2045: temperature map writes, start PWM, slope (bits 6:0), hysteresis (bits 4:0).
    - Lines 3521–3545: default map and full-speed assumptions.
    - Lines 3594–3647: polarity sanity check (`0x14 & 0x87`) and repair path.
    - Lines 3797–3806: PWM availability masking.
