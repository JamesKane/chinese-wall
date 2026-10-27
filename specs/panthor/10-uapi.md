# 10: Userspace ABI (ioctls, mmap, fdinfo, sysfs)

> **Licensing note.** The uAPI header that defines these structures and
> constants (`include/uapi/drm/panthor_drm.h`) is **MIT-licensed**. The clean
> room may use that header directly for structure layouts, enum values and
> ioctl numbers. This section does not transcribe it. It documents the
> **semantics**: validation rules, error codes, side effects, ordering, and
> version history. Field names below are the header's own names, used as
> mnemonics.

The goal is binary compatibility with Mesa's Panthor kernel interface (the CSF
paths of the Gallium driver and of `panvk`). Statements are **[ABI]** unless
tagged otherwise.

## 10.1 General conventions

### 10.1.1 Device node and driver identity

- The driver is a DRM render-capable driver named `panthor`. Every ioctl
  below can be used on the render node and needs no DRM master. The one
  exception is the high/realtime group priorities (10.6.1).
- The driver features are: render node, GEM buffer objects, binary sync
  objects, and **timeline** sync objects. The generic DRM sync-object ioctls
  (create, destroy, wait, signal, timeline query and transfer, sync-file
  import/export) and generic PRIME (dma-buf) handle conversion must therefore
  be provided. The generic GEM-close ioctl releases BO handles.
- The driver version is reported through the generic DRM version ioctl as
  **major 1, minor 9** at this baseline. Userspace checks the minor version to
  decide which features exist (10.11).

### 10.1.2 Structure rules

- Every ioctl argument structure is a multiple of 8 bytes, with explicit
  padding fields. **Padding fields must be zero.** The driver rejects nonzero
  padding with `EINVAL` (exceptions are noted per ioctl).
- New fields are added only at the end of a structure or in place of padding.
  Zero in a new field always means the old behavior.
- The generic DRM ioctl dispatcher accepts a request whose encoded argument
  size differs from the kernel's. It zero-extends shorter input and truncates
  output to the caller's size. A new kernel therefore accepts an old binary's
  shorter structs.

### 10.1.3 Object arrays (stride-versioned)

Variable-length arrays are passed as an *object array* descriptor of 16 bytes:
`stride` (u32), `count` (u32), `array` (u64 user pointer).

Rules for every object array the kernel reads:

1. `count == 0`: the array is empty, and `stride` and `array` are ignored.
2. `stride` smaller than the element's **minimum size** gives `EINVAL`. The
   minimum size is the size of the element as it was when it first entered the
   uAPI (table below).
3. `stride` equal to the kernel's element size: the array is copied in one
   block.
4. `stride` smaller than the kernel size (but at least the minimum): each
   element is copied and the kernel zero-fills the fields it lacks.
5. `stride` larger than the kernel size: each element is copied, and any
   **nonzero** byte beyond the kernel's size gives `E2BIG`. Zero bytes there are
   ignored.
6. A bad user pointer gives `EFAULT`. Allocation failure gives `ENOMEM`.

| Element | Kernel size (bytes) | Minimum size (bytes) |
|---|---|---|
| sync op | 16 | 16 |
| queue submit | 40 | 40 |
| queue create | 8 | 8 |
| VM bind op | 48 | 48 |
| BO sync op | 24 | 24 |

### 10.1.4 Output query structures (DEV_QUERY)

When the kernel writes a structure out to a user buffer of size *U*:

1. *U* smaller than the structure's minimum size gives `EINVAL`.
2. The kernel copies min(*U*, kernel size) bytes.
3. If *U* is bigger than the kernel size, the extra bytes are cleared to zero.

### 10.1.5 Common errors

| Error | Meaning |
|---|---|
| `ENODEV` | The device was unplugged or removed. Returned by VM_CREATE, BO_CREATE, VM_BIND and GROUP_SUBMIT, and by mmap |
| `ENOENT` | Unknown BO handle (BO ioctls), or unknown sync object handle in a wait |
| `EINVAL` | Any malformed argument, unknown VM, group or heap handle, or a sync object handle in a signal op that cannot be found |
| `EFAULT` | Bad user pointer |
| `E2BIG` | Nonzero trailing bytes in an over-sized element (10.1.3), or label too long |
| `EBUSY` | Per-file object table full (VMs, groups, heaps per VM) |
| `EINTR` / restart | An interruptible wait (reservation locking, GPU wake-up) was interrupted |

## 10.2 Ioctl table

Every ioctl is read-write (`_IOWR`) on the DRM ioctl type `'d'`. The command
number is `0x40` (the driver command base) plus the index.

