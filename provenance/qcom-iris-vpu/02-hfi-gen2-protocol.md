# Provenance: `specs/qcom-iris-vpu/02-hfi-gen2-protocol.md`

## Baseline Commits and Empirical Sources
- **Mainline Linux:** `d135ab453a7400a34930bdda1ef4bdc8acfac80e` (`/home/jkane/Projects/OS/linux`)
- **Radxa Linux Kernel (`linux-7.0.11`):** `radxa/kernel@c00e1e17e65104c8143a8f3d559ef723ae5824d3` (`/home/jkane/.gemini/antigravity-cli/brain/2761ccec-ec33-4e2c-9581-a942f4d0f19e/scratch/radxa-kernel`)
- **Empirical Board Documentation (AbyssBSD):** `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/`
- **Extraction Date:** `2026-10-27`

## Source Files Read
1. `/home/jkane/.gemini/antigravity-cli/brain/2761ccec-ec33-4e2c-9581-a942f4d0f19e/scratch/radxa-kernel/drivers/media/platform/qcom/iris/iris_hfi_queue.h` (lines 1–183) and `iris_hfi_queue.c` (lines 1–318)
   - Extracted the 288-byte `iris_hfi_queue_table_header` layout, the 56-byte `iris_hfi_queue_header` layout, the 3 circular queues (`IFACEQ_NUMQ = 3`, `IFACEQ_QUEUE_SIZE = 1048576` bytes = `262144` dwords each), SFR 4 KiB layout, and dword-indexed circular queue read/write logic.
2. `/home/jkane/.gemini/antigravity-cli/brain/2761ccec-ec33-4e2c-9581-a942f4d0f19e/scratch/radxa-kernel/drivers/media/platform/qcom/iris/iris_hfi_gen2_packet.h` (lines 1–126) and `iris_hfi_gen2_packet.c` (lines 1–297)
   - Extracted the 32-byte `iris_hfi_header` structure (`size`, `session_id`, `header_id`, `reserved[4]`, `num_packets`), the 32-byte `iris_hfi_packet` structure (`size`, `type`, `flags`, `payload_info`, `port`, `packet_id`, `reserved[2]`, `payload[]`), the 64-byte `iris_hfi_buffer` structure, and `HFI_CMD_SYSTEM_INIT` UBWC packet construction.
3. `/home/jkane/.gemini/antigravity-cli/brain/2761ccec-ec33-4e2c-9581-a942f4d0f19e/scratch/radxa-kernel/drivers/media/platform/qcom/iris/iris_hfi_gen2_defines.h` (lines 1–261)
   - Extracted all `HFI_CMD_*`, `HFI_PROP_*`, `HFI_PAYLOAD_*`, `HFI_PORT_*`, `HFI_FW_FLAGS_*`, `HFI_HOST_FLAGS_*`, `HFI_BUF_HOST_FLAG_*`, `HFI_BUF_FW_FLAG_*`, and `HFI_COLOR_FMT_*` hex constants.
4. `/home/jkane/.gemini/antigravity-cli/brain/2761ccec-ec33-4e2c-9581-a942f4d0f19e/scratch/radxa-kernel/drivers/media/platform/qcom/iris/iris_hfi_gen2_command.c` (lines 1–1429) and `iris_hfi_gen2_response.c` (lines 1–1026)
   - Extracted session open/close, port configuration, property subscription, settings change (`DRC`) parsing, drain, resume, and buffer command/response sequences.
5. `/home/jkane/.gemini/antigravity-cli/brain/2761ccec-ec33-4e2c-9581-a942f4d0f19e/scratch/radxa-kernel/drivers/media/platform/qcom/iris/iris_vpu_buffer.h` (lines 1–294) and `iris_vpu_buffer.c` (lines 1–2274)
   - Extracted internal scratch/persist buffer types (`BIN`, `ARP`, `COMV`, `NON_COMV`, `LINE`, `DPB`, `PERSIST`, `VPSS`) and sizing formulas for H.264, HEVC, and VP9.
