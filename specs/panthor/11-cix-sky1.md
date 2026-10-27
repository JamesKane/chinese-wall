# 11: CIX Sky1 (Immortalis-G720) Integration

This section covers what changes when the target is the **Arm
Immortalis-G720 in the CIX Sky1 SoC** (used on the Radxa Orion O6 and
Orange Pi 6 Plus boards) instead of the RK3588 Mali-G610 that sections 00–10 were written
around. It has two parts:

- **11.1–11.3:** what is different about an arch 12 GPU, as a checklist
  against the arch-generic text in 02–07.
- **11.4–11.10:** Sky1 platform integration: resources, ACPI probe,
  power-up sequence, core harvesting, coherency and memory attributes,
  system-cache allocation, DVFS.

Mainline has no Sky1 GPU node at the baseline. Sections 11.4 onward come
from the CIX vendor devicetree and the Sky1-Linux Panthor patches (CIX
vendor code adapted to Panthor), and are marked *(vendor)* where the fact
comes only from there.

## 11.1 GPU identification

| Property | Value | Tag |
|---|---|---|
| Arch major | 12 | [HW] |
| Product major | 0 | [HW] |
| Product name rule | (12, 0) is reported as Immortalis-G720 when the shader-present mask has **≥ 10 bits set** and `GPU_FEATURES` bit 2 (ray intersection) is set. Otherwise Mali-G720 (≥ 6 cores) or Mali-G620 (02 §2.3.1) | [POLICY] (logging only) |
| Expected core count | 10 (the SoC vendor markets the part as "MC10") | public product data, not in source |
| Ray tracing | `GPU_FEATURES` bit 2 expected set on the Immortalis part | [HW] |
| Firmware image | `arm/mali/arch12.8/mali_csffw.bin` | [HW] path rule (05 §5.1) |

- **[HW]** The driver must not hard-code any of the above. Read `GPU_ID`,
  `SHADER_PRESENT` and `GPU_FEATURES` (03 §3.1) and derive everything from
  them. The table is for bring-up sanity checks.
- **[ABI]** `GPU_FEATURES` is passed to userspace unchanged (10). Do not mask
  the ray bits.

- **[HW]** *(vendor)* The Sky1 firmware package installs the GPU image as
  `arm/mali/arch12.8/mali_csffw.bin`, stating the part is arch 12.8. So
  `GPU_ID.ARCH_MINOR` = 8 is expected. (The package also places symlinks at
  `arm/mali/mali_csffw.bin` and `mali_csffw.bin`.) The image is Arm
  proprietary, redistributable.

> OPEN QUESTION: The VERSION_MAJOR/MINOR/STATUS fields of `GPU_ID` (silicon
> revision) for Sky1 are unknown. No revision-specific workaround exists in
> the driver for any arch 12 revision.

## 11.2 What does not change from the RK3588 path

Arch 12 sits in the 10–13 group everywhere the driver branches on
architecture. A clean-room driver that already handles the Mali-G610 needs
**no new code paths** for the G720 in these areas:

| Area | Arch 12 behavior | See |
|---|---|---|
| Power control | Legacy GPU-control `L2_PWRON`/`L2_PWROFF` registers. Host powers only the L2. Firmware powers shader and tiler cores. No PWR block at `0x0800` | 03 §3.4 |
| Interrupts | Three lines (job, MMU, GPU). The PWR interrupt block does not exist, so nothing else shares the `"gpu"` line | 01 §1.1.3, 03 §3.8 |
| Soft reset | Same `GPU_CMD` soft reset and `RESET_COMPLETED` wait | 03 §3.3 |
| MMU | Same AS register layout, page-table format, LOCK/UNLOCK and fault decoding. VA and PA widths come from `MMU_FEATURES` | 04 §4.12 |
| Firmware image format and MCU boot | Same image header, entry types and boot sequence | 05 |
| CSF interface | Same GLB/CSG/CS layout. Version-dependent fields keyed on the **interface version**, not the GPU arch | 06 §6.2.3 |
| Scheduling, heaps, BOs, ioctls | No arch-dependent behavior | 07–10 |

**[HW]** The only arch-gated register step that *becomes available* on arch
12 (it is skipped on arch 10) is the custom L2 address hash (`ASN_HASH[0..2]`
plus `L2_CONFIG` bit 24, 03 §3.4.1 step 2). Sky1 does not program it
(11.8.3).

## 11.3 Firmware interface version on arch 12

- **[HW]** The arch 12 firmware is a separate image from arch 10. A driver
  must read `GLB_VERSION` after boot and select the version-dependent layout
  from 06 §6.2.3, not assume the layout used with the RK3588 image.
