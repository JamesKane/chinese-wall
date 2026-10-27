# 03: I2S Controllers, Audio DMA, Sound-Card Topology and DSP

All facts in this section come from the CIX vendor drivers and devicetree
and from community ACPI fixes; mainline Linux has no driver for the I2S
controllers or the DSP at the baseline. Statements are *(vendor)* unless
marked otherwise.

## 3.1 I2S controllers

### 3.1.1 Instances

| I2S | Base | SPI | Type | Data lines (`pin-out-num`) | Orion O6 use | CRU clocks (APB, core) | CRU reset |
|---|---|---|---|---|---|---|---|
| 0 | `0x07020000` | 219 | single-channel | 1 | analog codec (shares the HDA pads) | 16, 26 | 0 |
| 1 | `0x07030000` | 220 | single-channel | 1 | Bluetooth / modem PCM | 17, 27 | 1 |
| 2 | `0x07040000` | 221 | single-channel | 1 | internal loopback capture | 18, 28 | 2 |
| 3 | `0x07050000` | 222 | multi-channel | 6 | speaker amplifier (lines 2–5 transmit) | 19, 29 | 3 |
| 4 | `0x07060000` | 223 | multi-channel | 4 | DP-in capture (lines 0–3 receive) | 20, 30 | 4 |
| 5 … 9 | `0x07070000` … `0x070B0000` | 224 … 228 | multi-channel | 4 | **DP0 … DP4 audio out** (lines 0–3 transmit) | 21 … 25, 31 … 35 | 5 … 9 |

- **[HW]** I2S5 … I2S9 are wired inside the SoC to the audio input of DP
  transmitter 0 … 4. They have no pads.
- **[HW]** Under ACPI only the multi-channel instances exist, as `CIXH6011`
  (00 §0.6). On Orion O6 only I2S5, 6, 8 and 9 are enabled in the vendor
  DT (DP2 is the eDP port and is disabled there).
- The single-channel IP (I2S0–2) has a different register layout. It is
  not needed for DP audio under ACPI and is out of scope here.

> OPEN QUESTION: The single-channel register map is available in the vendor
> driver if a later spec needs analog-codec or Bluetooth audio.

### 3.1.2 Multi-channel register map

All registers 32-bit. Only offsets `0x00`–`0x24` and `0x3C` are used.

| Offset | Name | Reset value | Access |
|---|---|---|---|
| `0x00` | `CTRL` | `0x01900000` | RW |
| `0x04` | `INTR_STAT` | `0x00003300` | RW (status; write 0 to clear the error bits) |
| `0x08` | `SRR` (sample rate and resolution) | 0 | RW |
| `0x0C` | `CID_CTRL` (clock strobes and interrupt masks) | 0 | RW |
| `0x10` | `TFIFO_STAT` (transmit FIFO level) | 0 | RO |
| `0x14` | `RFIFO_STAT` (receive FIFO level) | 0 | RO |
| `0x18` | `TFIFO_CTRL` (transmit thresholds) | `0x000F0000` | RW |
| `0x1C` | `RFIFO_CTRL` (receive thresholds) | `0x000F0000` | RW |
| `0x20` | `DEV_CONF` (framing) | `0x00000208` | RW |
| `0x24` | `POLL_STAT` | `0x00000003` | RO |
| `0x3C` | FIFO data port (transmit and receive) | — | DMA target |

`CTRL` (`0x00`):

| Bits | Field | Meaning |
|---|---|---|
| `[7:0]` | `I2S_EN` | Enable per data line (bit n = line n) |
| `[15:8]` | `TR_CFG` | Direction per data line: 1 = transmit, 0 = receive |
| `[16]` … `[19]` | `LOOP_BACK_0_1` … `LOOP_BACK_6_7` | Internal loopback between line pairs |
| `[20]` | `SFR_RST` | Register-file reset (not used by Linux) |
| `[21]` | `T_MS` | Transmitter is clock master (drives SCK/WS) |
| `[22]` | `R_MS` | Receiver is clock master |
| `[23]` | `TFIFO_RST` | 0 = reset transmit FIFO pointers; hardware sets it back to 1 one clock later |
| `[24]` | `RFIFO_RST` | Same for the receive FIFO |
| `[25]` | `TSYNC_RST` | 1 = transmit synchroniser out of reset |
| `[26]` | `RSYNC_RST` | 1 = receive synchroniser out of reset |
| `[27]` | `TSYNC_LOOP_BACK` | Transmit sync loopback |
| `[28]` | `RSYNC_LOOP_BACK` | Receive sync loopback |

