# 02: Register Map (Mali Valhall/5th-gen CSF GPUs)

This section lists the MMIO registers that a driver for Mali CSF GPUs
(architecture major 10 through 14) touches. Sequences that use these registers
are in later sections (03 GPU control, 04 MMU, 05/06 firmware).

## 2.1 Access conventions

- **[HW]** The register window is one contiguous MMIO region. On RK3588 it is
  2 MiB at physical `0xFB000000`. All offsets below are relative to the start
  of the window.
- **[HW]** All registers are 32 bits wide and must be accessed with 32-bit
  loads and stores.
- **[HW]** A "64-bit register" is a pair of adjacent 32-bit registers: low word
  at offset `N`, high word at `N+4`.
  - Writes: write the low word, then the high word.
  - Configuration reads (feature and present masks): read low then high.
  - Free-running counters (`GPU_CYCLE_COUNT`, `GPU_TIMESTAMP`): read high,
    then low, then high again. Repeat until both high reads match, so a carry
    between the two halves is not missed.
- **[HW]** Device-ordered (non-relaxed) MMIO accessors are needed wherever a
  register write must be ordered against normal memory writes, for example a
  doorbell ring after updating shared memory.

### 2.1.1 Standard interrupt register block

Each interrupt source (GPU, MMU, JOB, PWR) has a block of four 32-bit
registers at a source-specific base:

| Offset from block base | Name | Access | Meaning |
|---|---|---|---|
| `+0x0` | `INT_RAWSTAT` | R | Pending events, not masked |
| `+0x4` | `INT_CLEAR` | W | Write 1 to acknowledge (clear) a bit |
| `+0x8` | `INT_MASK` | RW | 1 = event may assert the IRQ line |
| `+0xC` | `INT_STAT` | R | `RAWSTAT & MASK` |

**[HW]** The interrupt lines are level-triggered. An event must be acknowledged
through `INT_CLEAR` before the line deasserts. A driver may poll `INT_RAWSTAT`
for completion events with the interrupt masked.

## 2.2 Block overview

| Base | Size | Block | Notes |
|---|---|---|---|
| `0x00000` | ~0x400 | GPU control | Identification, features, reset, cache, legacy power (arch ≤ 13) |
| `0x00020` | 0x10 | GPU interrupt block | Inside GPU control |
| `0x00700` | 0x08 | MCU control | Microcontroller (firmware CPU) enable and status |
| `0x00800` | ~0x400 | PWR control | **Arch 14+ only**: new power controller |
| `0x00800` | 0x10 | PWR interrupt block | Arch 14+ only |
| `0x01000` | 0x10 | JOB interrupt block | Firmware → host notifications |
| `0x02000` | 0x10 | MMU interrupt block | |
| `0x02400` | 0x40 × N | MMU address-space (AS) slots | N = bits set in `GPU_AS_PRESENT` |
| `0x10000` | 4 KiB page | `LATEST_FLUSH_ID` | Read-only page that can be mapped into userspace |
| `0x80000` | 64 KiB × M | Doorbell pages | Doorbell `i` at `0x80000 + i × 0x10000` |

## 2.3 GPU control block (base `0x0000`)

