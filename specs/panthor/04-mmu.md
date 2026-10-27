# 04: GPU MMU and Virtual Memory

This section covers the GPU memory-management unit: address-space (AS) slots,
the commands that program them, the page-table format, the GPU virtual address
(VA) layout, how mapping requests from userspace become page-table updates,
fault handling, and the order of teardown.

Register names and offsets are in 02-register-map (2.7 for the MMU block, 2.3.3
for `GPU_CMD`, 2.8 for exception codes). They are not repeated here.

Terminology:

- **VM**: one GPU virtual address space, meaning one set of page tables plus
  the list of mappings in it. A process can own several VMs. The driver also
  owns one VM for the firmware microcontroller (the **MCU VM**).
- **AS slot**: one hardware register set (`AS_*` at `0x2400 + n × 0x40`) that
  points the GPU at one VM's page tables. There are fewer slots than VMs.
- **Lock region**: a naturally aligned, power-of-two VA range that the MMU
  holds locked while its page tables are being changed (4.4).

No difference between architecture majors 10 to 14 was found in any part of
this section (4.12).

## 4.1 Hardware discovery

| Source | Use |
|---|---|
| `MMU_FEATURES[7:0]` | VA bits. Sets the VM size, the page-table depth, and `AS_TRANSCFG.INA_BITS` |
| `MMU_FEATURES[15:8]` | PA bits. Sets the page-table output size and the DMA addressing mask of the device |
| `AS_PRESENT` | Bitmask of the slots that exist |
| Platform DMA-coherence attribute (for example the devicetree `dma-coherent` property) | Selects the coherent or non-coherent configuration in 4.3, 4.6 and 4.7 |

- **[HW]** At least two slots are needed: slot 0 for the MCU VM and at least
  one more for user VMs. Linux refuses to probe if `AS_PRESENT` has no bit set
  above bit 0.
- **[POLICY]** Linux only declares the device coherent when the platform says
  so **and** `COHERENCY_FEATURES` advertises ACE-Lite. In that case it selects
  ACE-Lite in `COHERENCY_PROTOCOL` (03-gpu-control). The full ACE protocol is
  never used on CSF GPUs. If the platform says the device is coherent but the
  GPU does not support ACE-Lite, probe fails.
- **[POLICY]** On a host whose CPU word is 32 bits, Linux clamps the VA-bits
  value to 32 because its page-table code passes VAs as native words. It
  rewrites the cached `MMU_FEATURES` value, so **[ABI]** userspace also sees
  the clamped VA bits in the GPU-info query (10-uapi). A 64-bit implementation
  need not clamp.

## 4.2 Address-space slot management

### 4.2.1 Ownership rules

- **[HW]** At any moment a slot is programmed with at most one VM. The GPU
  only translates through slots. A VM with no slot cannot be used by the GPU
  or the MCU.
- **[HW]** The firmware selects the page tables for a command-stream group
  (CSG) by AS number. The host writes the VM's slot number into the CSG
  configuration word when it starts the group on a CSG slot (06-csf-interface).
  So the VM must hold a slot before the group starts and must keep it while the
  group stays resident.
- **[HW]/[POLICY]** The MCU VM always uses **slot 0**. User VMs never get slot 0.
  The firmware is believed to expect its own address space in AS 0.
  > OPEN QUESTION: The source pins the MCU VM to slot 0 without saying whether
  > the firmware requires that or it is only a convention. Treat it as required.
- **[POLICY]** The MCU VM is made active once when firmware is loaded and again
  after every GPU reset. It is never marked idle, so it is never evicted. It
  loses its slot only on suspend, unplug or reset.

### 4.2.2 Activity tracking and LRU eviction **[POLICY]**

Each VM has an "active user" count. In practice there is one user for each CSG
slot that holds a group using that VM, plus the MCU.

Activation (0 → 1 users):

1. If the VM still holds a slot from an earlier activation:
   - If that slot is marked faulted (4.9), re-program it (step 4).
   - Otherwise mark the VM active and take it off the LRU list. No register
     access is needed.
2. Otherwise pick a slot. For the MCU VM this is slot 0. For a user VM it is the
   lowest-numbered slot ≥ 1 that is not allocated.
3. If that slot number is not in `AS_PRESENT` (meaning all slots are taken),
   take the VM at the head of the LRU list (the least recently idled VM that
   still holds a slot), run the disable sequence (4.3.2) on its slot in
   "recycle" form, and take the slot from it. If the LRU list is empty, fail
   with a busy error. The scheduler should prevent this case (see below).
4. Program the slot with the enable sequence (4.3.1). Mark the VM active.

Deactivation (1 → 0 users): put the VM at the tail of the LRU list and keep
its slot programmed. A VM keeps its slot until another VM needs one, so idle
and resume cycles do not cost any MMU work.

