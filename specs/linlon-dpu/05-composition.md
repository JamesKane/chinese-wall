# 05: Composition, Scaling, Split/Merge and Write-back Path

## 5.1 Compositor

### 5.1.1 Data paths into the compositor

**[HW]** Valid sources for a compositor input (from the block's
valid-input list, typically):

| Source | When |
|---|---|
| A read layer of the same pipeline | Unscaled plane |
| A scaler of the same pipeline | Scaled plane |
| The merger of the same pipeline | Plane that was split across two scalers (§5.4) |
| The other pipeline's compositor | This pipeline is a master and the other is its slave (00 §0.3) |

### 5.1.2 Programming

For each input n (0 … 4) whose configuration changed:

- Active input:
  1. `CU_INPUT_ID<n>` = source input ID (02 §2.2.2).
  2. `CU_INPUT<n>_SIZE` = width × height of the plane on screen.
  3. `CU_INPUT<n>_OFFSET` = x, y of the plane on screen.
  4. `CU_INPUT<n>_CONTROL` = `EN`, plus the blend bits, plus plane alpha in
     `[15:8]`.
- Inactive input: `CU_INPUT_ID<n>` = 0 and `CU_INPUT<n>_CONTROL` = 0.

Then always write `CU_SIZE` = composition width × height.

Blend mode → control bits (**[HW]**):

| Blend mode | `PAD` | `PMUL` | Pixel alpha used |
|---|---|---|---|
| None (ignore pixel alpha) | 1 | 0 | No |
| Pre-multiplied | 0 | 1 | Yes, colour already multiplied |
| Coverage (straight alpha) | 0 | 0 | Yes |

- **[HW]** Plane alpha (8-bit, 0xFF = opaque) is applied in all modes.
- **[POLICY]** For formats without an alpha channel, Linux forces blend
  mode "none".
- **[HW]** The composed output carries **no alpha**. When a slave pipeline's
  compositor feeds the master, that input must use blend mode "none" and
  plane alpha 0xFF, at **input index 0** (the bottom).

### 5.1.3 Z-order

- **[HW]** Input index is the z-order: higher index is on top. Each index
  holds at most one source.
- **[POLICY]** Linux takes the planes on a display sorted by their zpos
  property, rejects two planes with equal zpos, and numbers them 0, 1, 2, …
  When a plane will be split across two layers (§5.4), it takes two
  consecutive numbers: left half at k, right half at k + 1.
- **[POLICY]** With a slave pipeline: all of the display's planes (slave and
  master) are sorted together. Slave planes must all sort **below** master
  planes. Slave planes keep their numbers 0 … k in the slave compositor.
  Master planes are renumbered by subtracting k, so the lowest master plane
  gets input 1 and input 0 stays free for the slave's composed output.
  Linux rejects a configuration where a master plane sorts below a slave
  plane.

### 5.1.4 Limits

| Check | Rule |
|---|---|
| Composition size | Equals the display mode's active area (`hdisplay` × `vdisplay`, the full width, even for dual-link) |
| Composition size range | 64 ≤ width ≤ the CU max line; 64 ≤ height ≤ max lines (**[HW]**) |
| Plane on screen | x + w ≤ composition width; y + h ≤ composition height; w, h > 0 (**[POLICY]**, Linux does not clip) |

## 5.2 Choosing a path for a plane

**[POLICY]** For each plane, with source size (w, h) after rotation and
on-screen size (W, H):

- Scaling is needed if (w, h) ≠ (W, H).
- Image enhancement is turned on if W ≥ 2w or H ≥ 2h (an upscale of 2× or
  more).
- Split is needed when scaling **and** w or W is outside the scaler's
  width range (§5.3.1).

| Scaling or enhancement | Split | Path |
|---|---|---|
| No | No | Layer → compositor |
| Yes | No | Layer → scaler → compositor |
| Yes | Yes | Layer L → scaler 0 ┐ merger → compositor; layer R → scaler 1 ┘ (§5.4) |

- **[HW]** A scaler is free only if no other user in this commit has
  claimed it. The two scalers of a pipeline are shared by all its layers and
  the write-back path.

## 5.3 Scaler

### 5.3.1 Limits

