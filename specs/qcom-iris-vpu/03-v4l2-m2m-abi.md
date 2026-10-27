# V4L2 Memory-to-Memory (`M2M`) Userspace ABI

## 1. Device Nodes and V4L2 Capabilities

### 1.1 Registered Video Device Nodes
- **[ABI]** The driver registers two Video4Linux2 (`V4L2`) character devices (`VFL_TYPE_VIDEO`):
  1. **Stateful Video Decoder:** Card/Entity name `"qcom-iris-decoder"`, driver name `"qcom-iris"`, bus info `"platform:aa00000.video-codec"`.
  2. **Stateful Video Encoder:** Card/Entity name `"qcom-iris-encoder"`, driver name `"qcom-iris"`, bus info `"platform:aa00000.video-codec"`.
- **[ABI]** Both video nodes report `device_caps = V4L2_CAP_VIDEO_M2M_MPLANE | V4L2_CAP_STREAMING` (`0x00004000 | 0x04000000 = 0x04004000`) in `VIDIOC_QUERYCAP`.
- **[POLICY]** Each video device supports up to `16` concurrent codec instances (`MAX_SESSIONS = 16`), subject to a hardware macroblock processing budget of `2,034,000` macroblocks per second across all active instances (equivalent to 4096×2160 @ 60 fps or 8192×4320 @ 30 fps).

### 1.2 Supported Queue Types and Memory Modes
- **[ABI]** Both decoder and encoder instances use multi-planar V4L2 buffer types exclusively:
  - **Input Queue (`OUTPUT`):** `V4L2_BUF_TYPE_VIDEO_OUTPUT_MPLANE` (`10`). Accepts `V4L2_MEMORY_MMAP` (`1`), `V4L2_MEMORY_USERPTR` (`2`), and `V4L2_MEMORY_DMABUF` (`4`).
  - **Output Queue (`CAPTURE`):** `V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE` (`9`). Accepts `V4L2_MEMORY_MMAP` (`1`), `V4L2_MEMORY_USERPTR` (`2`), and `V4L2_MEMORY_DMABUF` (`4`).

---

## 2. Formats, Resolutions, and Buffer Geometry Formulas

### 2.1 Supported FourCC Formats and Resolution Limits

| Instance Type | Queue | V4L2 FourCC | HFI Format / Codec ID | Num Planes | Resolution Range (Width × Height, Step) | Tag |
|---|---|---|---|---|---|---|
| **Decoder** | `OUTPUT_MPLANE` | `V4L2_PIX_FMT_H264` (`'H264'`) | `HFI_CODEC_DEC_AVC` (`1`) | `1` | `96×96` to `8192×8192`, step `2×2` (default `1280×720`) | **[ABI]** |
| **Decoder** | `OUTPUT_MPLANE` | `V4L2_PIX_FMT_HEVC` (`'HEVC'`) | `HFI_CODEC_DEC_HEVC` (`3`) | `1` | `96×96` to `8192×8192`, step `2×2` (default `1280×720`) | **[ABI]** |
| **Decoder** | `OUTPUT_MPLANE` | `V4L2_PIX_FMT_VP9` (`'VP90'`) | `HFI_CODEC_DEC_VP9` (`5`) | `1` | `96×96` to `8192×8192`, step `2×2` (default `1280×720`) | **[ABI]** |
| **Decoder** | `CAPTURE_MPLANE` | `V4L2_PIX_FMT_NV12` (`'NV12'`) | `HFI_COLOR_FMT_NV12` (`1`) | `1` | `96×96` to `8192×8192`, step `2×2` (8-bit linear Y/UV single-FD plane) | **[ABI]** |
| **Decoder** | `CAPTURE_MPLANE` | `V4L2_PIX_FMT_QC08C` (`'Q08C'`) | `HFI_COLOR_FMT_NV12_UBWC` (`2`) | `1` | `96×96` to `8192×8192`, step `2×2` (8-bit Qualcomm UBWC compressed; native/preferred format) | **[ABI]** |
| **Decoder** | `CAPTURE_MPLANE` | `V4L2_PIX_FMT_QC10C` (`'Q10C'`) | `HFI_COLOR_FMT_NV12_TP10_UBWC` (`3`) | `1` | `96×96` to `8192×8192`, step `2×2` (10-bit TP10 Qualcomm UBWC compressed; enumerated for 10-bit streams) | **[ABI]** |
| **Encoder** | `OUTPUT_MPLANE` | `V4L2_PIX_FMT_NV12` (`'NV12'`) | `HFI_COLOR_FMT_NV12` (`1`) | `1` | `128×128` to `4096×2160`, step `2×2` (default `1280×720`) | **[ABI]** |
| **Encoder** | `OUTPUT_MPLANE` | `V4L2_PIX_FMT_QC08C` (`'Q08C'`) | `HFI_COLOR_FMT_NV12_UBWC` (`2`) | `1` | `128×128` to `4096×2160`, step `2×2` | **[ABI]** |
| **Encoder** | `CAPTURE_MPLANE` | `V4L2_PIX_FMT_H264` (`'H264'`) | `HFI_CODEC_ENC_AVC` (`2`) | `1` | `128×128` to `4096×2160`, step `2×2` | **[ABI]** |
| **Encoder** | `CAPTURE_MPLANE` | `V4L2_PIX_FMT_HEVC` (`'HEVC'`) | `HFI_CODEC_ENC_HEVC` (`4`) | `1` | `128×128` to `4096×2160`, step `2×2` | **[ABI]** |

