# 02: Operation

Register names refer to 01. "Source s" registers are at the per-source
offset (01 §1.1). DPCD addresses are those of the public DisplayPort
standard.

## 2.1 Power-up and host initialisation

**[HW]** Order of operations to make the core usable (AUX and HPD working,
link still down):

1. Make sure the DPU power domain of unit n is on (00 §0.4).
2. Enable `apb_clk`.
3. Pulse `dp_reset`: assert, wait ≥ 10 µs, deassert. On unit 2 also pulse
   `phy_reset` the same way.
4. Wait ≥ 100 µs.
5. Identify the PHY from `dp_phy + 0xFC` (00 §0.2) and, for a USB-C port,
   look up the Type-C PHY that controls lane muxing (sky1-dp-tx 03).
6. Write `TRANSMITTER_ENABLE` = 1.
7. Write `INTERRUPT_MASK` = `0x1C` (01 §1.4.1).
8. Write `AUX_CLOCK_DIVIDER` = `apb_clk` rate in MHz (integer, 10 … 400).
9. Run the PHY's "prepare" step (sky1-dp-tx 03/04).
10. Unit 2 with an eDP panel: run the panel power-on step and wait 50 ms
    (sky1-dp-tx 04).

- **[POLICY]** *(vendor)* The vendor driver does this at probe, again
  whenever the connector is probed while uninitialised, and again (steps
  3–9 only, followed by a 100–200 µs wait) before every new link bring-up
  after a previous one (§2.5). It skips it entirely when taking over a
  firmware-lit display (§2.12).
- **[HW]** Shutdown of the host side, in order: PHY "exit", panel power-off
  (unit 2), `TRANSMITTER_ENABLE` = 0, `INTERRUPT_MASK` = `0x3F`, disable
  `apb_clk`.
- **[POLICY]** *(vendor)* The interrupt is requested after the AUX channel
  and host initialisation are done. It is a level interrupt serviced in
  thread context.

## 2.2 HPD detection and interrupt handling

### 2.2.1 Interrupt handler

**[HW]** On an interrupt:

1. Read `INTERRUPT_CAUSE` once and `INTERRUPT_MASK`. If no unmasked cause
   is set, the interrupt is not from this core.
2. Bit 5 (HDCP timer): hand to the HDCP logic.
3. Bit 0 (HPD event): schedule the **plug/unplug handler** (§2.2.2). If
   `HPD_INPUT_STATE[0]` is 1 at this moment, delay it by
   `cix,delay_after_hpd` ms *(vendor, default 0)*.
4. Bit 1 (HPD IRQ, short pulse): schedule the **sink-attention handler**
   (§2.2.3).

- **[POLICY]** *(vendor)* Both handlers run in a freezable work queue so
  that they stop during system suspend.

### 2.2.2 Plug/unplug handler

**[HW]** The live HPD level is `HPD_INPUT_STATE[0]`. Because HPD events
can bounce, the handler always re-reads the level rather than trusting the
interrupt.

1. If HPD is high and the port uses a USB-C combo PHY, wait until the
   Type-C side has put the PHY into DisplayPort mode *(vendor: poll every
   100 ms, up to 100 times)*. Training before that fails.
2. Re-initialise the host side if it is not initialised (§2.1).
3. **HPD low** → disconnect:
   1. Tear the link down at once (§2.11.2), even if the display stack has
      not yet disabled the output. *(Community fix: a fast unplug/replug
      otherwise leaves stale link state and the next training fails.)*
   2. Forget the active-stream count (reset to 0).
   3. If MST was active, tear down the topology and leave MST mode.
   4. Report "disconnected" to the display stack, the audio jack and HDCP.
4. **HPD high** → connect:
   1. Read the sink capabilities (§2.4). If the read times out, retry up to
      5 times 100 ms apart *(community fix for sinks that power up
      slowly; the DP standard allows a sink up to 500 ms)*. If it still
      fails, treat the port as disconnected.
   2. Set up MST if allowed (§2.9).
   3. Report "connected". In MST mode the SST connector itself reports
      "disconnected" and the branch's ports appear instead.
5. **HPD high but already connected** *(community fix)*: read DPCD
   `0x200`–`0x207` (sink count, IRQ vector, lane status). If clock recovery
   or channel equalisation is lost on any active lane, perform a full
   disconnect (step 3) and then a connect (step 4). This recovers from an
   unplug/replug that was too fast to produce two HPD events.

### 2.2.3 Sink-attention handler (HPD IRQ)

**[HW]** A short HPD pulse means the sink wants service. If HPD is now
low, ignore it. Otherwise:

1. Read DPCD `0x200`–`0x207`.
2. If the link is enabled and clock recovery or channel equalisation is not
   OK on the active lanes, re-run link training (§2.5) without changing the
   stream setup.
3. If DPCD `0x201` bit 2 (`CP_IRQ`) is set, read the HDCP 2.2 `RxStatus`
   byte (DPCD `0x69493`) and pass it to the HDCP logic.
4. Pass the event to the CEC-over-AUX logic (§2.13.3).
5. In MST mode, service the sideband message queue through the
   event-status-indicator range starting at DPCD `0x2002`.