- **[POLICY]** The free-slot search assumes the `AS_PRESENT` bits are
  contiguous from bit 0.
  > OPEN QUESTION: Whether any GPU reports a non-contiguous `AS_PRESENT`. A
  > robust driver should search `AS_PRESENT & ~allocated & ~1` directly.
- **[POLICY]** The scheduler limits the number of distinct VMs it runs at once
  to popcount(`AS_PRESENT` with bit 0 cleared). This keeps the number of
  active VMs no larger than the number of user slots, so the LRU always has an
  idle candidate when one is needed.
- **[POLICY]** Activation is serialized against page-table updates on the same
  VM, so a VM is never activated while it has a lock region outstanding (4.4).

## 4.3 AS enable and disable

### 4.3.1 Enable (bind a VM to slot *n*) **[HW]**

1. Unmask the slot's fault bit (bit *n*) in the MMU `INT_MASK`. If the slot was
   marked faulted, first write bit *n* to `INT_CLEAR` and clear the faulted
   mark.
2. Write `AS_TRANSTAB` = physical address of the VM's root translation table
   (4.6.2).
3. Write `AS_MEMATTR` = the VM's memory-attribute word (4.7).
4. Write `AS_TRANSCFG` (value below).
5. Issue `UPDATE` with the command protocol (4.3.3).

Each 64-bit register is written low word first, then high word (2.1).

`AS_TRANSCFG` value used for every VM:

| Field | Value | Notes |
|---|---|---|
| ADRMODE | 6 (AArch64, 4 KiB granule) | The 64 KiB-granule mode is never used |
| INA_BITS | 55 − VA bits | VA bits from `MMU_FEATURES`, after any clamp |
| OUTA_BITS | 0 | Left at 0. See open question |
| SL_CONCAT | 0 | |
| PTW_MEMATTR | 2 (write-back) | In both coherent and non-coherent configurations |
| PTW_SH | 2 (outer shareable) if coherent, 0 (non-shareable) otherwise | |
| PTW_RA | 1 | |
| DISABLE_HIER_AP, DISABLE_AF_FAULT, WXN, XREADABLE | 0 | |

Worked values for 48 VA bits: non-coherent `0x0000_0000_4200_01C6`, coherent
`0x0000_0000_6200_01C6`.

> OPEN QUESTION: OUTA_BITS is left at 0 even though the page tables are built
> for the PA width from `MMU_FEATURES`. Presumably 0 means "use the hardware
> maximum", but the source does not confirm this.

> OPEN QUESTION: In the non-coherent case the walker is told the tables are
> write-back cacheable but non-shareable, while the CPU writes them through
> its own cache and then cleans them to memory. This seems to depend on the
> L2 invalidate that comes before every UNLOCK (4.4) to drop stale table lines
> from the GPU L2. A clean-room driver should keep that invalidate.

### 4.3.2 Disable (unbind slot *n*) **[HW]**

1. Mask the slot's fault bit in the MMU `INT_MASK`.
2. Issue a `GPU_CMD` cache flush: L2 clean+invalidate, load/store cache
   clean+invalidate, other caches invalidate. That is `GPU_CMD = 0x00023304`.
   Wait for it to finish (4.5).
3. If the VM in the slot has a lock region outstanding, issue `UNLOCK`.
4. **Recycle form** (the slot is about to be re-programmed right away for
   another VM): stop here. The next enable overwrites all three registers.
5. **Full form**: write `AS_TRANSTAB` = 0, `AS_MEMATTR` = 0,
   `AS_TRANSCFG` = 1 (ADRMODE "unmapped", every access faults), then issue
   `UPDATE`.

If any step fails, the sequence stops and returns an error. A command-protocol
timeout has already scheduled a GPU reset (4.3.3).

> OPEN QUESTION: Whether `UPDATE` itself invalidates the slot's TLB, or
> whether some other step does. The source does nothing else to invalidate a
> TLB when it moves a slot from one VM to another. It relies on the cache flush
> plus `UPDATE` with the new `AS_TRANSTAB`.

### 4.3.3 AS command protocol **[HW]**

To issue any `AS_COMMAND` on slot *n*:

1. Poll `AS_STATUS` until bit 0 (AS_ACTIVE) reads 0.
2. Write the command code to `AS_COMMAND`.
3. Poll `AS_STATUS` again until bit 0 reads 0.

- **[POLICY]** Linux polls every 10 µs for up to 100 ms per wait, busy-waiting
  (no sleep). On timeout it logs that AS_ACTIVE is stuck, schedules a full GPU
  reset, and returns a timeout error.
- **[HW]** The per-slot "command complete" interrupts (MMU `INT_*` bits
  [31:16]) exist but are never unmasked. Polling is enough.
- **[POLICY]** All AS register access is serialized by one global lock that
  also protects slot assignment.

## 4.4 Page-table update sequence (LOCK / UNLOCK)

### 4.4.1 Sequence **[HW]**