| Index | Nr | Name | Arg size | Request code | Since |
|---|---|---|---|---|---|
| 0 | 0x40 | DEV_QUERY | 16 | `0xC0106440` | 1.0 |
| 1 | 0x41 | VM_CREATE | 16 | `0xC0106441` | 1.0 |
| 2 | 0x42 | VM_DESTROY | 8 | `0xC0086442` | 1.0 |
| 3 | 0x43 | VM_BIND | 24 | `0xC0186443` | 1.0 |
| 4 | 0x44 | VM_GET_STATE | 8 | `0xC0086444` | 1.0 |
| 5 | 0x45 | BO_CREATE | 24 | `0xC0186445` | 1.0 |
| 6 | 0x46 | BO_MMAP_OFFSET | 16 | `0xC0106446` | 1.0 |
| 7 | 0x47 | GROUP_CREATE | 56 | `0xC0386447` | 1.0 |
| 8 | 0x48 | GROUP_DESTROY | 8 | `0xC0086448` | 1.0 |
| 9 | 0x49 | GROUP_SUBMIT | 24 | `0xC0186449` | 1.0 |
| 10 | 0x4A | GROUP_GET_STATE | 16 | `0xC010644A` | 1.0 |
| 11 | 0x4B | TILER_HEAP_CREATE | 40 | `0xC028644B` | 1.0 |
| 12 | 0x4C | TILER_HEAP_DESTROY | 8 | `0xC008644C` | 1.0 |
| 13 | 0x4D | BO_SET_LABEL | 16 | `0xC010644D` | 1.4 |
| 14 | 0x4E | SET_USER_MMIO_OFFSET | 8 | `0xC008644E` | 1.5 |
| 15 | 0x4F | BO_SYNC | 16 | `0xC010644F` | 1.7 |
| 16 | 0x50 | BO_QUERY_INFO | 16 | `0xC0106450` | 1.7 |

Direction of the main fields ("in", "out", "in/out"):

| Ioctl | In | Out |
|---|---|---|
| DEV_QUERY | `type`, `size`, `pointer` (and, for timestamp, the pointed-to `flags`) | `size` (only when `pointer` = 0), the pointed-to structure |
| VM_CREATE | `flags`, `user_va_range` | `id`, `user_va_range` |
| VM_DESTROY | `id` | — |
| VM_BIND | `vm_id`, `flags`, `ops` | `ops.count` (synchronous mode, on failure) |
| VM_GET_STATE | `vm_id` | `state` |
| BO_CREATE | `size`, `flags`, `exclusive_vm_id` | `size` (rounded), `handle` |
| BO_MMAP_OFFSET | `handle` | `offset` |
| GROUP_CREATE | everything except `group_handle` | `group_handle` |
| GROUP_DESTROY | `group_handle` | — |
| GROUP_SUBMIT | `group_handle`, `queue_submits` | — |
| GROUP_GET_STATE | `group_handle` | `state`, `fatal_queues` (and see 10.6.4) |
| TILER_HEAP_CREATE | `vm_id`, `initial_chunk_count`, `chunk_size`, `max_chunks`, `target_in_flight` | `handle`, `tiler_heap_ctx_gpu_va`, `first_heap_chunk_gpu_va` |
| TILER_HEAP_DESTROY | `handle` | — |
| BO_SET_LABEL | `handle`, `label` | — |
| SET_USER_MMIO_OFFSET | `offset` | — |
| BO_SYNC | `ops` | — |
| BO_QUERY_INFO | `handle` | `extra_flags`, `create_flags`, `pad` (zeroed) |

## 10.3 DEV_QUERY

Two-step protocol:

- **Size probe** (`pointer` = 0): for a known `type`, the kernel writes the
  current kernel structure size into `size` and returns 0. An unknown `type`
  gives `EINVAL`.
- **Fetch** (`pointer` ≠ 0): the kernel copies the structure following 10.1.4,
  with `size` as the user buffer size. An unknown `type` gives `EINVAL`.

> Note: the header text says `size` is updated after a fetch too. The driver
> does **not** write `size` back on a fetch, only on a size probe. Userspace
> must not rely on `size` changing after a fetch.

| Type | Value | Structure | Kernel size | Minimum size | Since |
|---|---|---|---|---|---|
| GPU_INFO | 0 | GPU information | 112 | 96 (through `tiler_present`) | 1.0 |
| CSIF_INFO | 1 | Command-stream interface information | 24 | 24 | 1.0 |
| TIMESTAMP_INFO | 2 | Timestamp information | 56 | 16 (through `current_timestamp`) | 1.1 (flags in 1.8) |
| GROUP_PRIORITIES_INFO | 3 | Allowed group priorities | 4 | 4 | 1.2 |
| MMU_INFO | 4 | MMU information | 8 | 8 | 1.9 |

### 10.3.1 GPU_INFO

Fixed at probe time. Most fields are raw register values (02-register-map):

