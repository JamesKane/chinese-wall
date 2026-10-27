# 02: Register Map

All registers are 32-bit and accessed with 32-bit reads and writes. Offsets
are bytes from the start of the block. 64-bit addresses are written as a
`_LOW` word followed by a `_HIGH` word at offset + 4.

Field notation: `[m:n]` is a bit range, `[n]` a single bit.

## 2.1 Address space layout

| Range | Contents |
|---|---|
| `0x0000–0x01FF` | GCU (always block 0) |
| `0x0200 × k` for k = 1 … N−1 | Block k. N is `GLB_CORE_INFO[7:0]` (03 §3.2) |
| `0xFE00–0xFFFF` | PERIPH block, **only** on parts that have one (the original D71) |

- **[HW]** Every block is exactly `0x200` bytes. Blocks are packed in
  order. Some slots are **reserved** (type `0xFF`) and must be skipped.

## 2.2 Common block header (all blocks except where noted)

| Offset | Name | Access | Fields |
|---|---|---|---|
| `0x000` | `BLOCK_INFO` | RO | `[3:0]` number of sub-blocks; `[7:4]` block instance ID; `[15:8]` block type (§2.2.1). `[15:4]` together form the block's **input ID base** |
| `0x004` | `PIPELINE_INFO` | RO | `[3:0]` number of output IDs; `[11:8]` number of valid input IDs |
| `0x008` | `MAX_LINE_SIZE` | RO | Line-buffer width in pixels for this block. **Not present on parts with a PERIPH block**, which use the global line size instead |
| `0x020 + 4i` | `VALID_INPUT_ID<i>` | RO | Input IDs this block can accept. i < `PIPELINE_INFO[11:8]` (up to 9) |
| `0x060 + 4i` | `OUTPUT_ID<i>` | RO | Output IDs. i < `PIPELINE_INFO[3:0]` (up to 5) |
| `0x080 + 4i` | `INPUT_ID<i>` | RW | Selected source for input i (§2.2.2) |
| `0x0A0` | `IRQ_RAW_STATUS` | RO | Raw interrupt bits (block specific) |
| `0x0A4` | `IRQ_CLEAR` | **WO** | Write 1s to clear the matching raw bits. Do not read |
| `0x0A8` | `IRQ_MASK` | RW | 1 = source enabled |
| `0x0AC` | `IRQ_STATUS` | RO | Raw status AND mask |
| `0x0B0` | `STATUS` | RW | Sticky error and state bits (block specific) |
| `0x0C0` | `INFO` | RO | Block-specific capability word |
| `0x0D0` | `CONTROL` | RW | Block control. Bit 0 is "enable" for most blocks |
| `0x0D4` | `SIZE` | RW | Output size, in `HV_SIZE` encoding (§2.2.3), for blocks that have one |
| `0x0E0` | `IN_SIZE` | RW | Input size, in `HV_SIZE` encoding, for blocks that have one |
| `0x100…` | plane pointers | RW | See the layer blocks |

- **[HW]** `STATUS` error bits are sticky. Linux clears an error bit by a
  read-modify-write that writes 0 to just that bit and the read value to
  every other bit.

> OPEN QUESTION: Whether `STATUS` error bits are "write 0 to clear" or
> plain read/write is not stated. Linux's write-back would set any other bit
> that read back as 1, which is harmless only if the other bits are
> read-only or write-1-has-no-effect. The CU path masks out bit 31 before the
> write-back, which hints that bit 31 (`ACTIVE`) should not be written as 1.

### 2.2.1 Block types

| Type | Block | Notes |
|---|---|---|
| `0x00` | GCU | |
| `0x01` | LPU | One per pipeline. Instance = pipeline |
| `0x02` | CU | One per pipeline. Instance = pipeline |
| `0x03` | DOU | One per pipeline. Instance = pipeline |
| `0x04` | AEU | Not used by Linux |
| `0x05` | Global LT coefficient table | Not programmed by Linux |
| `0x06` | Global scaler coefficient table | Up to 4 instances. Address recorded, never programmed |
| `0x07` | Global SC coefficient table | Not used |
| `0x08` | PERIPH | |
| `0x09` | LPU (trusted) | Not used |
| `0x0A` | AEU (trusted) | Not used |
| `0x10` | LPU read layer | Instance = pipeline × 4 + layer |
| `0x11` | LPU write-back layer | Instance = pipeline |
| `0x20` | CU splitter | Instance = pipeline |
| `0x21` | CU scaler | Instance = pipeline × 2 + scaler |
| `0x22` | CU merger | Instance = pipeline |
| `0x30` | DOU image processor (IPS) | Instance = pipeline |
| `0x31` | DOU back-end / timing (BS) | Instance = pipeline |
| `0x32` | DOU forward-transform (gamma) coefficient table | Instance = pipeline |
| `0x40` | AEU DS | Not used |
| `0x41` | AEU AES | Not used |
| `0xFF` | Reserved slot | Skip |

