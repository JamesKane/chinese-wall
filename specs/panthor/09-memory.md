# 09: Buffer Objects and Memory Management

This section covers buffer objects (BOs). It describes how they are created and
backed, how CPU and GPU caching is handled (especially on non-coherent SoCs
such as RK3588), mmap semantics (including the flush-ID MMIO page), dma-buf
sharing, kernel-internal buffers, memory reclaim, and labels.

GPU virtual address spaces, page tables and the VM_BIND engine are in 04-mmu.
Ioctl argument layouts and error codes are collected in 10-uapi.

## 9.1 Buffer object model

- **[ABI]** A BO is a page-granular block of system memory that a DRM file
  refers to by a nonzero 32-bit handle. Its size is the requested size rounded
  up to the CPU page size. The rounded size is returned to userspace.
- **[POLICY]** Memory is **allocated lazily**: creating a BO reserves no pages.
  Pages are allocated on first use, which is any of:
  - a CPU page fault on a userspace mapping,
  - a kernel CPU mapping,
  - a GPU map operation (VM_BIND map), or
  - a BO sync request on a BO that already has pages (no-op otherwise).
- **[POLICY]** Fresh pages are zero-filled. This matters for security, because
  userspace must never see old data.
- **[POLICY]** Pages are allocated from non-movable memory. BOs may be pinned
  for long periods, and movable or CMA regions would be blocked by that.
- **[POLICY]** If transparent huge pages are available, BO storage comes from a
  dedicated shared-memory mount that hands out huge pages when the object size
  allows ("within size" policy). A module parameter, enabled by default, can
  turn this off. Huge pages let the GPU map 2 MiB blocks (04-mmu) and the CPU
  map 2 MiB entries.

## 9.2 Creation flags

| Flag | Bit | Meaning | Tag |
|---|---|---|---|
| `NO_MMAP` | 0 | The BO will never be CPU-mapped by userspace. Asking for its mmap offset fails with `EPERM`, and mmap fails with `EINVAL` | **[ABI]** |
| `WB_MMAP` | 1 | Force a write-back cacheable CPU mapping even when the GPU is not IO-coherent. Userspace then owns cache maintenance (9.4). Available from driver version 1.7 | **[ABI]** |

- **[ABI]** `NO_MMAP` and `WB_MMAP` together are rejected (`EINVAL`). So is any
  other bit.
- **[ABI]** **VM-private BOs**: these are not a flag. A nonzero
  `exclusive_vm_id` at creation makes the BO private to that VM:
  - It can only ever be GPU-mapped into that VM. Mapping it into any other VM
    fails with `EINVAL`.
  - It cannot be exported as a dma-buf (`EINVAL`).
  - **[POLICY]** It shares the VM's reservation (fence) object, so submissions
    on that VM don't have to track it separately. This is cheaper than an
    external BO.
  - The VM ID must name a live VM in the same file, otherwise `EINVAL`.
- **[ABI]** BO_QUERY_INFO (10-uapi) returns the creation flags. For a BO
  imported from another driver it returns 0, plus an "imported" indication.

## 9.3 Coherency model

### 9.3.1 Determining coherency

- **[HW]** Whether the GPU is IO-coherent with the CPU is a platform property.
  The devicetree `dma-coherent` property says so (01-platform). RK3588 does
  **not** mark the GPU coherent.
- **[HW]** If the platform says coherent, the GPU must support the ACE-Lite
  protocol (`COHERENCY_FEATURES` bit 0, 02 section 2.3). Otherwise
  initialization fails. The ACE protocol is never used with CSF GPUs.
- **[ABI]** The chosen mode is reported to userspace as `selected_coherency`
  in the GPU info query: 0 = ACE-Lite, 31 = none (10-uapi, version 1.7+).

### 9.3.2 CPU mapping attributes

| Device | BO flags | Userspace and kernel CPU mapping |
|---|---|---|
| Coherent | any | Write-back cacheable. Hardware keeps CPU and GPU coherent |
| Non-coherent (RK3588) | default | **Write-combine** (normal non-cacheable). No CPU cache maintenance needed |
| Non-coherent | `WB_MMAP` | Write-back cacheable. **Explicit cache maintenance required** (9.4) |
| Any | imported from another driver | Whatever the exporter provides. Treat as cached and never assume coherency (9.6) |