| Field | Source |
|---|---|
| `gpu_id` | `GPU_ID` |
| `gpu_rev` | `REVID` |
| `csf_id` | `CSF_ID` |
| `l2_features`, `tiler_features`, `mem_features`, `mmu_features`, `core_features` | Same-named feature registers |
| `thread_features`, `max_threads`, `thread_max_workgroup_size`, `thread_max_barrier_size` | `THREAD_*` registers |
| `coherency_features` | `COHERENCY_FEATURES` (what the GPU supports, not what is in use) |
| `texture_features[0..3]` | `TEXTURE_FEATURES[0..3]` |
| `as_present` | `AS_PRESENT` |
| `selected_coherency` | Coherency protocol actually in use: 0 ACE-Lite, 31 none (09 section 9.3.1). Since 1.7. Before 1.7 these bytes were padding and read as 0 |
| `shader_present`, `l2_present`, `tiler_present` | `*_PRESENT` from the GPU block (arch ≤ 13) or the PWR block (arch 14+). `shader_present` may be narrowed by a platform-provided "shader-present" value (01-platform), for example on SoCs with fused-off cores |
| `core_features` | `CORE_FEATURES` |
| `pad` | 0 |
| `gpu_features` | 64-bit `GPU_FEATURES` (arch 11+, otherwise 0) |

The header supplies helper decodings for `gpu_id`, `csf_id` and the
`mmu_features` VA-bits field. **[ABI]** Mesa picks its hardware backend from
`gpu_id` and sizes its VA layout from the VA bits.

### 10.3.2 CSIF_INFO

| Field | Meaning |
|---|---|
| `csg_slot_count` | Number of CSG slots the firmware exposes and the scheduler uses (07) |
| `cs_slot_count` | Command streams per CSG, which is the maximum queues per group |
| `cs_reg_count` | CS work registers per stream: firmware CS features [7:0] + 1 |
| `scoreboard_slot_count` | Scoreboard slots per stream |
| `unpreserved_cs_reg_count` | **4**. The kernel's ring-buffer code freely clobbers the top 4 CS registers (indices `cs_reg_count − 4` … `cs_reg_count − 1`) when it calls a userspace stream. Userspace must not expect them to survive across a submission |
| `pad` | 0 |

> Note: the header's description writes the clobbered range in terms of
> `cs_slot_count`. That is a typo. The range is relative to `cs_reg_count`.

### 10.3.3 TIMESTAMP_INFO

This query **reads input**: the kernel first reads the user structure, with
the size rules of 10.1.3 rule 5 applied to `size`, so it can get `flags`.

`flags`:

| Bits | Meaning |
|---|---|
| 0 | Sample the GPU timestamp (`current_timestamp`) |
| [3:1] | CPU clock: 0 = none, 1 = monotonic, 2 = monotonic-raw. Other values give `EINVAL` |
| 4 | Read the GPU timestamp offset register (`timestamp_offset`) |
| 5 | Sample the GPU cycle counter (`cycle_count`) |
| 6 | Report the timestamp frequency (`timestamp_frequency`) |
| 7 | Report how long the sampling took (`duration_nsec`) |
| others | `EINVAL` |

- `flags` = 0 means bits 0, 4 and 6 (the version 1.1 behavior). A pre-1.8
  caller with a 24-byte or shorter buffer gets exactly that.
- Fields not requested come back as 0.
- `timestamp_frequency` is the frequency of the system counter that drives
  the GPU timestamp. It is 0 if the platform has no architected timer.
- The GPU is powered up for the query. This can fail, and the error is
  returned.
- **[POLICY]** If two or more sources are requested (GPU timestamp, CPU
  clock, cycle counter), or the duration is requested, sampling runs with
  preemption and local interrupts disabled so that the samples are close
  together. Order: GPU timestamp, then CPU clock, then cycle counter.
  `duration_nsec` covers all three.
- CPU timestamps are adjusted for the caller's time namespace and returned
  as seconds plus nanoseconds.
- **[HW]** Since 1.6 the firmware is told to enable its cycle counter, so
  `CYCLE_COUNT` actually advances (06-csf-interface).

### 10.3.4 GROUP_PRIORITIES_INFO

`allowed_mask` bit *p* is set when the caller may create groups of priority
*p* (0 low, 1 medium, 2 high, 3 realtime). Low and medium are always allowed.
High and realtime need the `CAP_SYS_NICE` capability **or** being the current
DRM master. `pad[3]` = 0.

### 10.3.5 MMU_INFO

`page_size_bitmap`: GPU page sizes that VM_BIND supports. At this baseline it
is 4 KiB | 2 MiB (`0x201000`). See 04-mmu.

## 10.4 VM ioctls

Details of address-space layout, page tables and the bind engine are in 04-mmu.
This section gives the userspace-visible rules.

### 10.4.1 VM_CREATE

