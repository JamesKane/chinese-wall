# 05: Firmware Image Format, Loading and MCU Boot

Mali CSF GPUs contain a microcontroller (the **MCU**) that runs Arm-supplied
firmware. The firmware implements the Command Stream Frontend: it schedules
command-stream groups onto the hardware on the host's behalf. The host driver
must load the firmware image into memory, map it into an MCU address space,
start the MCU, and then talk to it through a shared-memory interface.

This section covers:

- where the image comes from and its binary format (5.1, 5.2)
- how sections are placed in the MCU address space (5.3)
- how the host finds the shared interface and enumerates its slots (5.4)
- the boot sequence and the initial global configuration (5.5, 5.6)
- halt, suspend/resume, reset and hang detection (5.7, 5.8)
- top-level handling of the JOB interrupt (5.9)
- protected mode (5.10)

The full request/acknowledge protocol and the complete layout of the global,
CSG and CS interface blocks are in 06-csf-interface. Only the fields needed to
boot and to discover the interface are given here.

Register names (`MCU_CONTROL`, `MCU_STATUS`, `AS_*`, JOB `INT_*`, doorbells)
are those of 02-register-map.

All multi-byte fields in the firmware image and in shared memory are
**little-endian**.

> OPEN QUESTION: The source assumes a little-endian host and never byte-swaps.
> The MCU is an Arm M-profile core, which is little-endian here, so
> little-endian is assumed throughout. A big-endian host would need to swap
> every field.

## 5.1 Firmware file name and selection

- **[HW]** There is one firmware image per GPU *architecture* (ARCH_MAJOR and
  ARCH_MINOR of `GPU_ID`, 02 §2.3.1). Product, version and revision fields do
  not affect the choice.
- **[POLICY]** The image is requested from the system firmware store under this
  path:

  `arm/mali/arch<ARCH_MAJOR>.<ARCH_MINOR>/mali_csffw.bin`

  The two numbers are written in unpadded decimal. This matches the layout of
  the `linux-firmware` repository, which installs the files under
  `/lib/firmware/arm/mali/...`. A different OS can pick its own location, but
  the file contents are the same.
- Images known to the Linux driver (declared as firmware dependencies):

| Path | GPUs |
|---|---|
| `arm/mali/arch10.8/mali_csffw.bin` | Arch 10.8: Mali-G610 (**RK3588**), G710, G510, G310 |
| `arm/mali/arch10.10/mali_csffw.bin` | Arch 10.10 |
| `arm/mali/arch10.12/mali_csffw.bin` | Arch 10.12 |
| `arm/mali/arch11.8/mali_csffw.bin` | Arch 11.8 (G615/G715 family) |
| `arm/mali/arch12.8/mali_csffw.bin` | Arch 12.8 (G620/G720 family, expected for **CIX Sky1**, see 11 §11.1) |
| `arm/mali/arch13.8/mali_csffw.bin` | Arch 13.8 (G625/G725 family) |
| `arm/mali/arch14.8/mali_csffw.bin` | Arch 14.8 (Mali-G1 family) |

> OPEN QUESTION: The product-to-arch-minor column is inferred. The source only
> derives the path from `GPU_ID`. The RK3588 Mali-G610 reports `GPU_ID`
> ARCH_MAJOR = 10 and ARCH_MINOR = 8, so it uses `arch10.8`. The other rows
> follow the product table in 02 and are not confirmed.

- **[POLICY]** If the file is missing, device initialization fails. There is no
  fallback to another architecture's image.

## 5.2 Binary image format

The image is a fixed 20-byte file header followed by a sequence of
variable-length *entries*.

### 5.2.1 File header (offset 0, 20 bytes)

| Offset | Size | Field | Rule |
|---|---|---|---|
| `0x00` | 4 | Magic | Must equal `0xC3F13A6E` (bytes `6E 3A F1 C3` on disk) |
| `0x04` | 1 | Header minor version | Not checked |
| `0x05` | 1 | Header major version | Must equal `0`. Reject any other value |
| `0x06` | 2 | Padding | Should be zero. Not checked |
| `0x08` | 4 | Firmware version hash | Informational. Not checked |
| `0x0C` | 4 | Padding | Should be zero. Not checked |
| `0x10` | 4 | Image size | Number of bytes, from offset 0, covered by header plus entries |

Validation **[HW]** (format rules; the host must reject an image that breaks
them):

1. The file must be at least 20 bytes long.
2. The magic must match.
3. Header major version must be 0.
4. Image size must be ≤ the file length. Bytes past image size are ignored by
   entry parsing, but section payloads can still refer to them (5.2.3).
5. Entries are parsed from offset `0x14` until the parse offset reaches image
   size. An entry that would run past image size is an error.

### 5.2.2 Entry header (4 bytes)

Every entry starts with one 32-bit word:

| Bits | Field | Meaning |
|---|---|---|
| [7:0] | TYPE | Entry type (table below) |
| [15:8] | SIZE | Total entry size in bytes, **including** this 4-byte header. Must be a multiple of 4 |
| [29:16] | – | Not interpreted |
| 30 | UPDATABLE | Entry may be updated at runtime. Not used by the Linux host |
| 31 | OPTIONAL | Entry can be skipped if the host does not understand TYPE |

Rules:

- **[HW]** Each entry header must start at a 4-byte-aligned file offset, and
  SIZE must be a multiple of 4. Otherwise reject the image.
- **[HW]** The payload is the `SIZE − 4` bytes that follow the header. The next
  entry starts immediately after it. Payload length is therefore at most 251
  bytes.
- **[HW]** An entry whose payload would extend past image size is an error.

> OPEN QUESTION: SIZE < 4 is not explicitly rejected. A payload length that
> goes negative would also run past image size and be rejected, so a clean-room
> parser should reject SIZE < 4 outright.

Entry types:

| TYPE | Name | Linux handling |
|---|---|---|
| 0 | Interface / memory section | Parsed and loaded (5.2.3) |
| 1 | Firmware configuration | Recognized, skipped |
| 2 | Firmware unit-test (FUTF) | Recognized, skipped |
| 3 | Trace buffer | Recognized, skipped |
| 4 | Timeline metadata | Recognized, skipped |
| 5 | (not defined) | Treated as unknown |
| 6 | Build-info metadata | Parsed for logging only (5.2.4) |
| other | Unknown | Skipped if OPTIONAL = 1, otherwise reject the image |

- **[POLICY]** Types 1–4 are skipped whatever the OPTIONAL bit says. A driver
  that does not implement them can safely do the same: the firmware boots and
  runs without host support for them.

> OPEN QUESTION: The payload layouts of types 1 (configuration), 2 (unit
> test), 3 (trace buffer) and 4 (timeline metadata) are not described by the
> source studied. Type 3 presumably describes a ring buffer the host can read
> for firmware logs, and type 4 a blob of timeline-tracing metadata. Neither
> is needed for basic operation.

### 5.2.3 Type 0: Interface / memory section entry

Offsets are relative to the start of the entry (the entry header is at +0).

| Offset | Size | Field | Meaning |
|---|---|---|---|
| `+0x00` | 4 | Entry header | TYPE = 0 |
| `+0x04` | 4 | FLAGS | See table below |
| `+0x08` | 4 | VA_START | First MCU virtual address of the section |
| `+0x0C` | 4 | VA_END | MCU virtual address one past the end (exclusive) |
| `+0x10` | 4 | DATA_START | File offset of initial contents |
| `+0x14` | 4 | DATA_END | File offset one past the end of initial contents (exclusive) |
| `+0x18` | SIZE − 0x18 | NAME | Optional section name: raw bytes up to the end of the entry. Not guaranteed to be NUL-terminated and may include trailing NUL padding. Informational only |

FLAGS:

| Bit(s) | Name | Meaning |
|---|---|---|
| 0 | RD | Readable by the MCU |
| 1 | WR | Writable by the MCU. If clear, map read-only |
| 2 | EX | Executable by the MCU. If clear, map no-execute |
| [4:3] | CACHE_MODE | 0 = none (uncached), 1 = cached, 2 = uncached-coherent, 3 = cached-coherent |
| 5 | PROT | Protected-mode section (5.10) |
| [29:6] | – | Must be zero |
| 30 | SHARED | Section is shared with the host. The host keeps a CPU mapping of it |
| 31 | ZERO | Zero-fill the part of the section not covered by initial data |

Validation, in the order the Linux host applies it. Each failure rejects the
whole image unless stated otherwise.

1. **[HW]** The payload must hold at least the 20 bytes of FLAGS..DATA_END.
2. **[HW]** DATA_END ≥ DATA_START.
3. **[HW]** VA_END ≥ VA_START.
4. **[HW]** DATA_END ≤ total *file* length. This is checked against the file
   length, not against the header's image size.
5. **[HW]** VA_START and VA_END must both be aligned to the MCU VM page size
   (4 KiB, see 5.3).
6. **[HW]** No FLAGS bit outside {0, 1, 2, 3, 4, 5, 30, 31} may be set.
7. **[POLICY]** If PROT is set, the entry is **skipped**: no memory is
   allocated and nothing is mapped. A warning is logged and parsing continues.
8. **[HW]** If VA_START = `0x04000000` (the shared interface address, 5.4),
   SHARED must be set.
9. **[HW]** (DATA_END − DATA_START) ≤ (VA_END − VA_START).
10. **[POLICY]** If VA_END = VA_START (empty section), the entry is skipped.

After all entries are parsed, **[HW]** exactly one loaded section must start
at MCU VA `0x04000000`. If none does, reject the image.

> OPEN QUESTION: The RD bit is accepted but never used. Every mapping the host
> creates is readable. Whether firmware ever ships a section with RD clear is
> unknown.

### 5.2.4 Type 6: Build-info metadata entry

