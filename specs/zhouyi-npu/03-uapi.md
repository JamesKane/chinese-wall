# 03: `/dev/aipu` Userspace ABI

> **Licensing note.** The vendor's userspace header is licensed
> **GPL-2.0 WITH Linux-syscall-note**, not MIT. A userspace program may
> include it. A BSD-licensed kernel should not copy it. This section
> gives every layout, number and rule needed to rebuild the interface
> from scratch. Field names are the header's own names, used here only as
> mnemonics. All layouts were checked by compiling the header for an
> LP64 target (AArch64 and x86-64 agree).

Statements are **[ABI]** unless tagged otherwise. The target is binary
compatibility with the vendor Zhouyi user-mode runtime running on a
64-bit kernel.

## 3.1 Device node and general rules

- The device is a character device named **`aipu`** (`/dev/aipu`), with
  permissions `0666`. On Linux it is a "misc" device with a dynamic minor.
  There is only one node for the whole NPU.
- `open` powers the NPU up (00 §0.5) and fails with `EINVAL` if the
  driver has not finished initializing.
- Last `close` of a handle: jobs submitted through it are cancelled
  (02 §2.7), then every buffer it allocated is freed. A buffer still
  linked into a TCB chain is freed later.
- Supported file operations: `ioctl`, `mmap`, `poll`, `open`, `close`.
  **There is no `read` or `write`.**
- Only **64-bit** userspace is supported. The 32-bit compatibility path
  passes structures through unconverted. 32-bit layouts (4-byte pointers)
  would change the sizes and request codes and do not match.
- Unknown request code: `ENOTTY`.
- Structures are copied in and out whole. There is no size versioning
  and no reserved-field checking. Padding bytes are ignored.
- **[POLICY]** Copy faults on most ioctls are reported as `EINVAL`, not
  `EFAULT` (exceptions in §3.10).

## 3.2 Ioctl table

Magic (type) is `'A'` (`0x41`). Direction/size encoding is the standard
Linux one (dir in `[31:30]`: 1 = write to kernel, 2 = read from kernel,
3 = both; size in `[29:16]`; type in `[15:8]`; number in `[7:0]`).

| Nr | Name | Encoded dir | Encoded size | Request code | Real data flow |
|---|---|---|---|---|---|
| 0 | QUERY_CAP | R | 424 | `0x81A84100` | out: capability record (§3.4.1) |
| 1 | QUERY_PARTITION_CAP | R | 128 | `0x80804101` | out: **array** of partition records, one per partition (§3.4.2) |
| 2 | REQ_BUF | RW | 56 | `0xC0384102` | in/out (§3.5.1) |
| 3 | FREE_BUF | W | 32 | `0x40204103` | in (§3.5.2) |
| 4 | DISABLE_SRAM | none | 0 | `0x00004104` | — (§3.5.4) |
| 5 | ENABLE_SRAM | none | 0 | `0x00004105` | — (§3.5.4) |
| 6 | SCHEDULE_JOB | W | 136 | `0x40884106` | in (§3.7.1) |
| 7 | QUERY_STATUS | RW | 24 | `0xC0184107` | in/out plus a user array (§3.7.2) |
| 8 | KILL_TIMEOUT_JOB | W | 4 | `0x40044108` | in: **8 bytes** read (§3.7.4) |
| 9 | REQ_IO | RW | 16 | `0xC0104109` | in/out (§3.7.6) |
| 10 | GET_HW_STATUS | R | 4 | `0x8004410A` | out (§3.7.3) |
| 11 | ABORT_CMD_POOL | none | 0 | `0x0000410B` | — (§3.7.4) |
| 12 | DISABLE_TICK_COUNTER | none | 0 | `0x0000410C` | — (§3.7.5) |
| 13 | ENABLE_TICK_COUNTER | none | 0 | `0x0000410D` | — (§3.7.5) |
| 14 | CONFIG_CLUSTERS | W | 32 | `0x4020410E` | in (§3.7.5) |
| 15 | ALLOC_DMA_BUF | R | 16 | `0x8010410F` | **in/out** despite the R encoding (§3.9) |
| 16 | FREE_DMA_BUF | W | 4 | `0x40044110` | in: `int` fd |
| 17 | GET_DMA_BUF_INFO | RW | 24 | `0xC0184111` | in/out (§3.9) |
| 18 | GET_DRIVER_VERSION | R | 8 | `0x80084112` | out: string written **at the address in `arg`** (§3.7.7) |
| 19 | ATTACH_DMA_BUF | RW | 24 | `0xC0184113` | in/out (§3.9) |
| 20 | DETACH_DMA_BUF | W | 4 | `0x40044114` | in: `int` fd |
| 21 | ALLOC_GRID_ID | R | 4 | `0x80044115` | out: `int` (§3.7.7) |
| 22 | ALLOC_GROUP_ID | RW | 4 | `0xC0044116` | in/out (§3.7.7) |
| 23 | FREE_GROUP_ID | W | 4 | `0x40044117` | in (§3.7.7) |
| 24 | BUF_CACHE_INVALID | W | 32 | `0x40204118` | in: buffer descriptor (§3.6) |
| 25 | BUF_CACHE_FLUSH | W | 32 | `0x40204119` | in: buffer descriptor (§3.6) |

