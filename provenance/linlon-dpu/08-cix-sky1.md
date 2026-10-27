# Provenance: 08-cix-sky1

- Baseline: d135ab453a7400a34930bdda1ef4bdc8acfac80e
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- All drivers/gpu/drm/arm/display/* files are GPL-2.0. Only facts (offsets, bits, values, sequences) were taken; no code or comments copied.
- Sources read:
  - arch/arm64/boot/dts/cix/sky1-power.h (GPL-2.0-only; numeric IDs only): 14-20
  - include/dt-bindings/clock/cix,sky1.h (GPL-2.0-only OR BSD-2-Clause): 42-72
  - include/dt-bindings/reset/cix,sky1-s5-system-control.h (GPL-2.0-only OR BSD-2-Clause): 28-38, 64-71, 122-131
  - drivers/reset/reset-sky1.c (GPL-2.0-only; register offsets, bits, polarity, delays only): 20-21, 43-48, 72-82, 108-115, 166-175, 266-312, 350-351
  - arch/arm64/boot/dts/cix/sky1.dtsi (BSD-3-Clause): 228-256 (SCMI agents), 510-513 (syscon), 765-768 (s5_syscon); grep for dpu/display/dp (no hits)
  - arch/arm64/boot/dts/cix/sky1-pinfunc.h: 112-114 (DP2 panel pins)
  - git grep -il sky1 over drivers/ and Documentation/devicetree (no display or DP driver/binding)
  - Cross-reference: specs/panthor/11-cix-sky1.md (SCMI transport facts)
  - Not read: any vendor (CIX/Radxa) BSP kernel, ACPI tables, SCMI firmware.
