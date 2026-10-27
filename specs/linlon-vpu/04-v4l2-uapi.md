# 04: V4L2 Userspace Interface

This section covers the userspace ABI the vendor driver offers, limited to
what existing userspace (ffmpeg and GStreamer V4L2 memory-to-memory
plugins, with Sky1 patches) relies on. Standard V4L2 semantics apply
unless stated otherwise. All items are **[ABI]** unless tagged otherwise.

## 4.1 Device nodes and capabilities

| Node name (`video_device` name) | Role |
|---|---|
| `mvxdec` | Stateful decoder |
| `mvxenc` | Stateful encoder |

- Both are `/dev/videoN` nodes (numbers assigned dynamically) under one
  V4L2 device. Userspace finds them by name or by probing formats.
- `VIDIOC_QUERYCAP`: driver `"mvx"`, card `"Linlon Video device"`, bus
  info `"platform:mvx"`. Device caps = `V4L2_CAP_VIDEO_M2M |
  V4L2_CAP_VIDEO_M2M_MPLANE | V4L2_CAP_EXT_PIX_FORMAT | V4L2_CAP_STREAMING`.
- Both single-planar and multi-planar buffer types are accepted on both
  queues. OUTPUT is the input to the engine (bitstream for decode, raw for
  encode). CAPTURE is the result.
- Memory types: `MMAP`, `USERPTR` and `DMABUF`. `VIDIOC_EXPBUF` is
  supported. Timestamps are copied from OUTPUT to CAPTURE
  (`V4L2_BUF_FLAG_TIMESTAMP_COPY`). Zero `bytesused` is allowed.
- One open file handle is one session (03).

## 4.2 Pixel formats

### 4.2.1 Compressed formats

| Fourcc | Codec | Decode | Encode |
|---|---|---|---|
| `H264` | H.264 | yes | yes |
| `HEVC` | HEVC | yes | yes |
| `VP80` | VP8 | yes | yes |
| `VP90` | VP9 | yes | yes |
| `AV01` | AV1 | yes | no |
| `MPG2` | MPEG-2 ES | yes | no |
| `MPG4` | MPEG-4 part 2 | yes | no |
| `H263` | H.263 | yes | no |
| `VC1G` / `VC1L` | VC-1 Annex G / L | listed, but filtered out (below) | no |
| `AVS1` / `AVS2` | AVS / AVS2 (vendor fourccs) | listed, but filtered out | no |
| `RV00` | RealVideo (vendor fourcc) | not exposed | no |
| `JPEG`, `MJPG` | JPEG | not exposed through V4L2 | not exposed |

- **[POLICY]** The vendor driver removes VC-1, RealVideo, AVS and AVS2 from
  enumeration on every port. It also removes formats disabled by `FUSE`.
  JPEG has firmware but no V4L2 format entry.

### 4.2.2 Raw formats