- **[POLICY]** *(vendor)* The `LINK_STATUS_UPDATED` bit is read but
  ignored; automated test requests are not handled.

## 2.3 AUX channel

### 2.3.1 One AUX transaction (≤ 16 bytes)

**[HW]**

1. Read `AUX_STATUS`. If bit 2 (request in progress) or bit 1 (reply in
   progress) is set, the channel is busy: fail with "busy".
2. Write the address to `AUX_ADDRESS`.
3. For a write, push each data byte into `AUX_WRITE_FIFO` (one write per
   byte, up to 16).
4. Write `AUX_COMMAND`: request code in `[11:8]` (01 §1.4.2); for a
   transaction with data, byte count − 1 in `[3:0]`; for a zero-length
   transaction (I2C address-only start/stop, or a probe), set `[12]`
   instead.
5. Poll `AUX_STATUS` about every 1 ms. Bit 0 = reply received → step 6.
   Bit 3 = reply error → fail with "timeout".
6. Read `AUX_REPLY_CODE` (01 §1.4.3) and return it to the caller.
7. For a read with an ACK reply, check that `AUX_REPLY_DATA_COUNT[7:0]`
   equals the requested length (fail with "I/O error" if not), then read
   that many bytes from `AUX_REPLY_DATA`, one per read.

- **[POLICY]** *(vendor)* The poll gives up after the third check
  (≈ 2–3 ms).

> OPEN QUESTION: The core has its own reply-timeout detection (interrupt
> source 3; `AUX_REPLY_TIMEOUT_INTERVAL` left at its reset value). That it
> enforces the DisplayPort 400 µs reply timeout and reports it through
> `AUX_STATUS[3]` is assumed, not shown.
- **[HW]** The AUX time base is 1 MHz derived from `apb_clk`
  (`AUX_CLOCK_DIVIDER`). A wrong divider gives a wrong AUX bit rate and
  every transaction fails.

### 2.3.2 Longer transfers, retries and error handling

- **[HW]** A request longer than 16 bytes is split into consecutive
  16-byte transactions with increasing addresses.
- **[HW]** Before each request the driver checks `HPD_INPUT_STATE[0]`. If
  HPD is low it returns at once with the bytes transferred so far (0),
  instead of waiting for timeouts.
- **[HW]** NACK and DEFER replies are not retried inside the transaction
  routine. They are returned to the caller, which applies the standard
  DisplayPort rules (retry DEFER, at least 7 times for native requests; for
  I2C, retry DEFER and honour partial-write status updates).
- **[POLICY]** *(vendor)* On a hardware error the vendor driver returns the
  number of bytes completed before the failing chunk. Module parameter
  `aux_timeout_ms` (default 50) is defined but effectively unused.

### 2.3.3 I2C-over-AUX (EDID, DDC/CI)

- **[HW]** Standard DisplayPort I2C-over-AUX: I2C read/write requests with
  the 7-bit slave address in `AUX_ADDRESS`, MOT set on every request
  except the last of a transfer, address-only requests to start/stop.
  EDID is read from slave `0x50` (segment pointer `0x30`).
- **[POLICY]** *(vendor)* EDID read retries: up to 50 more attempts 20 ms
  apart. Without an EDID the driver offers the mode implied by a
  downstream port, or else standard modes up to 1920×1080 with 1080p
  preferred.

## 2.4 Sink capability discovery

**[HW]** On connect the driver reads (all DPCD, public DP standard):

| DPCD | Use |
|---|---|
| `0x000`–`0x00F` (and the extended capability block at `0x2200` when flagged) | Revision, max link rate (`0x001`), max lane count (`0x002[4:0]`), enhanced framing (`0x002[7]`), TPS3 (`0x002[6]`), TPS4 (`0x003[7]`), no-AUX handshake / fast training (`0x003[6]`), 0.5 % down-spread (`0x003[0]`), "video timing without MSA" (`0x007[6]`), branch device (`0x005`) |
| `0x080`… | Downstream port information |
| `0x021` | MST capability (only if MST is allowed) |
| `0x070`–`0x071`, `0x02E`, `0x700`–`0x701`, `0x2009` | PSR support and capabilities, ALPM, eDP revision and power-control capability, sink resynchronisation latency (§2.10) |
| `0x2210` | `[3]` VSC SDP colorimetry supported |
| `0x2214` | Adaptive-sync SDP capability |
| `0x200` | Sink count |

- **[HW]** Usable maximum lanes = min(sink, `cix,dp-lane-number`); usable
  maximum rate = min(sink, `cix,dp-max-rate`).
- **[POLICY]** *(vendor)* The USB-C PHY may give DP only 2 lanes; the
  PHY reports this and the training code honours it (sky1-dp-tx 03).

## 2.5 Link configuration and training

### 2.5.1 Rate, lane count and colour depth selection

**[POLICY]** *(vendor, with community fix)* At mode-set time:

1. **Pixel format and bpc.** Try, in order, RGB 4:4:4, YCbCr 4:2:2, YCbCr
   4:2:0 (each only if the sink lists it; 4:2:0 also when the mode is
   4:2:0-only, and only if the RCSU window exists). For each format start
   at the sink's EDID bpc (forced to 10 for a BT.2020 colorspace), clamp to
   the user "max bpc" (8 … 10), and step down by 2 until the minimum (6 for
   RGB, 8 for YCbCr). Accept the first combination for which
   pixel clock ≤ max rate × max lanes × 8 / bpp. bpp = 3 × bpc (4:4:4),
   2 × bpc (4:2:2), 1.5 × bpc (4:2:0).
2. **Link rate (SST).** Walk RBR, HBR, HBR2, HBR3 (162000, 270000, 540000,
   810000 kHz link symbol clock) up to the maximum rate. Pick the first rate
   whose capacity with the maximum lane count leaves at least **5 %
   margin**: pixel clock × 100 ≤ 95 × (rate × lanes × 8 / bpp). If none
   does, use the maximum rate. *(The original vendor code used no margin;
   the 5 % rule is a community change made because zero margin failed in
   practice.)*
3. **Link rate (MST).** Always the maximum rate.
4. **Lane count.** At the chosen rate, take the smallest of 1, 2, 4 lanes
   (up to the maximum) with pixel clock ≤ rate × lanes × 8 / bpp (no
   margin here). MST always uses the maximum lane count.

- **[HW]** The "rate" in these formulas is the link symbol clock in kHz;
  rate × 8 is the payload bit rate per lane after 8b/10b in kbit/s.
- **[POLICY]** *(vendor)* Mode filtering: reject modes above 600 MHz pixel
  clock when 1 pixel per clock applies, modes at or above 1200 MHz, and
  modes whose clock exceeds max rate × max lanes × 8 / 18 (6 bpc RGB; 12
  for 4:2:0-only modes).

### 2.5.2 Link configuration

**[HW]** Before training:

1. Write `LANE_COUNT_SET` = lanes. If the sink supports enhanced framing,
   write `ENHANCED_FRAMING_ENABLE` = 1.
2. If the sink supports 0.5 % down-spread, write `DOWNSPREAD_CONTROL` = 1
   and DPCD `0x107` = `0x10`; otherwise write 0 to both.
3. Write DPCD `0x101` = lanes, with bit 7 set for enhanced framing.
4. Write DPCD `0x108` = `0x01` (8b/10b coding).
5. Write DPCD `0x100` = bandwidth code.
6. If variable refresh is enabled, write DPCD `0x107` bit 7 (ignore MSA
   timing parameters). *(Caution: the vendor driver writes this byte as
   the bit alone and so clears the down-spread bit of step 2.)*
7. Write `LINK_BW_SET` = bandwidth code.
8. Tell the PHY the new rate and lane count (sky1-dp-tx 03/04).
9. Poll `PHY_STATUS` every 1 ms, up to 100 ms, until the lane-ready bits
   of all the port's lanes (01 §1.6) are set. Fail if they never are.

### 2.5.3 Clock recovery (TPS1)

**[HW]** Standard DisplayPort clock-recovery phase:

1. Write `DISABLE_SCRAMBLING` = 1. Start with swing and pre-emphasis at
   level 0 on every lane.
2. Write `TRAINING_PATTERN_SET` = 1 and DPCD `0x102` = `0x21` (TPS1,
   scrambling disabled).
3. Repeat:
   1. Write the current per-lane drive settings to DPCD `0x103`… (one byte
      per active lane) and program the same swing and pre-emphasis into the
      PHY.
   2. Wait the clock-recovery interval the DP standard prescribes (it
      depends on the DPCD revision and DPCD `0x00E`).
   3. Read DPCD `0x202`–`0x207`. If `CR_DONE` is set on every active lane,
      clock recovery succeeded.
   4. Stop with failure if every lane already reports maximum swing, or if
      the same swing level has been requested 5 times in a row.
   5. Take the sink's adjust requests (DPCD `0x206`–`0x207`).
4. Bound the loop (the vendor driver allows 512 iterations).

- **[POLICY]** *(vendor)* Adjust requests are combined: the **highest**
  swing and the highest pre-emphasis requested by any lane are applied to
  **all** lanes, and the PHY receives lane 0's value for every lane. Swing
  level 3 is flagged "max swing reached"; pre-emphasis level ≥ 2 is flagged
  "max pre-emphasis reached".

> OPEN QUESTION: Flagging pre-emphasis level 2 as the maximum suggests the
> PHYs do not support pre-emphasis level 3, but the PHY code (sky1-dp-tx
> 03/04) is the authority. Whether per-lane drive settings work is untested
> by the vendor driver.

### 2.5.4 Channel equalisation (TPS2/3/4)

**[HW]**

1. Choose the highest pattern both sides support: TPS4 if DPCD `0x003[7]`,
   else TPS3 if DPCD `0x002[6]`, else TPS2.
2. TPS4: write `TRAINING_PATTERN_SET` = **4**, `DISABLE_SCRAMBLING` = 0
   (TPS4 is scrambled), and DPCD `0x102` = `0x07`. TPS2/TPS3: write
   `TRAINING_PATTERN_SET` = 2 or 3 and DPCD `0x102` = `0x22` or `0x23`.
