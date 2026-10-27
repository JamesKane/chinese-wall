# 01: Register Map (Zhouyi v3, host view)

All registers are 32 bits wide, accessed as aligned 32-bit words, and
little-endian. Offsets are from the NPU window base (`0x14260000` on
Sky1). Names are the hardware mnemonics used by the vendor's register
documentation numbering ("A1.1.x"). The register documentation itself was
not available. Field meanings come from how the vendor driver uses each
register.

## 1.1 Window layout

| Range | Block |
|---|---|
| `0x0000`–`0x006B` | TSM global registers (§1.2, §1.3) |
| `0x0800`–`0x09FF` | Command pool *n* registers, `0x800 + 0x40·n`, *n* = 0–7 (§1.5) |
| `0x0C00`–`0x0CFF` | Cluster *n* registers, `0xC00 + 0x20·n`, *n* = 0–7 (§1.4) |
| `0x1F00` | Debug page selection (§1.6) |
| `0x2000`–`0x322C` | Debug-link window into the selected cluster/core (§1.6) |

- **[HW]** The register window is 64 KiB. Debug register access from
  userspace is accepted up to offset `0x322C` (03 §3.7).
- **[HW]** Zhouyi v1/v2 parts use a different "host control" layout at
  `0x00`–`0x68` (control, status, start PC and so on). It does not apply
  to v3, except that offset `0x50` holds the revision value on every
  version (§1.2).

## 1.2 TSM identification and configuration

| Offset | Name | Access | Fields |
|---|---|---|---|
| `0x010` | `TSM_CONFIG` | RW | `[2:0]` scheduling policy (reserved). The driver never writes it |
| `0x014` | `TSM_BUILD_INFO` | RO | `[25:24]` maximum partitions − 1 (so 1–4); `[19:16]` number of command pools |
| `0x018` | `TSM_STATUS` | RW | `[31]` the last TSM command failed (shared by all pools); `[7:0]` one bit per command pool: set = pool full/busy |
| `0x020` | `TSM_SOFT_RESET` | RW | `[0]` write 1 to start a global soft reset, hardware clears it on completion; `[1]` hardware sets it when the reset is done |
| `0x050` | `TSM_REVISION` | RO | `[27:24]` implementation; `[23:16]` primary part; `[15:8]` secondary part; `[7:4]` major revision; `[3:0]` minor |

**[HW]** Values of `0x050` and what they identify:

| `0x050` value | Identified as | ISA enum (03 §3.3) |
|---|---|---|
| `0x00000000` | Zhouyi v1 | 1 |
| `0x00000100` | Zhouyi v2 (Z2) | 2 |
| `0x00000200` | Zhouyi v2 (Z3) | 3 |
| `0x00000300` | Zhouyi v2 (X1) | 4 |
| `0x00010000` | Zhouyi **v3**, called r0p2 | 5 |
| `0x00010003` | Zhouyi **v3**, called r0p3 | 5 |
| `0x00010100` | Zhouyi v3_1 r0p0 | 6 |
| anything else | unsupported | — |

- **[HW]** For v3 the "secondary part" field `[15:8]` is 0 and the
  primary part field `[23:16]` is non-zero. Either test tells v3 apart
  from v3_1.

> OPEN QUESTION: The labels "r0p2"/"r0p3" do not match the major/minor
> field decode (`0x00010003` decodes as major 0, minor 3). Treat the
> table values as opaque. Which value Sky1 silicon returns is not
> recorded.

### 1.2.1 Clearing `TSM_STATUS[31]`

**[HW]** After any TSM command the host reads `TSM_STATUS`. If bit 31 is
set, the host clears it by writing back the value it read with bit 31
forced to 0. The other bits are written back unchanged.

## 1.3 TSM command interface and tick counter

| Offset | Name | Access | Fields |
|---|---|---|---|
| `0x000` | `TSM_CMD_SCHD_CTRL_HANDLE` | WO | Writing it issues a command, see below |
| `0x004` | `TSM_CMD_SCHD_CTRL_INFO` | RW | `[15:0]` command tag ID. It is attached to the next command and reported back in the bad-command / executing-command registers |
| `0x008` | `TSM_CMD_SCHD_ADDR_HIGH` | RW | Device address `[63:32]` of the head TCB to dispatch |
| `0x00C` | `TSM_CMD_SCHD_ADDR_LOW` | RW | Device address `[31:12]` of the head TCB, in bits `[31:12]`. Must be 4 KiB aligned |
| `0x060` | `TICK_COUNTER_LOW` | RO | Free-running counter bits `[31:0]` |
| `0x064` | `TICK_COUNTER_HIGH` | RO | Counter bits `[63:32]` |
| `0x068` | `TICK_COUNTER_CONTROL_STATUS` | RW | `[8]` overflow flag; `[1]` clear; `[0]` see note |

