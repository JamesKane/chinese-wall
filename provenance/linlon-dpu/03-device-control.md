# Provenance: 03-device-control

- Baseline: d135ab453a7400a34930bdda1ef4bdc8acfac80e
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- All drivers/gpu/drm/arm/display/* files are GPL-2.0. Only facts (offsets, bits, values, sequences) were taken; no code or comments copied.
- Sources read:
  - drivers/gpu/drm/arm/display/komeda/d71/d71_dev.c: 13-256 (IRQ decode, clear, masks), 258-265 (vblank mask), 267-306 (opmode, flush), 308-319 (soft reset), 349-466 (discovery), 556-601 (TBU connect/disconnect)
  - drivers/gpu/drm/arm/display/include/malidp_utils.h: 16-24 (poll helper: tries and sleep ranges)
  - drivers/gpu/drm/arm/display/komeda/d71/d71_component.c: 409-465 (layer line sizes, PALPHA init), 527-555, 557-576 (component disable incl. CU input control), 668-693, 837-869 (scaler init SC_CONTROL=0), 945-972, 1015-1044, 1126-1159, 1161-1164, 1253-1356 (probe dispatch, unknown block -> error)
  - drivers/gpu/drm/arm/display/komeda/komeda_pipeline.c: 163-230 (dense layer/scaler index), 274-292 (input verification)
  - drivers/gpu/drm/arm/display/komeda/komeda_crtc.c: 112-201 (prepare/unprepare: opmode, clocks), 203-248 (event handling), 250-402 (flush, enable, two-phase disable, 1 s wait)
  - drivers/gpu/drm/arm/display/komeda/komeda_pipeline_state.c: 1277-1356 (two-phase disable, update/disable per component)
  - drivers/gpu/drm/arm/display/komeda/komeda_dev.h: 16-61, 141-151 (event bits, mode enum)
  - drivers/gpu/drm/arm/display/komeda/komeda_kms.c: 30-46 (IRQ dispatch), 57-85 (commit tail)
- Second pass, 2026-10-27, Sky1-Linux sources (see SOURCE-BASELINE.md):
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0005 (drivers/gpu/drm/cix/linlon-dp, GPL-2.0, (C) Arm Technology (China)), applied to an empty tree; line numbers refer to the applied files: hw/dp_component.c 1628-1638; drivers/gpu/drm/cix/linlon-dp/linlondp_crtc.c 436-490 (3-frame flip timeout, soft reset); hw/dp_dev.c 310-316 (SRST by RMW)