- **[POLICY]** Linux fails probe on any block type not in the "used" set
  above (it accepts 0x00–0x03, 0x05, 0x06, 0x10, 0x11, 0x20–0x22, 0x30–0x32,
  and treats 0x05 as a no-op).

> OPEN QUESTION: If a Linlon-D6 exposes AEU (types 0x04, 0x40, 0x41;
> probably an AFBC encoder for write-back) or trusted blocks, a driver that
> fails on unknown types will not probe. A clean-room driver should skip
> unknown types instead.

### 2.2.2 Input ID encoding

- **[HW]** An input ID is `(source BLOCK_INFO & 0xFFF0) + output_port`:
  - `[15:8]` source block type
  - `[7:4]` source block instance
  - `[3:0]` source output port (0 for every block except the splitter,
    which has ports 0 = left and 1 = right)
- **[HW]** Writing 0 to an `INPUT_ID` register disconnects that input.

### 2.2.3 Size and offset encodings

| Encoding | Layout |
|---|---|
| `HV_SIZE` | `[12:0]` horizontal, `[28:16]` vertical (pixels, up to 8191) |
| `HV_OFFSET` | `[11:0]` horizontal, `[27:16]` vertical (pixels, up to 4095) |
| `HV_CROP` | `[11:0]` first edge (left or top), `[27:16]` second edge (right or bottom) |

## 2.3 GCU (block 0)

| Offset | Name | Access | Fields |
|---|---|---|---|
| `0x000` | `GLB_ARCH_ID` | RO | Architecture ID (only recorded) |
| `0x004` | `GLB_CORE_ID` | RO | `[31:16]` product ID; `[15:12]` major version; `[11:4]` minor version; `[3:0]` status |
| `0x008` | `GLB_CORE_INFO` | RO | `[7:0]` total block slots, **including** the GCU and reserved slots; `[10:8]` number of pipelines |
| `0x010` | `GLB_IRQ_STATUS` | RO | Top-level interrupt summary (§2.3.2) |
| `0x0A0–0x0AC` | `GCU_IRQ_RAW_STATUS/CLEAR/MASK/STATUS` | | GCU interrupt (§2.3.3) |
| `0x0B0` | `GCU_STATUS` | RO/RW | §2.3.4 |
| `0x0D0` | `GCU_CONTROL` | RW | `[2:0]` operating mode request (§2.3.1); `[16]` `SRST` soft reset (self-clearing) |
| `0x0D4` | `GCU_CONFIG_VALID0` | RW | `[0]` `CVAL`: write 1 to latch pipeline 0's shadow registers at the next frame boundary. On the Linlon-D6 it reads 1 until the latch has happened, then 0 *(vendor)* |
| `0x0D8` | `GCU_CONFIG_VALID1` | RW | `[0]` `CVAL`: the same for pipeline 1 |
| `0x100` | `GCU_CONFIGURATION_ID0` | RO | `[15:0]` maximum line size (pixels); `[31:16]` maximum number of lines. Valid only on parts **without** PERIPH |
| `0x104` | `GCU_CONFIGURATION_ID1` | RO | `[2:0]` rich layers; `[5:3]` pipelines; `[8:6]` scalers; `[16]` dual-link (display split) supported; `[17]` integrated TBU present. Valid only on parts without PERIPH |

Known product IDs:

| `GLB_CORE_ID[31:16]` | Product |
|---|---|
| `0x0071` | Mali-D71 |
| `0x0032` | Mali-D32 |
| `0x0060` | Linlon-D6 |

- **[HW]** The minor-version field is 8 bits at `[11:4]` and status is 4 bits
  at `[3:0]`. (Earlier Linux releases decoded these wrongly as 4 bits at
  `[11:8]` and 8 bits at `[7:0]`. Any version number logged by an old kernel
  is unreliable.)

### 2.3.1 Operating modes (`GCU_CONTROL[2:0]`, `GCU_STATUS[2:0]`)

