# MDSS v8.1 Wrapper, UBWC 4.0, VBIF, and DPU1 Pipeline Specification (`SC8280XP`)

## 1. MDSS Wrapper (`MDSS_BASE + 0x00000`)

**[HW]** The MDSS top-level wrapper resides at offset `+0x00000` (`0x0ae00000` for `MDSS0`, `0x22000000` for `MDSS1`, size `0x1000`). It provides hardware version identification, level-1 interrupt aggregation, and Universal Bandwidth Compression (`UBWC`) decoder static parameters.

| Offset | Register Name | Access | Description |
|---|---|---|---|
| `0x00000` | `HW_VERSION` | RO | Hardware revision: bits `[31:28]` `MAJOR` (`8`), bits `[27:16]` `MINOR` (`1`), bits `[15:0]` `STEP` (expected `0x80010000` for SC8280XP MDSS v8.1). |
| `0x00010` | `HW_INTR_STATUS` | RO | Level-1 interrupt status bitmap: bit `0` `MDP`, bit `4` `DSI_0`, bit `5` `DSI_1`, bit `12` `DP_0`, bit `13` `DP_1`, bit `14` `DP_2`, bit `15` `DP_3`. |
| `0x00058` | `UBWC_DEC_HW_VERSION` | RO | UBWC decoder hardware version (`0x40000300` for UBWC 4.0 decoder). |
| `0x00144` | `UBWC_STATIC` | RW | UBWC static memory bank configuration: bits `[2:0]` `UBWC_SWIZZLE`, bit `3` `UBWC_BANK_SPREAD`, bits `[6:4]` `HIGHEST_BANK_BIT` (`HBB - 13`), bit `8` `UBWC_MIN_ACC_LEN` (`0` = 64B, `1` = 32B), bit `12` `MACROTILE_MODE` (`0` = 4-channel, `1` = 8-channel). |
| `0x00150` | `UBWC_CTRL_2` | RW | UBWC control 2 register (`2` for UBWC 4.0, `1` for UBWC 3.0). |
| `0x00154` | `UBWC_PREDICTION_MODE` | RW | UBWC prediction mode (`1` for UBWC 4.0 / 3.0, `0` for UBWC 2.0). |

### 1.1 UBWC 4.0 Initialization Sequence on SC8280XP

**[HW]** On SC8280XP (`MDSS v8.1`, UBWC encoder version `0x40000000`, decoder version `0x40000300`), the UBWC parameters are:
- `ubwc_swizzle` = `6` (`0b110`: Level 2 & 3 bank swizzling enabled, Level 1 disabled).
- `bank_spread` = `1` (enabled).
- `macrotile_mode` = `1` (8-channel macrotiling).
- `highest_bank_bit` (`HBB`): `16` on standard 8-channel LPDDR5 configurations (`HIGHEST_BANK_BIT` field = `16 - 13 = 3`, giving `UBWC_STATIC = 0x103e`), or `15` when configured by firmware/SMEM (`HIGHEST_BANK_BIT` field = `15 - 13 = 2`, giving `UBWC_STATIC = 0x102e` or `0x001e` when UEFI leaves `HBB = 14` / `MACROTILE_MODE = 0`).

**[HW]** To program the MDSS UBWC decoder at initialization:
1. Write `UBWC_STATIC` (`0x00144`) = `(macrotile_mode << 12) | ((highest_bank_bit - 13) << 4) | (bank_spread << 3) | (ubwc_swizzle & 0x7)`.
2. Write `UBWC_CTRL_2` (`0x00150`) = `0x00000002`.
3. Write `UBWC_PREDICTION_MODE` (`0x00154`) = `0x00000001`.

**[POLICY]** On boards where UEFI has already initialized `MDSS0` and memory controllers (such as the Radxa Dragon Q8B where `UBWC_STATIC` reads `0x0000001e`), preserving the firmware's `UBWC_STATIC` value or querying SMEM item `490` (`DDR_INFO`) avoids bank-bit mismatches between the GPU and display scanout.

---

## 2. Virtual Bus Interface (`VBIF_RT`, `MDSS_BASE + 0xb0000`)

**[HW]** `VBIF_RT` resides at `MDSS_BASE + 0xb0000` (`0x0aeb0000` on `MDSS0`, `0x220b0000` on `MDSS1`, size `0x1000`). It arbitrates AXI read requests from DPU source pipes (`XIN` ports `0..13`) to the memory interconnect.

| Offset | Register Name | Access | Description |
|---|---|---|---|
| `0x0000` | `VBIF_VERSION` | RO | VBIF hardware revision |
| `0x0004` | `VBIF_CLK_FORCE_CTRL0` | RW | Clock gating override 0 (`bit 0` = global force-on, `bits [31:16]` = per-XIN `0..15` force-on) |
| `0x0008` | `VBIF_CLK_FORCE_CTRL1` | RW | Clock gating override 1 (`bits [15:0]` = per-XIN `16..31` force-on) |
| `0x00d8` | `VBIF_QOS_RP_REMAP_000` | RW | Realtime QoS priority remapping base |
| `0x00fc` | `VBIF_FP_QOS_REMAP_000` | RW | Forced priority QoS remapping base |
| `0x010c` | `VBIF_OUT_RD_LIM_CONF0` | RW | OT read limit for clients `0..3` (8 bits per XIN) |
| `0x0110` | `VBIF_OUT_RD_LIM_CONF1` | RW | OT read limit for clients `4..7` |
| `0x0114` | `VBIF_OUT_RD_LIM_CONF2` | RW | OT read limit for clients `8..11` |
| `0x0118` | `VBIF_OUT_RD_LIM_CONF3` | RW | OT read limit for clients `12..13` |
| `0x0120` | `VBIF_OUT_WR_LIM_CONF0` | RW | OT write limit for clients `0..3` |
| `0x012c` | `VBIF_OUT_AXI_AOOO_EN` | RW | AXI out-of-order enable per XIN |
| `0x0130` | `VBIF_OUT_AXI_AOOO` | RW | AXI out-of-order control |
| `0x0158` | `VBIF_IN_RD_LIM_CONF0` | RW | Input read limit `0..3` |
| `0x0160` | `VBIF_OUT_AXI_AMEMTYPE_CONF0` | RW | AXI `AMEMTYPE` for XINs `0..7` (4 bits per XIN, `XIN_i` at bits `[4*i+2 : 4*i]`) |
| `0x0164` | `VBIF_OUT_AXI_AMEMTYPE_CONF1` | RW | AXI `AMEMTYPE` for XINs `8..13` (4 bits per XIN, `XIN_i` at bits `[4*(i-8)+2 : 4*(i-8)]`) |
| `0x0170` | `VBIF_XIN_HALT_CTRL0` | RW | XIN halt request (`bits [15:0]`) and halt ack status (`bits [31:16]`) |
| `0x0550` | `VBIF_QOS_REMAP_RT_BASE` | RW | 8 registers (`0x550 + 4*lvl`, `lvl = 0..7`) for RT QoS remapping on DPU 5.x–8.x |
| `0x0590` | `VBIF_QOS_REMAP_NRT_BASE` | RW | 8 registers (`0x590 + 4*lvl`, `lvl = 0..7`) for NRT QoS remapping on DPU 5.x–8.x |

### 2.1 VBIF Initialization Sequence

**[HW]** On SC8280XP, XIN ports `0..13` must be configured as cacheable/non-coherent AXI transactions (`AMEMTYPE = 3`) with an 8-level QoS remap table:
1. For `xin_id` in `0..7`, set bits `[4*xin_id + 2 : 4*xin_id]` of `VBIF_OUT_AXI_AMEMTYPE_CONF0` (`0x0160`) to `0x3` (resulting value `0x33333333`).
2. For `xin_id` in `8..13`, set bits `[4*(xin_id - 8) + 2 : 4*(xin_id - 8)]` of `VBIF_OUT_AXI_AMEMTYPE_CONF1` (`0x0164`) to `0x3` (resulting value `0x00333333`).
3. Program the 8-entry QoS remap table (`level 0..7` -> values `{3, 3, 4, 4, 5, 5, 6, 6}` for RT and `{3, 3, 3, 3, 3, 3, 3, 3}` for NRT): for each `lvl` in `0..7` and each `xin_id` (`0..6`), write `(val << (xin_id * 4))` into `0x0550 + 4*lvl` (RT) and `0x0590 + 4*lvl` (NRT).

---

## 3. SC8280XP DPU 8.0/8.1 Hardware Catalog (`MDSS_BASE + 0x01000`)

**[HW]** All DPU sub-block offsets below are relative to `DPU_BASE` (`MDSS_BASE + 0x01000` = `0x0ae01000` on `MDSS0`, `0x22001000` on `MDSS1`).

**[HW]** Global DPU capabilities on SC8280XP:
- Maximum Layer Mixer width: `2560` pixels (`max_mixer_width = 2560`), maximum blend stages per mixer: `11` (`max_mixer_blendstages = 11`).
- Maximum VIG SSPP input width: `2560` pixels (up to `5120` with decimation/split), maximum QSEED3lite upscaling factor: `20x`, downscaling factor: `4x`.
- Maximum DPU core clock (`mdp_clk`): `650 MHz`.
- SmartDMA v2.5 enabled (`Multi-Rectangle` `REC0`/`REC1` per SSPP), active CTL configuration enabled (`CTL_FETCH_PIPE_ACTIVE`, `CTL_LAYER_ACTIVE`, `CTL_INTF_ACTIVE`), 3D merge reset enabled, widebus (`2 pixels/clock`) enabled on all DP/eDP interfaces.