`INTR_STAT` (`0x04`): `[0]` transmit underrun, `[3:1]` underrun line code,
`[4]` receive overrun, `[7:5]` overrun line code, `[8]`/`[9]`/`[10]`/`[11]`
transmit FIFO empty / almost empty / full / almost full, `[12]`…`[15]`
the same for the receive FIFO.

`SRR` (`0x08`):

| Bits | Field |
|---|---|
| `[10:0]` | Transmit sample-rate divider |
| `[15:11]` | Transmit resolution = bits per sample − 1 |
| `[26:16]` | Receive sample-rate divider |
| `[31:27]` | Receive resolution = bits per sample − 1 |

`CID_CTRL` (`0x0C`):

| Bits | Field | Meaning |
|---|---|---|
| `[7:0]` | `I2S_STROBE` | Per-line clock strobe: **1 = clock stopped**, 0 = running |
| `[8]` | `STROBE_TS` | Transmit clock: 1 = stopped |
| `[9]` | `STROBE_RS` | Receive clock: 1 = stopped |
| `[15]` | `INTREQ_MASK` | Global interrupt enable (1 = enabled) |
| `[23:16]` | `I2S_MASK` | Per-line error interrupt enable (1 = enabled) |
| `[24]` … `[27]` | transmit FIFO empty / almost empty / full / almost full interrupt enables |
| `[28]` … `[31]` | receive FIFO interrupt enables, same order |

`TFIFO_CTRL` / `RFIFO_CTRL`: `[15:0]` almost-empty threshold, `[31:16]`
almost-full threshold.

`DEV_CONF` (`0x20`), transmitter in `[5:0]`, receiver in `[11:6]` with the
same layout:

| Bit (tx / rx) | Field |
|---|---|
| 0 / 6 | `SCK_POLAR`: 1 = normal bit clock, 0 = inverted bit clock |
| 1 / 7 | `WS_POLAR`: 1 = frame clock inverted |
| 2 / 8 | `APB_ALIGN_LR`: sample alignment in the FIFO word |
| 3 / 9 | `I2S_ALIGN_LR`: left-justified data within the slot |
| 4 / 10 | `DATA_WS_DEL`: one-bit delay of data after WS (cleared for I2S) |
| 5 / 11 | `WS_DSP_MODE`: one-cycle WS pulse (DSP/TDM framing) |

Framing encodings (transmit shown; receive identical with the `[11:6]`
bits):

| Format | `I2S_ALIGN_LR` | `DATA_WS_DEL` | `WS_DSP_MODE` |
|---|---|---|---|
| I2S | 1 | 0 | 0 |
| Right-justified | 0 | 1 | 0 |
| Left-justified | 1 | 1 | 0 |
| DSP A | 1 | 1 | 1 |
| DSP B | 1 | 0 | 1 |

`APB_ALIGN_LR` is always 0. Polarity: normal bit clock and frame clock =
`SCK_POLAR` 1, `WS_POLAR` 0; inverted bit clock = `SCK_POLAR` 0.

> OPEN QUESTION: The `DATA_WS_DEL` encoding looks inverted against the
> usual I2S definition (I2S has a one-bit delay but the field is 0). It is
> recorded as the vendor driver programs it, which is known to work with
> the DP transmitters.

`POLL_STAT` (`0x24`): `[0]` tx empty, `[1]` tx almost empty, `[2]` tx
underrun, `[4]` rx full, `[5]` rx almost full, `[6]` rx overrun.

### 3.1.3 Parameters