| Offset | Name | Width | Access | Description |
|---|---|---|---|---|
| `0x000` | `GPU_ID` | 32 | R | Product and version ID (see 2.3.1). Reads 0 if the GPU is unpowered or absent |
| `0x004` | `L2_FEATURES` | 32 | R | Bits [7:0] = log2(cache line size in bytes). Other fields are passed to userspace unchanged |
| `0x008` | `CORE_FEATURES` | 32 | R | Passed to userspace unchanged |
| `0x00C` | `TILER_FEATURES` | 32 | R | Passed to userspace unchanged |
| `0x010` | `MEM_FEATURES` | 32 | R | Bit 0 = core groups are L2-coherent |
| `0x014` | `MMU_FEATURES` | 32 | R | [7:0] = VA bits, [15:8] = PA bits |
| `0x018` | `AS_PRESENT` | 32 | R | Bitmask of implemented MMU address-space slots |
| `0x01C` | `CSF_ID` | 32 | R | CSF hardware version, passed to userspace |
| `0x020` | GPU `INT_*` | 4×32 | – | Interrupt block, see 2.3.2 |
| `0x030` | `GPU_CMD` | 32 | W | Command register, see 2.3.3 |
| `0x034` | `GPU_STATUS` | 32 | R | See 2.3.4 |
| `0x03C` | `GPU_FAULT_STATUS` | 32 | R | Fault status for GPU-level (non-MMU) faults |
| `0x040` | `GPU_FAULT_ADDR` | 64 | R | Faulting address for GPU-level faults |
| `0x048` | `L2_CONFIG` | 32 | RW | Bit 24 = enable custom ASN hash (see `ASN_HASH`) |
| `0x050` | `PWR_KEY` | 32 | W | Write `0x2968A819` to unlock the `PWR_OVERRIDE*` registers |
| `0x054` | `PWR_OVERRIDE0` | 32 | RW | Power override (needs unlock) |
| `0x058` | `PWR_OVERRIDE1` | 32 | RW | Power override (needs unlock) |
| `0x060` | `GPU_FEATURES` | 64 | R | Arch 11+. Bit 2 = ray intersection, bit 5 = ray traversal. Linux reads it on every arch. OPEN QUESTION: presumed to read 0 on arch 10 |
| `0x088` | `TIMESTAMP_OFFSET` | 64 | RW | Offset added to the timestamp counter |
| `0x090` | `CYCLE_COUNT` | 64 | R | Free-running GPU cycle counter (counter read rule) |
| `0x098` | `TIMESTAMP` | 64 | R | GPU timestamp (counter read rule). Driven by the system timer frequency |
| `0x0A0` | `THREAD_MAX_THREADS` | 32 | R | Passed to userspace |
| `0x0A4` | `THREAD_MAX_WORKGROUP_SIZE` | 32 | R | Passed to userspace |
| `0x0A8` | `THREAD_MAX_BARRIER_SIZE` | 32 | R | Passed to userspace |
| `0x0AC` | `THREAD_FEATURES` | 32 | R | Passed to userspace |
| `0x0B0`–`0x0BC` | `TEXTURE_FEATURES[0..3]` | 4×32 | R | Passed to userspace |
| `0x100` | `SHADER_PRESENT` | 64 | R | Shader core bitmask (arch ≤ 13) |
| `0x110` | `TILER_PRESENT` | 64 | R | Tiler bitmask (arch ≤ 13) |
| `0x120` | `L2_PRESENT` | 64 | R | L2 slice bitmask (arch ≤ 13) |
| `0x140` | `SHADER_READY` | 64 | R | Powered-on cores |
| `0x150` | `TILER_READY` | 64 | R | |
| `0x160` | `L2_READY` | 64 | R | |
| `0x180` | `SHADER_PWRON` | 64 | W | Write mask to request power-on |
| `0x190` | `TILER_PWRON` | 64 | W | |
| `0x1A0` | `L2_PWRON` | 64 | W | |
| `0x1C0` | `SHADER_PWROFF` | 64 | W | Write mask to request power-off |
| `0x1D0` | `TILER_PWROFF` | 64 | W | |
| `0x1E0` | `L2_PWROFF` | 64 | W | |
| `0x200` | `SHADER_PWRTRANS` | 64 | R | Cores in a power transition |
| `0x210` | `TILER_PWRTRANS` | 64 | R | |
| `0x220` | `L2_PWRTRANS` | 64 | R | |
| `0x240` | `SHADER_PWRACTIVE` | 64 | R | |
| `0x250` | `TILER_PWRACTIVE` | 64 | R | |
| `0x260` | `L2_PWRACTIVE` | 64 | R | |
| `0x280` | `REVID` | 32 | R | Revision, passed to userspace |
| `0x2C0`–`0x2C8` | `ASN_HASH[0..2]` | 3×32 | RW | Custom L2 address hash. Only takes effect when `L2_CONFIG` bit 24 is set |
| `0x300` | `COHERENCY_FEATURES` | 32 | R | Bitmask of supported protocols. Bit n = protocol n below |
| `0x304` | `COHERENCY_PROTOCOL` | 32 | RW | 0 = ACE-Lite, 1 = ACE, 31 = none |

