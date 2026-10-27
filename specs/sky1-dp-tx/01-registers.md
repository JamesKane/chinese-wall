# 01: Register Map

All registers are 32-bit and accessed with 32-bit reads and writes. Offsets
are bytes from the start of the `dp` window (00 §0.2) unless a section says
otherwise. Field notation: `[m:n]` is a bit range, `[n]` a single bit.

Register names are the core's hardware mnemonics. "Used" says whether the
vendor driver programs the register; registers marked "not used" are listed
for completeness with no known behaviour.

## 1.1 Address space layout

| Range | Contents |
|---|---|
| `0x000–0x03F` | Link configuration (§1.2) |
| `0x080–0x0FF` | Core enables, misc control, identification (§1.3) |
| `0x100–0x15F` | AUX channel, HPD, interrupts, timers (§1.4) |
| `0x280` | PHY status (§1.6) |
| `0x400–0x49F` | HDCP (§1.11) |
| `0x500–0x52B` | MST control (§1.7) |
| `0x800 + 0x200·s` (s = 0 … 3) | Main-stream (video) block of source s (§1.5) |
| `0x900 + 0x200·s` | Secondary-channel (audio/SDP) block of source s (§1.8) |
| `0x980–0x9FF` | GTC, eDP CRC, PSR, direct-SDP registers of source 0 (§1.9); the CRC/PSR group repeats for source 1 at `0xB90–0xBBF` |

- **[HW]** Per-source registers repeat with a stride of **`0x200`**:
  source s's copy of a source-0 register at offset X is at X + `0x200`·s.
- **[HW]** Sky1 uses sources 0 and 1 (00 §0.5).

## 1.2 Link configuration

| Offset | Name | Used | Fields / values |
|---|---|---|---|
| `0x000` | `LINK_BW_SET` | yes | Link rate as the **DPCD bandwidth code**: `0x06` RBR, `0x0A` HBR, `0x14` HBR2, `0x1E` HBR3 |
| `0x004` | `LANE_COUNT_SET` | yes | Number of lanes: 1, 2 or 4 |
| `0x008` | `ENHANCED_FRAMING_ENABLE` | yes | `[0]` 1 = enhanced framing (set when the sink supports it) |
| `0x00C` | `TRAINING_PATTERN_SET` | yes | 0 = off (normal data), 1 = TPS1, 2 = TPS2, 3 = TPS3, **4 = TPS4** (the DPCD code for TPS4 is 7; the core uses 4) |
| `0x010` | `LINK_QUAL_PATTERN_SET` | not used | Compliance test patterns |
| `0x014` | `DISABLE_SCRAMBLING` | yes | `[0]` 1 = scrambler off |
| `0x018` | `DOWNSPREAD_CONTROL` | yes | `[0]` 1 = 0.5 % down-spread (SSC) on |
| `0x01C` | `EDP_CAPABILITY_CONFIG` | not used | |
| `0x020` | `HBR2_SCRAMBLER_RESET` | not used | |
| `0x024` | `DISPLAYPORT_VERSION` | not used | |
| `0x028` | `ALPM_POWER_SET` | not used | |
| `0x02C` | `LANE_REMAP` | not used | |
| `0x030–0x038` | `CUSTOM_80BIT_PATTERN` (3 words: bits 31:0, 63:32, 79:64) | not used | |

## 1.3 Core enables, misc and identification

| Offset | Name | Used | Fields / values |
|---|---|---|---|
| `0x080` | `TRANSMITTER_ENABLE` | yes | `[0]` 1 = transmitter enabled. Set at host initialisation, cleared at shutdown |
| `0x090` | `SOFT_RESET` | yes | Bit s per source. Written with the same source mask as `SOURCE_ENABLE` after link training, 0 at link shutdown |
| `0x094` | `SOURCE_ENABLE` | yes | Bit s = source s enabled. SST: `0x1`. MST: `(1 << streams) − 1` |
| `0x098` | `FEC_ENABLE` | not used | |
| `0x0C0` | `FORCE_SCRAMBLER_RESET` | yes | Bit s per source; written with the same mask as `SOURCE_ENABLE` |
| `0x0F8` | `CORE_FEATURES` | not used | RO capability word |
| `0x0FC` | `CORE_REVISION` | read only for logging | RO |

