# Provenance: `specs/qcom-iris-vpu/03-v4l2-m2m-abi.md`

## Baseline Commits and Empirical Sources
- **Mainline Linux:** `d135ab453a7400a34930bdda1ef4bdc8acfac80e` (`/home/jkane/Projects/OS/linux`)
- **Radxa Linux Kernel (`linux-7.0.11`):** `radxa/kernel@c00e1e17e65104c8143a8f3d559ef723ae5824d3` (`/home/jkane/.gemini/antigravity-cli/brain/2761ccec-ec33-4e2c-9581-a942f4d0f19e/scratch/radxa-kernel`)
- **Empirical Board Documentation (AbyssBSD):** `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/`
- **Extraction Date:** `2026-10-27`

## Source Files Read
1. `/home/jkane/.gemini/antigravity-cli/brain/2761ccec-ec33-4e2c-9581-a942f4d0f19e/scratch/radxa-kernel/drivers/media/platform/qcom/iris/iris_vidc.c` (lines 1–727)
   - Extracted V4L2 video device registration (`qcom-iris-decoder`, `qcom-iris-encoder`), `V4L2_CAP_VIDEO_M2M_MPLANE | V4L2_CAP_STREAMING`, ioctl callbacks (`ENUM_FMT`, `G_FMT`, `S_FMT`, `TRY_FMT`, `G_SELECTION`, `S_SELECTION`, `TRY_DECODER_CMD`, `DECODER_CMD`, `TRY_ENCODER_CMD`, `ENCODER_CMD`, `SUBSCRIBE_EVENT`).
2. `/home/jkane/.gemini/antigravity-cli/brain/2761ccec-ec33-4e2c-9581-a942f4d0f19e/scratch/radxa-kernel/drivers/media/platform/qcom/iris/iris_vdec.c` (lines 1–533) and `iris_venc.c` (lines 1–612)
   - Extracted supported decoder and encoder formats (`H264`, `HEVC`, `VP9`, `NV12`, `QC08C`, `QC10C`), resolution limits, selection crop/compose handling, and `DECODER_CMD` / `ENCODER_CMD` drain and start logic.
3. `/home/jkane/.gemini/antigravity-cli/brain/2761ccec-ec33-4e2c-9581-a942f4d0f19e/scratch/radxa-kernel/drivers/media/platform/qcom/iris/iris_buffer.c` (lines 1–998) and `iris_vpu_buffer.c` (lines 1–2274)
   - Extracted exact linear `NV12`, `QC08C` (8-bit UBWC `32×8` tile), `QC10C` (10-bit TP10 UBWC `48×4` tile), and bitstream buffer stride, scanline, metadata, and `sizeimage` calculations.
4. `/home/jkane/.gemini/antigravity-cli/brain/2761ccec-ec33-4e2c-9581-a942f4d0f19e/scratch/radxa-kernel/drivers/media/platform/qcom/iris/iris_ctrls.c` (lines 1–1642) and `iris_platform_vpu2.c` (lines 1–168)
   - Extracted V4L2 control definitions, ranges, defaults, and HFI property mappings for SC8280XP VPU 2.0.
5. `/home/jkane/.gemini/antigravity-cli/brain/2761ccec-ec33-4e2c-9581-a942f4d0f19e/scratch/radxa-kernel/drivers/media/platform/qcom/iris/iris_vb2.c` (lines 1–354) and `iris_state.c` (lines 1–281)
   - Extracted streaming start/stop, DRC state transitions, timestamp conversion (`ns` ↔ `us`), and V4L2 buffer flag mappings.