### 3.1 DPU Sub-Block Address Map

| Sub-Block Type | Instance | Offset from `DPU_BASE` | Size | Key Pairings / Sub-Block Offsets |
|---|---|---|---|---|
| `MDP_TOP` | `top` | `0x00000` | `0x00494` | Split display, clock force controls, danger/safe status |
| `CTL` | `ctl_0` | `0x015000` | `0x00208` | Active CTL (`HAS_ACTIVE_CTL`), 6 DSPP flush registers |
| `CTL` | `ctl_1` | `0x016000` | `0x00208` | Active CTL |
| `CTL` | `ctl_2` | `0x017000` | `0x00208` | Active CTL (used by UEFI on Dragon Q8B for HDMI) |
| `CTL` | `ctl_3` | `0x018000` | `0x00208` | Active CTL |
| `CTL` | `ctl_4` | `0x019000` | `0x00208` | Active CTL |
| `CTL` | `ctl_5` | `0x01a000` | `0x00208` | Active CTL |
| `SSPP` (VIG) | `sspp_0` (`VIG0`) | `0x004000` | `0x00334` | `XIN_0`, `clk_ctrl = VIG0`, QSEED3lite `+0xa00`, CSC-10bit `+0x1a00` |
| `SSPP` (VIG) | `sspp_1` (`VIG1`) | `0x006000` | `0x00334` | `XIN_4`, `clk_ctrl = VIG1`, QSEED3lite `+0xa00`, CSC-10bit `+0x1a00` |
| `SSPP` (VIG) | `sspp_2` (`VIG2`) | `0x008000` | `0x00334` | `XIN_8`, `clk_ctrl = VIG2`, QSEED3lite `+0xa00`, CSC-10bit `+0x1a00` (UEFI pipe) |
| `SSPP` (VIG) | `sspp_3` (`VIG3`) | `0x00a000` | `0x00334` | `XIN_12`, `clk_ctrl = VIG3`, QSEED3lite `+0xa00`, CSC-10bit `+0x1a00` |
| `SSPP` (DMA) | `sspp_8` (`DMA0`) | `0x024000` | `0x00334` | `XIN_1`, `clk_ctrl = DMA0`, RGB/YUV unpacked & UBWC, no scaler/CSC |
| `SSPP` (DMA) | `sspp_9` (`DMA1`) | `0x026000` | `0x00334` | `XIN_5`, `clk_ctrl = DMA1`, RGB/YUV unpacked & UBWC, no scaler/CSC |
| `SSPP` (DMA) | `sspp_10` (`DMA2`) | `0x028000` | `0x00334` | `XIN_9`, `clk_ctrl = DMA2`, Cursor-capable |
| `SSPP` (DMA) | `sspp_11` (`DMA3`) | `0x02a000` | `0x00334` | `XIN_13`, `clk_ctrl = DMA3`, Cursor-capable |
| `LM` | `lm_0` | `0x044000` | `0x00400` | Paired with `lm_1`, `pingpong_0`, `dspp_0` |
| `LM` | `lm_1` | `0x045000` | `0x00400` | Paired with `lm_0`, `pingpong_1`, `dspp_1` |
| `LM` | `lm_2` | `0x046000` | `0x00400` | Paired with `lm_3`, `pingpong_2`, `dspp_2` (UEFI mixer) |
| `LM` | `lm_3` | `0x047000` | `0x00400` | Paired with `lm_2`, `pingpong_3`, `dspp_3` |
| `LM` | `lm_4` | `0x048000` | `0x00400` | Paired with `lm_5`, `pingpong_4`, no DSPP |
| `LM` | `lm_5` | `0x049000` | `0x00400` | Paired with `lm_4`, `pingpong_5`, no DSPP |
| `DSPP` | `dspp_0` | `0x054000` | `0x01800` | PCC (`+0x1700`, len `0x90`, v4.0) |
| `DSPP` | `dspp_1` | `0x056000` | `0x01800` | PCC (`+0x1700`, len `0x90`, v4.0) |
| `DSPP` | `dspp_2` | `0x058000` | `0x01800` | PCC (`+0x1700`, len `0x90`, v4.0) |
| `DSPP` | `dspp_3` | `0x05a000` | `0x01800` | PCC (`+0x1700`, len `0x90`, v4.0) |
| `MERGE_3D` | `merge_3d_0` | `0x04e000` | `0x00008` | Merges `pingpong_0` and `pingpong_1` |
| `MERGE_3D` | `merge_3d_1` | `0x04f000` | `0x00008` | Merges `pingpong_2` and `pingpong_3` |
| `MERGE_3D` | `merge_3d_2` | `0x050000` | `0x00008` | Merges `pingpong_4` and `pingpong_5` |
| `PINGPONG` | `pingpong_0` | `0x069000` | `0x000d4` | `merge_3d_0`, Dither v2.0 at `+0x0e0` (len `0x20`) |
| `PINGPONG` | `pingpong_1` | `0x06a000` | `0x000d4` | `merge_3d_0`, Dither v2.0 at `+0x0e0` |
| `PINGPONG` | `pingpong_2` | `0x06b000` | `0x000d4` | `merge_3d_1`, Dither v2.0 at `+0x0e0` (UEFI pingpong) |
| `PINGPONG` | `pingpong_3` | `0x06c000` | `0x000d4` | `merge_3d_1`, Dither v2.0 at `+0x0e0` |
| `PINGPONG` | `pingpong_4` | `0x06d000` | `0x000d4` | `merge_3d_2`, Dither v2.0 at `+0x0e0` |
| `PINGPONG` | `pingpong_5` | `0x06e000` | `0x000d4` | `merge_3d_2`, Dither v2.0 at `+0x0e0` |
| `DSC` (v1.2) | `dsc_0` | `0x080000` | `0x00004` | DCE0 slice 0: `enc` `+0x100` (len `0x9c`), `ctl` `+0xf00` (len `0x10`), native 4:2:2/4:2:0 |
| `DSC` (v1.2) | `dsc_1` | `0x080000` | `0x00004` | DCE0 slice 1: `enc` `+0x200` (len `0x9c`), `ctl` `+0xf80` (len `0x10`) |
| `DSC` (v1.2) | `dsc_2` | `0x081000` | `0x00004` | DCE1 slice 0: `enc` `+0x100`, `ctl` `+0xf00`, native 4:2:2/4:2:0 |
| `DSC` (v1.2) | `dsc_3` | `0x081000` | `0x00004` | DCE1 slice 1: `enc` `+0x200`, `ctl` `+0xf80` |
| `DSC` (v1.2) | `dsc_4` | `0x082000` | `0x00004` | DCE2 slice 0: `enc` `+0x100`, `ctl` `+0xf00`, native 4:2:2/4:2:0 |
| `DSC` (v1.2) | `dsc_5` | `0x082000` | `0x00004` | DCE2 slice 1: `enc` `+0x200`, `ctl` `+0xf80` |
| `CDM` | `cdm_0` | `0x079200` | `0x00228` | Chroma Down-Module (`4:4:4` RGB/YUV to `4:2:2`/`4:2:0` YUV) |
| `INTF` | `intf_0` | `0x034000` | `0x002c0` | `DP_0` (`controller_id = 0`, MST stream 0) |
| `INTF` | `intf_1` | `0x035000` | `0x002c0` | `DSI_0` (`controller_id = 0`) |
| `INTF` | `intf_2` | `0x036000` | `0x002c0` | `DSI_1` (`controller_id = 1`) |
| `INTF` | `intf_3` | `0x037000` | `0x002c0` | `DP_0` (`controller_id = 0`, MST stream 1) |
| `INTF` | `intf_4` | `0x038000` | `0x002c0` | `DP_1` (`controller_id = 1`, MST stream 0) |
| `INTF` | `intf_5` | `0x039000` | `0x002c0` | `DP_3` / `eDP` (`controller_id = 3`, stream 0) |
| `INTF` | `intf_6` | `0x03a000` | `0x002c0` | `DP_2` / `eDP` (`controller_id = 2`, stream 0; Dragon Q8B HDMI) |
| `INTF` | `intf_7` | `0x03b000` | `0x002c0` | `DP_2` (`controller_id = 2`, MST stream 1) |
| `INTF` | `intf_8` | `0x03c000` | `0x002c0` | `DP_1` (`controller_id = 1`, MST stream 1) |

---

## 4. `MDP_TOP` Registers (`DPU_BASE + 0x00000`)

**[HW]** `MDP_TOP` controls split-display synchronization, internal pipe clock gating, and danger/safe QoS status monitoring:

| Offset | Register Name | Access | Description |
|---|---|---|---|
| `0x02ec` | `SPLIT_DISPLAY_EN` | RW | Bit `0`: enable split-display synchronization |
| `0x02f0` | `SPLIT_DISPLAY_UPPER_PIPE_CTRL` | RW | Bit `0` `SMART_PANEL_FREE_RUN`, bit `1` `INTF_MASTER` (`0` = INTF_1, `1` = INTF_2), bit `4` `TE_LINE_INTER_WATERMARK_MASK` |
| `0x02f4` | `DSP_CLK_CTRL0` | RW | Clock force control 0: `VIG0` (`ctrl` bit `0`, `status` bit `1`), `DMA0` (`8`/`9`), `CURSOR0` (`16`/`17`), `CURSOR1` (`24`/`25`) |
| `0x02f8` | `DSP_CLK_CTRL1` | RW | Clock force control 1: `VIG1` (`0`/`1`), `DMA1` (`8`/`9`) |
| `0x0300` | `DSP_CLK_CTRL2` | RW | Clock force control 2: `VIG2` (`0`/`1`), `DMA2` (`8`/`9`) |
| `0x0304` | `SPLIT_DISPLAY_LOWER_PIPE_CTRL` | RW | Lower pipe split-display control |
| `0x0308` | `DSP_CLK_CTRL3` | RW | Clock force control 3: `VIG3` (`0`/`1`), `DMA3` (`8`/`9`) |
| `0x0360` | `DANGER_STATUS` | RO | Per-SSPP danger watermark status bitmap |
| `0x0364` | `SAFE_STATUS` | RO | Per-SSPP safe watermark status bitmap (`1` = safe) |
| `0x03ac` | `DSP_CLK_CTRL4` | RW | Clock force control 4: `REG_DMA` (`16`/`17`) |
| `0x0454` | `HDMI_DP_CORE_SELECT` | RW | Legacy HDMI/DP selector (`0` = DP) |
| `0x0460` | `DP_PHY_INTF_SEL` | RW | Maps DP PHY `0..2` (3 bits per PHY: PHY0 `[2:0]`, PHY1 `[5:3]`, PHY2 `[8:6]`) to `intf_idx + 1` |

**[HW]** To enable or disable dynamic clock gating on a source pipe (`VIG0..3`, `DMA0..3`), clear the corresponding `ctrl` bit (`0` or `8`) in `DSP_CLK_CTRL0..3` to allow hardware auto-gating, or set it to `1` to force the pipe clock on.

---

## 5. Control Path (`CTL_0..5`, `DPU_BASE + 0x15000..0x1a000`)

**[HW]** Each `CTL` block binds Layer Mixers (`LM`), source pipes (`SSPP`), post-processing blocks (`MERGE_3D`, `DSC`, `PINGPONG`, `CDM`), and Timing Interfaces (`INTF`) into an atomic display pipeline, and latches register updates on the next vertical blanking edge when `CTL_FLUSH` is written.

### 5.1 CTL Register Map

| Offset | Register Name | Access | Description |
|---|---|---|---|
| `0x000 + 4*lm` | `CTL_LAYER(lm)` (`lm = 0..4`) | RW | Stage blend assignment for `LM_0..4` (`CTL_LAYER_LM5` is at `0x024`) |
| `0x014` | `CTL_TOP` | RW | Mode select (`0` = video mode, `2` = command mode) |
| `0x018` | `CTL_FLUSH` | RW | Master flush trigger bitmap (clears to `0` at vsync when latched) |
| `0x01c` | `CTL_START` | WO | Write `1` to trigger command-mode / writeback frame start |
| `0x020` | `CTL_PACK_3D` | RW | 3D packing control |
| `0x024` | `CTL_LAYER_LM5` | RW | Stage blend assignment for `LM_5` |
| `0x030` | `CTL_SW_RESET` | RW | Write `1` to software-reset the CTL path; poll until bit `0` reads `0` |
| `0x040 + 4*lm` | `CTL_LAYER_EXT(lm)` (`lm = 0..5`) | RW | Extension 1 stage blend bits (bit 3 of stage index, plus `REC1` for `VIG0..3` and `DMA0..1`) |
| `0x070 + 4*lm` | `CTL_LAYER_EXT2(lm)` (`lm = 0..5`) | RW | Extension 2 stage blend bits (`DMA2..3` `REC0` and `REC1`) |
| `0x0a0 + 4*lm` | `CTL_LAYER_EXT3(lm)` (`lm = 0..5`) | RW | Extension 3 stage blend bits (`VIG4..5`, `DMA4..5` on larger SoCs) |
| `0x0b8 + 4*lm` | `CTL_LAYER_EXT4(lm)` (`lm = 0..5`) | RW | Extension 4 stage blend bits (bit 4 of stage index when >7 stages) |
| `0x0d0` | `CTL_PREPARE` | RW | Command-mode frame prepare |
| `0x0e4` | `CTL_MERGE_3D_ACTIVE` | RW | Active `MERGE_3D` bitmap (`bit i` = `merge_3d_i`) |
| `0x0e8` | `CTL_DSC_ACTIVE` | RW | Active `DSC` bitmap (`bit i` = `dsc_i`, `i = 0..5`) |
| `0x0ec` | `CTL_WB_ACTIVE` | RW | Active Writeback bitmap (`bit 2` = `WB_2`) |
| `0x0f0` | `CTL_CWB_ACTIVE` | RW | Active Concurrent Writeback bitmap |
| `0x0f4` | `CTL_INTF_ACTIVE` | RW | Active `INTF` bitmap (`bit i` = `intf_i`, `i = 0..8`) |
| `0x0f8` | `CTL_CDM_ACTIVE` | RW | Active `CDM` bitmap (`bit 0` = `cdm_0`) |
| `0x0fc` | `CTL_FETCH_PIPE_ACTIVE` | RW | Active SSPP fetch pipe bitmap (see Section 5.3) |
| `0x100` | `CTL_MERGE_3D_FLUSH` | RW | Sub-flush bitmap for `MERGE_3D_0..2` (`bits [2:0]`) |
| `0x104` | `CTL_DSC_FLUSH` | RW | Sub-flush bitmap for `DSC_0..5` (`bits [5:0]`) |
| `0x108` | `CTL_WB_FLUSH` | RW | Sub-flush bitmap for `WB_2` (`bit 2`) |
| `0x10c` | `CTL_CWB_FLUSH` | RW | Sub-flush bitmap for `CWB` |
| `0x110` | `CTL_INTF_FLUSH` | RW | Sub-flush bitmap for `INTF_0..8` (`bit i` = `intf_i`, e.g., `bit 6` for `INTF_6`) |
| `0x114` | `CTL_CDM_FLUSH` | RW | Sub-flush bitmap for `CDM_0` (`bit 0`) |
| `0x128` | `CTL_PERIPH_FLUSH` | RW | Sub-flush bitmap for DP/DSI peripheral timing/MSA (`bit i` = `intf_i`) |
| `0x12c` | `CTL_PIPE_ACTIVE` | RO | Hardware-reported active pipe status |
| `0x130` | `CTL_LAYER_ACTIVE` | RW | Active Layer Mixer bitmap (`bit i` = `lm_i`, `i = 0..5`) |
| `0x134` | `CTL_INTF_MASTER` | RW | Master interface ID for dual-INTF split (`bit i` = `intf_i`) |
| `0x13c + 4*d` | `CTL_DSPP_n_FLUSH(d)` (`d = 0..3`) | RW | Sub-flush bitmap for `DSPP_d` (`bit 2` = `DSPP_MAIN`, `bit 4` = `PCC`) |

### 5.2 Blend Stage Assignment in `CTL_LAYER*`

**[HW]** Each Layer Mixer `lm` (`0..5`) supports up to 11 blend stages:
- `STAGE_BASE` = `1` (background/bottom-most hardware plane, or border color if `BORDER_OUT` bit `24` of `CTL_LAYER(lm)` is set).
- `STAGE_0`..`STAGE_10` = values `2`..`12` (`stage_value = stage_index + 2`).
- Value `0` (`STAGE_UNUSED`) disconnects the SSPP from mixer `lm`.

**[HW]** Each SSPP (`REC0` and SmartDMA `REC1`) has a 3-bit low field `[2:0]` in `CTL_LAYER(lm)` or `CTL_LAYER_EXT*(lm)`, a bit-3 field in `CTL_LAYER_EXT(lm)`, and a bit-4 field in `CTL_LAYER_EXT4(lm)`:

| Pipe / Multi-Rect | Low 3-Bit Register & Shift (`stage_val & 0x7`) | Bit-3 Shift (`CTL_LAYER_EXT`, `(stage_val >> 3) & 1`) | Bit-4 Shift (`CTL_LAYER_EXT4`, `(stage_val >> 4) & 1`) |
|---|---|---|---|
| `VIG0` (`REC0`) | `CTL_LAYER(lm)` bits `[2:0]` | `CTL_LAYER_EXT(lm)` bit `0` | `CTL_LAYER_EXT4(lm)` bit `0` |
| `VIG1` (`REC0`) | `CTL_LAYER(lm)` bits `[5:3]` | `CTL_LAYER_EXT(lm)` bit `2` | `CTL_LAYER_EXT4(lm)` bit `1` |
| `VIG2` (`REC0`) | `CTL_LAYER(lm)` bits `[8:6]` | `CTL_LAYER_EXT(lm)` bit `4` | `CTL_LAYER_EXT4(lm)` bit `2` |
| `VIG3` (`REC0`) | `CTL_LAYER(lm)` bits `[28:26]` | `CTL_LAYER_EXT(lm)` bit `6` | `CTL_LAYER_EXT4(lm)` bit `3` |
| `DMA0` (`REC0`) | `CTL_LAYER(lm)` bits `[20:18]` | `CTL_LAYER_EXT(lm)` bit `8` | `CTL_LAYER_EXT4(lm)` bit `4` |
| `DMA1` (`REC0`) | `CTL_LAYER(lm)` bits `[23:21]` | `CTL_LAYER_EXT(lm)` bit `10` | `CTL_LAYER_EXT4(lm)` bit `5` |
| `DMA2` (`REC0`) | `CTL_LAYER_EXT2(lm)` bits `[2:0]` | `CTL_LAYER_EXT(lm)` bit `25` | `CTL_LAYER_EXT4(lm)` bit `6` |
| `DMA3` (`REC0`) | `CTL_LAYER_EXT2(lm)` bits `[5:3]` | `CTL_LAYER_EXT(lm)` bit `27` | `CTL_LAYER_EXT4(lm)` bit `7` |
| `VIG0` (`REC1`) | `CTL_LAYER_EXT(lm)` bits `[19:17]` | `CTL_LAYER_EXT(lm)` bit `16` | `CTL_LAYER_EXT4(lm)` bit `8` |
| `VIG1` (`REC1`) | `CTL_LAYER_EXT(lm)` bits `[23:21]` | `CTL_LAYER_EXT(lm)` bit `20` | `CTL_LAYER_EXT4(lm)` bit `9` |
| `VIG2` (`REC1`) | `CTL_LAYER_EXT(lm)` bits `[15:13]` | `CTL_LAYER_EXT(lm)` bit `12` | `CTL_LAYER_EXT4(lm)` bit `10` |
| `VIG3` (`REC1`) | `CTL_LAYER_EXT(lm)` bits `[31:29]` | `CTL_LAYER_EXT(lm)` bit `28` | `CTL_LAYER_EXT4(lm)` bit `11` |
| `DMA0` (`REC1`) | `CTL_LAYER_EXT2(lm)` bits `[10:8]` | `CTL_LAYER_EXT2(lm)` bit `11` | `CTL_LAYER_EXT4(lm)` bit `12` |
| `DMA1` (`REC1`) | `CTL_LAYER_EXT2(lm)` bits `[14:12]` | `CTL_LAYER_EXT2(lm)` bit `15` | `CTL_LAYER_EXT4(lm)` bit `13` |
| `DMA2` (`REC1`) | `CTL_LAYER_EXT2(lm)` bits `[18:16]` | `CTL_LAYER_EXT2(lm)` bit `19` | `CTL_LAYER_EXT4(lm)` bit `14` |
| `DMA3` (`REC1`) | `CTL_LAYER_EXT2(lm)` bits `[22:20]` | `CTL_LAYER_EXT2(lm)` bit `23` | `CTL_LAYER_EXT4(lm)` bit `15` |
| `BORDER_OUT` | `CTL_LAYER(lm)` bit `24` (`1` = enable mixer border color output at `STAGE_BASE`) | — | — |