- The request codes must be accepted exactly as listed, including the
  ones whose encoded direction or size does not match the real transfer.

## 3.3 Enumerations

| Name | Values |
|---|---|
| Architecture | 0 = Zhouyi |
| ISA version | 1 v1, 2 v2 (Z2), 3 v2 (Z3), 4 v2 (X1), **5 v3**, 6 v3_1 |
| Buffer data type | 0 none, 1 text (code), 2 read-only data, 3 stack, 4 static (weights), 5 reuse (feature maps), **6 TCB** |
| Buffer region | 0 default DDR, 1 SRAM, 2 DTCM, 3 GM |
| ASID | 0–31 |
| Job execution flags (bits) | `0x01` SRAM mutex; `0x02` QoS slow; `0x04` QoS fast; `0x08` single group; `0x10` multi-group; `0x20` debug-dispatch; `0x40` segment MMU |
| Job end state | 1 done, 2 exception, 3 coredump |
| I/O direction | 0 read, 1 write (32-bit enum) |
| Hardware status | 0 idle, 1 busy, 2 exception (32-bit enum) |
| GM policy (sysfs, DT) | 0 none, 1 shared, 2 half-divided |

## 3.4 Capability records

### 3.4.1 QUERY_CAP record (424 bytes, 8-byte aligned)

| Offset | Size | Field | v3 / Sky1 value |
|---|---|---|---|
| 0 | 4 | `partition_cnt` | Number of partitions (1) |
| 4 | 4 | `asid_cnt` | 32 with an IOMMU. Otherwise the highest ASID named by reserved memory + 1 |
| 8 | 256 | `asid_base[32]` | u64 base of each ASID (02 §2.8). ASIDs with no memory read `0xFFFFFFFFFFFFFFFF`. Entries at or above `asid_cnt` are undefined |
| 264 | 4 | `is_homogeneous` | 1 if every partition has the same arch, version and config |
| 268 | 4 | (padding) | |
| 272 | 8 | `dtcm_base` | 0 on v3 |
| 280 | 4 | `dtcm_size` | 0 on v3 |
| 284 | 4 | `gm0_size` | GM bytes for region 0: all of GM under the shared policy, half under half-divided |
| 288 | 4 | `gm1_size` | 0 under the shared policy, half of GM under half-divided |
| 292 | 4 | (padding) | |
| 296 | 128 | `partition_cap` | Partition 0's record (§3.4.2), filled only when `is_homogeneous` = 1. Otherwise undefined |

- Unused fields may hold leftover kernel data. Only the fields above are
  defined.

### 3.4.2 Partition record (128 bytes)

