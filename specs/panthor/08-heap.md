# 08: Tiler Heap Management

The tiler (the vertex/binning stage of a render pass) writes its output into a
growable pool of memory called the **tiler heap**. Userspace asks the kernel to
create a heap, then puts the heap's GPU address into its command streams. The
firmware draws memory from the heap. When the heap runs dry, the firmware asks
the host for another chunk through the **tiler out-of-memory (OOM)** event on
the command-stream (CS) interface.

This section covers the memory layouts the GPU and firmware read, the policy
for growing a heap, and how the OOM event is serviced. The CS interface
mechanics (request and acknowledge bits, doorbells) are in 06-csf-interface.
Group scheduling is in 07-scheduler. The ioctls are in 10-uapi.

## 8.1 Objects and terminology

| Term | Meaning |
|---|---|
| Heap context | A small block of GPU-visible memory that the firmware and GPU use to track a heap's state. One per heap |
| Heap context pool | One buffer per VM that holds up to 128 heap contexts back to back |
| Heap chunk | A contiguous range of GPU virtual memory, all chunks in one heap being the same size, that the tiler fills. Chunks form a singly linked list through a header at the start of each chunk |
| Heap handle | A userspace-visible 32-bit identifier for a heap (10-uapi) |

- **[POLICY]** Heap pools are per VM. A heap's context and chunks are mapped
  only into that VM. Nothing is shared between processes.
- **[POLICY]** All heap memory (pool buffer and chunks) is mapped in the VM's
  kernel-reserved VA range, above the userspace range (04-mmu). It is never
  CPU-mappable by userspace and never gets a userspace buffer handle.

## 8.2 Heap context

### 8.2.1 Size and alignment

- **[HW]** A heap context is **32 bytes**.
- **[HW]** Each heap context must start on a GPU cache-line boundary, because
  both CPU and GPU access it and it must not share a cache line with another
  context. The GPU L2 cache line size is `1 << (L2_FEATURES[7:0])` bytes (02,
  section 2.3).
- **[POLICY]** Contexts are packed at a stride of 32 bytes rounded up to the L2
  line size. With a 64-byte line, which is typical, the stride is 64 bytes.

### 8.2.2 Contents

- **[HW]** The host only *initializes* the heap context, by writing zeros over
  the whole stride-sized slot before giving its address to userspace. After
  that the firmware and GPU own the contents. The host never reads or writes it
  again.
- **[HW]** The firmware may write heap-context state back to memory while a
  command-stream group is being suspended. So the heap context memory (and the
  VM that maps it) must remain valid until every group that used it has been
  fully evicted from the firmware (07-scheduler).

> OPEN QUESTION: The source treats the 32-byte context as opaque and defines no
> fields inside it. Its internal layout (for example where the current chunk
> pointer or free-space counters sit) is not documented. A clean
> implementation needs only to zero it, so the layout is not needed for driver
> correctness.

### 8.2.3 Heap context pool buffer

| Property | Value |
|---|---|
| Created | **[POLICY]** Lazily, on the first heap creation in a VM. One pool per VM |
| Capacity | **[POLICY]** 128 heap contexts |
| Size | **[POLICY]** 128 × stride, rounded up to 4 KiB (8 KiB for a 64-byte stride) |
| GPU mapping | Read/write, **not executable**, GPU-cacheable (the uncached map flag is not used) |
| CPU access | Kernel-only. Kept permanently mapped in the kernel so contexts can be zeroed |
| Userspace mmap | Forbidden (no-mmap buffer) |
| Context *i* GPU VA | pool base VA + *i* × stride |
| Lifetime | Destroyed when the VM is destroyed. Destroying the pool destroys every heap in it |

- **[POLICY]** A heap's index inside the pool (0–127) is part of its handle
  (10-uapi). When the pool is full, heap creation fails with `EBUSY`.
- **[POLICY]** Heap creation on a VM that is being destroyed fails with
  `EINVAL`.

## 8.3 Heap chunks

### 8.3.1 Chunk size and count limits

| Parameter | Constraint | Tag |
|---|---|---|
| Chunk size | Multiple of the CPU page size, and between 128 KiB and 8 MiB inclusive. All chunks of a heap have the same size | **[ABI]** |
| Initial chunk count | At least 1, and no more than the maximum chunk count | **[ABI]** |
| Maximum chunk count | Upper bound on chunks the heap may ever hold, initial chunks included | **[ABI]** |
| Target in flight | Number of render passes that may be in flight before the host stops granting new chunks (8.4.3) | **[ABI]** |

Requests that break these rules fail with `EINVAL`.