3. Up to 5 times: write drive settings (as in §2.5.3), wait the
   equalisation interval (DPCD `0x00E`, per the DP standard), read DPCD `0x202`–`0x207`, and
   succeed if clock recovery, channel EQ, symbol lock and inter-lane align
   are all good; otherwise apply the new adjust requests.

### 2.5.5 End of training and fallback

**[HW]** Whether training succeeded or not, end it immediately:

1. Write DPCD `0x102` = 0.
2. Write `TRAINING_PATTERN_SET` = 0 and `DISABLE_SCRAMBLING` = 0.

There must be no delay between these writes and the start of normal data
(the sink expects idle patterns promptly after training).

**[POLICY]** *(vendor)* On failure, step the rate down (HBR3 → HBR2 → HBR
→ RBR), recompute the lane count for the new rate (§2.5.1 step 4), and
train again; up to 10 attempts per pass and 3 passes, restoring the
maximum rate between passes. If everything fails the driver **continues
anyway** and enables the stream; the sink then usually requests retraining
through HPD IRQ.

### 2.5.6 Enabling the sources

**[HW]** *(vendor sequence)* After training, with mask M = `0x1` (SST) or
`(1 << streams) − 1` (MST):

1. `SOFT_RESET` = M.
2. `SOURCE_ENABLE` = M.
3. `FORCE_SCRAMBLER_RESET` = M.
4. `MST_ENABLE` = 1 for MST, 0 for SST.

Before training, `VIDEO_STREAM_ENABLE` of every source is written 0.

### 2.5.7 Fast (AUX-less) retraining

**[POLICY]** *(vendor)* Only when a full training has already succeeded,
the sink supports the no-AUX handshake (DPCD `0x003[6]`) and
`cix,dp-fasttrain-default-on` is set:

1. Program `LANE_COUNT_SET`, `ENHANCED_FRAMING_ENABLE`, `LINK_BW_SET` and
   the PHY rate as in §2.5.2 (no DPCD writes).
2. `DISABLE_SCRAMBLING` = 1, `TRAINING_PATTERN_SET` = 1, wait 0.5–1 ms.
3. `TRAINING_PATTERN_SET` = channel-EQ pattern (§2.5.4), program the last
   successful drive settings into the PHY, wait 0.5–1 ms.
4. End training as in §2.5.5 without the DPCD write.

Full training is used if this fails.

### 2.5.8 Sink power state

- **[HW]** Before training, set the sink to D0: read DPCD `0x600`, set
  `[2:0]` = 1, write it (vendor: up to 3 attempts, 300–500 µs apart), then
  wait for the sink to wake *(vendor: 4 ms; 1 ms when PSR is in use)*.
- **[HW]** When shutting the link down with HPD still high, write `[2:0]`
  = 2 (D3) and wait 200 µs, unless `support_d3_cmd = "no"`.

### 2.5.9 Link bring-up order (summary)

**[HW]** Complete order for one SST link:

1. Host initialised (§2.1), HPD high, sink capabilities read (§2.4).
2. All `VIDEO_STREAM_ENABLE` = 0.
3. PHY prepare and power-on with the 24 MHz reference (sky1-dp-tx 03/04).
4. Sink to D0 (§2.5.8).
5. Train (§2.5.2–2.5.5), or fast-train (§2.5.7).
6. Enable the sources (§2.5.6).
7. Unit 2 with a panel: enable the panel/backlight (sky1-dp-tx 04).
8. Set up and start the stream (§2.6).

## 2.6 Video stream setup

### 2.6.1 Mode adjustments required by the core

- **[HW]** *(vendor)* The core needs a horizontal back porch (htotal −
  hsync_end) of **at least 12 pixels**. If a mode has less, the vendor
  driver widens htotal by the shortfall and raises the pixel clock in
  proportion, so the refresh rate is kept. The adjusted timing must be
  given to the DPU as well.
- **[POLICY]** *(vendor)* The vendor driver **inverts both sync polarities**
  of every mode before programming the DPU and the DPTX (a positive-sync
  mode is programmed as negative-sync, and vice versa).

> OPEN QUESTION: Whether the polarity inversion compensates for an
> inversion in the DPU → DPTX path, or for an inverted meaning of
> `MAIN_STREAM_POLARITY`, is not stated. Both the DPU timing and the DPTX
> registers receive the inverted flags, so the MSA sent to the sink carries
> inverted polarity bits. Test with a sink that honours MSA polarity.

- **[POLICY]** *(vendor)* For an eDP panel the panel's preferred mode is
  always used, whatever mode was requested.

### 2.6.2 Timing registers (source s)

**[HW]** From the adjusted mode (all values in pixels or lines):

| Register | Value |
|---|---|
| `MAIN_STREAM_HTOTAL` | htotal |
| `MAIN_STREAM_VTOTAL` | vtotal |
| `MAIN_STREAM_HSWIDTH` | hsync_end − hsync_start |
| `MAIN_STREAM_VSWIDTH` | vsync_end − vsync_start |
| `MAIN_STREAM_HRES` | hdisplay |
| `MAIN_STREAM_VRES` | vdisplay |
| `MAIN_STREAM_HSTART` | htotal − hsync_start |
| `MAIN_STREAM_VSTART` | vtotal − vsync_start |
| `MAIN_STREAM_POLARITY` | `[0]` = 1 if HSYNC positive, `[1]` = 1 if VSYNC positive (after §2.6.1 inversion) |
| `USER_SYNC_POLARITY` (source 0 only) | `0xC` OR the same two polarity bits |
| `USER_PIXEL_COUNT` | 1 or 2 pixels per clock (§2.6.5) |