| Value | Mode |
|---|---|
| 0 | Inactive |
| 1 | TBU connect |
| 2 | TBU disconnect |
| 3 | Display 0 active |
| 4 | Display 1 active |
| 5 | Displays 0 and 1 active |

### 2.3.2 `GLB_IRQ_STATUS` (0x010)

| Bit | Source |
|---|---|
| 0 | GCU |
| 8 | LPU0 |
| 9 | LPU1 |
| 10, 11 | ATU0, ATU1 (pipeline 0) |
| 12, 13 | ATU2, ATU3 (pipeline 1) |
| 16 | CU0 |
| 17 | CU1 |
| 24 | DOU0 |
| 25 | DOU1 |

> OPEN QUESTION: "ATU" blocks are never decoded by Linux. They may belong
> to the AEU or to TBU address translation.

### 2.3.3 GCU interrupt bits

| Bit | Name | Meaning |
|---|---|---|
| 0 | `CVAL0` | Pipeline 0 shadow registers were latched (flip done) |
| 1 | `CVAL1` | The same for pipeline 1 |
| 4 | `MODE` | Operating-mode change completed (enabled by Linux, never acted on) |
| 11 | `ERR` | Error. See `GCU_STATUS.MERR` |

### 2.3.4 `GCU_STATUS` (0x0B0)

| Bit | Name | Meaning |
|---|---|---|
| `[2:0]` | `MODE` | Current operating mode |
| 4 | `MERR` | Mode error (sticky) |
| 8 | `TCS0` | TBU 0 connected to the TCU |
| 9 | `TCS1` | TBU 1 connected to the TCU |
| 31 | `ACTIVE` | GCU active |

## 2.4 PERIPH (offset `0xFE00`, legacy D71 only)

Present only if `BLOCK_INFO` at `0xFE00` reads block type `0x08`.

| Offset | Name | Fields |
|---|---|---|
| `0x1D4` | `PERIPH_CONFIGURATION_ID` | `[0]` 1 = 4096-pixel lines, 0 = 2048; `[4]` 1 = 2 rich layers, 0 = 1; `[8]` dual-link supported; `[12]` integrated TBU present; `[16]` AFBC DMA present |

- **[HW]** With a PERIPH block, the maximum number of lines is 4096 and
  blocks have no per-block `MAX_LINE_SIZE` register.

## 2.5 LPU (type 0x01)

| Offset | Name | Fields |
|---|---|---|
| `0x0A0–0x0AC` | IRQ registers | §2.5.1 |
| `0x0B0` | `LPU_STATUS` | §2.5.2 |
| `0x0B4` | `LPU_TBU_STATUS` | §2.5.3 |
| `0x0C0` | `LPU_INFO` | Not decoded |
| `0x0D0` | `LPU_RAXI_CONTROL` | `[6:0]` outstanding-transaction cap A; `[14:8]` outstanding cap B; `[15]` cap-B enable; `[21:16]` burst length; `[27:24]` AxQOS; `[31]` ordering |
| `0x0D4` | `LPU_WAXI_CONTROL` | `[5:0]` outstanding-write cap; `[21:16]` burst length; `[27:24]` AxQOS; `[31]` ordering |
| `0x0D8` | `LPU_TBU_CONTROL` | `[5:0]` TBU outstanding cap; `[16]` `TLBPEN` (TLB prefetch enable) |

- **[POLICY]** Linux never writes the AXI control registers. It leaves the
  reset values. The field layout above is the D71 one. **The Linlon-D6 uses
  a different layout (§2.5.4).**

### 2.5.1 LPU interrupt bits

| Bit | Name | Meaning |
|---|---|---|
| 9 | `OVR` | Overrun |
| 10 | `IBSY` | Input busy (meaning inferred from the name only) |
| 11 | `ERR` | Error. See `LPU_STATUS` and `LPU_TBU_STATUS` |
| 12 | `EOW` | End of write-back frame |
| 13 | `PL0` | Programmable line 0 reached |

### 2.5.2 `LPU_STATUS`

| Bit | Name |
|---|---|
| `[3:0]` | `AXIED`: ID of the AXI transaction that errored |
| 4 | `AXIE`: AXI error (sticky) |
| 5 | `AXIRP`: AXI read pending |
| 6 | `AXIWP`: AXI write pending |
| 11 | `FEMPTY`: FIFO empty (sticky) |
| 14 | `FFULL`: FIFO full (sticky) |
| 16–19 | `ACE0`–`ACE3`: per-layer error for layers 0–3, probably AFBC decode errors (inferred) (sticky) |
| 31 | `ACTIVE` |

