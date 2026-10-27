# Provenance: 07-uapi

- Baseline: d135ab453a7400a34930bdda1ef4bdc8acfac80e
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- All drivers/gpu/drm/arm/display/* files are GPL-2.0. Only facts (offsets, bits, values, sequences) were taken; no code or comments copied.
- Sources read:
  - drivers/gpu/drm/arm/display/komeda/komeda_kms.c: 20-56 (driver name/features, dumb pitch), 201-219
  - drivers/gpu/drm/arm/display/komeda/komeda_plane.c: 141-271 (planes, types, possible CRTCs, properties, zpos range)
  - drivers/gpu/drm/arm/display/komeda/komeda_crtc.c: 431-461 (mode status codes), 564-693 (CRTC/encoder setup, colour mgmt)
  - drivers/gpu/drm/arm/display/komeda/komeda_wb_connector.c: 64-235 (write-back connector behaviour, clones)
  - drivers/gpu/drm/arm/display/komeda/komeda_dev.c: 22-106 (sysfs, debugfs), drivers/gpu/drm/arm/display/include/malidp_product.h: 29-39 (config_id layout)
  - drivers/gpu/drm/arm/display/komeda/komeda_kms.c: 89-121 (equal zpos rejected); komeda_plane.c 73-80 (disabled CRTC)
  - Mesa kmsro driver-name list: NOT read; stated as an open question from general knowledge
- Second pass, 2026-10-27, Sky1-Linux sources (see SOURCE-BASELINE.md):
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0005 (drivers/gpu/drm/cix/linlon-dp, GPL-2.0, (C) Arm Technology (China)), applied to an empty tree; line numbers refer to the applied files: drivers/gpu/drm/cix/linlon-dp/linlondp_kms.c 71 (driver name), drivers/gpu/drm/cix/linlon-dp/linlondp_plane.c 486-540 (plane properties), drivers/gpu/drm/cix/linlon-dp/linlondp_crtc.c 869-900, drivers/gpu/drm/cix/linlon-dp/linlondp_dev.c 136-168 (aclk_freq_fixed sysfs), drivers/gpu/drm/cix/linlon-dp/linlondp_drm.h