- **[HW]** The 8 MiB ceiling keeps the chunk size, counted in 4 KiB units,
  inside a 12-bit field: 8 MiB / 4 KiB = 2048, which is ≤ 4095. See 8.3.3.

### 8.3.2 Chunk memory

- **[POLICY]** Each chunk is its own buffer object. It is GPU-mapped
  read/write, **not executable**, GPU-cacheable. It cannot be mmapped by
  userspace and is private to the heap's VM.
- **[POLICY]** Chunk VAs are allocated from the VM's kernel VA range with 2 MiB
  alignment when the chunk is 2 MiB or larger, and 4 KiB alignment otherwise.
- **[HW]** A chunk's GPU VA is at least 4 KiB aligned. The low 12 bits of a
  chunk address are reused to hold its size (8.3.3).

### 8.3.3 Chunk header layout

Each chunk begins with a **64-byte header**. The tiler writes its data after
the header.

| Offset | Size | Field | Meaning |
|---|---|---|---|
| `0x00` | 8 | `NEXT` | Encoded pointer to the next chunk, or 0 for end of list |
| `0x08` | 56 | reserved | Must be zero |

Encoding of a 64-bit chunk reference (used both in `NEXT` and in the
CS heap registers, 8.4):

| Bits | Content |
|---|---|
| [63:12] | Bits [63:12] of the chunk's GPU VA (the VA is 4 KiB aligned) |
| [11:0] | Chunk size in units of 4 KiB (`size >> 12`) |

Example: a 2 MiB chunk at VA `0x7F_FE40_0000` is encoded as
`0x7F_FE40_0000 | 0x200` = `0x7F_FE40_0200`.

- **[HW]** The host zeroes the whole 64-byte header when it creates a chunk.

### 8.3.4 Initial chunk list

At heap creation the host allocates the initial chunks and links them before
returning:

1. Allocate chunk C1. Its `NEXT` = 0.
2. For each further chunk C*k*, write into its header
   `NEXT` = encoded reference to C*k−1* (the chunk allocated just before it).
3. Report the **last-allocated** chunk as the heap's *first chunk*.

The list userspace receives is therefore C*n* → C*n−1* → … → C1 → end.

- **[HW]** Userspace passes the heap context VA and the first-chunk VA to the
  GPU through its own command streams and descriptors. The kernel returns both
  from heap creation (10-uapi). The first-chunk VA is returned as a plain VA,
  without the size bits.
- **[POLICY]** Chunks added later through the OOM path (8.4) are created with
  `NEXT` = 0. The firmware links them in.

## 8.4 Servicing a tiler OOM event

### 8.4.1 Event delivery

- **[HW]** The firmware signals tiler OOM for one command stream by toggling the
  `TILER_OOM` bit (bit 26) of that CS's acknowledge word, so that it differs
  from the request word, and raising the CSG interrupt (06-csf-interface).
- **[HW]** At the same time it publishes, in the CS **output** block:

| CS output offset | Width | Field | Meaning |
|---|---|---|---|
| `0xC0` | 32 | `HEAP_VT_START` | Counter of vertex/tiling passes started on this heap |
| `0xC4` | 32 | `HEAP_VT_END` | Counter of vertex/tiling passes finished |
| `0xCC` | 32 | `HEAP_FRAG_END` | Counter of fragment passes finished |
| `0xD0` | 64 | `HEAP_ADDRESS` | GPU VA of the heap context that ran out of memory |

- **[HW]** The host answers through the CS **input** block:

| CS input offset | Width | Field | Meaning |
|---|---|---|---|
| `0x20` | 64 | `HEAP_START` | Encoded reference (8.3.3) to the new chunk, or 0 |
| `0x28` | 64 | `HEAP_END` | Same value as `HEAP_START` |

### 8.4.2 Host response sequence

1. **[POLICY]** Do the work outside interrupt context, in a worker that is
   allowed to allocate memory. It must not run on any path that a GPU job's
   completion fence depends on, because a blocking allocation there can
   deadlock memory reclaim.
2. **[HW]** Leave `TILER_OOM` unacknowledged in the interrupt handler. Other CS
   events (fault, fatal) can be acknowledged immediately.
3. If the group is no longer resident on a CSG slot, stop. **[HW]** The
   firmware raises the OOM event again when the group is rescheduled.
4. Read `HEAP_ADDRESS`, `HEAP_VT_START`, `HEAP_VT_END` and `HEAP_FRAG_END`.
   Compute:
   - render passes in flight = `VT_START − FRAG_END`
   - fragment passes pending = `VT_END − FRAG_END`
