# DisplayPort / eDP Controller, AUX Channel, Link Training, MSA, and Transfer Unit Specification (`SC8280XP`)

## 1. DisplayPort / eDP Controller Instances and Sub-Regions

**[HW]** Each MDSS instance (`MDSS0` at `0x0ae00000`, `MDSS1` at `0x22000000`) integrates four DisplayPort / Embedded DisplayPort controllers:

| Controller | `MDSS0` Base | `MDSS1` Base | `MDSS` IRQ Bit | Primary `INTF` | MST Stream 1 `INTF` | Dragon Q8B Routing (`MDSS0`) |
|---|---|---|---|---|---|---|
| `DP0` | `0x0ae90000` | `0x22090000` | Bit `12` | `INTF_0` | `INTF_3` | USB-C Port 0 (via QMP Combo PHY 0 `0x088eb000`) |
| `DP1` | `0x0ae98000` | `0x22098000` | Bit `13` | `INTF_4` | `INTF_8` | USB-C Port 1 (via QMP Combo PHY 1 `0x08903000`) |
| `DP2` | `0x0ae9a000` | `0x2209a000` | Bit `14` | `INTF_6` | `INTF_7` | Onboard Chrontel CH7218A HDMI Bridge (via DP2 PHY `0x0aec2000`) |
| `DP3` (`eDP`) | `0x0aea0000` | `0x220a0000` | Bit `15` | `INTF_5` | — | Internal eDP / DP3 (via DP3 PHY `0x0aec5000`) |

**[HW]** Each DP controller MMIO block is divided into four clock-domain sub-regions relative to `DP_BASE`:

| Sub-Region | Offset from `DP_BASE` | Size | Clock Domain | Purpose |
|---|---|---|---|---|
| `AHB` | `+0x0000` | `0x0200` | `disp_cc_mdss_ahb_clk` | Hardware version, software reset, top-level interrupt status/mask |
| `AUX` | `+0x0200` | `0x0200` | `disp_cc_mdss_dptxN_aux_clk` (`19.2 MHz`) | AUX FIFO, native/I2C-over-AUX transactions, HPD state machine |
| `LINK` | `+0x0400` | `0x0c00` | `disp_cc_mdss_dptxN_link_clk` | Main link control, lane mapping, link training, TU, test patterns |
| `P0` | `+0x1000` | `0x0400` | `disp_cc_mdss_dptxN_pixel0_clk` | Stream 0 `MVID`/`NVID`, timing generator, active/total/porch, widebus |
| `P1` | `+0x2000` (`DP0` only) | `0x0200` | `disp_cc_mdss_dptx0_pixel1_clk` | Stream 1 (`MST`) pixel domain registers |

---

## 2. Complete DP Controller Register Map

### 2.1 `AHB` Sub-Region (`DP_BASE + 0x0000`)

| Offset | Register Name | Access | Description |
|---|---|---|---|
| `0x0000` | `DP_HW_VERSION` | RO | Controller revision (`0x10040000` = v1.4) |
| `0x0010` | `DP_SW_RESET` | RW | Bit `0` `SW_RESET`: write `1` to assert reset, then `0` to release |
| `0x0020` | `DP_INTR_STATUS` | RW1C/RW | Top-level interrupt status & mask (see bit table below) |
| `0x0024` | `DP_INTR_STATUS2` | RW1C/RW | Secondary interrupt status & mask (link, TU, video ready) |
| `0x0028` | `DP_INTR_STATUS3` | RW1C/RW | Tertiary interrupt status & mask |

**[HW]** `DP_INTR_STATUS` (`0x0020`) bit definitions (status bits are `[14:0]`, write-1-to-clear acknowledgment bits are `[14:0]` or shifted by `+1`, and interrupt enable bits are shifted by `+2` in each 3-bit group):
- Bit `0` (`0x0001`): `AUX_I2C_DONE` (ack bit `1` `0x0002`, mask bit `2` `0x0004`).
- Bit `3` (`0x0008`): `WRONG_ADDR` (ack bit `4` `0x0010`, mask bit `5` `0x0020`).
- Bit `6` (`0x0040`): `CONSECUTIVE_TIMEOUT` (ack bit `7` `0x0080`, mask bit `8` `0x0100`).
- Bit `9` (`0x0200`): `CONSECUTIVE_NACK_DEFER` (ack bit `10` `0x0400`, mask bit `11` `0x0800`).
- Bit `12` (`0x1000`): `WRONG_READDATA_COUNT` (ack bit `13` `0x2000`, mask bit `14` `0x4000`).
- Bit `15` (`0x8000`): `DPPHY_AUX_ERROR` (ack bit `16` `0x10000`, mask bit `17` `0x20000`).
- Combined AUX status mask: `0x00009249`, ack shift `<< 1`, enable mask `0x00024924`.

**[HW]** `DP_INTR_STATUS2` (`0x0024`) bit definitions:
- Bit `0` (`0x0001`): `READY_FOR_VIDEO` (ack bit `1` `0x0002`, mask bit `2` `0x0004`).
- Bit `3` (`0x0008`): `IDLE_PATTERNS_SENT` (ack bit `4` `0x0010`, mask bit `5` `0x0020`).
- Bit `6` (`0x0040`): `FRAME_END` (ack bit `7` `0x0080`, mask bit `8` `0x0100`).
- Bit `9` (`0x0200`): `CRC_UPDATED` (ack bit `10` `0x0400`, mask bit `11` `0x0800`).

