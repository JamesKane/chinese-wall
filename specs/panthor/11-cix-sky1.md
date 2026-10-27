# 11: CIX Sky1 (Immortalis-G720) Integration

This section covers what changes when the target is the **Arm
Immortalis-G720 in the CIX Sky1 SoC** (used on the Radxa Orion O6 and
Orange Pi 6 Plus boards) instead of the RK3588 Mali-G610 that sections 00–10 were written
around. It has two parts:

- **11.1–11.3:** what is different about an arch 12 GPU, as a checklist
  against the arch-generic text in 02–07.
- **11.4–11.8:** Sky1 platform resources (power domains, performance
  domains, clocks, resets) and the gaps that the baseline source does not
  fill.

> **Status: partial.** At the source baseline, mainline has **no Sky1 GPU
> devicetree node, no binding entry for a Sky1 compatible, and no Sky1 entry
> in the driver's SoC match table**. Everything below about the GPU block
> itself (MMIO base, interrupt numbers, which clocks and domains it consumes)
> is therefore an OPEN QUESTION. What *is* fixed is the set of Sky1 platform
> identifiers the GPU will have to use, and the generic driver behavior that
> applies once the node exists.

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

> OPEN QUESTION: The arch minor of the Sky1 part is not stated anywhere in the
> baseline source. The firmware path above assumes `GPU_ID.ARCH_MINOR` = 8,
> which is the only arch 12 image the driver declares. If the silicon reports
> a different minor, no matching image is known and probe fails (05 §5.1).

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
plus `L2_CONFIG` bit 24, 03 §3.4.1 step 2). Whether Sky1 needs it is open
(11.6).

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

## 11.4 Devicetree status

- **[HW]** The binding (`arm,mali-valhall-csf`) requires a SoC-specific string
  before the generic one, and lists only `mediatek,mt8196-mali`,
  `nxp,imx95-mali` and `rockchip,rk3588-mali`. A Sky1 node needs a new SoC
  string (presumably `cix,sky1-mali`, not yet defined) followed by
  `"arm,mali-valhall-csf"`.
- **[POLICY]** Because the driver also matches the generic string with no
  per-SoC data, a Sky1 node with any SoC string plus the generic string would
  bind with default behavior: no ASN hash, no special quirks.
- The binding already permits what Sky1 is likely to need: up to 5
  `power-domains` with names, 1–3 clocks named `"core"`, `"coregroup"`,
  `"stacks"`, and optional `dma-coherent`.

> OPEN QUESTION: MMIO base and size, the three GIC SPI numbers and their
> trigger type, and whether the GPU sits behind an SMMU are not described in
> the baseline source. They must come from a later upstream devicetree, the
> vendor's firmware tables, or the SoC documentation.

> OPEN QUESTION: Sky1 boards often boot with ACPI rather than a devicetree.
> The driver described in 00–10 is devicetree-only. No ACPI identifier or
> resource layout for the GPU appears in the source.

## 11.5 Sky1 platform resources for the GPU

These identifiers are fixed by the public Sky1 devicetree headers and SoC
file. They are what a Sky1 GPU node is expected to reference.

### 11.5.1 Power and performance domains

Sky1 exposes power and DVFS through two SCMI agents (SCMI is a public Arm
specification, DEN0056):

| SCMI agent | Transport | Protocol | Role |
|---|---|---|---|
| Power-management processor | Mailbox (channel 8 on the AP→PM and PM→AP mailboxes), shared memory at `0x06590000` (TX) and `0x065A0000` (RX), 128 bytes each | `0x13` (Performance domain management) | DVFS. Each performance domain is used as a power-domain provider with one cell |
| | | `0x14` (Clock management) | Clocks. One cell |
| Trusted firmware | SMC, function ID `0xC2000001`, own shared memory | `0x11` (Power domain management) | Device power on/off. One cell |

GPU-related domain IDs:

| Identifier | Value | Provider | Meaning |
|---|---|---|---|
| `SKY1_PD_GPU` | 21 | SCMI power domain (SMC agent) | GPU power island on/off |
| `SKY1_PERF_GPU_CORE` | 0 | SCMI performance (mailbox agent) | GPU shader-core clock/voltage domain |
| `SKY1_PERF_GPU_TOP` | 1 | SCMI performance (mailbox agent) | GPU top-level clock/voltage domain (presumably the non-shader logic, not confirmed) |

- **[HW]** The GPU power domain must be on before any GPU register access,
  as on RK3588 (01 §1.4.2). On Sky1 that is an SCMI `POWER_STATE_SET` on
  domain 21 through the trusted-firmware agent. No GPU regulator appears in
  the Sky1 board files, which suggests the firmware sequences the rail and
  island itself (not confirmed).
- **[HW]** Frequency and voltage are owned by the power-management processor
  through **two** performance domains. The OS requests a level per domain
  (SCMI `PERFORMANCE_LEVEL_SET`), and the firmware picks the clock and
  voltage. The OS does not set a GPU clock rate directly or drive
  `mali-supply`/`sram-supply`.
- On Sky1 the CPU clusters already use the same pattern (each CPU node
  references a performance domain with the name `"perf"`), which is the
  public precedent for how a GPU node would reference its two performance
  domains.

> OPEN QUESTION: The relation between the two GPU performance domains (must
> TOP be at or above some level whenever CORE is raised, and are they meant
> to be moved together) is not described. A safe policy is to request both
> together from one combined operating-point table, raising TOP first and
> lowering it last.

> OPEN QUESTION: The level tables for domains 0 and 1 are not in the source.
> They are discovered at runtime with SCMI `PERFORMANCE_DESCRIBE_LEVELS`.
> Whether the levels are reported in kHz or abstract units depends on the
> firmware's `PERFORMANCE_DOMAIN_ATTRIBUTES` reply.

### 11.5.2 Interaction with the driver's DVFS model (01 §1.5)

01 §1.5 describes two DVFS modes: a DT operating-point table with a
regulator, or a single power-domain provider that owns performance
(MT8196). Sky1 fits neither exactly:

- **[POLICY]** Linux attaches *all* listed power domains when there is more
  than one, and links them to the GPU device so they switch together (01
  §1.4.1 step 2). The device itself then has no operating-point table from
  any one domain. With no DT OPP table either, Linux's DVFS setup fails and
  so does probe.
- A clean-room driver for Sky1 therefore needs one of:
  1. a combined operating-point table (in DT or in the driver) whose entries
     each name a level for both performance domains, applied by sending one
     SCMI level request per domain; or
  2. no DVFS: leave both performance domains at their firmware default
     level and skip the governor.
- **[HW]** Whatever the mechanism, the busy/idle accounting of 01 §1.5.2 is
  unchanged. Only the "apply this frequency" step differs.

> OPEN QUESTION: Whether a Sky1 GPU node will list `SKY1_PD_GPU` and the two
> performance domains together, or wrap them in a single combined provider,
> is a future upstream decision. This spec cannot predict it.

### 11.5.3 Clocks

The Sky1 clock IDs (consumed through the SCMI clock protocol, `0x14`) that
belong to the GPU:

| Identifier | Value | Likely binding name |
|---|---|---|
| `CLK_TREE_GPU_CLK_CORE` | 31 | `"core"` |
| `CLK_TREE_GPU_CLK_STACKS` | 32 | `"stacks"` |
| `CLK_TREE_GPU_CLK_400M` | 30 | unknown |
| `CLK_TREE_GPU_CLK_200M` | 272 | unknown |
| `CLK_TREE_GPUTOP_PLL` | 230 | source PLL for the TOP domain |
| `CLK_TREE_GPUCORE_PLL` | 231 | source PLL for the CORE domain |

- **[HW]** As on every platform, the driver must have the `"core"` clock
  enabled before touching the GPU. `"stacks"` and `"coregroup"` are optional
  (01 §1.1.3).
- **[POLICY]** When performance domains own DVFS, the driver should only
  gate clocks and must not set their rates. Setting a rate through the clock
  protocol behind the performance firmware's back may conflict with its
  level choice.

