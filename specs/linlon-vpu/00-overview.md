# 00: Overview and Platform Integration

## 0.1 What the block is

- **[HW]** The Sky1 video processing unit (VPU) is a multi-core video engine
  ("MVE"). It decodes H.264, HEVC, VP8, VP9, AV1, MPEG-2, MPEG-4/H.263,
  VC-1, RealVideo, AVS, AVS2 and JPEG. It encodes H.264, HEVC, VP8, VP9 and
  JPEG. Each core contains an embedded processor that runs vendor firmware.
  The host never programs codec registers directly.
- **[HW]** The engine has these parts:
  - up to **8 cores** in the architecture, **4 on Sky1** (register `NCORES`),
    some of which may be fused off (harvested);
  - **4 logical session IDs (LSIDs)** (register `NLSID`). An LSID is a
    hardware context slot. It holds one session's MMU root, interrupt
    lines, core limits and bus attributes;
  - a **hardware job scheduler** with a 4-entry job queue. It assigns
    queued LSIDs to free cores;
  - a **per-LSID internal MMU** that translates the firmware's 32-bit
    virtual addresses to 40-bit physical addresses (02).
- **[HW]** One session is one codec instance (one decoder or encoder
  stream). A session is attached to an LSID while it runs. More sessions
  than LSIDs can exist. The host time-multiplexes them (03 §3.7).
- **[HW]** Each session runs its own copy of the codec firmware. The
  firmware image is chosen by codec and direction, for example
  `h264dec.fwb` or `hevcenc.fwb` (§0.9).

### 0.1.1 Hardware revision identification

| `HARDWARE_ID[31:16]` | Family name in the vendor driver |
|---|---|
| `0x5650` | v500 |
| `0x5655` | v550 |
| `0x5660`, `0x5661` | v61 |
| `0x5662`, `0x5663`, `0x5664` | v52_v76 (the Sky1 family) |

- **[HW]** `HARDWARE_ID[15:8]` is the revision and `[7:0]` the patch level.
  The vendor driver refuses any other product value.
- **[HW]** `SVNREV` (`0x30`) holds a build/variant magic value. Two values
  change behaviour:

| `SVNREV` | Meaning |
|---|---|
| `0xE0C1AFE1` | Variant with software-controlled per-core power-off and memory repair. The ENPWOFF sequence in §0.7.3 applies |
| `0x80E11117` | Variant limited to 4096×4096 for H.264/HEVC/VP9/AV1/AVS2 (otherwise 8192×8192) |

> OPEN QUESTION: Which `HARDWARE_ID` and `SVNREV` values Sky1 silicon
> reports is not recorded in the sources. The vendor DT names the block
> `linlon-v8`. The Sky1 power-off sequence is written for the `0xE0C1AFE1`
> variant, which suggests Sky1 is that variant.

## 0.2 Instances and addresses (Sky1)

| Item | Value |
|---|---|
| Instances | 1 |
| RCSU window (resource 0) | `0x1423_0000`, 64 KiB |
| VPU register window (resource 1) | `0x1424_0000`, 64 KiB |
| Interrupt | GIC SPI 326, level-high (GSIV 358) |
| Reserved memory (DT, disabled) | `vpu_private` `0xBDE0_0000` 8 MiB; `vpu_protected` `0xBE60_0000` 24 MiB (both `no-map`, status disabled) |

- **[HW]** All engine interrupts share the **single** interrupt line.
  Per-LSID status is read from register `IRQVE` (01).
- **[POLICY]** The vendor driver requests the line as shared.
- **[HW]** The reserved-memory nodes are disabled on Sky1 and not used by
  the non-secure path. They belong to the protected (secure video) flow,
  which is out of scope.

## 0.3 Bus mastering, IOMMU and DMA addressing

- **[HW]** The VPU masters memory through its **own internal MMU only**.
  The Sky1 vendor DT gives it no `iommus` property, so no system SMMU sits
  in front of it. Page-table entries hold **physical** addresses.
- **[HW]** Physical addresses are **40 bits** wide. All memory that the VPU
  touches (page tables, firmware pages, message queues, user buffers) must
  be below 2^40.
- **[HW]** The VPU is **not cache-coherent** with the CPUs. The host must
  clean caches before the VPU reads and invalidate them after the VPU
  writes (02 §2.6).
- **[POLICY]** The vendor driver sets a 40-bit DMA mask, a maximum segment
  size of 2 GiB, and uses scatter-gather buffers (pages need not be
  physically contiguous).

> OPEN QUESTION: The per-LSID `STREAMID` register (01) suggests the block
> can emit SMMU stream IDs on other SoCs. Its use on Sky1 is unknown. The
> vendor driver never writes it.

## 0.4 Devicetree binding (vendor)

