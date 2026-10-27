# 00: Audio Subsystem Overview, Address Map, Power and ACPI Attachment

The Sky1 audio subsystem ("AUDSS") is a self-contained island in the
`0x07000000`–`0x07FFFFFF` window. It holds a Tensilica HiFi5 DSP, an Arm
DMA-350 controller, ten Cadence-derived I2S controllers, an IPbloq HD-Audio
controller, a mailbox pair to the DSP, a clock-and-reset unit (CRU) and the
subsystem RCSU. All of it sits in one power domain.

Related facts already specified elsewhere (not repeated here):

| Topic | Where |
|---|---|
| Mailbox register map, FIFO channel protocol, AP↔DSP instances | `sky1-soc` §1.1 |
| SCMI clocks (`CLK_TREE_*` IDs), CLKT format, `CIXHA010` | `sky1-soc` §1.2, §1.4 |
| SCMI power domain 0 = audio; `POWER_STATE_SET` | `sky1-soc` §1.3 |
| S5 resets `AUDIO_HIFI5` (30) and `AUDIO_HIFI5_NOC` (31), `AUDIO_RCSU` (91) | `sky1-soc` §2.1, Annex A |
| RSTL, RSNL, DLKL tables and `CIXA1019` | `sky1-soc` §2.2, §2.3 |
| Audio RCSU writes (`0x34` remap, `0x1000` bus timeout) | `sky1-soc` §2.4.2 |
| PPRS power resource, DMRP memory repair | `sky1-soc` §2.5 |
| DP transmitter audio input, channel-status programming, HDMI codec ABI | `sky1-dp-tx` §2.8, §5.4 |

## 0.1 Block diagram

```
                      SCMI clocks (PM firmware)
       audio_clk0 294.912M  audio_clk1 344.064M  audio_clk2 270.9504M
       audio_clk3 316.1088M audio_clk4 800M      audio_clk5 48M
                                  |
   S5 reset 31 (NOC) ---> +-------v--------+ <--- power domain 0 / PPRS
   S5 reset 30 (DSP) -+   | CRU 0x07110000 |  clocks, gates, block resets,
                      |   +--+--+--+--+--+-+  DSP control, DMA IRQ routing
                      |      |  |  |  |  |
     +----------------v-+    |  |  |  |  +--> MCLK0..4 pads
     | HiFi5 DSP        |<---+  |  |  |
     | IRAM/DRAM/ext SRAM|      |  |  +--> HDA 0x070C0000 --> HDA link --> codec
     +--+----------+----+       |  |       (ALC256 / ALC269 on boards)
        |  mailbox |           |  |
        |  0x070F0000 (AP->DSP) |  +--> DMA-350 0x07010000 (8 ch, 20 req)
        |  0x07100000 (DSP->AP) |            |  request lines
        |                       |            v
        |                       +--> I2S0..I2S9 0x07020000..0x070B0000
        |                              I2S0-2: single-channel (codec, BT, loopback)
        |                              I2S3-4: multi-channel (speaker, DP-in)
        |                              I2S5-9: multi-channel -> DP0..DP4 audio in
        +--> DRAM through the audio NoC (address offset, §0.3)
```

## 0.2 Address map

| Base | Size | Block | GIC SPI (GSIV = SPI + 32) |
|---|---|---|---|
| `0x07000000` | 64 KiB | Audio RCSU (`sky1-soc` §2.4.2) | — |
| `0x07000000` | 16 MiB | Whole-island window claimed by the DSP node (vendor DT) | 229 (DSP watchdog) |
| `0x07010000` | 64 KiB | DMA-350 | 230 (one line for all channels) |
| `0x07020000` + n × `0x10000` | 64 KiB each | I2S n, n = 0 … 9 (I2S9 at `0x070B0000`) | 219 + n |
| `0x070C0000` | 64 KiB | IPbloq HDA controller | 234 |
| `0x070F0000` | 64 KiB | Mailbox AP→DSP ("MB0") | 232 |
| `0x07100000` | 64 KiB | Mailbox DSP→AP ("MB1") | 233 |
| `0x07110000` | 64 KiB | Clock and reset unit (CRU) | — |

> OPEN QUESTION: `0x070D0000` and `0x070E0000` are unaccounted for. The
> CRU has gates and resets for an audio **watchdog** and **timer**
> (01 §1.3), which are probably there; no node describes them. SPI 231 is
> likewise unassigned.