- **[HW]** On a coherent device, BO pages are never mapped uncached. The kernel
  zeroes freshly allocated pages through a cached alias without an explicit
  flush. An uncached alias could then read stale memory, or a dirty line
  evicted later could corrupt the data. A driver for a coherent platform must
  either always map cached or explicitly clean freshly zeroed pages first.
- **[POLICY]** On a non-coherent device, before a BO is CPU-mapped
  write-combined, its pages are first mapped for device DMA. That step cleans
  any dirty cache lines left over from zeroing, so the uncached view is
  consistent.

> OPEN QUESTION: A future refinement (GPU-only BOs mapped non-shareable)
> would need the same clean-after-zero step even on coherent devices. That
> mode does not exist in this baseline.

### 9.3.3 GPU mapping attributes (summary)

GPU-side cacheability is chosen per mapping at VM_BIND time, not per BO. 04-mmu
has the details:

- **[ABI]** By default a mapping is GPU-cacheable (write-back, inner/outer).
  With the `MAP_UNCACHED` bind flag it is GPU non-cacheable.
- **[HW]** On a non-coherent device, cacheable GPU memory must use the GPU's
  internal ("Midgard inner") shareability. Inner- and outer-shareable
  write-back memory is not allowed while coherency is disabled. On a coherent
  device, use CPU-inner shareability.
- **[HW]** Non-coherent example: data the CPU writes through a write-combine
  mapping reaches memory, but the GPU may still hold stale lines from earlier
  reads in its own caches. Userspace command streams are responsible for GPU
  cache flush/invalidate (CS instructions). The kernel also does a
  flush-and-invalidate before each job, subject to flush-ID elimination
  (07-scheduler, 10-uapi `latest_flush`).

## 9.4 CPU cache maintenance (BO_SYNC)

**[ABI]** Driver version 1.7 adds an ioctl that does CPU cache maintenance on
ranges of BOs. It takes an array of operations (stride-versioned, 10-uapi).
Each operation names a BO handle, a type, an offset and a size.

| Type | Value | Effect |
|---|---|---|
| Flush | 0 | Write dirty CPU cache lines in the range back to memory (clean). Use after CPU writes, before the GPU reads |
| Flush and invalidate | 1 | Clean, then invalidate the range, so later CPU reads fetch from memory. Use before the CPU reads data the GPU wrote |

Rules:

1. **[ABI]** If `offset + size` overflows or exceeds the BO size, fail with
   `EINVAL`.
2. **[ABI]** If the BO was imported from another driver, fail with `EINVAL`.
   Userspace must use the dma-buf's own CPU-access synchronization instead,
   because the exporter may need more than cache maintenance (for example
   migrating data from device memory).
3. **[ABI]** An unknown type fails with `EINVAL`.
4. **[ABI]** If the BO's CPU mapping is write-combine (non-coherent device, no
   `WB_MMAP`), succeed without doing anything.
5. **[ABI]** A size of 0 succeeds without doing anything.
6. **[ABI]** A BO with no pages allocated yet succeeds without doing anything.
7. Otherwise, apply the operation to the physical range. **[HW]** The range is
   widened to whole CPU cache lines: the start is rounded down and the end up.
8. **[ABI]** Operations run in array order. The first failure stops processing
   and returns its error. Earlier operations stay done, and nothing reports how
   many succeeded. An unknown handle gives `ENOENT`. An empty array succeeds.

- **[POLICY]** On a coherent device the maintenance steps are no-ops, but
  argument validation still applies.
- **[HW]** Doing "flush and invalidate" as a single clean-then-invalidate per
  range is necessary. A combined bidirectional "for device" sync on arm64 only
  cleans, so the invalidate has to be issued separately.

> OPEN QUESTION: Userspace may also do cache maintenance itself (for example
> arm64 `DC CVAC`/`DC CIVAC` from EL0 when the kernel allows it) on its own
> cached BOs. The uAPI forbids this only for imported BOs. Whether Mesa relies
> on BO_SYNC exclusively or on userspace maintenance for native BOs cannot be
> told from the kernel source.

## 9.5 mmap semantics

### 9.5.1 BO mappings

