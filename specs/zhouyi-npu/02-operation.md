# 02: Operation

Register names refer to 01. Unless stated otherwise every statement
describes the v3 part as used on Sky1: one partition, one cluster, and
command pool 0 used for partition 0.

## 2.1 Power-up and global soft reset

**[HW]** Bring-up order:

1. Turn on the NPU core power domains (DT: SCMI power domains 16–18;
   ACPI: put each `CRE*` child in D0; see 00 §0.5, §0.7). Until this is
   done the register window must not be accessed.
2. Optionally attach the perf domain and pick an operating point (00 §0.6).
3. Read `TSM_REVISION` (`0x050`) and check that the part is v3 (§2.2).
4. Global soft reset: write 1 to `TSM_SOFT_RESET` (`0x020`), wait
   **50 µs**, then read it back. The reset succeeded only if the value
   is exactly `0x2` (done bit set, start bit clear). Any other value
   means failure.
5. Read `TSM_BUILD_INFO` for the partition and pool limits.
6. Check that every cluster listed in the cluster-to-partition table has
   `CLUSTER_CONFIG[12]` set.
7. Initialize each partition (§2.3).

- **[HW]** The soft reset covers the whole NPU (TSM, every pool and every
  cluster), not one partition. After it, every partition must be
  initialized again.
- **[POLICY]** The 50 µs wait is the vendor default. The driver does not
  poll.

> OPEN QUESTION: Step 6 indexes clusters by their position in the table,
> not by the cluster ID inside each entry. This makes no difference on
> Sky1 (cluster 0 only).

## 2.2 Identification

- **[HW]** `TSM_REVISION` `0x00010000` or `0x00010003` means v3 (01 §1.2).
  The device then reports ISA version 5, architecture 0 ("Zhouyi") and
  configuration number 0 to userspace (03 §3.3).
- **[HW]** For each cluster in a partition: read `CLUSTER_CONFIG` to get
  the core count `[11:8]` and TEC count `[3:0]`, and read its GM size
  through the debug window (01 §1.6). Userspace sees these counts in the
  partition capability (03 §3.4.2).
- **[POLICY]** The driver reads the GM size with cluster 0 selected in
  the debug window, even for clusters other than 0. It keeps one GM size
  for the whole device.

## 2.3 Partition initialization

**[HW]** For partition *p* (it uses command pool *p*):

1. For each cluster *c* in the partition: write `CLUSTER_CONTROL(c)` =
   (*p* << 16) | `0x1000` | (`CLUSTER_CONFIG(c)` & `0xFFF`). This enables
   every core, AIFF and TEC.
2. Write `CMD_POOL_CONFIG(p)` = *p* (see the open question in 01 §1.5).
3. Write `CMD_POOL_SECURE(p)` = 1 (non-secure).
4. Write `CMD_POOL_INTR_CTRL(p)` = `0x63D` (01 §1.7).
5. Write `TICK_COUNTER_CONTROL_STATUS` = `0x3` (counter stopped and cleared).

- **[HW]** The command pool is **not** created here. Pools are created
  lazily when the first job is submitted (§2.5.1).

## 2.4 Task control blocks (memory format)

**[HW]** A TCB is **128 bytes**, little-endian, naturally aligned, and
stored in DMA-visible memory. TCBs of one job sit back to back. A job's
chain has this order: zero or more **init** TCBs, then **task** TCBs
grouped in fours (a group spans 4 TCBs), and the last task TCB ends the
job. Userspace builds every TCB. The kernel reads a few fields and edits
the `next` field (§2.5.2). It also builds one task TCB of its own per
job (the "hold" TCB, §2.5.2).

### 2.4.1 Common header (all TCB types)

| Offset | Size | Field | Meaning |
|---|---|---|---|
| 0 | 4 | `flag` | `[1:0]` type: 0 init, 1 task, 2 loop task; `[6]` group end; `[7]` grid end; `[20:16]` number of segment-MMU configurations |
| 4 | 4 | `next` | Link to the next TCB chain (low 32 bits of its device address; 0 = end of chain) |

### 2.4.2 Task TCB (type 1 or 2)