5. **[POLICY]** Check the counters. Unless `FRAG_END ≤ VT_END < VT_START`, treat
   the request as invalid. Also treat it as invalid if `HEAP_ADDRESS` does not
   correspond to a live heap context in this VM's pool. An address resolves to
   a heap through (address − pool base) / stride. The offset must fit in 32 bits
   and the index must be < 128.
6. **[POLICY]** Grant or refuse (8.4.3). To grant, create one new chunk (8.3.2)
   with a zeroed header and `NEXT` = 0, and set the reply to its encoded
   reference. To refuse, set the reply to 0.
7. **[HW]** Write the reply value to **both** `HEAP_START` and `HEAP_END`.
8. **[HW]** Acknowledge: set the `TILER_OOM` bit of the CS request word equal to
   the same bit of the CS acknowledge word. Then toggle this CS's bit in the
   CSG doorbell-request word and ring the CSG doorbell (06-csf-interface).
9. If the group was descheduled between steps 4 and 8, the chunk cannot be
   handed over. Remove that chunk from the heap and free it, matching on its VA
   with the low 12 bits cleared. Do not acknowledge.
10. **[POLICY]** If step 5 found the request invalid, do not acknowledge. Mark
    the queue as fatally faulted and let the scheduler tear the group down
    (07-scheduler). The group then reports a fatal queue fault (10-uapi,
    GROUP_GET_STATE).

> OPEN QUESTION: Step 5 does not allow for counter wrap-around. The 32-bit
> counters would need about 4 billion passes to wrap, and the source ignores
> the case. Whether `VT_END == VT_START` (all tiling done) can legitimately
> occur during an OOM is unclear.

### 8.4.3 Grant policy, and what happens at the limit

- **[POLICY]** The host refuses (replies 0) when either is true:
  - render passes in flight is **greater than** the heap's target-in-flight
    value, or
  - the heap already holds its maximum chunk count.
- **[POLICY]** A failed chunk allocation (system out of memory) is handled like
  reaching the limit: reply 0 and acknowledge. It is not treated as fatal.
- **[HW]** When the reply is 0, the firmware tries other ways to get memory:
  1. wait for in-flight fragment passes to finish, which releases heap memory
     back to the heap, or
  2. call a tiler-OOM exception handler that userspace installed for the
     queue. The handler is expected to do **incremental rendering**: flush
     the partial render with a fragment pass so the tiler memory can be
     reused.

  If neither works, the tiling job fails and the queue is flagged as faulted.
  Its later jobs fail without waiting for the job timeout.
- **[ABI]** The incremental-rendering handler is supplied and implemented by
  userspace (Mesa) through its command streams. The kernel takes no part in it
  beyond replying 0.

> OPEN QUESTION: The exact firmware fallback order and the exception code
> reported when every fallback fails are not visible in the driver source. The
> `OUT_OF_MEMORY` exception code (`0x60`, 02 section 2.8) is the likely
> candidate.

- **[POLICY]** The host does not use the pending-fragment count in its decision.
  It is computed but ignored.
- **[POLICY]** The host never shrinks a heap. Chunks stay allocated until the
  heap is destroyed. The exception is the hand-back in step 9 of 8.4.2.

## 8.5 Heap and pool lifetime

| Event | Effect |
|---|---|
| Heap create | **[POLICY]** Create the VM's pool if needed. Allocate and link the initial chunks. Claim a free context slot and zero it. Return handle, context VA and first-chunk VA |
| Heap destroy | **[POLICY]** Free the heap's chunks and context slot **at once**, without waiting for the GPU. Userspace must make sure no queued or running work still uses the heap. If it does, GPU page faults will kill the groups involved |
| VM destroy | **[POLICY]** Destroy every heap in the VM's pool, then the pool buffer. Later heap creation on that VM fails |
| Group suspend | **[HW]** The firmware may write heap context state during suspend, so VM teardown must wait until all related groups are evicted (07-scheduler) |

## 8.6 Accounting

- **[POLICY]** A pool's reported size is the pool buffer size plus the total
  size of all chunks in all its heaps. It is reported per open file as
  driver-internal resident memory, and as active memory while the VM holds a
  hardware address space (10-uapi, fdinfo).

> OPEN QUESTION: Heap chunks are not kept permanently pinned in this baseline.
> They are CPU-mapped only briefly, to initialize the header. That makes them
> eligible for the swap-based eviction in 09-memory section 9.8 while their VM
> is idle, and they are restored before the VM's next job runs. The usage
> documentation, by contrast, says such internal buffers are pinned for their
> whole lifetime. A clean implementation may simply pin them.
