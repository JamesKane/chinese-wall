# 04: Layers, Pixel Formats and AFBC

## 4.1 Hardware pixel-format IDs

**[HW]** `L_FORMAT` / `LW_FORMAT` take a 6-bit ID: `[5:3]` group,
`[2:0]` index. The table gives each ID with the DRM FourCC it corresponds
to (FourCC codes are public, from the DRM fourcc header), which layer kinds
accept it, and the rotations allowed.

Layer kinds: **R** = rich read layer, **S** = simple read layer,
**W** = write-back layer.
Rotation column: **flip** = H/V mirror only; **all** = 0/90/180/270 plus
mirror; **none** = no rotation or mirror.

### 4.1.1 Linear (uncompressed) buffers

| ID | Group/idx | FourCC (DRM) | Kinds | Rotation |
|---|---|---|---|---|
| `0x00` | 0/0 | `AR30` (ARGB2101010) | R S W | flip |
| `0x01` | 0/1 | `AB30` (ABGR2101010) | R S W | flip |
| `0x02` | 0/2 | `RA30` (RGBA1010102) | R S W | flip |
| `0x03` | 0/3 | `BA30` (BGRA1010102) | R S W | flip |
| `0x08` | 1/0 | `AR24` (ARGB8888) | R S W | flip |
| `0x09` | 1/1 | `AB24` (ABGR8888) | R S W | flip |
| `0x0A` | 1/2 | `RA24` (RGBA8888) | R S W | flip |
| `0x0B` | 1/3 | `BA24` (BGRA8888) | R S W | flip |
| `0x10` | 2/0 | `XR24` (XRGB8888) | R S W | flip |
| `0x11` | 2/1 | `XB24` (XBGR8888) | R S W | flip |
| `0x12` | 2/2 | `RX24` (RGBX8888) | R S W | flip |
| `0x13` | 2/3 | `BX24` (BGRX8888) | R S W | flip |
| `0x18` | 3/0 | `RG24` (RGB888) | R S W | none |
| `0x19` | 3/1 | `BG24` (BGR888) | R S W | none |
| `0x20` | 4/0 | `RA15` (RGBA5551) | R S | flip |
| `0x21` | 4/1 | `AB15` (ABGR1555) | R S | flip |
| `0x22` | 4/2 | `RG16` (RGB565) | R S | flip |
| `0x23` | 4/3 | `BG16` (BGR565) | R S | flip |
| `0x24` | 4/4 | `R8  ` (R8) | S | none |
| `0x2A` | 5/2 | `YUYV` | R | flip |
| `0x2B` | 5/3 | `UYVY` | R | flip |
| `0x2E` | 5/6 | `NV12` (2-plane 4:2:0) | R W | flip |
| `0x2F` | 5/7 | `YU12` (3-plane 4:2:0) | R | flip |
| `0x36` | 6/6 | `X0L2` (10-bit packed 4:2:0, 2×2 tiles) | R | flip |
| `0x37` | 6/7 | `P010` (2-plane 10-bit 4:2:0) | R | flip |

- **[HW]** The layout follows the DRM FourCC definitions exactly: within a
  pixel, the FourCC name lists components from most to least significant
  bit of a little-endian word.
- **[HW]** The 24-bit RGB formats cannot be rotated or mirrored when linear.
- **[HW]** `R8` is accepted only by simple layers.

### 4.1.2 AFBC (compressed) buffers

| ID | FourCC | Kinds | Rotation | Superblock sizes | Required AFBC features |
|---|---|---|---|---|---|
| `0x01` | `AB30` | R S | all | 16×16, 32×8 | any subset of {tiled, SC, sparse, YTR, split} |
| `0x09` | `AB24` | R S | all | 16×16, 32×8 | any subset of {tiled, SC, sparse, YTR, split} |
| `0x19` | `BG24` | R S | all | 16×16, 32×8 | any subset of {tiled, SC, sparse, YTR, split} |
| `0x21` | `AB15` | R S | all | 16×16, 32×8 | any subset of {tiled, SC, sparse, YTR} |
| `0x23` | `BG16` | R S | all | 16×16, 32×8 | any subset of {tiled, SC, sparse, YTR} |
| `0x29` | `YUYV` | R | all | 16×16 | any subset of {tiled, sparse} |
| `0x2E` | `YUV420_8BIT` (`YU08`) | R | all | 16×16 | any subset of {tiled, sparse} |
| `0x37` | `YUV420_10BIT` (`YU10`) | R | all | 16×16 | any subset of {tiled, sparse} |

- **[HW]** AFBC uses the component order with R in the lowest bits (ABGR /
  BGR). Its hardware ID matches the linear ID of the same component order,
  except packed 4:2:2, which uses `0x29` for AFBC and `0x2A` for linear.
- **[HW]** The write-back layer cannot produce AFBC.
- **[HW]** 32×8 superblocks with 90° or 270° rotation are **not**
  supported.