Every change to the page tables of a VM that currently holds a slot follows
this sequence:

1. Compute the lock region that covers the VA range to be changed (4.4.2).
   Write it to `AS_LOCKADDR` and issue `LOCK` (4.3.3).
2. Change the page-table entries in memory. On a non-coherent system, clean
   each modified entry and each newly allocated table from the CPU caches to
   memory (4.6.4).
3. Issue a `GPU_CMD` cache flush (L2 clean+invalidate, LSC clean+invalidate,
   other invalidate, `0x00023304`) and wait for it to finish (4.5).
4. Issue `UNLOCK`.

If step 3 or step 4 fails, schedule a GPU reset. The software lock-region state
is cleared either way.

- **[HW]** If the VM holds no slot, the page tables are changed without any
  register access or cache flush. The next enable (4.3.1) makes the GPU see
  the new tables.
- **[POLICY]** The page-table changes of one bind operation (4.8) are done
  under a single lock region. If the operation must widen its range partway
  through (for example to split a 2 MiB block, 4.8.4), the driver issues a
  second `LOCK` that covers the **union** of the old and new regions. Only one
  lock region is outstanding per VM at a time. If the new range is already
  inside the current lock region, the driver skips the second `LOCK`.
- **[POLICY]** Linux never uses `FLUSH_PT` (4) or `FLUSH_MEM` (5). TLB and
  walk-cache invalidation of the locked range is assumed to happen on `UNLOCK`.
  Data-cache maintenance always goes through the global `GPU_CMD` cache flush,
  not through AS commands. The driver notes as future work an optional
  physical-range flush that would narrow this.
  > OPEN QUESTION: The exact semantics of `FLUSH_PT` and `FLUSH_MEM` on CSF
  > GPUs, and whether they are deprecated in favour of `GPU_CMD` flushes.
  > Neither command is needed to reproduce Linux behaviour.
- **[HW] (inferred)** GPU accesses that hit the locked region are held off
  until `UNLOCK`, so the GPU never sees a half-updated mapping. The driver
  depends on this when it widens the lock to keep a split-and-remap atomic.
  > OPEN QUESTION: Whether locked accesses stall or fault. The source implies
  > they stall.

### 4.4.2 `AS_LOCKADDR` encoding **[HW]**

The lock region is a naturally aligned block of 2^(*w*+1) bytes. The register
holds:

| Bits | Content |
|---|---|
| [5:0] | *w* = log2(region size) − 1 |
| [63:15] (at least) | Region base address. Low address bits below the region size are ignored by the hardware |

Computation, for a VA range [*start*, *end*) with *end* > *start*:

1. *h* = 1 + the index of the highest bit that differs between *start* and
   *end* − 1. This is the bit width needed to cover both ends with one aligned
   block. If *start* = *end* − 1, *h* = 0.
2. *w* = max(*h*, 15) − 1. The minimum region is 2^15 = 32 KiB, so *w* ≥ 14.
3. Base = *start* rounded down to a multiple of 2^(*w*+1).
4. `AS_LOCKADDR` = base | *w*.

Example: range `0x1_0000_3000` to `0x1_0000_5000` (8 KiB). The highest
differing bit between `0x1_0000_3000` and `0x1_0000_4FFF` is bit 14, so *h* =
15 and *w* = 14. The region is 32 KiB at `0x1_0000_0000`. The value written is
`0x0000_0001_0000_000E`.

> OPEN QUESTION: The width of the size field is assumed to be 6 bits, because
> *w* never exceeds 63 and the base is always at least 32 KiB-aligned. The
> source only places *w* in the low bits.

Note: Linux clears only the address bits below bit *w*, not below bit *w*+1.
This gives the same register value whenever *h* ≥ 15. When the 32 KiB minimum
applies, the base can keep a stray bit 14, which the hardware ignores. A
clean-room implementation should align the base to the full region size as in
step 3, and track the region it actually locked.

## 4.5 Cache flush interaction **[HW]**

- The `GPU_CMD` cache-flush command and its completion bit
  (`CLEAN_CACHES_COMPLETED` in the GPU interrupt block) are shared by the whole
  GPU. **[POLICY]** Linux allows only one flush in flight. A request made while
  another flush is pending fails with an I/O error.
- Completion: wait for `CLEAN_CACHES_COMPLETED`. **[POLICY]** The timeout is
  100 ms. Before declaring a timeout, check `INT_RAWSTAT` in case the event
  arrived but was not yet handled. On a real timeout, schedule a GPU reset.
- The MMU paths always use the full flush (L2 and LSC clean+invalidate, other
  invalidate). This flush is needed:
  - before `UNLOCK` after a page-table change (4.4.1), so that dirty lines for
    memory that was just unmapped are written back while the old translation
    still holds, and stale lines for remapped memory are dropped; and
  - before a slot is unbound or recycled (4.3.2).