### 2.5.3 `LPU_TBU_STATUS`

| Bit | Name |
|---|---|
| 1 | `TCF` |
| 2 | `TTNG` |
| 8 | `TITR` |
| 16 | `TEMR` |
| 31 | `TTF` |

All are sticky error flags.

> OPEN QUESTION: The meaning of the TBU status bits is not given beyond
> their mnemonics. They plausibly report translation-configuration fault,
> translation-table not granted, invalid translation request, error
> response from the TCU, and translation fault, but this is a guess.

### 2.5.4 Linlon-D6 AXI control fields *(vendor)*

**[HW]** On the Linlon-D6 the LPU AXI registers are laid out as follows.
"Default" is the value the vendor driver treats as already present (it
skips writing a value equal to it), so it is presumably the reset value.

| Register | Field | Bits | Allowed values | Default |
|---|---|---|---|---|
| `LPU_RAXI_CONTROL` | outstanding cap A | `[7:0]` | 4 … 128 | 32 |
| | outstanding cap B | `[15:8]` | 4 … 128 | 32 |
| | cap-B enable | `[22]` | 0, 1 | 0 |
| | burst length | `[21:16]` | 4, 8, 16, 32 | 16 |
| | ARQOS | `[27:24]` | 0 … 15 | 15 |
| | ordering | `[31]` | 0, 1 | 0 |
| `LPU_WAXI_CONTROL` | outstanding cap | `[5:0]` | 1 … 32 | 16 |
| | burst length | `[21:16]` | 4, 8, 16, 32 | 16 |
| | AWQOS | `[27:24]` | 0 … 15 | 15 |
| | ordering | `[31]` | 0, 1 | 0 |
| `LPU_TBU_CONTROL` | TBU outstanding cap | `[5:0]` | 1 … 16 | 8 |
| `L_CONTROL` / `LW_CONTROL` | `ARCACHE` / `AWCACHE` | `[31:28]` | 0x0, 0x1, 0x2, 0x3, 0x7, 0xA, 0xB, 0xE, 0xF | 0x3 |

> OPEN QUESTION: The burst-length field is written as the plain beat count
> (16 is written as 16, not as a code), so values above 32 would spill past
> bit 21. The vendor source keeps the D71 bit positions for cap B in
> comments, which suggests the move to bit 22 is a deliberate D6 change.

- **[POLICY]** *(vendor)* The vendor driver reads these as optional
  per-pipeline devicetree properties (`lpu_raxi_aoutstdcapb`,
  `lpu_raxi_boutstdcapb`, `lpu_raxi_ben`, `lpu_raxi_burstlen`,
  `lpu_raxi_arqos`, `lpu_raxi_ord`, `lpu_waxi_outstdcapb`,
  `lpu_waxi_burstlen`, `lpu_waxi_awqos`, `lpu_waxi_ord`,
  `lpu_tbu_doutstdcapb`, `lpu_l0_arcache` … `lpu_l3_arcache`,
  `lpu_lw_arcache`) and writes them after block discovery and after each
  resume. The Orion O6 devicetree sets exactly the defaults.

## 2.6 Read layer (type 0x10)

| Offset | Name | Fields |
|---|---|---|
| `0x020…`, `0x060…` | header | Common |
| `0x008` | `MAX_LINE_SIZE` | Parts without PERIPH: maximum line in pixels |
| `0x0C0` | `LAYER_INFO` | `[0]` `RF`: rich layer; `[1]` `CM`: has an RGB→RGB matrix; `[6:4]` AFBC buffer size code; `[31:16]` maximum YUV line size (parts without PERIPH) |
| `0x0D0` | `L_CONTROL` | §2.6.1 |
| `0x0D4` | `L_RICH_CONTROL` | Rich layers only. §2.6.2 |
| `0x0D8` | `L_FORMAT` | Hardware pixel-format ID (04 §4.1) |
| `0x0DC` | `L_LT_COEFFTAB` | `[11:0]` input-transform (degamma) table select; `[27:16]` forward-transform (gamma) table select. A select value is the table block's `BLOCK_INFO[11:4]` (low nibble of the type, then the instance). Not written by mainline Linux; used by the vendor driver (04 §4.9) |
| `0x0E0` | `L_IN_SIZE` | `HV_SIZE`: source size |
| `0x0E4` | `L_PALPHA` | Alpha expansion map (04 §4.4) |
| `0x100` / `0x104` | `P0_PTR_LOW/HIGH` | Plane 0 address (AFBC: header address) |
| `0x108` | `P0_STRIDE` | Plane 0 stride in bytes (04 §4.4) |
| `0x110` / `0x114` | `P1_PTR_LOW/HIGH` | Plane 1 address (AFBC: payload or end address) |
| `0x118` | `P1_STRIDE` | Plane 1 stride (rich layers) |
| `0x120` / `0x124` | `P2_PTR_LOW/HIGH` | Plane 2 address (rich layers) |
| `0x130 + 4i`, i = 0–11 | `YUV_RGB_COEFF<i>` | YUV→RGB matrix (rich layers; 04 §4.5) |
| `0x160` | `AD_CONTROL` | AFBC decoder control (§2.6.3) |
| `0x164` | `AD_H_CROP` | `HV_CROP`: AFBC left/right crop |
| `0x168` | `AD_V_CROP` | `HV_CROP`: AFBC top/bottom crop |
| `0x170 + 4i`, i = 0–11 | `RGB_RGB_COEFF<i>` | RGB→RGB matrix (if `LAYER_INFO.CM`). Not written by mainline Linux. The vendor driver writes i = 0–8 in the same Q3.12 format as the IPS matrix (06 §6.2.1) |

