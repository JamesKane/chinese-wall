# Provenance: 00-overview

- Baseline: d135ab453a7400a34930bdda1ef4bdc8acfac80e
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- All drivers/gpu/drm/arm/display/* files are GPL-2.0. Only facts (offsets, bits, values, sequences) were taken; no code or comments copied.
- Sources read:
  - git log/show of 8fd7576fd6d8 (Linlon-D6 support), bf5348ed7257 (binding), ec902de02217 (GLB_CORE_ID fix): commit messages and diffs
  - drivers/gpu/drm/arm/display/include/malidp_product.h: 1-41 (product IDs)
  - drivers/gpu/drm/arm/display/komeda/d71/d71_regs.h: 471-534 (block types, constants)
  - drivers/gpu/drm/arm/display/komeda/d71/d71_dev.c: 349-466 (discovery; two-phase-disable rationale), 618-644 (identify)
  - drivers/gpu/drm/arm/display/komeda/komeda_pipeline.c: 332-344 (slave pipeline rule)
  - Documentation/devicetree/bindings/display/arm,komeda.yaml (GPL-2.0-only OR BSD-2-Clause): all
