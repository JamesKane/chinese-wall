# Provenance: 05-composition

- Baseline: d135ab453a7400a34930bdda1ef4bdc8acfac80e
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- All drivers/gpu/drm/arm/display/* files are GPL-2.0. Only facts (offsets, bits, values, sequences) were taken; no code or comments copied.
- Sources read:
  - drivers/gpu/drm/arm/display/komeda/d71/d71_component.c: 578-624 (compositor programming), 695-793 (filter table select, phases, control), 871-910 (downscaling clock check), 912-922 (splitter), 974-988 (merger), 837-869, 945-972, 1015-1044 (limits)
  - drivers/gpu/drm/arm/display/komeda/komeda_pipeline_state.c: 432-555 (scaler checks, write-back no upscale), 557-755 (splitter, merger, compositor input, compiz output no alpha), 839-911 (path choice, enhancement and split triggers), 913-1131 (split arithmetic, layer split), 1133-1182 (write-back paths)
  - drivers/gpu/drm/arm/display/komeda/komeda_kms.c: 89-174 (zpos normalisation, slave zorder)
  - drivers/gpu/drm/arm/display/komeda/komeda_plane.c: 14-95 (blending_zorder offset by max slave zorder; blend mode)
  - drivers/gpu/drm/arm/display/komeda/komeda_pipeline.c: 294-305 (right-layer pairing)
  - drivers/gpu/drm/arm/display/komeda/komeda_wb_connector.c: 10-63, 113-140 (write-back data flow, connector only with wb_layer)
  - drivers/gpu/drm/arm/display/komeda/d71/d71_regs.h: 512-522 (scaler and merger constants)
- Second pass, 2026-10-27, Sky1-Linux sources (see SOURCE-BASELINE.md):
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0005 (drivers/gpu/drm/cix/linlon-dp, GPL-2.0, (C) Arm Technology (China)), applied to an empty tree; line numbers refer to the applied files: hw/dp_component.c 787-840 (right-part scaler crop), drivers/gpu/drm/cix/linlon-dp/linlondp_pipeline_state.c 685-760 (compositor size and input size in side-by-side)