> OPEN QUESTION: The meaning of `SOFT_RESET` and `FORCE_SCRAMBLER_RESET`
> is not explained. The vendor driver leaves `SOFT_RESET` at the source
> mask during normal operation, which suggests the bits act as per-source
> "release" bits or are self-clearing pulses. Follow the vendor write
> sequence (02 §2.5.6).

## 1.4 AUX channel, HPD, interrupts and timers

| Offset | Name | Access | Fields / values |
|---|---|---|---|
| `0x100` | `AUX_COMMAND` | W | `[3:0]` byte count − 1; `[11:8]` AUX request command (§1.4.2); `[12]` address-only transaction (no data phase). Writing it starts the transaction |
| `0x104` | `AUX_WRITE_FIFO` | W | One data byte per write (`[7:0]`), up to 16, written **before** `AUX_COMMAND` |
| `0x108` | `AUX_ADDRESS` | RW | 20-bit AUX address (DPCD address, or I2C 7-bit slave address for I2C-over-AUX) |
| `0x10C` | `AUX_CLOCK_DIVIDER` | RW | `[8:0]` divider from `apb_clk` to the internal **1 MHz** AUX time base. Integer = APB frequency in MHz; valid range 10 … 400 |
| `0x110` | `AUX_REPLY_TIMEOUT_INTERVAL` | RW | Not programmed (reset value used) |
| `0x128` | `HPD_INPUT_STATE` | R | `[0]` 1 = HPD line is high (sink present) |
| `0x130` | `INTERRUPT_STATE` | R | `[0]` HPD state, `[1]` AUX request pending, `[2]` AUX reply received, `[3]` AUX reply timeout. Raw status; read by the HDCP code only |
| `0x134` | `AUX_REPLY_DATA` | R | One reply byte per read (`[7:0]`), in order |
| `0x138` | `AUX_REPLY_CODE` | R | Reply command field of the last reply (§1.4.3) |
| `0x13C` | `AUX_REPLY_COUNT` | R | Not used |
| `0x140` | `INTERRUPT_CAUSE` | R | Latched interrupt causes (§1.4.1) |
| `0x144` | `INTERRUPT_MASK` | RW | Same bit layout; **1 = masked** (disabled) |
| `0x148` | `AUX_REPLY_DATA_COUNT` | R | `[7:0]` number of data bytes in the last reply |
| `0x14C` | `AUX_STATUS` | R | `[0]` reply received; `[1]` reply in progress; `[2]` request in progress; `[3]` reply error |
| `0x150` | `AUX_REPLY_CLOCK_WIDTH` | R | Not used |
| `0x154` | `AUX_PHY_WAKE_ACK_DETECTED` | R | Not used |
| `0x158` | `HOST_TIMER` | RW | General-purpose down-counter (§1.4.4) |
| `0x15C` | `MST_TIMER` | RW | Not used |

### 1.4.1 Interrupt sources (`INTERRUPT_CAUSE` / `INTERRUPT_MASK`)

| Bit | Source |
|---|---|
| 0 | HPD event: the HPD level changed (plug or unplug) |
| 1 | HPD IRQ: a short HPD pulse from the sink (sink-requested attention) |
| 2 | AUX reply received |
| 3 | AUX reply timeout |
| 4 | General-purpose (host) timer expired |
| 5 | HDCP timer expired |

- **[POLICY]** *(vendor)* Operating mask `0x1C`: HPD event, HPD IRQ and
  HDCP timer enabled; AUX and host-timer interrupts masked (AUX is polled).
  At shutdown the mask is `0x3F` (everything masked).
- **[HW]** The interrupt handler treats the line as "not ours" when
  `INTERRUPT_CAUSE & ~INTERRUPT_MASK` is zero.

> OPEN QUESTION: The vendor driver never writes `INTERRUPT_CAUSE`, so it is
> presumably **clear-on-read**. This is inferred, not shown. Read it exactly
> once per interrupt.

### 1.4.2 AUX request command encoding (`AUX_COMMAND[11:8]`)

**[HW]** The field holds the DisplayPort AUX request command:

| Value | Request |
|---|---|
| `0x8` | Native write |
| `0x9` | Native read |
| `0x0` | I2C-over-AUX write |
| `0x1` | I2C-over-AUX read |
| `0x2` | I2C write-status-update |
| + `0x4` | I2C "middle of transaction" (MOT) flag, OR'd into an I2C request |

