# Provenance: 06-output

- Baseline: d135ab453a7400a34930bdda1ef4bdc8acfac80e
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- All drivers/gpu/drm/arm/display/* files are GPL-2.0. Only facts (offsets, bits, values, sequences) were taken; no code or comments copied.
- Sources read:
  - drivers/gpu/drm/arm/display/komeda/d71/d71_component.c: 1046-1091 (IPS programming), 1126-1159 (IPS capabilities), 1166-1208 (BS programming, prefetch, drift timeout, dual-link)
  - drivers/gpu/drm/arm/display/komeda/d71/d71_regs.h: 416-424 (BS field widths), 437-448 (IPS bits)
  - drivers/gpu/drm/arm/display/komeda/komeda_color_mgmt.c: 69-128 (gamma sample positions, CTM conversion); komeda_color_mgmt.h (precision 12, 65 coeffs, 9 CTM coeffs)
  - drivers/gpu/drm/drm_color_mgmt.c: 136-148 (S31.32 to Qm.n conversion; behaviour only); include/drm/drm_color_mgmt.h: 43-51 (LUT rounding; behaviour only)
  - drivers/gpu/drm/arm/display/komeda/komeda_crtc.c: 23-63 (colour config, clock ratio), 112-164 (aclk set once), 404-486 (mode fixup/valid), 675 (gamma LUT size, no degamma)
  - drivers/gpu/drm/arm/display/komeda/komeda_pipeline_state.c: 656-665 (composition size = hdisplay/vdisplay), 757-815 (depth/format choice)
  - drivers/gpu/drm/arm/display/komeda/komeda_kms.c: 206-219 (4096x4096 limits)
- Second pass, 2026-10-27, Sky1-Linux sources (see SOURCE-BASELINE.md):
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0005 (drivers/gpu/drm/cix/linlon-dp, GPL-2.0, (C) Arm Technology (China)), applied to an empty tree; line numbers refer to the applied files: hw/dp_component.c 1136-1240 (IPS: degamma, RGB->YUV load, dither, SBS/IM), 1286-1340 (BS: PPC, VD/VM, TM, CRC, PM, active-width adjust)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0005 (drivers/gpu/drm/cix/linlon-dp, GPL-2.0, (C) Arm Technology (China)), applied to an empty tree; line numbers refer to the applied files: drivers/gpu/drm/cix/linlon-dp/linlondp_crtc.c 592-760 (smart aclk table, aclk minimum in SBS, mode_valid 600/1200 MHz thresholds, >= rounding, mode_fixup divisor), 869-900 (CTM_EXT, protected mode), 919-955 (CRTC setup in SBS)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0005 (drivers/gpu/drm/cix/linlon-dp, GPL-2.0, (C) Arm Technology (China)), applied to an empty tree; line numbers refer to the applied files: drivers/gpu/drm/cix/linlon-dp/linlondp_pipeline_state.c 1218-1330 (SBS plane split), 1546-1602 (SBS display data path); drivers/gpu/drm/cix/linlon-dp/linlondp_pipeline.c 383-441 (SBS assembly); drivers/gpu/drm/cix/linlon-dp/linlondp_dev.c 372-375, 440-453 (pixel_per_cycle, side_by_side_master DT properties)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0005 (drivers/gpu/drm/cix/linlon-dp, GPL-2.0, (C) Arm Technology (China)), applied to an empty tree; line numbers refer to the applied files: drivers/gpu/drm/cix/linlon-dp/linlondp_drm.h (HDR EOTF enum, CTM_EXT layout)
