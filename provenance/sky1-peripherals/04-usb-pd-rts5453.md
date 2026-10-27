# Provenance: sky1-peripherals/04-usb-pd-rts5453

- Baseline: mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e plus Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f (patches/) and Sky1-Linux/linux@665339c3109ba8fbce67aca29e5327e134d6b2b6 (vendor DT, BSD-3)
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Only facts (offsets, bits, IDs, sequences, table formats) taken; no code or comments copied.
- Sources read:
  - Sky1-Linux/linux-sky1@57e018a patches/0004 drivers/usb/typec/rts5453.c: 4713-5325 (register addresses, status bit layout, ack payload, event order, orientation/mode/role/HPD decoding, alt-mode capabilities, properties read, PM)
  - Sky1-Linux/linux-sky1@57e018a patches/0065: 321-355 (CIXH200D match)
  - Sky1-Linux/linux-sky1@57e018a patches/0026: 1-60 (O6 PD enable, signal chain); patches/0030: 1-40 (O6N uses a different PD part)
  - Sky1-Linux/linux-sky1@57e018a patches/0001: 1708-1750 (O6 PD id values); patches/0020: 1763-1830 (Orange Pi 6 Plus PD nodes)
  - Sky1-Linux/linux@665339c vendor sky1-orion-o6.dts (BSD-3): 1713-1785 (O6 PD nodes, connector properties, graph links)
  - Sky1-Linux/acpi-to-dts-tools@9ddd22c (MIT): docs/ACPI-to-DTS-Mapping.md 108-117, 501-508
  - Cross-checked against specs/sky1-dp-tx/03-usbdp-combo-phy.md §3.11 (this repository)