| Offset | Size | Field | Meaning |
|---|---|---|---|
| 0 | 4 | `id` | Partition index |
| 4 | 4 | `arch` | 0 |
| 8 | 4 | `version` | 5 |
| 12 | 4 | `config` | 0 on v3 |
| 16 | 8 | `info.reg_base` | **Physical address** of the NPU register window (`0x14260000`) |
| 24 | 4 | `cluster_cnt` | Clusters in this partition |
| 28 | 96 | `clusters[8]` | 12-byte entries: `core_cnt` (u32, from hardware), `en_core_cnt` (u32, currently enabled), `tec_cnt` (u32, TECs per core) |
| 124 | 4 | (padding) | |

- QUERY_PARTITION_CAP writes `partition_cnt` × 128 bytes to `arg`. The
  caller must supply room for all partitions, even though the request
  code encodes only one record.

## 3.5 Buffer allocation

### 3.5.1 REQ_BUF (56 bytes) and the buffer descriptor (32 bytes)

Request:

| Offset | Size | Field | Dir | Meaning |
|---|---|---|---|---|
| 0 | 8 | `bytes` | in | Size. Must be non-zero. Rounded up to whole pages (4 KiB on Sky1) |
| 8 | 4 | `align_in_page` | in | Alignment **in pages**. Must be a power of two (0 is rejected) |
| 12 | 4 | `data_type` | in | §3.3. Use **6 (TCB)** for any buffer that holds TCBs |
| 16 | 1 | `region` | in | Requested region type (§3.3). Must be < 4 |
| 17 | 1 | `asid` | in | Requested ASID. Must be < 32 |
| 18 | 6 | (padding) | | |
| 24 | 32 | `desc` | out | Buffer descriptor (below) |

Buffer descriptor (also the argument of FREE_BUF and the cache ioctls):

| Offset | Size | Field | Meaning |
|---|---|---|---|
| 0 | 8 | `pa` | NPU device address (IOVA with an IOMMU). Not a CPU physical address |
| 8 | 8 | `dev_offset` | The `mmap` offset. Equal to `pa` |
| 16 | 8 | `bytes` | Size actually reserved (page-rounded) |
| 24 | 1 | `region` | Region the buffer actually came from |
| 25 | 1 | `asid` | ASID the buffer actually came from |
| 26 | 6 | (padding) | |

Rules:

- The device address (`pa`) is a multiple of `align_in_page` × page size.
- Region fallback: GM and DTCM are not available on v3/Sky1, and SRAM is
  skipped. Every request is served from DDR, and `desc.region` reports 0.
  If SRAM is disabled (§3.5.4), SRAM requests also go to DDR.
- ASID fallback: if the requested ASID (> 0) is full, the next higher
  ASID is tried, up to the ASID count. `desc.asid` reports the one used.
  ASID 0 does not fall back.
- A buffer of type TCB is tracked by the kernel. A job's last task TCB
  must lie inside a TCB-type buffer (§3.7.1). TCB chains of one job must
  lie in one buffer.
- Without an IOMMU, a buffer that would break the 4 GiB / top-1-GiB rule
  (02 §2.8) is freed again and the request fails with `ENOMEM`.
- Errors: `EINVAL` for bad arguments, `ENOMEM` when no space is left.

### 3.5.2 FREE_BUF

Only `desc.pa` is used. It must be an address inside an allocated buffer
(normally its start). Freeing a TCB buffer that is still linked to a
pending job is deferred and returns success. `EINVAL` if no region holds
the address.

### 3.5.3 mmap

- Map with `offset` = `desc.dev_offset` and `length` ≤ `desc.bytes`
  (page multiples). The kernel looks up the region that holds the
  address, and the mapping starts at that address.
- The mapping is **uncached** (strongly ordered / device type on Arm).
  Userspace must not assume unaligned or exclusive accesses work.
- **[POLICY]** The vendor kernel does not bound the mapping to the
  buffer. It allows mapping on to the end of the 1 GiB region. A clean
  implementation should limit the mapping to the buffer.
- `EINVAL` if the offset is in no region.

### 3.5.4 DISABLE_SRAM / ENABLE_SRAM

A per-handle counter of "SRAM disabled" requests. Disable fails with
`EPERM` if any SRAM is allocated, and with `EINVAL` if the driver manages
no memory. Enable fails with `EPERM` if the global count is 0. On Sky1
there is no SRAM, so these only change bookkeeping.