- **[ABI]** Userspace gets a per-BO fake file offset from BO_MMAP_OFFSET, then
  mmaps the DRM file descriptor at that offset. Offsets are opaque, page
  aligned, unique per BO, and always lower than the user MMIO offset (9.5.2).
- **[ABI]** The mapping must be **shared**. A private writable
  (copy-on-write-capable) mapping is rejected with `EINVAL`.
- **[ABI]** mmap of a `NO_MMAP` BO fails with `EINVAL`.
- **[POLICY]** No pages are inserted at mmap time. They are inserted on fault,
  and the fault handler allocates backing pages if needed. A huge-page-aligned
  region backed by a huge page may be mapped with one 2 MiB entry.
- **[ABI]** Touching a page beyond the BO's size raises `SIGBUS`. A failed
  backing allocation during a fault also raises `SIGBUS`.
- **[POLICY]** The mapping is excluded from core dumps and cannot be grown with
  mremap.
- **[POLICY]** While a BO has at least one userspace mapping it counts as
  "mmapped" for reclaim (9.8). Eviction first removes all CPU mappings, and the
  next access faults the pages back in.
- **[ABI]** mmap of an imported BO is passed on to the exporting driver.

### 9.5.2 User MMIO region and the flush-ID page

- **[ABI]** File offsets at or above the **user MMIO offset** select device
  MMIO pages instead of BOs. The base depends on the process:

| Process type | User MMIO offset |
|---|---|
| 64-bit | `1 << 56` |
| 32-bit (compat), or explicitly chosen | `1 << 43` (fits a 32-bit mmap2 page offset) |

- **[ABI]** The default is picked when the file is opened, from the opening
  task's word size. A process can override it per file with
  SET_USER_MMIO_OFFSET (version 1.5+). This exists for emulators such as FEX
  that run 32-bit guest code inside a 64-bit host process.
- **[ABI]** Only one MMIO page is defined so far. At *user MMIO offset* + 0 is
  the **`LATEST_FLUSH_ID` page**: the 4 KiB register page at GPU offset
  `0x10000` (02 section 2.9).
- **[ABI]** Mapping constraints (else `EINVAL`):
  - the mapping must be shared,
  - its length must be exactly one CPU page,
  - it must not be writable or executable. It also cannot be made writable
    later, because later write permission is refused.
- **[ABI]** Any other offset in the MMIO range fails with `EINVAL`.
- **[POLICY]** The page is not copied into a child process on fork.

Power-management behavior:

- **[HW]** The register page may only be accessed while the GPU is powered.
  Accessing it while suspended can cause an external abort.
- **[POLICY]** The page is inserted at fault time:
  - GPU powered: the real register page, mapped **uncached** (device memory).
  - GPU suspended or suspending: a kernel RAM page holding the value **1**.
- **[POLICY]** On every suspend and every resume, all existing user mappings of
  the MMIO range are torn down, so the next access faults in the right backing
  page. Use a lock so that a fault cannot install a stale mapping while the
  power state is changing.
- **[ABI]** The dummy value is 1, not 0, because in a submission
  `latest_flush = 0` means "always flush". 1 is a valid, old flush ID, so the
  kernel's flush-elimination logic decides correctly. While the GPU is
  suspended, its caches are empty anyway.
- **[ABI]** Mesa maps this page and copies its value into each queue
  submission's `latest_flush` field (10-uapi).

## 9.6 dma-buf export and import

### 9.6.1 Export

- **[ABI]** Any BO without an exclusive VM can be exported as a dma-buf.
  Exporting a VM-private BO fails with `EINVAL`.
- **[POLICY]** The exported buffer shares the BO's reservation object, so
  fences attached by GPU jobs are visible to importers.
- **[POLICY]** The exporter supports attaching and mapping for importing
  devices, CPU mmap, kernel virtual mapping, and begin/end CPU access.
- **[HW]** Begin and end CPU access must do cache maintenance for every view of
  the memory:
  - **Begin CPU access** (device to CPU): make each importer's DMA mapping, the
    exporter's own device mapping and any kernel virtual mapping coherent for
    CPU reads (invalidate).
  - **End CPU access** (CPU to device): clean the kernel virtual mapping and
    make each importer's mapping and the device's own mapping coherent for
    device reads.

### 9.6.2 Import