| Parameter | Value |
|---|---|
| Slot width | 32 bits always |
| Sample formats | 16-, 24- (in 32-bit container) and 32-bit little-endian |
| Channels | 2 … 16; two per enabled data line |
| Rates | 8 kHz … 192 kHz in the 48 kHz and 44.1 kHz families |
| FIFO thresholds | almost empty 4, almost full 12 (both directions) |
| DMA burst | 4 words; bus width follows the sample format |

### 3.1.4 Programming

**[HW]** Probe or power-up:

1. Enable the I2S APB and core clocks (CRU 16 + n, 26 + n), pulse CRU reset
   n (1–2 µs).
2. Disable all lines (`CTRL.I2S_EN` = 0), hold both synchronisers in reset
   (`TSYNC_RST` = `RSYNC_RST` = 0), stop both clocks (`STROBE_TS` =
   `STROBE_RS` = 1), and clear the global interrupt enable.

**[HW]** Format (per direction): write `DEV_CONF` framing and polarity
(§3.1.2), the master bit (`T_MS` or `R_MS`), and `TR_CFG`: for transmit
the configured transmit-line mask; for receive the inverse of the receive
mask.

**[HW]** Stream parameters, for sample rate *fs* and channel count *c*:

1. Select the core clock source: `audio_clk0` for multiples of 8 kHz,
   `audio_clk2` for multiples of 11.025 kHz (01 §1.1.3). Other rates are
   rejected.
2. Let *F* be the resulting core clock. As clock master, *F* must be at
   least 12 × 32 × *fs* (6 × 2 × slot width); as slave at least 14 × 32 ×
   *fs*.
3. As clock master: sample-rate divider = round(*F* / (2 × 32 × *fs*)), into `SRR`
   `[10:0]` (tx) or `[26:16]` (rx). Example: 294.912 MHz at 48 kHz → 96.
4. Resolution = sample bits − 1, into `SRR` `[15:11]` or `[31:27]`.
5. Active lines: if *c* < 2 × (lines in the mask), use the lowest-numbered
   *c*/2 lines of the mask; more than 2 × lines is an error.

**[HW]** Stream open, before the DMA is armed: start that direction's
clock (`STROBE_TS`/`STROBE_RS` = 0), then pulse its FIFO reset (write 0
then 1 to `TFIFO_RST`/`RFIFO_RST`), then write the thresholds, then set
`INTREQ_MASK`. The FIFO reset must come **after** the clock is running and
**before** the DMA can see a request.

**[HW]** Start: set the active lines' `I2S_MASK` bits (error interrupts),
clear their `I2S_STROBE` bits, then set their `I2S_EN` bits together with
`TSYNC_RST` (or `RSYNC_RST`). Stop: the reverse — clear `I2S_EN` and the
synchroniser bit, set the strobes, clear the masks.

- **[POLICY]** *(community)* Start the DMA **before** the I2S and stop it
  **after** the I2S.
- **[HW]** Interrupt: read `INTR_STAT`; if non-zero, clear `[0]` and `[4]`
  by writing them as 0. Errors are informational; Linux only counts them.
- **[HW]** After power loss the registers must be rewritten (Linux restores
  a cached copy of `0x00`–`0x20` after pulsing the reset).

**[HW]** DP output configuration (I2S5–9, from the sound card): I2S
framing, bit clock inverted, frame clock normal, I2S is clock master,
transmit-line mask `0x0F`, up to 8 channels.

## 3.2 Audio DMA-350

The DMA engine is an **Arm CoreLink DMA-350**. Its register map, command
link format and trigger interface are those of the public Arm *DMA-350
Technical Reference Manual*; only Sky1-specific integration is described.

### 3.2.1 Instances

| Item | Audio DMA | FCH DMA |
|---|---|---|
| ACPI `_HID` | `CIXH1006` | `CIXHA014` |
| Vendor DT compatible | `arm,dma350-no-pause` | `arm,dma350-full` |
| Base | `0x07010000`, 64 KiB | `0x04190000`, 64 KiB |
| Interrupt | SPI 230, **one line for all channels** | SPI 303, one line |
| Channels / trigger inputs | 8 / 20 | 8 / 8 |
| Clock | CRU output 9 (`axiclk`) | SCMI `CLK_TREE_FCH_DMA_ACLK` (239) |
| Reset | CRU reset 15 (`dma_reset`) | FCH reset 11 (`dma_reset`) |
| Power | audio domain | FCH (not the audio domain) |

