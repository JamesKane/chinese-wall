# 02: Internal MMU, Memory Map and Coherency

## 2.1 Overview

- **[HW]** Each session has its own two-level page table, which translates
  the firmware's **32-bit virtual address (VA)** space to **40-bit physical
  addresses (PA)**. The table is attached to the LSID with `MMU_CTRL`.
  Every core that runs the session uses the same table.
- **[HW]** Pages are **4 KiB**. There are no large pages or blocks.
- **[HW]** Translation caches are per LSID. After changing a mapping that
  may already be cached, write the LSID's `FLUSH_ALL` register.

## 2.2 Page-table format

### 2.2.1 Address split

| VA bits | Use |
|---|---|
| `[31:22]` | Level-1 index (1024 entries) |
| `[21:12]` | Level-2 index (1024 entries) |
| `[11:0]` | Byte offset within the 4 KiB page |

- **[HW]** Each table is exactly one 4 KiB page holding 1024 32-bit
  entries. It must be 4 KiB aligned.

### 2.2.2 Entry format (both levels)

| Bits | Field | Meaning |
|---|---|---|
| `[1:0]` | AP | Access permission (§2.2.3) |
| `[29:2]` | PA[39:12] | Physical page frame number of the next-level table (level 1) or of the data page (level 2) |
| `[31:30]` | ATTR | Memory attribute (§2.2.3) |

- **[HW]** An entry of all zeroes is invalid (unmapped).
- **[HW]** Encoding: `entry = (ATTR << 30) | ((PA >> 12) << 2) | AP`.

### 2.2.3 Field encodings

| ATTR | Name | Used for |
|---|---|---|
| 0 | private | Firmware text and BSS, page-table links, the level-1 descriptor |
| 1 | shared, coherent | The seven host↔firmware communication pages |
| 2 | shared, read-only | (unused) |
| 3 | shared, read-write | User buffers, firmware RPC allocations, the firmware print buffer |

| AP | Name |
|---|---|
| 0 | no access |
| 1 | read-only |
| 2 | executable |
| 3 | read-write |

- **[POLICY]** Level-1 entries that point at level-2 tables use ATTR 0 and
  AP 1.
- **[POLICY]** VA ranges that are reserved but not yet backed use level-2
  entries with ATTR 0, AP 0 and PA `0x1000`. This is a dummy no-access
  page. It lets a later resize keep the same VA. Code that counts mapped
  pages treats PA `0` and PA `0x1000` as "not mapped".

> OPEN QUESTION: How ATTR 1 ("shared coherent") differs from ATTR 3 at the
> bus level, for example which AXI cache/shareability attributes each
> produces, is not documented.

### 2.2.4 Attaching the table: `MMU_CTRL`

- **[HW]** `MMU_CTRL` holds an entry in the same format that points at the
  level-1 table. The vendor driver writes
  `(0 << 30) | ((L1_PA >> 12) << 2) | 3` (ATTR private, AP read-write).

## 2.3 Firmware virtual memory map

Two firmware protocol generations use different maps (03 §3.1.2). The
major protocol version in the firmware header picks the map. The Sky1
firmware is expected to be protocol 3 (see 03).

### 2.3.1 Protocol 3

| VA range | Contents |
|---|---|
| `0x0000_0000`–`0x000F_FFFF` | Firmware instance 0 (core 0) |
| `0x0100_0000 × n` … `+0x000F_FFFF` | Firmware instance *n* (core *n*, *n* = 1…7) |
| `0x1007_9000` | Message queue, host → firmware (host-written page) |
| `0x1007_A000` | Message queue, firmware → host (firmware-written page) |
| `0x1007_B000` | Input-buffer queue, host → firmware |
| `0x1007_C000` | Input-buffer return queue, firmware → host |
| `0x1007_D000` | Output-buffer queue, host → firmware |
| `0x1007_E000` | Output-buffer return queue, firmware → host |
| `0x1007_F000` | RPC area |
| `0x1010_0000`–`0x1017_FFFF` | Firmware print buffer (512 KiB) |
| `0x2000_0000`–`0x6FFF_FFFF` | "Protected" region: bitstream buffers and firmware RPC allocations of region 0 |
| `0x7000_0000`–`0xEFFF_FFFF` | Frame-buffer region: frame buffers and RPC allocations of region 1 |
| `0xF000_0000`–`0xFFFF_FFFF` | Reserved for engine-internal use. Do not map |

### 2.3.2 Protocol 2 (differences)

| VA range | Contents |
|---|---|
| `0x8000_0000 + 0x1000_0000 × (n−1)` … `+0x000F_FFFF` | Firmware instance *n* (*n* = 1…7). Instance 0 is still at `0x0` |
| `0x2000_0000`–`0x4FFF_FFFF` | Protected / bitstream region |
| `0x5000_0000`–`0x7FFF_FFFF` | Frame-buffer region |