### 2.2 `AUX` Sub-Region (`DP_BASE + 0x0200`)

| Offset | Register Name | Access | Description |
|---|---|---|---|
| `0x0200` | `DP_DP_HPD_CTRL` | RW | Bit `0` `HPD_EN` (`1` = enable HPD detector) |
| `0x0204` | `DP_DP_HPD_INT_STATUS` | RO | Bit `0` `PLUG_INT`, bit `1` `IRQ_HPD_INT`, bit `2` `REPLUG_INT`, bit `3` `UNPLUG_INT`, bits `[31:28]` `STATE_STATUS` (`0`=disconnected, `2`=connected, `4`=pending) |
| `0x0208` | `DP_DP_HPD_INT_ACK` | WO | Write-1-to-clear bits `[3:0]` corresponding to `HPD_INT_STATUS` |
| `0x020c` | `DP_DP_HPD_INT_MASK` | RW | Interrupt enable mask for bits `[3:0]` (`0x0000000f` = enable all HPD interrupts) |
| `0x0210` | `DP_DP_HPD_REFTIMER` | RW | Reference timer (`0x00030013` for `19.2 MHz` AUX clock, bit `16` `REFTIMER_ENABLE`) |
| `0x0218` | `DP_DP_HPD_EVENT_TIME_0` | RW | Glitch/connect filter timings (`0x3e80fa00`: `250 us` glitch, `2 ms` IRQ_HPD) |
| `0x021c` | `DP_DP_HPD_EVENT_TIME_1` | RW | Disconnect/replug timings (`0x0fa001f4`: `500 us` replug, `100 ms` unplug) |
| `0x0220` | `DP_AUX_CTRL` | RW | Bit `0` `AUX_EN` (`1` = enable AUX controller), bit `1` `AUX_RESET` (`1` = reset AUX block) |
| `0x0224` | `DP_AUX_DATA` | RW | AUX TX/RX FIFO register: bit `0` `AUX_WRITE` (`0` = TX write, `1` = RX read), bits `[15:8]` `DATA` byte, bits `[23:16]` `INDEX` (`0..19`), bit `31` `INDEX_WRITE` (`1` = set FIFO pointer to `INDEX`) |
| `0x0228` | `DP_AUX_TRANS_CTRL` | RW | Bits `[8:0]` `NUM_OF_BYTES` (`len - 1`), bit `8` `TRANS_TYPE` (`0` = Native AUX, `1` = I2C-over-AUX), bit `9` `GO` (`1` = start transaction), bit `10` `NO_SEND_ADDR`, bit `11` `NO_SEND_STOP` |
| `0x022c` | `DP_AUX_TIMEOUT_COUNT` | RW | Timeout counter (`0x0000ffff`) |
| `0x0230` | `DP_AUX_LIMITS` | RW | Retry limits (`0x0000ffff`) |
| `0x0234` | `DP_AUX_STATUS` | RO | AUX reply status, bits `[7:0]` bytes read |

### 2.3 `LINK` Sub-Region (`DP_BASE + 0x0400`)

