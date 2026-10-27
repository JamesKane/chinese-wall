# Provenance: sky1-peripherals/05-pinctrl

- Baseline: mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e plus Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f (patches/) and Sky1-Linux/linux@665339c3109ba8fbce67aca29e5327e134d6b2b6 (vendor DT, BSD-3)
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Only facts (offsets, bits, IDs, sequences, table formats) taken; no code or comments copied.
- Sources read:
  - mainline drivers/pinctrl/cix/pinctrl-sky1-base.c (GPL-2.0): 28-48 (field masks, drive table, default), 283-298 (mux write), 352-410 (pull and drive writes)
  - mainline drivers/pinctrl/cix/pinctrl-sky1.c (GPL-2.0): 98-165 (S5 pad list, 66 pads), 353-497 (FCH pad list, 138 pads; PM; compatibles); pad names only
  - mainline arch/arm64/boot/dts/cix/sky1-pinfunc.h (GPL-2.0-only OR BSD-2-Clause): all 1-401 (Annex A generated mechanically by joining with the pad lists above)
  - mainline Documentation/devicetree/bindings/pinctrl/cix,sky1-pinctrl.yaml (GPL-2.0-only OR BSD-2-Clause): all (pinmux encoding, drive-strength values)
  - mainline arch/arm64/boot/dts/cix/sky1.dtsi: 516-519, 819-822 (controller addresses)
  - Sky1-Linux/linux-sky1@57e018a patches/0003: 2217-2260 (vendor masks), 2355-2460 (vendor mux/config writes), 2515-2540 (vendor triple encoding), 3285-3300 (FCH context-loss flag), 3309-3372 (save/restore), 3384-3390 (CIXHA016/CIXHA017), 3415-3475, 3481-3585 (unused ACPI pin-group helper header), 6215-6232 (PULL_UP/PULL_DOWN/ST/DS_LEVEL values)
  - Sky1-Linux/linux-sky1@57e018a patches/0091: 1-45, 1369-1417 (ACPI mode: firmware-configured pins, no groups)
  - Sky1-Linux/linux-sky1@57e018a patches/0063 339 (CIXH1007 = GPT); patches/0065 96-103 (CIXH1006 = DMA)
  - Sky1-Linux/linux@665339c vendor sky1-orion-o6.dts (BSD-3): 353-610, 605-940 (pin groups; Table 5.4.2 extracted with a script)
  - Sky1-Linux/acpi-to-dts-tools@9ddd22c (MIT): docs/ACPI-to-DTS-Mapping.md 234-283, 501-505; acpi-to-dts.sh 95-96