### 1.4.3 AUX reply code (`AUX_REPLY_CODE`)

**[HW]** The value is the 4-bit DisplayPort reply command:

| Bits | Meaning |
|---|---|
| `[1:0]` | Native reply: 0 ACK, 1 NACK, 2 DEFER |
| `[3:2]` | I2C reply: 0 ACK, 1 NACK, 2 DEFER |

A value of 0 is "ACK" for both native and I2C requests.

### 1.4.4 `HOST_TIMER`

| Bits | Field |
|---|---|
| `[19:0]` | Count (reads the current value; counts down to 0) |
| `[29]` | Interrupt enable (raises interrupt source 4) |
| `[30]` | Auto-reload |
| `[31]` | Enable |

- **[HW]** The timer ticks at the 1 MHz AUX time base, so the count is in
  microseconds (maximum ≈ 1.05 s).
- **[POLICY]** *(vendor)* Used only as a busy-wait delay by the USB-C PHY
  code (sky1-dp-tx 03). The HDCP block has its own timer (§1.11).

## 1.5 Main-stream block (source s at `0x800 + 0x200·s`)

Offsets below are relative to the source's block base.

| Offset | Name | Used | Fields / values |
|---|---|---|---|
| `0x00` | `VIDEO_STREAM_ENABLE` | yes | `[0]` 1 = transmit the main video stream of this source |
| `0x04` | `SECONDARY_STREAM_ENABLE` | yes | `[0]` 1 = transmit secondary data (SDPs, audio) of this source |
| `0x08` | `SECONDARY_DATA_WINDOW` | yes | Blanking-period capacity for secondary data, in link symbols (02 §2.6.3) |
| `0x0C` | `INPUT_STATUS` | not used | RO |
| `0x10` | `DATA_CONTROL` | yes | Opaque vendor value (02 §2.6.3) |
| `0x14` | `MAIN_STREAM_OVERRIDE` | yes | Pixel-format override when colorimetry is carried in a VSC SDP (§1.5.3) |
| `0x20` | `MAIN_STREAM_HTOTAL` | yes | Total pixels per line |
| `0x24` | `MAIN_STREAM_VTOTAL` | yes | Total lines per frame |
| `0x28` | `MAIN_STREAM_POLARITY` | yes | `[0]` HSYNC polarity, `[1]` VSYNC polarity (02 §2.6.2) |
| `0x2C` | `MAIN_STREAM_HSWIDTH` | yes | HSYNC width, pixels |
| `0x30` | `MAIN_STREAM_VSWIDTH` | yes | VSYNC width, lines |
| `0x34` | `MAIN_STREAM_HRES` | yes | Active pixels per line |
| `0x38` | `MAIN_STREAM_VRES` | yes | Active lines |
| `0x3C` | `MAIN_STREAM_HSTART` | yes | Pixels from the leading edge of HSYNC to the first active pixel (sync width + back porch) |
| `0x40` | `MAIN_STREAM_VSTART` | yes | Lines from the leading edge of VSYNC to the first active line |
| `0x44` | `MAIN_STREAM_MISC0` | yes | The DP MSA **MISC0** byte (§1.5.1) |
| `0x48` | `MAIN_STREAM_MISC1` | yes | The DP MSA **MISC1** byte (§1.5.2) |
| `0x4C` | `MVID` | sync mode only | M value for synchronous clock mode |
| `0x50` | `TU_CONFIG` | yes | Transfer-unit setup (§1.5.4) |
| `0x54` | `NVID` | sync mode only | N value for synchronous clock mode |
| `0x58` | `USER_PIXEL_COUNT` | yes | Pixels delivered per input clock by the DPU: 1 or 2 |
| `0x5C` | `USER_DATA_COUNT` | yes | Bytes of active video per line **per lane** (02 §2.6.3) |
| `0x60` | `MAIN_STREAM_INTERLACED` | not used | |
| `0x64` | `USER_SYNC_POLARITY` | yes | Polarity of the DPU-side input signals (§1.5.5) |
| `0x68` | `USER_CONTROL` | not used | |
| `0x6C` | `USER_FIFO_STATUS` | not used | RO |
| `0x70` | `FRAMING_STATUS` | not used | RO |

### 1.5.1 `MAIN_STREAM_MISC0`

**[HW]** Same layout as the MISC0 byte of the DisplayPort Main Stream
Attributes:

| Bits | Field | Values |
|---|---|---|
| `[0]` | Synchronous clock | 1 = link and stream clocks locked (MVID/NVID used). The vendor driver uses 0 (asynchronous) |
| `[2:1]` | Component format | 0 RGB, 1 YCbCr 4:2:2, 2 YCbCr 4:4:4 |
| `[3]` | Dynamic range | 1 = CTA (limited) range. The vendor driver sets it together with any YCbCr format |
| `[4]` | YCbCr colorimetry | 0 BT.601, 1 BT.709 |
| `[7:5]` | Bits per component | 0 = 6, 1 = 8, 2 = 10, 3 = 12, 4 = 16 |

*(vendor)* Register values used: RGB = `0x00`, YCbCr 4:2:2 = `0x0A`, YCbCr
4:4:4 = `0x0C` in `[3:0]`, plus the bpc code in `[7:5]`.

### 1.5.2 `MAIN_STREAM_MISC1`

| Bits | Field |
|---|---|
| `[6]` | 1 = colorimetry and pixel encoding are carried in a VSC SDP and the MISC0 format fields are to be ignored (DP 1.4 rule) |
| `[7]` | 1 = Y-only (monochrome) |

### 1.5.3 `MAIN_STREAM_OVERRIDE`

| Bits | Field |
|---|---|
| `[2:0]` | Bits per component: 0 = 6, 1 = 8, 2 = 10, 3 = 12, 4 = 16 |
| `[5:3]` | Pixel encoding override: 4 = YCbCr 4:2:0, 5 = DSC. 0 = none |
| `[6]` | 1 = override enabled |

- **[HW]** *(vendor)* Used only when MISC1 bit 6 is set (VSC SDP
  colorimetry). YCbCr 4:2:0 is only possible this way, because MISC0 has no
  4:2:0 code.

### 1.5.4 `TU_CONFIG`

| Bits | Field |
|---|---|
| low bits (at least `[6:0]`) | Transfer-unit size in symbols. The vendor driver always uses 64; the exact field width is not known |
| `[23:16]` | Integer part of the average number of valid data symbols per TU |
| `[27:24]` | Fractional part of the same, in 1/16 units. The core spreads the fraction over successive TUs |

### 1.5.5 `USER_SYNC_POLARITY`

| Bits | Field |
|---|---|
| `[0]` | HSYNC polarity of the input |
| `[1]` | VSYNC polarity of the input |
| `[3:2]` | Data-enable polarity; the vendor driver writes `0b11` (active high) |

- **[POLICY]** *(vendor)* Only source 0's copy is ever written, also in MST.

## 1.6 PHY status

| Offset | Name | Fields |
|---|---|---|
| `0x280` | `PHY_STATUS` | `[3:0]` bit i = lane i of the PHY is ready |

- **[POLICY]** *(vendor)* After setting the link rate the driver waits for
  bits `[N−1:0]` to be set, where N is the **maximum** lane count of the
  port (normally 4), not the trained lane count (02 §2.5.2).

## 1.7 MST control

| Offset | Name | Used | Fields |
|---|---|---|---|
| `0x500` | `MST_ENABLE` | yes | `[0]` 1 = MST framing (MTPs) |
| `0x504` | `MST_PID_TABLE_INDEX` | yes | Slot index (0 … 63) for the next access to `MST_PID_TABLE_ENTRY`; auto-increments after each access |
| `0x508` | `MST_PID_TABLE_ENTRY` | yes | Owner of the indexed time slot as a one-hot source mask (bit s = source s); 0 = unallocated |
| `0x50C` | `SST_SOURCE_SELECT` | not used | |
| `0x510` | `MST_ALLOCATION_TRIGGER` | yes | Write 1 to switch to the selected payload table at the next allocation-change point; reads 0 when done |
| `0x514` | `MST_PID_TABLE_SELECT` | yes | Which of the **two** payload tables (0 or 1) register accesses and the next trigger refer to |
| `0x518` | `MST_ACTIVE_PAYLOAD_TABLE` | yes | RO: payload table currently in use (0 or 1) |
| `0x520` | `MST_ACTIVE` | not used | |
| `0x524` | `MST_LINK_FRAME_COUNT` | not used | |
| `0x528` | `MSO_CONFIG` | not used | eDP multi-SST operation |