- In particular, if the arch 12.8 image reports interface ≥ 4.0.0, the CSG
  endpoint request must go to the 64-bit `CSG_EP_REQ_64` field. If it reports
  ≥ 4.1.0, MCU halt must use the `GLB_REQ.STATE` field, not `GLB_REQ.HALT`
  (06 §6.4.5). Getting either wrong leaves groups unscheduled or the MCU not
  halting on suspend.

> OPEN QUESTION: The actual `GLB_VERSION` reported by the arch 12.8 firmware
> image is not recorded in the source. A driver written to support every
> layout listed in 06 §6.2.3 does not need to know it in advance.


## 11.4 Resources and attachment

| Item | Value |
|---|---|
| ACPI `_HID` | `CIXH5000` |
| DT compatible *(vendor)* | `"arm,mali-valhall"` (not the mainline `arm,mali-valhall-csf` binding) |
| MMIO 0: GPU RCSU | `0x15000000`, 64 KiB (`reg-names` `"gpu_rcsu"`) |
| MMIO 1: GPU registers | `0x15010000`, `0x480000` bytes (`"gpu"`) |
| Interrupts (GIC SPI / ACPI GSIV, level-high) | JOB 237 / 269, MMU 238 / 270, GPU 239 / 271 |
| Power domain | SCMI power domain 21 (TF-A SMC agent, Sky1 SoC spec 01 §1.3) |
| Performance domain | SCMI perf domain 0, firmware name `"gpu_core"` (mailbox agent) |
| IOMMU | None in the vendor DT (no `iommus`); Sky1-Linux treats it as not IOMMU-mapped |
| Protected memory *(vendor)* | Reserved region at `0xBCE00000` for the vendor kbase driver's protected mode; not used by Panthor |

- **[HW]** Resource order matters: **resource 0 is the RCSU**, resource 1
  the GPU. This holds under DT and ACPI.
- **[HW]** Interrupt names: the vendor DT uses upper-case `"JOB"`, `"MMU"`,
  `"GPU"` (mainline binding: lower-case). Under ACPI the interrupts are
  **positional**: `_CRS` interrupt 0 = JOB, 1 = MMU, 2 = GPU.
- **[POLICY]** *(vendor)* Sky1-Linux detects Sky1 by the `CIXH5000` HID
  under ACPI or the `arm,mali-valhall` compatible under DT, and applies all
  Sky1 steps below only then.

### 11.4.1 Clocks

| Vendor name | SCMI clock | Mainline-binding name | Use |
|---|---|---|---|
| `gpu_clk_core` | `CLK_TREE_GPU_CLK_CORE` (31) | `core` (or unnamed first clock) | Required |
| `gpu_clk_stacks` | `CLK_TREE_GPU_CLK_STACKS` (32) | `stacks` | Optional |
| `gpu_clk_200M` | `CLK_TREE_GPU_CLK_200M` (272) | — | Optional "backup" clock, enabled with the others |
| `gpu_clk_400M` | `CLK_TREE_GPU_CLK_400M` (30) | — | Optional "backup" clock |

- **[HW]** Under ACPI the clock names come from the DSDT `CLKT`, which uses
  the vendor names (Sky1 SoC spec 01 §1.4).
- **[POLICY]** Do not set GPU clock rates directly; DVFS goes through the
  performance domain (11.10).

> OPEN QUESTION: What the 200 MHz and 400 MHz "backup" clocks feed is not
> stated. Enabling them with the core clock is the safe choice.

### 11.4.2 Resets

| Name | S5 index | Word / bit |
|---|---|---|
| `"gpu_reset"` | 9 (`SKY1_GPU_RESET_N`) | `0x400` bit 6 |
| `"gpu_rcsu_reset"` | 119 (`SKY1_GPU_RCSU_RESET_N`) | `0x800` bit 28 |

- **[HW]** Active-low (Sky1 SoC spec 02 §2.1). Only `gpu_reset` is pulsed
  by the driver (11.5). `gpu_rcsu_reset` is listed but never toggled.
- **[POLICY]** Under ACPI the resets come from the DSDT `RSTL` and are
  treated as optional: if absent, the reset pulse is skipped.

## 11.5 Power-up sequence

**[HW]** *(vendor, attributed to the CIX GPU development guide)* Before any
GPU register access:

1. **Power domain on.** SCMI `POWER_STATE_SET` domain 21, state 0 (on).
   This also configures the interconnect's non-secure access permissions
   for the GPU. Under DT the SMC SCMI agent does it. Under ACPI the DSDT has
   no SCMI provider, so issue the SMC call directly (Sky1 SoC spec 01
   §1.3.2).
2. **Clocks on** (11.4.1).
3. **Assert `gpu_reset`.** Wait 10–20 µs.
4. **Release `gpu_reset`.**
5. **Enable Q-channel clock gating:** set bit 0 of RCSU `PGCTRL`
   (RCSU + `0x218`), read-modify-write.

Then continue with the generic sequence (GPU soft reset, 03 §3.3).

- **[POLICY]** Linux registers the SCMI power-domain provider early on
  Sky1 because step 1 must precede the first GPU register access, and the
  access-permission setup is part of it.

## 11.6 Core harvesting

**[HW]** Some shader cores may be fused off. The GPU's own
`SHADER_PRESENT` does not reflect that; the RCSU does:

1. Read v = RCSU + `0x304`.
2. available = NOT((v AND `0x0FFFFFF0`) >> 4) AND `0x00550555`.
3. Use `SHADER_PRESENT` AND available as the shader-present mask everywhere
   (core power masks, userspace queries).

`0x00550555` is the full part's shader-present pattern: 10 cores at bit
positions 0, 2, 4, 6, 8, 10, 16, 18, 20, 22. A set bit in the fuse field
`[27:4]` removes the core at that bit position.

- **[ABI]** Userspace sees the reduced mask as the shader-present value.

## 11.7 Coherency and memory attributes

- **[HW]** The ACPI DSDT declares the GPU **coherent** (`_CCA` = 1). That is
  **wrong** for how Sky1 shares memory: the display processor (DPU) is a
  non-snooping master that reads scanout buffers straight from the system
  cache / DRAM. With CPU-side DMA treated as coherent, cache maintenance is
  skipped and the display shows stale tiles.
- **[HW]** Treat the GPU as **non-coherent for CPU DMA purposes** (do cache
  maintenance, map buffers write-combined) regardless of `_CCA`.
  Sky1-Linux overrides `_CCA` for `CIXH5000` when the MCFG table has OEM ID
  `CIXTEK` and OEM table ID `SKY1EDK2`. The vendor DT has
  `system-coherency = <0>` and no `dma-coherent`.
- **[HW]** *(vendor)* GPU bus protocol: program `COHERENCY_PROTOCOL` = 0
  (**ACE-Lite**), not 31 (none). With ACE-Lite, GPU L2 evictions go through
  the interconnect's home nodes into the system-level cache, where the DPU
  can see them. If `COHERENCY_FEATURES` reports the shareable-cache bit
  (bit 5), also set bit 5 in `COHERENCY_PROTOCOL`. The GPU hardware also
  supports full ACE, but Sky1 does not use it.
- **[HW]** *(vendor)* Memory attributes: because the GPU has **no IOMMU**
  to override attributes and is not fully coherent, map GPU memory as
  **inner and outer non-cacheable**, with the GPU's internal-shareable
  ("Midgard inner") shareability and no explicit allocation hints. This
  replaces the cached attribute the generic driver would derive from the
  page-table MAIR (04 §4.5). ACE-Lite alone does not guarantee the DPU sees
  GPU writes in time.
- **[HW]** Page-table walks are not coherent: write page tables with CPU
  cache maintenance (as for the non-coherent case in 04).

## 11.8 System-level cache allocation (PBHA)

### 11.8.1 Registers

These are arch-12 GPU-control registers that the generic driver does not
use:

| Offset (GPU block) | Name | Meaning |
|---|---|---|
| `0x320 + 4n`, n = 0–3 | `SYSC_PBHA_OVERRIDE<n>` | Page-based hardware attribute override |
| `0x340 + 4n`, n = 0–7 | `SYSC_ALLOC<n>` | System-cache allocation policy, one byte per AXI ID: byte k of `SYSC_ALLOC<n>` governs ID 4n + k |

### 11.8.2 Sky1 values

**[HW]** *(vendor)* Before powering the L2 (03 §3.4.1), write:

| Register | Value |
|---|---|
| `SYSC_PBHA_OVERRIDE3` | `0x22000000` |
| `SYSC_ALLOC0` | `0x00230000` |
| `SYSC_ALLOC1` | `0x00000023` |
| `SYSC_ALLOC2` | `0x00000000` |
| `SYSC_ALLOC3` | `0x00000000` |
| `SYSC_ALLOC4` | `0x00523222` |
| `SYSC_ALLOC5` | `0x00523200` |
| `SYSC_ALLOC6` | `0x00000022` |
| `SYSC_ALLOC7` | `0x00000032` |

