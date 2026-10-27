# Provenance: `specs/qcom-fastrpc/02-fastrpc-uapi-and-qnn-compat.md`

## Baseline Commits and Empirical Sources
- **Mainline Linux:** `d135ab453a7400a34930bdda1ef4bdc8acfac80e` (`/home/jkane/Projects/OS/linux`)
- **Radxa Linux Kernel (`linux-7.0.11`):** `radxa/kernel@c00e1e17e65104c8143a8f3d559ef723ae5824d3` (`/home/jkane/.gemini/antigravity-cli/brain/2761ccec-ec33-4e2c-9581-a942f4d0f19e/scratch/radxa-kernel`)
- **Empirical Board Documentation (AbyssBSD):** `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/`
- **Extraction Date:** `2026-10-27`

## Source Files Read
1. `/home/jkane/Projects/OS/linux/include/uapi/misc/fastrpc.h` (lines 1–173)
   - Extracted all 13 `FASTRPC_IOCTL_*` definitions (`_IOWR('R', 1..13, ...)`), `enum fastrpc_map_flags` (`FASTRPC_MAP_STATIC = 0`, `FASTRPC_MAP_RESERVED = 1`, `FASTRPC_MAP_FD = 2`, `FASTRPC_MAP_FD_DELAYED = 3`, `FASTRPC_MAP_FD_NOMAP = 16`), `enum fastrpc_proc_attr` (`FASTRPC_MODE_DEBUG`..`FASTRPC_MODE_PRIVILEGED`), `FASTRPC_ATTR_SECUREMAP = 1`, `FASTRPC_POLL_MODE = 1`, and exact UAPI structure layouts (`fastrpc_invoke_args`, `fastrpc_invoke`, `fastrpc_init_create`, `fastrpc_init_create_static`, `fastrpc_alloc_dma_buf`, `fastrpc_req_mmap`, `fastrpc_mem_map`, `fastrpc_req_munmap`, `fastrpc_mem_unmap`, `fastrpc_ioctl_set_option`, `fastrpc_ioctl_capability`).
2. `/home/jkane/Projects/OS/linux/drivers/misc/fastrpc.c` (lines 1435–2657)
   - Extracted `is_session_rejected` unsigned PD enforcement rules, `fastrpc_device_open` / `fastrpc_device_release` / `fastrpc_device_ioctl` dispatch, `fastrpc_dmabuf_alloc` DMA-BUF export operations, `fastrpc_set_option`, `fastrpc_get_dsp_info`, and `/dev/fastrpc-cdsp` + `/dev/fastrpc-cdsp-secure` dual device registration.
3. `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/npu.md` (lines 1–147)
   - Extracted QNN HTP / `libcdsprpc.so` userspace runtime interaction and unsigned PD requirements on Dragon Q8B.