## 1.8 Secondary-channel block (source s at `0x900 + 0x200·s`)

Offsets below are relative to the block base.

| Offset | Name | Used | Fields / values |
|---|---|---|---|
| `0x00` | `AUDIO_ENABLE` | yes | `[0]` 1 = send audio packets |
| `0x04` | `INPUT_SELECT` | yes | Audio input selection; the vendor driver writes the source number (0) |
| `0x08` | `CHANNEL_COUNT` | yes | Number of audio channels (1 … 8), written as the plain count |
| `0x10` | `INFOFRAME_ENABLE` | yes | Bit b = SDP buffer b (0 … 6) is transmitted |
| `0x14` | `INFOFRAME_RATE` | yes | Bit b = buffer b is repeated every frame (vendor driver sets it for every buffer it writes) |
| `0x18` | `MAUD` | not used | Audio M value (synchronous audio clock only) |
| `0x1C` | `NAUD` | not used | Audio N value |
| `0x20` | `AUDIO_CLOCK_MODE` | not used | Reset value used |
| `0x24` | `3D_VSC_DATA` | not used | |
| `0x28–0x30` | `AUDIO_FIFO`, `AUDIO_FIFO_LAST`, `AUDIO_FIFO_READY` | not used | Register-fed audio path; Sky1 feeds audio over I2S instead |
| `0x34` | `INFOFRAME_SELECT` | yes | Buffer number (0 … 6) addressed by `INFOFRAME_DATA` |
| `0x38` | `INFOFRAME_DATA` | yes | Byte FIFO into the selected buffer: 4 header bytes then 32 data bytes, one byte (`[7:0]`) per write |
| `0x3C` | `TIMESTAMP_INTERVAL` | yes | Audio timestamp packet interval; the vendor driver writes `0x1F`, and only source 0's copy |
| `0x40` | `CS_SOURCE_FORMAT` | yes | IEC 60958 channel status: `[7:4]` source number, `[3]` 0 = linear PCM |
| `0x44` | `CS_CATEGORY_CODE` | yes | Channel-status category code (0 = general) |
| `0x48` | `CS_LENGTH_ORIG_FREQ` | yes | `[7:4]` word-length code, `[3:0]` original-sampling-frequency code (02 §2.8.2) |
| `0x4C` | `CS_FREQ_CLOCK_ACCURACY` | yes | `[7:4]` sampling-frequency code; low bits presumably clock accuracy (always written 0 = level II) |
| `0x50` | `CS_COPYRIGHT` | yes | 0 = copyright asserted bit clear |
| `0x54` | `AUDIO_CHANNEL_MAP` | yes | Eight 4-bit fields; the vendor driver writes `0x87654321` (identity mapping) |
| `0x58` | `AUDIO_FIFO_OVERFLOW` | not used | RO |
| `0x5C` | `PACKET_COUNT` | not used | RO |
| `0x60` | `CHANNEL_USER_DATA` | not used | |
| `0x64` | `DATA_PACKET_ID` | not used | |
| `0x68` | `ADAPTIVE_SYNC_ENABLE` | yes | `[0]` 1 = adaptive-sync (VRR) operation for this source |

- **[HW]** *(vendor)* The vendor driver only drives SDP buffers on sources
  0 and 1 and refuses buffer numbers ≥ 7.

> OPEN QUESTION: The register names for sources 1–3 list a
> `GTC_AUX_FRAME_SYNC` at block offset `0x5E`, which is not word-aligned and
> overlaps `PACKET_COUNT`. It is probably a header typo. Not used.

## 1.9 Source-0 extras (GTC, eDP CRC, PSR, direct SDP)

| Offset | Name | Used | Fields / values |
|---|---|---|---|
| `0x980` | `GTC_COUNT_CONFIG` | not used | Global time code |
| `0x984` | `GTC_COMMAND_EDGE` | not used | |
| `0x988` | `GTC_AUX_FRAME_SYNC` | not used | |
| `0x990` | `EDP_CRC_ENABLE` | yes (PSR) | `[0]` 1 = compute frame CRCs |
| `0x994–0x99C` | `EDP_CRC_RED/GREEN/BLUE` | not used | RO CRC results |
| `0x9A0` | `PSR_3D_ENABLE` | yes (PSR) | `[0]` 1 = PSR function enabled in the core |
| `0x9A4` | `PSR_CONFIG` | not used | |
| `0x9A8` | `PSR_STATE` | yes (PSR) | Requested PSR state: 0 = inactive / exit, 1 = enter PSR, 3 = single-frame update (02 §2.10) |
| `0x9AC` | `PSR_INTERNAL_STATE` | not used | RO |
| `0x9B0–0x9BC` | `PSR2_UPDATE_TOP/BOTTOM/LEFT/WIDTH` | not used | PSR2 selective-update region |
| `0x9E0–0x9F8` | `SDP_LANE_SELECT`, `SDP_BUFFER_INDEX`, `_DATA_COUNT`, `_DATA`, `_DONE`, `_BUSY`, `_ENABLE` | not used | Direct SDP interface |