| Offset | Size | Field | | Offset | Size | Field |
|---|---|---|---|---|---|---|
| 8 | 4 | loop count | | 52 | 4 | `sp` (stack) |
| 12 | 4 | `spc` start PC | | 56 | 4 | `pp` |
| 16 | 4 | `interrupt` (`[8]` = raise a TEC-level interrupt) | | 60 | 4 | `dp` |
| 20 | 2 | group ID | | 64 | 4 | `cp` |
| 22 | 2 | grid ID | | 68 | 4 | `pprint` (printf buffer) |
| 24 | 2 | reserved | | 72 | 4 | `pprofiler` |
| 26 | 2 | task ID | | 76 | 2 | core ID |
| 28–32 | 2 each | grid dim x, y, z | | 78 | 2 | cluster ID |
| 34–38 | 2 each | group dim x, y, z | | 80 | 2 | reserved |
| 40–44 | 2 each | group ID x, y, z | | 82 | 2 | TEC ID |
| 46–50 | 2 each | task ID x, y, z | | 84–112 | 4 each | `fmdp`, `tap`, `dap`, `pap`, `idp`, `dsize`, `tcbp`, global param |
| | | | | 116 | 12 | reserved |

- **[HW]** Address-like fields (`spc`, `pprint`, `tcbp` and the data
  pointers) are 32-bit **offsets from the ASID 0 base** (§2.8).
  `tcbp` is the TCB's own offset. The interrupt TCB pointer register
  reports this value (01 §1.5).
- **[HW]** The core-ID and cluster-ID fields record where the task ran.
  The driver only reads them, in debug builds.

### 2.4.3 Init TCB (type 0), cluster variant

| Offset | Size | Field |
|---|---|---|
| 8 | 8 | reserved |
| 16 | 4 | GM control |
| 20 | 2 | grid ID |
| 22 | 2 | reserved |
| 24 | 8 | GM region control × 2 (u32 each) |
| 32 | 16 | GM region address × 2 (u64 each, stored as low word then high word) |
| 48 | 32 | **ASID 0–3 base** × 4 (u64 each, low word then high word) |
| 80 | 8 | DTCM address (u64) |
| 88 | 40 | reserved |

**[HW]** In ASID entries the low word holds control bits: `[4:0]` size
code, `[5]` write enable, `[6]` read enable, `[7]` DTCM enable, `[11:8]`
reserved.

> OPEN QUESTION: The size code encoding is not documented, and neither is
> how the low control bits combine with the address (a 4 KiB-aligned base
> leaves `[11:0]` free).

### 2.4.4 Init TCB (type 0), core variant

| Offset | Size | Field |
|---|---|---|
| 8 | 16 | reserved |
| 24 | 40 | Segment-MMU configuration: control (u32), remap (u32), then 4 × (ctrl0, ctrl1) u32 pairs |
| 64 | 24 | reserved |
| 88 | 40 | Segment-MMU configuration for the next core (same 40-byte format) |

> OPEN QUESTION: How the hardware tells the cluster and core variants of
> an init TCB apart (position in the chain or some flag bits) is not
> visible. Userspace builds them, so a kernel only needs to keep them
> unchanged.

## 2.5 Job submission

### 2.5.1 Command pool lifecycle

**[HW]** Each command is issued as follows (the commands are in 01 §1.3):

1. Write any address or tag registers the command needs.
2. Write `TSM_CMD_SCHD_CTRL_HANDLE`.
3. Read `TSM_STATUS`. If bit 31 is set the command failed. Clear it
   (01 §1.2.1).

| Operation | Register writes (pool *p*) |
|---|---|
| Create | Only if `CMD_POOL_STATUS(p)[6]` (idle) = 1: HANDLE = `0x1` \| (*p* << 16) \| `0x1000` (map all clusters). Afterwards the idle bit must read 0 |
| Dispatch with new head | ADDR_HIGH = head[63:32]; ADDR_LOW = head[31:0]; INFO = tag; HANDLE = `0x4` \| QoS \| (*p* << 16) |
| Dispatch (continue) | INFO = tag; HANDLE = `0x4` \| QoS \| (*p* << 16). The address registers are not rewritten |
| Debug-dispatch | ADDR_HIGH/LOW = head; INFO = tag; HANDLE = `0x5` \| QoS \| (*p* << 16) \| (core << 24) |
| Abort | HANDLE = `0x3` \| (*p* << 16), then wait 50 µs |
| Destroy | ADDR_HIGH = 0; ADDR_LOW = 0; INFO = 0; HANDLE = `0x2` \| (*p* << 16) |

QoS is `0x000` (slow) or `0x200` (fast). The tag is the low 16 bits of
the userspace job ID.