> On arch 14+, the `*_PRESENT` masks are read from the PWR block instead (2.5).
> The legacy `PWRON`/`PWROFF` registers are used only on arch 10–13.

### 2.3.1 `GPU_ID` fields

| Bits | Field | RK3588 (Mali-G610) |
|---|---|---|
| [31:28] | ARCH_MAJOR | 10 |
| [27:24] | ARCH_MINOR | – |
| [23:20] | ARCH_REV | – |
| [19:16] | PRODUCT_MAJOR | 7 |
| [15:12] | VERSION_MAJOR | – |
| [11:4] | VERSION_MINOR | – |
| [3:0] | VERSION_STATUS | – |

Known products, keyed by (ARCH_MAJOR, PRODUCT_MAJOR):

| Key | Product |
|---|---|
| (10, 2) | Mali-G710 |
| (10, 3) | Mali-G510 |
| (10, 4) | Mali-G310 |
| (10, 7) | **Mali-G610 (RK3588)** |
| (11, 2) | Mali-G715-Immortalis if ≥10 cores and ray intersection; Mali-G715 if ≥7 cores; otherwise Mali-G615 |
| (11, 3) | Mali-G615 |
| (12, 0) | Mali-G720-Immortalis if ≥10 cores and ray intersection; Mali-G720 if ≥6 cores; otherwise Mali-G620 |
| (12, 1) | Mali-G620 |
| (13, 0) | Mali-G925-Immortalis if ≥10 cores and ray intersection; Mali-G725 if ≥6 cores; otherwise Mali-G625 |
| (13, 1) | Mali-G625 |
| (14, 0) | Mali-G1-Ultra |
| (14, 1) | Mali-G1-Premium |
| (14, 3) | Mali-G1-Pro |

"Cores" is the population count of the shader-present mask. The names only
matter for logging. The hardware-relevant split is: **arch 10–13 use the
legacy GPU-control power registers. Arch 14 uses the PWR block.**

### 2.3.2 GPU interrupt bits (block at `0x020`)

| Bit | Name | Meaning |
|---|---|---|
| 0 | `FAULT` | GPU fault (see `GPU_FAULT_STATUS` and `GPU_FAULT_ADDR`) |
| 1 | `PROTM_FAULT` | Fault in protected mode |
| 8 | `RESET_COMPLETED` | Soft or hard reset finished |
| 9 | `POWER_CHANGED` | A single power transition completed |
| 10 | `POWER_CHANGED_ALL` | All pending power transitions completed |
| 17 | `CLEAN_CACHES_COMPLETED` | `FLUSH_CACHES` command finished |
| 18 | `DOORBELL_MIRROR` | Doorbell mirror |
| 19 | `MCU_STATUS_CHANGED` | MCU status changed |

### 2.3.3 `GPU_CMD` encoding

Bits [7:0] hold the command type. Bits [31:8] hold the payload.

| Command | Type | Payload | Value |
|---|---|---|---|
| Soft reset | 1 | 1 | `0x101` |
| Hard reset | 1 | 2 | `0x201` |
| Flush caches | 4 | see below | `0x04 \| payload<<8` |

Flush-caches payload: three 4-bit fields, each holding bit 0 = clean (write
back) and bit 1 = invalidate:
- payload bits [3:0]: L2 cache
- payload bits [7:4]: load/store cache (LSC)
- payload bits [11:8]: other caches

