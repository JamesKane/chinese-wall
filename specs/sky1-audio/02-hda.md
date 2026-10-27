# 02: IPbloq HD-Audio Controller

The audio subsystem contains one **HD-Audio controller** licensed from
IPbloq, at **`0x070C0000`** (64 KiB, GIC SPI **234**, GSIV 266). Its register
file and DMA engine follow the public *Intel High Definition Audio
Specification* (rev. 1.0a): `GCAP`, `GCTL`, CORB/RIRB, stream descriptors,
BDLs and immediate commands behave as specified unless listed below.

| Boot | Identification |
|---|---|
| DT | `compatible = "cix,sky1-ipbloq-hda"` (public binding; vendor DT the same) |
| ACPI | `_HID` `CIXH6020` |

It drives the board's analog codec only. **DisplayPort audio does not go
through HDA** on Sky1: the five DP transmitters take audio over I2S from
I2S5 … I2S9 (03 §3.1, and `sky1-dp-tx` §2.8). There is no HDMI/DP codec on
the HDA link.

## 2.1 Resources

| Resource | Value |
|---|---|
| Registers | `0x070C0000`, 64 KiB |
| Interrupt | SPI 234, level-high |
| Clocks | CRU output 7 (HDA system clock, 400 MHz nominal) and CRU output 8 (48 MHz HDA link clock). Both share CRU gate bit 14 |
| Clock names | Mainline binding: `ipg` (output 7), `per` (output 8). Vendor DT: `sysclk`, `clk48m` |
| Reset | CRU block reset 14 (vendor DT name `hda`; mainline binding unnamed) |
| Power | Audio domain (00 §0.4) |
| DMA window | Bus address = CPU address − `0x90000000`; 2 GiB; 32-bit (§2.3) |
| Reserved memory | Vendor DT: `0xD0C00000`, 2 MiB DMA pool. ACPI: `RSVL` (§2.6) |
| Board GPIO (Orion O6) | `pdb-gpios`: codec power-down control, FCH GPIO bank 3 line 5, active-high (§2.5) |
| Pins | HDA `BITCLK`, `RST#`, `SYNC`, `SDO0`, `SDO1`, `SDI0`, `SDI1` (share pads with I2S0) |

## 2.2 Deviations from the Intel HDA specification

### 2.2.1 Register access width

**[HW]** Only **aligned 32-bit** accesses work. An 8-bit or 16-bit
register (for example `GCAP`, `STATESTS`, `CORBWP`, `RIRBCTL`, `SDnCTL`
bytes, `SDnLVI`) must be read as the containing 32-bit word and the field
extracted. A write must be a read-modify-write of the containing word,
with the other bytes written back unchanged.

- **[HW]** Take care with write-1-to-clear bytes that share a word with
  the target (for example `RIRBSTS` next to `RIRBCTL`, `SDnSTS` next to
  `SDnCTL`): writing back a read value of 1 clears them. Mask such bits
  to 0 in the write-back unless clearing is intended.

> OPEN QUESTION: Whether the IP silently ignores, mis-decodes or faults on
> narrow accesses is not stated. Linux simply never issues them.

### 2.2.2 RIRB response interrupt cannot be cleared

**[HW]** The RIRB "response interrupt" flag (`RIRBSTS.RINTFL`) **cannot be
cleared** on this IP. With the RIRB interrupt enabled, the interrupt line
stays asserted and the CPU is stormed. Therefore:

- **[HW]** Program `RIRBCTL` with the RIRB DMA run bit **only**; leave the
  response-interrupt enable off. (`RINTCNT` may still be set; it has no
  effect.)
- **[HW]** Obtain codec responses by **polling** `RIRBWP` after each
  command. Unsolicited responses also land in the RIRB and are only seen
  when the RIRB is polled.
- **[HW]** Stream interrupts (`INTCTL` stream bits, `SDnSTS.BCIS`) and the
  controller interrupt work normally and are used for period completion.

### 2.2.3 Stream reset timing