- **[ABI]** Importing a dma-buf that this same device exported gives back the
  **original BO**, with another reference, not a wrapper. It is not flagged as
  imported, and its creation flags are preserved.
- **[ABI]** Importing a dma-buf from any other exporter makes a wrapper BO:
  - its size is the dma-buf size and its creation flags are 0,
  - it is flagged `IS_IMPORTED` in BO_QUERY_INFO,
  - its pages are whatever the exporter's scatter list gives,
  - it cannot be used with BO_SYNC (`EINVAL`),
  - mmap is forwarded to the exporter,
  - it is never evicted by this driver,
  - it can be GPU-mapped into any VM (it has no exclusive VM).
- **[ABI]** Imported memory must be treated as **cached and non-coherent**,
  even on a coherent GPU. CPU access must be bracketed by the dma-buf's own
  synchronization calls.

### 9.6.3 Implicit synchronization

- **[ABI]** GPU job fences are added to the reservations of the VM and of all
  external BOs mapped in it with **bookkeeping** usage. That usage does not
  take part in implicit synchronization. So other drivers (for example a
  display controller) will **not** wait on panthor jobs implicitly through a
  shared dma-buf.
- **[ABI]** Userspace (Mesa WSI) has to do cross-device sync explicitly: export
  a sync file from a syncobj and import it into the dma-buf (generic dma-buf
  sync-file import/export), or pass fences through the window system.

## 9.7 Kernel-internal buffer objects

The driver makes several BOs for its own use. They never get a userspace
handle, are always `NO_MMAP`, and are always exclusive to the VM they are
mapped into. Mapping flags use the VM_BIND map-flag meanings from 04-mmu.

| Buffer | VM | Size | GPU mapping | Kernel CPU map | Pinned |
|---|---|---|---|---|---|
| Firmware image sections | Firmware (MCU) VM | Per section header (05-firmware) | At the fixed VA the firmware image names. Executable, writable and cacheable as the section header says. Uncached unless the section asks for cached mode | Only for sections shared with the host | Yes |
| Per-queue firmware interface block | Firmware VM | 8 KiB: input page at +0, output page at +4 KiB | No-exec, uncached | Yes, zeroed | Yes |
| Group suspend buffer | Firmware VM | Firmware-specified per-CSG suspend size | No-exec, cached | No | Yes |
| Group protected-mode suspend buffer | Firmware VM | Firmware-specified protected suspend size | No-exec, cached | No | Yes |
| Queue ring buffer | Group's VM | `ringbuf_size` from GROUP_CREATE: power of two, 4–64 KiB | No-exec, uncached | Yes | Yes (kept CPU-mapped) |
| Group sync objects | Group's VM | 16 bytes × queue count | No-exec, uncached | Yes, zeroed | Yes |
| Per-queue profiling slots | Group's VM | 32 bytes × slot count (enough for the most profiled jobs that fit in the ring) | No-exec, uncached | Yes | Yes |
| Heap context pool | Owning VM | 08 section 8.2.3 | No-exec, cached | Yes | Yes |
| Heap chunk | Owning VM | Heap chunk size | No-exec, cached | Only at creation | No (see 08 section 8.6) |
| Sparse backing ("dummy") BO | Any VM of the file | 2 MiB | Mapped repeatedly to back sparse ranges (04-mmu) | No | No |

Group sync object layout (16 bytes per queue). **[HW]** The ring-buffer
commands that the kernel emits use it (07-scheduler):

| Offset | Size | Field |
|---|---|---|
| `0x0` | 8 | Sequence number of the last completed job |
| `0x8` | 4 | Status: nonzero means error |
| `0xC` | 4 | Padding, zero |

Profiling slot layout (32 bytes): cycle counter before (8), cycle counter
after (8), timestamp before (8), timestamp after (8).

- **[POLICY]** Kernel BOs in user VMs get VAs from the VM's kernel range, above
  the userspace range. Alignment is 2 MiB for buffers of 2 MiB or more, 4 KiB
  otherwise. The mapped size is rounded up to the VM page size and no further,
  so fixed-address firmware sections never overlap.
- **[POLICY]** All BOs mapped into the firmware VM are pinned (never evicted).
- **[POLICY]** One sparse backing BO is created per open file and shared by all
  of that file's VMs. It is *not* pre-populated. Its pages may come as one huge
  page or as many 4 KiB pages.