| Offset | Size | Field | Meaning |
|---|---|---|---|
| `+0x00` | 4 | Entry header | TYPE = 6 |
| `+0x04` | 4 | META_START | File offset of the metadata blob |
| `+0x08` | 4 | META_SIZE | Length of the blob in bytes |

Blob format: the ASCII prefix `git_sha: ` (9 bytes, including the trailing
space), then the firmware source revision as text, ending in a NUL byte that
must be the last byte of the blob.

- **[POLICY]** The host only logs the revision string. Every failure here is
  non-fatal and the entry is ignored if any of these hold: the blob falls
  outside the file, META_SIZE ≤ 9, the prefix does not match, or the last byte
  is not NUL.

## 5.3 MCU virtual address space and section placement

### 5.3.1 The MCU VM

- **[HW]** The MCU accesses memory through the GPU MMU using **address-space
  slot 0**. The MCU's address space must be bound to AS 0, and AS 0 must not be
  handed to user GPU contexts.
- **[HW]** The MCU is a 32-bit core, so its virtual address space is limited
  to 4 GiB (`0x0000_0000`–`0xFFFF_FFFF`). Every MCU address in the firmware
  image and in the shared interface is a 32-bit field.
- **[HW]** The MCU VM uses the same page-table format as other GPU address
  spaces (AArch64 stage-1 style, 4 KiB granule, see 04-mmu). The smallest page
  size is 4 KiB, and 2 MiB block mappings may be used. Section alignment is
  checked against 4 KiB.
- **[POLICY]** Linux layout of the MCU VM:

| Range | Use |
|---|---|
| `0x0000_0000`–`0xFFFF_FFFF` | Whole range is host-managed. Firmware sections are mapped at the fixed addresses given in the image |
| `0x0400_0000`–`0x07FF_FFFF` (64 MiB) | Window from which the host allocates addresses for its own MCU-visible objects (5.3.4). The firmware's shared interface section also starts at `0x0400_0000` and is placed first, at its fixed address |

> OPEN QUESTION: Whether the firmware requires host-allocated objects (queue
> rings, suspend buffers) to sit inside `0x04000000`–`0x07FFFFFF`, or whether
> any free MCU VA works, is not established. Linux always uses that window.

### 5.3.2 AS 0 programming for the MCU

Before the MCU is started, the host programs AS 0 (registers in 02 §2.7.2,
command sequence in 04-mmu):

1. **[HW]** Unmask the MMU fault interrupt bit for AS 0.
2. **[HW]** `AS_TRANSTAB` = physical address of the MCU VM's top-level
   translation table.
3. **[HW]** `AS_MEMATTR` = the memory attribute table (5.3.3).
4. **[HW]** `AS_TRANSCFG`:
   - ADRMODE = 6 (AArch64, 4 KiB granule)
   - INA_BITS = 55 − (VA bits from `MMU_FEATURES[7:0]`). Linux uses the full
     GPU VA width here, not 32.
   - PTW_MEMATTR = 2 (write-back)
   - PTW_RA = 1
   - PTW_SH = 2 (outer shareable) only if the GPU is I/O-coherent with the
     CPU, otherwise 0
5. **[HW]** Issue `AS_COMMAND` = UPDATE (1) and wait for `AS_STATUS` bit 0 to
   clear.

### 5.3.3 Section cache mode and memory attributes

The host turns section FLAGS into page permissions and a memory-attribute
index:

| Section FLAGS | Page mapping |
|---|---|
| WR = 0 | Read-only (AP[2] = 1) |
| WR = 1 | Read/write |
| EX = 0 | No-execute (XN bits set) |
| EX = 1 | Executable |
| CACHE_MODE = 1 (cached) | AttrIndx 1 (cacheable), inner shareable |
| CACHE_MODE = 0, 2 or 3 | AttrIndx 0 (non-cacheable), outer shareable |

- **[POLICY]** Linux maps the two *coherent* cache modes (2 and 3) as plain
  non-cacheable. A driver on an I/O-coherent system may map them as
  shareable/coherent instead.

> OPEN QUESTION: The intended attribute for "cached-coherent" (3) is probably
> write-back with CPU-coherent shareability (`AS_MEMATTR` shareability 1 or 2,
> or memory type 0 "shared"). This is not confirmed.

`AS_MEMATTR` value Linux writes (eight 8-bit entries, entry *i* in bits
[8i+7:8i], encoding as in 02 §2.7.3):

| Index | Intended type | Entry (non-coherent GPU) | Entry (coherent GPU) |
|---|---|---|---|
| 0 | Normal non-cacheable | `0x4C` | `0x4C` |
| 1 | Normal write-back, read+write allocate | `0x8F` | `0x9F` |
| 2 | Device | `0x4C` | `0x4C` |
| 3 | Inner non-cacheable / outer write-back | `0x8C` | `0x9C` |
| 4–7 | Unused | `0x4C` | `0x4C` |

Full register: `0x4C4C4C4C_8C4C8F4C` on a non-coherent GPU (RK3588),
`0x4C4C4C4C_9C4C9F4C` on a coherent one.

