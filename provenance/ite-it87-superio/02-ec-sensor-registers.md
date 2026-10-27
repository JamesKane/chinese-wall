# Provenance: ite-it87-superio/02-ec-sensor-registers

- **Baseline**: `mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e` (`v7.3-16755-gd135ab453a74`)
- **Extracted**: `2026-10-27` by Claude (Opus 5.5) in the dirty room, for James Kane
- **Methodology**: Functional facts only (port numbers, key bytes, register offsets, bit positions, conversion constants, sequences). Every statement was rewritten in fresh prose. No code, comments or identifiers were copied. Driver-internal feature-flag names and structure names were translated into plain capability statements.
- **Sources Read**:
  - `drivers/hwmon/it87.c`:
    - Lines 197–283: EC register addresses (config, alarms, fan, fan-limit, fan high-byte, voltage, temp, limit, enable, offset, vendor ID, auto base, TEMP456 enable).
    - Lines 723–777: ADC step values (12 mV, 10.9 mV), doubling for scaled inputs, 16-bit fan conversion (1,350,000 / (count×2)), temperature encoding.
    - Lines 909–930: EC index/data access and busy-flag policy.
    - Lines 1015–1150: polling cadence (1.5 s), read order for fan low/high bytes, alarm register assembly.
    - Lines 1260–1300: offset write gate via `0x5C` bit 7.
    - Lines 1340–1450: temperature sensor-type encoding (`0x51`, `0x55` bit 7, `0x0A` bits 6:4 = AMD-TSI).
    - Lines 2234–2350: alarm bit assignments, intrusion clear (config bit 5), beep classes.
    - Lines 2387–2420: internal-rail labels.
    - Lines 3450–3592: limit fix-ups, voltage/tach enable fix-ups, fan presence bits (`0x13` bits 6:4, `0x0C` bits 4/5/2), PWM5/PWM6 test via `0x0B`, monitoring start mask `0x3E`.
    - Lines 3714–3780: vendor ID `0x90` and config bit 7 check, scaled-input assignment, TEMP456 field decoding.
    - Lines 3821–3881: resume sequence.