## 3.6 Cache maintenance

BUF_CACHE_FLUSH (clean to the point of coherency) and BUF_CACHE_INVALID
(clean and invalidate) take a buffer descriptor. Only `pa` is used.

- **[POLICY]** *(vendor)* The operation covers the whole IOVA region
  containing `pa`, not just the buffer. It is a no-op without an IOMMU.
  An unknown address returns 0 (success).
- Userspace calls these around CPU writes and reads of NPU data. A
  clean implementation may act on just the buffer range.

## 3.7 Jobs

### 3.7.1 SCHEDULE_JOB descriptor (136 bytes)

| Offset | Size | Field | v3 use |
|---|---|---|---|
| 0 | 4 | `is_defer_run` | 0 = run now. 1 = create a deferred job (with `do_trigger` = 0) or trigger one (with `do_trigger` = 1) |
| 4 | 4 | `version_compatible` | Ignored on v3 |
| 8 | 4 | `core_id` | Core for debug-dispatch. Must be < `en_core_cnt` of cluster 0 when the debug-dispatch flag is set |
| 12 | 4 | `partition_id` | **Required.** Must be < `partition_cnt` |
| 16 | 4 | `do_trigger` | See `is_defer_run` |
| 20 | 4 | `aipu_arch` | 0 |
| 24 | 4 | `aipu_version` | **Must be 5** on v3 hardware |
| 28 | 4 | `aipu_config` | Ignored on v3 |
| 32 | 16 | `start_pc_addr`, `intr_handler_addr`, `data_0_addr`, `data_1_addr` | v1/v2 only, ignored |
| 48 | 8 | `job_id` | Caller-chosen ID, unique per thread. The low 16 bits become the hardware tag |
| 56 | 4 | `enable_prof` | **Must be 0 on Sky1** (see below) |
| 60 | 4 | (padding) | |
| 64 | 8 | `profile_fd` | Ignored in the vendor build |
| 72 | 8 | `profile_pa` | Ignored in the vendor build |
| 80 | 4 | `profile_sz` | Ignored in the vendor build |
| 84 | 4 | `enable_poll_opt` | 0 = completion is waited for by the submitting thread. Non-zero = by any thread using this handle (§3.8). Applies to v3 too |
| 88 | 4 | `exec_flag` | §3.3 flags |
| 92 | 4 | `dtcm_size_kb` | Ignored on v3 |
| 96 | 8 | `head_tcb_pa` | **Required.** Device address of the job's first TCB (first init TCB) |
| 104 | 8 | `first_task_tcb_pa` | **Required.** First task TCB |
| 112 | 8 | `last_task_tcb_pa` | **Required.** Last task TCB. Must be in a TCB-type buffer. The kernel edits its `next` field (02 §2.5.2) |
| 120 | 8 | `tail_tcb_pa` | Last TCB of the chain (informational) |
| 128 | 4 | `is_coredump_en` | Non-zero: on a fault, do not abort the pool; wait for the coredump signal (02 §2.7) |
| 132 | 4 | (padding) | |

Semantics:

- Submission does not block. Completion is reported through `poll` and
  QUERY_STATUS.
- Validation failure: `EINVAL`. NPU suspended or 0 cores enabled:
  `ENODEV`. Hold-TCB allocation failure: `ENOMEM`. A pool that has taken
  a debug-dispatch: `EINVAL`.
- **[POLICY]** If the hardware rejects the dispatch (TSM fail bit), the
  call returns `EFAULT`, but the vendor driver **leaves the job queued**
  in the deferred state. The device then reports busy until the handle is
  closed. A clean implementation should drop the job.
- Deferred jobs (v3): creation only records the job. Triggering looks up
  the job by (calling thread, `job_id`) and dispatches it. If none is
  found: `EINVAL`.
- **[POLICY]** *(vendor)* `enable_prof` ≠ 0 makes the vendor kernel call
  SoC profiling hooks that Sky1 does not provide (a kernel crash). The
  vendor runtime must therefore never set it on Sky1. A clean
  implementation should ignore it and report zero counters.