- **[POLICY]** A modifier is accepted if its block size is listed and its
  feature bits are a subset of the allowed set. The DRM AFBC modifier
  encoding (the public Arm AFBC format-modifier layout in the DRM fourcc
  header) defines the bits.

## 4.2 Advertised modifier list

**[ABI]** The modifiers offered on every plane (filtered per format by
§4.1.2) are, as AFBC block size plus feature flags:

| Block size | Features |
|---|---|
| 16×16 | none · sparse · YTR+sparse · YTR · split+sparse+YTR · tiled+sparse · tiled · tiled+SC+split+sparse+YTR · tiled+SC+sparse+YTR · tiled+SC+YTR |
| 32×8 | YTR+sparse · YTR · split+sparse+YTR · tiled+SC+split+sparse+YTR · tiled+SC+sparse+YTR · tiled+SC+YTR |
| — | linear |

- **[POLICY]** These follow two rules: RGB AFBC always uses YTR, and tiled
  RGB always uses SC (solid colour). Split is offered only together with
  sparse.

## 4.3 AFBC buffer geometry

**[HW]** For an AFBC framebuffer of width W and height H:

| Quantity | Rule |
|---|---|
| Alignment unit | 16×16 superblock: 16 × 16. 32×8: 32 × 8. With tiled headers, both multiplied by 8 (so 128×128 or 256×64) |
| Aligned size | W′ = W rounded up to the unit width, H′ = H rounded up to the unit height |
| Superblock count | n = W′ × H′ ÷ 256 |
| Header size | 16 bytes per superblock |
| Body start alignment | 1024 bytes, or 4096 bytes with tiled headers |
| Buffer start (plane 0 offset) | Must be a multiple of the body start alignment |
| Payload offset | n × 16 rounded up to the body start alignment |
| Bytes per superblock body | 256 × bpp ÷ 8, rounded up to 128 |
| bpp | The format's bits per pixel, except `YUV420_8BIT` = 12 and `YUV420_10BIT` = 15 |
| Total size | payload offset + n × body bytes. The buffer object must hold at least plane-0 offset + total size |

- **[HW]** AFBC buffers are single-plane.

## 4.4 Programming a read layer

Inputs: framebuffer (format, modifier, planes, pitches, offsets), source
rectangle (x, y, w, h in whole pixels), rotation and mirror flags.

1. **Addresses.**
   - Linear: for each plane i, address = plane base + plane offset +
     (x′ ÷ block width) × bytes per block + y′ × pitch, where x′ and y′ are
     x and y divided by the chroma subsampling factors for planes ≥ 1.
     Write planes 2 (if present), then 1, then 0 to `P2/P1/P0_PTR`.
   - AFBC: `P0_PTR` = buffer base + plane-0 offset (the header).
     `P1_PTR` = header address + payload offset **if tiled headers**,
     otherwise header address + total size − 1 (the last byte of the
     buffer). The source rectangle is applied with the crop registers
     (step 4), not with the address.
2. **Strides.** `P0_STRIDE` = pitch₀ × (plane-0 block height).
   `P1_STRIDE` = pitch₁ × (plane-1 block height) if there are ≥ 2 planes.
   Block height is 1 for every format here except `X0L2` (2).
   - **[HW]** A three-plane format must have equal pitches for planes 1
     and 2. The hardware has no `P2_STRIDE`.
3. **Format.** `L_FORMAT` = hardware ID (§4.1).
4. **AFBC.** `AD_CONTROL` = 0 for linear. Otherwise set `AEN`, plus
   `WB` (32×8), `YT` (YTR), `BS` (split) and `TH` (tiled) to match the
   modifier. Then:
   - `L_IN_SIZE` = W′ × H′ (aligned size of the whole buffer)
   - `AD_H_CROP` = left: x; right: W′ − x − w
   - `AD_V_CROP` = top: y; bottom: H′ − y − h
   For linear buffers `L_IN_SIZE` = w × h.
5. **YUV (rich layers only).**
   - `L_RICH_CONTROL`: packed 4:2:2 (`YUYV`) uses bilinear if AFBC,
     replication if linear. `UYVY` uses replication. All 4:2:0 formats use
     the JPEG-style setting (0).
     **[POLICY]** The 4:2:0 choice is Linux's. Both are supported.
   - `YUV_RGB_COEFF0–11` = the matrix for the plane's colour encoding and
     range (§4.5).
6. **Control.** Read-modify-write `L_CONTROL`: set `EN`, set `ROT`,
   `HFLIP` and `VFLIP` from the requested transform, and set `TBU_EN` if
   the addresses are IOMMU virtual addresses (01 §1.5).

- **[HW]** `L_PALPHA` = `0xFFAA5500` maps 2-bit alpha codes 0, 1, 2, 3 to
  8-bit alpha 0x00, 0x55, 0xAA, 0xFF (one byte per code, code 0 in the low
  byte). It is written once at initialisation.

