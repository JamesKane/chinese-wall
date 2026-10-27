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