Example: clean and invalidate L2 and LSC, leave others alone =
`0x04 | (0x033 << 8)` = `0x3304`.

Completion is signaled by `CLEAN_CACHES_COMPLETED` (flush) or
`RESET_COMPLETED` (reset) in the GPU interrupt block.

### 2.3.4 `GPU_STATUS` bits

| Bit | Meaning |
|---|---|
| 0 | GPU active |
| 1 | Power active |
| 4 | Page fault pending |
| 7 | Protected mode active |
| 8 | Debug enabled |

## 2.4 MCU control block (base `0x0700`)

| Offset | Name | Values |
|---|---|---|
| `0x700` | `MCU_CONTROL` | 0 = disable, 1 = enable, 2 = auto |
| `0x704` | `MCU_STATUS` | 0 = disabled, 1 = enabled, 2 = halted, 3 = fatal |

## 2.5 PWR control block (base `0x0800`, arch 14+ only)

| Offset (absolute) | Name | Width | Description |
|---|---|---|---|
| `0x800` | PWR `INT_*` | 4×32 | Interrupt block, see 2.5.1 |
| `0x820` | `PWR_STATUS` | 64 | See 2.5.2 |
| `0x828` | `PWR_COMMAND` | 32 | See 2.5.3 |
| `0x830` | `PWR_CMDARG` | 64 | Command argument (core mask) |
| `0x900` | `L2_PRESENT` | 64 | |
| `0x908` | `L2_READY` | 64 | |
| `0x910` | `L2_PWRTRANS` | 64 | |
| `0x918` | `L2_PWRACTIVE` | 64 | |
| `0x940` | `TILER_PRESENT` | 64 | |
| `0x948` | `TILER_READY` | 64 | |
| `0x950` | `TILER_PWRTRANS` | 64 | |
| `0x958` | `TILER_PWRACTIVE` | 64 | |
| `0xA00` | `SHADER_PRESENT` | 64 | |
| `0xA08` | `SHADER_READY` | 64 | |
| `0xA10` | `SHADER_PWRTRANS` | 64 | |
| `0xA18` | `SHADER_PWRACTIVE` | 64 | |
| `0xB80` | `BASE_PRESENT` | 64 | |
| `0xB88` | `BASE_READY` | 64 | |
| `0xB90` | `BASE_PWRTRANS` | 64 | |
| `0xB98` | `BASE_PWRACTIVE` | 64 | |
| `0xBC0` | `STACK_PRESENT` | 64 | |
| `0xBC8` | `STACK_READY` | 64 | |
| `0xBD0` | `STACK_PWRTRANS` | 64 | |

### 2.5.1 PWR interrupt bits

| Bit | Meaning |
|---|---|
| 0 | Power changed (single) |
| 1 | Power changed (all) |
| 2 | Delegation changed |
| 3 | Reset completed |
| 4 | Retract completed |
| 5 | Inspect completed |
| 30 | Command not allowed |
| 31 | Command invalid |

### 2.5.2 `PWR_STATUS` bits (64-bit)

| Bit(s) | Meaning |
|---|---|
| 0, 1, 8, 14, 15 | Host is *allowed* to control domain L2, TILER, SHADER, BASE, STACK (bit = domain number) |
| 16, 17, 24, 30, 31 | Domain is *delegated* (bit = 16 + domain number) |
| 33 | Soft reset allowed |
| 34 | Fast reset allowed |
| 41 | Power command pending |
| 42 | Reset pending |
| 43 | Retract pending |
| 44 | Inspect pending |

### 2.5.3 `PWR_COMMAND` encoding

`value = (subdomain << 16) | (domain << 8) | command`

| Command | Code | | Domain | Number |
|---|---|---|---|---|
| Power up | `0x10` | | L2 | 0 |
| Power down | `0x11` | | TILER | 1 |
| Delegate | `0x20` | | SHADER | 8 |
| Retract | `0x21` | | BASE | 14 |
| Soft reset | `0x31` | | STACK | 15 |
| Fast reset | `0x32` | | | |
| Inspect | `0xF0` | | | |