- **[HW]** On a non-coherent GPU, write-back memory must use shareability 0
  (GPU-internal "inner"). Inner- and outer-shareable write-back attributes are
  not allowed when the GPU is not coherent with the CPU.
- **[POLICY]** Index 3 is approximated as full write-back. Only indices 0 and
  1 are used by firmware sections.

### 5.3.4 Loading section contents

For every accepted section:

1. **[HW]** Allocate physically backed memory of size
   `VA_END − VA_START`, rounded up to the MCU page size. Map it at exactly
   `VA_START` in the MCU VM, with the attributes from 5.3.3. Mapping more than
   the section size is not allowed: neighbouring sections may start on the next
   MCU page, even when the CPU page size is larger.
2. **[HW]** Copy file bytes `[DATA_START, DATA_END)` to section offset 0.
3. **[HW]** If ZERO is set, write zeros from offset `DATA_END − DATA_START` to
   the end of the section.
4. If ZERO is clear, the rest of the section has no required contents.
   **[POLICY]** Linux allocates zeroed pages anyway. If DATA is empty and ZERO
   is clear, the host writes nothing.
5. **[HW]** On a non-coherent system, clean (write back) the CPU caches for the
   section so the MCU sees the data.
6. **[HW]** Keep a host copy of the initial data bytes, together with FLAGS and
   placement. They are needed to reload the section after a full reset (5.7.3).
7. **[HW]** For a SHARED section, keep a permanent CPU mapping. **[POLICY]**
   On a non-coherent system Linux maps it write-combined (uncached). On a
   coherent system it uses a normal cached mapping.
8. The section memory must stay resident (pinned) while the firmware can run.

Host-allocated MCU objects (later sections and 06/07 describe them in full)
are also mapped into the MCU VM:

- **[POLICY]** Each queue's ring-buffer control page pair: 8 KiB, mapped
  uncached and no-execute, with the input page at offset 0 and the output page
  at +4 KiB. It is zero-filled before first use.
- **[POLICY]** Per-group suspend buffers (normal and protected). Sizes come from
  `CSG_SUSPEND_SIZE` and `CSG_PROTM_SUSPEND_SIZE` (5.4.3). They are mapped
  cached and no-execute.

## 5.4 Locating the shared interface and discovering slots

### 5.4.1 The shared interface section

- **[HW]** The firmware section that starts at MCU VA `0x04000000` holds the
  host↔firmware interface. Its first bytes are the **global control block**.
- **[HW]** Interface blocks contain MCU addresses (`*_INPUT_VA`,
  `*_OUTPUT_VA`). The host converts an MCU address *A* to a CPU pointer as
  `shared_cpu_base + (A − 0x04000000)`. This works only if the whole object
  `[A, A + size)` lies inside the shared section. Otherwise the host must
  treat the interface as invalid and fail initialization.
- **[HW]** The control blocks are written by the firmware during boot. The
  host may read them only **after** the MCU has signalled boot completion
  (5.5).

The field names below are mnemonics used across this specification.
06-csf-interface is authoritative for every field not needed for discovery.

### 5.4.2 Global control block (at shared offset `0x0000`, read-only to host)

| Offset | Name | Meaning |
|---|---|---|
| `0x00` | `GLB_VERSION` | Interface version: [31:24] major, [23:16] minor, [15:0] patch. 0 means the firmware did not boot |
| `0x04` | `GLB_FEATURES` | Global feature bits (logged by Linux, see 06) |
| `0x08` | `GLB_INPUT_VA` | MCU address of the global *input* block (host writes) |
| `0x0C` | `GLB_OUTPUT_VA` | MCU address of the global *output* block (firmware writes) |
| `0x10` | `GLB_GROUP_NUM` | Number of CSG slots |
| `0x14` | `GLB_GROUP_STRIDE` | Byte stride between CSG control blocks |
| `0x18` | `GLB_PRFCNT_SIZE` | Performance-counter buffer size information (see 06) |
| `0x1C` | `GLB_INSTR_FEATURES` | Instrumentation features. Valid only if version ≥ 1.1.0 |

Size checked by the host: 0x20 bytes.

### 5.4.3 CSG control blocks

CSG slot *g* (0 ≤ *g* < `GLB_GROUP_NUM`) has its control block at shared
offset **`0x1000 + g × GLB_GROUP_STRIDE`**.

| Offset | Name | Meaning |
|---|---|---|
| `0x00` | `CSG_FEATURES` | Group features |
| `0x04` | `CSG_INPUT_VA` | MCU address of the CSG input block |
| `0x08` | `CSG_OUTPUT_VA` | MCU address of the CSG output block |
| `0x0C` | `CSG_SUSPEND_SIZE` | Bytes needed for a normal suspend buffer |
| `0x10` | `CSG_PROTM_SUSPEND_SIZE` | Bytes needed for a protected-mode suspend buffer |
| `0x14` | `CSG_STREAM_NUM` | Number of CS slots in this group |
| `0x18` | `CSG_STREAM_STRIDE` | Byte stride between CS control blocks |