| Offset | Register Name | Access | Description |
|---|---|---|---|
| `0x0400` | `DP_MAINLINK_CTRL` | RW | Bit `0` `MAINLINK_EN`, bit `1` `MAINLINK_RESET`, bit `16` `FB_BOUNDARY_SEL` (`0x02000000` default) |
| `0x0404` | `DP_STATE_CTRL` | RW | Link state machine command bitmap (see Section 4) |
| `0x0408` | `DP_CONFIGURATION_CTRL` | RW | Framing, lane count, BPC, colorimetry, MVID/NVID mode (see Section 5) |
| `0x040c` | `DP_SOFTWARE_MVID` | RW | 24-bit software `MVID` value (scaled for widebus/link rate; see Section 5.1) |
| `0x0414` | `DP_SOFTWARE_NVID` | RW | 24-bit software `NVID` value (`NVID * 5` on link rates >= HBR2) |
| `0x0418` | `DP_TOTAL_HOR_VER` | RW | `(vtotal << 16) \| htotal` (in full pixels, NOT divided by 2) |
| `0x041c` | `DP_START_HOR_VER_FROM_SYNC` | RW | `((v_back_porch + vsync_len) << 16) \| (h_back_porch + hsync_len)` |
| `0x0420` | `DP_HSYNC_VSYNC_WIDTH_POLARITY` | RW | `((vsync_pol << 31) \| (vsync_len << 16) \| (hsync_pol << 15) \| hsync_len)` (`pol`: `1` = active-low, `0` = active-high) |
| `0x0424` | `DP_ACTIVE_HOR_VER` | RW | `(vdisplay << 16) \| hdisplay` |
| `0x0428` | `DP_MISC1_MISC0` | RW | DisplayPort MSA `MISC0` (`bits [7:0]`) and `MISC1` (`bits [15:8]`); bit `0` = `1` (synchronous clock), bits `[7:5]` = BPC (`0`=6bpc, `1`=8bpc, `2`=10bpc) |
| `0x042c` | `DP_VALID_BOUNDARY` | RW | `(tu_size_desired << 16) \| valid_boundary_link` (see Section 6) |
| `0x0430` | `DP_VALID_BOUNDARY_2` | RW | `(boundary_moderation_en << 0) \| (upper_boundary_count << 1) \| (lower_boundary_count << 16) \| (valid_lower_boundary_link << 20)` |
| `0x0434` | `DP_LOGICAL2PHYSICAL_LANE_MAPPING` | RW | Logical-to-physical lane map (`0x000000e4` = `{0->0, 1->1, 2->2, 3->3}`) |
| `0x0438` | `DP_MAINLINK_READY` | RO | Bit `0` `READY_FOR_VIDEO`, bit `3` `TPS1_SENT`, bit `4` `TPS2_SENT`, bit `5` `TPS3_SENT`, bit `6` `IDLE_PATTERNS_SENT`, bit `16` `TPS4_SENT` |
| `0x043c` | `DP_MAINLINK_LEVELS` | RW | Safe-to-exit link FIFO level (`bits [5:0]`, default `0x00000002`, programmed to `safe_to_exit_level`) |
| `0x044c` | `DP_TU` | RW | Transfer unit size (`tu_size_minus1` in bits `[5:0]`) |
| `0x0458` | `DP_HBR2_COMPLIANCE_SCRAMBLER_RESET` | RW | Bit `16` (`0x00010000`) + `0x01ff` (`511` symbols) for ASSR/HBR2 compliance |
| `0x0480` | `DP_TEST_80BIT_CUSTOM_PATTERN_REG0` | RW | Custom 80-bit test pattern bits `[31:0]` |
| `0x0484` | `DP_TEST_80BIT_CUSTOM_PATTERN_REG1` | RW | Custom 80-bit test pattern bits `[63:32]` |
| `0x0488` | `DP_TEST_80BIT_CUSTOM_PATTERN_REG2` | RW | Custom 80-bit test pattern bits `[79:64]` |
| `0x0494` | `DP_COMP_CTL` | RW | DSC compression control |

### 2.4 `P0` Sub-Region (`DP_BASE + 0x1000`)

| Offset | Register Name | Access | Description |
|---|---|---|---|
| `0x1000` | `DP_TIMING_ENGINE_EN` | RW | Pixel domain timing engine enable |
| `0x1004` | `DP_INTF_CONFIG` | RW | Bit `0` `DATABUS_WIDEN` (`1` = enable 64-bit 2-pixels/clk widebus input from DPU `INTF`) |
| `0x1008` | `DP_INTF_HSYNC_CTL` | RW | Pixel domain HSYNC control |
| `0x100c`..`0x104c` | `DP_INTF_*` | RW | Pixel domain timing mirrors |
| `0x1080` | `DP_P0CLK_SW_RESET` | RW | Pixel clock domain software reset |

---

## 3. AUX Channel Transactions (Native AUX & I2C-over-AUX)

### 3.1 AUX and HPD Initialization

**[HW]** Before performing any AUX transaction:
1. Ensure `disp_cc_mdss_dptxN_aux_clk` (`19.2 MHz`) and the PHY's AUX block are enabled and calibrated (`QSERDES_DP_PHY_PD_CTL` bit `0` `PWRDN=1`, bit `2` `AUX_PWRDN=1`, `QSERDES_DP_PHY_AUX_CFG0..9` written; see `03-dispcc-and-combo-phy.md`).
2. Reset and enable the AUX controller: read `DP_AUX_CTRL` (`0x0220`), set bit `1` (`AUX_RESET`), write, delay `1000 us`, clear bit `1`, set bit `0` (`AUX_EN`), and write `0x00000001`.
3. Write `DP_AUX_TIMEOUT_COUNT` (`0x022c`) = `0x0000ffff` and `DP_AUX_LIMITS` (`0x0230`) = `0x0000ffff`.
4. To enable hardware HPD detection (used on dedicated eDP/DP bridges such as `DP2` -> CH7218A on Dragon Q8B):
   - Write `DP_DP_HPD_REFTIMER` (`0x0210`) = `0x00030013` (`19.2 MHz` reference + bit `16` enable).
   - Write `DP_DP_HPD_INT_ACK` (`0x0208`) = `0x0000000f`.
   - Write `DP_DP_HPD_INT_MASK` (`0x020c`) = `0x0000000f`.
   - Write `DP_DP_HPD_CTRL` (`0x0200`) = `0x00000001` (`HPD_EN`).

### 3.2 AUX FIFO Header Packing (`DP_AUX_DATA` at `0x0224`)

**[HW]** An AUX transaction message consists of a 4-byte command/address/length header followed (for writes) by `1..16` data payload bytes, written sequentially into the 20-byte hardware FIFO via `DP_AUX_DATA` (`0x0224`).

**[HW]** Given:
- `request`: 4-bit AUX request code:
  - Native AUX Write: `0x8` (`1000b`), Native AUX Read: `0x9` (`1001b`).
  - I2C-over-AUX Write: `0x0` (`0000b`, or `0x4` with `MOT=1`), I2C-over-AUX Read: `0x1` (`0001b`, or `0x5` with `MOT=1`).