Subdomain bit 0 = RTU (ray tracing unit).

## 2.6 JOB interrupt block (base `0x1000`)

Firmware raises these bits to notify the host.

| Bit | Meaning |
|---|---|
| 31 | Global interface has a request or acknowledge to process |
| 0..N-1 | CSG (command stream group) slot *n* has a request or acknowledge to process |

During boot, the host waits for bit 31 to learn that the firmware has started.

## 2.7 MMU

### 2.7.1 MMU interrupt block (base `0x2000`)

| Bits | Meaning |
|---|---|
| [15:0] | Bit *n* = page or bus fault on address space *n* |
| [31:16] | Bit 16+*n* = command complete on address space *n*. **[POLICY]** Linux masks these off and polls `AS_STATUS` instead |

### 2.7.2 Address-space slot registers

Slot *n* base = `0x2400 + n × 0x40`.

| Offset | Name | Width | Description |
|---|---|---|---|
| `+0x00` | `AS_TRANSTAB` | 64 | Physical address of the top-level translation table |
| `+0x08` | `AS_MEMATTR` | 64 | Eight 8-bit memory-attribute entries, indexed by the page-table AttrIndx field (2.7.3) |
| `+0x10` | `AS_LOCKADDR` | 64 | Lock region: see 04-mmu |
| `+0x18` | `AS_COMMAND` | 32 | 0 NOP, 1 UPDATE, 2 LOCK, 3 UNLOCK, 4 FLUSH_PT, 5 FLUSH_MEM |
| `+0x1C` | `AS_FAULTSTATUS` | 32 | [7:0] exception type (2.8), [9:8] access type (0 atomic, 1 execute, 2 read, 3 write), bit 10 fault origin (1 = decoder fault, 0 = slave fault; informational), [31:16] source ID |
| `+0x20` | `AS_FAULTADDRESS` | 64 | Faulting GPU virtual address |
| `+0x28` | `AS_STATUS` | 32 | Bit 0 = AS active (a command is still running) |
| `+0x30` | `AS_TRANSCFG` | 64 | Translation configuration (2.7.4) |
| `+0x38` | `AS_FAULTEXTRA` | 64 | Extra fault information |

The minimum lock-region size is 32 KiB (2^15).

### 2.7.3 `AS_MEMATTR` per-entry encoding (AArch64 mode)

Each 8-bit entry:

| Bits | Field | Values |
|---|---|---|
| [1:0] | Explicit allocation hints | bit 0 = write-allocate, bit 1 = read-allocate (used when [3:2] = 3) |
| [3:2] | Inner allocation policy | 2 = implementation-defined, 3 = explicit (use [1:0]) |
| [5:4] | Shareability | 0 = Midgard inner domain, 1 = CPU inner domain, 2 = CPU inner + shader coherent |
| [7:6] | Memory type | 0 = shared, 1 = inner/outer non-cacheable, 2 = inner/outer write-back, 3 = fault |

### 2.7.4 `AS_TRANSCFG`

| Bits | Field | Values |
|---|---|---|
| [3:0] | ADRMODE | 1 = unmapped (translation disabled, all accesses fault), 2 = identity, 6 = AArch64 4 KiB granule, 8 = AArch64 64 KiB granule |
| [10:6] | INA_BITS | Input address size, encoded as (55 − VA bits). For example, 48-bit VA gives 7 |
| [18:14] | OUTA_BITS | Output address size |
| 22 | SL_CONCAT | Start-level concatenation |
| [25:24] | PTW_MEMATTR | Page-table walk cacheability: 1 = non-cacheable, 2 = write-back |
| [29:28] | PTW_SH | Page-table walk shareability: 0 = non-shareable, 2 = outer, 3 = inner |
| 30 | PTW_RA | Page-table walk read-allocate |
| 33 | DISABLE_HIER_AP | Disable hierarchical access permissions |
| 34 | DISABLE_AF_FAULT | Do not fault on Access Flag = 0 |
| 35 | WXN | Write implies execute-never |
| 36 | XREADABLE | Execute permission implies read |

