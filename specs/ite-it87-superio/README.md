# ITE IT8689E / IT87952E Super I/O Hardware Monitor Functional Specification

**[HW]** Target hardware: the **Environment Controller (EC)** logical device of ITE IT87-family LPC Super I/O chips, as fitted to current AMD AM5 desktop boards (the X870E reference machine). Two chips are in scope:

| Chip | Super I/O device ID | Role on typical boards | ADC step | Fans | PWM outputs | Temperature inputs |
|---|---|---|---|---|---|---|
| IT8689E | `0x8689` | Primary Super I/O | 12 mV | 6 (16-bit) | 6 | 6 (inputs 4–6 shared with voltage inputs 10–12) |
| IT87952E | `0x8695` | Secondary Super I/O (fan/sensor expander) | 10.9 mV | 3 (16-bit) | 3 | 3 (one can be a PECI/AMD-TSI digital source) |

The chip's other logical devices (UARTs, parallel port, GPIO, watchdog) are out of scope, except where the GPIO logical device's pin-multiplexing registers decide which fan and PWM pins exist.

> OPEN QUESTION: Many Gigabyte AM5 boards are reported to use an **IT8696E** as the primary chip. The reference driver does not recognize device ID `0x8696`, so nothing here is confirmed for it. Before you implement, read the device ID registers on the X870E itself (section 01, §2). If the chip is an IT8696E, treat this specification as a starting hypothesis only: the IT8689E register model is the closest known relative.

**[POLICY]** Each statement in this suite is tagged:

- **[HW]**: required by the chip or by board firmware. Any driver must obey it.
- **[POLICY]**: a choice made by the reference driver. Informative only.
- **[ABI]**: an interface that is visible outside the driver (here, the board firmware's WMI method).

## Relationship to existing permissive drivers

**[POLICY]** The basic IT87 Environment Controller layout (Super I/O entry key, EC base address, the first three fan counters, nine voltage inputs, three temperature inputs) is already implemented by permissively licensed BSD drivers, which the clean room may read directly. This specification concentrates on what those drivers lack for IT8689E/IT87952E: 16-bit fans 4–6, PWM channels 4–6, the separate-duty-register ("newer") automatic fan control model, the extended temperature and voltage inputs, the SMBus interlock, the "always in configuration mode" behavior of the IT87952E, and a dual-chip probe.

## Specification Index

| # | Section | Scope | Status |
|---|---|---|---|
| 01 | [Super I/O Configuration Access and Chip Detection](01-superio-config-and-detection.md) | Index/data ports `0x2E`/`0x4E`, entry/exit keys, logical device select, device ID/revision, EC activation and base address, the always-configured IT87952E, pin-mux registers that remove fans and PWMs, SMBus interlock register, dual-chip probe, ACPI resource ownership | Draft |
| 02 | [Environment Controller Sensor Registers](02-ec-sensor-registers.md) | EC index/data port pair, identification and start-up checks, voltage inputs and scaling, temperature inputs and sensor-type selection (diode/thermistor/PECI/AMD-TSI), 16-bit fan tachometers, alarm and beep registers, chassis intrusion, resume | Draft |
| 03 | [Fan PWM Control](03-fan-pwm-control.md) | Per-channel PWM control registers, on/off vs PWM modes for fans 1–3, manual duty registers, automatic control (temperature map, off/start/full temperatures, start PWM, slope, hysteresis), PWM output frequency, polarity sanity check | Draft |
| 04 | [Gigabyte WMI Temperature Interface](04-gigabyte-wmi-temperature.md) | Board firmware WMI GUID, method IDs, argument and return encoding for up to six firmware-reported temperatures | Draft |

## Critical Summary for Clean-Room Implementers

1. **[HW]** **Never send the configuration-exit command to an IT87952E.** This chip powers up in configuration mode. Leaving that mode can hang the system or make the second Super I/O address stop responding. Only exit configuration mode on a chip where you actually sent the entry key.
2. **[HW]** **Disable the IT8689E's SMBus functions around every EC access.** On IT8689E, two bits in a logical-device-4 configuration register enable SMBus-side access to the EC. Clear them (keeping a saved copy) before each burst of EC index/data cycles, and restore the saved value afterward.
3. **[HW]** **The EC index/data pair is stateful and shared with firmware.** Treat the index/data port pair as a single critical section. If the platform's ACPI tables declare an operation region over the EC ports, firmware code may access the chip at the same time as the OS. See section 01, §7.
4. **[HW]** **Temperature inputs 4–6 and voltage inputs 10–12 share register addresses** on IT8689E. A per-input mode field decides which one each address reports.
5. **[HW]** **The duty register doubles as the automatic "start PWM" value.** On these chips the manual duty cycle and the automatic-mode start PWM are the same register (`base + 3` of each channel's automatic block). The PWM control register only selects the mode and the temperature source.