Size checked by the host: 0x1C bytes.

### 5.4.4 CS control blocks

CS slot *s* of CSG *g* has its control block at shared offset
**`0x1000 + g × GLB_GROUP_STRIDE + 0x40 + s × CSG_STREAM_STRIDE`**.

| Offset | Name | Meaning |
|---|---|---|
| `0x00` | `CS_FEATURES` | [7:0] = work-register count − 1. [15:8] = number of scoreboards. Bit 16 = compute, bit 17 = fragment, bit 18 = tiler |
| `0x04` | `CS_INPUT_VA` | MCU address of the CS input block |
| `0x08` | `CS_OUTPUT_VA` | MCU address of the CS output block |

Size checked by the host: 0x0C bytes.

### 5.4.5 Discovery procedure and validation

1. Make sure the CPU mapping of the shared section exists and the section is
   at least 0x20 bytes.
2. Read `GLB_VERSION`. If it is 0, fail: the firmware did not come up.
3. Resolve `GLB_INPUT_VA` and `GLB_OUTPUT_VA` to CPU pointers (5.4.1). The
   minimum object sizes are in 5.4.6.
4. Check that `GLB_GROUP_NUM` is in range:
   - **[HW]** at most 31. JOB interrupt bits 0–30 are the only per-CSG bits
     (02 §2.6).
   - **[POLICY]** at least 3.
5. For each CSG slot:
   1. Its control block must lie entirely inside the shared section.
   2. Resolve its input/output VAs.
   3. `CSG_STREAM_NUM` must be ≤ 32 **[HW]** (per-CS bits in the CSG
      request/acknowledge words are 32 wide) and ≥ 8 **[POLICY]**.
   4. **[POLICY]** For every slot after the first, `CSG_FEATURES`,
      `CSG_SUSPEND_SIZE`, `CSG_PROTM_SUSPEND_SIZE` and `CSG_STREAM_NUM` must
      match slot 0. Linux treats all CSG slots as interchangeable and rejects
      firmware that says otherwise.
   5. For each CS slot: its control block must lie inside the shared section,
      its input/output VAs must resolve, and **[POLICY]** its `CS_FEATURES`
      must match CS 0 of CSG 0.
6. From CS 0 of CSG 0, record the work-register count
   (`CS_FEATURES[7:0] + 1`). **[ABI]** This count is reported to userspace
   together with a fixed "unpreserved register" count of 4 (see 10-uapi).

### 5.4.6 Minimum interface object sizes (for bounds checks)

The host bounds-checks each resolved object against the shared section using
the size of the layout it knows about:

| Object | Bytes |
|---|---|
| Global input | 0x84 |
| Global output | 0x1C |
| CSG input | 0x58 |
| CSG output | 0x20 |
| CS input | 0x58 |
| CS output | 0xD8 |

### 5.4.7 Interface version gates

| Version (major.minor.patch) | Effect |
|---|---|
| ≥ 1.1.0 | `GLB_INSTR_FEATURES` is valid. Treat it as 0 below this version |
| ≥ 4.0.0 | The CSG endpoint request is a 64-bit field at a new offset (see 06) |
| ≥ 4.1.0 | `GLB_REQ` / `GLB_ACK` carry a 3-bit **state** field [14:12] that replaces the single HALT bit (5.7.1) |

## 5.5 MCU boot sequence

Prerequisites, all from 03-gpu-control and 04-mmu: the GPU is powered and
identified, has been soft-reset, has its coherency mode and L2 configuration
set, and has the **L2 cache powered on**.

1. **[HW]** Get the JOB interrupt line. In the devicetree binding it is the
   interrupt named `job`.
2. **[HW]** Build the MCU VM, then parse and load the firmware image (5.2,
   5.3.4).
3. **[HW]** Bind the MCU VM to AS 0 (5.3.2).
4. **[HW]** Enable JOB interrupt events. **[POLICY]** Linux enables all 32
   bits: it writes all-ones to JOB `INT_CLEAR`, then all-ones to JOB
   `INT_MASK`.
5. **[HW]** Write `MCU_CONTROL` = 2 (AUTO).
6. **[HW]** Wait for JOB interrupt bit 31 (global interface). That bit is the
   firmware's "boot complete" signal. Before it arrives, other JOB bits are
   acknowledged and ignored.
7. **[POLICY]** Timeout is 1 s. When the timeout expires, check JOB
   `INT_STAT` bit 31 once more before failing, in case the event arrived but
   its interrupt was missed. On failure, read `MCU_STATUS` for diagnosis:
   0 disabled, 1 enabled, 2 halted, 3 fatal.
8. **[HW]** Discover the interface (5.4). This is done on first boot only. On
   later boots the layout is assumed unchanged.
9. **[HW]** Write the initial global configuration and ring doorbell 0 (5.6).
10. **[POLICY]** Start the watchdog (5.8).