- **[POLICY]** Each kernel BO gets a fixed descriptive label (for example
  "ring buffer", "heap chunk") for debugging. Userspace cannot label them.

## 9.8 Memory reclaim (eviction to swap)

The driver implements a memory-pressure callback. Under pressure it can evict
idle BO contents to swap. There is no "purgeable" or "don't need" uAPI, so
contents are never thrown away, only swapped out.

- **[POLICY]** Reclaim is enabled only when swap space is available.
  Otherwise the driver reports nothing reclaimable.
- **[POLICY]** Each BO with pages is in one of these reclaim classes. The list
  runs from easiest to hardest to reclaim:

| Class | Condition | Needs to block? |
|---|---|---|
| Unused | Has pages, not pinned, not GPU-mapped, no userspace mapping | No |
| CPU-mapped | Has userspace mappings, not GPU-mapped | No |
| GPU-mapped, one VM | Non-evicted GPU mappings in exactly one VM | Yes |
| GPU-mapped, several VMs | Non-evicted GPU mappings in more than one VM | Yes |
| Unreclaimable | No pages, pinned (kernel CPU map, firmware VM, in-progress map op), or imported | — |

- **[POLICY]** Scan order: unused, then CPU-mapped. Then, only if the caller is
  allowed to block, single-VM BOs (visiting VMs in least-recently-active
  order), and finally multi-VM BOs. All locks are taken with try-lock, and a
  contended BO is skipped. A scan that frees nothing while candidates remain
  returns "try again" up to twice before giving up.
- Evicting one BO:
  1. Remove all userspace CPU mappings of the BO.
  2. **[POLICY]** For GPU-mapped classes, wait at most 10 kernel timer ticks
     (10–40 ms depending on the tick rate) for all fences on the BO's
     reservation, bookkeeping fences included. If they don't signal, skip the
     BO.
  3. **[HW]** For each VM mapping the BO, tear down the GPU mappings safely:
     lock the VA region, unmap the pages, flush, unlock (04-mmu). Mark each
     mapping "evicted".
  4. Drop the device DMA mapping and give the pages back to the
     shared-memory layer, which can then swap them out.
- Restoring:
  - **[POLICY]** CPU access: a fault brings pages back in (9.5.1).
  - **[POLICY]** GPU access: before a GROUP_SUBMIT's jobs get their
    dependencies, every evicted BO in the target VM is reloaded and its evicted
    GPU mappings are re-created. A job therefore never runs against an evicted
    mapping. If restoring fails, the submission fails.
- **[POLICY]** Each BO counts how many times it has been evicted (saturating),
  for debugging.

> OPEN QUESTION: Eviction of a GPU-mapped BO depends on all of its users'
> fences having signalled. Work the firmware runs from a heap after a tiler OOM
> grant is covered by the job fence. So is any BO accessed only through
> addresses that userspace computed. This relies on the reservation tracking
> every job that can touch the VM, which holds for this driver because every
> job adds its fence to the VM reservation.

## 9.9 BO labels

- **[ABI]** BO_SET_LABEL (version 1.4+) attaches a NUL-terminated string of up
  to 4096 bytes, NUL included, to a BO. A longer string fails with `E2BIG`. A
  label pointer of 0 removes the label. An empty string is kept as a distinct,
  empty label, not treated as removal.
- **[ABI]** Setting a label replaces any earlier one. Labels have no effect on
  behavior. They appear only in debugging output, as part of a per-device BO
  listing with creator process, size, residency, eviction count and state.
- **[ABI]** Unknown handle: `ENOENT`. Nonzero padding: `EINVAL`. Bad pointer:
  `EFAULT`.

## 9.10 Memory accounting visible to userspace

- **[ABI]** Per-file memory statistics (fdinfo, 10-uapi) report:
  - standard DRM totals for handle-visible BOs: total, shared, resident,
    active, purgeable. Purgeable is always 0. A BO counts as resident if it has
    pages or is imported.
  - driver-internal **resident** memory: the size of every group-owned kernel
    BO (both suspend buffers, sync objects, ring buffers, firmware interface
    blocks, profiling slots) plus every heap pool (08 section 8.6).
  - driver-internal **active** memory: the part of the above belonging to
    groups currently on a CSG slot, or to VMs currently holding an address
    space.
