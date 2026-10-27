# 00: Overview, Platform Resources and Attachment

## 0.1 Architecture

The Zhouyi v3 NPU is built in layers. The terms below are used throughout
this spec.

| Term | Meaning |
|---|---|
| **TEC** | The smallest execution engine. Each core has 1–15 TECs. |
| **AIFF** | A fixed-function accelerator inside each core. There are 1–15 per core. |
| **Core** | An AIPU core. It contains TECs and AIFFs. |
| **Cluster** | 1–15 cores plus a shared on-cluster memory called **GM** (global memory). A device has at most 8 clusters. |
| **Partition** | A group of clusters that share a power/compute domain and one command pool. There are at most 4. |
| **TSM** | The task scheduling manager. It is the host-facing front end that takes command pools, walks TCB chains in memory and dispatches tasks to cores. |
| **Command pool** | A TSM work queue. The host creates, dispatches, aborts and destroys pools with commands. A device has at most 8 (TSM build info says how many). |
| **TCB** | Task control block. A fixed 128-byte record in memory. A job is a chain of TCBs (02 §2.4). |

- **[HW]** All partitions share one register window, one interrupt line
  and the TSM. The partition is a logical grouping that the host sets up
  in the cluster control registers (01 §1.4).
- **[HW]** Cluster, core, AIFF and TEC counts are not fixed. They are read
  from the per-cluster configuration register (01 §1.4). The partition
  and command-pool limits are read from TSM build info (01 §1.2).
- **[HW]** *(vendor)* On Sky1 the vendor DT configures **one cluster
  (cluster 0) in one partition (partition 0)**. The comment on that
  property calls the part "x2". The SoC has three NPU-core power domains
  (§0.5), so cluster 0 presumably holds **3 cores**.

> OPEN QUESTION: No boot log or register dump was available, so the real
> core count, AIFF count and TEC count that cluster 0 reports on Sky1 are
> unknown. The driver logs them at probe ("Core Count per Cluster",
> "TEC Count per Core"). A dump of register `0xC00` on real hardware
> would settle this. The Sky1-Linux docs quote 30 TOPS for the NPU.

- **[HW]** The ISA version register identifies the part as Zhouyi v3 by
  revision value (01 §1.2). The same driver also supports v1, v2 and v3_1
  parts, but the Sky1 kernel configuration builds only the v3 support.
  This spec covers v3 only.

## 0.2 MMIO and interrupt

| Resource | Value | Source |
|---|---|---|
| Register window | `0x14260000`, size `0x10000` (64 KiB) | DT `reg`; ACPI `_CRS` memory descriptor 0 |
| Highest register offset used | `0x322C` | driver bound on debug register access |
| Interrupt | GIC **SPI 327**, level-high (ACPI GSIV **359** = SPI + 32) | DT `interrupts`; ACPI `_CRS` interrupt 0 |

- **[HW]** There is one interrupt for the whole NPU (all partitions and
  command pools). The handler finds the source by reading each command
  pool's interrupt status (02 §2.6).
- **[POLICY]** The vendor driver requests the line as shared.

## 0.3 Clocks and resets

**[HW]** The NPU node names **no clocks and no resets**, in DT or ACPI.
The vendor driver never touches a clock or reset line. SCMI clock IDs for
the NPU exist in the public dual-licensed clock header but nothing uses
them:

| SCMI clock ID | Clock |
|---|---|
| 64 | NPU memory clock |
| 65 | NPU system clock |
| 66 | NPU debug clock |

Reset lines exist in the S5 reset controller (sky1-soc 02, Annex A):

| S5 index | Line | Word | Bit |
|---|---|---|---|
| 10 | NPU top | `0x400` | 7 |
| 11 | NPU core 0 | `0x400` | 8 |
| 12 | NPU core 1 | `0x400` | 9 |
| 13 | NPU core 2 | `0x400` | 10 |
| 123 | NPU RCSU | `0x804` | 0 |

- **[HW]** *(vendor)* NPU frequency is set only through the SCMI
  performance domain (§0.6). Reset release is assumed to happen as part of
  power-domain power-on in firmware.

> OPEN QUESTION: Whether firmware (TF-A for DT power domains, AML power
> resources under ACPI) releases the five NPU reset lines on power-on, or
> whether an OS must deassert them, is not shown. The vendor driver works
> without touching them. An OS that sees the TSM soft reset (02 §2.1) fail
> should try deasserting these lines (active-low, write 1).

> OPEN QUESTION: No source gives an MMIO base for an NPU RCSU, even though
> an NPU RCSU reset line exists. `0x14270000` is the camera CSI RCSU, not
> the NPU's. No NPU clock-gating, harvesting or DMRP (memory repair)
> register was found anywhere. See 01 §1.8.