- `address`: 20-bit DPCD or I2C target address (`0x00000`–`0xfffff`, e.g., `0x00050` for EDID I2C address `0x50`).
- `size`: number of payload bytes (`0` for address-only I2C-over-AUX packet, or `1..16` data bytes).

**[HW]** Construct the 4 header bytes `header[0..3]`:
1. `header[0] = ((request & 0x0f) << 4) | ((address >> 16) & 0x0f)`.
2. `header[1] = (address >> 8) & 0xff`.
3. `header[2] = address & 0xff`.
4. `header[3] = (size > 0) ? (size - 1) : 0`.

**[HW]** Step-by-step AUX transfer execution:
1. Write the 4 header bytes into `DP_AUX_DATA` (`0x0224`):
   - For `i = 0`: write `(header[0] << 8) | 0x80000000` (`INDEX = 0`, `INDEX_WRITE = 1`, `AUX_WRITE = 0`).
   - For `i = 1..3`: write `(header[i] << 8)`.
2. If the transaction is a write (`(request & 1) == 0`) and `size > 0`:
   - For `j = 0 .. size - 1`: write `(tx_buf[j] << 8)` to `DP_AUX_DATA` (`0x0224`).
3. Configure `DP_AUX_TRANS_CTRL` (`0x0228`):
   - Set `trans_ctrl = 0x00000200` (`GO` bit `9`).
   - If the transaction is I2C-over-AUX (`(request & 0x8) == 0`):
     - Set bit `8` (`TRANS_TYPE = 1`, `0x00000100`).
     - If `MOT` (`(request & 0x4) != 0`) is set, set bit `11` (`NO_SEND_STOP = 1`, `0x00000800`); if `size == 0`, also set bit `10` (`NO_SEND_ADDR = 1`, `0x00000400`).
   - Set bits `[7:0]` (`NUM_OF_BYTES`) = `(size > 0 && (request & 1) == 0) ? (4 + size - 1) : 3` (i.e., `3` for any read or 0-byte write, `3 + size` for a write with data).
   - Write `trans_ctrl` to `DP_AUX_TRANS_CTRL` (`0x0228`).
4. Wait for completion in `DP_INTR_STATUS` (`0x0020`, either via interrupt or polling up to `50 ms`):
   - Wait until any bit in `0x00009249` (`AUX_I2C_DONE | WRONG_ADDR | CONSECUTIVE_TIMEOUT | CONSECUTIVE_NACK_DEFER | WRONG_READDATA_COUNT | DPPHY_AUX_ERROR`) is set in `DP_INTR_STATUS`.
   - Clear `GO` (`bit 9`) in `DP_AUX_TRANS_CTRL` (`0x0228`).
   - Acknowledge the status bits by writing `(status & 0x00009249) << 1` back to `DP_INTR_STATUS` (`0x0020`).
   - If `DPPHY_AUX_ERROR` (`bit 15`) occurred, recalibrate the PHY AUX CFG1 setting and pulse `AUX_RESET` (`bit 1` of `DP_AUX_CTRL`), then retry (up to 32 times per VESA DP standard).