### 2.6.3 Transfer unit, data count, secondary window

**[HW]** With P = pixel clock (kHz), B = bits per pixel, R = link symbol
clock (kHz), L = lane count:

1. `TU_CONFIG`: write 64 first. Then compute the average number of valid
   symbols per 64-symbol TU, A = (P × B / 8) × 64 / (L × R). Write
   64 | (integer part of A) << 16 | (fractional part of A in 1/16) << 24.
2. `USER_DATA_COUNT` = ceil( ceil(hdisplay × B / 8) / L′ ), where L′ = L
   in SST and **4 in MST** (MST maps stream data like a 4-lane link).
3. `SECONDARY_DATA_WINDOW` = (htotal − hdisplay) × (R / 40) × 9 / P,
   integer arithmetic in that order.
4. `DATA_CONTROL`: see the table below.

| Condition | `DATA_CONTROL` |
|---|---|
| SST (default) | `0x2004` |
| MST, hdisplay = 2560 or 640 | `0x201F` |
| MST, hdisplay = 1920 | `0x2008` |
| MST, hdisplay = 1280 | `0x200D` |
| MST, other widths | `0x2010` |
| 2 pixels per clock and 4 lanes at HBR3 (overrides the above) | `0x0004` |

> OPEN QUESTION: `DATA_CONTROL` fields are undocumented. The MST values
> are described in the vendor source as values found by experiment. The
> low bits may be an input-FIFO threshold.

> OPEN QUESTION: `MVID`/`NVID` are only written in synchronous-clock mode,
> which the vendor driver never uses. In asynchronous mode the core is
> assumed to measure the stream clock itself.

### 2.6.4 Pixel format and bit depth (MISC0/MISC1/override)

**[HW]** Two ways to signal the format:

1. **MSA only** (RGB, YCbCr 4:4:4, YCbCr 4:2:2, Y-only): write MISC0 with
   the format and bpc (01 §1.5.1), MISC1 bit 6 = 0 (bit 7 = 1 for Y-only),
   `MAIN_STREAM_OVERRIDE` = 0.
2. **VSC SDP colorimetry** (used when the sink sets DPCD `0x2210[3]` and
   the format is YCbCr 4:2:0 or the colorspace is sYCC 601, opYCC 601, or
   any BT.2020 variant): MISC1 bit 6 = 1, `MAIN_STREAM_OVERRIDE` = enable
   (bit 6) | bpc code | (4 << 3 for 4:2:0), and send a VSC SDP (§2.7.1).
   MISC0 is written but its format fields are ignored by the sink.

- **[HW]** Synchronous-clock bit (MISC0 `[0]`) = 0.
- **[POLICY]** *(vendor)* Unsupported bpc values fall back to 8.

### 2.6.5 Pixels per clock and the RCSU adapter

**[HW]** At mode-set, before the stream starts:

1. Pixels per clock = 2 if 600 MHz < pixel clock < 1200 MHz in SST and the
   format is not YCbCr 4:2:0; else 1. (Vendor property `pixel_per_cycle`
   overrides.) The DPU must be configured identically (DPU spec 08 §8.6).
2. Program the RCSU adapter of the master pipeline (01 §1.10): encoding
   field = 0 (RGB/4:4:4), 2 (4:2:2) or 3 (4:2:0); 2-pixel bit = 1 for 2
   pixels per clock.
3. Write `USER_PIXEL_COUNT` = 1 or 2.

- **[POLICY]** *(vendor)* The RCSU adapter is not programmed for MST
  streams, so MST is limited to RGB/4:4:4 at 1 pixel per clock.
- **[POLICY]** *(vendor)* PSR is turned off while 2 pixels per clock is in
  use.

### 2.6.6 Stream start and stop

**[HW]** Start stream s (link already up):

1. Enable `vid_clk s`.
2. Write MISC0/MISC1/override (§2.6.4).
3. Write any SDPs (§2.7).
4. If variable refresh: `ADAPTIVE_SYNC_ENABLE` = 1.
5. Write `TU_CONFIG` and the timing registers (§2.6.2–2.6.3).
6. MST only: update the payload table (§2.9).
7. Arm PSR in the sink if used (§2.10).
8. `VIDEO_STREAM_ENABLE` = 1, then `SECONDARY_STREAM_ENABLE` = 1.
9. If audio was playing before (resume or replug), reprogram and re-enable
   audio (§2.8).

**[HW]** Stop stream s:

1. Unit 2: disable the panel/backlight.
2. `INFOFRAME_ENABLE`, `INFOFRAME_SELECT`, `INFOFRAME_RATE` = 0.
3. `ADAPTIVE_SYNC_ENABLE` = 0.
4. Leave PSR (§2.10.3) and disarm it in the sink.
5. `SECONDARY_STREAM_ENABLE` = 0, then `VIDEO_STREAM_ENABLE` = 0.
6. Disable `vid_clk s`.