> OPEN QUESTION: The difference between `MCU_CONTROL` = 1 (ENABLE) and
> 2 (AUTO) is not documented in the source. Linux always uses AUTO.
> Presumably AUTO lets the hardware enable and disable the MCU automatically
> as the GPU powers up and down.

## 5.6 Initial global configuration

This is done after every successful MCU start, including after a fast reset.

### 5.6.1 Global input/output fields used at boot

Offsets are within the global input block (CPU address resolved from
`GLB_INPUT_VA`) and the global output block (from `GLB_OUTPUT_VA`).

| Block | Offset | Name | Width | Use at boot |
|---|---|---|---|---|
| Input | `0x00` | `GLB_REQ` | 32 | Requests and configuration bits |
| Input | `0x04` | `GLB_ACK_IRQ_MASK` | 32 | Which `GLB_ACK` changes raise JOB bit 31 |
| Input | `0x08` | `GLB_DB_REQ` | 32 | CSG doorbell requests (06) |
| Input | `0x10` | `GLB_PROGRESS_TIMER` | 32 | Progress timeout |
| Input | `0x14` | `GLB_PWROFF_TIMER` | 32 | Shader-core power-off hysteresis |
| Input | `0x18` | `GLB_ALLOC_EN` | 64 | Mask of shader cores the firmware may use |
| Input | `0x80` | `GLB_IDLE_TIMER` | 32 | GPU idle hysteresis |
| Output | `0x00` | `GLB_ACK` | 32 | Firmware acknowledgements |
| Output | `0x08` | `GLB_DB_ACK` | 32 | CSG doorbell acknowledgements |
| Output | `0x10` | `GLB_HALT_STATUS` | 32 | Reason for halt (5.8) |

`GLB_REQ` / `GLB_ACK` bits used in this section (06 has the full list):

| Bit(s) | Name | Kind |
|---|---|---|
| 0 | HALT | Level, for interface < 4.1. Not acknowledged in `GLB_ACK` |
| 1 | CFG_PROGRESS_TIMER | Toggle request: apply `GLB_PROGRESS_TIMER` |
| 2 | CFG_ALLOC_EN | Toggle request: apply `GLB_ALLOC_EN` |
| 3 | CFG_PWROFF_TIMER | Toggle request: apply `GLB_PWROFF_TIMER` |
| 7 | COUNTER_EN | Level: enable the GPU cycle/timestamp counters |
| 8 | PING | Toggle request: firmware liveness check |
| 10 | IDLE_EN | Level: enable idle detection (uses `GLB_IDLE_TIMER`) |
| [14:12] | STATE | Interface ≥ 4.1: 0 active, 1 halt, 2 sleep, 3 suspend |
| 26 | IDLE | Firmware event: GPU went idle |

**Request model (summary, 06 is normative).** For a *toggle* request, the
host sets the `GLB_REQ` bit to the inverse of the matching `GLB_ACK` bit. When
the firmware has acted on it, it copies the bit into `GLB_ACK`, so the request
is complete once the two bits are equal again. A *level* bit is simply written
to the wanted value. A *firmware event* runs the other way: the firmware
toggles the `GLB_ACK` bit, and the host acknowledges by copying it into
`GLB_REQ`. After changing `GLB_REQ`, the host must ring **doorbell 0**
(02 §2.9) to make the firmware re-read it.

### 5.6.2 Timer value encoding (`GLB_PWROFF_TIMER`, `GLB_IDLE_TIMER`)

| Bits | Field | Meaning |
|---|---|---|
| [30:0] | VALUE | Timeout in units of **1024 ticks** of the selected source |
| 31 | SOURCE | 0 = system timestamp counter (the same clock as `GPU_TIMESTAMP`, i.e. the SoC architected-timer frequency). 1 = GPU cycle counter (GPU core clock) |

Converting a timeout of *T* µs:

1. Choose a source. **[POLICY]** Linux uses the system timer (SOURCE = 0) at
   its advertised frequency *f* when that frequency is known. Otherwise it
   uses the GPU core clock rate as *f*, with SOURCE = 1.
2. VALUE = ceil(*T* × *f* / (1 000 000 × 1024)), clamped to `0x7FFFFFFF`.
3. **[POLICY]** If no frequency is known at all, write `0xFFFFFFFF` (maximum
   value, GPU cycle counter). This keeps core power management working.

### 5.6.3 Boot-time values

| Field | Value written | Tag |
|---|---|---|
| `GLB_ALLOC_EN` | `SHADER_PRESENT` (all shader cores) | [POLICY] |
| `GLB_PWROFF_TIMER` | 10 000 µs (10 ms), encoded per 5.6.2 | [POLICY] |
| `GLB_PROGRESS_TIMER` | `2 560 000` (= 2.5 × 2^30 GPU cycles ÷ 1024). No source bit | [POLICY] |
| `GLB_IDLE_TIMER` | 800 µs, encoded per 5.6.2 | [POLICY] |
| `GLB_ACK_IRQ_MASK` | Bits 1, 2, 3, 8, 10, 26 = `0x0400050E`. If interface ≥ 4.1, also bits [14:12], giving `0x0400750E` | [POLICY] |