| Property | Value on Sky1 |
|---|---|
| `compatible` | `"armChina,linlon-v8"`. The driver also matches `arm,mali-mve`, `arm,mali-v500`, `arm,mali-v550`, `arm,mali-v61`, `armChina,linlon-v5/-v6/-v7` |
| `reg` | two entries: RCSU window, VPU registers |
| `reg-names` | `"vpu_rcsu"`, `"vpu"` |
| `interrupts` | one |
| `power-domains` | 6 entries: VPU top, cores 0–3 (SCMI device power domains), then the SCMI performance domain |
| `power-domain-names` | `"vpu_top"`, `"vpu_core0"` … `"vpu_core3"`, `"perf"` |
| `clocks` / `clock-names` | one clock, `"vpu_clk"` = SCMI clock `CLK_TREE_VPU_APBCLK` (67 in the public `cix,sky1.h` clock binding) |
| `resets` / `reset-names` | `"vpu_reset"` = S5 reset 14 (`SKY1_VPU_RESET_N`); `"vpu_rcsu_reset"` = reset 142 (`SKY1_VPU_RCSU_RESET_N`) |
| `busattr` (optional) | 4 × u32, AXI attributes per LSID (01 §1.4). Not present on Sky1 |

- **[HW]** The resources are found by **index**, not by name: memory 0 =
  RCSU, memory 1 = VPU registers, interrupt 0.
- **[POLICY]** The vendor driver never requests `vpu_rcsu_reset`. It only
  pulses `vpu_reset`.

## 0.5 ACPI attachment (vendor)

| Item | Value |
|---|---|
| `_HID` | `CIXH3010` |
| `_CRS` order | Memory32Fixed/QWord #0 = RCSU (`0x14230000`, 64 KiB), #1 = VPU registers (`0x14240000`, 64 KiB), Interrupt #0 = GSIV 358 |
| `CLKT` | one entry named `"vpu_clk"` (SCMI clock 67) |
| `RSTL` | entry named `"vpu_reset"` (format in `sky1-soc/02`) |
| `_DSD` | optional `busattr` integer package (4 entries) |
| Child devices | up to 4 children whose 4-character ACPI name starts with `"CRE"` (for example `CRE0`…`CRE3`): one per VPU core |

- **[HW]** Under ACPI there is no SCMI power-domain list. The VPU device
  object's own power state (`_PS0`/`_PS3` or power resources) controls
  VPU top. Each `CREn` child's power state controls core *n*. The vendor
  driver runtime-resumes and suspends these child devices to power cores
  on and off. It does not rely on the parent's power state to cascade
  ("ignore parent").
- **[HW]** The core power-on path in firmware must also perform the core's
  **SRAM memory repair** (see `sky1-soc/02` §2.5). A `CREn` child may
  expose a method named `REPR` that only performs memory repair. An
  alternative build of the vendor driver calls `REPR` instead of changing
  the child's power state.
- **[POLICY]** At most 5 power "domains" (top + 4 cores) are accepted. The
  top domain uses a 1000 ms runtime-PM autosuspend delay.

> OPEN QUESTION: No Sky1 DSDT was available. The `CREn` naming, the use of
> `REPR`, and whether `CREn` uses `_PS0`/`_PS3` or `_PR0` power resources
> are inferred from what the driver looks for. The public Sky1 ACPI-to-DT
> notes map `CIXH3010` to `cix,sky1-vpu` and give no other details.

## 0.6 Clocks, resets, power and performance domains

| Resource | Use |
|---|---|
| `vpu_clk` (SCMI clock 67, APB) | Must be on before any register access. The core clocks are not exposed to the host |
| `vpu_reset` (S5 reset 14) | Pulsed on every power-up (§0.7) |
| SCMI power domain 11 | VPU top |
| SCMI power domains 12–15 | VPU core 0–3 (core *n* = domain 12 + *n*) |
| SCMI performance domain 9 | VPU frequency (DT name `"perf"`) |
| RCSU `PGCTRL[0]` | Q-channel clock gating enable (`sky1-soc/02` §2.4.3) |
| RCSU `STRAP_PIN0[31:28]` | Per-core automatic power-off enable (ENPWOFF) |
| RCSU `STRAP_PIN2[7:4]` | Core harvesting fuses (1 = core fused off) |

- **[HW]** Core domain *n* must be on only for cores that are present
  (§0.7.1). Domains of harvested cores stay off.
- **[POLICY]** DVFS: the vendor driver attaches to the `"perf"` domain and
  copies its operating points. It runs an on-demand governor (100 ms poll,
  up-threshold 90, down-differential 5). The load is estimated from the
  session macroblock rate, with this table (load in macroblocks/s,
  frequency for 4 active cores; the frequency is scaled by 4/active cores
  and rounded up to the next table frequency):

| MB/s ≤ | Frequency | Rationale |
|---|---|---|
| 486 000 | 150 MHz | 1080p60 |
| 972 000 | 300 MHz | 1080p120 |
| 1 458 000 | 480 MHz | 1080p180 |
| 2 073 600 | 600 MHz | 4K60 |
| 4 147 200 | 800 MHz | 4K120 |
| 8 294 400 (and above) | 1200 MHz | 8K60 |