When the last stream is stopped, shut the link down (§2.11.2).

## 2.7 Secondary data packets (SDPs)

**[HW]** Each source has 7 SDP buffers (0 … 6). To load buffer b:

1. Clear bit b of `INFOFRAME_ENABLE`.
2. Write b to `INFOFRAME_SELECT`.
3. Write 36 bytes to `INFOFRAME_DATA`, one per write: header bytes HB0–HB3,
   then data bytes DB0–DB31 (unused bytes 0).
4. Set bit b of `INFOFRAME_RATE` (send every frame).
5. Set bit b of `INFOFRAME_ENABLE`.

**[POLICY]** *(vendor)* Buffer assignment:

| Buffer | Content |
|---|---|
| 0 | AVI InfoFrame (code present but disabled) |
| 1 | VSC SDP (colorimetry) |
| 2 | Audio InfoFrame |
| 3 | HDR static metadata (Dynamic Range and Mastering InfoFrame) |

### 2.7.1 Packet formats

**[HW]** Standard DP 1.4 formats:

| Packet | HB0 | HB1 | HB2 | HB3 | Payload |
|---|---|---|---|---|---|
| VSC SDP, revision 5 (colorimetry) | `0x00` | `0x07` | `0x05` | `0x13` | DB16 `[7:4]` pixel encoding, `[3:0]` colorimetry; DB17 `[2:0]` bpc code (1 = 8, 2 = 10, 3 = 12, 4 = 16), `[7]` CTA range; DB18 `[2:0]` content type |
| InfoFrame SDP (HDR, AVI) | `0x00` | `0x80` + CTA InfoFrame type (`0x87` HDR, `0x82` AVI) | `0x1D` | `0x4C` (version `0x13` << 2) | DB0 = InfoFrame version, DB1 = InfoFrame length, then the CTA-861 payload |
| Audio InfoFrame | standard DP audio InfoFrame SDP | | | | CTA-861 audio InfoFrame (channel count, speaker allocation) |

- **[POLICY]** *(vendor)* In the VSC SDP the vendor driver writes pixel
  encoding 1 for **both** YCbCr 4:4:4 and 4:2:2 (the DP standard uses 2 for
  4:2:2) and 3 for 4:2:0. A clean-room driver should follow the standard.
- **[POLICY]** *(vendor)* HDR metadata from userspace is written to buffer
  3 when it changes; a colorspace change forces a full mode-set.

## 2.8 Audio

### 2.8.1 Path

- **[HW]** Audio enters each DPTX over **I2S** from a dedicated SoC I2S
  controller: I2S5 → DP 0, I2S6 → DP 1, I2S7 → DP 2, I2S8 → DP 3, I2S9 →
  DP 4 (ACPI devices `CIXH6011` instances 2–6). The DMA and I2S controller
  are outside this spec.
- **[HW]** I2S format: standard I2S framing, **bit clock inverted**, frame
  clock normal; the I2S controller is the clock master. Up to 8 channels,
  playback only.
- **[HW]** *(vendor)* No `MAUD`/`NAUD` are programmed; the core is assumed
  to derive the audio timing from the I2S input itself.

### 2.8.2 Programming (source 0)

**[HW]** On stream parameters (rate, width, channels):

1. `CS_SOURCE_FORMAT` = source number << 4 (linear PCM).
2. `CS_CATEGORY_CODE` = 0.
3. `CS_LENGTH_ORIG_FREQ` = word-length code << 4 | original-frequency
   code; `CS_FREQ_CLOCK_ACCURACY` = sampling-frequency code << 4. Codes:

| Sample rate | Sampling-frequency code | Original-frequency code |
|---|---|---|
| 32 kHz | `0xC` | `0x3` |
| 44.1 kHz | `0x0` | `0xF` |
| 48 kHz (also the default) | `0x4` | `0xB` |
| 88.2 kHz | `0x1` | `0xE` |
| 96 kHz | `0x5` | `0xA` |
| 176.4 kHz | `0x3` | `0xC` |
| 192 kHz | `0x7` | `0x8` |

| Sample width | Word-length code |
|---|---|
| 16 bit | `0x4` |
| 18 bit | `0x2` |
| 20 bit | `0xC` |
| 24 bit (also the default) | `0xD` |

4. `CS_COPYRIGHT` = 0; `TIMESTAMP_INTERVAL` (source 0) = `0x1F`;
   `AUDIO_CHANNEL_MAP` = `0x87654321`; `CHANNEL_COUNT` = channels;
   `INPUT_SELECT` = 0.
5. Send an audio InfoFrame (buffer 2) with the channel count and CTA
   speaker allocation.
6. On playback start: `AUDIO_ENABLE` = 1. On stop: `AUDIO_ENABLE` = 0.

- **[POLICY]** *(vendor, community fixes)* Audio is only configured while
  a sink is plugged. On unplug or link shutdown, `AUDIO_ENABLE` = 0 is
  written but the "playing" state is kept, so steps 1–6 are redone when
  the stream is restarted. Audio is always on source 0 (no MST audio).