- `flags` ≠ 0 gives `EINVAL`.
- Let *F* = 2^(VA bits from `mmu_features`). If *F* < 256 MiB, fail with
  `EINVAL`.
- User VA range:
  - `user_va_range` ≠ 0: the requested size. It fails with `EINVAL` if larger
    than *F* − 256 MiB.
  - `user_va_range` = 0: **[POLICY]** default. If the calling process's CPU
    address-space size is below *F*, use that size, so CPU and GPU addresses
    can match. Otherwise use *F*/2 if *F* > 4 GiB, else *F* − 256 MiB. The
    result is capped at *F* − 256 MiB.
- The kernel VA range is the largest power of two that fits in
  *F* − user range, placed at the top of the space. On return,
  `user_va_range` holds **the start of the kernel range**. Because of the
  power-of-two rounding this can be larger than the requested value. Userspace
  owns [0, returned `user_va_range`).
- `id` gets a VM handle in 1..32. **[POLICY]** At most 32 live VMs per file;
  beyond that, `EBUSY`.
- No GPU page tables are made active until a job runs (04-mmu).

### 10.4.2 VM_DESTROY

- `pad` ≠ 0 gives `EINVAL`. Unknown `id` gives `EINVAL`.
- Effects: the handle is released at once. The VM's tiler heap pool and all
  its heaps are destroyed (08). Every mapping in the VM is removed. New heap
  creation and new bind jobs on it are refused. Groups bound to the VM keep it
  alive internally, but their in-flight work will fault. **[POLICY]** Teardown
  first waits for any pending scheduler pass, so firmware state write-back
  during group suspend cannot fault (08 section 8.5).

### 10.4.3 VM_BIND

Argument: `vm_id`, `flags`, and an object array of bind ops (48 bytes each:
`flags`, `bo_handle`, `bo_offset`, `va`, `size`, `syncs`).

- Unknown `vm_id` gives `EINVAL`.
- `flags` bit 0 = **ASYNC**.

> OPEN QUESTION: the driver does not reject unknown bits in the VM_BIND
> `flags` field. They are silently ignored. A clean implementation should
> probably reject them, but strictness here is not observed ABI.

Per-op validation (both modes):

| Check | Error |
|---|---|
| `va`, `size` and `bo_offset` must all be multiples of the VM page size (4 KiB) | `EINVAL` |
| [`va`, `va` + `size`) must lie inside the user VA range (no overflow, no overlap with the kernel range) | `EINVAL` |
| Op type (`flags` bits [31:28]) must be 0 MAP, 1 UNMAP or 2 SYNC_ONLY | `EINVAL` |
| **MAP**: only READONLY (bit 0), NOEXEC (1), UNCACHED (2) and SPARSE (3) may be set besides the type | `EINVAL` |
| MAP, not sparse: `bo_handle` must name a BO. `size` ≤ BO size and `bo_offset` + `size` ≤ BO size | `EINVAL` (an unknown handle also gives `EINVAL` here, not `ENOENT`) |
| MAP, sparse (1.9+): `bo_handle` = 0, `bo_offset` = 0 and NOEXEC set. The range is backed by a per-file 2 MiB dummy buffer repeated cyclically. GPU reads return undefined data, and writes land in shared scratch memory | `EINVAL` |
| MAP of a VM-private BO into a different VM | `EINVAL` |
| MAP into a VM marked unusable | `EINVAL` |
| **UNMAP**: no flag bits besides the type. `bo_handle` = 0 and `bo_offset` = 0 | `EINVAL` |
| **SYNC_ONLY**: no flag bits besides the type. `bo_handle`, `bo_offset`, `va` and `size` all 0. `syncs` not empty. Only meaningful in ASYNC mode | `EINVAL` |

MAP semantics: a new mapping replaces any existing mappings it overlaps,
splitting them where needed. UNMAP removes whatever falls in the range, which
may also split mappings. READONLY makes the mapping GPU read-only. NOEXEC makes
it non-executable. UNCACHED makes it GPU non-cacheable (09 section 9.3.3).

**Synchronous mode** (ASYNC clear):

- Every op must have an empty `syncs` array, otherwise `EINVAL`.
- An op with `size` = 0 is skipped as a no-op, before any other check.
- Ops run one by one in array order. The page tables are updated before the
  ioctl returns.
- On the first failure the kernel writes the index of the failing op to
  `ops.count` (= number of ops that completed) and returns the error. Earlier
  ops stay in effect.
- **[ABI]** No implicit wait happens on GPU work that may still use an
  unmapped range. Userspace has to synchronize first.

**Asynchronous mode** (ASYNC set):

- All ops are validated and turned into jobs on the VM's private, in-order bind
  queue **before** anything is queued. Any validation error fails the whole
  ioctl, and nothing is queued. A VM that is destroyed or unusable rejects new
  async ops with `EINVAL`.