The communication pages and the print buffer are at the same addresses as
in protocol 3.

### 2.3.3 Firmware instance contents

Within each instance region (base *B*):

| VA | Contents |
|---|---|
| *B* + `0x1000` … | Text: the first `text_length` bytes of the `.fwb` file, header included. Mapped executable (AP 2), private. Shared physical pages across all instances |
| *B* + `bss_start_address` + 4 KiB × *i* | BSS page *i*, for each *i* < `bss_bitmap_size` with bitmap bit *i* set: a private zeroed page **per instance**, read-write. If the page's VA falls inside the "master RW" window instead, see the next row |
| *B* + VA in [`master_rw_start_address`, + `master_rw_size`) | Shared BSS: zeroed pages read-write, the **same** physical pages in every instance |

- **[HW]** The header fields are defined in 03 §3.1.1. Bitmap bit *i* is bit
  (*i* mod 32) of 32-bit word *i*/32.
- **[HW]** Map every instance for every core that may run the session.
  **Instance 0 must always be mapped**, even if core 0 is disallowed or
  harvested, because every core reads the start of instance 0 when it
  starts.
- **[POLICY]** The vendor driver maps instances 0…*h*, where *h* is the
  highest usable core. It copies the firmware text once, through the
  lowest mapped instance. It frees the mappings by unmapping 4 MiB per
  instance, which covers more than the 1 MiB region.

> OPEN QUESTION: The text is placed at *B* + `0x1000`, but cores are said
> to read instance-0 addresses `0x0`–`0x18` at start. Whether the first
> page of an instance is covered by the BSS bitmap (`bss_start_address` =
> 0) or by a hardware boot vector is not shown. The header field `rasc_jmp`
> (the start address) is never used by the host.

## 2.4 Communication pages

- **[HW]** Each of the seven communication pages (§2.3.1) is one zeroed
  4 KiB page. It is mapped with ATTR 1 (shared coherent), AP 3, and the
  host keeps it mapped for the whole session.
- **[HW]** The firmware print buffer is 512 KiB of zeroed pages (they need
  not be contiguous), mapped with ATTR 3, AP 3.

## 2.5 User buffers and firmware allocations

- **[HW]** Bitstream buffers must be mapped in the protected/bitstream
  region. Frame buffers (raw or AFBC) must be mapped in the frame-buffer
  region. The firmware receives only the VA, in buffer messages (03 §3.5).
- **[HW]** User buffers are mapped with ATTR 3, AP 3, one page at a time.
  Physically discontiguous pages are fine. Each plane of a multi-planar
  buffer may be mapped separately. Plane offsets inside a page are carried
  in the VA.
- **[POLICY]** VA allocation is next-fit from a per-region cursor, aligned
  to 4 KiB (or to the larger alignment the firmware asked for), and wraps to
  the start of the region when the end is reached. After the decoder
  reports new sequence parameters, the frame-buffer cursor is reset to the
  start of its region, because all output buffers are then re-queued.
- **[HW]** Firmware memory requests (RPC, 03 §3.6) name region 0
  (protected/bitstream) or 1 (frame-buffer). They give a current size and a
  maximum size. The host reserves VA for the maximum size and backs only the
  current size. A resize grows or shrinks the backing at the same VA. After
  a resize or free, write `FLUSH_ALL`.

## 2.6 Cache maintenance and ordering

- **[HW]** The engine is not coherent with CPU caches. For every structure
  the CPU writes and the engine reads (page tables, queue pages, the RPC
  area, firmware text), clean the CPU cache to the point of coherency
  before ringing the doorbell. For every structure the engine writes,
  invalidate before reading.
- **[HW]** Page tables: clean each changed entry (level 1 and level 2).
  Clean a newly allocated zeroed table page as a whole.
- **[HW]** Queues: write the payload, then a barrier, then clean the
  payload. Then update the write index, then a barrier, then clean the
  index. Readers invalidate the peer page, then read the index, then read
  the payload.
- **[POLICY]** The vendor driver maps user buffers without explicit cache
  maintenance in the decode-output path. It relies on the buffer framework.

## 2.7 Interaction with power and memory repair

- **[HW]** All page tables and session memory are in DRAM, so they survive
  engine power-off. LSID registers, including `MMU_CTRL`, do not. A session
  re-attached after power-up must have its LSID set up again (01 §1.4.1).
- **[HW]** Core SRAMs need memory repair after each core power-on. On Sky1
  the firmware power-domain-on operation does this (00 §0.7).