DSP-private memories and system memory reserved for audio are listed in 03
§3.5 and §0.3 below.

## 0.3 How audio bus masters see system memory

**[HW]** The DSP, the DMA-350 and the HDA controller reach DRAM through the
audio NoC, which presents DRAM at **CPU physical address − `0x90000000`**:

| Master | Evidence | CPU range → bus range |
|---|---|---|
| HDA | DT `dma-ranges` on the HDA sub-bus; mainline driver offset; ACPI fallback | `0x9000_0000`–`0x1_0FFF_FFFF` → `0x0000_0000`–`0x7FFF_FFFF` (2 GiB) |
| DMA-350 | `arm,ram-map = <0xc0000000 0x30000000>` | from `0xC000_0000` → from `0x3000_0000` |
| DSP | firmware memory map (03 §3.5) | `0xCDE0_0000` → `0x3DE0_0000`, `0xCE00_0000` → `0x3E00_0000` |

- **[HW]** Any address the OS gives one of these masters (HDA CORB, RIRB,
  BDLs, buffers, position buffer; DMA-350 command links and buffers; DSP
  vrings) must be translated by subtracting `0x90000000`, and the memory
  must lie inside the window.
- **[POLICY]** Linux places audio buffers in dedicated reserved regions
  (table below) so that the allocator never returns memory outside the
  window.

| CPU address | Size | Use | Bus address |
|---|---|---|---|
| `0xCDE00000` | 16 KiB | DSP vring 0 | `0x3DE00000` |
| `0xCDE04000` | 16 KiB | DSP vring 1 | `0x3DE04000` |
| `0xCDE08000` | 1 MiB | DSP virtio buffer pool | `0x3DE08000` |
| `0xCE000000` | 16 MiB | DSP firmware working memory | `0x3E000000` |
| `0xCF000000` | 16 MiB | DSP shared heap (vendor DMA-buf heap) | — |
| `0xD0000000` | 12 MiB | I2S/DMA-350 PCM buffers | `0x40000000` |
| `0xD0C00000` | 2 MiB | HDA buffers | `0x40C00000` |

All are `no-map` in the vendor devicetree. The last two are DMA pools.

> OPEN QUESTION: The audio RCSU word `0x34` = `0x20000000` (`sky1-soc`
> §2.4.2) is called an "address remap". How it relates to the
> `0x90000000` offset is not stated. Treat both as fixed facts.

## 0.4 Power domain

- **[HW]** Everything in §0.2 is in **one power domain**: SCMI power domain
  **0** under DT, the ACPI power resource **`PPRS`** under ACPI. With the
  domain off, any AP access to an audio register raises an **asynchronous
  SError** (fatal on arm64). This includes the DMA-350 identification
  registers, which a generic DMA driver might read at probe.
- **[HW]** Under ACPI the clock controller device (`CIXH6061`) carries the
  power resource. Putting that device in **D0** runs `PPRS._ON`, which
  enables the power gates and runs DMRP memory repair (`sky1-soc` §2.5).
- **[POLICY]** Under ACPI, Sky1-Linux does not register the SCMI power
  domains at all, and holds the audio domain on from the clock controller's
  probe onwards.
- **[HW]** Under DT, the CRU and the DMA-350 name power domain 0. The
  vendor DT additionally names it `"audio"` on the clock controller.

## 0.5 Bring-up order

**[HW]** A driver set for this subsystem must respect this order. Steps 1–4
belong to the CRU driver (01). Later steps can run in any order once their
own clocks and resets exist.

1. Power the domain (ACPI: D0 on `CIXH6061` → `PPRS._ON`; DT: SCMI domain 0
   on).
2. Enable the SCMI parent clocks `audio_clk0` … `audio_clk5` and set their
   rates (01 §1.2).
3. Pulse the audio **NoC reset** (S5 index 31): assert, wait ≥ 1 µs,
   release. The mainline driver instead asserts it before step 2 and
   releases it after step 2; either order has the clocks running when the
   NoC leaves reset.
4. Write the audio RCSU (`0x34`, `0x1000`; `sky1-soc` §2.4.2), then any CRU
   register.
5. DMA-350: enable its CRU clock, pulse its CRU block reset, then touch its
   registers (03 §3.2).
6. I2S n: enable its two CRU clocks, pulse its CRU block reset (03 §3.1).
7. HDA: enable its two clocks, pulse its CRU block reset, then run the
   standard HDA controller reset (02).