- Each op's `syncs` are processed as in 10.5 (waits become dependencies,
  signals receive that op's completion fence). Signals become visible only
  after the whole ioctl succeeds.
- Ops run in array order on the VM bind queue, each once its dependencies
  have signalled.
- If an op fails while running (for example page-table allocation), the VM is
  marked **unusable**: later MAP ops fail, jobs targeting the VM fail, and
  VM_GET_STATE reports UNUSABLE. UNMAP ops are still accepted. The only
  recovery is a new VM.

### 10.4.4 VM_GET_STATE

- Unknown `vm_id` gives `EINVAL`. `state` is output only and not checked.
- `state` = 0 USABLE, 1 UNUSABLE (10.4.3).

## 10.5 Synchronization operations (sync ops)

GROUP_SUBMIT queue submits and async VM_BIND ops each carry a `syncs` array of
16-byte sync ops: `flags`, `handle`, `timeline_value`.

`flags`:

| Bits | Meaning |
|---|---|
| [7:0] | Handle type: 0 binary sync object, 1 timeline sync object. Others give `EINVAL` |
| 31 | 0 = **wait**, 1 = **signal** |
| [30:8] | Must be 0 (`EINVAL`) |

- For a binary sync object, `timeline_value` must be 0 (`EINVAL`).
- For a timeline object, `timeline_value` is the point. **[ABI]** A timeline
  op with point 0 acts on the object's current fence: a wait waits for it, a
  signal replaces it, just like a binary object.

**Batch processing**, identical for GROUP_SUBMIT (one job per queue submit)
and async VM_BIND (one job per op):

1. Make all jobs and validate them. Any failure fails the ioctl with nothing
   submitted.
2. Collect every **signal** op across all jobs. For each distinct
   (handle, point), look up the sync object (`EINVAL` if missing) and note the
   fence currently attached at that point, if there is one.
3. Lock and prepare the reservation objects. GROUP_SUBMIT also restores any
   evicted BOs in the VM (09 section 9.8).
4. For each job in array order:
   - For each **wait** op: if some job *earlier in this batch* signals the
     same (handle, point), depend on that job's fence. Otherwise use the
     fence noted in step 2. If the pair is not signalled anywhere in the batch,
     look up the object's fence at that point.
     - Unknown handle gives `ENOENT`.
     - No fence (a binary object never signalled, or a timeline point not yet
       submitted) gives `EINVAL`. There is no wait-for-submit behavior, so
       userspace must submit signals before the waits that depend on them.
     - A timeline point that has already signalled and been collected waits on
       an already-signalled stub.
   - Arm the job, then record its completion fence as the new fence of every
     (handle, point) it signals, so later jobs in the batch pick it up.
5. From here nothing can fail. Push all jobs to their queues in array order.
   Then publish the fences: a binary object (or point 0) gets its fence
   **replaced**. A timeline point gets a new point **added** to the timeline
   chain.

Consequences:

- A wait in job *k* on a pair that only a *later* job *j* > *k* signals binds
  to the pair's pre-batch fence, not to job *j*. If there is no pre-batch fence
  the submission fails with `EINVAL`.
- A pair signalled by several jobs ends up holding the last such job's fence.
- Out-fences are sync objects only. There is no separate out-fence fd. Use the
  generic sync-object-to-sync-file export for that.

## 10.6 Group ioctls

Group and queue scheduling semantics are in 07-scheduler.

### 10.6.1 GROUP_CREATE

Checks run in this order:

| # | Check | Error |
|---|---|---|
| 1 | `queues.count` is 0 or greater than 32 | `EINVAL` |
| 2 | Queue array copy (10.1.3) | `EINVAL` / `E2BIG` / `EFAULT` / `ENOMEM` |
| 3 | `priority` > 3 | `EINVAL` |
| 4 | `priority` is high (2) or realtime (3) and the caller has neither `CAP_SYS_NICE` nor DRM master | `EACCES` |
| 5 | `pad` ≠ 0 | `EINVAL` |
| 6 | `compute_core_mask` or `fragment_core_mask` not a subset of `shader_present`, or `tiler_core_mask` not a subset of `tiler_present` | `EINVAL` |
| 7 | popcount of a mask smaller than its `max_*_cores` | `EINVAL` |
| 8 | `queues.count` greater than CSIF `cs_slot_count` | `EINVAL` |
| 9 | Unknown `vm_id` | `EINVAL` |
| 10 | Per queue: `pad[0..2]` ≠ 0, `ringbuf_size` not a power of two in [4 KiB, 64 KiB], or queue `priority` > 15 | `EINVAL` |
| 11 | More than 128 live groups in the file | `EBUSY` |
| — | Allocation failures (firmware suspend buffers, ring buffers, sync objects) | `ENOMEM` etc. |