> OPEN QUESTION: `GLB_PROGRESS_TIMER` appears to count in units of 1024 GPU
> cycles with no source selector, since Linux writes the cycle count shifted
> right by 10. Its exact semantics (per-CS time without forward progress
> before the firmware raises a progress-timer event) belong in 06/07.

Worked example (RK3588, 24 MHz system counter): power-off =
ceil(10 000 × 24 000 000 / 1 024 000 000) = 235 = `0x000000EB`. Idle =
ceil(800 × 24e6 / 1.024e9) = 19 = `0x00000013`.

Sequence:

1. Write `GLB_ALLOC_EN`, `GLB_PWROFF_TIMER`, `GLB_PROGRESS_TIMER`,
   `GLB_IDLE_TIMER` and `GLB_ACK_IRQ_MASK`.
2. In `GLB_REQ`, set level bits IDLE_EN (10) and COUNTER_EN (7) to 1, leaving
   the other bits alone.
3. In `GLB_REQ`, toggle CFG_ALLOC_EN (2), CFG_PWROFF_TIMER (3) and
   CFG_PROGRESS_TIMER (1). Each becomes the inverse of its `GLB_ACK` bit.
4. **[HW]** Make sure all writes to shared memory are visible before the
   doorbell write (write barrier, or device-ordered MMIO write).
5. Ring doorbell 0 (write 1 to MMIO `0x80000`).
6. **[POLICY]** Linux does not wait for these acknowledgements at boot. They
   complete in the background.

## 5.7 Halt, stop, suspend/resume and reset

### 5.7.1 Requesting a halt

1. **[HW]** If interface ≥ 4.1: set `GLB_REQ[14:12]` = 1 (HALT). Otherwise:
   set `GLB_REQ` bit 0 = 1.
2. **[HW]** Ring doorbell 0.
3. **[HW]** Wait until `MCU_STATUS` = 2 (halted). For interface ≥ 4.1, also
   wait until `GLB_ACK[14:12]` = 1. For interface < 4.1 there is no `GLB_ACK`
   for HALT, so only `MCU_STATUS` is checked.
4. **[POLICY]** Poll every 10 µs with a 1 s timeout. If the timeout expires,
   the halt failed and the next restart must be a full reload.

To leave the halted state before the next boot (fast restart only):

- **[HW]** For interface ≥ 4.1, set `GLB_REQ[14:12]` = 0 (ACTIVE). Otherwise,
  clear `GLB_REQ` bit 0. No doorbell is rung: the MCU is not running, and the
  firmware reads the value when it starts.

### 5.7.2 Stopping the MCU

1. **[HW]** Mask the JOB interrupt (`INT_MASK` = 0) and wait for any running
   handler to finish.
2. **[HW]** Write `MCU_CONTROL` = 0 (DISABLE).
3. **[HW]** Wait until `MCU_STATUS` = 0. **[POLICY]** Poll every 10 µs with a
   100 ms timeout. A timeout is logged but not treated as fatal.

### 5.7.3 Fast restart and full reload

The host keeps one flag, "fast restart possible". It is cleared at the start
of every stop and set only when a graceful halt (5.7.1) succeeded.

**Before reset or suspend:**

1. **[POLICY]** Cancel the watchdog.
2. If this is *not* a hang recovery, request a halt (5.7.1). If the halt
   completes, mark fast restart possible.
3. Stop the MCU (5.7.2).

**After reset or resume:**

1. **[HW]** Re-bind the MCU VM to AS 0 (5.3.2). AS registers are lost across a
   GPU reset or power-down.
2. Then one of:
   - **Fast restart** (halt succeeded): section memory is left as it is. The
     firmware resumes from the state it saved in its own writable sections,
     which live in system RAM and survive GPU power-down. Clear the halt
     request (5.7.1).
   - **Full reload**: for every loaded section, copy the saved initial data
     again and re-apply zero-fill (5.3.4 steps 2, 3 and 5). **[POLICY]** Linux
     reloads *all* sections, read-only ones included, to recover from any
     memory corruption. Strictly, only writable (WR = 1) sections can have
     been changed by the MCU, and Linux has a mode that reloads only those,
     but the full-reload path is the one it actually uses.
3. **[HW]** Boot the MCU (5.5 steps 4–7).
4. **[HW]** Redo the initial global configuration (5.6), including after a
   fast restart. A halted-then-restarted firmware must be configured again
   before use.
5. **[POLICY]** If the restart fails, the device is declared unusable.

**[POLICY] When each path is used:**

| Trigger | Halt attempted? | Restart type |
|---|---|---|
| System or runtime suspend → resume | Yes | Fast if the halt succeeded. If the fast resume fails, Linux retries once with a full reload |
| Resume while a reset was pending at suspend time | – | Forced full reload |
| Reset after a hang or timeout (watchdog, scheduler, fault) | No | Always full reload |