8. DSP: optional (03 §3.5).

- **[HW]** On every later power-up of the domain, repeat steps 2–4 and
  restore the CRU registers (01 §1.5). The vendor driver rewrites RCSU
  `0x34` on every resume; the bus-timeout word is written only at probe.

> OPEN QUESTION: Mainline Linux never writes the audio RCSU. Either the
> firmware already does, or the mainline driver works without it for the
> HDA path. The vendor and ACPI drivers both write it, so a new driver
> should.

## 0.6 ACPI devices

Hardware IDs below come from driver ACPI match tables and are
authoritative. The public ACPI-to-DT mapping notes list several of them
with wrong device types, noted in the last column.

| `_HID` | Instances | Device | Mapping notes say |
|---|---|---|---|
| `CIXH6061` | 1 | Audio clock controller (CRU clock half) | — |
| `CIXH6062` | 1 | Audio reset controller (CRU reset half) | — |
| `CIXHA018` | 5 | Generic register region ("syscon"); one instance is the CRU register block | "Clock Reset Unit", `cix,sky1-cru` |
| `CIXH1006` | 1 | Audio DMA-350 | — |
| `CIXHA014` | 1 | FCH DMA-350 (peripheral DMA at `0x04190000`) | — |
| `CIXH6011` | 7 | I2S multi-channel controllers I2S3 … I2S9 | "temperature sensor" |
| `CIXH6020` | 1 | IPbloq HDA controller | "PL330 DMA" |
| `CIXH6000` | 1 | HiFi5 DSP | "thermal sensor" |
| `CIXH6070` | 1 | Sound card description (board SSDT; no registers) | — |
| `CIXHA001` | 2 of 8 | AP↔DSP mailboxes | — |

- **[ABI]** *(community)* `CIXH6011` instances are numbered in DSDT order.
  Instances **2 … 6** are I2S5 … I2S9, which feed DP0 … DP4. Instances 0
  and 1 are therefore I2S3 and I2S4.
- **[HW]** The single-channel controllers I2S0 … I2S2 have no ACPI device
  on the Orion O6 firmware.