- **[HW]** The channel and trigger counts are also readable from the
  controller's configuration registers; the table gives the values the
  vendor DT declares.
- **[HW]** With a single interrupt line, every channel's interrupt arrives
  on SPI 230; the handler must check each channel's status.
- **[HW]** *(vendor)* The audio instance does **not** support channel
  pause/resume (its compatible says so). Stop and restart instead.
- **[HW]** The FCH DMA's clock appears in another device's `CLKT`
  (`sky1-soc` §1.4.1). Do not give the audio DMA the SCMI clock 239; that
  bypasses the CRU gate and the first register access faults.

### 3.2.2 Probe order (audio instance)

**[HW]** 1. Audio domain and CRU ready (00 §0.5). 2. Enable CRU output 9.
3. Release CRU reset 15. 4. Only now read the identification and
configuration registers. 5. Set the controller-level **non-secure
interrupt enable** (offset `0x20C` from the controller base, bit 0,
"interrupt on any channel"); without it no channel interrupt is
signalled.

### 3.2.3 Interrupt routing through the CRU

**[HW]** Each channel's interrupt reaches the AP only if the matching bit
in CRU `AP_IRQ` (`0x07110054`) is 1 (bit n = channel n). Set it when a
channel is put in use. Without it, transfers complete but no interrupt
arrives; I2S then underruns and its error interrupt storms.

> OPEN QUESTION: Clearing a bit presumably routes that channel's
> interrupt to the DSP instead. Linux never clears it.

### 3.2.4 Address translation

**[HW]** The engine sees system memory at CPU address − `0x90000000`
(00 §0.3). The vendor DT (and the ACPI `_DSD`) give it as two pairs:

| Property | CPU address | DMA address | Applied to |
|---|---|---|---|
| `arm,ram-map` | `0xC0000000` | `0x30000000` | memory addresses ≥ CPU `0xC0000000` |
| `arm,reg-map` | `0x07010000` | `0x20000000` | other addresses (see question) |

- **[HW]** Memory addresses (buffers and command-link structures) must be
  translated with the RAM pair. **[POLICY]** Linux puts all PCM buffers in
  the reserved region at `0xD0000000` (12 MiB) so they always translate.
- **[HW]** Peripheral FIFO addresses are given **untranslated** (CPU
  address of the I2S FIFO port, for example `0x0707003C` for I2S5).

> OPEN QUESTION: The `arm,reg-map` pair is read and would be applied to
> memory addresses below `0xC0000000`, which looks wrong; Linux avoids it
> by never using such memory. Its intended meaning (perhaps the DSP's
> local view of the audio peripherals) is unknown.

### 3.2.5 Peripheral transfers

**[HW]** For an I2S stream, each channel uses hardware flow control from
one trigger input:

| Item | Value |
|---|---|
| Trigger select (`SRCTRIGINCFG`/`DESTRIGINCFG` `[7:0]`) | request line (§3.3) |
| Trigger type (`[9:8]`) | 2 = hardware request |
| Trigger mode (`[11:10]`) | 3 = peripheral flow control |
| Playback | destination trigger used; source = memory (incrementing, normal non-cacheable), destination = FIFO port (fixed, device memory) |
| Capture | source trigger used; source = FIFO port, destination = memory |
| Cyclic buffers | a ring of command-link entries, one per period, each raising "done" interrupt; the last links back to the first |

> OPEN QUESTION: Linux uses the same control-register bit (26) for "use
> source trigger" and "use destination trigger". The public TRM assigns
> them different bits. A new driver should follow the TRM. Capture DMA on
> Sky1 is untested in the sources.

## 3.3 Request lines

**[HW]** The audio DMA's 20 trigger inputs are wired as: line 2n = I2S n
transmit, line 2n + 1 = I2S n receive (n = 0 … 9).