> OPEN QUESTION: Whether `L_PALPHA` also governs 1-bit alpha (the 5551
> formats) is not stated. The map gives 0x00 and 0xFF at codes 0 and 3, so
> a 1-bit code would need its own expansion rule.

### 4.4.1 Rotation and the data flow

- **[HW]** The layer applies rotation and mirroring while fetching. After a
  90° or 270° rotation the layer's output width and height are swapped
  relative to the source rectangle. Every later stage (scaler, compositor)
  sees the rotated size.
- **[HW]** For 90°/270°, the layer's line-buffer limit applies to the source
  **height** (because columns become lines).

## 4.5 YUV→RGB matrices

**[HW]** `YUV_RGB_COEFF0–11` hold a 3×3 matrix and a 3-entry offset vector
in 12 words, 10-bit fractional precision (1.0 = 1024), signed:

- words 0–2: R row (Y, Cb, Cr coefficients)
- words 3–5: G row
- words 6–8: B row
- words 9–11: offsets subtracted from Y, Cb, Cr (8-bit-scale values
  expressed on a 10-bit scale, so 64 = 16 × 4, 512 = 128 × 4)

These are standard colorimetry constants (ITU-R BT.601 / BT.709 / BT.2020)
scaled by 1024 for full range, and by 1024 × 255 ÷ 219 for limited-range
luma and 1024 × 255 ÷ 224 for limited-range chroma. A clean-room driver can
derive them from the ITU specifications. For bring-up comparison, the values
Linux uses are:

| Encoding / range | R row | G row | B row | Offsets |
|---|---|---|---|---|
| BT.601 limited | 1192, 0, 1634 | 1192, −401, −832 | 1192, 2066, 0 | 64, 512, 512 |
| BT.601 full | 1024, 0, 1436 | 1024, −352, −731 | 1024, 1815, 0 | 0, 512, 512 |
| BT.709 limited | 1192, 0, 1836 | 1192, −218, −546 | 1192, 2163, 0 | 64, 512, 512 |
| BT.709 full | 1024, 0, 1613 | 1024, −192, −479 | 1024, 1900, 0 | 0, 512, 512 |
| BT.2020 (any range) | 1024, 0, 1476 | 1024, −165, −572 | 1024, 1884, 0 | 0, 512, 512 |

- **[POLICY]** Linux uses the full-range BT.2020 matrix even when limited
  range is requested.

> OPEN QUESTION: The register encoding of negative coefficients (two's
> complement in how many bits) is not stated. Linux writes 32-bit signed
> integers, so plain 32-bit two's complement is safe.

## 4.6 Linear buffer constraints

- **[HW]** Each plane's pitch × block height must be a multiple of **16
  bytes** (the AXI bus width).
- **[HW]** Source x, y, w and h must be multiples of the format's chroma
  subsampling factors and of the plane-0 block size (2 for `X0L2`).
- **[HW]** The source rectangle must lie inside the framebuffer.
- **[POLICY]** Linux checks each plane object is large enough to hold
  `height` lines from the plane's start offset.

## 4.7 Layer size limits

| Check | Rule |
|---|---|
| Source width | 4 ≤ w ≤ the layer's max RGB line (03 §3.2) |
| Source height | 4 ≤ h ≤ the max number of lines |
| Line length | The source width (or source height if rotated 90°/270°) must be ≤ the layer's max line: the YUV limit for chroma-subsampled formats, otherwise the RGB limit |
| AFBC line | Line length after AFBC crop must be ≤ the max line for 32×8, or ≤ **half** the max line for 16×16 |
| `YUV420_10BIT` | Line length > 2046 is not allowed when the AFBC left crop is not a multiple of 4 |
| `X0L2` | Line length > 2046 is not allowed when the plane-0 address is not 16-byte aligned |

- All of these are **[HW]** limits. The last two are hardware erratum-like
  limits stated without explanation.

## 4.8 Programming the write-back layer

Inputs: write-back framebuffer, output rectangle in the framebuffer (Linux
always uses the full buffer at 0, 0), and the upstream source (compositor,
scaler or merger, 05 §5.5).

1. Plane addresses, strides and `LW_FORMAT` as for a linear read layer
   (§4.4 steps 1–3).
2. `LW_IN_SIZE` = output width × height.
3. `LW_INPUT_ID0` = input ID of the upstream block.
4. Read-modify-write `LW_CONTROL`: set `EN` and `OFM`, and set `TBU_EN` if
   using IOMMU addresses.

- **[HW]** Write-back size limits: 64 ≤ width ≤ write-back max line;
  64 ≤ height ≤ max lines.
- **[HW]** The `EOW` interrupt (LPU) marks the end of the frame written.
- **[HW]** The write-back layer has no rotation.
