# Provenance: sky1-soc/03-gpt-timer

- Baseline: mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e plus the Sky1-Linux revisions below
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Only facts (offsets, bits, IDs, sequences, table formats) taken; no code or comments copied.
- Sources read:
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0063 (whole patch: register offsets, TCTL bits, setup order, counter read, compare order, interrupt clear, clock/irq/ACPI CIXH1007)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0080 commit message (vendor clock names)
  - Sky1-Linux/linux@665339c3109ba8fbce67aca29e5327e134d6b2b6 arch/arm64/boot/dts/cix/sky1.dtsi (BSD-3-Clause): 3118-3160 (four instances, IRQs, clocks, reset)
  - mainline include/dt-bindings/clock/cix,sky1.h 251, 270 (timer clock IDs); mainline drivers/reset/reset-sky1.c (FCH timer reset bit 20)
  - Not read: ACPI GTDT / processor _LPI tables (not available)