### 3.7.2 QUERY_STATUS (24 bytes) and status entries (56 bytes)

Query:

| Offset | Size | Field | Dir | Meaning |
|---|---|---|---|---|
| 0 | 4 | `max_cnt` | in | Capacity of the user array. Must be ≥ 1 |
| 4 | 4 | `of_this_thread` | in | 1 = only jobs submitted by the calling thread; 0 = any thread's jobs on this handle |
| 8 | 8 | `status` | in | User pointer to an array of `max_cnt` status entries |
| 16 | 4 | `poll_cnt` | out | Entries written |
| 20 | 4 | (padding) | | |

Status entry:

| Offset | Size | Field | Meaning |
|---|---|---|---|
| 0 | 8 | `job_id` | |
| 8 | 4 | `thread_id` | Kernel thread ID of the submitter |
| 12 | 4 | `state` | 1 done, 2 exception, 3 coredump |
| 16 | 8 | `pdata.tick_counter` | NPU tick counter at the end interrupt, if the tick counter was enabled. Otherwise 0 |
| 24 | 8 | `pdata.execution_time_ns` | 0 on Sky1 |
| 32 | 24 | `pdata` read/write/cycle totals (6 × u32: read MSB, read LSB, write MSB, write LSB, cycle MSB, cycle LSB) | 0 on Sky1 |

- Only **ended** jobs are returned, oldest first, and only jobs
  submitted through this handle. Each returned job is removed from the
  kernel, so a job is reported exactly once.
- Return value: 0 on success. **[POLICY]** If the status array cannot be
  written, the vendor kernel returns the positive number of bytes not
  copied (the jobs are already removed). If the query struct itself
  cannot be read, it returns 0 and does nothing.

### 3.7.3 GET_HW_STATUS (4 bytes)

Writes 1 (busy) if any job is running or deferred, otherwise 0 (idle).
The value 2 (exception) is never returned.

### 3.7.4 ABORT_CMD_POOL and KILL_TIMEOUT_JOB

- ABORT_CMD_POOL (no argument; meant for debuggers): `ENODEV` if no
  pool exists. Otherwise it aborts pool 0, marks every job that has not
  succeeded as exception, and wakes waiters. `EFAULT` if the TSM rejects
  the abort. The jobs are still reported through QUERY_STATUS.
- KILL_TIMEOUT_JOB: documented as v1/v2 only. The kernel reads **8
  bytes** (a u64 job ID) from `arg` although the request code says 4. It
  removes the caller's job with that ID without touching hardware.
  **[POLICY]** The vendor kernel misbehaves if no such job exists. A clean
  implementation should return `EINVAL` and should not need this ioctl
  on v3.

### 3.7.5 Cluster configuration and tick counter

CONFIG_CLUSTERS (32 bytes): eight u32 `en_core_cnt` values, one per
cluster index of partition 0.

- `EBUSY` if any job is running or deferred. `EINVAL` if a value exceeds
  that cluster's core count.
- Otherwise each cluster is reprogrammed (02 §2.10) and the pool is
  destroyed. If the last value written is 0, later submissions fail with
  `ENODEV` until a non-zero configuration is set.

ENABLE_TICK_COUNTER / DISABLE_TICK_COUNTER: a reference count. The
hardware counter starts on the 0 → 1 transition and stops and clears on
the 1 → 0 transition. Both return 0. The counter is off by default.

### 3.7.6 REQ_IO (16 bytes): debug register access

| Offset | Size | Field | Meaning |
|---|---|---|---|
| 0 | 4 | `partition_id` | Must be < `partition_cnt` |
| 4 | 4 | `offset` | Register offset. Must be ≤ `0x322C` |
| 8 | 4 | `rw` | 0 read, 1 write. Other values do nothing |
| 12 | 4 | `value` | Value to write, or value read back |

`EINVAL` on a bad partition or offset. The offset should be a multiple of
4 (the vendor kernel does not check). This works on v3 although the
header says v1/v2.

### 3.7.7 Identity and miscellaneous