### 2.2 Exact Stride, Scanline, and Plane Size Formulas (`sizeimage` and `bytesperline`)
- **[HW]** The VPU hardware imposes strict stride and scanline alignment requirements on raw frame buffers (`num_planes = 1`, where Luma and Chroma reside in a single contiguous allocation). In the formulas below, `round_up(x, a)` denotes rounding `x` up to the next multiple of `a`, and `ceil_div(x, y)` denotes `(x + y - 1) / y`:
  1. **Linear `V4L2_PIX_FMT_NV12` (8-bit 4:2:0 semi-planar):**
     - `y_stride = round_up(width, 128)` (`bytesperline` reported in `v4l2_plane_pix_format`).
     - `y_scanline = round_up(height, 32)`.
     - `uv_stride = round_up(width, 128)`.
     - `uv_scanline = round_up((height + 1) >> 1, 16)`.
     - `y_plane_bytes = y_stride * y_scanline`.
     - `uv_plane_bytes = uv_stride * uv_scanline`.
     - **`sizeimage = round_up(y_plane_bytes + uv_plane_bytes, 4096)`** (with a minimum of `4096` bytes).
     - Note: When sending `HFI_CMD_BUFFER` to the VPU for a linear `NV12` decoder output buffer, `HFI_BUFFER.buffer_size` is set to `y_stride * y_scanline * 3 / 2`.
  2. **UBWC 8-bit `V4L2_PIX_FMT_QC08C` (`NV12_UBWC`, tile size `32×8`):**
     - `y_stride = round_up(width, 128)` (`bytesperline`).
     - `y_scanline = round_up(height, 32)`.
     - `y_data_bytes = round_up(y_stride * y_scanline, 4096)`.
     - `y_meta_stride = round_up(ceil_div(width, 32), 64)`.
     - `y_meta_scanline = round_up(ceil_div(height, 8), 16)`.
     - `y_meta_bytes = round_up(y_meta_stride * y_meta_scanline, 4096)`.
     - `uv_stride = round_up(width, 128)`.
     - `uv_scanline = round_up((height + 1) >> 1, 32)`.
     - `uv_data_bytes = round_up(uv_stride * uv_scanline, 4096)`.
     - `uv_meta_stride = round_up(ceil_div((width + 1) >> 1, 16), 64)`.
     - `uv_meta_scanline = round_up(ceil_div((height + 1) >> 1, 8), 16)`.
     - `uv_meta_bytes = round_up(uv_meta_stride * uv_meta_scanline, 4096)`.
     - **`sizeimage = round_up(y_data_bytes + y_meta_bytes + uv_data_bytes + uv_meta_bytes, 4096)`**.
  3. **UBWC 10-bit `V4L2_PIX_FMT_QC10C` (`TP10_UBWC`, tile size `48×4`):**
     - `y_stride = round_up(round_up(width, 192) * 4 / 3, 256)` (`bytesperline`).
     - `y_scanline = round_up(height, 16)`.
     - `y_data_bytes = round_up(y_stride * y_scanline, 4096)`.
     - `y_meta_stride = round_up(ceil_div(width, 48), 64)`.
     - `y_meta_scanline = round_up(ceil_div(height, 4), 16)`.
     - `y_meta_bytes = round_up(y_meta_stride * y_meta_scanline, 4096)`.
     - `uv_stride = round_up(round_up(width, 192) * 4 / 3, 256)`.
     - `uv_scanline = round_up((height + 1) >> 1, 16)`.
     - `uv_data_bytes = round_up(uv_stride * uv_scanline, 4096)`.
     - `uv_meta_stride = round_up(ceil_div((width + 1) >> 1, 24), 64)`.
     - `uv_meta_scanline = round_up(ceil_div((height + 1) >> 1, 4), 16)`.
     - `uv_meta_bytes = round_up(uv_meta_stride * uv_meta_scanline, 4096)`.
     - **`sizeimage = round_up(y_data_bytes + y_meta_bytes + uv_data_bytes + uv_meta_bytes, 4096)`**.
  4. **Compressed Bitstream Buffer Size (`H264` / `HEVC` / `VP9`):**
     - `bytesperline = 0`.
     - For Decoder `OUTPUT_MPLANE`: `mbs = ceil_div(width, 16) * ceil_div(height, 16)`. Base size = `(mbs * 16 * 16 * 3 / 2) / 2` (half of an uncompressed 4:2:0 frame); if `mbs > 3600` (above 720p), divide by `2` again (quarter of a frame). Clamp to at least `min(base_size, 4 MiB)` and align to `4096` bytes. If userspace requests a larger `sizeimage` in `S_FMT`, honor userspace's larger value (aligned to `4096`).
     - For Encoder `CAPTURE_MPLANE`: Minimum `2 * 1024 * 1024` (2 MiB) or resolution-scaled bitstream size aligned to `4096` bytes.