## 2.8 Exception (fault) codes

These codes appear in `AS_FAULTSTATUS[7:0]` and in firmware-reported CS and
CSG fault fields. Codes ≤ `0x3F` are not faults (status or termination). Codes
> `0x3F` are faults.

| Code | Name | | Code | Name |
|---|---|---|---|---|
| 0x00 | OK | | 0x80 | GPU_BUS_FAULT |
| 0x04 | TERMINATED | | 0x88 | GPU_SHAREABILITY_FAULT |
| 0x05 | KABOOM | | 0x89 | SYSTEM_SHAREABILITY_FAULT |
| 0x06 | EUREKA | | 0x8A | GPU_CACHEABILITY_FAULT |
| 0x08 | ACTIVE | | 0xC0–0xC4 | TRANSLATION_FAULT level 0–4 |
| 0x0F | CS_RESOURCE_TERMINATED | | 0xC8–0xCB | PERMISSION_FAULT level 0–3 |
| 0x40 | CS_CONFIG_FAULT | | 0xD9–0xDB | ACCESS_FLAG_FAULT level 1–3 |
| 0x41 | CS_UNRECOVERABLE | | 0xE0 | ADDRESS_SIZE_FAULT (input) |
| 0x44 | CS_ENDPOINT_FAULT | | 0xE4–0xE7 | ADDRESS_SIZE_FAULT (output) level 0–3 |
| 0x48 | CS_BUS_FAULT | | 0xE8–0xEB | MEMORY_ATTRIBUTE_FAULT level 0–3 |
| 0x49 | CS_INSTR_INVALID | | | |
| 0x4A | CS_CALL_STACK_OVERFLOW | | | |
| 0x4B | CS_INHERIT_FAULT | | | |
| 0x50 | INSTR_INVALID_PC | | | |
| 0x51 | INSTR_INVALID_ENC | | | |
| 0x55 | INSTR_BARRIER_FAULT | | | |
| 0x58 | DATA_INVALID_FAULT | | | |
| 0x59 | TILE_RANGE_FAULT | | | |
| 0x5A | ADDR_RANGE_FAULT | | | |
| 0x5B | IMPRECISE_FAULT | | | |
| 0x60 | OUT_OF_MEMORY | | | |
| 0x68 | CSF_FW_INTERNAL_ERROR | | | |
| 0x69 | CSF_RES_EVICTION_TIMEOUT | | | |

## 2.9 Doorbells and the flush-ID page

- **[HW]** Doorbell *i* is rung by writing `1` to offset `0x80000 + i × 0x10000`.
  Doorbell 0 belongs to the firmware global interface. Other doorbells are
  assigned to command-stream groups or queues (06-csf-interface).
- **[HW]** No register reports the doorbell count. The usable count is limited
  by the size of the MMIO window. On RK3588 (2 MiB window) that is doorbells
  0–23. **[POLICY]** Linux uses doorbell 0 for the global interface and
  doorbell *n*+1 for CSG slot *n* (07-scheduler), so it needs at most
  1 + (number of CSG slots) doorbells.

> OPEN QUESTION: whether the hardware implements doorbell pages beyond those a
> driver actually uses, or the firmware advertises a limit.
- **[HW]** Offset `0x10000` holds `LATEST_FLUSH_ID`, a counter of completed
  cache flushes. Its 4 KiB page can be mapped read-only and uncached into
  userspace. **[ABI]** Mesa expects this mapping (10-uapi).
- **[POLICY]** While the GPU is powered down, the register page cannot be read.
  Linux then backs the userspace mapping with a dummy page that holds the value 1
  (09-memory).
