# Provenance: `specs/qcom-fastrpc/01-glink-rpc-and-smmu-sessions.md`

## Baseline Commits and Empirical Sources
- **Mainline Linux:** `d135ab453a7400a34930bdda1ef4bdc8acfac80e` (`/home/jkane/Projects/OS/linux`)
- **Radxa Linux Kernel (`linux-7.0.11`):** `radxa/kernel@c00e1e17e65104c8143a8f3d559ef723ae5824d3` (`/home/jkane/.gemini/antigravity-cli/brain/2761ccec-ec33-4e2c-9581-a942f4d0f19e/scratch/radxa-kernel`)
- **Empirical Board Documentation (AbyssBSD):** `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/`
- **Extraction Date:** `2026-10-27`

## Source Files Read
1. `/home/jkane/Projects/OS/linux/drivers/misc/fastrpc.c` (lines 31–2832)
   - Lines 31–135: Extracted domain IDs (`ADSP=0`..`GDSP=4`), `FASTRPC_MAX_SESSIONS = 14`, `FASTRPC_ALIGN = 128`, `FASTRPC_MAX_FDLIST = 16`, `FASTRPC_MAX_CRCLIST = 64`, `FASTRPC_CTX_MAX = 256`, `FASTRPC_INIT_HANDLE = 1`, `FASTRPC_DSP_UTILITIES_HANDLE = 2`, `FASTRPC_MAX_STATIC_HANDLE = 20`, `FASTRPC_CTXID_MASK`, `FASTRPC_CTXID_SEQ_MASK`, `FASTRPC_DSP_PD_NOTIFY_CTX = 0xABCDABCD`, scalar bit macros (`REMOTE_SCALARS_*`, `FASTRPC_BUILD_SCALARS`), Remote Method IDs (`FASTRPC_RMID_INIT_*`), Protection Domain IDs (`ROOT_PD=0`, `USER_PD=1`, `SENSORS_PD=2`), and `FASTRPC_POLL_RESPONSE = 0xdecaf`.
   - Lines 136–210: Extracted wire structures `fastrpc_phy_page` (16 B), `fastrpc_invoke_buf` (8 B), `fastrpc_remote_dmahandle` (12 B), `fastrpc_remote_buf` (16 B), `fastrpc_msg` (40 B), `fastrpc_invoke_rsp` (16 B), and internal init request/response payloads.
   - Lines 362–379, 2470–2482: Extracted `fastrpc_ipa_to_dma_addr`, `fastrpc_sid_offset`, and `default_soc_data` (`sid_pos = 32`, `dma_addr_bits = 32`).
   - Lines 644–692, 984–1434: Extracted overlap sorting/deduplication (`fastrpc_get_buff_overlaps`), metadata size calculation (`fastrpc_get_meta_size`), payload size calculation (`fastrpc_get_payload_size`), argument marshaling (`fastrpc_get_args`, `fastrpc_put_args`), GLINK transmit (`fastrpc_invoke_send`), and poll mode completion (`poll_for_remote_response`).
   - Lines 1453–2275, 2719–2783: Extracted internal RPC calls (`fastrpc_init_create_process`, `fastrpc_init_create_static_process`, `fastrpc_init_attach`, `fastrpc_release_current_dsp_process`, `fastrpc_get_info_from_dsp`, `fastrpc_req_mmap`, `fastrpc_req_munmap`, `fastrpc_req_mem_map`, `fastrpc_req_mem_unmap`) and GLINK response callback (`fastrpc_rpmsg_callback`).
2. `/home/jkane/.gemini/antigravity-cli/brain/2761ccec-ec33-4e2c-9581-a942f4d0f19e/scratch/radxa-kernel/arch/arm64/boot/dts/qcom/sc8280xp.dtsi` (lines 7481–7630)
   - Extracted `compute-cb@1`..`compute-cb@14` session numbers (`1`..`14`) and Apps SMMU stream IDs (`0x3181`..`0x3188`, `0x318b`..`0x318f`, mask `0x0420`).