- On success `group_handle` is 1..128.
- Side effects: per-group and per-queue kernel buffers are allocated (09
  section 9.7). The group starts idle and is not resident on a CSG slot until
  it has work.
- `max_*_cores` = 0 is not rejected.

> OPEN QUESTION: what the firmware does with a group whose maximum core count
> is 0 is not shown.

### 10.6.2 GROUP_DESTROY

- `pad` ≠ 0 gives `EINVAL`. A handle that is unknown or not yet fully created
  gives `EINVAL`.
- Effects: the handle is freed at once and can be reused. The group is marked
  destroyed, so further submits fail. If it is resident it is evicted at the
  next scheduler pass. Otherwise it is removed from the run queues and
  terminated. Its unfinished jobs are cancelled and their fences signal with an
  error (07-scheduler).

### 10.6.3 GROUP_SUBMIT

Argument: `group_handle`, `pad`, and an object array of queue submits (40 bytes
each: `queue_index`, `stream_size`, `stream_addr`, `latest_flush`, `pad`,
`syncs`).

| Check | Error |
|---|---|
| Top-level `pad` ≠ 0 | `EINVAL` |
| Queue-submit `pad` ≠ 0 | `EINVAL` |
| Exactly one of `stream_size` and `stream_addr` is 0 | `EINVAL` |
| `stream_addr` not 64-byte aligned, or `stream_size` not a multiple of 8 | `EINVAL` |
| `latest_flush` has any of bits [30:24] set | `EINVAL` |
| Unknown group, or group destroyed, timed out, fatally faulted or terminated | `EINVAL` |
| `queue_index` ≥ the group's queue count | `EINVAL` |
| Sync op errors | 10.5 |

- All queue submits in one call go to the **same group**, and therefore the
  same VM. `group_handle` is effectively taken as 16 bits.
- `count` = 0 is a valid no-op.
- Each queue submit becomes one job on its queue. The kernel writes a few
  instructions into the queue's ring buffer that flush caches when needed,
  call the user stream (`stream_addr`, `stream_size`), and update the group's
  sync object (07-scheduler).
- **Empty submit** (`stream_size` = 0 and `stream_addr` = 0): pure sync
  point. It runs no GPU work. Its completion fence is that of the previous job
  on the same queue, after its own waits are met. `latest_flush` is ignored.
- **`latest_flush`**: the value userspace read from the `LATEST_FLUSH_ID` page
  (09 section 9.5.2) when it built the stream. The kernel skips its automatic
  pre-job cache flush and invalidate if no flush is needed since that ID. 0
  forces the flush.
- **Ordering**: jobs on one queue run in submission order. Queues of the same
  group, and different groups, run concurrently with no implicit ordering.
  Cross-queue ordering must be expressed with sync ops, or with CS-level
  synchronization in the command streams.
- Job completion fences are attached to the VM's reservation and to every
  external BO mapped in the VM, with **bookkeeping** usage. They take no part
  in implicit synchronization (09 section 9.6.3).
- **[POLICY]** Each job uses ring-buffer space that depends on the profiling
  mask in force when it is created (10.10). A job waits for space if the ring
  is full.
- **[POLICY]** A job that runs longer than the timeout (5 s at this baseline)
  causes the group to be flagged timed out and may trigger a GPU reset
  (07-scheduler).

### 10.6.4 GROUP_GET_STATE

- `pad` ≠ 0 gives `EINVAL`. Unknown handle gives `EINVAL`.
- On success the kernel **zeroes the whole structure**, including
  `group_handle`, before filling in:

| `state` bit | Name | Meaning | Since |
|---|---|---|---|
| 0 | TIMEDOUT | The group had unfinished jobs when it was stopped (timeout or reset) | 1.0 |
| 1 | FATAL_FAULT | At least one queue hit a fatal fault. The bitmask is in `fatal_queues` | 1.0 |
| 2 | INNOCENT | The group was killed by a reset that other groups caused. Only set together with TIMEDOUT and without FATAL_FAULT | 1.3 |

- Once TIMEDOUT or FATAL_FAULT is set, the group accepts no more submits.
  **[ABI]** Mesa uses this to detect device loss and recreate the group or
  context, using INNOCENT to tell guilty contexts from innocent ones.

## 10.7 Tiler heap ioctls

See 08-heap for the memory model and the OOM protocol.

### 10.7.1 TILER_HEAP_CREATE

| Check | Error |
|---|---|
| Unknown `vm_id`, or VM being destroyed | `EINVAL` |
| `initial_chunk_count` = 0, or > `max_chunks` | `EINVAL` |
| `chunk_size` not page-aligned, or outside [128 KiB, 8 MiB] | `EINVAL` |
| 128 heaps already in this VM | `EBUSY` |
| Allocation failure | `ENOMEM` |