## 4.6 Page-table format

### 4.6.1 Format **[HW]**

The GPU walks page tables in the **Arm VMSAv8-64 stage-1 translation-table
descriptor format, 4 KiB granule** (Arm Architecture Reference Manual for
A-profile, chapter D8). ADRMODE 6 in `AS_TRANSCFG` selects this format. It is
the plain AArch64 format, not the older Mali-specific LPAE variant: leaf pages
at the last level use descriptor type `0b11`, and the Access Flag is honoured.

Configuration used by Panthor:

| Parameter | Value |
|---|---|
| Granule | 4 KiB (9 index bits per level, 512 eight-byte entries per table) |
| Input (VA) size | VA bits from `MMU_FEATURES` (after clamp) |
| Output (PA) size | PA bits from `MMU_FEATURES`, capped at 48 |
| Leaf sizes used | 4 KiB pages (last level) and 2 MiB blocks (level 2). **[POLICY]** 1 GiB level-1 blocks are never created |
| Translation regime | One range, starting at VA 0 (TTBR0-style). No ASID/TTBR1 split |
| Security state | Non-secure bits (NS, NSTable) left at 0 |
| Hardware dirty-bit management | Not used |
| Contiguous hint | Not used |

- Number of levels = ceil((VA bits − 12) / 9), ending at level 3. The root
  table has 2^(VA bits − 12 − 9 × (levels − 1)) entries. With 48 VA bits there
  are 4 levels starting at level 0, and the root holds 512 entries (4 KiB).
- **[POLICY]** Linux's table code accepts only these PA widths: 32, 36, 40,
  42, 44 or 48 (larger values are capped at 48). Any other `MMU_FEATURES`
  PA-bits value makes VM creation fail.
- The output-address field is bits [47:12] for both table and leaf
  descriptors.

### 4.6.2 Descriptor contents **[HW] format / [POLICY] choices**

Table descriptor (levels 0 to 2): bits [1:0] = `0b11`, [47:12] = PA of the
next-level table. Every other bit is 0: no APTable, XNTable or NSTable
hierarchical controls. **[POLICY]** Bits [58:55] are ignored by the walker,
and Linux uses one of them as a private software marker in table descriptors.

Leaf descriptor (4 KiB page at level 3 with [1:0] = `0b11`, or 2 MiB block at
level 2 with [1:0] = `0b01`):

| Bits | Field | Value |
|---|---|---|
| [4:2] | AttrIndx | 1 for cacheable mappings, 0 for uncached mappings (4.7) |
| 5 | NS | 0 |
| 6 | AP[1] | 1 always (the "unprivileged" access bit). Every mapping is created as user-accessible |
| 7 | AP[2] | 1 if read-only, otherwise 0 |
| [9:8] | SH | 3 (inner shareable) for cacheable mappings, 2 (outer shareable) for uncached |
| 10 | AF | 1 always, so Access Flag faults never happen |
| 11 | nG | 1 always |
| [47:12] | Output address | Physical (DMA) address of the page or block |
| 53 | PXN | 1 if no-exec |
| 54 | UXN/XN | 1 if no-exec |
| other | | 0 |

Mapping flags to descriptor bits:

| Map flag (10-uapi) | Effect |
|---|---|
| (none) | Read/write, executable, cacheable: AttrIndx 1, SH 3, AP[2] 0, XN 0 |
| READONLY | AP[2] = 1 |
| NOEXEC | PXN = UXN = 1 |
| UNCACHED | AttrIndx 0, SH 2 |

> OPEN QUESTION: Which of AP[1] and AP[2] the GPU actually enforces. Linux
> always sets AP[1] and relies on AP[2] for read-only. A clean-room driver
> should do the same.

### 4.6.3 Table memory **[HW]**

- Each table is one 4 KiB-aligned, physically contiguous, zero-filled page. The
  root table is allocated when the VM is created. Its size depends on VA bits
  (4.6.1), and it is aligned to at least its own size.
- The GPU walks tables by **physical** address. Table memory must lie within
  the PA width the GPU can reach and must not depend on any IOMMU translation.
  **[POLICY]** Linux refuses table pages whose DMA address differs from their
  CPU physical address.
- Leaf descriptors hold the **device (DMA) address** of each data page, in
  other words the address as seen from the GPU's bus.
- **[POLICY]** Bind operations can run in a context that must not block on
  memory allocation (4.8.5). For that reason Linux pre-reserves the worst-case
  number of table pages for each operation before queuing it, and returns the
  unused pages afterwards:
  - Map of [*va*, *va*+*size*): one table for each 512 GiB, 1 GiB and 2 MiB
    aligned window that the range touches. That is one level-1, one level-2 and
    one level-3 table per window.
  - Unmap: up to two level-3 tables, one for each end of the range that is not
    2 MiB-aligned (for splitting a block).
  - All of these are drawn from a cache of 4 KiB pages. The root table is
    allocated directly.