| Fourcc | Meaning | Planes (mplane fourcc) |
|---|---|---|
| `NV12` / `NM12` | 4:2:0, Y + interleaved CbCr | 1 / 2 |
| `NV21` / `NM21` | 4:2:0, Y + CrCb | 1 / 2 |
| `YU12` / `YM12` | 4:2:0 3-plane (I420) | 1 / 3 |
| `P010` | 4:2:0 10-bit in 16-bit MSB containers | 1 |
| `M010` (vendor) | P010, multi-plane | 2 |
| `YUYV`, `UYVY` | 4:2:2 packed | 1 |
| `Y210` (vendor) | 4:2:2 10-bit packed | 1 |
| `GREY` | 8-bit luma | 1 |
| `Y10L` (vendor) | 10-bit luma in 16-bit LSB | 1 |
| `BA24` (ARGB32), `RGB4` (RGB32) | Engine format ARGB 8888 (`0x2104`) | 1 |
| `RA24` (BGRA32) | Engine format ABGR 8888 (`0x3104`) | 1 |
| `AB24` (RGBA32) | Engine format RGBA 8888 (`0x0104`) | 1 |
| `AR24` (ABGR32), `BGR4` (BGR32) | Engine format BGRA 8888 (`0x1104`) | 1 |
| `RGB3`, `BGR3` | 24-bit RGB / BGR | 1 |
| `RGBM` (vendor) | RGB 3-plane | 3 |
| `Y0A8` (vendor) | YUV420 AFBC 8-bit | 1 |
| `Y0AA` (vendor) | YUV420 AFBC 10-bit | 1 |
| `Y2A8` / `Y2AA` (vendor) | YUV422 AFBC 8 / 10-bit | 1 |
| `YXA8` / `YXAA` (vendor) | Monochrome AFBC 8 / 10-bit | 1 |
| `Y0YL` (vendor) | Y0L2 Arm packed 10-bit | 1 |
| `Y0AB` (vendor) | AQB1 Arm packed 10-bit | 1 |
| `Y4P3`, `Y2P1`, `Y0P2`, `Y0P3` (vendor) | 10-bit 16-bit-LSB 4:4:4 3-plane, 4:2:2 1-plane, 4:2:0 2-plane, 4:2:0 3-plane | 3 / 1 / 2 / 3 |
| `YM24` | 4:4:4 3-plane 8-bit | 3 |
| `AR15`, `AR12`, `RGBP` | OSD overlay formats (encode only) | 1 |

- **[POLICY]** Removed from enumeration in this driver build: Y0L2, AQB1,
  YUV444 (8/10), the four 16-bit-LSB 10-bit formats, BGR888, the OSD
  formats. The 32-bit RGB formats are never offered as decoder outputs.
- Decoder CAPTURE enumeration depends on the stream once it is known. For
  H.264/HEVC/VP9/AV1, 4:2:2 formats are removed. For 8-bit streams, 10-bit
  formats (AFBC 10, P010, Y10) are removed. For 10-bit streams the decoder
  output follows the stream: setting an 8-bit format on a 10-bit stream is
  switched to its 10-bit partner (NV12 → P010, NM12 → M010, Y0A8 → Y0AA)
  after `SEQUENCE_PARAMETERS`, and the reverse also holds.
- Monochrome streams offer only monochrome AFBC. Colour streams do not
  offer it.

### 4.2.3 Plane geometry (raw, linear)

- **[HW]** Stride per plane = ceil(width × *b* / 2) bytes, and plane size =
  ceil(height × *v* / 2) × stride, with *b* (bytes per 2 pixels) and *v*
  (rows per 2 lines) as follows:

| Format | Plane 0 (*b*, *v*) | Plane 1 | Plane 2 |
|---|---|---|---|
| I420 | (2, 2) | (1, 1) | (1, 1) |
| NV12 / NV21 | (2, 2) | (2, 1) | — |
| P010, 4:2:0 2-plane 10 | (4, 2) | (4, 1) | — |
| YUYV / UYVY | (4, 2) | — | — |
| Y210, 4:2:2 1P-10, 32-bit RGB | (8, 2) | — | — |
| RGB888 / BGR888 | (6, 2) | — | — |
| RGB 3-plane, YUV444 | (2, 2) each | | |
| YUV444 10 | (4, 2) each | | |
| Y8 / Y10 | (2, 2) / (4, 2) | — | — |

- No extra stride alignment is applied (alignment 1 byte). Userspace may
  ask for a larger `bytesperline`, which is honoured. For odd sizes with
  two planes, the chroma stride is at least ceil(width/2) × *b*₁.
- Single-plane fourccs (`NV12`, `YU12`, …) put all planes in one buffer,
  contiguous and in order.

### 4.2.4 AFBC buffer size

- **[HW]** Firmware's `FRAME_ALLOC_PARAM` AFBC byte count is the
  authoritative decode size. `sizeimage` = max(that value, 4096), rounded up
  to 4096 (twice the value if interlaced).