- There is no padding field. `target_in_flight` is not range-checked.
- Output:
  - `handle` = (`vm_id` << 16) | heap index (0–127).
  - `tiler_heap_ctx_gpu_va` = the heap context address. Userspace gives this
    to the tiler and firmware through its command streams.
  - `first_heap_chunk_gpu_va` = plain VA of the head chunk of the
    pre-linked initial list (08 section 8.3.4).

### 10.7.2 TILER_HEAP_DESTROY

- `pad` ≠ 0 gives `EINVAL`.
- VM = `handle` >> 16. Unknown VM gives `EINVAL`. VM has never had a heap (no
  pool) gives `ENOENT`. Heap index (`handle` & 0xFFFF) not live gives `EINVAL`.
- The heap's memory is freed **immediately**, without waiting for the GPU.
  Userspace must make sure the heap is idle first.

## 10.8 BO ioctls

The memory model is in 09-memory.

### 10.8.1 BO_CREATE

| Check | Error |
|---|---|
| `size` = 0, `pad` ≠ 0, or unknown `flags` bits (only bit 0 NO_MMAP and bit 1 WB_MMAP are valid) | `EINVAL` |
| NO_MMAP and WB_MMAP both set | `EINVAL` |
| `exclusive_vm_id` ≠ 0 and not a live VM of this file | `EINVAL` |

- On success: `size` = the size rounded up to the CPU page size, and `handle`
  = a nonzero GEM handle. No memory is allocated yet (09 section 9.1).
- `exclusive_vm_id` ≠ 0 makes a VM-private BO (09 section 9.2).

### 10.8.2 BO_MMAP_OFFSET

- `pad` ≠ 0 gives `EINVAL`. Unknown handle gives `ENOENT`. A NO_MMAP BO gives
  `EPERM`.
- `offset` = the fake offset to pass to mmap on the DRM fd (09 section 9.5.1).

### 10.8.3 BO_SET_LABEL (1.4+)

- `pad` ≠ 0 gives `EINVAL`. Unknown handle gives `ENOENT`.
- `label` = 0 clears the label. Otherwise it points to a NUL-terminated string
  of at most 4096 bytes including the NUL. Longer gives `E2BIG`, and a bad
  pointer gives `EFAULT`.
- An empty string is a distinct, valid label.

### 10.8.4 SET_USER_MMIO_OFFSET (1.5+)

- `offset` must be exactly `1 << 43` or `1 << 56`, otherwise `EINVAL`. It sets
  the per-file base of the user MMIO mmap range (09 section 9.5.2). Mappings
  made earlier are not affected.

### 10.8.5 BO_SYNC (1.7+)

Object array of 24-byte ops (`handle`, `type`, `offset`, `size`). Semantics,
order of checks and errors are in 09 section 9.4. Summary:

- empty array: succeed
- unknown handle: `ENOENT`
- out of range: `EINVAL`
- imported BO: `EINVAL`
- unknown type: `EINVAL`
- write-combine-mapped BO, zero size, or no pages yet: succeed doing nothing

Processing stops at the first error.

### 10.8.6 BO_QUERY_INFO (1.7+)

- Unknown handle gives `ENOENT`. Input `pad` is **not** checked, and it is
  written back as 0.
- `create_flags` = the BO_CREATE flags (0 for foreign imports).
- `extra_flags` bit 0 = IS_IMPORTED: the BO came from a **different** driver's
  dma-buf. A dma-buf this device exported and then re-imported is not flagged
  (09 section 9.6.2).
- **[ABI]** Mesa uses this on imported buffers to decide whether it may do
  CPU cache maintenance itself or must use dma-buf sync.

## 10.9 mmap on the DRM file descriptor

| File offset | Target |
|---|---|
| < user MMIO offset | BO mapping by fake offset (09 section 9.5.1) |
| = user MMIO offset + 0 | `LATEST_FLUSH_ID` page: read-only, shared, exactly one CPU page (09 section 9.5.2) |
| other offsets ≥ user MMIO offset | `EINVAL` |

The user MMIO offset defaults to `1 << 56` for 64-bit callers and `1 << 43`
for 32-bit callers, and can be changed with SET_USER_MMIO_OFFSET. The
header-provided convenience constant picks the right default from the size of
`unsigned long`. Offsets are compared as unsigned 64-bit values.

## 10.10 Profiling knobs, fdinfo and sysfs

### 10.10.1 sysfs `profiling`

- Attribute file: `profiling` in the GPU platform device's sysfs directory,
  for example `/sys/bus/platform/drivers/panthor/<addr>.gpu/profiling`.
  Readable and writable.
- Value: a bitmask, accepting decimal, hex (`0x`) or octal input.
  - bit 0 = sample **cycle counts** per job
  - bit 1 = sample **timestamps** per job
  - Other bits give `EINVAL`.