The same allocation values appear in the vendor devicetree as
`pbha { int_id_override = <2 0x23>, <4 0x23>, <16 0x22>, <17 0x32>,
<18 0x52>, <21 0x32>, <22 0x52>, <24 0x22>, <28 0x32>; }`: pairs of
(AXI ID, policy byte), which pack into the `SYSC_ALLOC` words above.

**[HW]** *(vendor)* Also write, for the performance counters of the
load/store memory events:

| Offset | Value |
|---|---|
| `0x3068` | `0x00000001` |
| `0x306C`, `0x3070`, `0x3074`, `0x307C` | `0xFFFFFFFF` |

> OPEN QUESTION: The meaning of each policy byte (0x22, 0x23, 0x32, 0x52)
> and of `SYSC_PBHA_OVERRIDE3` is not documented. They are taken as opaque
> platform tuning. Omitting them is expected to cost performance (system
> cache use), not correctness, but this is not proven.

> OPEN QUESTION: Which block the `0x3068`–`0x307C` registers belong to is
> not named. They only affect counters; they can be skipped.

### 11.8.3 Address hash

**[POLICY]** No Sky1 code programs `ASN_HASH` / `L2_CONFIG` bit 24. Use the
GPU's built-in hash.

## 11.9 Core groups and L2

- **[POLICY]** The driver powers only bit 0 of `L2_PRESENT` (03 §3.4.1).
- A single-bit `L2_PRESENT` is expected; no Sky1 code handles more than one
  L2 slice.

> OPEN QUESTION: Not confirmed on hardware by any source read. Log
> `L2_PRESENT` and `SHADER_PRESENT` at bring-up.

## 11.10 DVFS

- **[HW]** Frequency and voltage are owned by the power-management
  processor. The OS requests levels on SCMI performance domain 0
  (`"gpu_core"`) with `PERFORMANCE_LEVEL_SET`. No clock rate is set and no
  regulator is driven. (The vendor DT's `gpu-microvolt = <820000>` is for
  the vendor kbase driver's power model only.)
- **[HW]** Levels are discovered with `PERFORMANCE_DESCRIBE_LEVELS`.
  Reported by Sky1-Linux: under ACPI 6 levels from 72 MHz to 1000 MHz;
  the vendor guide lists 350, 600, 800 and 1000 MHz as normal levels, plus
  72 MHz (2 cores active) and 216 MHz (6 cores active) as **dynamic core
  scaling** levels.
- **[HW]** The two lowest levels are only valid if the driver also limits
  active shader cores to 2 or 6. A driver without core scaling must not
  select them.
- **[HW]** *(vendor)* Firmware stability: the PM firmware serves all 12
  performance domains over one mailbox channel and falls behind under
  frequent requests; the SCMI channel can then lock up for every user
  (clocks and resets included). Firmware advertises fast channels but
  rejects `DESCRIBE_FASTCHANNEL`. Therefore:
  - **[POLICY]** do not resend a level that is already set;
  - **[POLICY]** send at most one level change per 100 ms;
  - **[POLICY]** on errors back off exponentially (0.5 s up to 4 s); after
    3 consecutive failures, move to a mid-range level;
  - **[POLICY]** use a governor sampling period of 100 ms.
- **[HW]** Performance domain 1 (GPU top) is not used by any Sky1 driver.

> OPEN QUESTION: Whether the 72/216 MHz levels are included in what
> firmware reports to the OS (the ACPI report says 6 levels, which suggests
> yes) must be checked, and those levels filtered out unless core scaling
> is implemented.

## 11.11 Bring-up checklist

1. Power domain 21 on (SMC SCMI), clocks on, pulse `gpu_reset`, set RCSU
   `PGCTRL` bit 0 (11.5).
2. Read `GPU_ID` (expect arch 12.8, product major 0), `SHADER_PRESENT`, and
   apply the RCSU harvesting mask (11.6).
3. `COHERENCY_PROTOCOL` = ACE-Lite (+ bit 5 if supported). CPU-side:
   non-coherent DMA, write-combined mappings. GPU memory attributes:
   non-cacheable (11.7).
4. Program `SYSC_PBHA_OVERRIDE3` and `SYSC_ALLOC0–7` (11.8), then power the
   L2 and load `arm/mali/arch12.8/mali_csffw.bin`.
5. Read `GLB_VERSION` and use the matching interface layout (11.3).
6. DVFS last, rate-limited, without the core-scaling levels (11.10).