- GET_DRIVER_VERSION: `arg` is the address of a caller buffer of at
  least 16 bytes. The kernel writes the NUL-terminated string
  **`"1.0.0"`** (6 bytes). Returns 0 on success.
- ALLOC_GRID_ID: writes a 32-bit int with the next grid ID. On v3 it is a
  16-bit counter starting at 0 that wraps and needs no free.
- ALLOC_GROUP_ID / FREE_GROUP_ID (4 bytes: `group_size` u16 at 0,
  `first_id` u16 at 2): v3_1 only. On v3 both return `EINVAL`.

## 3.8 poll

- `poll` reports readable (`POLLIN | POLLRDNORM`) when at least one ended,
  not-yet-collected job on this handle is visible to the caller:
  submitted with `enable_poll_opt` ≠ 0, or submitted by the polling
  thread.
- Waiters are woken when the completion worker finishes a job (02 §2.6).
  The usual userspace loop is: `poll`, then QUERY_STATUS until
  `poll_cnt` is 0.
- **[POLICY]** Wait queues are kept per submitting thread (or per handle
  with `enable_poll_opt`). A thread that polls without having submitted
  a job is never woken.

## 3.9 dma-buf export and import

ALLOC_DMA_BUF (16 bytes):

| Offset | Size | Field | Dir |
|---|---|---|---|
| 0 | 4 | `fd` | out: new dma-buf file descriptor (read/write, close-on-exec) |
| 4 | 4 | (padding) | |
| 8 | 8 | `bytes` | in: size (non-zero) |

GET_DMA_BUF_INFO and ATTACH_DMA_BUF (24 bytes):

| Offset | Size | Field | Dir |
|---|---|---|---|
| 0 | 4 | `fd` | in: dma-buf fd (> 0) |
| 4 | 4 | (padding) | |
| 8 | 8 | `pa` | out: NPU device address |
| 16 | 8 | `bytes` | out: size |

- **Export**: ALLOC_DMA_BUF allocates DDR in ASID 0 (page-aligned) and
  wraps it as a dma-buf. Importers get a mapping of it. `mmap` on the
  dma-buf maps the same memory uncached. FREE_DMA_BUF(fd) frees the
  backing store.
- GET_DMA_BUF_INFO returns the NPU address and size of a dma-buf that
  this driver exported. **[POLICY]** The vendor kernel does not check that
  the fd really is one of its own. A clean implementation must.
- **Import**: ATTACH_DMA_BUF attaches a foreign dma-buf to the NPU, maps
  it bidirectionally, and returns the device address of its **first**
  segment and that segment's length. The buffer must therefore be
  contiguous in NPU address space, which the IOMMU normally provides.
  DETACH_DMA_BUF(fd) undoes it. `EINVAL` if the fd was not attached.
- An imported buffer is not inside an ASID region. Userspace must make
  sure its address is reachable from the ASID that uses it (02 §2.8).

## 3.10 Error summary

| Error | Meaning |
|---|---|
| `EINVAL` | Bad argument, unknown buffer or fd, copy fault on most ioctls, driver not ready at `open` |
| `ENOMEM` | Out of memory or NPU address space |
| `ENODEV` | NPU suspended or all cores disabled (SCHEDULE_JOB). No command pool (ABORT_CMD_POOL) |
| `EBUSY` | CONFIG_CLUSTERS while jobs are active |
| `EPERM` | SRAM enable/disable conflicts |
| `EFAULT` | TSM rejected a command (dispatch, abort, create) |
| `ENOTTY` | Unknown request |
| positive | QUERY_STATUS, GET_DRIVER_VERSION, ALLOC_GRID_ID when a copy to user memory fails (bytes not copied) |

## 3.11 Sysfs attributes (informative)

**[ABI]** *(vendor)* On the NPU platform device:

| Attribute | Mode | Behaviour |
|---|---|---|
| `ext_registers` | 0644 | Read: a text table of selected registers. Write: `"<offset-hex> <value-hex>"` writes one register (offset must be a multiple of 4) |
| `gm_policy` | 0644 | Read: `"[n] <description>"`. Write: `0`, `1` or `2` sets the GM policy (§3.3) |