### 5.3 `CTL_FETCH_PIPE_ACTIVE` and `CTL_FLUSH` Bit Encodings

**[HW]** `CTL_FETCH_PIPE_ACTIVE` (`0x0fc`) must have the bit set for every SSPP actively fetching pixels on this CTL:

| SSPP | `CTL_FETCH_PIPE_ACTIVE` Bit | `CTL_FLUSH` (`0x018`) Bit |
|---|---|---|
| `VIG0` | Bit `16` (`0x00010000`) | Bit `0` (`0x00000001`) |
| `VIG1` | Bit `17` (`0x00020000`) | Bit `1` (`0x00000002`) |
| `VIG2` | Bit `18` (`0x00040000`) | Bit `2` (`0x00000004`) |
| `VIG3` | Bit `19` (`0x00080000`) | Bit `18` (`0x00040000`) |
| `DMA0` | Bit `0` (`0x00000001`) | Bit `11` (`0x00000800`) |
| `DMA1` | Bit `1` (`0x00000002`) | Bit `12` (`0x00001000`) |
| `DMA2` | Bit `2` (`0x00000004`) | Bit `24` (`0x01000000`) |
| `DMA3` | Bit `3` (`0x00000008`) | Bit `25` (`0x02000000`) |

**[HW]** Additional `CTL_FLUSH` (`0x018`) bits required when updating pipeline blocks:

| Block | `CTL_FLUSH` (`0x018`) Bit | Notes |
|---|---|---|
| `LM_0`..`LM_5` | Bits `6`, `7`, `8`, `9`, `10`, `20` | Flush Layer Mixer `0..5` |
| `CTL` | Bit `17` (`0x00020000`) | Flush `CTL_LAYER*` and `CTL_*_ACTIVE` registers |
| `CDM_0` | Bit `26` (`0x04000000`) | Also write `bit 0` to `CTL_CDM_FLUSH` (`0x114`) |
| `WB` | Bit `16` (`0x00010000`) | Also write `CTL_WB_FLUSH` (`0x108`) |
| `MERGE_3D` | Bit `23` (`0x00800000`) | Also write `CTL_MERGE_3D_FLUSH` (`0x100`) |
| `CWB` | Bit `28` (`0x10000000`) | Also write `CTL_CWB_FLUSH` (`0x10c`) |
| `DSPP_0`..`DSPP_3` | Bit `29` (`0x20000000`) | Also write `CTL_DSPP_n_FLUSH(d)` (`0x13c + 4*d`) |
| `PERIPH` | Bit `30` (`0x40000000`) | Also write `CTL_PERIPH_FLUSH` (`0x128`) |
| `INTF` | Bit `31` (`0x80000000`) | Also write `CTL_INTF_FLUSH` (`0x110`) |
| `DSC` | Bit `22` (`0x00400000`) | Also write `CTL_DSC_FLUSH` (`0x104`) |

---

## 6. Source Surface Processing Pipe (`SSPP`, `VIG0..3` & `DMA0..3`)

**[HW]** Each SSPP fetches a 1-, 2-, 3-, or 4-plane surface from memory via `VBIF_RT`, unpacks pixels, optionally decompresses UBWC 4.0, and (on `VIG0..3`) performs 10-bit YUV-to-RGB CSC and QSEED3lite scaling.

### 6.1 SSPP Register Map (Offsets from `SSPP_BASE`)

| Offset | Register Name | Access | Description |
|---|---|---|---|
| `0x000` | `SSPP_SRC_SIZE` | RW | `REC0` source crop size: `(height << 16) \| width` |
| `0x004` | `SSPP_SRC_IMG_SIZE` | RW | Full image size: `(img_height << 16) \| img_width` |
| `0x008` | `SSPP_SRC_XY` | RW | `REC0` source crop offset: `(src_y << 16) \| src_x` |
| `0x00c` | `SSPP_OUT_SIZE` | RW | `REC0` destination size on mixer: `(dst_h << 16) \| dst_w` |
| `0x010` | `SSPP_OUT_XY` | RW | `REC0` destination offset on mixer: `(dst_y << 16) \| dst_x` |
| `0x014` | `SSPP_SRC0_ADDR` | RW | Plane 0 (`REC0` RGB/Y or UBWC color data) 32-bit IOVA/PA |
| `0x018` | `SSPP_SRC1_ADDR` | RW | Plane 1 (`REC0` UV or SmartDMA `REC1` RGB) 32-bit IOVA/PA |
| `0x01c` | `SSPP_SRC2_ADDR` | RW | Plane 2 (`REC0` V or UBWC meta-0) 32-bit IOVA/PA |
| `0x020` | `SSPP_SRC3_ADDR` | RW | Plane 3 (`REC0` alpha or UBWC meta-1) 32-bit IOVA/PA |
| `0x024` | `SSPP_SRC_YSTRIDE0` | RW | Plane 0 stride in bits `[15:0]`, Plane 1 (`REC0` UV or `REC1`) stride in bits `[31:16]` |
| `0x028` | `SSPP_SRC_YSTRIDE1` | RW | Plane 2 stride in bits `[15:0]`, Plane 3 stride in bits `[31:16]` |
| `0x030` | `SSPP_SRC_FORMAT` | RW | Pixel format, chroma sample, unpack count, tile/UBWC (see Section 6.2) |
| `0x034` | `SSPP_SRC_UNPACK_PATTERN` | RW | Component byte order for `REC0` (`C0` bits `[1:0]`, `C1` `[9:8]`, `C2` `[17:16]`, `C3` `[25:24]`) |
| `0x038` | `SSPP_SRC_OP_MODE` | RW | Flip (`LR` bit `0`, `UD` bit `1`), UBWC (`BWC_EN` bit `31`, `LOSSLESS` bits `[30:29]=0`), `PE_OVERRIDE` bit `31` (`0x80000000`) |
| `0x03c` | `SSPP_SRC_CONSTANT_COLOR` | RW | Constant fill color (`REC0`) when solid fill (`SRC_FORMAT` bit `22`) is set |
| `0x048` | `SSPP_FETCH_CONFIG` | RW | AXI burst configuration: reset default `0x00000087` (`MAL_32B`, 16-beat burst, 8 lines) |
| `0x050` | `SSPP_DECIMATION_CONFIG` | RW | Horizontal/vertical decimation factors |
| `0x060` | `SSPP_DANGER_LUT` | RW | 8-slot QoS danger LUT (linear RGB: `0x0000000f`, macrotile: `0x0000ff00`) |
| `0x064` | `SSPP_SAFE_LUT` | RW | 8-slot QoS safe LUT (linear RGB: `0x0000ffff`, macrotile: `0x0000ffff`) |
| `0x068` | `SSPP_CREQ_LUT_0` | RW | Creq LUT low (`0x33332222` linear, `0x11111111` macrotile) |
| `0x06c` | `SSPP_CREQ_LUT_1` | RW | Creq LUT high (`0x00007766` linear, `0x00002222` macrotile) |
| `0x074` | `SSPP_QOS_CTRL` | RW | Bit `16` `DANGER_SAFE_EN`, bit `0` `CREQ_VBLANK_EN` |
| `0x100` | `SSPP_SW_PIX_EXT_C0_LR` | RW | Plane 0 (`RGB`/`Y`) software pixel extension `(right << 8) \| left` |
| `0x104` | `SSPP_SW_PIX_EXT_C0_TB` | RW | Plane 0 (`RGB`/`Y`) software pixel extension `(bottom << 8) \| top` |
| `0x108` | `SSPP_SW_PIX_EXT_C0_REQ_PIXELS` | RW | Plane 0 required pixels: `(req_v << 16) \| req_h` |
| `0x110`..`0x118` | `SSPP_SW_PIX_EXT_C1C2_*` | RW | Planes 1/2 (`U`/`V`) pixel extension `LR`, `TB`, `REQ_PIXELS` |
| `0x120`..`0x128` | `SSPP_SW_PIX_EXT_C3_*` | RW | Plane 3 (`Alpha`) pixel extension `LR`, `TB`, `REQ_PIXELS` |
| `0x134` | `SSPP_CDP_CNTL` | RW | Client Driven Prefetch (`REC0`): bit `0` `EN`, bit `1` `PRELOAD_ADDR_AHEAD` (`1`=64 lines), bit `2` `TILE_AMORTIZE`, bit `3` `UBWC_META_PRELOAD_EN` |
| `0x13c` | `SSPP_CDP_CNTL_REC1` | RW | Client Driven Prefetch (`REC1`) |
| `0x15c` | `SSPP_OUT_SIZE_REC1` | RW | SmartDMA `REC1` destination size `(dst_h << 16) \| dst_w` |
| `0x160` | `SSPP_OUT_XY_REC1` | RW | SmartDMA `REC1` destination offset `(dst_y << 16) \| dst_x` |
| `0x164` | `SSPP_SRC_XY_REC1` | RW | SmartDMA `REC1` source offset `(src_y << 16) \| src_x` |
| `0x168` | `SSPP_SRC_SIZE_REC1` | RW | SmartDMA `REC1` source size `(src_h << 16) \| src_w` |
| `0x170` | `SSPP_MULTIRECT_OPMODE` | RW | Bit `0` `REC0_EN`, bit `1` `REC1_EN`, bit `2` `TIME_MULTIPLEX` (`0` = parallel, `1` = time-multiplexed when vertical regions do not overlap by <8 lines) |
| `0x174` | `SSPP_SRC_FORMAT_REC1` | RW | Pixel format for `REC1` |
| `0x178` | `SSPP_SRC_UNPACK_PATTERN_REC1` | RW | Unpack pattern for `REC1` |
| `0x17c` | `SSPP_SRC_OP_MODE_REC1` | RW | Flip / UBWC mode for `REC1` |
| `0x180` | `SSPP_SRC_CONSTANT_COLOR_REC1` | RW | Constant fill color for `REC1` |

