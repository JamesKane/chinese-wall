# Provenance: sky1-dp-tx/01-registers

- Baseline: Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- drivers/gpu/drm/cix/dptx/* is GPL-2.0. Only facts (offsets, bits, values, sequences) were taken; no code or comments copied.
- Sources read (patches/0005 applied to an empty tree; line numbers refer to the applied files under drivers/gpu/drm/cix/dptx/):
  - trilin_dptx_reg.h 22-452 (all register offsets, source stride, source-1 CRC/PSR copies, odd GTC_AUX_FRAME_SYNC offset)
  - trilin_dptx.h 58-160 (interrupt bits, AUX command/status/reply-code bits, MISC0/MISC1/override fields, polarity shifts, TU size, interrupt masks 0x3F/0x1C)
  - trilin_dptx.c 145-166 (PHY_STATUS lane-ready mask), 364-408 (TPS4 core code 4 vs DPCD 7), 541-549, 588-644 (link config registers, downspread), 763-835 (AUX register usage), 906-929 (AUX divider semantics: 1 MHz base, range 10-400, from a vendor comment restated), 954-1074 (TU_CONFIG fields, timing registers, USER_SYNC_POLARITY written for source 0 only, DATA_CONTROL, USER_PIXEL_COUNT), 1090-1116 (MVID/NVID sync mode only), 1257-1280, 1332, 1370 (PSR registers and values), 1384-1512 (MISC/override values), 1540-1559, 1713-1772 (MST payload table, select, trigger, active table), 1991-2114 (TRANSMITTER/SOFT_RESET/SOURCE_ENABLE/FORCE_SCRAMBLER_RESET/MST_ENABLE values), 2344-2346, 2433-2435 (interrupt mask values), 2490-2495 (HPD_INPUT_STATE), 2684-2720 (cause/mask handling), 2762-2845 (register dump ranges)
  - trilin_host_tmr.h 27-41, trilin_host_tmr.c 18-254 (HOST_TIMER fields, 1 MHz tick)
  - dptx_infoframe.c 31-116 (SDP buffer select/enable/rate/data FIFO, 7 buffers, sources 0-1 only)
  - trilin_dptx_audio.h 10-34, trilin_dptx_audio.c 14-80 (secondary-channel audio registers and values)
  - trilin_drm.c 1011-1066 (RCSU CFG_ADAPTER offsets 0x300/0x304/0x320 and bit fields)
  - hdcp/dptx_reg.h 84-129 (HDCP register offsets); hdcp2_hw_tx.c 40-210 (grep only: register names/access); trilin_dptx.c 3081-3096 (HDCP timer at 0x48C)
  - patches/0136 (additional channel-status codes), 0137 (AUDIO_ENABLE on shutdown)
  - Sky1-Linux/linux@665339c arch/arm64/boot/dts/cix/sky1.dtsi 2836-2985 (dp_rcsu window sizes 0x304/0x320)
- Inferences labelled in the text: INTERRUPT_CAUSE clear-on-read; TU size field width; SOFT_RESET semantics; MISC0 equals the DP MSA MISC0 byte (by comparison of the vendor values with the public DP standard)