| I2S | TX line | RX line | Orion O6 use |
|---|---|---|---|
| 0 | 0 | 1 | both |
| 1 | 2 | 3 | both |
| 2 | 4 | 5 | both |
| 3 | 6 | (7) | TX only |
| 4 | (8) | 9 | RX only |
| 5 | 10 | (11) | TX (DP0) |
| 6 | 12 | (13) | TX (DP1) |
| 7 | 14 | (15) | TX (DP2) |
| 8 | 16 | (17) | TX (DP3) |
| 9 | 18 | (19) | TX (DP4) |

Values in parentheses follow the pattern but no source uses them.

- **[HW]** DT: two cells, (request line, channel); channel `0xFF` = any
  free channel. Names `tx` and `rx`.
- **[HW]** ACPI: each I2S `_CRS` carries `FixedDMA` descriptors;
  `DmaRequestLine` = request line, `Channel` = channel (`0xFF` = any),
  matched by name through `_DSD` `dma-names` (`tx`, `rx`). The DMA
  controller must register as an ACPI DMA provider for the I2S devices to
  resolve them.

## 3.4 Sound card topology

**[ABI]** *(community)* Under ACPI the board SSDT has a device `CIXH6070`
with no resources. Its presence means "create the Sky1 sound card"; the
topology is fixed:

| Link | CPU side | Codec side | Stream name |
|---|---|---|---|
| 0 | `CIXH6011` instance 2 (I2S5), playback | audio input of DP0 (DP device `_UID` 0) | `HDMI/DP 0` |
| 1 | instance 3 (I2S6) | DP1 | `HDMI/DP 1` |
| 2 | instance 4 (I2S7) | DP2 | `HDMI/DP 2` |
| 3 | instance 5 (I2S8) | DP3 | `HDMI/DP 3` |
| 4 | instance 6 (I2S9) | DP4 | `HDMI/DP 4` |

- **[ABI]** Card name **`cix_sky1`** (matches the ALSA UCM profile
  directory). Each link has a line-out jack named `HDMI/DP,pcm=N`.
  Missing I2S devices are skipped and later links renumbered.
- **[HW]** The PCM data path for each link is DMA-350 → I2S FIFO. Link
  format as in §3.1.4 (DP output configuration).
- **[ABI]** Codec ABI and ELD behaviour: `sky1-dp-tx` §5.4.
- **[POLICY]** The DT card (`cix,sky1-sound-card`) allows arbitrary links
  (format, TDM slots, `mclk-fs`, `mclk-idx`, jack flags per link). The
  Orion O6 DT defines Bluetooth (I2S1, DSP A, 1×16-bit slot), loopback
  (I2S2) and the five DP links; only the DP links are enabled.
- The HDA controller registers its own separate card (02 §2.7).

## 3.5 Audio DSP (HiFi5) — brief

Optional: needed only for DSP-offloaded audio (the vendor Sound Open
Firmware path). Not needed for HDA or DP audio.

### 3.5.1 Identity and resources

| Item | Value |
|---|---|
| Core | Cadence Tensilica HiFi5 |
| DT / ACPI | `cix,sky1-hifi5` / `CIXH6000` |
| Interrupt | SPI 229: DSP watchdog expiry |
| Resets | S5 reset 30 (`dsp`, core); CRU resets 12, 13 (`mb0`, `mb1`) |
| Clocks | CRU outputs 3, 4, 5, 6, 14, 15 |
| Control | CRU `INFO_HIFI0`–`2` via `cix,dsp-ctrl` → audio CRU |
| Mailboxes | AP→DSP `0x070F0000`, DSP→AP `0x07100000`, FIFO channel 9 on both (`sky1-soc` §1.1) |
| Firmware | `dsp_fw.bin`, ELF image, loaded by program headers (`PT_LOAD` segments by physical address) |

### 3.5.2 Memory map