5. If `AUX_I2C_DONE` (`bit 0`) is set and the transaction was a read (`(request & 1) == 1`) with `size > 0`:
   - Write `0x80000001` (`INDEX_WRITE = 1, INDEX = 0, AUX_WRITE = 1`) to `DP_AUX_DATA` (`0x0224`).
   - Read `DP_AUX_DATA` (`0x0224`) once and discard it (this dummy read consumes the sink's 1-byte AUX reply status header).
   - For `j = 0 .. size - 1`: read `val = DP_AUX_DATA` (`0x0224`), check that `index = (val >> 16) & 0xff` equals `j + 1`, and store `rx_buf[j] = (val >> 8) & 0xff`.

---

## 4. Link Training Sequence (TPS1 Clock Recovery & TPS2/3/4 Channel Equalization)

**[HW]** SC8280XP supports 1, 2, or 4 lanes at four standard VESA link rates:
- **RBR**: `162000 kHz` (`1.62 Gbps/lane`, DPCD `LINK_BW_SET` `0x100` = `0x06`).
- **HBR**: `270000 kHz` (`2.70 Gbps/lane`, DPCD `LINK_BW_SET` `0x100` = `0x0a`).
- **HBR2**: `540000 kHz` (`5.40 Gbps/lane`, DPCD `LINK_BW_SET` `0x100` = `0x14`).
- **HBR3**: `810000 kHz` (`8.10 Gbps/lane`, DPCD `LINK_BW_SET` `0x100` = `0x1e`).

**[HW]** `DP_STATE_CTRL` (`0x0404`) command bits (always write `0x00000000` first before writing a new state command):
- `0x00000001`: `LINK_TRAINING_PATTERN1` (`TPS1` for Clock Recovery).
- `0x00000002`: `LINK_TRAINING_PATTERN2` (`TPS2` for Channel EQ on DP 1.1 sinks).
- `0x00000004`: `LINK_TRAINING_PATTERN3` (`TPS3` for Channel EQ on DP 1.2+ sinks where `DPCD 0x002` bit `6` `TPS3_SUPPORTED` is set).
- `0x00000008`: `LINK_TRAINING_PATTERN4` (`TPS4` for Channel EQ on DP 1.4+ sinks where `DPCD 0x003` bit `7` `TPS4_SUPPORTED` is set; note `TPS4` is sent scrambled, so DPCD `0x102` bit `5` `SCRAMBLING_DISABLE` must be `0`).
- `0x00000010`: `LINK_SYMBOL_ERR_MEASURE`.
- `0x00000020`: `LINK_PRBS7`.
- `0x00000040`: `LINK_TEST_CUSTOM_PATTERN`.
- `0x00000080`: `SEND_VIDEO` (normal active video transmission).
- `0x00000100`: `PUSH_IDLE` (send idle patterns prior to disabling stream).

### 4.1 Step-by-Step Link Training Algorithm

**[HW]** Before starting Link Training, the PHY PLL and TX lanes must be configured for the target `(link_rate, lane_count)`, and `disp_cc_mdss_dptxN_link_clk` (programmed to `link_rate_khz / 10` in kHz, i.e., `162`, `270`, `540`, or `810 MHz`), `dptxN_link_intf_clk`, and `dptxN_pixel0_clk` must be running:
1. **Controller Link Setup**:
   - Write `DP_LOGICAL2PHYSICAL_LANE_MAPPING` (`0x0434`) = `0x000000e4`.
   - Write `DP_HBR2_COMPLIANCE_SCRAMBLER_RESET` (`0x0458`) = `0x000101ff` (if ASSR / eDP is used, also write `0x01` to DPCD `0x10a` and set bit `10` `ASSR` in `DP_CONFIGURATION_CTRL`).
   - Write `DP_CONFIGURATION_CTRL` (`0x0408`) with lane count, enhanced framing, and BPC (see Section 5).
   - Pulse `MAINLINK_RESET` (bit `1`) in `DP_MAINLINK_CTRL` (`0x0400`), then set `MAINLINK_EN` (`0x02000001`).
   - Wake sink by writing `0x01` (`D0`) to DPCD `SET_POWER` (`0x00600`), and if `DOWNSPREAD_CTRL` (`0x003` bit `0`) is supported, write `0x10` or `0x00` to DPCD `0x107` and `0x01` (8b/10b coding) to DPCD `0x108`.
   - Write `bw_code` (`0x06`, `0x0a`, `0x14`, or `0x1e`) to DPCD `LINK_BW_SET` (`0x00100`) and `lane_count | (enhanced_frame ? 0x80 : 0)` to DPCD `LANE_COUNT_SET` (`0x00101`).
2. **Phase 1: Clock Recovery (`TPS1`)**:
   - Initialize all 4 lanes to `voltage_swing = 0`, `pre_emphasis = 0`, and program the PHY TX `DRV_LVL` and `EMP_POST1_LVL` registers.
   - Write `0x00000000` then `0x00000001` (`LINK_TRAINING_PATTERN1`) to `DP_STATE_CTRL` (`0x0404`).
   - Write `0x21` (`TRAINING_PATTERN_1 | SCRAMBLING_DISABLE`) to DPCD `TRAINING_PATTERN_SET` (`0x00102`), followed by the 4-byte `TRAINING_LANE0_3_SET` array at DPCD `0x00103..0x00106`.
   - Loop up to 10 iterations (or until maximum voltage swing `3` fails or the same voltage swing fails 5 times consecutively):
     - Wait the DPCD clock-recovery interval (`100 us` for DP 1.4 / `TRAINING_AUX_RD_INTERVAL == 0`, or `4 ms * interval`).
     - Read 6 bytes of link status from DPCD `0x00202..0x00207`.
     - If all active lanes have `CR_DONE` set (`bit 0` for lane 0/2, `bit 4` for lane 1/3 in `0x202`/`0x203`), Phase 1 succeeds; proceed to Phase 2.
     - Otherwise, extract the sink's requested `voltage_swing` (`0..3`) and `pre_emphasis` (`0..3`) from DPCD `ADJUST_REQUEST_LANE0_1` (`0x00206`) and `ADJUST_REQUEST_LANE2_3` (`0x00207`), program the PHY TX registers with the new maximum requested level, and write the updated `TRAINING_LANE0_3_SET` to DPCD `0x00103..0x00106` (setting `MAX_SWING_REACHED` bit `2` if `swing == 3` and `MAX_PRE_EMPHASIS_REACHED` bit `5` if `pre_emphasis == 3`).
3. **Phase 2: Channel Equalization (`TPS4`, `TPS3`, or `TPS2`)**:
   - Select the highest supported training pattern:
     - If sink supports `TPS4` (DPCD `0x003` bit `7`): `state_ctrl = 0x08`, `dpcd_pattern = 0x07` (note: bit `5` `SCRAMBLING_DISABLE` is `0` for `TPS4`).
     - Else if sink supports `TPS3` (DPCD `0x002` bit `6`): `state_ctrl = 0x04`, `dpcd_pattern = 0x23`.
     - Else: `state_ctrl = 0x02`, `dpcd_pattern = 0x22`.
   - Write `0x00000000` then `state_ctrl` to `DP_STATE_CTRL` (`0x0404`).
   - Write `dpcd_pattern` to DPCD `TRAINING_PATTERN_SET` (`0x00102`).
   - Loop up to 5 iterations:
     - Wait the DPCD channel-equalization interval (`400 us` if `TRAINING_AUX_RD_INTERVAL == 0`, else `4 ms * interval`).
     - Read 6 bytes of link status from DPCD `0x00202..0x00207`.
     - If any active lane lost `CR_DONE`, abort Phase 2 and fail training at this rate.
     - If all active lanes have `CHANNEL_EQ_DONE` (`bit 1`/`5`), `SYMBOL_LOCKED` (`bit 2`/`6`), and `INTERLANE_ALIGN_DONE` (`DPCD 0x204` bit `0`), Phase 2 succeeds!
     - Otherwise, update PHY TX `voltage_swing` and `pre_emphasis` from DPCD `0x206..0x207` and write DPCD `0x103..0x106`.
4. **End Training**:
   - Write `0x00` (`TRAINING_PATTERN_DISABLE`) to DPCD `TRAINING_PATTERN_SET` (`0x00102`).

**[POLICY]** If Clock Recovery fails, reduce `link_rate` one step down (`HBR3 -> HBR2 -> HBR -> RBR`); if already at `RBR`, halve `lane_count` (`4 -> 2 -> 1`) and restore the maximum `link_rate`. If Channel Equalization fails, halve `lane_count` first or step down `link_rate`.

---

## 5. Main Stream Attributes (`MSA`), `DP_CONFIGURATION_CTRL`, and `MVID`/`NVID`

### 5.1 `DP_CONFIGURATION_CTRL` (`0x0408`) Bit Encoding

**[HW]** `DP_CONFIGURATION_CTRL` (`0x0408`) configures the main link framing and pixel format:
- Bit `0` `SYNC_ASYNC_CLK`: `1` = synchronous clock (`MVID`/`NVID` locked to pixel/link clock), `0` = asynchronous.
- Bit `1` `STATIC_DYNAMIC_CN`: `0` = static.
- Bits `[3:2]` `PIXEL_FORMAT`: `0` = RGB, `1` = YUV422, `2` = YUV444.
- Bits `[5:4]` `NUM_OF_LANES`: `0` = 1 lane, `1` = 2 lanes, `3` = 4 lanes (`(lane_count == 4) ? 3 : (lane_count - 1)`).
- Bit `6` `ENHANCED_FRAMING`: `1` = enhanced framing enabled, `0` = standard framing.
- Bit `7` `SEND_VSC`: `1` = send VSC SDP (required for YUV420).
- Bits `[9:8]` `BPC`: `0` = 6 bpc, `1` = 8 bpc (`24-bit RGB`), `2` = 10 bpc (`30-bit RGB`).
- Bit `10` `ASSR`: `1` = Alternate Scrambler Seed Reset (eDP).
- Bit `13` `LSCLK_DIV`: `1` = use `SOFTWARE_MVID` / `SOFTWARE_NVID` (`0x00002000` must be set).
- Example: 4 lanes, enhanced framing, 8 bpc RGB, synchronous clock, software MVID/NVID -> `0x00002171` (or `0x00000171` when hardware MVID counter is used with `NVID`).

### 5.2 `MVID` and `NVID` Calculation and Widebus/HBR2+ Scaling

**[HW]** The DisplayPort main link symbol clock `f_link` is `link_rate_khz` (`162000`, `270000`, `540000`, or `810000` kHz), while the controller's internal link clock `dptxN_link_clk` runs at `f_link / 10` (`16.2`, `27`, `54`, or `81 MHz` in older dividers, or `f_link / 5` on V5 PHYs) and the widebus pixel clock `dptxN_pixel0_clk` runs at `pclk_khz / 2`.
1. Given the RCG divider values `M` and `N` programmed into `DISP_CC_MDSS_DPTX*_PIXEL0_CMD_RCGR` (where the RCG input is the PHY VCO divided clock `f_vco_div`, and `f_pixel0 = f_vco_div * M / N`):
   - Start with base values `mvid = M` and `nvid = (~(N - M)) & 0xffff` (i.e. the raw 16-bit `M` and `N` rational fraction where `mvid / nvid = f_pixel0 / f_vco_div`).
   - Multiply `mvid` to normalize to a nominal `NVID` near `0x8000` (`32768`):
     - Calculate `mult = 0x8000 / nvid`, and set `mvid *= mult`, `nvid *= mult`.
2. **Mandatory Widebus and High-Bit-Rate Scaling** (required by the Qualcomm DP controller hardware):
   - Because widebus (`DATABUS_WIDEN` at `P0 + 0x1004` bit `0`) feeds **2 pixels per pixel-clock cycle**, `mvid` must be **doubled** (`mvid = mvid * 2`).
   - For link rates:
     - If `link_rate == 162000` (RBR) or `270000` (HBR): `nvid` is scaled by `1` (or `3` if the PHY VCO divider outputs `f_link * 5 / 3`).
     - If `link_rate == 540000` (HBR2): multiply `nvid` by `2` (`nvid *= 2`).
     - If `link_rate == 810000` (HBR3): multiply `nvid` by `3` (`nvid *= 3`).
   - equivalently, in pure frequency terms without RCG registers, `MVID / NVID = f_pixel_full_hz / f_link_symbol_hz`: choose `NVID = 0x8000` (`32768`, or a multiple of the link clock denominator such as `810000`), and compute `MVID = round(pclk_khz * NVID / link_rate_khz)`.
   - Write `mvid` to `DP_SOFTWARE_MVID` (`0x040c`) and `nvid` to `DP_SOFTWARE_NVID` (`0x0414`).

---

## 6. Transfer Unit (`TU`) Sizing and Valid Boundary Algorithm

**[HW]** The DisplayPort controller packs incoming active video pixels into fixed-size Transfer Units (`TU` of `32..64` link symbols per lane) interspersed with stuffing symbols. Incorrect `DP_TU` (`0x044c`), `DP_VALID_BOUNDARY` (`0x042c`), or `DP_VALID_BOUNDARY_2` (`0x0430`) values cause FIFO underflow/overflow (black screen or tearing).

### 6.1 Fast Analytical / Empirical TU Formula (No Boundary Moderation)

**[HW]** Each Transfer Unit of `tu_size` (`32..64`, e.g., `64` or `33..41`) link symbol cycles carries `tu_size * lanes` link bytes. Over the same time interval `(tu_size / f_link_symbol)`, the video stream produces `tu_size * (f_pixel * bpp / 8) / f_link_symbol` bytes, which is `valid_bytes_per_lane = tu_size * (f_pixel * bpp / (8 * lanes)) / f_link_symbol`.
1. Define the link utilization ratio:
   $$\text{ratio} = \frac{\text{pclk\_khz} \times \text{bpp}}{8 \times \text{lanes} \times \text{link\_rate\_khz}}$$
   where `bpp` is bits per pixel (`24` for 8 bpc RGB, `30` for 10 bpc RGB, `18` for 6 bpc RGB).
2. For a chosen `tu_size` in `[32, 64]` (selecting the `tu_size` in `32..64` whose product `tu_size * ratio` has the smallest non-zero fractional remainder right below an integer ceiling):
   - `valid_boundary_link = ceil(tu_size * ratio)`.
   - Write `DP_VALID_BOUNDARY` (`0x042c`) = `(tu_size << 16) | valid_boundary_link`.
   - Write `DP_TU` (`0x044c`) = `tu_size - 1`.
   - Write `DP_VALID_BOUNDARY_2` (`0x0430`) = `0x00000000` (when boundary moderation is disabled).
   - Write `DP_MAINLINK_LEVELS` (`0x043c`) = `0x00000002` (or `safe_to_exit_level` from Section 6.2).
   - *Empirical example (`1920x1080@60` at `pclk = 148500 kHz`, `24 bpp`, `4 lanes`, `HBR3 = 810000 kHz`)*: `ratio = (148500 * 24) / (8 * 4 * 810000) = 11 / 80 = 0.1375`. With `tu_size = 33`, `33 * 0.1375 = 4.5375 -> valid_boundary_link = 5`, `DP_VALID_BOUNDARY = (33 << 16) | 5 = 0x00210005`, `DP_TU = 32` (`0x20`).

### 6.2 Full Exact Fixed-Point TU & Boundary Moderation Algorithm

**[HW]** When `tu_size * ratio` does not land sufficiently close to an integer across a full scanline, the hardware supports **Boundary Moderation** (`DP_VALID_BOUNDARY_2` at `0x0430`), alternating between `upper_boundary_count` TUs of size `valid_boundary_link` and `lower_boundary_count` TUs of size `valid_lower_boundary_link = valid_boundary_link - 1`:
1. Use Q16.16 or Q32.32 fixed-point arithmetic (`SCALE = 65536`):
   - `lwidth = hdisplay` (or compressed slice byte width * 8 / bpp when DSC is enabled).
   - `hbp_relative_to_pclk = htotal - hdisplay`.
   - `ratio_fp = (pclk_khz * bpp * SCALE) / (8 * lanes * link_rate_khz)`.
   - `err_fp = 1000 * SCALE`.
2. Search candidate `tu_size` in `32..64`:
   - `new_valid_fp = tu_size * ratio_fp`.
   - `valid_boundary_link = ceil_fp(new_valid_fp)`.
   - `n_tus = floor((lwidth * bpp / 8) / (valid_boundary_link * lanes))`.
   - `even_distribution = ((n_tus % lanes) == 0) ? 1 : 0`.
   - Check if `n_tus * valid_boundary_link * (1 - ratio_fp / valid_boundary_link)` exceeds the available horizontal blanking symbols; if not, compute `temp_err_fp = (valid_boundary_link * SCALE) - new_valid_fp` and pick the `tu_size` minimizing `temp_err_fp` (preferring `even_distribution == 1`).
3. If the best `tu_size` still leaves a line-end accumulation error (`extra_buffer_margin`) that would overflow the controller's pixel-to-symbol FIFO before the end of `lwidth`, enable **Boundary Moderation**:
   - Search `tu_size` in `32..64`, `i_upper` in `1..15`, and `i_lower` in `1..15`:
   - `valid_boundary_link = ceil_fp(tu_size * ratio_fp)`.
   - `valid_lower_boundary_link = valid_boundary_link - 1`.
   - Effective average valid symbols per TU:
     $$\text{avg\_valid} = \frac{i_{\text{upper}} \times \text{valid\_boundary\_link} + i_{\text{lower}} \times \text{valid\_lower\_boundary\_link}}{i_{\text{upper}} + i_{\text{lower}}}$$
   - Select `(tu_size, i_upper, i_lower)` such that `avg_valid >= tu_size * ratio` and `avg_valid - (tu_size * ratio)` is minimized across all candidates.
   - Set `boundary_moderation_en = 1`, `upper_boundary_count = i_upper`, `lower_boundary_count = i_lower`.
4. Compute `safe_to_exit_level` (for `DP_MAINLINK_LEVELS` at `0x043c`):
   - `extra_bytes = ceil((n_tus + 1) * (valid_boundary_link - tu_size * ratio))`.
   - `safe_to_exit_level = clamp(ceil(extra_bytes * 8 / (lanes * bpp)) + 2, 2, 62)`.
5. Program the four TU registers:
   - `DP_VALID_BOUNDARY` (`0x042c`) = `((tu_size & 0x7f) << 16) | (valid_boundary_link & 0x7f)`.
   - `DP_TU` (`0x044c`) = `(tu_size - 1) & 0x3f`.
   - `DP_VALID_BOUNDARY_2` (`0x0430`) = `(boundary_moderation_en & 0x1) | ((upper_boundary_count & 0x7fff) << 1) | ((lower_boundary_count & 0x0f) << 16) | ((valid_lower_boundary_link & 0x3f) << 20)`.
   - `DP_MAINLINK_LEVELS` (`0x043c`): update bits `[5:0]` to `safe_to_exit_level` while preserving bits `[31:6]`.
   - *Example (`1920x1080@60` at `148500 kHz`, `24 bpp`, `4 lanes`, `810000 kHz`)*: `tu_size = 41`, `valid_boundary_link = 6`, `valid_lower_boundary_link = 5`, `upper_boundary_count = 13`, `lower_boundary_count = 5`, `boundary_moderation_en = 1` -> `DP_VALID_BOUNDARY = 0x00290006`, `DP_TU = 0x00000028`, `DP_VALID_BOUNDARY_2 = 0x0055001b`.

---

## 7. Stream Enable and Disable Sequence

### 7.1 Enabling Active Video Stream

**[HW]** Once Link Training (Section 4) has succeeded:
1. Program `DP_SOFTWARE_MVID` (`0x040c`), `DP_SOFTWARE_NVID` (`0x0414`), `DP_TOTAL_HOR_VER` (`0x0418`), `DP_START_HOR_VER_FROM_SYNC` (`0x041c`), `DP_HSYNC_VSYNC_WIDTH_POLARITY` (`0x0420`), `DP_ACTIVE_HOR_VER` (`0x0424`), `DP_MISC1_MISC0` (`0x0428`), `DP_CONFIGURATION_CTRL` (`0x0408`), and the TU registers (`0x042c`, `0x0430`, `0x043c`, `0x044c`).
2. Enable widebus in the DP pixel domain: write `0x00000001` (`DATABUS_WIDEN`) to `DP_INTF_CONFIG` (`DP_BASE + 0x1004`).
3. Command the DP link controller to send active video: write `0x00000000` then `0x00000080` (`SEND_VIDEO`) to `DP_STATE_CTRL` (`0x0404`).
4. Poll `DP_MAINLINK_READY` (`0x0438`) bit `0` (`READY_FOR_VIDEO`) or `DP_INTR_STATUS2` (`0x0024`) bit `0` until `READY_FOR_VIDEO` asserts.
5. Flush the DPU `CTL` (`CTL_PERIPH_FLUSH`, `CTL_INTF_FLUSH`, `CTL_FLUSH`) and enable the DPU timing engine (`INTF_TIMING_ENGINE_EN` = `1`).

### 7.2 Cleanly Disabling Active Video Stream (Mode-Change or DPMS Off)

**[HW]** Hardware requires the pixel clock and DPU timing engine to still be running when transitioning the DP controller from `SEND_VIDEO` to `PUSH_IDLE`:
1. Write `0x00000000` then `0x00000100` (`PUSH_IDLE`) to `DP_STATE_CTRL` (`0x0404`).
2. Poll `DP_MAINLINK_READY` (`0x0438`) bit `6` (`IDLE_PATTERNS_SENT`) or `DP_INTR_STATUS2` (`0x0024`) bit `3` (up to `20 ms`) until idle patterns have been transmitted.
3. Disable the DPU timing engine: write `0x00000000` to `INTF_TIMING_ENGINE_EN` (`INTF_BASE + 0x000`), flush `CTL`, and wait one vertical frame period (`~20 ms`) for the current scanout frame to drain.
4. Clear `DATABUS_WIDEN` in `DP_INTF_CONFIG` (`DP_BASE + 0x1004` = `0`).
5. Disable the DP main link: clear bit `0` (`MAINLINK_EN`) in `DP_MAINLINK_CTRL` (`0x0400`).