---

## 3. Selection / Cropping Targets (`VIDIOC_G_SELECTION` / `VIDIOC_S_SELECTION`)

- **[ABI]** **Decoder (`V4L2_BUF_TYPE_VIDEO_CAPTURE` / `CAPTURE_MPLANE`):**
  - Supports read-only `VIDIOC_G_SELECTION` for `V4L2_SEL_TGT_COMPOSE` (`0x0100`), `V4L2_SEL_TGT_COMPOSE_DEFAULT` (`0x0101`), and `V4L2_SEL_TGT_COMPOSE_BOUNDS` (`0x0102`), returning the visible rectangle `(left, top, width, height)` reported by the firmware via `HFI_PROP_CROP_OFFSETS`.
- **[ABI]** **Encoder (`V4L2_BUF_TYPE_VIDEO_OUTPUT` / `OUTPUT_MPLANE`):**
  - Supports `VIDIOC_G_SELECTION` and `VIDIOC_S_SELECTION` for `V4L2_SEL_TGT_CROP` (`0x0000`), `V4L2_SEL_TGT_CROP_DEFAULT` (`0x0001`), and `V4L2_SEL_TGT_CROP_BOUNDS` (`0x0002`), rounding `width` and `height` to a multiple of `2` within the `OUTPUT_MPLANE` coded frame dimensions.

---

