# Provenance: spacemit-k3/02-pinctrl-gpio

- Baseline: mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Only facts taken (bit positions, offsets, pin ranges, drive tables, unlock keys, register order). Tables were rebuilt in new layouts. Policy behaviour is described in prose, not by function structure.
- Sources read (primary, mainline):
  - drivers/pinctrl/spacemit/pinctrl-k1.c 1-827 (GPL-2.0): pad field layout 28-50; I/O power-domain offsets and unlock keys 52-71; K3 pin-to-offset and pin-to-domain rules 163-218; drive tables 271-335; drive and voltage handling 337-431; mux, GPIO-request, pinconf parsing 442-702; probe (clock names, apbc syscon) 763-826
  - drivers/pinctrl/spacemit/pinctrl-k1.c 1099-1456: K3 pin names, per-pin GPIO function and supply class (summarized as ranges by a throwaway script), compatibles
  - drivers/pinctrl/spacemit/pinctrl-k1.h 1-40: supply-class encoding
  - drivers/gpio/gpio-spacemit-k1.c 1-377 (GPL-2.0 OR MIT): register order and K3 offsets, bank offsets, interrupt handling, generic-chip flags
  - Documentation/devicetree/bindings/pinctrl/spacemit,k1-pinctrl.yaml 1-160 and gpio/spacemit,k1-gpio.yaml 40-95 (dual-licensed)
  - arch/riscv/boot/dts/spacemit/k3.dtsi 911-927 (GPIO node), 1178-1185 (pinctrl node); k3-pinctrl.dtsi 8, 748-756 (MIT dual)
- Secondary (2026-10-27):
  - spacemit-com/docs-chip@df43cfe4 en/key_stone/k3/k3_docs/k3_usermanual/03_pinout.md 833-858, 882, 1851-1931 (MFPR address table: confirms pin-ID-to-offset rule and the gap at 0x20C/0x210); 16_peripherals/gpio.md 1-298 (register semantics, reset values, other GPIO instances; CPMASK description absent)
  - spacemit-com/edk2-platforms@0b8b7182 (BSD-2-Clause-Patent) Silicon/Spacemit/K3/Drivers/PinCtrlDxe/PinCtrlDxe.h 20-135 (field definitions, domain offsets, unlock keys), PinCtrlDxe.c 50-54 and 210-335 (pad-index numbering, unlock before read-back); GPIO/GpioDxe.h 25-80 (bank offsets, GPIO pad defaults); Platform/Spacemit/K3/MUSE-Pico/MUSE-Pico.dsc 271-287 (MFPR base, unlock address 0xD4015050, 128 GPIOs)
  - spacemit-com/linux-6.18@7d17e358 arch/riscv/boot/dts/spacemit/k3-pinctrl.dtsi 3, 2765-2771 (same pin-ID numbering as mainline)
- FreeBSD check: ../freebsd-src/sys/riscv/spacemit/files.spacemit and a grep of sys/dev/gpio, sys/dev/fdt, sys/riscv (no SpacemiT pinctrl or GPIO driver)