| Parameter | Value | Tag |
|---|---|---|
| Input and output width | 4 … scaler max line (2048 on PERIPH parts, otherwise `MAX_LINE_SIZE`) | [HW] |
| Input and output height | 4 … 4096 | [HW] |
| Maximum upscale | 64× in each direction | [HW] |
| Maximum downscale | 6× in each direction | [HW] |
| Upscale when the input is the compositor (write-back) | not allowed | [HW] |
| Split overlap for scaling | 8 pixels | [HW] |
| Extra split overlap when the enhancer is on | 1 pixel | [HW] |

### 5.3.2 Filter table selection (`SC_COEFFTAB`)

**[HW]** Choose the horizontal table from the horizontal ratio
r = input width ÷ output width, and the vertical table the same way:

| Ratio r | Horizontal select | Vertical select |
|---|---|---|
| r ≤ 1 (upscale or 1:1) | `0x62` | `0x6A` |
| 1 < r ≤ 1.5 | `0x63` | `0x6B` |
| 1.5 < r ≤ 2 | `0x64` | `0x6C` |
| 2 < r ≤ 2.75 | `0x65` | `0x6D` |
| r > 2.75 | `0x66` | `0x6E` |

Write `SC_COEFFTAB` = horizontal select in `[15:0]` | vertical select in
`[31:16]`. Compare with integer arithmetic: for example "r ≤ 1.5" is
in ≤ out + ⌊out ÷ 2⌋, and "r ≤ 2.75" is in ≤ 2·out + ⌊3·out ÷ 4⌋.

> OPEN QUESTION: These select values are magic. They look like
> (coefficient-table block type 6 in bits 7:4) | (table instance), with
> instances 2–6 for horizontal and 10–14 for vertical, which would mean
> preset tables in the global scaler coefficient blocks. This is not
> confirmed.

### 5.3.3 Phase registers

All phases are unsigned 16.16 fixed point. Let:

- `Wt_in`, `Wt_out`: the full (unsplit) horizontal input and output size
- `Ht_in`: the full vertical input size, `H_out`: this scaler's output
  height
- `W_in`: this scaler's own horizontal input size (one half if split)
- `W_out`: this scaler's horizontal output size including crop
- `Lc`: left crop of this scaler's output

**[HW]** Phase step (the source distance between output samples):

- `SC_H_DELTA_PH` = ⌊Wt_in × 2¹⁶ ÷ Wt_out⌋
- `SC_V_DELTA_PH` = ⌊Ht_in × 2¹⁶ ÷ H_out⌋

**[HW]** Initial phase (the source position of the first output pixel's
centre, relative to the first source pixel this scaler receives):

- Unsplit, or the left half of a split:
  `SC_H_INIT_PH` = ⌊Wt_in × 2¹⁵ ÷ Wt_out⌋ (half a step)
- Right half of a split: let d = Wt_out − W_out + Lc, minus 1 more if the
  enhancer is on. Then
  `SC_H_INIT_PH` = ⌊(Wt_in × (2d + 1) − 2 × Wt_out × (Wt_in − W_in)) × 2¹⁵ ÷ Wt_out⌋
  This is the centre of output pixel d mapped into source space, minus the
  source position where the right half starts.
- `SC_V_INIT_PH` = ⌊Ht_in × 2¹⁵ ÷ H_out⌋

### 5.3.4 Control and the other registers

1. `SC_COEFFTAB` (§5.3.2).
2. `SC_IN_SIZE` = this scaler's input size.
3. `SC_OUT_SIZE` = this scaler's output size **including** crop.
4. `SC_H_CROP` = left crop | right crop (§5.4).
5. Phases (§5.3.3).
6. `SC_CONTROL`:
   - `SCL` if the size changes
   - `AP` if the next stage blends with pixel alpha (blend mode not "none")
   - `IENH` if enhancement is on
   - `LS` if this scaler handles one half of a **layer** split, meaning its
     input is a layer, not the hardware splitter
7. `SC_INPUT_ID0` = source input ID.

- **[POLICY]** *(vendor)* For the **right half** of a split that really
  scales, the vendor driver reprograms the output so the left crop is at
  most 1 pixel: `SC_OUT_SIZE` width = output width − left crop (+ 1 if the
  enhancer is on) and `SC_H_CROP` = left 1 (enhancer on) or 0, right
  unchanged. This suggests the hardware applies the left crop poorly on the
  right half and the phase arithmetic (§5.3.3) already skips those pixels.

> OPEN QUESTION: Mainline and vendor drivers disagree on the right-half
> crop (above). Check on hardware.