## 4. V4L2 Controls (`VIDIOC_QUERYCTRL`, `VIDIOC_G_EXT_CTRLS`, `VIDIOC_S_EXT_CTRLS`)

### 4.1 Decoder Controls

| V4L2 Control ID | Hex ID | Type | Range / Values | Default | HFI Property | Tag |
|---|---|---|---|---|---|---|
| `V4L2_CID_MIN_BUFFERS_FOR_CAPTURE` | `0x00980927` | `INTEGER` (RO) | `1`..`32` | `4` (or FW `MIN_OUTPUT_COUNT`) | `HFI_PROP_BUFFER_FW_MIN_OUTPUT_COUNT` | **[ABI]** |
| `V4L2_CID_MPEG_VIDEO_H264_PROFILE` | `0x00990a6b` | `MENU` | Baseline (`0`), Constrained Baseline (`1`), Main (`2`), High (`4`), Constrained High (`15`) | High (`4`) | `HFI_PROP_PROFILE` | **[ABI]** |
| `V4L2_CID_MPEG_VIDEO_H264_LEVEL` | `0x00990a67` | `MENU` | `1.0` (`0`) .. `6.2` (`19`) | `6.1` (`18`) | `HFI_PROP_LEVEL` | **[ABI]** |
| `V4L2_CID_MPEG_VIDEO_HEVC_PROFILE` | `0x00990b67` | `MENU` | Main (`0`), Main Still Picture (`1`), Main 10 (`2`) | Main (`0`) | `HFI_PROP_PROFILE` | **[ABI]** |
| `V4L2_CID_MPEG_VIDEO_HEVC_LEVEL` | `0x00990b68` | `MENU` | `1.0` (`0`) .. `6.2` (`12`) | `6.1` (`11`) | `HFI_PROP_LEVEL` | **[ABI]** |
| `V4L2_CID_MPEG_VIDEO_VP9_PROFILE` | `0x00990b30` | `MENU` | Profile 0 (`0`), Profile 2 (`2`) | Profile 0 (`0`) | `HFI_PROP_PROFILE` | **[ABI]** |
| `V4L2_CID_MPEG_VIDEO_VP9_LEVEL` | `0x00990b31` | `MENU` | `1.0` (`0`) .. `6.0` (`11`) | `6.0` (`11`) | `HFI_PROP_LEVEL` | **[ABI]** |
| `V4L2_CID_MPEG_VIDEO_DEC_DISPLAY_DELAY` | `0x00990953` | `INTEGER` | `0`..`16383` (`0` = low-latency immediate output when enabled) | `0` | Low-latency decode mode | **[ABI]** |
| `V4L2_CID_MPEG_VIDEO_DEC_DISPLAY_DELAY_ENABLE` | `0x00990954` | `BOOLEAN` | `0`..`1` | `0` | Low-latency decode enable | **[ABI]** |

### 4.2 Encoder Controls