### 2.6.1 `L_CONTROL`

| Bits | Name | Meaning |
|---|---|---|
| 0 | `EN` | Layer enable |
| 4 | `IT` | Input-transform (degamma) table enable (vendor only) |
| 5 | `R2R` | RGB→RGB matrix enable (vendor only) |
| 6 | `FT` | Forward-transform (gamma) table enable (vendor only) |
| `[9:8]` | `ROT` | 0 = 0°, 1 = 90°, 2 = 180°, 3 = 270° |
| 10 | `HFLIP` | Mirror horizontally |
| 11 | `VFLIP` | Mirror vertically |
| 16 | `TBU_EN` | Addresses are IOMMU virtual addresses |
| `[31:28]` | `ARCACHE` | AXI read cache attributes: bit 28 bufferable, 29 modifiable, 30 read-allocate, 31 write-allocate |

- **[HW]** Rotation is counter-clockwise, which matches the DRM convention.
- **[POLICY]** Linux updates only `EN`, `ROT`, `HFLIP`, `VFLIP` and `TBU_EN`
  through read-modify-write. `ARCACHE` keeps its reset value (`0x3`,
  bufferable + modifiable, is the suggested default).

### 2.6.2 `L_RICH_CONTROL`

| Bits | Meaning |
|---|---|
| `[1:0]` | 4:2:2 chroma upsampling: 0 = bilinear, 1 = replication |
| `[2]` | 4:2:0 chroma upsampling: 0 = JPEG-style, 1 = MPEG-style |

> OPEN QUESTION: The exact field widths are inferred from the two values
> used. Bits above 2 are unknown.

### 2.6.3 `AD_CONTROL` (AFBC decoder)

| Bit | Name | Meaning |
|---|---|---|
| 0 | `AEN` | AFBC decode enable. 0 = linear buffer |
| 1 | `YT` | YUV-transform (YTR) was applied by the encoder |
| 2 | `BS` | Split-block mode |
| 3 | `WB` | Wide block: 32×8 superblocks (0 = 16×16) |
| 4 | `TH` | Tiled headers (AFBC 1.2 layout) |

## 2.7 Write-back layer (type 0x11)

| Offset | Name | Fields |
|---|---|---|
| `0x080` | `LW_INPUT_ID0` | Source (the compositor, a scaler or the merger) |
| `0x0D0` | `LW_CONTROL` | `[0]` `EN`; `[4]` `OFM` output-format mode (Linux always sets it); `[15:8]` `LALPHA` alpha value written for formats with alpha; `[16]` `TBU_EN`; `[31:28]` `AWCACHE` |
| `0x0D4` | `LW_PROG_LINE` | Programmable line (not written by Linux) |
| `0x0D8` | `LW_FORMAT` | Hardware pixel-format ID |
| `0x0E0` | `LW_IN_SIZE` | `HV_SIZE` |
| `0x100`/`0x104`/`0x108` | `LW_P0_PTR_LOW/HIGH`, `LW_P0_STRIDE` | Plane 0 |
| `0x110`/`0x114`/`0x118` | `LW_P1_PTR_LOW/HIGH`, `LW_P1_STRIDE` | Plane 1 (NV12 chroma) |
| `0x130 + 4i`, i = 0–11 | `LW_RGB_YUV_COEFF<i>` | RGB→YUV matrix for YUV output. Not written by Linux |

