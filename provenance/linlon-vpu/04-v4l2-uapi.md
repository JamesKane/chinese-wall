# Provenance: linlon-vpu/04-v4l2-uapi

- Baseline: vendor sources below
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Driver source: drivers/media/platform/cix/ as created by Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0010-media-cix-Add-Sky1-video-codec-VPU-driver.patch (extracted with git apply into an empty tree; paths and line numbers below are in the extracted files). Later patches touching it: 0127 (Kconfig select lines only; no facts used). 0132 and 0139 were checked and do not touch drivers/media/platform/cix.
- Licence: every driver file carries SPDX GPL-2.0-only; the header text also contains an Arm Technology (China) "confidential and proprietary" notice and an "or (at your option) any later version" GPL paragraph, which contradict each other and the SPDX tag. No file is dual-licensed or MIT. Only facts (offsets, bits, values, layouts, sequences, ABI numbers) were taken; no code or comments were copied or reworded.
- Note: linux/mvx-v4l2-controls.h is the vendor UAPI header (GPL-2.0-only SPDX, no Linux-syscall-note). Only numeric values (fourccs, control IDs, flag values, ioctl numbers) were used. Ioctl values were computed from _IOWR with hand-computed structure sizes.
- Sources read:
  - if/v4l2/mvx_ext_v4l2.c 60-237 (node names, caps, ioctl table)
  - if/v4l2/mvx_v4l2_vidioc.c 310-394 (buffer counts, plane sizes), 692-830 (vb2 setup, querycap, enum fmt, frame sizes), 1022-1125 (g_selection), 1391-1510 (s_selection, s_parm), 1563-1672 (encoder/decoder commands), 1868-1910 (EOS event on LAST), 1982-2102 (events, private ioctls)
  - if/v4l2/mvx_v4l2_buffer.c 180-300, 405-465 (flag translation)
  - if/v4l2/mvx_v4l2_ctrls.c 40-120 (menus), 400-1000 (control handling), 1100-1900 (control definitions and ranges)
  - if/v4l2/mvx_v4l2_session.c 480-520 (source-change event); if/v4l2/mvx_v4l2_session.h 62 (256 fps cap)
  - linux/mvx-v4l2-controls.h 30-661 (vendor fourccs, flags, control enum, ioctl definitions, argument structures)
  - if/mvx_session.c 86-490 (format maps), 3065-3143 (AFBC size), 3154-3316 (try_format clamps and sizes), 3371-3437 (decoder output filter), 3659-3722 (enumeration and removed formats), 4155-4312 (silent requeue, reallocation), 4420-4550 (bit-depth switch, buffer minimum), 6588-6700 (priority, frame-size limits)
  - if/mvx_session.h 80-83 (buffer minima); if/mvx_buffer.c 50, 66-196, 494-545 (stride/size rules)
  - dev/mvx_hwreg_v52_v76.c 30-120 (format capability list); dev/mvx_dev.c 200-245 (fuse filtering)