### 5.3.5 Downscaling clock requirement

**[HW]** When a scaler downscales in either direction, `aclk` must be fast
enough, relative to the output pixel clock, to read the extra source lines
in time. With h_in, v_in the scaler input size, v_out the output height,
and the mode's htotal and hdisplay:

- **Horizontal-only downscale (v_in = v_out):**
  aclk ÷ pxclk ≥ h_in ÷ (hdisplay − 3)
- **Any vertical downscale:**
  aclk ÷ pxclk ≥ (h_in × v_in) ÷ ((htotal − 1) × v_out − 2 × v_in)

Evaluate as aclk × denominator ≥ pxclk × numerator, using integers. Use the
`aclk` rate that will actually be set (01 §1.2) and the adjusted pixel clock
(06 §6.4).

## 5.4 Splitting a plane across two scalers

**[HW]** reason: one scaler's line buffer is limited (§5.3.1). A wider
plane is processed as two halves, each scaled separately, then joined by the
merger.

**Overlap.** So that the filter has context at the seam, each half's source
extends past the middle by an overlap of 8 pixels (if scaling) plus 1 (if
enhancing).

Layer-split procedure, for a plane with source width w (or height, if
rotated 90°/270°, where "left/right" always means the left/right of the
**displayed** image) and on-screen width W, at screen x:

1. Source split:
   - with scaling: both halves get (w rounded up to even) ÷ 2 + overlap
   - enhancement only: left = (w rounded up to even) ÷ 2 + overlap;
     right = ⌊w ÷ 2⌋ + overlap
   - for YUV formats with non-zero overlap, round both halves up to even
2. Source position: the left half starts at the source x. The right half
   **ends** at the source's right edge. With horizontal mirroring, swap the
   roles: the left displayed half comes from the right of the source.
3. Screen split: left on-screen width = ⌈W ÷ 2⌉; right = W − left. Left at
   screen x, right at screen x + left width.
4. Scaled sizes: each half scales to (W × half input ÷ w). The left half
   crops the excess on its **right**. The right half crops it on its
   **left**. Each scaler's output size includes its crop.
5. Program each half on its own layer (two layers of the **same kind**, rich
   or simple, in the same pipeline), each feeding its own scaler with `LS`
   set, then:
6. Merger: `MG_INPUT_ID0` = left scaler, `MG_INPUT_ID1` = right scaler,
   `MG_SIZE` = W × H, `MG_CONTROL` = enable. Then the merger feeds one
   compositor input.

Merger limits (**[HW]**): 4 ≤ merged width ≤ the merger max line (4032 on
PERIPH parts); 4 ≤ merged height ≤ 4096.

- **[POLICY]** For a split without scaling, Linux would split into two
  halves with no overlap (left = half rounded up to even) and send both to
  the compositor directly on adjacent inputs. In Linux this path is never
  selected for planes, since nothing requests a split without scaling.
- **[POLICY]** Linux pairs a layer with the next higher-numbered layer of
  the same kind for the right half.

## 5.5 Write-back data path

The write-back layer writes the **composed image** of a display to memory.

| Need | Path |
|---|---|
| No scaling | compositor → write-back layer |
| Downscaling within the scaler's range | compositor → scaler → write-back |
| Downscaling wider than the scaler's range | compositor → splitter → scaler 0 + scaler 1 → merger → write-back |

- **[HW]** Write-back is only possible while the display's timing
  generator runs (the compositor must produce frames).
- **[HW]** Write-back scaling can only **downscale** (§5.3.1). Output size
  is the write-back framebuffer size.
- **[HW]** Splitter: `SP_INPUT_ID0` = compositor, `SP_SIZE` = input size,
  `SP_OVERLAP_SIZE` = 8 if scaling, otherwise 0, `SP_CONTROL` = enable.
  Port 0 feeds the left scaler and port 1 the right. With the hardware
  splitter in front, the scalers must **not** set `LS`.
- **[HW]** Splitter limits: 4 ≤ width ≤ the splitter max line;
  4 ≤ height ≤ max lines.
- **[POLICY]** The split arithmetic of §5.4 is reused, with the enhancer
  overlap added to the scaler halves but not to `SP_OVERLAP_SIZE`.
- **[POLICY]** The write-back connector exists only for pipelines that have
  a write-back layer, and only on the CRTC whose master is that pipeline.