`TSM_CMD_SCHD_CTRL_HANDLE` fields:

| Bits | Field | Values |
|---|---|---|
| `[3:0]` | Command | 0 none, 1 **create** pool, 2 **destroy** pool, 3 **abort** pool, 4 **dispatch**, 5 **debug-dispatch** |
| `[9:8]` | QoS of the pool / dispatch | 0 = slow, 2 = fast (the values the driver writes) |
| `[12]` | Cluster map (create only) | 0 = map one cluster to the pool, 1 = map all clusters. Read-only on single-cluster TSMs |
| `[19:16]` | Command pool number | 0–7 |
| `[27:24]` | Core for debug-dispatch | Core index inside the cluster |

> OPEN QUESTION: The register description gives the QoS field as
> "0/1/2 = slow/fast", which leaves the meaning of 1 unclear. The driver
> only ever writes 0 (slow) or 2 (fast).

> OPEN QUESTION: Tick-counter bit 0 behaves the opposite way to its
> "enable" label. The driver writes `0x0` to start counting and `0x3`
> (clear + bit 0) to stop and clear. Either bit 0 is a "stop" bit or the
> label is wrong. Use the driver's values.

> OPEN QUESTION: `TSM_CMD_SCHD_ADDR_LOW` is documented as holding only
> bits `[31:12]`, yet the driver sometimes dispatches from a TCB that is
> not 4 KiB aligned (02 §2.5.3). Whether the low 12 bits are honoured is
> unknown. A clean implementation should keep dispatch heads 4 KiB
> aligned.

## 1.4 Cluster registers (cluster *n* at `0xC00 + 0x20·n`)

| Offset | Name | Access | Fields |
|---|---|---|---|
| `+0x00` | `CLUSTER_CONFIG` | RO | `[12]` cluster present; `[11:8]` number of cores; `[7:4]` AIFFs per core; `[3:0]` TECs per core |
| `+0x04` | `CLUSTER_CONTROL` | RW | `[17:16]` partition ID this cluster belongs to; `[12]` cluster enable; `[11:8]` cores enabled; `[7:4]` AIFFs enabled per core; `[3:0]` TECs enabled per core |

- **[HW]** The three "enabled" counts should be non-zero for a working
  cluster. Writing 0 in `[11:8]` (with `[12]` still set) is how the
  driver idles a cluster for suspend (02 §2.10). That write can set
  `TSM_STATUS[31]`, which must then be cleared (§1.2.1).
- **[HW]** The usual "enable everything" value is: partition ID in
  `[17:16]`, bit 12 set, and `CLUSTER_CONFIG[11:0]` copied into `[11:0]`.

## 1.5 Command pool registers (pool *n* at `0x800 + 0x40·n`)

| Offset | Name | Access | Fields |
|---|---|---|---|
| `+0x00` | `CMD_POOL_CONFIG` | RW | `[3:0]` partition the pool serves (see question below) |
| `+0x04` | `CMD_POOL_STATUS` | RW, W1C | `[31:24]` physical ID of the cluster that finished last; `[6]` idle; `[5]` signal; `[4]` error; `[3]` fault; `[2]` exception; `[0]` done |
| `+0x08` | `CMD_POOL_INTR_CTRL` | RW | Enables: `[11]` pool-level, `[10]` cluster-level, `[9]` core-level, `[8]` TEC-level; `[5]` signal, `[4]` error, `[3]` fault, `[2]` exception, `[0]` done |
| `+0x0C` | `CMD_POOL_INTR_STATUS` | RO | `[31:24]` cluster physical ID; `[23:20]` core dynamic ID; `[19:16]` TEC dynamic ID; `[11]` pool-level, `[10]` cluster-level, `[9]` core-level, `[8]` TEC-level; `[5]` signal, `[4]` error, `[3]` fault, `[2]` exception, `[0]` done |
| `+0x10` | `CMD_POOL_IRQ_SIGNAL` | RO | `[31:16]` number of pending signals; `[14:0]` signal ID |
| `+0x14` | `CMD_POOL_FIRST_BAD_CMD` | RW | `[31]` tag valid; `[15:0]` tag of the first failed command. The host writes 0 after reading it |
| `+0x18` | `CMD_POOL_LAST_EXEC_CMD` | RO | `[15:0]` tag of the command issued most recently |
| `+0x1C` | `CMD_POOL_SECURE` | RW | `[0]` 0 = secure pool, 1 = non-secure pool |
| `+0x20` | `CMD_POOL_IRQ_SIGNAL_FLAG` | RO | See §1.5.1 |
| `+0x24` | `CMD_POOL_INTR_TCB_PTR` | RO | 32-bit TCB pointer (TCBP) of the TCB that raised the interrupt, as an offset from the ASID 0 base (02 §2.8) |