- Tables freed by an unmap go back to the cache.

### 4.6.4 Cache maintenance for table memory **[HW]**

- **Non-coherent system:** after the CPU writes any descriptor, or zero-fills a
  newly allocated table, clean those bytes from the CPU caches to memory (a
  "sync for device" of the entries) before the GPU may walk them, which means
  before `UNLOCK` or the enabling `UPDATE`. Tables must stay mapped for DMA in
  the device-bound direction for as long as they exist.
- **Coherent system:** no CPU cache maintenance is needed. The walker snoops
  CPU caches (`PTW_SH` = outer, 4.3.1).

### 4.6.5 Leaf-size selection **[POLICY]**

To map a physically contiguous run, the driver uses 2 MiB blocks wherever the
VA and the PA are both 2 MiB-aligned and at least 2 MiB is left. Otherwise it
uses 4 KiB pages up to the next 2 MiB VA boundary. Unmaps step in the same way.
The page-table code Linux uses cannot unmap part of a block, which leads to the
split rule in 4.8.4.

## 4.7 MAIR and `AS_MEMATTR`

### 4.7.1 MAIR used by the page tables **[POLICY]**

The PTE AttrIndx selects one of eight attributes. Panthor uses a standard
AArch64 MAIR layout:

| Index | AArch64 MAIR byte | Meaning | Used by Panthor |
|---|---|---|---|
| 0 | `0x44` | Normal, inner and outer non-cacheable | Uncached mappings |
| 1 | `0xFF` | Normal, inner and outer write-back, read- and write-allocate | Cacheable mappings (default) |
| 2 | `0x04` | Device-nGnRE | Never |
| 3 | `0xF4` | Normal, inner non-cacheable, outer write-back RW-allocate | Never |
| 4–7 | `0x00` | – | Never |

### 4.7.2 Translation to `AS_MEMATTR` **[HW] encoding / [POLICY] mapping**

The GPU does not read an AArch64 MAIR. Each MAIR byte is translated into the
Mali 8-bit encoding (02-register-map 2.7.3) and packed into `AS_MEMATTR`, with
byte *i* holding index *i*. For each byte, take *outer* = MAIR[7:4] and
*inner* = MAIR[3:0]:

- **Non-cacheable result**: chosen if *outer* has neither bit 0 nor bit 1 set
  (Device or non-cacheable outer), or *outer* bit 2 is clear, or *inner* bit 2
  is clear. In other words, anything that is not write-back on both levels. The
  result is memory type 1 (non-cacheable), shareability 0, explicit allocation
  with no allocate hints: **`0x4C`**.
- **Write-back result**: memory type 2 (write-back), allocation policy 3
  (explicit), with write-allocate = *inner* bit 0 and read-allocate = *inner*
  bit 1. The shareability field is:
  - **non-coherent**: 0 ("Midgard inner"). With this setting, the PTE value
    SH = inner-shareable means "shareable inside the GPU only". The hardware
    does not allow inner- or outer-shareable write-back memory when IO
    coherency is off.
  - **coherent**: 1 ("CPU inner"). With this setting, SH = inner-shareable
    keeps its normal Arm meaning, coherent with the CPU.

Resulting `AS_MEMATTR` values for the MAIR in 4.7.1:

| Configuration | `AS_MEMATTR` | Index 1 (cacheable) | Index 0 (uncached) |
|---|---|---|---|
| Non-coherent | `0x4C4C4C4C_8C4C8F4C` | `0x8F` | `0x4C` |
| Coherent | `0x4C4C4C4C_9C4C9F4C` | `0x9F` | `0x4C` |

- **[HW]** The same `AS_MEMATTR` value applies to every VM on a given device.
- **[POLICY]** Firmware sections whose cache mode asks for an IO-coherent
  ("shared") memory type are mapped uncached for now. The Mali "shared" memory
  type (type 0) is not used anywhere (05-firmware).

## 4.8 VM lifecycle and VM_BIND semantics

### 4.8.1 VA layout of a user VM

The VM covers [0, 2^VA bits). It is split into a **user region** at the bottom
and a **kernel region** at the top:

```
0                              K                         2^VA_BITS
|--------- user region --------|------ kernel region ------|
  mapped only by VM_BIND          driver-owned objects
                                  (tiler heap chunks, etc.)
```

How the boundary *K* is chosen when the VM is created (user-requested size *U*,
0 meaning "choose for me"; *F* = 2^VA bits):

1. If *F* < 256 MiB, fail. The kernel region must be at least 256 MiB.
2. If *U* ≠ 0, use it.
   Otherwise, if the calling process's CPU address-space size *T* is smaller
   than *F*, use *T*, so that buffers can sit at the same address on the CPU
   and the GPU.
   Otherwise use *F*/2 if *F* > 4 GiB, or *F* − 256 MiB if not.