- **[POLICY]** For encode input, the vendor driver computes a worst-case
  size: blocks = ceil(w/16) × (ceil(h/16) + 1), both rounded up to multiples
  of 8 (tiled). Size = round128(blocks × 16) + round128(blocks × *p*). The
  payload *p* per 16×16 block is 384 (4:2:0 8-bit), 480 (4:2:0 10-bit), 512
  (4:2:2 8-bit), 656 (4:2:2 10-bit), 256 (Y8) or 320 (Y10), each rounded up
  to 128 bytes for a tiled body. Interlaced: half the block rows, doubled
  size.

### 4.2.5 Compressed buffer size

- **[POLICY]** `sizeimage` = max(requested, 4096, *w*×*h*×3 if *w*×*h* <
  720×480 else *w*×*h*), capped at 256 MiB for decode input and 192 MiB for
  encode output, rounded up to 4096.

### 4.2.6 Frame-size limits

- `VIDIOC_ENUM_FRAMESIZES` reports a stepwise range: minimum 144×144,
  step 2×2. The maximum is 8192×8192 for H.264/HEVC/VP9/AV1/AVS2 (4096×4096
  on the 4K variant, 00 §0.1.1), 2048×2048 for H.263/MPEG-4/VP8, 4096×4096
  for MPEG-2/RV, 2048×4096 for VC-1, 1920×1080 for AVS, and 16384² for
  encode / 32768² for decode for JPEG. Other formats: 16384² for encode,
  8192² for decode. For a raw fourcc, the limits of the compressed format
  on the other port apply.
- `S_FMT`/`TRY_FMT` clamp to 2…8192 (16384 for encode input and JPEG
  encode, 32768 for JPEG decode). The decoder CAPTURE size is always forced
  to the stream's coded size (or to the downscale/crop size, if set).

## 4.3 Stateful decoder sequence (as implemented)

1. `S_FMT(OUTPUT)` with the compressed fourcc (width/height may be 0).
   `REQBUFS(OUTPUT)`. Fewer than 8 buffers are raised to **8**.
2. `S_FMT(CAPTURE)` with the wanted raw fourcc. While the resolution is
   unknown, `REQBUFS(CAPTURE)` is satisfied with 1 buffer.
3. `STREAMON` on **both** queues. The firmware is loaded and started only
   when both queues are streaming. The first `STREAMON` returns at once.
4. Queue bitstream. On the first sequence header the driver raises
   `V4L2_EVENT_SOURCE_CHANGE` (changes = `V4L2_EVENT_SRC_CH_RESOLUTION`).
   Returned CAPTURE buffers are held back.
5. Userspace reads `G_FMT(CAPTURE)`, `G_SELECTION` and
   `V4L2_CID_MIN_BUFFERS_FOR_CAPTURE`. Then it either reallocates
   (`STREAMOFF`/`REQBUFS(0)`/`REQBUFS(n)`/`STREAMON` on CAPTURE) or keeps its
   buffers and issues `VIDIOC_DECODER_CMD(V4L2_DEC_CMD_START)`. Both paths
   flush the firmware output and re-queue buffers. A `STREAMON(CAPTURE)`
   during a pending change also completes the change.
6. Decoded frames are dequeued from CAPTURE with the input timestamp.
7. Drain: `VIDIOC_DECODER_CMD(V4L2_DEC_CMD_STOP)`. The last CAPTURE buffer
   carries `V4L2_BUF_FLAG_LAST`. Dequeuing it also queues `V4L2_EVENT_EOS`.
8. `STREAMOFF` sends STOP and the flushes (03 §3.8.3). All buffers are
   returned. Pending ones are flagged as errored.

- `TRY_DECODER_CMD` accepts `STOP` and `START` only, and clears flags, pts,
  speed and format.