- **[HW]** The suggested reset value of `LW_CONTROL` is `0x3000FF00`
  (`AWCACHE` = 3, `LALPHA` = 0xFF, disabled).

- **[HW]** For YUV output (NV12) the RGB→YUV matrix must be loaded
  (04 §4.8). Mainline Linux does not do this. The vendor driver does, with
  the encoding and range chosen on the write-back connector.

## 2.8 CU / compositor (type 0x02)

| Offset | Name | Fields |
|---|---|---|
| `0x080 + 4n`, n = 0–4 | `CU_INPUT_ID<n>` | Source of compositor input n |
| `0x0A0–0x0AC` | IRQ registers | `[9]` `OVR` overrun; `[11]` `ERR` |
| `0x0B0` | `CU_STATUS` | `[0]` `CPE` coprocessor error; `[1]` `ZME` (zorder or merge error, inferred); `[2]` `CFGE` configuration error; `[31]` `ACTIVE` |
| `0x0D0` | `CU_CONTROL` | `[0]` `COPROC`: coprocessor enable. Not set by Linux |
| `0x0D4` | `CU_SIZE` | `HV_SIZE`: composition (output) size |
| `0x0DC` | `CU_BG_COLOR` | Background colour. Not written by Linux |
| `0x0E0 + 0x10n` | `CU_INPUT<n>_SIZE` | `HV_SIZE` of input n |
| `0x0E4 + 0x10n` | `CU_INPUT<n>_OFFSET` | `HV_OFFSET`: position of input n in the output |
| `0x0E8 + 0x10n` | `CU_INPUT<n>_CONTROL` | `[0]` `EN`; `[1]` `PAD` (ignore pixel alpha, treat as opaque); `[2]` `PMUL` (pixel colour is premultiplied); `[15:8]` plane alpha |
| `0x130` / `0x134` | `CU_USER_LOW/HIGH` | User registers. Not used |

- **[HW]** Compositor inputs are blended in **index order**: input 0 is the
  bottom and input 4 the top (05 §5.1).
- **[HW]** Disabling an input needs **both** `CU_INPUT_ID<n>` = 0 and
  `CU_INPUT<n>_CONTROL.EN` = 0.

## 2.9 Scaler (type 0x21)

| Offset | Name | Fields |
|---|---|---|
| `0x080` | `SC_INPUT_ID0` | Source (a layer, the splitter, or the compositor for write-back) |
| `0x0D0` | `SC_CONTROL` | `[0]` `SCL` scaling enable; `[1]` `LS` line-split mode (software split, no hardware splitter upstream); `[4]` `AP` alpha processing (scale alpha channel too); `[8]` `IENH` image enhancement; `[16]` `RGBSM`; `[17]` `ASM` (not used by Linux) |
| `0x0DC` | `SC_COEFFTAB` | `[15:0]` horizontal filter table select; `[31:16]` vertical filter table select (05 §5.3.2) |
| `0x0E0` | `SC_IN_SIZE` | `HV_SIZE` |
| `0x0E4` | `SC_OUT_SIZE` | `HV_SIZE` (includes crop, 05 §5.4) |
| `0x0E8` | `SC_H_CROP` | `HV_CROP`: pixels dropped at the left and right of the output |
| `0x0EC` | `SC_V_CROP` | `HV_CROP`: top and bottom. Not written by Linux |
| `0x0F0` | `SC_H_INIT_PH` | Horizontal initial phase, 16.16 fixed point |
| `0x0F4` | `SC_H_DELTA_PH` | Horizontal phase step, 16.16 |
| `0x0F8` | `SC_V_INIT_PH` | Vertical initial phase, 16.16 |
| `0x0FC` | `SC_V_DELTA_PH` | Vertical phase step, 16.16 |
| `0x130` | `SC_ENH_LIMITS` | Enhancer limits. Not written by Linux |
| `0x134 + 4i`, i = 0–8 | `SC_ENH_COEFF<i>` | Enhancer coefficients. Not written by Linux |

## 2.10 Splitter (type 0x20)

| Offset | Name | Fields |
|---|---|---|
| `0x080` | `SP_INPUT_ID0` | Source |
| `0x0D0` | `SP_CONTROL` | `[0]` enable |
| `0x0D4` | `SP_SIZE` | `HV_SIZE` of the input |
| `0x0D8` | `SP_OVERLAP_SIZE` | `[12:0]` overlap in pixels added to each half |

