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
- Second pass, 2026-10-27, Sky1-Linux sources (see SOURCE-BASELINE.md):
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0005 (drivers/gpu/drm/cix/linlon-dp, GPL-2.0, (C) Arm Technology (China)), applied to an empty tree; line numbers refer to the applied files: hw/dp_regs.h (whole file, diffed against d71_regs.h: RAXI field positions, BS_CTRL_PPC, IPS_CTRL_IT/IM, block offsets, CU/IPS input IDs, LW RGB->YUV offset)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0005 (drivers/gpu/drm/cix/linlon-dp, GPL-2.0, (C) Arm Technology (China)), applied to an empty tree; line numbers refer to the applied files: hw/dp_axi.c 1-261 (AXI property ranges, allowed lists, defaults, register/field mapping)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0005 (drivers/gpu/drm/cix/linlon-dp, GPL-2.0, (C) Arm Technology (China)), applied to an empty tree; line numbers refer to the applied files: hw/dp_component.c 212-249 (LT_COEFFTAB select), 1136-1240 (IPS bits), 1286-1340 (BS bits), 1433-1460 (LT table type IDs 0x50/0x52), 1628-1638 (CONFIG_VALID readback poll)
  - patches/0038 commit message (IORT stream-ID grouping, reserved region size) and diff