- Empty CAPTURE buffers with no flags, or ones that need reallocation, are
  silently re-queued and never returned to userspace. This keeps GStreamer
  from reading an empty buffer as EOS.
- Decode CAPTURE minimum: for AFBC, the firmware's AFBC buffer count. For
  linear formats, the larger of the firmware's planar count and 4 (7 for
  AV1), for streams up to 8192×8192 (above that, the firmware value). `V4L2_CID_MIN_BUFFERS_FOR_CAPTURE` is
  read-only, 1–32. The CAPTURE queue needs at least 8 buffers queued
  before streaming can start (min-buffers-needed = 8) for raw formats.
- `V4L2_CID_MVE_VIDEO_MAX_BUFFERS_FOR_CAPTURE` (read-only) reports how many
  frame buffers fit in the frame-buffer VA region.

## 4.4 Stateful encoder sequence

1. `S_FMT(CAPTURE)` with the compressed fourcc. `S_FMT(OUTPUT)` with the
   raw fourcc and size.
2. Set controls (§4.7) and `S_PARM(OUTPUT)` timeperframe. The frame rate
   is clamped to ≤ 256 fps. A zero fraction returns the current value.
3. `REQBUFS` both, `STREAMON` both, then queue raw frames. Keyframes are
   flagged `V4L2_BUF_FLAG_KEYFRAME` on CAPTURE.
4. Drain: `VIDIOC_ENCODER_CMD(V4L2_ENC_CMD_STOP)`. The last bitstream buffer
   carries `V4L2_BUF_FLAG_LAST`.
- `S_PARM(CAPTURE)` is only checked, never applied (no frame-rate
  conversion).
- A queued OUTPUT buffer with `V4L2_BUF_FLAG_KEYFRAME` forces an IDR.
  `V4L2_CID_MPEG_VIDEO_FORCE_KEY_FRAME` also works.

## 4.5 Selection

| Device | Target | Meaning |
|---|---|---|
| decoder | `G_SELECTION` `CROP`, `*_DEFAULT`, `*_BOUNDS`, `COMPOSE_PADDED` | (0, 0, coded width, coded height). `CROP` returns the destination-crop rectangle if one is set |
| decoder | `G_SELECTION` `COMPOSE` / `COMPOSE_DEFAULT` | Visible rectangle of the CAPTURE port. For AFBC, left/top = firmware crop offsets (halved for the dual-downscaled AFBC plane) |
| decoder | `S_SELECTION` CAPTURE `COMPOSE` | Set the downscale target size |
| decoder | `S_SELECTION` CAPTURE `CROP` | Set the destination crop (firmware option 107) |
| encoder | `G_SELECTION` CAPTURE `COMPOSE` | H.264/HEVC: the coded-picture cropping window (frame cropping / conformance window) |
| encoder | `S_SELECTION` CAPTURE `COMPOSE` | Set the crop window written to the stream (options 62–65) |
| encoder | `S_SELECTION` OUTPUT `CROP` | Set the source crop (option 106) |

A rectangle with zero width or height gives `EINVAL`.

## 4.6 Events

| Event | When |
|---|---|
| `V4L2_EVENT_SOURCE_CHANGE` (`RESOLUTION`) | Decoder resolution, bit depth, chroma format or buffer requirements changed |
| `V4L2_EVENT_EOS` | When the `LAST` CAPTURE buffer is dequeued |
| `V4L2_EVENT_CTRL` | Control changes (standard) |

The event queue depth is 2 for EOS and source change.

## 4.7 Controls

### 4.7.1 Standard controls

