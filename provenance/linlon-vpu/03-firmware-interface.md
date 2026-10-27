# Provenance: linlon-vpu/03-firmware-interface

- Baseline: vendor sources below
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Driver source: drivers/media/platform/cix/ as created by Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0010-media-cix-Add-Sky1-video-codec-VPU-driver.patch (extracted with git apply into an empty tree; paths and line numbers below are in the extracted files). Later patches touching it: 0127 (Kconfig select lines only; no facts used). 0132 and 0139 were checked and do not touch drivers/media/platform/cix.
- Licence: every driver file carries SPDX GPL-2.0-only; the header text also contains an Arm Technology (China) "confidential and proprietary" notice and an "or (at your option) any later version" GPL paragraph, which contradict each other and the SPDX tag. No file is dual-licensed or MIT. Only facts (offsets, bits, values, layouts, sequences, ABI numbers) were taken; no code or comments were copied or reworded.
- Note: external/fw_v{2,3}/mve_protocol_def.h are Arm Technology (China) protocol definitions under the same contradictory GPL-2.0-only header. Constant values and structure layouts were taken as firmware-ABI facts; offsets were computed by hand from field types with natural alignment. Comments were not reproduced.
- Sources read:
  - if/mvx_firmware_cache.h 70-115 (header layout)
  - if/mvx_firmware_cache.c 112-170 (file names), 203-225 (validation), 280-340 (page counting)
  - if/mvx_firmware.c 60-85 (version selection), 152-300 (mapping and copy), 517-585 (protocol dispatch)
  - if/mvx_firmware_priv.h 40-60 (IDLE_ACK code)
  - if/mvx_firmware_v2.c 328-526 (ring read, checksum verification, duplicate SWITCHED_IN), 528-670 (ring write, checksum insertion, "-sum" detection, barriers/cache), 756-1222 (buffer descriptor parsing, response handling, error codes, min 144 check), 1225-1620 (FRAME and BITSTREAM descriptor filling, interlace, rotation, AFBC split), 1622-1915 (NALU, profile, level, GOP, CABAC, VP9 mapping), 1916-3158 (request encoding for every option and PARAM type, flush, EOS sentinel descriptors), 3159-3533 (RPC), 3534-3575 (print buffer), 3862-3914 (IDLE_ACK version gate)
  - if/mvx_firmware_v3.c 120-173 (protocol-3 differences)
  - external/fw_v3/mve_protocol_def.h 100-2019 and external/fw_v2/mve_protocol_def.h (diffed against v3) (all message codes, option indices, payload layouts, flags, format bitfields)
  - if/mvx_session.c 490-540 (watchdog), 773-1040 (pending-reply wait 3 s, doorbell + switch-in after each message, EOS), 1184-1225 (common options), 1326-2323 (encoder option order), 2324-2478 (decoder options, AV1 0x100), 2479-2534 (initial message order), 2976-3063 (session start), 3317-3360 (watchdog escalation, 15 expiries), 3451-3488 (flush and requeue), 3867-4068 (EOS, stream on/off, start), 4070-4700 (message handlers, idle count 2, switch-out handling, IRQ processing order)
  - dev/mvx_scheduler.c 60-420 (LSID selection, pending list), 600-760 (priority list, switch-in/out, send/trigger IRQ, flush), 760-900 (IRQ dispatch, terminate), 924-1000 (suspend/resume), 1110-1158 (job-frames policy by session count)
  - dev/mvx_dev.c 1088-1220 (suspend/resume interaction)