**[HW]** When resetting a stream descriptor (`SDnCTL.SRST` = 1) that was
running, wait **100 µs** after the controller reports the stream in reset
before clearing `SRST`. (The same delay is used by other non-Intel HDA IPs
in Linux.)

### 2.2.4 Stream position

**[POLICY]** Linux reads the stream position from `SDnLPIB` and does not
use the DMA position buffer for position reporting. Its stated reason is
that the completion interrupt can arrive before the data is consumed.

> OPEN QUESTION: Whether the DMA position buffer (`DPLBASE`/`DPUBASE`)
> works on this IP is not known. If used, its address must be translated
> like every other DMA address (§2.3).

### 2.2.5 Stream counts

**[HW]** Stream counts come from `GCAP` as usual: input streams
`GCAP[11:8]`, output streams `GCAP[15:12]`. Input descriptors come first,
output descriptors follow. Linux ignores the bidirectional count.

> OPEN QUESTION: The actual `GCAP` value is not recorded in any source.

### 2.2.6 Unsolicited responses and wake

- **[POLICY]** Since the RIRB is polled (§2.2.2), Linux detects jack changes
  by **polling the codec every 1 s**, including while the controller is
  otherwise runtime-suspended.
- **[POLICY]** Codec power-save timeout 100 ms. The codec power is
  dropped by the codec-level power management, not by gating the link.
- **[HW]** No wake-from-codec path is used. `WAKEEN` is not relied on, and
  no wake interrupt is wired to the system wake logic in any source.

## 2.3 DMA addressing

**[HW]** The controller is a 32-bit bus master behind the audio NoC
(00 §0.3):

| CPU physical | Controller bus address |
|---|---|
| `0x0_9000_0000` … `0x1_0FFF_FFFF` (2 GiB) | `0x0000_0000` … `0x7FFF_FFFF` |

- **[HW]** Every DMA address written to the controller must be translated:
  `CORBLBASE/CORBUBASE`, `RIRBLBASE/RIRBUBASE`, `DPLBASE/DPUBASE`,
  `SDnBDPL/SDnBDPU`, and **each buffer address inside every BDL entry**.
  Bus address = CPU physical − `0x90000000`.
- **[HW]** All of that memory must lie in the window above. Upper address
  words are always 0 on this platform.
- **[POLICY]** The vendor DT expresses the window as `dma-ranges` on a
  `simple-bus` wrapper around the HDA node. The mainline driver instead
  applies a fixed −`0x90000000` offset to every programmed address and
  limits DMA to 32 bits. The ACPI path installs the same window in
  software.
- **[POLICY]** Linux constrains PCM buffer and period sizes to multiples
  of 128 bytes on this controller.

## 2.4 Clocks, reset and controller bring-up

**[HW]** Power-up order:

1. Audio domain on and CRU ready (00 §0.5).
2. Enable CRU outputs 7 and 8 (gate bit 14).
3. Pulse CRU block reset 14: assert, hold (vendor: 10–20 µs; mainline:
   the reset driver's 50 µs), release.
4. Standard HDA controller reset: `GCTL.CRST` 0 → 1, wait for codecs to
   report in `STATESTS`.
5. Set up CORB and RIRB with translated addresses (§2.3) and RIRB
   interrupt **disabled** (§2.2.2); enable unsolicited responses in
   `GCTL.UNSOL`.
6. Probe codec addresses. **[POLICY]** Linux probes addresses 0 … 7 only.

**[HW]** Power-down (runtime suspend): stop all streams and the CORB/RIRB,
put the link in reset (`GCTL.CRST` = 0), then gate outputs 7 and 8. The
next power-up repeats steps 2–5. **[POLICY]** Linux does not hold the
block reset asserted while suspended.

## 2.5 Board codec (Orion O6, Orange Pi 6 Plus)

These are board facts from the vendor sources, for completeness.

| Board | Codec | Subsystem ID | Model string (vendor DT `cix,model`) |
|---|---|---|---|
| Radxa Orion O6 | Realtek ALC256 | `0x10EC129E` | `CIX SKY1 ORION O6 HDA` |
| Orange Pi 6 Plus | Realtek ALC269 family | — | `CIX SKY1 ORAPI 6P HDA` |
| CIX evaluation board | Realtek ALC256 | `0x10EC129E` | `CIX SKY1 EVB HDA` |

- **[HW]** *(vendor)* Orion O6: the codec's power-down pin is driven from
  FCH GPIO bank 3 line 5. Drive it high (codec on) and wait **20 ms**
  before the controller reset (§2.4 step 4). Drive it low on system
  suspend; on resume drive it high and wait 20 ms again. An optional
  second line (`depop-mute`) is driven high while active.
- **[POLICY]** *(vendor)* The vendor driver writes a board-specific table
  of codec verbs (subsystem ID, pin configuration defaults, vendor
  coefficient writes) before codec probe and after every resume.
  Mainline does not; it relies on the codec's own defaults and the
  generic codec driver's quirk table.
- **[HW]** *(vendor)* Orion O6 pin configuration defaults loaded by that
  table (codec node → default config): `0x12` → `0x40000000`, `0x13` →
  `0x411111F0`, `0x14` → `0x90170110` (internal speaker), `0x18` →
  `0x411111F0`, `0x19` → `0x01A11030` (mic), `0x1A` → `0x02A19040`
  (mic), `0x1B` → `0x02014020` (line out), `0x1D` → `0x4045C069`,
  `0x1E` → `0x411111F0`, `0x21` → `0x0121101F` (headphone).
- **[POLICY]** *(community)* For codec subsystem ID `0x10EC:0x129E` the
  community kernel disables automatic microphone switching and the codec's
  3 kΩ headset-detect pull-down, so the internal DMIC stays usable with a
  headset connected.

> OPEN QUESTION: Whether the UEFI firmware already programs the codec pin
> defaults (making the verb table unnecessary under ACPI) is not known.

## 2.6 ACPI attachment (`CIXH6020`)

- **[HW]** `_CRS`: one `Memory32Fixed` for the controller, one interrupt.
- **[HW]** Clocks come from the CRU's `CLKA` table (01 §1.6); the reset is
  CRU index 14 (00 §0.6.2).
- **[HW]** The DMA window of §2.3 is **not** described by an ACPI
  `_DMA` object in any source. An OS must apply it from built-in
  knowledge.
- **[HW]** Reserved DMA memory comes from a CIX table **`RSVL`** on the
  `CIXA1019` device (the CIX resource-table device, `sky1-soc` §2.3.3).
  `RSVL` evaluates to a package of entries, each a package of at least 4
  elements:

| Element | Type | Meaning |
|---|---|---|
| 0 | Integer | CPU physical base |
| 1 | Integer | Size in bytes |
| 2 | String | Flags |
| 3 | Reference | Consumer device |

- **[HW]** Use the entry whose element 3 is the HDA device. Its range must
  lie inside the DMA window; translate the base as in §2.3 and use the
  region as the controller's DMA pool. Reject a region outside the
  window.
- **[POLICY]** *(community)* The region may still be in the OS linear map
  (firmware did not remove it), so it is mapped write-back, relying on the
  platform's I/O coherency.

> OPEN QUESTION: The meaning of the `RSVL` flags string, and whether the
> region is the same `0xD0C00000`/2 MiB as in the vendor DT, are not
> visible. Whether HDA DMA is I/O-coherent with the CPU caches (the
> community code assumes so) is not proven.

## 2.7 Userspace-visible behaviour

- **[ABI]** Card driver name `cix-ipbloq-hda`. Card short name = the
  devicetree `model` property (mainline) or `cix,model` (vendor), else the
  driver name. Long name: short name, `at 0x<base> irq <n>`.
- **[ABI]** PCM devices, mixer controls and jack names are those of the
  generic HDA codec driver for the detected codec; nothing CIX-specific.