| V4L2 Control ID | Hex ID | Type | Range / Values | Default | HFI Property | Tag |
|---|---|---|---|---|---|---|
| `V4L2_CID_MIN_BUFFERS_FOR_OUTPUT` | `0x00980928` | `INTEGER` (RO) | `1`..`32` | `4` | `HFI_PROP_BUFFER_FW_MIN_OUTPUT_COUNT` | **[ABI]** |
| `V4L2_CID_MPEG_VIDEO_BITRATE_MODE` | `0x009909ce` | `MENU` | `VBR` (`0`), `CBR` (`1`), `CQ` (`2`) | `VBR` (`0`) | `HFI_PROP_BITRATE_CONTROL` | **[ABI]** |
| `V4L2_CID_MPEG_VIDEO_BITRATE` | `0x009909cf` | `INTEGER` | `1`..`160,000,000` bps | `64,000` | `HFI_PROP_TOTAL_BITRATE` | **[ABI]** |
| `V4L2_CID_MPEG_VIDEO_BITRATE_PEAK` | `0x009909d0` | `INTEGER` | `1`..`160,000,000` bps | `64,000` | `HFI_PROP_TOTAL_PEAK_BITRATE` | **[ABI]** |
| `V4L2_CID_MPEG_VIDEO_FRAME_RC_ENABLE` | `0x009909d7` | `BOOLEAN` | `0`..`1` | `1` | `HFI_PROP_BITRATE_CONTROL` (`RC_OFF` if `0`) | **[ABI]** |
| `V4L2_CID_MPEG_VIDEO_GOP_SIZE` | `0x009909cb` | `INTEGER` | `0`..`65535` | `30` | `HFI_PROP_MAX_GOP_FRAMES` | **[ABI]** |
| `V4L2_CID_MPEG_VIDEO_B_FRAMES` | `0x009909ca` | `INTEGER` | `0`..`2` | `0` | `HFI_PROP_MAX_B_FRAMES` | **[ABI]** |
| `V4L2_CID_MPEG_VIDEO_FORCE_KEY_FRAME` | `0x009909e5` | `BUTTON` | Trigger | — | `HFI_PROP_REQUEST_SYNC_FRAME` | **[ABI]** |
| `V4L2_CID_MPEG_VIDEO_HEADER_MODE` | `0x009909d8` | `MENU` | `SEPARATE` (`0`), `JOINED_WITH_1ST_FRAME` (`1`) | `JOINED` (`1`) | `HFI_PROP_SEQ_HEADER_MODE` | **[ABI]** |
| `V4L2_CID_MPEG_VIDEO_PREPEND_SPSPPS_TO_IDR` | `0x00990a92` | `BOOLEAN` | `0`..`1` | `0` | `HFI_PROP_SYNC_FRAME_SEQUENCE_HEADER` | **[ABI]** |
| `V4L2_CID_MPEG_VIDEO_H264_ENTROPY_MODE` | `0x00990a65` | `MENU` | `CAVLC` (`0`), `CABAC` (`1`) | `CABAC` (`1`) | `HFI_PROP_CABAC_SESSION` | **[ABI]** |
| `V4L2_CID_MPEG_VIDEO_H264_8X8_TRANSFORM` | `0x00990a69` | `BOOLEAN` | `0`..`1` | `1` | `HFI_PROP_8X8_TRANSFORM` | **[ABI]** |
| `V4L2_CID_MPEG_VIDEO_H264_I_FRAME_QP` / `P` / `B` / `MIN` / `MAX` | `0x00990a62`.. | `INTEGER` | `0`..`51` (`-1`..`51`) | `26` / `28` / `30` | `HFI_PROP_QP_PACKED` / `MIN_QP_PACKED` / `MAX_QP_PACKED` | **[ABI]** |
| `V4L2_CID_MPEG_VIDEO_HEVC_I_FRAME_QP` / `P` / `B` / `MIN` / `MAX` | `0x00990b5d`.. | `INTEGER` | `0`..`51` (`-1`..`51`) | `26` / `28` / `30` | `HFI_PROP_QP_PACKED` / `MIN_QP_PACKED` / `MAX_QP_PACKED` | **[ABI]** |
| `V4L2_CID_MPEG_VIDEO_HEVC_TIER` | `0x00990b69` | `MENU` | `MAIN` (`0`), `HIGH` (`1`) | `HIGH` (`1`) | `HFI_PROP_HEVC_TIER` | **[ABI]** |
| `V4L2_CID_ROTATE` | `0x00980922` | `INTEGER` | `0`, `90`, `180`, `270` (step `90`) | `0` | VPSS rotation | **[ABI]** |
| `V4L2_CID_HFLIP` / `V4L2_CID_VFLIP` | `0x00980914`..`15` | `BOOLEAN` | `0`..`1` | `0` | VPSS flip | **[ABI]** |

---

## 5. Stateful M2M State Machine, Events, and Commands