- **[POLICY]** If create fails, the driver tries destroy then create once.
  If destroy fails, it aborts, retries destroy, and gives up on a second
  failure. After every destroy it writes `CMD_POOL_INTR_CTRL` = `0x63D`
  again (TEC interrupts off).
- **[POLICY]** "Dispatch with new head" is used for the first job after a
  create, after an abort, or when that QoS queue is empty. "Continue" is
  used when the new job has already been linked behind the previous one
  in memory (§2.5.2).

### 2.5.2 Chaining jobs: the hold TCB

**[HW]** *(vendor)* The TSM walks a chain of TCBs through their `next`
fields. To append job *B* to a pool that is still running job *A*, the
driver does not touch *A*'s TCBs while they run. Instead it adds one
extra task TCB, the **hold TCB**, after every job:

1. When *A* is submitted, allocate a 128 + 1024 byte buffer in ASID 0
   (it is page-rounded, and its type must be TCB, 03 §3.5). At offset 256
   in it, write this 16-byte stub of NPU machine code:

   | Bytes | Value |
   |---|---|
   | 0–3 | `00 80 4F 01` |
   | 4–7 | `FF 7F 0C 40` |
   | 8–11 | `FF 7F 0C 40` |
   | 12–15 | `FF 7F 0C 40` |

2. Copy *A*'s last task TCB into the hold TCB (offset 0 of the buffer),
   then change it: `flag` = 1 (task) | group end (`0x40`), with grid end
   left clear; clear `interrupt[8]`; `next` = 0; `spc` = (buffer address
   + 256) − ASID 0 base; `tcbp` = buffer address − ASID 0 base.
3. Set *A*'s last task TCB `next` = low 32 bits of the hold TCB's device
   address. The hold TCB now ends *A*'s chain and runs the stub as a
   no-op task.
4. When *B* is submitted (with its own hold TCB built as above), set
   *A*'s hold TCB `next` = low 32 bits of *B*'s head TCB address, then
   issue a "continue" dispatch.
5. When *A* completes: set the previous job's hold TCB `next` back to 0
   (if it still points at *A*), and set *A*'s last task TCB `next` back
   to 0. The hold buffer can then be reused for a later job. The driver
   never reuses the one hold buffer that currently ends the queue.

> OPEN QUESTION: The stub's meaning is not documented. It is presumably
> an "exit" instruction followed by padding or no-ops. Copy it exactly.

> OPEN QUESTION: `next` stores only 32 bits, written as the low half of
> the absolute device address. The driver assumes the high half matches
> the pool's current head (`TSM_CMD_SCHD_ADDR_HIGH`). All TCB chains of a
> pool must therefore sit in the same 4 GiB window. The driver guarantees
> this for ASID 0 (§2.8). Whether `next` is absolute-low-32 or relative
> to ASID 0 is unproven, because the two agree only when the ASID 0 base
> is 4 GiB aligned.

> OPEN QUESTION: How the TSM notices a `next` link written after it has
> already read the hold TCB is not explained. The driver relies on the
> "continue" dispatch command. Ordering between the memory write and the
> register write (a write barrier) is required but not specified.

### 2.5.3 Queues, QoS and single-group jobs

- **[POLICY]** *(vendor)* Each pool keeps two software queues, slow and
  fast. A job flagged QoS-slow goes on the slow queue. Any other job goes
  on the fast queue, but its dispatch carries the fast QoS code only if
  it is flagged QoS-fast.
- **[POLICY]** *(vendor)* If the cluster has more than one enabled core,
  and both the previous job and the new job are flagged "single group"
  and neither uses segment MMU, the driver **skips the new job's init
  TCBs**. It links or dispatches from the job's first task TCB, so that
  consecutive single-group jobs can run on different cores at the same
  time. This assumes the skipped init TCBs would have set the same
  cluster state.

### 2.5.4 Debug-dispatch

**[HW]** Debug-dispatch (command 5) runs a job on one named core. A pool
that has taken a debug-dispatch cannot have other jobs linked to it. The
driver refuses further jobs on that pool until it is destroyed, and
destroys it as soon as the debug job ends.

### 2.5.5 Multi-core scheduling

- **[HW]** Spreading work over cores and TECs is done by the TSM in
  hardware. The host only enables the cores (`CLUSTER_CONTROL`) and feeds
  TCB chains to a pool. The task TCB group/grid fields and the core/TEC
  IDs in interrupt status tell where work ran.
