# Provenance: sky1-peripherals/01-gpio

- Baseline: mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e plus Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f (patches/) and Sky1-Linux/linux@665339c3109ba8fbce67aca29e5327e134d6b2b6 (vendor DT, BSD-3)
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Only facts (offsets, bits, IDs, sequences, table formats) taken; no code or comments copied.
- Sources read:
  - mainline drivers/gpio/gpio-cadence.c (GPL-2.0): 22-33 (offsets), 56-75 (bypass handling), 77-139 (mask/unmask, trigger encoding), 141-158 (chained handler), 185-301 (probe order, defaults)
  - mainline arch/arm64/boot/dts/cix/sky1.dtsi: 276-281 (S5 GPIO fixed clock), 446-508 (FCH GPIO nodes), 771-817 (S5 GPIO nodes)
  - mainline include/dt-bindings/clock/cix,sky1.h: 271 (FCH_GPIO_APB = 262)
  - Sky1-Linux/linux-sky1@57e018a patches/0058 (GPIO edge IRQ, PM, wake SMC, ACPI IDs): 1-427
  - Sky1-Linux/linux-sky1@57e018a patches/0067 (ack W1C, IRQ storm, GIC hwirq for SMC, restore ordering): 1-150
  - Sky1-Linux/linux-sky1@57e018a patches/0020 (Orange Pi 6 Plus DT): 1666-1700 (RTC/EC IRQ lines), 1763-1830 (PD IRQ line)
  - Sky1-Linux/linux-sky1@57e018a patches/0030: 1-100 (O6N pad comments GPIO001/GPIO031)
  - Sky1-Linux/linux@665339c vendor arch/arm64/boot/dts/cix/sky1.dtsi (BSD-3): 860-875 (GPIO fixed clocks), 3212-3340 (GPIO nodes, clock names, resets)
  - Sky1-Linux/linux@665339c vendor sky1-orion-o6.dts (BSD-3): 200-325 (regulators/LEDs and their GPIO lines), 358-410, 586-601 (pad-to-line comments), 1719-1760 (PD IRQ)
  - Sky1-Linux/acpi-to-dts-tools@9ddd22c (MIT): docs/ACPI-to-DTS-Mapping.md 39-40, 185, 228, 362; examples/generated-o6-from-acpi.dts 186-255 (GpioIo users), 437-525 (GPIO controllers, HID/address pairing); examples/generated-o6n-from-acpi.dts 526-600; acpi-to-dts.sh 1331-1410 (HID handling), 1640-1655 (GPIn label guess)