H.264/HEVC/VP8/VP9 profile, level (and HEVC tier); `BITRATE`,
`BITRATE_MODE`, `BITRATE_PEAK`, `FRAME_RC_ENABLE`; `GOP_SIZE`, `B_FRAMES`;
I/P/B frame QP and min/max QP for H.263, H.264, HEVC and VPx;
`H264_ENTROPY_MODE`; `MULTI_SLICE_MODE`, `MULTI_SLICE_MAX_MB`;
`MV_H/V_SEARCH_RANGE`; `CYCLIC_INTRA_REFRESH_MB`; `FORCE_KEY_FRAME`;
`MIN_BUFFERS_FOR_CAPTURE`/`OUTPUT`; JPEG compression quality and restart
interval; HDR10 CLL and mastering-display info (read-only, volatile, on
the decoder).

- `V4L2_CID_MPEG_VIDEO_BITRATE_MODE` takes two extra values: CBR+1
  ("standard") and CBR+2 ("constrained VBR").
- If mainline does not define them, the HEVC QP controls fall back to
  `V4L2_CID_MPEG_BASE` + 600 … 604 (min, max, I, P, B).

### 4.7.2 Vendor-private controls

Base = `V4L2_CTRL_CLASS_CODEC` (`0x00990000`) + `0x2000` = **`0x00992000`**.
"tri" = tri-state integer (−1 unset, 0 off, 1 on). All controls are
integers unless stated. Ranges are min…max (default).

| ID | Name | Type / range (default) |
|---|---|---|
| `0x992000` | AV1_PROFILE | menu |
| `0x992001` | NALU_FORMAT | menu 0…5: default, start codes, one NALU per buffer, 1-, 2-, 4-byte length |
| `0x992002` | STREAM_ESCAPING | tri |
| `0x992004` | VC1_PROFILE | menu |
| `0x992006` | IGNORE_STREAM_HEADERS | tri |
| `0x992007` | FRAME_REORDERING | tri |
| `0x992008` | INTBUF_SIZE | 0…INT_MAX (0) |
| `0x99200A` | GOP_TYPE | menu: none, bidi, low delay, pyramid, SVCT3, GDR |
| `0x99200B`–`0x99200D` | CONSTR_IPRED, ENTROPY_SYNC, TEMPORAL_MVP | tri |
| `0x99200E` / `0x99200F` | TILE_ROWS / TILE_COLS | 0…65536 (0) |
| `0x992010` | MIN_LUMA_CB_SIZE | 0…16 step 8 (0) |
| `0x992011` | MB_MASK | bitmask 0…0x7FFF (0x7FFF) |
| `0x992012` | VP9_PROB_UPDATE | menu (default: last entry) |
| `0x992013` / `0x992014` | BITDEPTH_CHROMA / BITDEPTH_LUMA | 0…255 (0) |
| `0x992015` | FORCE_CHROMA_FORMAT | −1…INT_MAX (0) |
| `0x992016` | RGB_TO_YUV_MODE | menu |
| `0x992017` | BANDWIDTH_LIMIT | 0…INT_MAX (0) |
| `0x992018` | CABAC_INIT_IDC | 0…4 (0) |
| `0x992019` | VPX_B_FRAME_QP | 0…63 (0) |
| `0x99201A` | SECURE_VIDEO | 0…1 (0) |
| `0x99201F` | HRD_BUFFER_SIZE | 0…2^30−1 (0) |
| `0x992020` / `0x992021` | INIT_QP_I / INIT_QP_P | 0…51 (0) |
| `0x992022` / `0x992023` | SAO_LUMA / SAO_CHROMA | 0…0xFFFF (0) |
| `0x992024` | QP_DELTA_I_P | 0…51 (0) |
| `0x992025` | QP_REF_RB_EN | 0…51 (0) |
| `0x992026`–`0x992029` | RC_CLIP_TOP/BOT, QP_MAP_CLIP_TOP/BOT | 0…255 (0) |
| `0x99202A` / `0x99202B` | MAX_QP_I / MIN_QP_I | 0…63 (63 / 0) |
| `0x99202C` / `0x99202D` | VISIBLE_WIDTH / VISIBLE_HEIGHT | 0…0xFFFF (0) |
| `0x99202E` / `0x99202F` | JPEG_QUALITY_LUMA / _CHROMA | 0…100 (0) |
| `0x992030` / `0x992031` | RC_I_MODE / RC_I_RATIO | 0…1 / 0…100 (0) |
| `0x992032` | INTER_MED_BUF_SIZE | 0…2^30−1 (0) |
| `0x992033` | SVCT3_LEVEL1_PERIOD | 0…255 (0) |
| `0x992034` / `0x992035` | GOP_RESET_PFRAMES / LTR_RESET_PERIOD | 0…255 / 0…0xFFFF (0) |
| `0x992036` | QP_FIXED | 0…63 (0) |
| `0x992037` / `0x992038` | GDR_NUMBER / GDR_PERIOD | 0…0xFFFF (1) |
| `0x992039` | MULIT_SPS_PPS (sic) | 0…0xFFFF (0) |
| `0x99203A` | ENABLE_VISUAL | 0…0x1F (0) |
| `0x99203B`–`0x99203D` | SCD_ENABLE / SCD_PERCENT / SCD_THRESHOLD | 0…1 (0) / 0…10 (3) / 0…2047 (100) |
| `0x99203E`–`0x992042` | AQ_SSIM_EN, AQ_NEG_RATIO, AQ_POS_RATIO, AQ_QPDELTA_LMT, AQ_INIT_FRM_AVG_SVAR | 0…1 (0), 0…63 (24), 0…63 (24), 0…7 (7), 0…15 (0) |
| `0x992043` | COLOR_CONVERSION | 0…0x1F (0) |
| `0x992044` | RGB2YUV_COLOR_CONV_COEF | (set by ioctl, §4.8) |
| `0x992045` | FORCED_UV_VALUE | 0…0x400 (0x400) |
| `0x992046` | DSL_INTERP_MODE | 0…255 (255 = unset) |
| `0x992047` | DISABLED_FEATURES | 0…INT_MAX (0) (03 §3.5.3 option 6 bits) |
| `0x992048` | ENABLE_ADAPTIVE_INTRA_BLOCK | 0…1 (0) |
| `0x992049` | JOB_FRAMES | 0…INT_MAX (1) |
| `0x99204A` | MAX_BUFFERS_FOR_CAPTURE | read-only, 1…`VIDEO_MAX_FRAME` |
| `0x99204B` | AV1_FSF | −1…1 (−1) |
| `0x99204C` | PRIORITY | 0…3 (2): 0 preemption, 1 high, 2 normal, 3 low. Only settable before the firmware is loaded |
| `0x99204D`–`0x992052` | ENC_INTRA_IPENALTY_ANGULAR/PLANAR/DC, ENC_INTER_IPENALTY_ANGULAR/PLANAR/DC | −1…31 (−1) |