Output port 0 is the left half and port 1 the right half.

## 2.11 Merger (type 0x22)

| Offset | Name | Fields |
|---|---|---|
| `0x080` | `MG_INPUT_ID0` | Left source |
| `0x084` | `MG_INPUT_ID1` | Right source |
| `0x0D0` | `MG_CONTROL` | `[0]` enable |
| `0x0D4` | `MG_SIZE` | `HV_SIZE` of the merged output |

## 2.12 DOU (type 0x03)

| Offset | Name | Fields |
|---|---|---|
| `0x0A0–0x0AC` | IRQ registers | `[8]` `UND` underrun; `[11]` `ERR`; `[13]` `PL0` programmable line 0 (vblank); `[14]` `PL1` programmable line 1 |
| `0x0B0` | `DOU_STATUS` | `[0]` `DRIFTTO` drift timeout (the register it pairs with is programmed only for dual-link); `[1]` `FRAMETO` frame timeout; `[2]` `TETO` tearing-effect timeout; `[8]` `CSCE` (probably a colour-conversion error, inferred); `[31]` `ACTIVE` |

## 2.13 IPS / image processor (type 0x30)

| Offset | Name | Fields |
|---|---|---|
| `0x080` / `0x084` | `IPS_INPUT_ID0/1` | Input 0 is the compositor. Input 1 is not used by Linux |
| `0x0C0` | `IPS_INFO` | `[10]` 4:2:0 chroma downsampling supported |
| `0x0D0` | `IPS_CONTROL` | §2.13.1 |
| `0x0D4` | `IPS_SIZE` | `HV_SIZE` |
| `0x0D8` | `IPS_DEPTH` | `[3:0]` output bits per component (8 or 10 used) |
| `0x130 + 4i`, i = 0–11 | `IPS_RGB_RGB_COEFF<i>` | Colour transform matrix (06 §6.2). Linux writes i = 0–8 |
| `0x170 + 4i`, i = 0–11 | `IPS_RGB_YUV_COEFF<i>` | RGB→YUV output matrix. Not written by Linux |

### 2.13.1 `IPS_CONTROL`

| Bit | Name | Meaning |
|---|---|---|
| 0 | `RGB` | Apply the RGB→RGB matrix (CTM) |
| 1 | `IT` | Apply an input-transform (degamma) table *(Linlon-D6, vendor)* |
| 4 | `FT` | Apply the forward transform (gamma) table |
| 8 | `YUV` | Convert the output to YCbCr |
| 9 | `CHD422` | Chroma downsample to 4:2:2 |
| 10 | `CHD420` | Chroma downsample to 4:2:0 (requires `CHD422` also set) |
| 11 | `LPF` | Chroma low-pass filter (not used) |
| 12 | `DITH` | Dithering (not used) |
| 16 | `CLAMP` | Output clamping (not used) |
| 17 | `SBS` | Side-by-side: input 1 carries the right half of the screen *(vendor, 06 §6.6)* |
| 18 | `IM` | Image merge: join inputs 0 and 1 into one output stream *(Linlon-D6, vendor)* |

- **[HW]** YCbCr output needs `IPS_RGB_YUV_COEFF` loaded (06 §6.3).
  Mainline Linux omits this. The vendor driver loads the BT.601
  limited-range matrix whenever `YUV` is set.

> OPEN QUESTION: How `IPS_CONTROL.IT` selects its degamma table (there is
> no table-select register in the IPS) is not visible in the vendor source.
> It may use a fixed table block.

## 2.14 Forward-transform coefficient table (type 0x32)

| Offset | Name | Fields |
|---|---|---|
| `0x080 + 4i`, i = 0–64 | `FT_COEFF<i>` | 65 gamma-curve points, 12-bit unsigned (06 §6.2.2) |

## 2.15 BS / back-end timing generator (type 0x31)

