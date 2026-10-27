# 07: Userspace-Visible Interface

The DPU driver has **no private ioctls**. Userspace (compositors, Mesa's
render-only display path, fbdev emulation) uses only the standard KMS
atomic interface. This section lists the KMS objects and properties
userspace sees, so that a replacement driver behaves the same.

## 7.1 Device identity

| Item | Value | Tag |
|---|---|---|
| DRM driver name | `"komeda"` | [ABI] |
| Driver description | "Arm Komeda Display Processor driver" | informative |
| Version | 0.1 | informative |
| Features | GEM, modesetting, atomic. No render node | [ABI] |

> OPEN QUESTION: Mesa's render-only ("kmsro") loader picks a display driver
> by its DRM driver name. As far as is publicly known, `"komeda"` is on
> Mesa's list. This cannot be checked from the kernel source. If the name
> changes, Panfrost/Panthor userspace may no longer pair the GPU with this
> display device.

## 7.2 Buffers

- **[ABI]** Buffers are contiguous-DMA GEM objects (dumb buffers and
  PRIME import).
- **[ABI]** Dumb-buffer creation rounds the pitch up to a multiple of 16
  bytes: pitch = round_up(⌈width × bpp ÷ 8⌉, 16).
- **[ABI]** Framebuffer creation fails (`EINVAL`) when the format and
  modifier pair is not in 04 §4.1, when the size checks of 04 §4.3 and
  4.6 fail, or when a three-plane format has unequal chroma pitches.

## 7.3 CRTCs, encoders, connectors

- **[ABI]** One CRTC per hardware pipeline, in pipeline order. CRTC i's
  master is pipeline i. If pipeline i's compositor can take the other
  pipeline's output, that other pipeline is CRTC i's slave (00 §0.3).
- **[ABI]** CRTC properties: gamma LUT (size 4096) and CTM. No degamma.
- **[ABI]** One encoder per CRTC, followed by the bridge or panel found on
  endpoint 0 of the pipeline's DT port.
- **[ABI]** One write-back connector per CRTC whose master pipeline has a
  write-back layer. Its encoder may be cloned with every real encoder on
  that CRTC (so the display and write-back can run together). It reports
  "connected", offers no modes, and accepts modes up to 4096 × 4096. Its
  formats are the write-back column of 04 §4.1.1. Its colour formats
  follow the IPS capability.

## 7.4 Planes

- **[ABI]** One plane per hardware read layer, for every pipeline, in
  pipeline then layer order. Name = `"LPU<p>_LAYER<l>"`.
- **[ABI]** Layer 0 of each pipeline is a **primary** plane. All others
  are **overlay** planes. Only the master's layer 0 is the CRTC's primary.
- **[ABI]** A plane's possible CRTCs: every CRTC whose master **or** slave is
  the plane's pipeline.
- **[ABI]** Formats: the rich or simple column of 04 §4.1 for the layer's
  kind (each FourCC listed once). Modifiers: the list in 04 §4.2, filtered
  per format.

Plane properties:

| Property | Values | Default | Tag |
|---|---|---|---|
| rotation | 0/90/180/270, reflect-X, reflect-Y | 0 | [ABI]. Per-format limits in 04 §4.1 |
| alpha | 0 … 0xFFFF (top 8 bits are used) | opaque | [ABI] |
| pixel blend mode | None, Pre-multiplied, Coverage | Pre-multiplied (DRM default) | [ABI] |
| COLOR_ENCODING | BT.601, BT.709, BT.2020 | BT.601 | [ABI] |
| COLOR_RANGE | limited, full | limited | [ABI] |
| zpos | 0 … 8, mutable | layer index (0–3) | [ABI] |

- **[ABI]** Two planes on one CRTC with the same zpos give `EINVAL`.
- **[ABI]** A plane update on a disabled CRTC gives `EINVAL`.
- **[POLICY]** Source coordinates are truncated to whole pixels.

## 7.5 Mode validation

- **[ABI]** Interlaced modes give "no interlace".
- **[ABI]** Modes whose pixel clock (halved for dual-link) cannot be
  produced exactly give "no clock". Modes whose required `aclk` cannot be
  reached give "clock high".

## 7.6 Sysfs attributes (on the platform device)

| Attribute | Content | Tag |
|---|---|---|
| `core_id` | `GLB_CORE_ID` as `0x%08x` | [ABI] (diagnostic) |
| `config_id` | Packed configuration word, §7.6.1 | [ABI] (diagnostic) |
| `aclk_hz` | Current `aclk` rate in Hz, decimal | [ABI] (diagnostic) |

### 7.6.1 `config_id` layout

| Bits | Field |
|---|---|
| `[15:0]` | Maximum source width of pipeline 0, layer 0 |
| `[17:16]` | Number of pipelines |
| `[19:18]` | Scalers per pipeline (pipeline 0) |
| `[22:20]` | Layers per pipeline (pipeline 0) |
| `[25:23]` | Rich layers per pipeline (pipeline 0) |
| `[31:26]` | 0 |

## 7.7 Debug interfaces (informative)

- **[POLICY]** debugfs `komeda/register` dumps registers. `komeda/err_verbosity`
  selects which error events are logged. These are not ABI.