## 2.9 MST basics

**[POLICY]** *(vendor)* MST is off unless `cix,dp-mst-default-on` is set
and the sink reports MST capability. At most 2 streams (sources 0, 1).

**[HW]** Sink side (public DP MST protocol):

1. On connect, write DPCD `0x111` (`MSTM_CTRL`) = 0 and wait for it to read
   back 0 *(vendor: up to 10 × 50 ms)*.
2. If MST is to be used, write `0x111` = `0x07` (MST enable, up-request
   enable, upstream is a source).
3. Topology discovery, payload allocation in the branch (DPCD `0x1C0`…)
   and the ACT handshake use sideband messages over AUX, serviced from the
   HPD IRQ handler (§2.2.3).

**[HW]** Source side, after each allocation change (slot 0 holds the MTP
header and is never allocated):

1. Read `MST_ACTIVE_PAYLOAD_TABLE`; use the **other** table T.
2. `MST_PID_TABLE_SELECT` = T.
3. `MST_PID_TABLE_INDEX` = 0, then write `MST_PID_TABLE_ENTRY` = 0
   64 times (clear).
4. For each stream s with slots (start ≥ 1, start + count ≤ 64):
   `MST_PID_TABLE_INDEX` = start, then write `1 << s` to
   `MST_PID_TABLE_ENTRY` once per slot.
5. `MST_PID_TABLE_SELECT` = T, `MST_ALLOCATION_TRIGGER` = 1.
6. Wait at least one frame *(vendor: 6–8 ms)*; `MST_ALLOCATION_TRIGGER`
   reads 0 on success.

- **[HW]** Link shutdown clears the payload table (64 zero writes from
  index 0).
- **[HW]** In MST mode the link always runs at the maximum rate and lane
  count, `USER_DATA_COUNT` uses 4 lanes (§2.6.3), and stream s comes from
  DPU pipeline s with `vid_clk s`.

## 2.10 Panel Self Refresh (eDP, source 0)

### 2.10.1 Capability and arming

**[HW]** PSR is possible when DPCD `0x070` is non-zero and the eDP DPCD
`0x701` bit 7 (power-state control) is set. PSR2 is detected (`0x070` = 3,
Y-coordinate required and ALPM in DPCD `0x02E[0]`) but **not used**.

**[POLICY]** *(vendor, community fix)* PSR is used only on source 0, only
when `cix,dp-psr-default-on` is not 0, not with variable refresh, and not
at 2 pixels per clock. If the sink lacks the no-AUX handshake (DPCD
`0x003[6]`), the main link is kept active during PSR.

Arming (at stream start):

1. DPCD `0x170` = 0, then `0x170` = `0x01` (PSR enable), plus `0x02`
   (main link active) when the link is kept active.
2. `EDP_CRC_ENABLE` = 1, `PSR_3D_ENABLE` = 1, wait 100–200 µs.

Disarming: `PSR_3D_ENABLE` = 0, `EDP_CRC_ENABLE` = 0, DPCD `0x170` = 0.

### 2.10.2 Entry

**[HW]**

