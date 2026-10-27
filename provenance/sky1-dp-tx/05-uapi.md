# Provenance: sky1-dp-tx/05-uapi

- Baseline: Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- drivers/gpu/drm/cix/dptx/* and sound/soc/cix/* are GPL-2.0. Only facts (names, numbers, capabilities) were taken; no code or comments copied.
- Sources read (patches/0005 applied to an empty tree; line numbers refer to the applied files under drivers/gpu/drm/cix/dptx/):
  - trilin_drm.c 236-357 (fallback/added modes), 431-516 (debugfs files, connector funcs), 1098-1111 (connector properties, max bpc 8-10), 1130-1184 (connector/encoder types, polling)
  - trilin_drm_mst.c 290-335 (MST connector properties), 891-935 (MST encoders, possible CRTCs 0x3, names)
  - trilin_dptx.h 178-180 (possible CRTC masks)
  - trilin_dptx.c 56-68 (module parameters), 3052-3068 (codec registration: I2S, 8 channels)
  - hdcp/cix_hdcp.c 28-120, 340-395 (misc device name, fops, event queue); hdcp/cix_hdcp.h 100-140 (event list); hdcp/cix_hdcp_ioctl_cmd.h 100-116 (ioctl magic and numbers); hdcp/cix_hdcp_ioctl.c 1-80 (grep only)
  - patches/0022 (HDMI-A connector via bridge chain), 0107 (no capture, _UID numbering; card topology names, cix_sky1 model), 0128 (CEC), 0135/0138 (ELD reporting and readiness), 0136 (sample rates)
