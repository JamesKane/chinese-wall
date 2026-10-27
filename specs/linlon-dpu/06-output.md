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

### 6.2.3 Display degamma *(Linlon-D6, vendor)*

- **[HW]** `IPS_CONTROL.IT` (bit 1) enables an input-transform (degamma)
  stage in the IPS. The vendor driver sets it when a CRTC degamma LUT is
  supplied and offers a degamma LUT property. Mainline offers none.
- **[POLICY]** *(vendor)* The vendor CRTC also offers a 12-entry
  `CTM_EXT` blob (3×4: matrix plus offsets, S31.32 each). Only the first 9
  entries reach the hardware, converted as in §6.2.1.

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

- **[HW]** When `YUV` is set, load `IPS_RGB_YUV_COEFF0–11` with an RGB→YUV
  matrix (04 §4.8.1 format). *(vendor)* The vendor driver always uses
  BT.601 limited range for the display output.
- **[HW]** *(vendor)* `IPS_CONTROL.DITH` (bit 12) is set when dithering is
  requested (a vendor debug option).

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

### 6.4.4 Two pixels per clock *(Linlon-D6, vendor)*

- **[HW]** `BS_CONTROL[22:21]` = 1 makes the timing generator emit **two
  pixels per pixel-clock cycle**. All horizontal timing values (active
  width, sync start, sync end, total) and the pixel clock are then divided
  by 2, the same way as for dual-link (§6.4.1). Dual-link and 2 pixels per
  clock multiply: the divisor is 2 × 2 = 4 when both are used.
- **[HW]** `BS_ACTIVESIZE` horizontal is the active width in **clock
  cycles**. With a merged side-by-side image (§6.6) at 2 pixels per clock
  that is half the screen width. Without the image merge but at 2 pixels
  per clock, double it back (the vendor driver multiplies by 2). With the
  merge at 1 pixel per clock, halve it.
- **[POLICY]** *(vendor)* A pipeline at 1 pixel per clock is limited to a
  600 MHz pixel clock. Modes between 600 and 1200 MHz switch to 2 pixels
  per clock **with side-by-side** (§6.6). Modes ≥ 1200 MHz are rejected.
  A devicetree property `pixel_per_cycle` on a pipeline node can force 1
  or 2.

> OPEN QUESTION: Whether 2 pixels per clock can be used without
> side-by-side (one pipeline producing two pixels per cycle) is unclear.
> The vendor driver always pairs them automatically, but handles the
> unmerged case in its active-width arithmetic.

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
- **[POLICY]** *(vendor)* On Sky1 `aclk` is fixed at 800 MHz (08 §8.3),
  which sidesteps this. In side-by-side mode the vendor driver does **not**
  double the `aclk` minimum even on a dual-link pipeline.
- **[POLICY]** *(vendor)* The vendor mode check accepts a pixel clock when
  the clock provider's rounded rate is **≥** the request, not only equal.

## 6.6 Side-by-side composition *(Linlon-D6, vendor)*

Used for very high pixel rates (§6.4.4). Two pipelines, master M and slave
S, each compose **half** of the screen. The output is a single display
driven by M.

**[HW]** data path:

```
 M layers → M compositor → M IPS input 0 ─┐
                                         ├→ M IPS (SBS, IM) → M BS → output
 S layers → S compositor → S IPS ─→ M IPS input 1
```

- Each compositor's size is (screen width ÷ 2) × screen height (plus an
  optional overlap that the vendor driver leaves at 0).
- **[HW]** S's IPS is programmed like a normal IPS with its own input from
  S's compositor. M's IPS takes M's compositor on input 0 and S's IPS on
  input 1, with `IPS_CONTROL.SBS` set. `IPS_CONTROL.IM` (image merge) is
  also set unless M is dual-link (with dual-link, each link carries one
  half instead).
- **[HW]** Only M's timing generator runs. It is programmed per §6.4.4.
- **[POLICY]** *(vendor)* Each plane is split at the screen's midline. The
  left part goes to a layer of M and the right part to the layer with the
  **same index** in S. A plane wholly on one side uses one layer. If the
  plane is scaled, each half's source width is proportional to its share of
  the on-screen width, plus the scaler overlap (8, plus 1 with the
  enhancer, 05 §5.3.1), and each scaler crops the overlap on the inner edge
  (M crops on the right, S on the left). Compositor input sizes exclude
  that crop.
- **[POLICY]** *(vendor)* Side-by-side can also be fixed in devicetree
  (`side_by_side_master = <pipeline index>`). Otherwise it is switched on
  and off automatically when a mode crosses the 600 MHz threshold, and the
  CRTC master/slave assignments are recomputed. The number of CRTCs does
  not change, but while side-by-side is on the slave pipeline is busy, so
  the DPU's second output cannot be used at the same time.
- **[POLICY]** *(vendor)* Write-back in side-by-side mode has its own
  path (write-back of both halves). It is not described here.

> OPEN QUESTION: Exact placement of the right-half crop when scaling in
> side-by-side (the vendor code carries FIXME notes there) should be
> checked on hardware.