### 6.2 `SSPP_SRC_FORMAT` (`0x030`) and `SSPP_SRC_UNPACK_PATTERN` (`0x034`) Encodings

**[HW]** `SSPP_SRC_FORMAT` (`0x030`) bitfields:
- Bits `[1:0]` `SRC_C0` (`G`/`Y` bit depth): `0` = 4-bit, `1` = 5-bit, `2` = 6-bit, `3` = 8-bit (`10-bit` when bit `8` `DX_FORMAT` is set).
- Bits `[3:2]` `SRC_C1` (`B`/`Cb` bit depth): `0` = 4-bit, `1` = 5-bit, `2` = 6-bit, `3` = 8-bit (`10-bit` with bit `8`).
- Bits `[5:4]` `SRC_C2` (`R`/`Cr` bit depth): `0` = 4-bit, `1` = 5-bit, `2` = 6-bit, `3` = 8-bit (`10-bit` with bit `8`).
- Bits `[7:6]` `SRC_C3` (`Alpha` bit depth): `0` = no alpha, `3` = 8-bit (`2-bit` with bit `8`).
- Bit `8` `DX_FORMAT`: `1` = 10-bit component format (`ARGB2101010`, `P010`, `TP10`).
- Bits `[10:9]` `BPS` (bytes per pixel minus 1 for interleaved formats): `0` = 1 Bpp, `1` = 2 Bpp, `2` = 3 Bpp, `3` = 4 Bpp.
- Bit `11` `INTERLEAVED`: `0` = interleaved, `1` = 2-plane pseudo-planar (`NV12`, `NV21`, `P010`), `2` (bits `[13:12]=2`) = 3-plane planar (`YUV420`).
- Bits `[15:14]` `UNPACK_COUNT`: number of components in Plane 0 minus 1 (`3` = 4 components, `2` = 3, `1` = 2, `0` = 1).
- Bit `17` `UNPACK_TIGHT`: `1` = tight packing, `0` = loose packing.
- Bit `18` `UNPACK_ALIGN_MSB`: `1` = MSB aligned (e.g., `P010`), `0` = LSB aligned.
- Bit `19` `SRCC3_EN`: `1` = enable per-pixel alpha channel (`ARGB8888`, `ABGR8888`), `0` = ignore alpha (`XRGB8888`, `XBGR8888`).
- Bit `20` `ROT90`: `1` = 90-degree UBWC tile rotation.
- Bit `21` `SRC_COLOR_SPACE`: `0` = RGB, `1` = YUV.
- Bit `22` `SOLID_FILL`: `1` = solid color fill from `SSPP_SRC_CONSTANT_COLOR` (no memory fetch).
- Bits `[24:23]` `CHROMA_SAMP`: `0` = `4:4:4`/RGB, `1` = `H2V1` (`4:2:2`), `2` = `H1V2`, `3` = `H2V2` (`4:2:0`).
- Bits `[26:25]` `FRAME_FORMAT`: `0` = linear, `2` = macrotile (`A5X`/`A6X` UBWC tile).

**[HW]** Component indices in `SSPP_SRC_UNPACK_PATTERN` (`0x034`) are: `0` = `C0` (`G`/`Y`), `1` = `C1` (`B`/`Cb`), `2` = `C2` (`R`/`Cr`), `3` = `C3` (`Alpha`). Common formats:

| Format | `SSPP_SRC_FORMAT` (Linear) | `SSPP_SRC_FORMAT` (UBWC) | `SSPP_SRC_UNPACK_PATTERN` |
|---|---|---|---|
| `XRGB8888` (`XR24`) | `0x000236ff` | `0x040236ff` (+ `OP_MODE` bit `31`) | `0x03020001` (`C3=3, C2=2, C1=0, C0=1`) |
| `ARGB8888` (`AR24`) | `0x000a36ff` | `0x040a36ff` (+ `OP_MODE` bit `31`) | `0x03020001` |
| `XBGR8888` (`XB24`) | `0x000236ff` | `0x040236ff` (+ `OP_MODE` bit `31`) | `0x03010002` (`C3=3, C2=1, C1=0, C0=2`) |
| `ABGR8888` (`AB24`) | `0x000a36ff` | `0x040a36ff` (+ `OP_MODE` bit `31`) | `0x03010002` |
| `RGB565` (`RG16`) | `0x00022216` | `0x04022216` (+ `OP_MODE` bit `31`) | `0x00020001` |
| `XRGB2101010` (`XR30`) | `0x000237ff` | `0x040237ff` (+ `OP_MODE` bit `31`) | `0x03020001` |
| `ARGB2101010` (`AR30`) | `0x000a37ff` | `0x040a37ff` (+ `OP_MODE` bit `31`) | `0x03020001` |
| `NV12` (`4:2:0` 8-bit) | `0x01a2083f` | `0x05a2083f` (+ `OP_MODE` bit `31`) | `0x00000201` |
| `P010` (`4:2:0` 10-bit) | `0x01a41b3f` | `0x05a2193f` (`TP10`) | `0x00000201` |

**[HW]** When UBWC is enabled (`FRAME_FORMAT = 2`, `SSPP_SRC_OP_MODE` bit `31` = `1`):
- For single-plane RGB UBWC (`AR24`, `XR24`, `AR30`, `XR30`), the metadata plane address goes into `SSPP_SRC2_ADDR` (`0x01c`) with its stride in `SSPP_SRC_YSTRIDE1[15:0]` (`0x028`), while the compressed pixel payload address goes into `SSPP_SRC0_ADDR` (`0x014`) with its stride in `SSPP_SRC_YSTRIDE0[15:0]` (`0x024`).
- For 2-plane YUV UBWC (`NV12_UBWC`, `TP10`), Y pixel payload -> `SRC0_ADDR`, UV pixel payload -> `SRC1_ADDR`, Y metadata -> `SRC2_ADDR`, UV metadata -> `SRC3_ADDR`.
- When software pixel extension override is enabled (`SSPP_SRC_OP_MODE` bit `31` `PE_OVERRIDE` = `1` for unpacked formats without scaling), program `SW_PIX_EXT_*_LR` = `0`, `SW_PIX_EXT_*_TB` = `0`, and `SW_PIX_EXT_*_REQ_PIXELS` = `(src_h << 16) | src_w` (dividing width/height by 2 for `C1C2` on `4:2:0`).

---

## 7. VIG Sub-Blocks: QSEED3lite Scaler (`+0xa00`) and 10-Bit CSC (`+0x1a00`)

### 7.1 QSEED3lite Scaler (`SSPP_VIG_BASE + 0x0a00`, Version `0x00030000`)

**[HW]** Each `VIG0..3` SSPP contains a QSEED3lite scaler at `+0x0a00`:

| Offset from `+0xa00` | Register Name | Access | Description |
|---|---|---|---|
| `0x000` | `QSEED3_OP_MODE` | RW | Bit `0` `SCALER_EN`, bit `1` `ALPHA_EN`, bits `[15:8]` `Y_RGB_FILTER_CFG`, bits `[23:16]` `UV_FILTER_CFG`, bits `[31:24]` `ALPHA_FILTER_CFG` |
| `0x004` | `QSEED3_PHASE_INIT` | RW | Initial phase shifts (`Y_H`, `Y_V`, `UV_H`, `UV_V`) |
| `0x008` | `QSEED3_PHASE_STEP_Y_H` | RW | Luma/RGB horizontal phase step (`(src_w << 21) / dst_w`, 21 fractional bits) |
| `0x00c` | `QSEED3_PHASE_STEP_Y_V` | RW | Luma/RGB vertical phase step (`(src_h << 21) / dst_h`) |
| `0x010` | `QSEED3_PHASE_STEP_UV_H` | RW | Chroma/Alpha horizontal phase step |
| `0x014` | `QSEED3_PHASE_STEP_UV_V` | RW | Chroma/Alpha vertical phase step |
| `0x018` | `QSEED3_PRELOAD` | RW | Preload values: `(y_v << 24) \| (y_h << 16) \| (uv_v << 8) \| uv_h` |
| `0x01c` | `QSEED3_DE_SHARPEN` | RW | Detail Enhancer sharpen level |
| `0x020` | `QSEED3_DE_SHARPEN_CTL` | RW | Detail Enhancer control |
| `0x024`..`0x03c` | `QSEED3_DE_*` | RW | Detail Enhancer shape, threshold, adjust data `0..2` |
| `0x060` | `QSEED3_DIR_WEIGHT` | RW | Directional filter weight (`0x00000040` on QSEED3lite) |
| `0x090` | `QSEED3_DST_SIZE` | RW | Scaler output size `(dst_h << 16) \| dst_w` |
| `0x094` | `QSEED3_COEF_RAM_CTRL` | RW | Coefficient LUT swap control |
| `0x100`..`0x180` | `QSEED3_SEP_LUT_0` | RW | 33 words (`132` bytes) Plane 0 separable filter coefficients |
| `0x300`..`0x380` | `QSEED3_SEP_LUT_1` | RW | 33 words (`132` bytes) Plane 1/2 separable filter coefficients |

**[HW]** When 1:1 unscaled (`src_w == dst_w` and `src_h == dst_h`) on an RGB surface, write `0x00000000` to `QSEED3_OP_MODE` (`+0xa00`) to bypass the scaler.

### 7.2 10-Bit Color Space Converter (`SSPP_VIG_BASE + 0x1a00`)

**[HW]** Each `VIG0..3` SSPP includes a 3x3 10-bit CSC at `+0x1a00` for YUV-to-RGB conversion:

| Offset from `+0x1a00` | Register Name | Access | Description |
|---|---|---|---|
| `0x000` | `CSC_10_OP_MODE` | RW | Installed at `SSPP_VIG_BASE + 0x1a00`: bit `0` `CSC_EN`, bit `1` `SRC_DATA_FORMAT` (`1`=YUV), bit `2` `DST_DATA_FORMAT` (`0`=RGB) |
| `0x004`..`0x014` | `CSC_10_COEFF_0..4` | RW | 9 signed S2.9 fixed-point matrix coefficients (`16 bits` per coeff, 2 per word in `0x04..0x10`, `C22` in `0x14`) |
| `0x018`..`0x020` | `CSC_10_PRE_CLAMP_0..2` | RW | Input clamp `(high_10b << 16) \| low_10b` for `Y`, `Cb`, `Cr` |
| `0x024`..`0x02c` | `CSC_10_POST_CLAMP_0..2` | RW | Output clamp `(high_10b << 16) \| low_10b` (`0x03ff0000` for full-range 10-bit RGB) |
| `0x030`..`0x038` | `CSC_10_PRE_BIAS_0..2` | RW | Input bias (`-64` = `0xffc0` for Y limited range, `-512` = `0xfe00` for Cb/Cr) |
| `0x03c`..`0x044` | `CSC_10_POST_BIAS_0..2` | RW | Output bias (`0x0000` for full-range RGB) |

**[HW]** Standard BT.601 limited-range YUV-to-RGB 10-bit CSC values:
- `COEFF_0..4` (`0x004`..`0x014`): `0x00000254`, `0x0396f370`, `0x02540409`, `0x00000254`, `0x00000331` (coeffs: `{0x254, 0x000, 0x331, 0x254, 0xf37, 0xe60, 0x254, 0x409, 0x000}`).
- `PRE_CLAMP_0..2` (`0x018`..`0x020`): `0x03ac0040`, `0x03c00040`, `0x03c00040`.
- `POST_CLAMP_0..2` (`0x024`..`0x02c`): `0x03ff0000`, `0x03ff0000`, `0x03ff0000`.
- `PRE_BIAS_0..2` (`0x030`..`0x038`): `0x0000ffc0`, `0x0000fe00`, `0x0000fe00`.
- `POST_BIAS_0..2` (`0x03c`..`0x044`): `0x00000000`, `0x00000000`, `0x00000000`.

---

## 8. Layer Mixer (`LM_0..5`, `DPU_BASE + 0x44000..0x49000`)

**[HW]** Each Layer Mixer composites up to 11 stages (`STAGE_0`..`STAGE_10` on top of `STAGE_BASE`) into an output canvas up to `2560` pixels wide. For modes wider than `2560` pixels (e.g., `3840x2160`), two paired Layer Mixers (`LM_0+LM_1`, `LM_2+LM_3`, or `LM_4+LM_5`) must run in left/right split mode and merge in their paired `MERGE_3D` block.

| Offset | Register Name | Access | Description |
|---|---|---|---|
| `0x000` | `LM_OP_MODE` | RW | Bit `31` `SPLIT_LEFT_RIGHT` (`1` = right-half mixer in a dual-LM pair, `0` = left-half or single mixer) |
| `0x004` | `LM_OUT_SIZE` | RW | Mixer output dimensions: `(height << 16) \| width` |
| `0x008` | `LM_BORDER_COLOR_0` | RW | Border color 0 (`(G_12b << 16) \| B_12b` or `(Alpha_8b << 24) \| (R_8b << 16)`) |
| `0x010` | `LM_BORDER_COLOR_1` | RW | Border color 1 (`(A_12b << 16) \| R_12b`) |
| `s_off + 0x00` | `LM_BLEND0_OP` | RW | Blend equation for stage `s` (`s = 0..10`; see table below) |
| `s_off + 0x04` | `LM_BLEND0_CONST_ALPHA` | RW | Constant alpha: `(fg_const_alpha_8b << 8) \| bg_const_alpha_8b` |
| `s_off + 0x08`..`0x14` | `LM_FG_COLOR_FILL_*` | RW | Dim-layer color fill (`COLOR_0`, `COLOR_1`, `SIZE`, `XY`) |

**[HW]** Stage base offsets `s_off` for stages `0..10`:
- `STAGE_0`: `0x020`, `STAGE_1`: `0x038`, `STAGE_2`: `0x050`, `STAGE_3`: `0x068`, `STAGE_4`: `0x080`, `STAGE_5`: `0x098`, `STAGE_6`: `0x0b0`, `STAGE_7`: `0x0c8`, `STAGE_8`: `0x0e0`, `STAGE_9`: `0x0f8`, `STAGE_10`: `0x110`.

**[HW]** `LM_BLEND0_OP` (`s_off + 0x00`) bitfields:
- Bits `[1:0]` `FG_ALPHA`: `0` = `FG_CONST`, `1` = `BG_CONST`, `2` = `FG_PIXEL`, `3` = `BG_PIXEL`.
- Bit `2` `FG_INV_ALPHA`: invert foreground alpha (`1 - alpha`).
- Bit `3` `FG_MOD_ALPHA`: modulate foreground pixel alpha by `fg_const_alpha`.
- Bits `[9:8]` `BG_ALPHA`: `0` = `FG_CONST`, `1` = `BG_CONST`, `2` = `FG_PIXEL`, `3` = `BG_PIXEL`.
- Bit `10` `BG_INV_ALPHA`: invert background alpha (`1 - alpha`).
- Bit `11` `BG_MOD_ALPHA`: modulate background alpha by `bg_const_alpha`.

**[HW]** Common blend modes:
- **Opaque plane** (`XRGB8888`): `LM_BLEND0_OP = 0x00000100` (`FG_ALPHA = FG_CONST`, `BG_ALPHA = BG_CONST`), `LM_BLEND0_CONST_ALPHA = 0x0000ff00` (`fg = 0xff, bg = 0x00`).
- **Premultiplied alpha plane** (`ARGB8888` with plane alpha `A`): `LM_BLEND0_OP = 0x00000e00` (`FG_ALPHA = FG_CONST`, `BG_ALPHA = FG_PIXEL | BG_INV_ALPHA | BG_MOD_ALPHA` if `A < 255`, or `0x00000600` if `A == 255`), `LM_BLEND0_CONST_ALPHA = (A << 8) | A`.
- **Coverage (non-premultiplied) alpha plane** (`ARGB8888`): `LM_BLEND0_OP = 0x00000602` (`FG_ALPHA = FG_PIXEL`, `BG_ALPHA = FG_PIXEL | BG_INV_ALPHA`), `LM_BLEND0_CONST_ALPHA = 0x0000ff00`.

---

## 9. `MERGE_3D`, `PINGPONG`, and `DSC 1.2`

### 9.1 `MERGE_3D_0..2` (`DPU_BASE + 0x4e000..0x50000`)

**[HW]** Each `MERGE_3D` block sits between a pair of Ping-Pong blocks (`PP_0+PP_1` -> `MERGE_3D_0`, `PP_2+PP_3` -> `MERGE_3D_1`, `PP_4+PP_5` -> `MERGE_3D_2`) and the downstream `INTF` or `DSC`:
- `0x000` `MERGE_3D_MUX`: `0x00000000`.
- `0x004` `MERGE_3D_MODE`: `0x00000000` = disabled (single Layer Mixer), `0x00000001` (`MODE_3D_H_ROW_INT`) = horizontally merge left (`PP_2n`) and right (`PP_2n+1`) Layer Mixers side-by-side.

