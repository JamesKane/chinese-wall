# Provenance: sky1-peripherals/03-spi-qspi

- Baseline: mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e plus Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f (patches/) and Sky1-Linux/linux@665339c3109ba8fbce67aca29e5327e134d6b2b6 (vendor DT, BSD-3)
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Only facts (offsets, bits, IDs, sequences, table formats) taken; no code or comments copied.
- Sources read:
  - mainline drivers/spi/spi-cadence.c (GPL-2.0-or-later): 27-102 (offsets, CR/ISR/ER bits), 165-315 (init, chip select, mode, baud), 396-491 (FIFO processing, ISR, TXFULL delay), 514-619 (transfer, unprepare, FIFO-depth probe), 642-777 (probe: clock/reset names, num-cs, Sky1 word sizes), 810-892 (PM, match table)
  - Sky1-Linux/linux-sky1@57e018a patches/0065: 1-33, 251-320 (CIXH2001 -> spi-cadence, CIXH2002 -> spi-cadence-quadspi with generic platform data)
  - Sky1-Linux/linux-sky1@57e018a patches/0012: 336-346 (CIXH2011 = PWM), 1050-1056 (CIXH1005 = watchdog)
  - mainline drivers/spi/spi-cadence-quadspi.c: 149-290 (register offsets, used only to state the map differs), 2102-2104, 2160-2166 (generic platform data)
  - mainline drivers/spi/spi-cadence-xspi.c (GPL-2.0-only): 25-150 (XSPI register offsets and bits), 370-379 (command register write order), 500-525, 970-995 (slave-DMA direction), 1255-1300 (region requirements); Documentation/devicetree/bindings/spi/cdns,xspi.yaml (GPL-2.0 OR BSD-2-Clause) 1-90
  - mainline arch/arm64/boot/dts/cix/sky1.dtsi: 362-380 (SPI nodes); include/dt-bindings/clock/cix,sky1.h 249-261
  - Sky1-Linux/linux@665339c vendor sky1.dtsi (BSD-3): 804-823 (XSPI fixed clocks), 998-1026 (SPI nodes), 1113-1125 (XSPI node); vendor sky1-orion-o6.dts 452-470 (SPI/XSPI pin groups), 2072-2132 (board SPI/XSPI usage)
  - Sky1-Linux/linux-sky1@57e018a patches/0020: 1905-1940 (Orange Pi 6 Plus SPI/XSPI)
  - Sky1-Linux/acpi-to-dts-tools@9ddd22c (MIT): docs/ACPI-to-DTS-Mapping.md 41, 100; examples/generated-o6-from-acpi.dts 676-683 (CIXH2001 at 0x04090000); acpi-to-dts.sh 99-103, 1505-1525