| Offset | Name | Fields |
|---|---|---|
| `0x0C0` | `BS_INFO` | Not decoded |
| `0x0D0` | `BS_CONTROL` | §2.15.1 |
| `0x0D4` | `BS_PROG_LINE` | Line number at which the DOU `PL0` interrupt fires |
| `0x0D8` | `BS_PREFETCH_LINE` | Lines before active video at which fetching starts |
| `0x0DC` | `BS_BG_COLOR` | Background colour. Not written by Linux |
| `0x0E0` | `BS_ACTIVESIZE` | `HV_SIZE`: active width and height |
| `0x0E4` | `BS_HINTERVALS` | `[11:0]` horizontal front porch; `[25:16]` horizontal back porch |
| `0x0E8` | `BS_VINTERVALS` | `[13:0]` vertical front porch; `[23:16]` vertical back porch |
| `0x0EC` | `BS_SYNC` | `[9:0]` hsync width; `[12]` hsync active-high; `[23:16]` vsync width; `[28]` vsync active-high |
| `0x100` | `BS_DRIFT_TO` | Dual-link drift timeout |
| `0x104` | `BS_FRAME_TO` | Frame timeout. Not written by Linux |
| `0x108` | `BS_TE_TO` | Tearing-effect timeout. Not written by Linux |
| `0x110`–`0x118` | `BS_T0/T1/T2_INTERVAL` | Not written by Linux |
| `0x120`/`0x124` | `BS_CRC0_LOW/HIGH` | Frame CRC, link 0 |
| `0x128`/`0x12C` | `BS_CRC1_LOW/HIGH` | Frame CRC, link 1 |
| `0x130` | `BS_USER` | Not used |

### 2.15.1 `BS_CONTROL`

| Bit | Name | Meaning |
|---|---|---|
| 0 | `EN` | Timing generator enable |
| 1 | `VM` | Video mode (continuous timing). Reset value has this set (`0x00000002`) |
| 2 | `BM` | Not used |
| 4 | `HMASK` | Not used |
| 5 | `VD` | Set **instead of** `VM` when the display path feeds only write-back, with no real output *(vendor)* |
| 8 | `TE` | Tearing-effect input enable (command-mode panels; not used) |
| 9 | `TS` | Not used |
| 12 | `TM` | Test pattern (vendor: debug option) |
| 16 | `DL` | Dual-link output |
| 17 | `SBS` | Side-by-side (not used) |
| 18 | `CRC` | Frame CRC generation enable (vendor: debug option) |
| 20 | `PM` | Protected mode (vendor: a CRTC property) |
| `[22:21]` | `PPC` | Pixels per clock: 0 = 1, 1 = 2 *(Linlon-D6, vendor)* |

## 2.16 Global coefficient blocks

| Block | Offset | Name | Notes |
|---|---|---|---|
| Global scaler coefficients (type 0x06 / 0x07) | `0x080` | `GLB_SC_COEFF_ADDR` | `[18]` R, `[17]` G, `[16]` B channel select. Low bits are the index |
| | `0x084` | `GLB_SC_COEFF_DATA` | `[31:16]` first coefficient, `[15:0]` second |
| Global LT coefficients (type 0x05) | `0x080` | `GLB_LT_COEFF_DATA` | 65 entries |

- **[HW]** A global scaler coefficient table holds up to 1024 entries.

- **[POLICY]** Mainline Linux never writes any global coefficient block.
  Scaling relies on the tables the hardware provides at reset.
- **[HW]** *(vendor)* On the Linlon-D6 each global LT block (type `0x05`)
  holds one 65-entry curve, written as 65 consecutive words starting at
  `0x080`. Tables are selected elsewhere by their `BLOCK_INFO[11:4]` value
  (type `0x05` → select `0x50 + instance`). The vendor driver uses table
  `0x50` for simple layers' degamma, `0x52` for gamma (forward transform)
  and the others for rich layers' degamma. It shares tables among layers
  with reference counting (one table can feed several layers only if they
  want the same curve).

> OPEN QUESTION: The global scaler tables are still never written, by
> either driver. The `SC_COEFFTAB` select values in 05 §5.3.2 follow the
> same "block type and instance" rule (type `0x06` → `0x60 + instance`),
> which strongly suggests they are preset tables in scaler coefficient
> blocks 2–6 and 10–14.

## 2.17 Linlon-D6 fixed block offsets *(vendor)*

The vendor driver uses these pipeline-0 block offsets directly during the
firmware handover (08 §8.5). They are consistent with discovery order on the
D6 and can be used as a sanity check, but a driver should still discover
blocks (03 §3.2).

| Block | Offset |
|---|---|
| LPU0 layer 0 | `0x0400` |
| LPU0 layer 1 | `0x0600` |
| LPU0 layer 2 | `0x0800` |
| LPU0 layer 3 | `0x0A00` |
| CU0 | `0x0E00` |
| CU0 splitter | `0x1000` |
| CU0 scaler 0 | `0x1200` |
| CU0 scaler 1 | `0x1400` |
| CU0 merger | `0x1600` |
| DOU0 IPS | `0x1A00` |
| DOU0 BS | `0x1E00` |