1. Read DPCD `0x2008` (PSR status, `[2:0]`). If it already reads 2 (active,
   displaying from the sink's frame buffer), abort.
2. `PSR_STATE` = 1.
3. Poll DPCD `0x2008` every 1 ms, up to 150 ms, until it reads 2.
4. If the link is not kept active: put the PHY into power state A3 and
   write `VIDEO_STREAM_ENABLE` (source 0) = 0.

- **[POLICY]** *(vendor)* On any failure write `PSR_STATE` = 0 and turn
  PSR off for the rest of the session.

### 2.10.3 Exit

**[HW]**

1. If the link was powered down: PHY to A2, then A0; sink to D0
   (§2.5.8); then `DISABLE_SCRAMBLING` = 1, `TRAINING_PATTERN_SET` = 1 for
   500–600 µs, then `TRAINING_PATTERN_SET` = 0 and `DISABLE_SCRAMBLING` = 0
   (lets the sink's receiver re-lock without AUX training).
2. Read DPCD `0x2008`. If it reads 0 (inactive): `VIDEO_STREAM_ENABLE` =
   1, `PSR_STATE` = 0, and (vendor) turn PSR off.
3. Otherwise `VIDEO_STREAM_ENABLE` = 1, wait 50–60 µs, `PSR_STATE` = 0,
   and poll DPCD `0x2008` (1 ms, up to 150 ms) for 0 (inactive) or 4
   (resynchronising).

## 2.11 Suspend, resume and teardown

### 2.11.1 System suspend/resume

**[POLICY]** *(vendor, community fixes)*

- Prepare: stop MST, cancel pending HPD work (without holding the driver
  lock), disable the interrupt, mark suspended. If no stream is active,
  shut the host side down (§2.1). Mark the connector "unknown" and forget
  the connected state.
- Early resume: in the display state saved for restore, detach every DP
  connector from its CRTC and mark it disconnected, so that the restore
  does not try to re-light a link that may have changed.
- Complete: re-initialise the host side (§2.1), enable the interrupt,
  resume MST, clear the suspended flag. The display stack then sees HPD
  and re-probes from scratch.
- Shutdown (power-off/reboot): no special action.

**[HW]** Register state is assumed lost across suspend; everything is
reprogrammed from §2.1.

### 2.11.2 Link shutdown

**[HW]** Order:

1. Sink to D3 if allowed (§2.5.8).
2. PHY to A3, then PHY "exit".
3. `AUDIO_ENABLE` = 0.
4. `MST_ENABLE` = 0, `SOURCE_ENABLE` = 0, `SOFT_RESET` = 0,
   `FORCE_SCRAMBLER_RESET` = 0.
5. Clear the MST payload table.

**[POLICY]** *(vendor)* The next bring-up performs a host re-initialisation
with reset (§2.1) first.

## 2.12 Firmware (GOP) handover

The UEFI firmware can leave a DP link trained and streaming (with the DPU
showing a splash, DPU spec 08 §8.5).

- **[HW]** Inputs: the `enabled_by_gop` property (0/1) of the DP node, and
  the SiP SMC `0xC200000F`, argument 1 = 1 ("get"). Bit 17 (`0x00020000`)
  of the result means "an OS DP driver has already run since boot"; then
  `enabled_by_gop` is ignored.
- **[POLICY]** *(vendor)* When the DP driver is removed, it calls the same
  SMC with argument 1 = 2 ("set") and argument 2 = `0x00020000`, so that a
  driver loaded later knows the firmware state is gone. (This is the "set"
  the DPU driver never calls.)

**[POLICY]** *(vendor)* Takeover when `enabled_by_gop` holds:

1. At probe: read the sink capabilities. If the sink is an MST branch,
   drop the takeover and initialise normally. Otherwise **skip** the reset
   and host initialisation (§2.1), mark the host initialised, and run the
   plug handler.
2. First link bring-up: skip PHY setup and training (the link is up).
3. First mode-set: read source 0's `HTOTAL`, `VTOTAL`, `HSWIDTH`,
   `VSWIDTH`, `HRES`, `VRES`, `HSTART`, `VSTART`, and recompute
   `SECONDARY_DATA_WINDOW` from `LINK_BW_SET`. If all match the requested
   mode and variable refresh is off: write `INTERRUPT_MASK` = `0x1C`,
   enable `vid_clk0`, write `VIDEO_STREAM_ENABLE` = 1 and
   `SECONDARY_STREAM_ENABLE` = 1, and keep everything else. If anything
   differs, do a full host initialisation with reset and a normal
   bring-up.
4. Afterwards behave normally.

> OPEN QUESTION: How the firmware programs `INTERRUPT_MASK`, the AUX
> divider and the PHY is not known; the vendor driver only rewrites the
> interrupt mask. A clean-room driver may simply always reset and retrain,
> at the cost of a visible blank.

## 2.13 Other features

### 2.13.1 DSC

**[HW]** Not used. The `dsc` window is not mapped, and
`MAIN_STREAM_OVERRIDE` has a DSC encoding code (5) that is never written.

### 2.13.2 Variable refresh (adaptive sync)

**[POLICY]** *(vendor)* Offered when the sink supports timing without MSA
(DPCD `0x007[6]`) and the EDID range spans more than 10 Hz. When userspace
enables it: DPCD `0x107` bit 7 (§2.5.2 step 6) and
`ADAPTIVE_SYNC_ENABLE` = 1. The vertical front porch is stretched by the
DPU (the vendor's in-kernel stretching is disabled). PSR is not used with
VRR.

### 2.13.3 CEC over AUX

**[HW]** HDMI-CEC tunnelling through a DP-to-HDMI converter uses the
standard DPCD CEC tunnelling registers (`0x3000`…) over AUX; no DPTX
registers are involved. *(Community addition.)*

### 2.13.4 HDCP

**[HW]** Summary only: HDCP 1.x/2.x cipher registers at `0x400–0x49C`
(01 §1.11), the HDCP timer interrupt (source 5) and the `CP_IRQ`
notification (§2.2.3). The authentication protocol runs in userspace
(05 §5.5).

## 2.14 Bring-up checklist (SST, USB-C port)

1. DPU n power domain on; `apb_clk` on; pulse `dp_reset`.
2. `TRANSMITTER_ENABLE` = 1, `INTERRUPT_MASK` = `0x1C`, AUX divider.
3. Wait for HPD (`HPD_INPUT_STATE[0]`) and for the Type-C side to select
   DP mode on the combo PHY.
4. Read DPCD `0x000`–`0x00F` over AUX. Read the EDID over I2C-over-AUX.
5. Pick a mode ≤ 600 MHz, RGB 8 bpc. Pick rate and lanes (§2.5.1).
6. Program the DPU pipeline 0 timing (with htotal adjusted for ≥ 12 pixel
   back porch), RCSU adapter = RGB, 1 pixel per clock.
7. PHY on (24 MHz reference), sink D0, train, enable sources (§2.5.9).
8. Program source 0 (§2.6), enable `vid_clk0`, enable the stream.
9. If the sink shows nothing, check `PHY_STATUS`, DPCD `0x202`–`0x207` and
   try the polarity setting of §2.6.1 both ways.