- **[ABI]** *(community)* Not every board exposes all five DP audio I2S
  devices. The Orange Pi 6 Plus firmware lacks one (DP2's). A sound-card
  driver must skip absent ones rather than wait for them.

### 0.6.1 What `CIXHA018` really is

**Finding:** `CIXHA018` is a **generic, featureless register-region device**
(the ACPI equivalent of a DT `"syscon"` node). It is **not** an "audio
clock/reset unit" (the "ACRU" name in the plan). One of its five instances
covers the audio CRU registers at `0x07110000`; the audio clock, reset,
DMA and DSP devices reach those registers through it.

Evidence:

1. **Driver behaviour.** The only driver that matches `CIXHA018` (community
   patch to the generic syscon code) maps resource 0 and exposes it as a
   plain 32-bit register map. It programs nothing and has no clock or reset
   logic.
2. **References from audio devices.** Under ACPI, `CIXH6061` and `CIXH6062`
   find their registers by following a `_DSD` device reference named
   **`audss_cru`** and using the register map of the device it points to.
   `CIXH1006` follows `_DSD` **`arm,remote-ctrl`** and `CIXH6000` follows
   **`cix,dsp-ctrl`** the same way. In the vendor DT all three properties
   point at the node `audss_cru` (`0x07110000`, compatible
   `"cix,audio-ss-cru", "simple-mfd", "syscon"`).
3. **Instance count.** The vendor DT has exactly five plain register-region
   nodes that have no ACPI HID of their own: the FCH CRU window
   (`0x0416009C`, 128 bytes), the audio CRU (`0x07110000`, 64 KiB), the
   sensor-hub CRU (`0x08080000`, 4 KiB), the GMAC RCSU (`0x09310000`,
   4 KiB) and the security-engine public registers (`0x05040000`, 4 KiB).
   This matches the mapping notes' count of 5 for `CIXHA018`. The other
   DT syscons (the S5 and FCH reset controllers, the PDC) have their own
   HIDs (`CIXHA020`, `CIXHA021`, `CIXHA019`).
4. **Naming.** The mapping notes call it "Clock Reset Unit". That fits two
   of the five instances by name (FCH CRU, audio CRU) but the notes' HID
   table is known to be unreliable.

> OPEN QUESTION: The base address of each `CIXHA018` instance is inferred
> from the vendor DT, not read from a DSDT. Confirm which instance is the
> audio CRU by its `Memory32Fixed` base (`0x07110000`) rather than by
> `_UID`.

- **[HW]** An OS should bind `CIXHA018` as "a register window other drivers
  may share". It must be available **before** `CIXH6061`, `CIXH6062`,
  `CIXH1006` and `CIXH6000` start.
- **[HW]** The CRU window is in the audio power domain. Do not read it
  until step 1 of §0.5 has run (a generic region driver must not touch the
  registers at bind time).

### 0.6.2 Tables and properties per device

| Device | `_CRS` | `_DSD` properties used | CIX tables |
|---|---|---|---|
| `CIXH6061` | (none needed; registers via `audss_cru`) | `audss_cru` (reference) | `CLKT` for the six `audio_clkN` parents; **`CLKA`** (01 §1.6); `RSTL` for S5 index 31 (name `"noc"`); power resource `PPRS` |
| `CIXH6062` | — | `audss_cru` (reference) | — |
| `CIXH1006` | Memory `0x07010000`, Interrupt GSIV 262 | `arm,remote-ctrl` (reference), `arm,reg-map`, `arm,ram-map` (2 integers each) | Its clock comes from `CLKA`; reset from `CIXH6062` (index 15) |
| `CIXH6011` | Memory, Interrupt, two or one `FixedDMA` | `cdns,pin-out-num`, `cdns,pin-rx-mask`, `cdns,pin-tx-mask`, `dma-names`, `id` (I2S index) | Clocks from `CLKA` (and `CLKT` for `audio_clkN`) |
| `CIXH6020` | Memory `0x070C0000`, Interrupt GSIV 266 | — | Clocks from `CLKA`; reset from `CIXH6062` (index 14); reserved memory from `RSVL` (02 §2.6) |
| `CIXH6000` | Memory, Interrupt (watchdog) | `cix,dsp-ctrl` (reference), `firmware-name`, `mboxes`, `mbox-names` | Clocks from `CLKA`; resets optional |
| `CIXH6070` | — | — | none known |

> OPEN QUESTION: The `CIXH6061` row is partly inferred. The driver asks
> for parent clocks by the names `audio_clk0` … `audio_clk5` and for a
> reset named `"noc"`; the mapping notes list an audio `RST0` index 31
> entry named `noc`. That the CLKT and RSTL objects sit on `CIXH6061`
> itself is assumed.

- **[HW]** *(community)* Under ACPI the clock controller, reset controller
  and DMA controller each "satisfy" their `_DEP` dependents once ready
  (I2S, DMA and HDA depend on the CRU halves; I2S depends on the DMA).
  An OS honouring `_DEP` must provide the same.
- **[HW]** *(community)* The DMA and HDA reset lines are not resolved
  through the generic RSTL handler, which only knows `RST0`/`RST1`. The
  community driver hard-codes: audio reset index **15** → `CIXH1006`,
  name `"dma_reset"`; index **14** → `CIXH6020`, name `"hda"`.

> OPEN QUESTION: Whether the DSDT has `RSTL` entries that reference the
> audio reset controller (rather than `RST0`/`RST1`), and what they name
> it, is unknown. The hard-coded pairs match the DT.

> OPEN QUESTION: The `_DSD` property names for `CIXH6011` are taken from
> the vendor DT and the driver's property reads. The FixedDMA request-line
> and channel values in each I2S `_CRS` are not visible; they are expected
> to match 03 §3.3.

## 0.7 Devicetree attachment (summary)

**[HW]** Public binding `cix,sky1-audss-cru`: one node at `0x07110000` that
is both clock provider (`#clock-cells = 1`) and reset provider
(`#reset-cells = 1`), with parent clocks `x8k` (`audio_clk0`), `x11k`
(`audio_clk2`), `sys` (`audio_clk4`) and `48m` (`audio_clk5`), power
domain 0 and S5 reset 31. Public binding `cix,sky1-ipbloq-hda`: see 02.

The vendor DT instead uses a parent register-region node with two children
(`cix,sky1-audss-clock`, `cix,sky1-audss-reset`) and six parent clocks
(`audio_clk0` … `audio_clk5`). Clock and reset index numbers are identical
in both (01 §1.3, §1.4).
