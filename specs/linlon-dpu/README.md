# Linlon-D6 DPU (Komeda-family display processor) Functional Specification

Target: the **Arm China Linlon-D6** display processor, as used in the
**CIX Sky1** SoC. The D6 is register-compatible with the **Arm Mali-D71**
for basic modesetting, so this spec describes the D71 programming model
(which also covers the Mali-D32) and notes where the D6 is unknown.

These documents describe hardware behaviour and the userspace-visible
interface so that a driver can be written without looking at the Linux
source. Each statement is tagged:

- **[HW]**: required by the hardware
- **[POLICY]**: a Linux design choice, informative only
- **[ABI]**: needed for compatibility with existing userspace (generic KMS
  clients, Mesa render-only display pairing)

| # | Section | Status |
|---|---------|--------|
| 00 | [Overview and architecture](00-overview.md) | draft |
| 01 | [Platform integration (DT, clocks, IRQ, IOMMU, power)](01-platform.md) | draft |
| 02 | [Register map](02-register-map.md) | draft |
| 03 | [Device control: discovery, modes, flush, interrupts, TBU, disable](03-device-control.md) | draft |
| 04 | [Layers, pixel formats and AFBC](04-layers.md) | draft |
| 05 | [Composition, scaling, split/merge, write-back path](05-composition.md) | draft |
| 06 | [Display output: colour processing and timing](06-output.md) | draft |
| 07 | [Userspace-visible interface](07-uapi.md) | draft |
| 08 | [CIX Sky1 integration](08-cix-sky1.md) | partial: no upstream display node or DP controller driver at baseline |

## Scope limits

- Linux uses only part of the hardware. Features it never programs are
  listed with their register locations but no behaviour: the AFBC encoder
  (AEU), coprocessor, input/forward transforms on layers, background
  colour, frame CRC, tearing-effect and command-mode timing, scaler
  enhancer coefficients, RGB→YUV output matrices.
- The encoder (the Sky1 DisplayPort controller) is out of scope.