### 9.2 `PINGPONG_0..5` (`DPU_BASE + 0x69000..0x6e000`) and Dither (`+0x0e0`)

**[HW]** On video-mode DisplayPort interfaces, the Ping-Pong tear-check block (`0x000`..`0x038`) is unused (`PP_TEAR_CHECK_EN` at `0x000` = `0`). The Ping-Pong dither sub-block at `PP_BASE + 0x0e0` dithers 10/12-bit mixer outputs down to 6-bit or 8-bit sinks:
- `PP_BASE + 0x0e0` `PP_DITHER_EN`: `1` = enable dither, `0` = bypass.
- `PP_BASE + 0x0e4` `PP_DITHER_BITDEPTH`: bits `[1:0]` `C0` (`G`), bits `[3:2]` `C1` (`B`), bits `[5:4]` `C2` (`R`), where `0`=4b, `1`=5b, `2`=6b, `3`=8b; bit `8` `TEMPORAL_DITHER_EN`.
- `PP_BASE + 0x0e8`..`0x0f4` `PP_DITHER_MATRIX_0..3`: 4x4 threshold matrix (`0x00005d7f`, `0x0000193b`, `0x00006e4c`, `0x00002a08`).

### 9.3 `DSC_0..5` (`DPU_BASE + 0x80000..0x82000`, VESA DSC 1.2)

**[HW]** Each Display Compression Engine (`DCE0` at `0x80000`, `DCE1` at `0x81000`, `DCE2` at `0x82000`) contains a common register (`DSC_CMN_MAIN_CNF` at `+0x000`, bit `0` = `1` for split-panel dual-slice mode) and two DSC 1.2 encoder slices:
- Even slice (`dsc_0`, `dsc_2`, `dsc_4`): encoder registers at `+0x100`..`+0x198`, control registers at `+0xf00`..`+0xf0c`.
- Odd slice (`dsc_1`, `dsc_3`, `dsc_5`): encoder registers at `+0x200`..`+0x298`, control registers at `+0xf80`..`+0xf8c`.
- At `ctl_base + 0x00` (`DSC_CTL`): write `(pingpong_idx & 0x7) | 0x8` to bind the DSC encoder to `PINGPONG_n`, or `0x0` to unbind.

---

## 10. Timing Engine Interface (`INTF_0..8`, `DPU_BASE + 0x34000..0x3c000`)

**[HW]** Each `INTF` generates video timing sync signals (`HSYNC`, `VSYNC`, `DE`) and streams pixel data from a Ping-Pong (or `MERGE_3D` / `DSC`) block to its paired DisplayPort or DSI controller.

### 10.1 INTF Register Map (Offsets from `INTF_BASE`)

| Offset | Register Name | Access | Description |
|---|---|---|---|
| `0x000` | `INTF_TIMING_ENGINE_EN` | RW | Bit `0`: `1` = enable timing generator, `0` = disable |
| `0x004` | `INTF_CONFIG` | RW | Bit `19` `ACTIVE_H_EN`, bit `20` `ACTIVE_V_EN`, bit `31` `PROG_FETCH_START_EN` |
| `0x008` | `INTF_HSYNC_CTL` | RW | `(hsync_period_clks << 16) \| pulse_width_clks` |
| `0x00c` | `INTF_VSYNC_PERIOD_F0` | RW | Total vertical period in pixel clocks (`vtotal * hsync_period_clks`) |
| `0x014` | `INTF_VSYNC_PULSE_WIDTH_F0` | RW | VSYNC pulse width in pixel clocks (`vsync_len * hsync_period_clks`) |
| `0x01c` | `INTF_DISPLAY_V_START_F0` | RW | Display vertical start in pixel clocks (`(v_back_porch + vsync_len) * hsync_period_clks`) |
| `0x024` | `INTF_DISPLAY_V_END_F0` | RW | Display vertical end in pixel clocks (`((v_back_porch + vsync_len + vdisplay) * hsync_period_clks) - 1`) |
| `0x03c` | `INTF_DISPLAY_HCTL` | RW | `(display_h_end << 16) \| display_h_start` (inclusive; in pixel clocks) |
| `0x040` | `INTF_ACTIVE_HCTL` | RW | `(active_h_end << 16) \| active_h_start` |
| `0x044` | `INTF_ACTIVE_V_START_F0` | RW | Active vertical start |
| `0x04c` | `INTF_ACTIVE_V_END_F0` | RW | Active vertical end |
| `0x050` | `INTF_BORDER_COLOR` | RW | Timing engine border color |
| `0x054` | `INTF_UNDERFLOW_COLOR` | RW | FIFO underflow replacement color (`0x00ff0000` red) |
| `0x058` | `INTF_HSYNC_SKEW` | RW | HSYNC skew (`0` on DP) |
| `0x05c` | `INTF_POLARITY_CTL` | RW | Bit `0` `HSYNC_LOW`, bit `1` `VSYNC_LOW`, bits `[3:2]` `DATA_EN_HIGH` (`0x3`); DP uses `0x0000000c` (active-high to DP controller) |
| `0x060` | `INTF_TEST_CTL` | RW | Test pattern generator control |
| `0x090` | `INTF_PANEL_FORMAT` | RW | Output panel bit depth: `0x213f` (`RGB888` 24-bit) or `0x21ff` (`RGB101010` 30-bit) |
| `0x170` | `INTF_PROG_FETCH_START` | RW | Programmable fetch start offset in pixel clocks before `DISPLAY_V_START_F0` |
| `0x234` | `INTF_CONFIG2` | RW | Bit `0` `DATABUS_WIDEN` (`1` = widebus 2 pixels/clk), bit `4` `DATA_HCTL_EN` (`1` = enable active HCTL), bit `12` `DCE_DATA_COMPRESS` |
| `0x238` | `INTF_DISPLAY_DATA_HCTL` | RW | Data horizontal window `(data_h_end << 16) \| data_h_start` |
| `0x23c` | `INTF_ACTIVE_DATA_HCTL` | RW | Active data horizontal window |
| `0x240` | `INTF_FRAME_LINE_COUNT_EN` | RW | Write `0x1` to enable frame and line counters |
| `0x244` | `INTF_FRAME_COUNT` | RO | Hardware vblank frame counter |
| `0x248` | `INTF_LINE_COUNT` | RO | Current vertical scanline counter |
| `0x25c` | `INTF_MUX` | RW | Source Ping-Pong selector (`0..5` = `PINGPONG_0..5`, `0xf` = disabled/unbound) |

### 10.2 DisplayPort Widebus Timing Formulas

**[HW]** On SC8280XP, all DisplayPort interfaces (`INTF_0`, `INTF_3`, `INTF_4`, `INTF_5`, `INTF_6`, `INTF_7`, `INTF_8`) operate with **widebus** enabled (`INTF_CONFIG2` bit `0` `DATABUS_WIDEN = 1` and bit `4` `DATA_HCTL_EN = 1`), transferring 2 pixels (64 bits) per `dptxN_pixel0_clk` cycle. Consequently, all horizontal timing counts programmed into `INTF` are divided by `2` (`>> 1`):
1. Given standard mode timings `hdisplay`, `hsync_start`, `hsync_end`, `htotal`, `vdisplay`, `vsync_start`, `vsync_end`, `vtotal`:
   - `h_back_porch = htotal - hsync_end`
   - `h_front_porch = hsync_start - hdisplay`
   - `hsync_pulse_width = hsync_end - hsync_start`
   - `v_back_porch = vtotal - vsync_end`
   - `v_front_porch = vsync_start - vdisplay`
   - `vsync_pulse_width = vsync_end - vsync_start`
2. Divide horizontal counts by 2 for widebus:
   - `hsync_period = htotal >> 1`
   - `pulse_width = hsync_pulse_width >> 1`
   - `h_start = (h_back_porch + hsync_pulse_width) >> 1`
   - `h_end = h_start + (hdisplay >> 1) - 1`
3. Apply the DisplayPort controller horizontal lead adjustment (`display_v_start` and `display_hctl` shift when widebus/DP is used):
   - `INTF_HSYNC_CTL` (`0x008`) = `(hsync_period << 16) | pulse_width`.
   - `INTF_VSYNC_PERIOD_F0` (`0x00c`) = `vtotal * hsync_period`.
   - `INTF_VSYNC_PULSE_WIDTH_F0` (`0x014`) = `vsync_pulse_width * hsync_period`.
   - `display_v_start = ((v_back_porch + vsync_pulse_width) * hsync_period) + h_start`.
   - `display_v_end = ((v_back_porch + vsync_pulse_width + vdisplay) * hsync_period) - (h_front_porch >> 1) - 1` (or `((v_back_porch + vsync_pulse_width + vdisplay) * hsync_period) - 1`).
   - `INTF_DISPLAY_HCTL` (`0x03c`) = `(h_end << 16) | h_start`.
   - `INTF_DISPLAY_DATA_HCTL` (`0x238`) = `(h_end << 16) | h_start`.
   - `INTF_CONFIG2` (`0x234`) = `0x00000011` (`DATABUS_WIDEN | DATA_HCTL_EN`).
4. Programmable Fetch (`INTF_PROG_FETCH_START` at `0x170`):
   - Set `vfp_fetch_lines` = `min(25, v_front_porch + v_back_porch + vsync_pulse_width - 2)` (typically `21` lines of prefetch when vertical blanking is >= 25 lines).
   - If `vfp_fetch_lines < v_back_porch + vsync_pulse_width`: `prog_fetch_start = ((v_back_porch + vsync_pulse_width - vfp_fetch_lines) * hsync_period) + 1`, and set `INTF_CONFIG` (`0x004`) bit `31` (`0x80000000`).