- **[POLICY]** The load of each session is
  ceil(w/16) × ceil(h/16) × fps. Encode counts double, VP8 encode a further
  ×4/3, and "legacy" decode formats (MPEG-2/4, H.263, VC-1, RV, AVS) ×8/3.

## 0.7 Power-up, power-down and core discovery

### 0.7.1 Core discovery and harvesting

- **[HW]** `NCORES` (`0x08`) is the number of implemented cores. `NLSID`
  (`0x0C`) is the number of LSIDs.
- **[HW]** The usable-core mask is
  `(~STRAP_PIN2 >> 4) & 0xF`, restricted to the lowest `NCORES` bits.
  If it is zero, no core is usable and the device must not be used.
- **[POLICY]** A module parameter can cap the active core count (default
  4). The mask is then trimmed to that many lowest set bits.

### 0.7.2 Power-up sequence (every runtime resume)

1. Turn on VPU top power (DT: domain 11; ACPI: the device itself).
2. Enable `vpu_clk`.
3. Assert `vpu_reset`, wait 10–20 µs, deassert it.
4. Set RCSU `PGCTRL` bit 0 (enable Q-channel clock gating).
5. On the first power-up only: read `HARDWARE_ID`, `SVNREV`, `FUSE`,
   `NCORES`, `NLSID` and `STRAP_PIN2`, and compute the core mask (§0.7.1).
6. Turn on the core power domain of every usable core (DT: domain 12+*n*;
   ACPI: `CREn`). Turning a core on must include its memory repair.
7. If `SVNREV` = `0xE0C1AFE1`, do the ENPWOFF step with "auto power-off
   only for absent cores" (§0.7.3).
8. Write `BUSCTRL` (01 §1.2).
9. Resume session scheduling. Any LSIDs that were in use must be set up
   again (03 §3.7). LSID registers do not survive power-off.

- **[HW]** The reset in step 3 powers the cores off. Memory repair is
  therefore only valid if it happens **after** that reset, which is why
  the core domains come up after it. Each core needs memory repair after
  every power-on.

### 0.7.3 ENPWOFF step (`SVNREV` = `0xE0C1AFE1` only)

1. Read-modify-write RCSU `STRAP_PIN0`: keep bits [27:0] and set [31:28] to
   *v*. On power-up, *v* = `~core_mask & 0xF`: present cores stay powered,
   absent cores may power off. On power-down, *v* = `0xF`: every core
   powers off automatically.
2. Write `RESET` (`0x50`) = 1 (engine soft reset) so the new straps take
   effect.
3. Write `CLKFORCE` (`0x24`) = 0. A forced clock or power bit would stop
   automatic power-off.

- **[POLICY]** With firmware profiling enabled the vendor driver writes
  `CLKFORCE` = `1 << 8` (scheduler clock forced on) instead of 0.

### 0.7.4 Power-down sequence (runtime suspend)

1. Ask every resident session that has no frames in progress to switch out
   (03 §3.7). Stop the scheduler. Wait up to 1000 ms **[POLICY]** for all
   LSIDs to become idle.
2. Detach every session from its LSID (03 §3.6.3).
3. Disable the interrupt and cancel any deferred interrupt work.
4. Clear RCSU `PGCTRL` bit 0.
5. If `SVNREV` = `0xE0C1AFE1`: pulse `vpu_reset` (10–20 µs), then do the
   ENPWOFF step with *v* = `0xF`.
6. Disable `vpu_clk`.
7. Turn off the usable cores' power domains, then (DT) release VPU top
   (autosuspend).

- **[POLICY]** System sleep reuses the runtime suspend/resume paths.

## 0.8 Interrupt handling summary

- **[HW]** On interrupt, read `IRQVE` (`0x18`). Bit *n* set means LSID *n*
  has signalled the host. For each set bit, write 0 to that LSID's
  `LIRQVE` register to acknowledge it, then process that LSID's session:
  serve any RPC request, then drain its message queue (03).
- **[POLICY]** The vendor driver acknowledges in hard-IRQ context and does
  the processing in an unbound work queue with one worker.

## 0.9 Firmware files

| File | Session |
|---|---|
| `h264dec.fwb`, `hevcdec.fwb`, `vp8dec.fwb`, `vp9dec.fwb`, `av1dec.fwb`, `mpeg2dec.fwb`, `mpeg4dec.fwb` (also used for H.263), `vc1dec.fwb`, `rvdec.fwb`, `avsdec.fwb`, `avs2dec.fwb`, `jpegdec.fwb` | Decoders |
| `h264enc.fwb`, `hevcenc.fwb`, `vp8enc.fwb`, `vp9enc.fwb`, `jpegenc.fwb` | Encoders |

- **[HW]** The name is `<codec><dec|enc>.fwb`. The direction is that of the
  compressed port: compressed input means `dec`, compressed output means
  `enc`.
- **[HW]** The files are shipped by the Sky1-Linux `sky1-firmware` package.
  Their format and loading are in 03 §3.1–§3.2.
- **[POLICY]** Each binary is loaded once and shared by all sessions that
  use it. The vendor loader uses the asynchronous kernel firmware loader
  with uevent fallback.
