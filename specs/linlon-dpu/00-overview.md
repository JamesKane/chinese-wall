# 00: Overview and Architecture

## 0.1 What the hardware is

The **Arm China Linlon-D6** is a display processor (DPU). It belongs to the
same family as the Arm **Mali-D71** and **Mali-D32** display processors (Arm
calls the family "Komeda"). At the register level the Linlon-D6 is presented
as **compatible with the Mali-D71 for basic modesetting**:

- **[HW]** It reports product ID `0x0060` in `GLB_CORE_ID` (02 §2.3), at the
  same offset as the D71 and D32.
- **[HW]** It exposes the same self-describing block layout as the D71
  (02 §2.1), so a driver discovers its blocks the same way.
- **[HW]** The devicetree binding lists `armchina,linlon-d6` with
  `arm,mali-d71` as fallback (01 §1.1).

The Linux support for the D6 is just the D71 programming model, with the D6
product ID added to the accept list. It touches no D6-specific feature.
Everything in this spec is therefore the **D71 programming model**, which the
D6 is stated to accept. Features the D6 may have beyond the D71 are not
described anywhere in the source.

- The CIX vendor driver for the D6 (shipped by the Sky1-Linux project) is a
  fork of the same D71 code with D6 additions. Those additions are folded
  into this spec and marked *(vendor)*: a changed AXI control layout,
  2 pixels per clock, side-by-side composition, per-layer degamma/matrix/
  gamma, display degamma, and RGB→YUV matrices (summary in 08 §8.7).

> OPEN QUESTION: Further D6 capabilities that neither driver uses are still
> unknown. A clean-room driver should read the self-description registers
> (`GLB_CORE_INFO`, `GCU_CONFIGURATION_ID0/1`, per-block `BLOCK_INFO`) and
> skip any block type it does not recognise, as described in 03 §3.2.

## 0.2 Block architecture

The DPU is a set of 512-byte register **blocks**, each describing itself.
Blocks are joined into one or two **pipelines**. Each pipeline has three
main units, plus global units shared by all pipelines.

```
            ┌──────────── pipeline n ──────────────────────────────────┐
 memory ──► │ LPU: Layer0..3 ──► CU: [Scaler0/1] ──► Compositor ──►   │
 (AXI read) │      (AFBC                [Splitter/Merger]             │
            │       decode)                       │                    │
            │                                     ├──► DOU: IPS ──► BS ├──► pixel bus
            │ LPU: Write-back layer ◄─────────────┘   (colour)  (timing)│    to encoder
 memory ◄── │      (AXI write)                                         │
            └──────────────────────────────────────────────────────────┘
   global:  GCU (control, mode, reset, flush) · scaler/LUT coefficient blocks
            · PERIPH (legacy configuration block, D71 only)
```

| Unit | Mnemonic | Role | Sub-blocks |
|---|---|---|---|
| Global Control Unit | GCU | Identification, soft reset, operating mode, config-valid (flush), top-level IRQ summary | none |
| Layer Processing Unit | LPU | Fetches pixels from memory. AXI and TBU (IOMMU) control | up to 4 read layers, 1 write-back layer |
| Composition Unit | CU | Blends up to 5 inputs. Scaling and line splitting | up to 2 scalers, 1 splitter, 1 merger |
| Display Output Unit | DOU | Colour processing and video timing | IPS (image processor), BS (back-end / timing generator), FT coefficient table |
| Peripheral | PERIPH | Configuration word on the original D71 only. Newer parts fold it into the GCU | none |

- **[HW]** Every block has a common header that gives its type, its
  instance number, and which other blocks may feed it (02 §2.2). A block's
  inputs are chosen by writing an **input ID** (the source block's
  type/instance plus an output port number) into the block's `INPUT_ID`
  registers. Writing 0 disconnects the input.
- **[HW]** Each layer is either **rich** (YUV capable, with chroma upsampling
  and a YUV→RGB matrix) or **simple** (RGB only). `LAYER_INFO` tells which.

## 0.3 Pipelines, displays and master/slave

- **[HW]** The DPU has 1 or 2 pipelines (`GLB_CORE_INFO`). Each pipeline has
  its own timing generator and can drive its own display.
- **[HW]** The compositor of one pipeline may accept the **output of the other
  pipeline's compositor** as an input. The receiving pipeline is then the
  *master*. The other is its *slave*: the slave's layers are blended first
  and then fed as the bottom input of the master's compositor. This lets one
  display use the layers of both pipelines. Whether this path exists is read
  from the master compositor's valid-input list.
- **[HW]** A pipeline may be **dual-link**: one pipeline drives two output
  links, each carrying half of every line. Dual-link support is a capability
  bit (02 §2.4). Using it is a board decision (01 §1.3).
- **[HW]** The GCU operating mode (03 §3.3) says which timing generators
  run: none, display 0, display 1, or both.

## 0.4 Register update model

- **[HW]** Almost all block registers are **shadowed**. Writes collect in
  the shadow copy and take effect only when software sets the
  **config-valid** bit for that pipeline (`GCU_CONFIG_VALID0/1`). The
  hardware latches the shadow set at the next frame boundary and raises the
  matching GCU `CVAL` interrupt. That interrupt is the "flip done" signal.
- **[HW]** The hardware does **not** latch shadow registers once the display
  output is off. Disabling a pipeline therefore takes two flushes (03 §3.6).

## 0.5 What the driver must do (summary)

1. Power on, enable the main clock (`aclk`), map the single MMIO region,
   read the product ID (01, 03 §3.1).
2. Soft-reset, then walk the block list and record each block's address and
   capabilities (03 §3.2).
3. Connect the integrated IOMMU TBUs if present and an IOMMU is in use
   (03 §3.7).
4. For each frame: program layers (04), compositor, scalers and so on (05),
   and colour and timing (06). Then set config-valid and wait for `CVAL`.
5. Handle one interrupt line that summarises all blocks (03 §3.5).

## 0.6 Document map

| # | Section |
|---|---|
| 01 | Platform integration: devicetree, clocks, interrupt, IOMMU, power |
| 02 | Register map |
| 03 | Device control: identification, discovery, modes, flush, interrupts, TBU, disable sequence |
| 04 | Layers: pixel formats, AFBC, layer and write-back programming |
| 05 | Composition, scaling, split/merge, write-back data path |
| 06 | Display output: colour processing and timing generator |
| 07 | Userspace-visible interface |
| 08 | CIX Sky1 integration |