**[POLICY] Order of operations for a recovery reset:** quiesce the scheduler →
pre-reset firmware stop (no halt) → MMU pre-reset → GPU soft reset (03) →
L2 power-on → MMU post-reset (AS re-enable) → firmware restart (full reload)
→ scheduler restart.

**[POLICY] Order of operations for suspend:** suspend the scheduler (CSGs
saved, 07) → firmware halt and stop → MMU suspend → GPU block suspend → clocks
off. In the GPU block suspend, a soft reset is done if the halt failed;
otherwise the L2 is just powered off. Resume runs in reverse: GPU block (L2
power-on), then MMU, then firmware restart.

### 5.7.4 Unplug or teardown

**[POLICY]** Stop the watchdog. If the device is powered, mask the JOB
interrupt and stop the MCU. Then unmap and free all sections, release the MCU
VM and AS 0, and power off the L2.

## 5.8 Hang detection

- **[POLICY] Host ping watchdog.** Every 12 s, the host toggles `GLB_REQ` bit 8
  (PING) and rings doorbell 0. It then waits up to 100 ms for `GLB_ACK` bit 8 to
  match. The wait busy-polls for about 10 µs, then sleeps until a JOB
  interrupt arrives or the timeout expires, then checks once more. If the ping
  times out, the host schedules a full reset (5.7.3). If it succeeds, the next
  ping is rescheduled. No ping is sent while a reset is pending. The watchdog
  restarts after every successful global configuration (5.6).
- **[HW] Firmware halt reasons.** `GLB_HALT_STATUS` (global output `0x10`)
  reports why the firmware halted:

| Value | Meaning |
|---|---|
| `0x00` | Normal halt (requested) |
| `0x4E` | Firmware panic |
| `0x4F` | Firmware internal watchdog expired |

- **[HW]** `MCU_STATUS` = 3 (FATAL) means the MCU hit an unrecoverable error.
- **[POLICY]** Linux defines the halt-status codes but never reads them, and it
  does not poll `MCU_STATUS` during normal operation. Hangs are found only by
  the ping, by scheduler-level timeouts (07), or by fault reports.

> OPEN QUESTION: Nothing in the source shows a firmware-initiated reset
> request, or how a spontaneous firmware halt (panic or watchdog) is
> signalled to the host. Presumably the MCU stops answering and the ping
> timeout catches it, but an interrupt (for example a GPU `MCU_STATUS_CHANGED`
> event, 02 §2.3.2) might also be raised. A robust driver could unmask
> `MCU_STATUS_CHANGED` and check `MCU_STATUS` and `GLB_HALT_STATUS`.

## 5.9 JOB interrupt handling (top level)

The JOB interrupt block (base `0x1000`, 02 §2.6) is the firmware's only
interrupt to the host. Bit 31 is global-interface activity. Bits 0–30 are CSG
slot activity.

1. **[HW]** On interrupt, read JOB `INT_STAT`. If it is 0, the interrupt came
   from another source on a shared line.
2. **[POLICY]** Linux masks the block (`INT_MASK` = 0) in hard-IRQ context and
   does the rest in a thread.
3. Loop:
   1. Read `status` = JOB `INT_RAWSTAT` & enabled mask. Stop if it is 0.
   2. **[HW]** Write `status` to JOB `INT_CLEAR` **before** processing it.
      The line is level-triggered, and clearing first means firmware updates
      made during processing raise the interrupt again instead of being lost.
   3. If the firmware is not yet marked booted and bit 31 is set, mark it
      booted (5.5 step 6).
   4. Wake every thread waiting on a firmware acknowledgement. Acknowledgement
      waits re-check the `ACK` words in shared memory when woken.
   5. If the firmware is not booted, discard the events.
   6. If bit 31 is set: global events = (`GLB_REQ` XOR `GLB_ACK`) restricted
      to the firmware-event bits [26:20]. Linux acts only on IDLE (bit 26),
      which it acknowledges by copying the bit into `GLB_REQ` and passes to
      the scheduler (07). Details are in 06.
   7. For each set bit *n* in 0–30: process CSG slot *n*'s request and
      acknowledge words (06, 07).
4. **[POLICY]** Restore `INT_MASK` to the enabled set.

## 5.10 Protected mode

- **[HW]** The firmware supports a protected (secure-content) execution mode.
  Signs of it in the interface:
  - sections flagged PROT in the image (5.2.3)
  - `CSG_PROTM_SUSPEND_SIZE` and a per-group protected suspend buffer
  - `GLB_REQ` bits 4 (PROTM_ENTER) and 23 (PROTM_EXIT)
  - `GPU_STATUS` bit 7 and the GPU `PROTM_FAULT` interrupt (02)
- **[POLICY]** Linux does **not** support protected mode:
  - PROT sections are skipped at load time (5.2.3 rule 7).
  - The host never requests protected-mode entry.
  - Protected suspend buffers are still allocated and programmed for each
    group, so the firmware's view of each group is complete.

> OPEN QUESTION: Loading PROT sections presumably needs memory that only the
> secure world can reach, plus a secure-monitor service. The requirements are
> not visible in the source.