IDs `0x992003`, `0x992005`, `0x992009` and `0x99201B`–`0x99201E` are
reserved.

## 4.8 Vendor-private ioctls

`_IOWR('V', BASE_VIDIOC_PRIVATE + n, T)` with `BASE_VIDIOC_PRIVATE` = 192.
The values below assume the natural-alignment sizes shown.

| n | Value | Purpose | Argument (bytes) |
|---|---|---|---|
| 1 | `0xC0E856C1` | Encoder ROI regions | u32 pic index, u8 qp present, u8 qp, u8 roi present, u8 count, 16 × {u16 mbx_left, mbx_right, mby_top, mby_bottom; s16 qp_delta; u16 prio; u16 force_intra} (232) |
| 2 | `0xC00856C2` | EPR QP | s32 qp, s32 EPR I-frame enable (8) |
| 4 | `0xC0EF56C4` | SEI user data unregistered | u8 flags, u8 uuid[16], char[221], u8 length (239) |
| 7 | `0xC00856C7` | Decoder downscale ratio | u32 horizontal, u32 vertical (8) |
| 8 | `0xC00856C8` | Long-term reference | u32 mode, u32 period (8) |
| 9 | `0xC00456C9` | Downscale position mode | s32 (4) |
| 10 | `0xC00456CA` | Mini-frame count | s32 (4) |
| 11 | `0xC01456CB` | Encoder statistics mode | u32 mms size, bitcost size, qp size, flags (bit 0 MMS, 1 bitcost, 2 QP, 3 drop), pic index (20) |
| 12 | `0xC01856CC` | Change rectangles | u32 pic index, u32 count, 2 × {u16 x_left, x_right, y_top, y_bottom} (24) |
| 13 | `0xC2CC56CD` | JPEG Huffman tables | u32 type mask (1 DC-Y, 2 AC-Y, 4 DC-C, 8 AC-C), 4 × u8[16] code lengths, 4 × u8[162] values (716) |
| 14 | `0xC02456CE` | Seamless resolution target | u32 mode, width, height, stride[3], size[3] (36) |
| 15 | `0xC01856CF` | YUV→RGB coefficients | s16 coef[3][3], u16 offset[3] (24) |
| 18 | `0xC01856D2` | RGB→YUV coefficients | s16 coef[9], u8 luma range[2], chroma range[2], RGB range[2] (24) |
| 19 | `0xC02856D3` | OSD configuration | u32 pic index, u32 count, 2 × 16-byte OSD config (40) |
| 20 | `0xC01856D4` | OSD info | u32 width[2], height[2], fourcc[2] (24) |
| 21 | `0xC01056D5` | Encoder lambda scale | 8 × u16 Q8 (16) |