## 0.4 SMMU and stream ID

| Item | Value |
|---|---|
| SMMU | The multimedia-hub SMMUv3 at `0x0B1B0000` (`smmu_mmhub` in vendor DT) |
| Stream ID | `0x1E` |
| SMMU PMU (TBUs) | NPU TBU0 PMCG pages `0x0B3D2000`, `0x0B3E2000` (SPI 182); NPU TBU1 PMCG pages `0x0B3F2000`, `0x0B402000` (SPI 186). Both disabled in vendor DT |

- **[HW]** *(vendor)* The NPU node in vendor DT has no `dma-coherent`
  property, but the SMMU node does. The driver treats NPU memory as
  non-coherent and gives userspace explicit cache-maintenance ioctls
  (03 §3.6).
- **[HW]** All device addresses the NPU emits (TCB pointers, buffer
  addresses) are **IOVAs** when the SMMU is in use. The driver works
  with and without an IOMMU. Its memory layout differs between the two
  (02 §2.8).

> OPEN QUESTION: Under ACPI the SMMU mapping comes from IORT, which was
> not available. Whether the ACPI tables put the NPU behind the SMMU (with
> stream ID `0x1E`) or leave it un-translated is not known. The driver
> decides at run time from whether the OS gave the device an IOMMU group.

> OPEN QUESTION: Whether NPU traffic is I/O-coherent through the SMMU is
> not stated. Assume it is not.

## 0.5 Power domains

**[HW]** In DT boot the NPU consumes three SCMI-over-SMC power domains
(sky1-soc 01 §1.3.3), with names that matter for lookup:

| DT `power-domain-names` entry | SCMI power domain | Meaning |
|---|---|---|
| `pd_core0` | 16 | NPU core 0 |
| `pd_core1` | 17 | NPU core 1 |
| `pd_core2` | 18 | NPU core 2 |
| `perf` | perf domain 8 (on the SCMI DVFS provider, not the power provider) | NPU DVFS |

- **[HW]** All three core domains must be on before the first register
  access, including the identification read (02 §2.2).
- **[HW]** The NPU node does not reference domain 19 ("NPU top").

> OPEN QUESTION: Domain 19 (NPU top) is never requested. Firmware may
> turn it on as the parent of 16–18, or it may simply be on after boot.
> An OS that cannot reach the TSM after powering 16–18 should also power 19.

### 0.5.1 Core mask property

**[POLICY]** *(vendor)* An optional integer property `core_mask` (DT
value `3`) controls how many core domains are used:

| `core_mask` | Behaviour |
|---|---|
| absent | Treated as 3 |
| `1` | Only core-domain 0 is powered and used |
| `0` or `2` | The driver binds but does nothing (NPU unused) |
| anything else (including `3`) | All three core domains are used |

> OPEN QUESTION: The encoding is odd (a value of 3 enables three
> domains, and 2 disables the NPU). It may be tied to the NPU harvesting
> fuse. The vendor nvmem node has a cell named `npu_harvest_0` at fuse
> byte offset 1, bit 7, width 1, which no driver reads. How the fuse
> relates to `core_mask` and to the cluster's reported core count is
> unknown.

## 0.6 DVFS

- **[HW]** NPU frequency is SCMI performance domain **8** (sky1-soc 01
  §1.2.3). Firmware reports the operating points. Under ACPI the
  Sky1-Linux notes report about **72 MHz to 1 GHz**.
- **[HW]** There is no NPU clock for the OS to program directly.
  Frequency changes are SCMI `PERFORMANCE_LEVEL_SET` on domain 8.
- **[POLICY]** *(vendor)* The driver copies the firmware's operating
  points into a frequency table, registers a devfreq device with the
  **userspace** governor and a 50 ms polling interval, and sets the rate
  through the perf domain when asked. With the userspace governor nothing
  changes the frequency unless an administrator does.
- **[POLICY]** The perf domain is kept "active" whenever the NPU is
  runtime-active, through a runtime-PM dependency.

### 0.6.1 DVFS under ACPI

**[ABI]** *(vendor)* Under ACPI the NPU device's `_DSD` must carry:

| `_DSD` property | Content |
|---|---|
| `power-domain-names` | A string list containing `"perf"` |
| `power-domains` | A reference list. The entry at the same index as `"perf"` is (reference to the SCMI DVFS protocol node, domain index), with the index cells counted by that node's `#power-domain-cells` |

- **[HW]** *(vendor)* The referenced provider node is a child of the SCMI
  (`CIXHA006`) ACPI device. Its `reg` property equals the SCMI protocol
  number (`0x13` for performance). The Sky1-Linux patches call the path
  `\_SB.SCMI.DVFS`. The child may report `_STA` = 0 (a namespace-only
  stub), so an OS must look up children whether or not they are present.
  The domain index argument is 8.