3. Clamp the result to at most *F* − 256 MiB. A user-supplied *U* above that
   limit is rejected.
4. Kernel region size = largest power of two ≤ (*F* − chosen user size).
   *K* = *F* − kernel region size.
5. **[ABI]** Return *K* to userspace as the actual user VA range. It can be
   larger than requested.

Example: 48 VA bits, 64-bit process with *T* ≥ 2^48 and *U* = 0. The user size
is 2^47, the kernel region is [2^47, 2^48), and *K* = 2^47.

- **[ABI]** User bind requests must lie entirely below *K*. Requests that
  overlap the kernel region, or that overflow when *va* + *size* is computed,
  are rejected.
- **[POLICY]** Driver-owned objects in the kernel region get their VA either
  at a caller-chosen address (reserved explicitly) or by automatic allocation
  within the "auto-VA" range. For user VMs the auto-VA range is the whole kernel
  region. Automatic allocation uses best fit, aligned to 2 MiB for sizes ≥ 2 MiB
  and 4 KiB otherwise.
- **[POLICY]** Driver-owned mappings use the same page-table path as user
  mappings but skip the user-range check.
- **[POLICY]** At most 32 VMs per open file. VM handles start at 1.

### 4.8.2 VA layout of the MCU VM

- **[HW]** The MCU is a 32-bit core and can only address 4 GiB, so the MCU VM
  covers [0, 4 GiB) whatever `MMU_FEATURES` reports.
- The whole 4 GiB is driver-owned. There is no user region.
- Firmware sections are mapped at the fixed VAs given in the firmware image
  (05-firmware). Read-only, no-exec and uncached attributes come from the
  section flags.
- The auto-VA range for the MCU VM is **[0x0400_0000, 0x0800_0000)**, which is
  64 MiB at 64 MiB. **[HW]** The firmware's shared interface section must sit at
  `0x0400_0000`. The driver allocates queue interface pages, CSG suspend
  buffers and other shared objects from the rest of this window.
- **[POLICY]** A driver-owned object's size is rounded up to the GPU page size
  (4 KiB), **not** the CPU page size. This matters on hosts with 16 KiB or
  64 KiB CPU pages: rounding to the CPU page size could make adjacent
  fixed-address firmware sections overlap.

### 4.8.3 Bind operations **[ABI] semantics**

Struct layouts are in 10-uapi. Every operation names a VA range [*va*, *va* +
*size*). *va*, *size* and the BO offset must all be multiples of the VM page
size, which is **4 KiB** (the smallest leaf size).