### 5.1 V4L2 Events (`VIDIOC_SUBSCRIBE_EVENT` / `VIDIOC_DQEVENT`)
- **[ABI]** Both decoder and encoder instances support subscribing to:
  - `V4L2_EVENT_EOS` (`2`): Emitted when the last buffer of a drain completes.
  - `V4L2_EVENT_SOURCE_CHANGE` (`5`, decoder only): Emitted with `changes = V4L2_EVENT_SRC_CH_RESOLUTION` (`1`) when the firmware reports `HFI_CMD_SETTINGS_CHANGE` on the bitstream port.
  - `V4L2_EVENT_CTRL` (`3`): Standard control change events.

### 5.2 Decoder Drain (`VIDIOC_DECODER_CMD`) and Encoder Stop (`VIDIOC_ENCODER_CMD`)
- **[ABI]** **Decoder Commands (`VIDIOC_TRY_DECODER_CMD` / `VIDIOC_DECODER_CMD`):**
  - `V4L2_DEC_CMD_STOP` (`1`): Initiates a drain sequence. If the session is actively streaming, the driver sends `HFI_CMD_DRAIN` (`0x01000006`) on `HFI_PORT_BITSTREAM`. Firmware processes all remaining queued bitstream buffers and sets `HFI_BUF_FW_FLAG_LAST` (`0x01000000`) on the final decoded capture buffer. The driver sets `V4L2_BUF_FLAG_LAST` (`0x00100000`) on that V4L2 capture buffer (or on a 0-byte dummy capture buffer if no frames remain) and queues `V4L2_EVENT_EOS`.
  - `V4L2_DEC_CMD_START` (`0`): Resumes decoding after a completed drain (`CMD_STOP`), transitioning the instance back from `DRAINING`/`DRC` to `STREAMING`.
- **[ABI]** **Encoder Commands (`VIDIOC_TRY_ENCODER_CMD` / `VIDIOC_ENCODER_CMD`):**
  - `V4L2_ENC_CMD_STOP` (`1`): Sends `HFI_CMD_DRAIN` (`0x01000006`) on `HFI_PORT_RAW`. Firmware encodes all remaining raw input frames, marks the final bitstream capture buffer with `HFI_BUF_FW_FLAG_LAST`, and the driver sets `V4L2_BUF_FLAG_LAST` and emits `V4L2_EVENT_EOS`.
  - `V4L2_ENC_CMD_START` (`0`): Resumes encoding after a completed drain.

### 5.3 Buffer Flags and Metadata Propagation (`VIDIOC_QBUF` / `VIDIOC_DQBUF`)
- **[ABI]** For every buffer queued via `VIDIOC_QBUF`:
  - The 64-bit V4L2 nanosecond timestamp (`v4l2_buffer.timestamp`) is converted to **microseconds** (`timestamp_ns / 1000`) in `HFI_BUFFER.timestamp` (`+0x20`) and returned unchanged by the firmware on the corresponding decoded/encoded `CAPTURE_MPLANE` buffer (`timestamp_us * 1000` back to nanoseconds), with `V4L2_BUF_FLAG_TSTAMP_SRC_COPY` (`0x00010000`).
  - Picture type reported by firmware via `HFI_PROP_PICTURE_TYPE` is mapped to V4L2 buffer flags on dequeued `CAPTURE_MPLANE` buffers:
    - `I` (`0x1`), `IDR` (`0x10`), or `CRA` (`0x1000`) → `V4L2_BUF_FLAG_KEYFRAME` (`0x00000008`).
    - `P` (`0x2`) → `V4L2_BUF_FLAG_PFRAME` (`0x00000010`).
    - `B` (`0x4`) → `V4L2_BUF_FLAG_BFRAME` (`0x00000020`).
  - If firmware sets `HFI_BUF_FW_FLAG_CORRUPT` (`0x20000000`), the driver sets `V4L2_BUF_FLAG_ERROR` (`0x00000040`) on the dequeued V4L2 buffer.
