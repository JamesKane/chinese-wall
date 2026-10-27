# Provenance: 02-register-map

- Baseline: d135ab453a7400a34930bdda1ef4bdc8acfac80e
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- All drivers/gpu/drm/arm/display/* files are GPL-2.0. Only facts (offsets, bits, values, sequences) were taken; no code or comments copied.
- Sources read:
  - drivers/gpu/drm/arm/display/komeda/d71/d71_regs.h: 1-541 (all register offsets, bit fields, block types, defaults)
  - drivers/gpu/drm/arm/display/komeda/d71/d71_component.c: 75-121 (0xA4 write-only, line-size register), 288-352, 486-513, 626-660, 795-829, 924-937, 990-1007, 1093-1118, 1210-1245, 1358-1424 (register dump functions: register names at offsets not otherwise defined), 176-185 (input ID encoding), 15-58 (instance -> pipeline/index mapping)
  - drivers/gpu/drm/arm/display/komeda/d71/d71_dev.c: 13-160 (status clearing pattern, CU bit-31 mask), 321-336 (header read: 4-byte strides), 385-410 (PERIPH/GCU config decode)
  - drivers/gpu/drm/arm/display/include/malidp_product.h: 10-27 (GLB_CORE_ID fields, product IDs); git show ec902de02217 (old field decode)
  - drivers/gpu/drm/arm/display/include/malidp_io.h: all (64-bit write order low then high)
  - drivers/gpu/drm/arm/display/komeda/komeda_event.c: 55-120 (event names; CPE = coprocessor)
  - Inferences labelled in the text (IBSY, ACE0-3, CSCE, ZME, DRIFTTO meanings; RCACHE bit meanings from AxCACHE defines lines 331-335)