Unknown ioctls return `ENOTTY`.

## 4.9 Buffer flags

Vendor flags live in the upper bits of `v4l2_buffer.flags`. Their meaning
depends on the queue. A second set lives in `v4l2_buffer.reserved2`.

| Value | Queue | Meaning |
|---|---|---|
| `0x01000000` / `0x02000000` / `0x03000000` | dec CAPTURE (in) | Rotate output 90° / 180° / 270° |
| `0x04000000` / `0x08000000` | dec CAPTURE (in) | Downscale ½ / ¼ |
| `0x10000000` | dec CAPTURE (out) | Frame present |
| `0x20000000` | dec CAPTURE (out) | Needs reallocation |
| `0x40000000` | dec CAPTURE (out) | Decode-only (do not display) |
| `0xC1000000` | bitstream | Codec config |
| `0xC2000000` | bitstream | End of sub-frame (output: more of this frame follows) |
| `0x01000000` / `0x02000000` | enc OUTPUT | Mirror horizontal / vertical |
| `0x10000000` / `0x20000000` / `0x30000000` | enc OUTPUT | Rotate 90° / 180° / 270° |
| `0x80000000` | enc OUTPUT | Buffer has ROI data |
| `0x08000000` | enc OUTPUT | EPR / general-buffer side data |
| `0x40000000` | enc OUTPUT | Change rectangles attached |
| `0x00200000` | enc OUTPUT | Reset rate control |
| `0x00400000` | enc CAPTURE | Encoder statistics buffer |
| `0x01000000` / `0x02000000` / `0x04000000` | AFBC frames | Tiled headers / tiled body / 32×8 superblock |
| `reserved2` bit 0 / 1 / 2 | enc OUTPUT | Mini-frame / OSD 1 / OSD 2 |

- Standard flags: `LAST` = EOS, `KEYFRAME` = sync frame (CAPTURE) or force
  IDR (enc OUTPUT) or end of frame (dec OUTPUT), `ERROR` = corrupt.
- A decoder bitstream buffer without `KEYFRAME` and with no NAL format set
  is still treated as a complete frame (03 §3.5.9).

> OPEN QUESTION: The vendor UAPI header contains an older, disabled set of
> these flag values that overlaps the active set. Only the active set
> (above) was used for this table. Which set the Sky1 userspace patches use
> has not been checked against the userspace source.
