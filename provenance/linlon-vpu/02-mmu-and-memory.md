# Provenance: linlon-vpu/02-mmu-and-memory

- Baseline: vendor sources below
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Driver source: drivers/media/platform/cix/ as created by Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0010-media-cix-Add-Sky1-video-codec-VPU-driver.patch (extracted with git apply into an empty tree; paths and line numbers below are in the extracted files). Later patches touching it: 0127 (Kconfig select lines only; no facts used). 0132 and 0139 were checked and do not touch drivers/media/platform/cix.
- Licence: every driver file carries SPDX GPL-2.0-only; the header text also contains an Arm Technology (China) "confidential and proprietary" notice and an "or (at your option) any later version" GPL paragraph, which contradict each other and the SPDX tag. No file is dual-licensed or MIT. Only facts (offsets, bits, values, layouts, sequences, ABI numbers) were taken; no code or comments were copied or reworded.
- Sources read:
  - if/mvx_mmu.h 40-80 (page size, PTE layout, ATTR and AP encodings)
  - if/mvx_mmu.c 52-90 (PA/VA widths, index widths, field shifts, two levels), 107-165 (index and field extraction), 174-235 (table walk; L1 link attributes), 238-310 (map/unmap, cache clean per entry), 325-395 (no-access dummy page at 0x1000), 602-720 (table/page allocation, DMA mapping, zero-page clean), 1055-1090 (reservation of capacity), 1144-1170 (external L2), 1276-1283 (PTE encoding)
  - if/mvx_session.c 3000-3010 (MMU_CTRL value)
  - if/mvx_firmware.c 40-60 (text at +0x1000), 85-150 (unmap 4 MiB per instance), 152-213 (per-instance mapping, shared vs private BSS), 215-300 (allocation, single text copy), 587-625 (instance 0 always mapped)
  - if/mvx_firmware_v2.c 672-756 (protocol-2 regions), 3576-3740 (queue pages ATTR 1, print buffer ATTR 3)
  - if/mvx_firmware_v3.c 35-120 (protocol-3 regions)
  - external/fw_v2/mve_protocol_def.h 95-200 and external/fw_v3/mve_protocol_def.h 95-205 (memory-region constants, queue addresses, print buffer)
  - if/mvx_buffer.c 197-420 (user-buffer mapping attributes, next-fit VA), if/mvx_session.c 2535-2578 (region choice per port), 4458-4550 (frame-buffer VA reset on sequence parameters)
  - if/mvx_firmware_v2.c 3172-3435 (RPC allocation VA search and flush)