- **[POLICY]** On Sky1 there is one partition and one pool, so all jobs
  from all processes share pool 0 and are serialized by the TSM in
  submission order within each QoS level.

## 2.6 Interrupts and completion

**[HW]** On the NPU interrupt, for each partition *p*:

1. Read `CMD_POOL_INTR_STATUS(p)`. The pool whose bit 11 is set is the
   source. If no bit in the `0xF1F` group is set, there is nothing to do.
2. Read `CMD_POOL_IRQ_SIGNAL(p)`, `CMD_POOL_INTR_TCB_PTR(p)`,
   `CMD_POOL_IRQ_SIGNAL_FLAG(p)` and the 64-bit tick counter.
3. If status bits `[4:2]` are set: read `CMD_POOL_FIRST_BAD_CMD(p)` (the
   tag is valid if bit 31 is set), then write 0 to it.
4. Acknowledge in `CMD_POOL_STATUS(p)`. **Fault and done arrive
   together**: on fault write `0x08`, then `0x01`. Otherwise write
   whichever single bit is set, in priority order done (`0x01`), error
   (`0x10`), exception (`0x04`).
5. If the event is a signal (bit 5), handle it (§2.6.1), then write
   `0x20` to `CMD_POOL_STATUS(p)`. The pending-signal count is expected
   to be 1.
6. Events that are TEC-level only (bit 8) without a signal are not job
   completions and are ignored after the acknowledge.

### 2.6.1 Signals

| Signal flag | Action |
|---|---|
| printf (`[29]`) with non-zero length | The TCB at (ASID 0 base + interrupt TCB pointer) has a `pprint` offset. The string at (ASID 0 base + `pprint`) is a debug message from the task |
| coredump (`[27]`) | The job whose task-TCB range contains the interrupting TCB is marked "coredump" |
| profiler (`[28]`) | Profiler data is ready (01 §1.5.1). The vendor build ignores it |

- **[HW]** A signal interrupt does not end a job.

### 2.6.2 Matching an interrupt to a job

**[HW]** Let *t* be the value of `CMD_POOL_INTR_TCB_PTR` (an ASID 0
offset). A running job has **ended** when either:

- *t* equals (job's last task TCB address − ASID 0 base) and the event
  has done or exception set; or
- the event has error or fault set, or the signal flag has an exception
  signal (`[31:30]`). The driver then ends the first running job it
  finds.

The job **failed** if status bits `[4:2]` were set or an exception
signal was present. Otherwise it succeeded.

- **[POLICY]** The driver records the tick counter value at the end
  interrupt as the job's tick count when the tick counter is enabled
  (03 §3.7).
- **[POLICY]** Completion is passed to a worker thread. It unlinks hold
  TCBs (§2.5.2 step 5), may destroy the pool, and wakes waiting threads
  (03 §3.8).

## 2.7 Error recovery

- **[HW]** On error, fault or an exception signal, the pool must be
  **aborted** (§2.5.1). The next job is then dispatched with a new head.
- **[POLICY]** If any queued job asked for coredump, the abort is skipped
  so that the coredump can complete. The coredump signal follows the
  fault.
- **[POLICY]** The pool is destroyed when every job has ended (and none
  wants coredump) and neither QoS queue ends in a grid-end TCB. Because
  hold TCBs never carry grid end, in practice the pool stays created
  until the last file handle closes, the clusters are reconfigured, or
  the device suspends.
- **[POLICY]** On close of a file handle: its jobs are dropped. If no
  other handle has jobs, the pool is destroyed, and if one of the dropped
  jobs was running the pool is also aborted. Any abort marks every job
  that has not succeeded as failed and wakes its waiters.
- **[HW]** A global soft reset (§2.1) is the last resort. The v3 driver
  does it only at probe.

> OPEN QUESTION: There is no hardware timeout or watchdog for v3 jobs in
> the driver. A hung job stays "running" until userspace aborts the pool
> (03 §3.7) or closes the device. A clean implementation should add a
> timeout that aborts the pool and, if that fails, soft-resets the NPU.

## 2.8 Address translation and ASIDs

- **[HW]** The NPU works with up to **32 ASIDs** (address-space
  regions). Each ASID is a 64-bit base address. Pointers inside TCBs and
  NPU programs are 32-bit offsets from a base. ASIDs 0–3 are loaded from
  the cluster init TCB (§2.4.3). The kernel never writes ASID registers.
  It only reports the bases (03 §3.4.1), and userspace puts them in the
  init TCB.
- **[HW]** *(vendor)* A buffer must not cross a 4 GiB boundary. Within an
  ASID whose span is larger than 2 GiB, the top **1 GiB** of the 4 GiB
  window ([base + 3 GiB, base + 4 GiB)) must not be used. For an ASID
  other than 0 whose base differs from ASID 0's, the forbidden range is
  only the top 512 MiB ([base + 3.5 GiB, base + 4 GiB)). The default
  ASID span is 3 GiB.

> OPEN QUESTION: Why the top of each window is excluded is not stated.
> NPU-internal resources (GM, local SRAM) are probably decoded there in the
> 32-bit view.

### 2.8.1 With an IOMMU (Sky1 default)

**[POLICY]** *(vendor)* Layout of the driver's IOVA space:

1. Set the device's DMA limit to 32 GiB (35-bit addresses).
2. Reserve the top 1 GiB of every 4 GiB window below 32 GiB in the IOVA
   allocator, so that no mapping ever lands there.
3. At probe, allocate *N* contiguous IOVA regions with backing memory.
   ASID *i* is region *i*, and its base is the region's IOVA:

   | System RAM | *N* | Region size | ASIDs |
   |---|---|---|---|
   | > 16 GiB | 6 | 1 GiB | 0–5 |
   | > 8 GiB | 4 | 1 GiB | 0–3 |
   | > 1 GiB | 1 | 1 GiB | 0 and 1 share the region |
   | ≤ 1 GiB | 1 | 256 MiB | 0 and 1 share |

4. If step 2 fails, fall back to a 32-bit limit (3 GiB).

- **[POLICY]** All user allocations are carved from these regions with
  a page bitmap. ASIDs without a region report base
  `0xFFFFFFFFFFFFFFFF`, and the reported ASID count is 32.
- **[POLICY]** This pins up to 6 GiB of RAM at probe time. A clean
  implementation can map per buffer instead, provided it keeps the ABI
  rules: a fixed base per ASID, each ASID within one 4 GiB window, and
  the excluded top 1 GiB.

### 2.8.2 Without an IOMMU

**[POLICY]** Memory comes only from `memory-region` reserved areas
(00 §0.9). Each area lists the ASIDs it serves. An ASID's base is the
lowest area address, rounded down to 4 GiB if it has several areas. All
areas of one ASID must share the same upper 32 address bits. A
`host-aipu-offset` property is subtracted to convert CPU physical
addresses to NPU addresses. If no area names an ASID, ASIDs 0 and 1
share it.

## 2.9 Cache and coherency

- **[HW]** Treat the NPU as **not cache-coherent**.
- **[POLICY]** *(vendor)* Userspace mappings of NPU buffers are made
  uncached (03 §3.6). The cache flush and invalidate ioctls do clean or
  clean+invalidate by virtual address over the kernel mapping of the
  **whole IOVA region** that contains the buffer, followed by a full
  barrier. They do nothing when there is no IOMMU.
- **[HW]** The kernel must make its own TCB edits (hold TCBs, `next`
  links) visible to the NPU before it issues the dispatch register write.

## 2.10 Power management and DVFS

- **[POLICY]** The device is runtime-active while any file handle is
  open. Opening `/dev/aipu` resumes it, and the last close lets it suspend.
- **[HW]** Suspend sequence:
  1. Refuse (busy) if any job is running or deferred.
  2. For each cluster write `CLUSTER_CONTROL` with 0 enabled cores (bit
     12 and the partition field left set), then clear `TSM_STATUS[31]`
     if it was set.
  3. Destroy the command pool if it exists.
  4. Turn the core power domains off (ACPI: `CRE*` children to D3).
- **[HW]** Resume sequence: power the core domains back on, then write
  `CLUSTER_CONTROL` with the full core count again. The vendor driver
  restores only cluster 0, which is enough for Sky1.
- **[POLICY]** While suspended (or after userspace sets 0 cores, 03
  §3.7), job submission fails with `ENODEV`.

> OPEN QUESTION: On resume the vendor driver does not repeat the soft
> reset or §2.3 steps 2–5. If the domains really lose state, pool
> configuration, secure mode and interrupt enables would be lost. A
> clean implementation should redo §2.1 steps 4–7 after every power-on.

- **[POLICY]** DVFS is userspace-governed (00 §0.6). The driver never
  changes frequency around jobs.
- **[POLICY]** System sleep uses the runtime suspend and resume paths.
