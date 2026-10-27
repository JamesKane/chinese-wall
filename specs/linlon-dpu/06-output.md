# 06: Display Output: Colour Processing and Timing

Output path per display, always on the **master** pipeline:
compositor → IPS → BS (timing generator) → pixel bus.

## 6.1 IPS programming

On every commit:

1. `IPS_INPUT_ID0` = the master compositor's input ID. `IPS_INPUT_ID1` = 0.
2. `IPS_SIZE` = composition size (the full mode width × height).
3. `IPS_DEPTH` = output bits per component (§6.3).
4. Read-modify-write `IPS_CONTROL`:
   - if colour management changed: update `FT` and `RGB` (§6.2)
   - always update `YUV`, `CHD422` and `CHD420` (§6.3)

## 6.2 Colour management

### 6.2.1 Colour transform matrix (CTM)

- **[HW]** `IPS_RGB_RGB_COEFF0–8` hold a 3×3 matrix, row-major (R′ row,
  G′ row, B′ row), applied to linear composed RGB when `IPS_CONTROL.RGB` is
  set.
- **[HW]** Coefficient format: signed fixed point with **12 fractional
  bits** and 3 integer bits including sign (Q3.12, so 1.0 = 4096, range
  −4.0 … +4.0 − 2⁻¹²).
- **[ABI]** Userspace supplies the matrix in sign-magnitude S31.32 (the
  standard DRM CTM blob). Conversion: drop the sign bit, shift the
  magnitude right by 20, clamp to 2¹⁴ − 1 (positive) or 2¹⁴ (negative), and
  negate for negative inputs. Write the result as a 32-bit two's-complement
  word.
- **[POLICY]** No CTM: clear `RGB`. Coefficients 9–11 (probably offsets)
  are never written.

### 6.2.2 Gamma (forward transform) table

- **[HW]** The pipeline's forward-transform block (type `0x32`) holds a
  65-point curve, `FT_COEFF0–64`, each a 12-bit output value (0 … 4096). It
  is applied when `IPS_CONTROL.FT` is set.
- **[HW]** The 65 points sit at **non-uniform** input positions on a 12-bit
  input scale (0 … 4095). The point spacing grows with input level:

| Points | Input positions | Spacing |
|---|---|---|
| 0–3 | 0, 4, 8, 12 | 4 |
| 4–7 | 16 … 28 | 4 |
| 8–11 | 32 … 56 | 8 |
| 12–15 | 64 … 112 | 16 |
| 16–19 | 128 … 224 | 32 |
| 20–23 | 256 … 448 | 64 |
| 24–39 | 512 … 992 | 32 |
| 40–63 | 1024 … 3968 | 128 |
| 64 | (end point) | written as 4096 |

- **[ABI]** Userspace supplies a 4096-entry DRM gamma LUT (16-bit values
  per channel). To build the table, sample the LUT at the input positions
  above and convert each 16-bit value v to 12 bits as
  round(v × 4095 ÷ 65535). Point 64 is always 4096.
- **[POLICY]** Linux samples only the **red** channel and loads one curve
  for all three components. Per-channel gamma is not supported.
- **[ABI]** The gamma LUT size reported to userspace is 4096. No degamma
  LUT is offered.

> OPEN QUESTION: The order of the CTM and the gamma table inside the IPS is
> not stated. DRM semantics expect CTM first, then gamma.

> OPEN QUESTION: Whether the hardware interpolates linearly between points,
> and whether it can hold separate R/G/B curves, is not described. The
> point layout (fine near black) suggests a curve meant for gamma encoding
> with linear interpolation.

> OPEN QUESTION: The odd point spacing in rows 24–39 (32, finer than the
> 64 used just below) is taken as found. It may be a deliberate density
> change or an error in the source.

## 6.3 Output colour format and depth

Chosen at each modeset from the sink's capabilities (**[POLICY]**):

1. Depth: the lowest bpc of all connectors on the display (8 if
   unknown), capped by the IPS's supported depths {8, 10}. Use the
   **largest** supported depth ≤ that bpc.