---

## 11. DPU Interrupt Controller (DPU 7.x / 8.x Layout)

**[HW]** DPU pipeline interrupts (vsync, underrun, pingpong done) are controlled by interrupt register banks relative to `DPU_BASE` (`0x0ae01000`). Each bank has three 32-bit registers: `STATUS` (`+0x0`, read-only masked status), `EN` (`+0x4`, read/write interrupt enable mask), and `CLEAR` (`+0x8`, write-1-to-clear latch).

| Bank Index (`1-based`) | Base Offset from `DPU_BASE` | `STATUS` | `EN` | `CLEAR` | Description |
|---|---|---|---|---|---|
| `1` (`MDP_SSPP_TOP0_INTR`) | `0x00010` | `0x00010` | `0x00014` | `0x00018` | Primary INTF vsync, underrun, Ping-Pong done (`PP_0..3`) |
| `2` (`MDP_SSPP_TOP0_INTR2`) | `0x00008` | `0x0000c` | `0x00008` | `0x00010`* | Secondary interrupts (`PP_4..5`); note `EN=0x08`, `STATUS=0x0c`, `CLEAR=0x2c` |
| `3` (`MDP_SSPP_TOP0_HIST_INTR`) | `0x0001c` | `0x0001c` | `0x00020` | `0x00024` | DSPP histogram interrupts |
| `9` (`INTF_0_INTR`) | `0x342b0` | `0x342b0` | `0x342b4` | `0x342b8` | `INTF_0` local interrupt bank |
| `12` (`INTF_3_INTR`) | `0x372b0` | `0x372b0` | `0x372b4` | `0x372b8` | `INTF_3` local interrupt bank |
| `13` (`INTF_4_INTR`) | `0x382b0` | `0x382b0` | `0x382b4` | `0x382b8` | `INTF_4` local interrupt bank |
| `14` (`INTF_5_INTR`) | `0x392b0` | `0x392b0` | `0x392b4` | `0x392b8` | `INTF_5` local interrupt bank |
| `15` (`INTF_6_INTR`) | `0x3a2b0` | `0x3a2b0` | `0x3a2b4` | `0x3a2b8` | `INTF_6` local interrupt bank |
| `16` (`INTF_7_INTR`) | `0x3b2b0` | `0x3b2b0` | `0x3b2b4` | `0x3b2b8` | `INTF_7` local interrupt bank |
| `17` (`INTF_8_INTR`) | `0x3c2b0` | `0x3c2b0` | `0x3c2b4` | `0x3c2b8` | `INTF_8` local interrupt bank |

### 11.1 `MDP_SSPP_TOP0_INTR` (`STATUS=0x010`, `EN=0x014`, `CLEAR=0x018`) Bit Assignments

**[HW]** On SC8280XP, all `INTF` VSYNC and Underrun interrupts are aggregated directly in Bank 1 (`DPU_BASE + 0x00010`..`0x00018`):

| Interface | `UNDERRUN` Bit | `VSYNC` Bit | Mask (`VSYNC`) | Notes |
|---|---|---|---|---|
| `INTF_0` (`DP_0`) | Bit `24` | Bit `25` | `0x02000000` | USB-C Port 0 primary stream |
| `INTF_1` (`DSI_0`) | Bit `26` | Bit `27` | `0x08000000` | DSI 0 |
| `INTF_2` (`DSI_1`) | Bit `28` | Bit `29` | `0x20000000` | DSI 1 |
| `INTF_3` (`DP_0` MST1) | Bit `30` | Bit `31` | `0x80000000` | USB-C Port 0 secondary stream |
| `INTF_4` (`DP_1`) | Bit `20` | Bit `21` | `0x00200000` | USB-C Port 1 primary stream |
| `INTF_5` (`DP_3` / `eDP`) | Bit `22` | Bit `23` | `0x00800000` | eDP / DP3 |
| `INTF_6` (`DP_2`) | Bit `16` | Bit `17` | `0x00020000` | Dragon Q8B HDMI (`CH7218A`) |
| `INTF_7` (`DP_2` MST1) | Bit `18` | Bit `19` | `0x00080000` | DP2 secondary stream |
| `INTF_8` (`DP_1` MST1) | Bit `12` | Bit `13` | `0x00002000` | USB-C Port 1 secondary stream |

---

## 12. Atomic Mode-Set and Page-Flip Programming Sequence

### 12.1 Full Mode-Set Sequence (Single Layer Mixer `width <= 2560`, e.g., `SSPP_VIG2` -> `LM_2` -> `CTL_2` -> `PINGPONG_2` -> `INTF_6`)

**[HW]** To configure a display pipeline from scratch or during a mode change (with `INTF_TIMING_ENGINE_EN` currently `0`):
1. **SSPP Configuration (`SSPP_BASE`)**:
   - Write `SSPP_SRC_FORMAT`, `SSPP_SRC_UNPACK_PATTERN`, `SSPP_SRC_OP_MODE`, `SSPP_SRC_SIZE`, `SSPP_SRC_IMG_SIZE`, `SSPP_SRC_XY`, `SSPP_OUT_SIZE`, `SSPP_OUT_XY`, `SSPP_SRC0_ADDR`..`SSPP_SRC3_ADDR`, `SSPP_SRC_YSTRIDE0..1`.
   - Write `SSPP_MULTIRECT_OPMODE` (`0x170`) = `0x00000001` (`REC0_EN`).
   - Write `SSPP_DANGER_LUT`, `SSPP_SAFE_LUT`, `SSPP_CREQ_LUT_0..1`, `SSPP_QOS_CTRL` (`0x00010001`), and `SSPP_CDP_CNTL` (`0x00000007` linear or `0x0000000f` UBWC).
   - If using `VIG0..3`, program `QSEED3_OP_MODE` (`+0xa00`) and `CSC_10_OP_MODE` (`+0x1a00`) (`0` for unscaled RGB).
2. **Layer Mixer Configuration (`LM_BASE`)**:
   - Write `LM_OUT_SIZE` (`0x004`) = `(vdisplay << 16) | hdisplay` and `LM_OP_MODE` (`0x000`) = `0`.
   - For each active stage `s`, write `LM_BLEND0_OP` (`0x20 + 0x18*s`) and `LM_BLEND0_CONST_ALPHA` (`0x24 + 0x18*s`).
3. **MERGE_3D & PINGPONG Configuration**:
   - Write `MERGE_3D_MODE` (`0x004`) = `0` (single mixer) or `1` (dual mixer).
   - Write `INTF_MUX` (`INTF_BASE + 0x25c`) = `pingpong_idx` (e.g., `2` for `PINGPONG_2`).
4. **INTF Timing Configuration (`INTF_BASE`)**:
   - Program `INTF_HSYNC_CTL`, `INTF_VSYNC_PERIOD_F0`, `INTF_VSYNC_PULSE_WIDTH_F0`, `INTF_DISPLAY_V_START_F0`, `INTF_DISPLAY_V_END_F0`, `INTF_DISPLAY_HCTL`, `INTF_DISPLAY_DATA_HCTL`, `INTF_POLARITY_CTL` (`0x0c`), `INTF_PANEL_FORMAT` (`0x213f`), `INTF_CONFIG2` (`0x11` for widebus), `INTF_PROG_FETCH_START`, `INTF_CONFIG` (`0x80000000`), and `INTF_FRAME_LINE_COUNT_EN` (`0x1`).
5. **CTL Binding and Flush (`CTL_BASE`)**:
   - Write `CTL_LAYER(lm)` (`0x000 + 4*lm`) with the stage assignments and bit `24` (`BORDER_OUT`).
   - Write `CTL_FETCH_PIPE_ACTIVE` (`0x0fc`), `CTL_LAYER_ACTIVE` (`0x130`), `CTL_MERGE_3D_ACTIVE` (`0x0e4`), and `CTL_INTF_ACTIVE` (`0x0f4`).
   - Write `CTL_INTF_FLUSH` (`0x110`) = `1 << intf_idx` (e.g., `0x40` for `INTF_6`).
   - Write `CTL_FLUSH` (`0x018`) = `sspp_flush_mask | lm_flush_mask | (1 << 17) | (1 << 31)` (e.g., `0x80020104` for `VIG2 + LM2 + CTL + INTF`).
6. **Start Timing Engine**:
   - After the DisplayPort link is trained and ready to accept video, write `0x00000001` to `INTF_TIMING_ENGINE_EN` (`INTF_BASE + 0x000`).

### 12.2 Atomic Page-Flip Sequence

**[HW]** To flip the scanout address on an already-running pipeline without changing mode or plane geometry:
1. Write the new 32-bit surface address(es) to `SSPP_SRC0_ADDR` (`+0x014`, and `SRC1..3_ADDR` if multi-plane/UBWC) and stride to `SSPP_SRC_YSTRIDE0..1` (`+0x024..+0x028`).
2. Write the SSPP's flush bit to `CTL_FLUSH` (`CTL_BASE + 0x018`; e.g., `0x00000004` for `VIG2`).
3. At the next vertical blanking edge (`VSYNC` interrupt in `MDP_SSPP_TOP0_INTR` at `DPU_BASE + 0x00010`), hardware latches the new address and automatically clears the bit in `CTL_FLUSH` back to `0`. The previous framebuffer is safe to reuse or unmap once `CTL_FLUSH` reads `0` after `VSYNC`.