- Default is 0 (disabled), to save power.
- **[POLICY]** The mask is captured **per job when the job is created**.
  Changing it affects only later submissions. Profiled jobs carry extra
  ring-buffer instructions that write cycle and timestamp samples
  before and after into the queue's profiling slots (09 section 9.7). Those
  samples are summed per group and then per file.

### 10.10.2 fdinfo keys

Reading the DRM fd's fdinfo yields the generic DRM client keys
(`drm-driver: panthor`, `drm-client-id`, and so on) plus:

| Key | Present when | Value |
|---|---|---|
| `drm-engine-panthor` | Timestamp profiling enabled, and the platform has an architected timer | Total GPU time used by the file's jobs, in ns. Timestamp ticks are converted using the system counter frequency |
| `drm-cycles-panthor` | Cycle profiling enabled | Total GPU cycles used by the file's jobs |
| `drm-maxfreq-panthor` | Always | Maximum GPU clock rate, in Hz |
| `drm-curfreq-panthor` | Always | Current GPU clock rate, in Hz (from the frequency scaler) |
| `panthor-resident-memory` | Always | Driver-internal BO memory: group kernel buffers plus heap pools (09 section 9.10) |
| `panthor-active-memory` | Always | The part of the above belonging to resident groups or VMs holding an address space |
| `drm-total-memory`, `drm-shared-memory`, `drm-resident-memory`, `drm-active-memory`, `drm-purgeable-memory` | Always | Generic DRM accounting of handle-visible BOs. Purgeable is always 0 |

The engine name is `panthor`. Sizes use the generic DRM size printing (KiB/MiB
suffixes).

### 10.10.3 debugfs (informative, not ABI)

**[POLICY]** debugfs offers a per-device BO listing (creator, size,
residency, eviction count, flags, label), a VM/mapping dump, scheduler state,
and a writable `shrink` file that runs a reclaim pass of N pages. None of
these is a stable interface.

## 10.11 Driver version history

| Minor | Changes visible to userspace |
|---|---|
| 0 | Initial interface |
| 1 | DEV_QUERY TIMESTAMP_INFO |
| 2 | DEV_QUERY GROUP_PRIORITIES_INFO. REALTIME group priority (3) |
| 3 | GROUP_GET_STATE INNOCENT flag |
| 4 | BO_SET_LABEL ioctl |
| 5 | SET_USER_MMIO_OFFSET ioctl |
| 6 | Firmware cycle counter enabled, so cycle-count values are meaningful |
| 7 | BO_CREATE WB_MMAP flag. BO_SYNC and BO_QUERY_INFO ioctls. GPU_INFO `selected_coherency` |
| 8 | TIMESTAMP_INFO `flags`, plus cycle count, CPU timestamp and duration fields |
| 9 | DEV_QUERY MMU_INFO. VM_BIND MAP_SPARSE flag |

Userspace is expected to check the minor version rather than probe by trial
and error. A clean implementation that reports minor *N* must implement every
feature up to *N*.

## 10.12 What Mesa relies on

No Mesa source was available for this extraction. The items below combine
what the kernel interface makes evident with general knowledge of Mesa's
Panthor backend, and need to be checked against a Mesa checkout.

- Device open: the version ioctl, then DEV_QUERY GPU_INFO and CSIF_INFO.
  GROUP_PRIORITIES_INFO from 1.2, TIMESTAMP_INFO for timestamp queries and
  calibration, MMU_INFO from 1.9 to decide on sparse support.
- mmap of the `LATEST_FLUSH_ID` page at device open, read before each submit
  to fill `latest_flush`.
- One VM per context, often with an explicit `user_va_range`. VM-private BOs
  (`exclusive_vm_id`) for most driver-internal allocations.
- Async VM_BIND with timeline sync objects to order bind operations against
  submissions. Sync VM_BIND for simple cases. VM_GET_STATE after errors.
- TILER_HEAP_CREATE per context, with the heap context VA and first-chunk VA
  programmed into command streams. Incremental rendering is implemented by
  Mesa's tiler-OOM exception handler (08 section 8.4.3).
- GROUP_SUBMIT with sync ops on binary and timeline sync objects, empty
  submits as pure sync points, and GROUP_GET_STATE to detect device loss.
- Newer Mesa: WB_MMAP + BO_SYNC for cached host-visible memory, BO_QUERY_INFO
  on imported dma-bufs, and BO_SET_LABEL for debugging.

> OPEN QUESTION: Each of the items above needs checking against Mesa
> (`src/panfrost/lib/kmod/` and `src/panfrost/vulkan/csf/`). In particular:
> whether Mesa ever depends on the unchecked VM_BIND `flags` bits, on the
> GROUP_GET_STATE zeroing of `group_handle`, or on `size` being left unchanged
> after a DEV_QUERY fetch.