> OPEN QUESTION: Which of these clocks a Sky1 GPU node would list, and in
> which order, is not in the source. The "likely binding name" column is a
> guess from the identifier names. The 400 MHz and 200 MHz clocks may be bus
> or APB clocks that the power-domain firmware already manages.

### 11.5.4 Resets

The Sky1 S5 system controller (`cix,sky1-s5-system-control`, MMIO
`0x16000000`, 4 KiB) has two GPU reset lines:

| Identifier | Index | Register offset | Bit |
|---|---|---|---|
| `SKY1_GPU_RESET_N` | 9 | `0x400` (reset group 0, S0 domain word 0) | 6 |
| `SKY1_GPU_RCSU_RESET_N` | 119 | `0x800` (reset group 0, RCSU word) | 28 |

- **[HW]** Both lines are active-low: clearing the bit holds the block in
  reset, setting it releases reset. Reading the bit as 0 means "in reset".
- **[POLICY]** The Sky1 reset driver waits 50–100 µs after each assert or
  deassert.
- **[POLICY]** The GPU driver uses **no** external reset controller (01
  §1.1.3). GPU recovery is done through the `GPU_CMD` soft reset (03 §3.3). On Sky1 these lines are expected to be handled by power-domain
  firmware when the island is switched, not by the GPU driver.

> OPEN QUESTION: "RCSU" is not expanded anywhere in the source. Whether the
> OS must ever deassert either GPU reset line itself (for example when
> firmware leaves the GPU in reset at handoff) is unknown.

## 11.6 Coherency and L2 address hash

- **[HW]** Coherency selection follows 01 §1.6 unchanged: ACE-Lite only if
  the platform marks the GPU `dma-coherent` *and* `COHERENCY_FEATURES` bit 0
  is set. Otherwise protocol 31 (none) and full CPU cache maintenance.
- Sky1 has an Arm CI-700 coherent interconnect (the reset and
  performance-domain maps both name a "CI700" block), so an I/O-coherent
  GPU port is plausible.

> OPEN QUESTION: Whether the Sky1 GPU is I/O-coherent is not stated in the
> source. Until confirmed, treat it as **non-coherent**. That is always safe
> (it costs cache maintenance, not correctness). Marking a non-coherent
> device coherent corrupts memory.

> OPEN QUESTION: Whether Sky1 needs a custom `ASN_HASH` (01 §1.3) to match
> its interconnect's address striping is not stated. Without one, the GPU
> uses its built-in hash. A wrong hash is expected to cost performance, not
> correctness, but this is not proven.

## 11.7 Core groups and L2

- **[POLICY]** The driver powers only bit 0 of `L2_PRESENT` (03 §3.4.1).
- A single-bit `L2_PRESENT` is expected even with 10 shader cores, because
  CSF GPUs slice one L2 internally instead of using several core groups.
  This expectation is not stated in the source.

> OPEN QUESTION: If the Sky1 GPU reports more than one bit in `L2_PRESENT`,
> the driver as specified runs at reduced width. Log `L2_PRESENT` and
> `SHADER_PRESENT` at bring-up to confirm.

## 11.8 Bring-up checklist

1. Get the GPU node data (11.4) from a source the clean room may use.
2. Turn on SCMI power domain 21 through the SMC agent (11.5.1).
3. Enable the `"core"` clock (and `"stacks"` if listed) through the SCMI
   clock protocol, without setting rates (11.5.3).
4. Read `GPU_ID` and check arch major 12, product major 0. Read
   `SHADER_PRESENT` (expect 10 bits), `L2_PRESENT` (expect 1 bit) and
   `GPU_FEATURES` (expect bit 2) (11.1, 11.7).
5. Select coherency protocol 31 unless coherency is confirmed (11.6).
6. Power the L2 and load `arm/mali/arch12.8/mali_csffw.bin` (03 §3.4, 05).
7. Read `GLB_VERSION` and use the matching interface layout (11.3).
8. Bring up DVFS last, or leave it off at first (11.5.2).