- **[HW]** `CMD_POOL_STATUS` bits 0 and 2–6 are write-1-to-clear. The
  driver clears one bit per write.
- **[HW]** The idle bit `[6]` reads 1 when the pool does not exist. The
  driver creates a pool only when this bit is 1, and treats it still
  reading 1 after a create as a failure.

> OPEN QUESTION: The register text says writing 1/2/3/4 to
> `CMD_POOL_CONFIG[3:0]` binds the pool to partition 1/2/3/4. The driver
> writes the 0-based partition ID (0 for partition 0). For the
> single-partition Sky1 setup the driver's value (0) works. The encoding
> for other partitions is unverified.

### 1.5.1 `CMD_POOL_IRQ_SIGNAL_FLAG`

| Bits | Meaning |
|---|---|
| `[31]` | Hardware exception signal |
| `[30]` | Software exception signal |
| `[29]` | printf signal: a task posted a debug-print buffer |
| `[28]` | Profiler signal |
| `[27]` | Coredump signal |
| `[15:0]` | Error code (exception signals) or printf length (printf signal) |
| `[19:0]` | Profiler signal: the profiler info buffer address `[31:12]` as an ASID-0 offset. The buffer holds two 32-bit words: (data offset from ASID 0 base, data size) |

**[HW]** "Exception signal" means `[31]` or `[30]` is set.

### 1.5.2 Interrupt status groups

**[HW]** In `CMD_POOL_INTR_STATUS`:

| Group | Bits | Use |
|---|---|---|
| Worth handling | `[11:8]`, `[4:0]` | Mask `0xF1F`. If none of these is set, the interrupt is not for this pool |
| Abnormal | `[4:2]` (error, fault, exception) | The job failed |
| Serious | `[4:3]` (error, fault) | The pool must be aborted |
| Event type | `[5:0]` | Passed on to completion handling (02 §2.6) |

## 1.6 Debug-link registers

| Offset | Name | Access | Fields |
|---|---|---|---|
| `0x1F00` | `DEBUG_PAGE_SELECTION` | RW | `[12]` debug window enable; `[11:4]` cluster select; `[3:0]` core select |
| `0x2070` | Cluster GM control (in the debug window) | RW | `[19:16]` GM size code; `[3:0]` GM remap enable |

**[HW]** GM size code to bytes:

| Code | Size |
|---|---|
| 0 | 512 KiB |
| *n* = 1–7 | 1 MiB × 2^(*n* − 1) (1, 2, 4, 8, 16, 32, 64 MiB) |

- **[HW]** To read GM size: write `0x1000` (enable, cluster 0, core 0) to
  `0x1F00`, read `0x2070`, then write 0 to `0x1F00`.

> OPEN QUESTION: The register text skips code 6 (it lists 0–5 and 7). The
> formula gives 32 MiB for 6. Nothing else in the `0x2000`–`0x322C`
> window is described in the sources.

## 1.7 Interrupt-enable values the driver uses

| Purpose | `CMD_POOL_INTR_CTRL` value |
|---|---|
| Normal operation | `0x63D` (cluster, core, signal, error, fault, exception, done) |
| With TEC-level interrupts (debug printf / profiling) | `0x73D` |
| Disabled | `0x000` |

- **[HW]** The pool-level enable (bit 11) is never set, but pool-level
  status (`INTR_STATUS[11]`) is still used to find the interrupting pool.

> OPEN QUESTION: Whether pool-level status bit 11 is reported without
> its enable bit is implied by the driver but not proven. An OS should
> also accept an interrupt that sets any bit in the 0xF1F group.

## 1.8 Sky1 glue: RCSU, DMRP and power wrapper

- **[HW]** *(vendor)* The Sky1 platform layer of the NPU driver does
  **no** register accesses of its own. It handles power domains, ACPI
  power-resource children, runtime PM and DVFS only (00 §0.5–§0.7). No
  NPU RCSU registers, power-wrapper registers, harvesting registers or
  DMRP (memory-repair) registers appear in the driver, the vendor DT or
  the ACPI notes.
- **[HW]** The only Sky1 NPU control points outside the NPU window are:
  the S5 reset lines (00 §0.3), SCMI power domains 16–19, SCMI perf domain
  8, the nvmem harvest cell (00 §0.5.1) and the SMMU stream (00 §0.4).

> OPEN QUESTION: On Sky1 the NPU SRAM repair and power-switch sequencing
> are presumably done by TF-A (DT) or by the `CRE*` power resources
> (ACPI). This adds no new DMRP information to sky1-soc 02 §2.5.