| Operation | Behaviour |
|---|---|
| MAP | Map [*bo_offset*, *bo_offset*+*size*) of a buffer object at *va* with the flags READONLY, NOEXEC, UNCACHED or SPARSE (4.6.2). The range must lie inside the BO. Any existing mappings in [*va*, *va*+*size*) are replaced: parts of them outside the range are kept and split as needed (4.8.4). A BO tied to one VM ("VM-exclusive", which shares that VM's reservation domain) can be mapped only into that VM |
| UNMAP | Remove all mappings in [*va*, *va*+*size*). Mappings that only partly overlap are trimmed, and a mapping that spans the whole range is split in two. BO handle, BO offset and flags other than the type must be 0. Unmapping a range with nothing mapped is not an error |
| SYNC_ONLY | No VM change. Exists only to wait on and signal sync objects in queue order. Only valid in asynchronous mode, with *va*, *size*, BO handle and BO offset all 0 and at least one sync object |

Sparse mapping (MAP with SPARSE):

- **[ABI]** BO handle and BO offset must be 0, and NOEXEC is required.
- The range is backed by a per-client 2 MiB dummy buffer that repeats every
  2 MiB of VA. VA *x* maps dummy offset (*x* mod 2 MiB).
  - **[ABI]** Reads return undefined data.
  - Writes are not discarded. They land in the dummy buffer. **[POLICY]** For
    that reason the dummy is per client (per open file), so data cannot leak
    between clients. A VM keeps its own reference to the dummy because it can
    outlive the file.
- **[POLICY]** The mapping is built in 2 MiB-VA-aligned chunks, so a chunk can
  use one 2 MiB block when the dummy is physically contiguous.

### 4.8.4 Splitting and remapping **[POLICY]**

When a MAP or UNMAP cuts an existing mapping, the remaining parts ("previous"
before the cut and "next" after it) keep their original flags and BO offsets.
The page tables must then show exactly the new layout.

The page-table code Linux uses cannot remove part of a 2 MiB block. So for
each cut mapping:

1. If the cut start is not 2 MiB-aligned, the previous part covers the
   enclosing 2 MiB window, and either that window is backed by a ≥ 2 MiB
   physically contiguous chunk or the mapping is sparse, then move the unmap
   start down to the 2 MiB boundary. Do the same at the end of the range,
   moving the end up to the next 2 MiB boundary.
2. Widen the lock region to cover the widened range (4.4.1).
3. Unmap the widened range.
4. Re-map the pieces of the previous and next parts that fall inside the
   widened range. Sparse pieces keep the "VA mod 2 MiB" offset rule.

A clean-room page-table implementation that can split a block in place (by
replacing a block descriptor with a table of 512 page descriptors) can skip the
widening. The final layout must still appear atomically, inside one lock
region.

Mappings of BOs whose pages have been reclaimed (4.8.6) have no page-table
entries. Splitting them changes only the bookkeeping.

### 4.8.5 Synchronous and asynchronous binds **[ABI]**

- **Synchronous** (no ASYNC flag): the operations in the array run in order
  before the call returns. Sync objects are not allowed. A zero-size operation
  is a no-op. On the first failure, processing stops and the returned op count
  is set to the index of the failed op, telling userspace how many succeeded.
  **[POLICY]** A failed synchronous op does not normally make the VM unusable.
  It does if the page tables could not be restored to a consistent state
  (allocation failure in the middle of a map, or a mismatch on unmap).
- **Asynchronous** (ASYNC flag): each operation becomes one job on the VM's
  single **bind queue**. Each job has its own wait and signal sync objects.
  - Jobs run strictly one at a time, in submission order. **[POLICY]** There is
    one bind queue per VM, with a depth of one job in flight and no timeout.
    The operations are CPU-only, so they never hang.
  - All validation and all memory reservation (BO pinning, page-table pages,
    mapping-tracking objects) happen at submit time, so a queued job cannot
    fail for lack of memory.
  - Bind completion is **not** implicitly ordered against GPU work. Userspace
    must use the sync objects. **[POLICY]** The completion fence is recorded
    on the VM's reservation domain only as bookkeeping, for memory-reclaim
    purposes.
  - **If an asynchronous job fails, the VM is marked unusable** (4.9.3),
    because userspace can no longer tell what state the VM is in.
  - Submitting to a destroyed or unusable VM fails right away.
- Synchronous operations and asynchronous jobs on the same VM are serialized
  with each other by a per-VM lock.

### 4.8.6 Memory reclaim interaction **[POLICY]**

Pages of BOs mapped into only one VM can be reclaimed under memory pressure.
Reclaim evicts every mapping of the BO: for each one, lock, unmap the pages,
flush, unlock (4.4), and mark the mapping as evicted while keeping it in the
bookkeeping. Before any GPU job is submitted on a VM, the driver brings back
the pages of every evicted BO mapped in that VM and re-creates their
page-table entries, using the same lock/flush/unlock sequence. A partly
evicted BO is fully restored at that point. VMs are kept on a reclaim LRU
that is bumped each time they become active.

## 4.9 Faults and the MMU interrupt

### 4.9.1 Interrupt handling **[HW]**

- MMU interrupt bits [15:0] are per-slot page or bus faults (bit *n* = slot
  *n*). Bits [31:16] (command complete) stay masked.
- **[POLICY]** At init, Linux enables all 16 fault bits. A slot's bit is then
  unmasked on enable and masked on disable (4.3).
- **[POLICY]** Linux uses the shared handling scheme for every interrupt block
  (02-register-map 2.1.1). The top half checks `INT_STAT`, writes 0 to
  `INT_MASK` and defers the work. The deferred handler loops on
  `INT_RAWSTAT & enabled-mask` until nothing is left, then restores `INT_MASK`.

For each pending fault bit *n*:

1. Read `AS_FAULTSTATUS` and `AS_FAULTADDRESS` of slot *n*.
2. Decode `AS_FAULTSTATUS`: [7:0] exception type (02-register-map 2.8), [9:8]
   access type (0 atomic, 1 execute, 2 read, 3 write), bit 10 fault origin
   (1 = "decoder" fault, 0 = "slave" fault, used only for logging), [31:16]
   source ID. `AS_FAULTEXTRA` is not read.
3. Mark slot *n* faulted.
4. Write bit *n* to MMU `INT_CLEAR`.
5. If a VM is bound to slot *n*, set that VM's "unhandled fault" flag.
6. Run the **full** disable sequence on slot *n* (4.3.2): mask its fault bit,
   flush caches, unlock if needed, then program ADRMODE "unmapped" and issue
   `UPDATE`. From then on, every access by work on that AS faults, which
   stops it.
7. The VM **keeps** its slot assignment. Only the faulted mark and the
   disabled state are new.

After all bits are handled, if any fault was seen, ask the scheduler for an
immediate scheduling pass (4.9.2).

- **[POLICY]** Faults are never resolved by mapping memory on demand. There is
  no grow-on-fault and no fault replay. Every MMU fault is fatal to the work
  that caused it.

### 4.9.2 Propagation to the scheduler **[POLICY]**

- On the forced scheduling pass, for every resident group whose VM has the
  unhandled-fault flag, the scheduler first processes that CSG's pending
  firmware events. If the firmware has not reported a specific fatal fault on
  any queue of the group, the scheduler marks **every** queue of the group as
  faulted. Faulted groups are then evicted and their jobs fail (07-scheduler).
- The slot stays disabled while the VM still has active users. A VM that is
  already active does not re-run the enable sequence. When the last user goes
  idle and the VM is later activated again, activation sees the faulted mark.
  It then clears the slot's interrupt, clears the VM's unhandled-fault flag,
  and re-programs the slot (4.2.2 step 1).

### 4.9.3 Unusable VMs **[POLICY] / [ABI]**

A VM is marked **unusable** when an asynchronous bind fails, or when the page
tables could not be kept consistent. Marking it unusable:

1. If the VM holds a slot, runs the full disable sequence on it. The slot stays
   assigned, and the VM stays bound in the unmapped state.
2. Makes every later MAP, and every new asynchronous bind, fail. UNMAP is still
   carried out.
3. **[ABI]** Is reported by the VM-state query as "unusable". The expected
   response from userspace is to destroy the VM and create a new one, much like
   a lost device.

GPU page faults do **not** make the VM unusable (4.9.1).

## 4.10 VM destruction ordering **[POLICY], with [HW] constraints**

When userspace destroys a VM, or the file is closed:

1. Mark the VM destroyed. New bind jobs and new heap pools are refused.
2. **[HW]** If the VM still has active users, wait until any pending
   scheduling pass has finished. When the firmware suspends a CSG it can write
   group state (for example tiler heap contexts) into this VM. Every group
   using the VM must be fully evicted before its mappings go away, or the GPU
   faults. Userspace normally destroys groups before VMs, so the evictions are
   already under way.
3. Destroy the VM's tiler heap pool. This frees driver-owned mappings in the
   kernel region.
4. Unmap the whole VA range [0, 2^VA bits), with lock/flush/unlock if a slot is
   held. Jobs still running on this VM now fault.
5. Drop the handle's reference. The VM object can outlive the handle, because
   jobs, BOs and heap objects can hold references to it.

When the last reference goes away:

6. Remove the VM from reclaim tracking.
7. Shut down its bind queue. If a GPU reset is in progress, restart the queue
   first so that it can drain.
8. Under the VM lock and the slot lock: if the VM holds a slot, run the full
   disable sequence (4.3.2), free the slot and remove the VM from the slot LRU.
9. Free the page tables (leaf-level tables first, root last). Release the
   reference to the sparse dummy buffer.

**[HW]** Page-table memory must not be freed or reused while any slot may
still point at it. That is why step 8 comes before step 9.

## 4.11 Power management and reset

### 4.11.1 Suspend and resume **[POLICY]**

- Suspend: for every slot that holds a VM, run the full disable sequence and
  release the slot (so VMs lose their slot and leave the LRU). Then mask the
  MMU interrupt and wait for any running handler to finish.
- Resume: clear all allocation and faulted marks, then clear and unmask the
  enabled MMU interrupt bits. Slots are not re-programmed eagerly. Each VM is
  re-bound on its next activation, starting with the MCU VM when the firmware
  restarts.

### 4.11.2 GPU reset **[POLICY]**

- Before reset: mask the MMU interrupt, stop every VM's bind queue, and
  remember that a reset is in progress, so that VMs created during the reset
  start with their queue stopped. Slots are **not** cleanly disabled, because
  the reset may be recovering from a stuck AS_ACTIVE bit.
- **[HW]** After reset, all AS slots are assumed to be unprogrammed.
- After reset: clear all allocation and faulted marks, detach every VM from its
  slot, unmask the MMU interrupt and restart the bind queues. The firmware
  restart re-activates the MCU VM in slot 0, and user VMs get slots again when
  their groups are scheduled.

### 4.11.3 Unplug **[POLICY]**

Mask the MMU interrupt (if the device is powered), then run the full disable
sequence on every slot that holds a VM and release it. No MMU register is
touched after this.

## 4.12 Architecture and SoC differences

- **Arch 10 to 14:** no differences in AS register layout, slot management,
  `AS_TRANSCFG`/`AS_MEMATTR` values, the LOCK/UNLOCK sequence or fault decoding.
  The cache flush always uses `GPU_CMD` on every architecture. The arch-14
  power-controller changes (03-gpu-control) do not affect the MMU.
- **Coherent vs non-coherent integration** is the only per-platform variation.
  It affects `AS_TRANSCFG.PTW_SH` (4.3.1), the `AS_MEMATTR` shareability field
  (4.7.2), CPU cache maintenance of tables (4.6.4) and the selected coherency
  protocol (4.1).
- **RK3588 (Mali-G610, arch 10):** non-coherent. No SoC-specific MMU handling
  exists in the driver.
