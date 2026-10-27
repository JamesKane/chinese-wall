# 01: Platform Integration

## 1.1 Devicetree binding

Source: the public devicetree binding for the "Arm Komeda display processor"
(dual-licensed GPL-2.0-only / BSD-2-Clause), summarised here.

### 1.1.1 DPU node

| Property | Requirement | Meaning |
|---|---|---|
| `compatible` | required | One of: `"arm,mali-d71"`; `"arm,mali-d32", "arm,mali-d71"`; `"armchina,linlon-d6", "arm,mali-d71"` |
| `reg` | required, 1 entry | The whole DPU register space (all blocks). The binding's example uses 128 KiB |
| `interrupts` | required, 1 entry | One combined interrupt line for all blocks |
| `clocks` / `clock-names` | required, 1 entry named `"aclk"` | Main processor (engine/AXI) clock |
| `#address-cells` = 1, `#size-cells` = 0 | required | Child `pipeline@N` nodes are addressed by index |
| `memory-region` | optional, 1 phandle | Reserved memory for framebuffers. Without it, buffers may be anywhere |
| `iommus` | optional, 5–10 entries | IOMMU stream IDs. Per pipeline: 4 for the four read layers and 1 for the write-back layer |
| `pipeline@0` | required | First pipeline |
| `pipeline@1` | optional | Second pipeline |

- **[HW]** The binding's `iommus` example orders the IDs as pipeline-0
  layers 0–3, pipeline-0 write-back, pipeline-1 layers 0–3, pipeline-1
  write-back. Each layer has its own AXI stream ID.

### 1.1.2 Pipeline child node

| Property | Requirement | Meaning |
|---|---|---|
| `reg` | required, 0 or 1 | Pipeline index. Must match the hardware pipeline number |
| `clocks` / `clock-names` | required, 1 entry named `"pxclk"` | Pixel clock reference for this pipeline's timing generator |
| `port` | graph port | Output links to the encoder or bridge. Endpoint 0 is the first link. Endpoint 1, if present, is the second link (dual-link) |

- **[POLICY]** Linux requires a DT node for **every** hardware pipeline it
  discovers and fails probe otherwise. Nodes with an index ≥ the hardware
  pipeline count are ignored with a warning.
- **[POLICY]** A pipeline is treated as dual-link when its port has **both**
  endpoint 0 and endpoint 1 connected. If the timing generator does not
  report dual-link support (02 §2.4), the DT dual-link request is ignored.
- **[POLICY]** Only endpoint 0 is attached as the downstream bridge. A
  pipeline with no endpoint gets no bridge (it can still drive write-back).

## 1.2 Clocks

| Clock | Scope | Requirement |
|---|---|---|
| `aclk` | whole DPU | **[HW]** Must be running for any register access, including identification and reset |
| `pxclk` | per pipeline | **[HW]** Must run at the pixel rate of that pipeline's output timing (halved for dual-link, 06 §6.4) |

- **[HW]** `aclk` must be at least the pixel clock, and at least **twice**
  the pixel clock on a dual-link pipeline (06 §6.5). Downscaling adds a
  stronger requirement (05 §5.3.4).
- **[POLICY]** Linux sets `aclk` to the clock framework's rounded value for
  that minimum when the first display turns on, and does not change it when
  the second display turns on. It gates `aclk` at runtime suspend and when
  both displays are off.
- **[POLICY]** Linux rejects a mode when `pxclk` cannot be produced at
  exactly the needed rate (the clock framework's rounded rate must equal the
  request), or when the best `aclk` rate is below the minimum.

## 1.3 Output links

- **[HW]** Each pipeline's timing generator drives a parallel pixel bus
  toward an encoder (for example a DisplayPort or HDMI controller). The DPU
  does not include the encoder or PHY.
- **[POLICY]** Linux exposes one encoder of type "TMDS" per pipeline as a
  placeholder and chains the real encoder as a bridge. The type has no
  hardware meaning.

## 1.4 Interrupt

- **[HW]** One level interrupt line. Its source is decoded through
  `GLB_IRQ_STATUS` (03 §3.5).
- **[POLICY]** Linux requests the line as shared.

## 1.5 Memory access and IOMMU

- **[HW]** The DPU is a 40-bit AXI master. All pointer registers are 64-bit
  low/high pairs (02).
- **[POLICY]** Linux sets a 40-bit DMA mask and a maximum DMA segment size
  of 4 GiB − 1.
- **[HW]** Without an IOMMU, every plane of a framebuffer must be physically
  contiguous (Linux allocates through a contiguous-DMA allocator).
- **[HW]** The DPU may include **integrated TBUs** (translation units of an
  Arm SMMU-style system MMU, one per pipeline). If the capability bit is set
  (02 §2.4) and the platform puts the DPU behind an IOMMU, the TBUs must be
  connected to the TCU at power-up (03 §3.7). Each layer then sets its
  "TBU enable" bit so that its addresses are translated (04 §4.4).
- **[POLICY]** Linux decides "addresses are virtual" per framebuffer, using
  only whether the device has an IOMMU domain.

- **[HW]** *(vendor)* The per-layer TBU-enable bit is for the **integrated**
  TBUs only. Behind an external SMMU with no integrated TBU, leave it 0.
  (Mainline Linux sets it whenever an IOMMU domain exists. The CIX vendor
  driver sets it only when the integrated-TBU capability bit is also set.)

## 1.6 Power management

Resume sequence (power-up), **[HW]** order:

1. Make the power domain active (platform specific, 08).
2. Enable `aclk`.
3. Unmask the interrupt sources (03 §3.5.3).
4. If an IOMMU is in use and TBUs are integrated, connect the TBUs (03 §3.7).
   Linux logs a failure here but does not fail the resume (**[POLICY]**).

Suspend is the reverse: disconnect the TBUs, mask interrupts, gate `aclk`.

- **[POLICY]** Linux holds a runtime-PM reference while any display is on,
  and reruns the resume sequence after system suspend only if the device
  was not already runtime-suspended.
- **[HW]** The binding has no `resets` property. The Sky1 vendor
  devicetree adds `resets`/`reset-names` (`"rcsu_reset"`, `"ip_reset"`);
  see 08 §8.4.
- **[POLICY]** At probe, Linux enables `aclk` only for identification and
  block discovery and gates it again before registering the display
  device.