2. Format: AND the sinks' supported formats (RGB 4:4:4 if none reported)
   with the IPS's {RGB 4:4:4, YCbCr 4:4:4, YCbCr 4:2:2, plus YCbCr 4:2:0
   if `IPS_INFO[10]`}. Pick the **lowest-numbered** one in the DRM
   output-colour-format enumeration, which puts RGB 4:4:4 first.

**[HW]** `IPS_CONTROL` bits for each format:

| Format | `YUV` | `CHD422` | `CHD420` |
|---|---|---|---|
| RGB 4:4:4 | 0 | 0 | 0 |
| YCbCr 4:4:4 | 1 | 0 | 0 |
| YCbCr 4:2:2 | 1 | 1 | 0 |
| YCbCr 4:2:0 | 1 | 1 | 1 |

`IPS_DEPTH[3:0]` = the depth (8 or 10).

## 6.4 Timing generator (BS)

### 6.4.1 Mode adjustment

- **[HW]** Interlaced modes are not supported.
- **[HW]** Dual-link: halve the mode's pixel clock and all horizontal
  timing values (active width, sync start, sync end, total) before
  programming the BS and the pixel clock. Each link then carries half of
  each line.
- **[POLICY]** The pixel clock actually used is the clock provider's
  rounded rate for the (possibly halved) clock. Linux reports that rate as
  the adjusted mode clock.

### 6.4.2 Register values

From the adjusted mode (horizontal values already halved for dual-link):

| Register | Value |
|---|---|
| `BS_ACTIVESIZE` | hactive \| vactive << 16 (`HV_SIZE`) |
| `BS_HINTERVALS` | front porch = hsync_start − hactive; back porch = htotal − hsync_end |
| `BS_VINTERVALS` | front porch = vsync_start − vactive; back porch = vtotal − vsync_end |
| `BS_SYNC` | hsync width = hsync_end − hsync_start; vsync width = vsync_end − vsync_start; `HSP` if +hsync; `VSP` if +vsync |
| `BS_PROG_LINE` | 4 |
| `BS_PREFETCH_LINE` | 5 |
| `BS_DRIFT_TO` | dual-link only: horizontal front porch + 16 |
| `BS_CONTROL` | `EN` \| `VM`, plus `DL` for dual-link (written whole) |

- **[HW]** Field widths limit the timings: hactive and vactive ≤ 8191,
  h front porch ≤ 4095, h back porch ≤ 1023, hsync ≤ 1023, v front porch
  ≤ 16383, v back porch ≤ 255, vsync ≤ 255. Linux does **not** check these
  and silently truncates. A clean-room driver should reject modes that do
  not fit.
- **[POLICY]** Prefetch of 5 lines and the vblank interrupt line of 4 (one
  line before prefetch) are Linux defaults. Pixels are fetched starting 5
  lines before active video, and the "vblank" event fires at line 4.
- **[POLICY]** The drift timeout of front porch + 16 is an unexplained
  Linux choice.

> OPEN QUESTION: The reference point for `BS_PROG_LINE` and
> `BS_PREFETCH_LINE` (lines before active video, or lines counted from the
> sync pulse) is not stated. The `PL0` event is treated as vblank, so it
> fires in or near the vertical blanking interval.

### 6.4.3 Mode limits

- **[POLICY]** Linux advertises a maximum framebuffer and mode size of
  4096 × 4096.
- **[HW]** Per-display width is also limited by the compositor's maximum
  line (05 §5.1.4) and the layers' line sizes (04 §4.7).

## 6.5 Clock requirements

- **[HW]** `pxclk` (per pipeline) = adjusted pixel clock (halved for
  dual-link).
- **[HW]** `aclk` ≥ `pxclk` for single-link, ≥ 2 × `pxclk` for dual-link.
- **[HW]** `aclk` ≥ the downscaling requirement (05 §5.3.5) whenever a
  scaler downscales.
- **[POLICY]** Linux sets `aclk` once, when the first display turns on, to
  the rounded rate of the minimum for **that** display. A second display
  with a higher requirement does not raise it. The mode check for the
  second display still compares against what the clock **could** reach.
  A clean-room driver should compute `aclk` from all active displays.