| Region | DSP address | AP physical address | Size |
|---|---|---|---|
| IRAM | `0x20200000` | `0x1C_8020_0000` | 128 KiB |
| DRAM0 | `0x20220000` | `0x1C_8022_0000` | 128 KiB |
| DRAM1 | `0x20240000` | `0x1C_8024_0000` | 128 KiB |
| External SRAM | `0x20400000` | `0x1C_8040_0000` | 1 MiB |
| vring 0 | `0x3DE00000` | `0xCDE00000` | 16 KiB |
| vring 1 | `0x3DE04000` | `0xCDE04000` | 16 KiB |
| virtio buffers | `0x3DE08000` | `0xCDE08000` | 1 MiB |
| Firmware working memory | `0x3E000000` | `0xCE000000` | 16 MiB |

- **[HW]** The SRAM regions are device memory to the AP (map uncached or
  write-combined); the DRAM regions are ordinary memory.

> OPEN QUESTION: The AP addresses of the DSP SRAMs (`0x1C_80xx_xxxx`) sit
> far from the audio register window; their relation to the RCSU remap
> word is not documented.

### 3.5.3 Control registers (CRU)

| Register | Bits | Meaning |
|---|---|---|
| `INFO_HIFI0` (`0x00`) | `[0]` | DSP core clock enable |
| | `[1]` | On-chip-debug halt on reset |
| `INFO_HIFI1` (`0x04`) | `[31:0]` | Alternate reset vector (aligned address); `[0]` doubles as "use alternate vector" |
| `INFO_HIFI2` (`0x08`) | `[0]` | `PWAITMODE` status/control |
| | `[1]` | `RunStall`: 1 = core stalled |
| | `[2]` | NMI to the DSP |

### 3.5.4 Boot sequence

**[HW]**

1. Enable the six DSP clocks; pulse mailbox resets 12 and 13 (≥ 1 µs).
2. Assert the DSP core reset (S5 30), wait ≥ 1 µs (≥ 10 DSP cycles), set
   `RunStall`.
3. While in reset, write the firmware entry address to `INFO_HIFI1` and set
   its bit 0. Keep it stable ≥ 10 cycles, then release the reset. The core
   stays stalled.
4. Zero the memory regions (except when resuming), copy the `PT_LOAD`
   segments, translating DSP addresses with §3.5.2.
5. Pulse the core reset again (assert ≥ 1 µs, release), then clear
   `RunStall`.
6. Wait for the "ready" message (below). **[POLICY]** Linux waits up to
   3000 polls of 200–500 µs, then enables the watchdog interrupt.

Stop: send "stop", wait ≤ 100 ms for its acknowledgement (skip if the
watchdog fired), then set `RunStall`.

### 3.5.5 IPC transport

**[HW]** Each mailbox message is two 32-bit words over the FIFO channel:
word 0 = message length in bytes (8), word 1 = payload.

| Payload | Direction | Meaning |
|---|---|---|
| 0, 1 | both | virtqueue 0 / 1 has work ("kick") |
| `0xFFF0` | DSP→AP | firmware ready |
| `0xFFF1` | DSP→AP | firmware crashed |
| `0xFFF2` / `0xFFF3` | AP→DSP / DSP→AP | system suspend / acknowledge |
| `0xFFF4` / `0xFFF5` | AP→DSP / DSP→AP | system resume / acknowledge |
| `0xFFF6` / `0xFFF7` | AP→DSP / DSP→AP | stop / acknowledge |

- **[HW]** Above that the firmware speaks standard virtio/rpmsg over the
  two vrings, using the open-source *rpmsg-lite* stack as the remote side.
- **[HW]** *(community)* After "ready" the AP must kick virtqueue 0 once
  (after ~10 ms) so that the firmware marks the link up and announces its
  endpoints.
- **[HW]** *(community)* The firmware does not kick the AP after queueing
  messages, and places them in the available ring rather than the used
  ring. Linux therefore polls both vrings every 50 ms.
- **[HW]** On a watchdog interrupt, mask it (it is level and stays
  asserted) and treat the DSP as crashed.

> OPEN QUESTION: The rpmsg services the firmware offers (and the SOF IPC
> carried over them) are not specified here.