Source 1 copies: `EDP_CRC_*` at `0xB90–0xB9C`, `PSR_*` at `0xBA0–0xBBC`
(not used).

> OPEN QUESTION: `PSR_STATE` value 3 ("single-frame update") appears only
> in a disabled code path. Values other than 0 and 1 are unverified.

## 1.10 RCSU video adapter (`dp_rcsu` window)

Offsets are relative to the `dp_rcsu` window (00 §0.2). The adapter sits
between the DPU pipeline outputs and the DPTX sources. Only the registers
below are known; everything is read-modify-write.

**Units 0, 1, 3, 4 (USB-C ports):**

| Offset | Name | Fields |
|---|---|---|
| `0x300` | `CFG_ADAPTER_VIDEO0` | Pipeline 0 input. `[29]` 1 = 2 pixels per clock; `[31:30]` pixel encoding: 0 RGB or YCbCr 4:4:4, 2 YCbCr 4:2:2, 3 YCbCr 4:2:0 |
| `0x304` | `CFG_ADAPTER_VIDEO1` | Pipeline 1 input, same layout |

**Unit 2 (`cfg_adapter_port = 2`):**

| Offset | Name | Fields |
|---|---|---|
| `0x320` | `CFG_ADAPTER_VIDEO` | Pipeline 0: `[16]` 2 pixels per clock, `[18:17]` encoding. Pipeline 1: `[19]` 2 pixels per clock, `[21:20]` encoding. Encoding codes as above |

- **[HW]** *(vendor)* The pipeline index used is that of the DPU pipeline
  that masters the CRTC (in side-by-side mode, the master pipeline).
- **[HW]** *(vendor)* Without this window the vendor driver refuses
  YCbCr 4:2:0 (it only advertises 4:2:0 when the window exists).

> OPEN QUESTION: The devicetree sizes the USB-C units' `dp_rcsu` window as
> `0x304` bytes, which ends just before `CFG_ADAPTER_VIDEO1` at `0x304`.
> Either the size is one word short or pipeline 1 is never configured on
> those units. Map at least `0x308` bytes. The rest of the RCSU (offsets
> below `0x300`) is undocumented.

## 1.11 HDCP block (summary only)

**[HW]** The HDCP 1.x/2.x cipher lives at `0x400–0x49C`:

| Offset | Registers |
|---|---|
| `0x400`, `0x404` | `HDCP_ENABLE`, `HDCP_MODE` |
| `0x408–0x414` | Session / master key words (`KS`, `KM`) |
| `0x418–0x41C` | `AN` / `RTX` (shared) |
| `0x424`, `0x428`, `0x42C` | `AUTH_IN_PROGRESS`, `R0_STATUS`, `CIPHER_CONTROL` |
| `0x430–0x43C` | `BKSV` / `RRX` (shared), `AKSV` |
| `0x440–0x44C` | `LC128` (4 words) |
| `0x450`, `0x454` | `REPEATER`, `STREAM_CIPHER_ENABLE` |
| `0x458–0x47C` | `M0`, AES input select, AES counter disable/advance/reset, `ECF`, `RN` |
| `0x480–0x488` | Random-number generator (`RNG_CIPHER_STORE_AN`, `RNG_CIPHER_AN`) |
| `0x48C` | `HDCP_HOST_TIMER` (same layout as §1.4.4, raises interrupt source 5) |
| `0x490` | `ENCRYPTION_STATUS` |
| `0x498–0x49C` | `CONTENT_TYPE_SELECT` (64 bits) |

The authentication protocol runs in a userspace daemon through a private
interface (05 §5.5). Its behaviour is out of scope.