> OPEN QUESTION: This contradicts sky1-soc 01 §1.2.2, which says the ACPI
> SCMI device has no per-protocol children. The Sky1-Linux DVFS patches
> rely on such a child existing for the DVFS protocol. The exact AML was
> not seen. Section 01 should be revisited.

> OPEN QUESTION: Under ACPI the generic "attach the device's own ACPI PM
> domain" step happens first, so the NPU device already has a power-domain
> owner. The driver works around this by attaching the perf domain by
> index. An OS without this conflict can just resolve the reference.

## 0.7 ACPI attachment

| Item | Value |
|---|---|
| `_HID` | `CIXH4000` |
| `_CRS` | Memory descriptor 0: register window (§0.2). Interrupt descriptor 0: GSIV 359, level-high |
| `_DSD` (read by the driver) | `core_mask` (u32, §0.5.1); `cluster-partition` (u32 pairs, §0.8); `gm-policy` (u32, §0.9); `core-id` (u32, default 0, unused on v3); `power-domain-names` and `power-domains` (§0.6.1) |
| CLKT, RSTL | None used by the NPU driver |
| Children | ACPI device objects whose 4-character name starts with **`CRE`** (for example `CRE0`–`CRE2`), one per NPU core power domain |

- **[HW]** *(vendor)* Each `CRE*` child carries the ACPI power resources
  for one NPU core domain. The driver finds the first *N* such children
  (N = 3, or 1 with `core_mask` = 1), skipping data-only nodes. It turns
  each one to D0 before touching registers and back to D3 on runtime
  suspend. The children are enumerated as ordinary platform devices, so
  they have their own `_HID`.
- **[POLICY]** The children are set to ignore their parent's power state,
  and the parent ignores its children, so each core domain is controlled
  only through its own `CRE*` device.

> OPEN QUESTION: The `_HID` of the `CRE*` children and the contents of
> their `_PR0`/`_PS0`/`_PS3` methods are unknown (no DSDT). Whether those
> methods also run memory repair (DMRP) for the NPU SRAMs, as the audio
> power resource does (sky1-soc 02 §2.5), is unknown.

> OPEN QUESTION: Under ACPI, whether `cluster-partition` and `gm-policy`
> are present in `_DSD` is unknown. Without them the driver falls back to
> one cluster in partition 0 and the "shared" GM policy, which matches
> the DT values.

## 0.8 Devicetree attachment

**[ABI]** *(vendor)* Vendor DT node (`npu: aipu@14260000`), properties the
driver reads:

| Property | Sky1 value | Meaning |
|---|---|---|
| `compatible` | `"armchina,zhouyi"` | Also matched: `armchina,zhouyi-v1`, `-v2`, `-v3` |
| `reg` | `0x14260000 0x10000` | §0.2 |
| `interrupts` | SPI 327, level-high | §0.2 |
| `power-domains` / `power-domain-names` | §0.5 | |
| `iommus` | (`smmu_mmhub`, `0x1E`) | §0.4 |
| `core_mask` | `3` | §0.5.1 |
| `cluster-partition` | `<0 0>` | Pairs of (cluster ID, partition ID). Absent → one cluster in partition 0 |
| `gm-policy` | `1` | 0 = no GM, 1 = GM shared by all QoS levels, 2 = GM split in halves between slow and fast QoS. Absent or > 2 → 1 |
| `memory-region` | (not present) | Used only when there is no IOMMU (§0.9) |

## 0.9 Reserved memory

- **[HW]** *(vendor)* Vendor DT defines a 512 MiB `shared-dma-pool`
  `aipu_res_0` at `0x90000000` (no-map), and the Orion O6 board file
  enables it. The NPU node does **not** reference it with `memory-region`.
  With the SMMU enabled the driver allocates its working memory through
  the IOMMU and never uses this pool.
- **[POLICY]** Without an IOMMU the driver uses only the regions listed in
  `memory-region`. Each one is a reserved-memory node named `memory`,
  `sram` or `dtcm` with an optional `asid` list and `host-aipu-offset`. On
  v3 `sram` regions are skipped. 02 §2.8 has the address rules.

> OPEN QUESTION: Whether ACPI provides an equivalent of `aipu_res_0`
> (for example through RSVL) is unknown. The pool seems to be a remnant
> of a non-IOMMU configuration.

## 0.10 Thermal

**[HW]** *(vendor)* SCMI sensor 7 is the NPU temperature. The vendor DT
NPU thermal zone has passive trips at 60 °C and 95 °C (1 °C hysteresis)
and no cooling map. The NPU driver has no thermal hooks.
