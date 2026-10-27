# Provenance: 04-layers

- Baseline: d135ab453a7400a34930bdda1ef4bdc8acfac80e
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- All drivers/gpu/drm/arm/display/* files are GPL-2.0. Only facts (offsets, bits, values, sequences) were taken; no code or comments copied.
- Sources read:
  - drivers/gpu/drm/arm/display/komeda/d71/d71_dev.c: 468-554 (format table, HW_ID packing, 32x8+rot90 restriction)
  - drivers/gpu/drm/arm/display/komeda/komeda_format_caps.c: 12-113 (caps lookup, AFBC bpp, modifier list)
  - drivers/gpu/drm/arm/display/komeda/komeda_format_caps.h: 15-39 (AFBC constants)
  - drivers/gpu/drm/arm/display/komeda/komeda_framebuffer.c: 42-161 (AFBC and linear size checks, 16-byte pitch, equal chroma pitches), 214-270 (src coordinate rules, pixel address)
  - drivers/gpu/drm/arm/display/komeda/d71/d71_component.c: 128-174 (rotation and AFBC control), 187-286 (layer programming, payload/end address, upsampling choice), 354-400 (layer validate: AFBC line, YUV420_10BIT/X0L2 limits), 409-465 (line sizes, ranges, PALPHA), 467-484, 515-519, 527-555 (write-back layer)
  - drivers/gpu/drm/arm/display/komeda/komeda_color_mgmt.c: 10-67 (YUV->RGB tables; values reproduced as numeric constants)
  - drivers/gpu/drm/arm/display/komeda/komeda_pipeline_state.c: 273-430 (layer check, AFBC crop, rotation swap, WB layer)
  - include/uapi/drm/drm_fourcc.h (MIT): FourCC codes and AFBC modifier names (public)
- Second pass, 2026-10-27, Sky1-Linux sources (see SOURCE-BASELINE.md):
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0005 (drivers/gpu/drm/cix/linlon-dp, GPL-2.0, (C) Arm Technology (China)), applied to an empty tree; line numbers refer to the applied files: hw/dp_component.c 212-310 (per-layer colour pipeline, PALPHA write, TBU_EN), 517-560 (WB RGB->YUV), 440-470 (layer colour capability from LAYER_INFO)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0005 (drivers/gpu/drm/cix/linlon-dp, GPL-2.0, (C) Arm Technology (China)), applied to an empty tree; line numbers refer to the applied files: drivers/gpu/drm/cix/linlon-dp/linlondp_color_mgmt.c 14-90 (YUV<->RGB tables; numeric constants), 92-212 (gamma/degamma sample points, 16->12 bit rounding)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0005 (drivers/gpu/drm/cix/linlon-dp, GPL-2.0, (C) Arm Technology (China)), applied to an empty tree; line numbers refer to the applied files: drivers/gpu/drm/cix/linlon-dp/linlondp_wb_connector.c 280-320 (write-back COLOR_ENCODING/RANGE)
