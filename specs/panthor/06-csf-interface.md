# 06: CSF Host↔Firmware Interface (Global, CSG and CS Interfaces)

This section defines the shared-memory interface between the host driver and
the CSF firmware running on the MCU. It covers block layouts, every known field
and bit, the request/acknowledge protocol, doorbells and the ring-buffer
interface. Loading the firmware image and booting the MCU are in
**05-firmware**. Choosing which groups occupy which slots, and when, is in
**07-scheduler**. Tiler heap chunk management is in **08-heap**. The
userspace-visible numbers derived from this interface are in **10-uapi**.

All offsets in the tables are byte offsets from the start of the block named
in the table heading. All fields are little-endian. Every offset and width
below is a **[HW]** firmware-ABI fact. For fields marked "unused by Linux", the
meaning column is inferred from the field's name only and has not been checked
against behavior.

## 6.1 Overview and terminology

- **[HW]** The interface has three levels:
  - **Global (GLB)**: one instance. Firmware-wide configuration, timers,
    watchdog ping, halt, and the global idle event.
  - **Command Stream Group (CSG)**: one instance per *CSG slot*. A slot holds
    one resident group of command streams that share an MMU address space,
    a priority and a suspend buffer.
  - **Command Stream (CS)**: one instance per CS slot inside each CSG slot.
    Each CS executes one ring buffer.
- **[HW]** Each level has three blocks:
  - **Control** block: read-only to the host. Firmware fills it during boot
    with version, features, counts, strides and sizes, and with the MCU
    virtual addresses of the input and output blocks.
  - **Input** block: written by the host and read by firmware.
  - **Output** block: written by firmware and read by the host. The host must
    never write it.
- **[HW]** Every control, input and output block lives inside the firmware's
  *shared interface section*. This is the firmware image section whose MCU
  virtual address is `0x04000000`, the start of the MCU shared region
  (`0x04000000`–`0x07FFFFFF`). See 05-firmware for how that section is found,
  mapped and zeroed. The host needs a CPU mapping of the whole section.
- **[HW]** "MCU VA" means an address in the MCU's own address space (MMU
  address space 0, set up in 05-firmware). Values in control blocks that point
  at input and output blocks are 32-bit MCU VAs.

## 6.2 Locating the blocks

### 6.2.1 Layout

| Block | Location |
|---|---|
| GLB control | Offset `0x0` of the shared section, which is MCU VA `0x04000000` |
| CSG *i* control | Shared-section offset `0x1000 + i × GLB_GROUP_STRIDE` |
| CS *j* control in CSG *i* | Shared-section offset `0x1000 + i × GLB_GROUP_STRIDE + 0x40 + j × CSG_STREAM_STRIDE` |
| Any input or output block | At the MCU VA held in the matching control block's `*_INPUT_VA` or `*_OUTPUT_VA` field |

- **[HW]** The constants `0x1000` (first CSG control block) and `0x40` (first
  CS control block within a CSG control block) are fixed by the firmware ABI.
- **[HW]** To convert an MCU VA *v* from a control block into a host pointer:
  `host_ptr = shared_section_host_base + (v − shared_section_mcu_base)`. The
  MCU base is `0x04000000`.
- **[POLICY]** Linux rejects (and fails firmware init) any input or output VA
  that is below the section base, or whose block would extend past the end of
  the section. A control block offset outside the section is rejected the same
  way. A clean-room driver should make the same bounds checks, because a
  corrupted or mismatched firmware could otherwise lead to wild host writes.

### 6.2.2 Discovery sequence

After the MCU has booted (the JOB interrupt global bit has fired, see
05-firmware):

1. Read `GLB_VERSION`. **[HW]** A value of 0 means the firmware did not boot
   or did not initialize the interface. Treat it as a boot failure.
2. Translate `GLB_INPUT_VA` and `GLB_OUTPUT_VA` into host pointers.
3. Read `GLB_GROUP_NUM`. **[HW]** At most 31 CSG slots can be used, because
   the JOB interrupt register has 31 per-CSG bits (bits 0–30, with bit 31 for
   the global interface; see 02-register-map §2.6). **[POLICY]** Linux
   refuses firmware reporting fewer than 3 or more than 31 groups.
4. For each CSG slot *i* < `GLB_GROUP_NUM`: locate its control block, translate
   its input and output VAs, and read `CSG_STREAM_NUM`. **[POLICY]** Linux
   requires 8 ≤ `CSG_STREAM_NUM` ≤ 32. The upper bound of 32 is a hard limit,
   because per-CS bit masks are 32 bits wide **[HW]**.
5. For each CS slot *j* < `CSG_STREAM_NUM`: locate its control block and
   translate its input and output VAs.
6. **[POLICY]** Linux requires every CSG slot to report identical
   `CSG_FEATURES`, `CSG_SUSPEND_SIZE`, `CSG_PROTM_SUSPEND_SIZE` and
   `CSG_STREAM_NUM`, and every CS slot to report identical `CS_FEATURES`. It
   refuses to run otherwise. Firmware seen so far meets this.

### 6.2.3 Interface version

`GLB_VERSION` encoding:

| Bits | Field |
|---|---|
| [31:24] | Major |
| [23:16] | Minor |
| [15:0] | Patch |

Version-dependent behavior (**[HW]**):

| Minimum version | Effect |
|---|---|
| 1.1.0 | `GLB_INSTR_FEATURES` (control `+0x1C`) is valid. Below 1.1.0 treat it as 0 |
| 4.0.0 | The CSG endpoint request moves to the 64-bit field `CSG_EP_REQ_64` at CSG input `+0x38`. The 32-bit `CSG_EP_REQ` at `+0x34` is not used |
| 4.1.0 | `GLB_REQ`/`GLB_ACK` bits [14:12] form the 3-bit `GLB_STATE` field (§6.4.2). Halt is requested via that field and not via `GLB_REQ.HALT` |

> OPEN QUESTION: Linux enforces no minimum interface version, only that it is
> non-zero. Mali-G610 (RK3588) firmware is believed to report a 1.x interface,
> but this is not stated anywhere in the source. A clean-room driver should log
> the version and support at least 1.0 through 4.1 as described here.

## 6.3 Request/acknowledge protocol

### 6.3.1 Word pairs

Communication uses pairs of 32-bit words: a *request* word in an input block,
written by one side, and an *acknowledge* word in the matching output block,
written by the other side. The pairs are:

| Request word (input) | Acknowledge word (output) | Bit *n* means |
|---|---|---|
| `GLB_REQ` | `GLB_ACK` | Global request or event bit (§6.4.2) |
| `GLB_DB_REQ` | `GLB_DB_ACK` | CSG slot *n* has new input to look at |
| `CSG_REQ` | `CSG_ACK` | CSG request or event bit (§6.5.2) |
| `CSG_DB_REQ` | `CSG_DB_ACK` | CS slot *n* in this CSG has new input to look at |
| `CS_REQ` | `CS_ACK` | CS request or event bit (§6.6.2) |

Plus one reversed pair, where firmware owns the request word:

| Request word (output, FW-written) | Acknowledge word (input, host-written) | Bit *n* means |
|---|---|---|
| `CSG_CS_IRQ_REQ` | `CSG_CS_IRQ_ACK` | CS slot *n* in this CSG has events pending |

### 6.3.2 Bit kinds

Each bit in a REQ/ACK pair is one of four kinds.

1. **Host-initiated toggle request.** **[HW]** The host asks for an action by
   setting the REQ bit to the *inverse* of the current ACK bit, so that REQ
   and ACK differ. Firmware performs the action, then makes ACK equal REQ. The
   request is complete when `(REQ XOR ACK) & bit == 0`. Re-issuing the same
   request means inverting again relative to ACK, never just writing 1.
   Examples: `GLB_REQ.PING`, `CSG_REQ.ENDPOINT_CONFIG`, every bit of
   `GLB_DB_REQ` and `CSG_DB_REQ`.
2. **Host-initiated state field.** **[HW]** The host writes the desired value
   (for example a CSG state code) into a multi-bit REQ field. Firmware copies
   the value into the same ACK field once the transition has finished. The
   request is complete when the whole field in ACK equals the field in REQ.
   **[HW]** A partial match, where only some bits of the field agree, is *not*
   completion. Examples: `CSG_REQ.STATE`, `CS_REQ.STATE`, `GLB_REQ.STATE`.
3. **Host-owned level/config bit.** The host writes 0 or 1 as a plain setting.
   Firmware may reflect it in ACK. Examples: `GLB_REQ.IDLE_EN`,
   `GLB_REQ.COUNTER_EN`, `CS_REQ.IDLE_*`. The pre-4.1 `GLB_REQ.HALT` is also
   this kind and is **not** acknowledged through `GLB_ACK` (§6.4.5).
4. **Firmware-initiated event.** **[HW]** Firmware signals an event by
   inverting the ACK bit, so that REQ and ACK differ. The host handles the
   event, then acknowledges it by copying the ACK bit's current value into REQ,
   so that they match again. Examples: `GLB_ACK.IDLE`, `CSG_ACK.SYNC_UPDATE`,
   `CS_ACK.FAULT`, `CS_ACK.TILER_OOM`.

**[HW]** In the reversed pair `CSG_CS_IRQ_REQ`/`CSG_CS_IRQ_ACK`, firmware
toggles bit *n* of `CSG_CS_IRQ_REQ` to say "CS *n* has events". The host
acknowledges by writing `CSG_CS_IRQ_ACK := CSG_CS_IRQ_REQ`, either the whole
word or just the bits it is handling.

### 6.3.3 Updating a REQ word safely

- **[HW]** A REQ word holds bits of all four kinds, so every update must be a
  read-modify-write that changes only the intended bits. To toggle mask *M*:
  `REQ_new = (NOT ACK_now AND M) OR (REQ_old AND NOT M)`. To set a field or
  level bit to *V* under mask *M*: `REQ_new = (V AND M) OR (REQ_old AND NOT M)`.
  To acknowledge events *E*: `REQ_new = (ACK_now AND E) OR (REQ_old AND NOT E)`.
- **[POLICY]** The host may update the same REQ word from both interrupt
  context and thread context. Linux serializes all writers of each REQ word
  with one lock per interface instance (one for GLB, one per CSG slot and one
  per CS slot). Updates that must cover several words atomically need a wider
  lock.
- **[HW]** Linux maps the shared section uncached or write-combined on the
  CPU. Atomic compare-and-swap on such a mapping can cause an external abort
  (SError) on Arm64 CPUs. Use a lock and plain 32-bit loads and stores, not
  atomics on the shared section.
- **[HW]** Fields must be accessed with single loads and stores of their
  natural width (32 or 64 bits). The compiler must not tear, merge or cache
  them, so treat them as volatile.

### 6.3.4 Waiting for acknowledgement

- **[HW]** For a set of host-initiated bits *M*, the request is complete when
  `(ACK AND M) == (REQ AND M)`. Snapshot REQ before waiting.
- **[HW]** Firmware raises a host interrupt when an ACK bit changes, but only
  for bits enabled in the corresponding `*_ACK_IRQ_MASK` word in the input
  block. For requests whose bits are not enabled there, the host must poll.
- **[POLICY]** Linux polls for about 10 µs, then sleeps until the JOB
  interrupt wakes it, then makes one last check. Timeouts: 100 ms for CSG
  requests and for ping. A CSG request timeout is treated as a firmware hang
  and triggers a GPU reset.

### 6.3.5 Doorbells

A doorbell is an MMIO write of the value `1` to `0x80000 + id × 0x10000`
(02-register-map §2.9).

| Doorbell | Use |
|---|---|
| 0 | Global doorbell. Makes firmware re-evaluate the GLB input block, `GLB_DB_REQ`, and through that any CSG input block |
| *k* ≥ 1 | CS (queue) doorbell. Assigned by the host through `CS_CONFIG.DOORBELL`. Makes firmware look at the ring buffers of all CS slots bound to *k* |

- **[HW]** Firmware does not watch input blocks for changes. It acts only
  when a doorbell is rung. After changing anything in an input block, the host
  must ring a doorbell.
- **[HW]** To make firmware look at a CSG slot's input (`CSG_REQ`, endpoint,
  CS changes), the host must (1) toggle bit *i* of `GLB_DB_REQ` for CSG slot
  *i* and (2) ring doorbell 0.
- **[HW]** To make firmware look at a CS slot's input (`CS_REQ`, heap
  pointers), the host must also toggle bit *j* of that CSG's `CSG_DB_REQ` for
  CS slot *j*, before the GLB step above.
- **[POLICY]** Linux gives every queue of a group the same doorbell,
  `CSG slot index + 1`. Doorbells 1..(number of CSG slots) are therefore used.

### 6.3.6 Ordering requirements

The full ordering for a CS-level change:

1. Write all CS input data fields.
2. Read-modify-write `CS_REQ`.
3. Toggle the CS bit in `CSG_DB_REQ`.
4. Toggle the CSG bit in `GLB_DB_REQ`.
5. Ring doorbell 0.

For a CSG-level change, skip steps 1–3 and write the CSG fields and `CSG_REQ`
instead.

- **[HW]** All shared-memory writes must be visible to the MCU before the
  doorbell MMIO write reaches the GPU. Use a doorbell write with device
  ordering against earlier normal-memory stores. On Arm64 this means a
  barrier-carrying (non-relaxed) MMIO write, or an explicit write barrier
  before a relaxed one.
- **[HW]** The ring-buffer contents must be visible before the `INSERT`
  pointer update (§6.7.3). Put an explicit write memory barrier between them.
  The `INSERT` store must in turn be ordered before the CS doorbell write.

### 6.3.7 Interrupt delivery and handling order

- **[HW]** Firmware notifies the host through the JOB interrupt block (base
  `0x1000`, 02-register-map §2.6). Bit 31 means "look at the GLB interface".
  Bit *i* (0–30) means "look at CSG slot *i*".
- **[HW]** Clear the JOB interrupt bits (write them to `INT_CLEAR`)
  *before* inspecting the interfaces. An event that arrives during handling
  then raises the interrupt again instead of being lost.
- **[HW]** For a CSG slot interrupt, the safe processing order is:
  1. Read `CSG_REQ`, `CSG_ACK`, `CSG_CS_IRQ_REQ` and `CSG_CS_IRQ_ACK`. If
     `REQ == ACK` and `CS_IRQ_REQ == CS_IRQ_ACK`, there is nothing to do.
  2. Write `CSG_CS_IRQ_ACK := CSG_CS_IRQ_REQ` (the value just read) *before*
     looking at any CS REQ/ACK. A CS event that firmware raises after this
     point toggles `CS_IRQ_REQ` again and is not missed.
  3. Acknowledge the CSG event bits (`SYNC_UPDATE`, `IDLE`,
     `PROGRESS_TIMER_EVENT`) by copying their ACK values into `CSG_REQ`.
     Then handle them.
  4. For every CS *j* whose bit differs between the `CS_IRQ_REQ` and
     `CS_IRQ_ACK` values read in step 1, compute
     `(CS_REQ XOR CS_ACK) AND {FATAL, FAULT, TILER_OOM, PROTM_PENDING}` and
     handle each event (§6.6.5).
  5. Ring the CSG: toggle any `CSG_DB_REQ` bits needed (for example for a
     CS whose fault was acknowledged), toggle the CSG bit in `GLB_DB_REQ`, and
     ring doorbell 0.
- **[POLICY]** Linux rings the CSG doorbell (step 5) at the end of every CSG
  interrupt, even when only events were acknowledged.

> OPEN QUESTION: It is not proven whether firmware needs a doorbell to notice
> that an event was acknowledged, or only to notice new requests. Ringing
> every time, as Linux does, is safe.

## 6.4 Global interface

### 6.4.1 GLB control block (shared-section offset `0x0`, read-only)

| Offset | Width | Name | Meaning |
|---|---|---|---|
| `0x00` | 32 | `GLB_VERSION` | Interface version (§6.2.3). 0 = firmware not booted |
| `0x04` | 32 | `GLB_FEATURES` | Global feature bits. Linux only logs them |
| `0x08` | 32 | `GLB_INPUT_VA` | MCU VA of the GLB input block |
| `0x0C` | 32 | `GLB_OUTPUT_VA` | MCU VA of the GLB output block |
| `0x10` | 32 | `GLB_GROUP_NUM` | Number of CSG slots |
| `0x14` | 32 | `GLB_GROUP_STRIDE` | Byte stride between CSG control blocks |
| `0x18` | 32 | `GLB_PRFCNT_SIZE` | Performance-counter buffer size information (unused by Linux) |
| `0x1C` | 32 | `GLB_INSTR_FEATURES` | Instrumentation features. Valid only from interface 1.1.0. Linux only logs it |

Block size: `0x20` bytes.

> OPEN QUESTION: The bit layouts of `GLB_FEATURES`, `GLB_PRFCNT_SIZE` and
> `GLB_INSTR_FEATURES` are not decoded in the source. An event-wait or
> instrumentation feature encoding, if one exists, is not present.

### 6.4.2 `GLB_REQ` / `GLB_ACK` bits

| Bit(s) | Name | Kind (§6.3.2) | Meaning |
|---|---|---|---|
| 0 | `HALT` | Level (pre-4.1) | 1 = ask the MCU to halt. Not acknowledged via `GLB_ACK`. See §6.4.5. Superseded by `STATE` from 4.1 |
| 1 | `CFG_PROGRESS_TIMER` | Toggle | Apply `GLB_PROGRESS_TIMER` |
| 2 | `CFG_ALLOC_EN` | Toggle | Apply `GLB_ALLOC_EN` (shader core enable mask) |
| 3 | `CFG_PWROFF_TIMER` | Toggle | Apply `GLB_PWROFF_TIMER` |
| 4 | `PROTM_ENTER` | Toggle | Request entry into protected mode (unused by Linux) |
| 5 | `PRFCNT_ENABLE` | Level | Enable performance-counter collection (unused by Linux) |
| 6 | `PRFCNT_SAMPLE` | Toggle | Take a performance-counter sample (unused by Linux) |
| 7 | `COUNTER_ENABLE` | Level | Counter enable (meaning inferred from the name: GPU cycle and timestamp counters available to firmware). Linux sets 1 |
| 8 | `PING` | Toggle | Watchdog ping. Firmware just acknowledges (§6.4.6) |
| 9 | `FIRMWARE_CONFIG_UPDATE` | Toggle | Apply firmware configuration-entry changes (unused by Linux) |
| 10 | `IDLE_ENABLE` | Level | Enable global idle detection using `GLB_IDLE_TIMER`. Linux sets 1 |
| 11 | – | – | Reserved |
| 12 | `SLEEP` | – | Pre-4.1 name for this bit. Unused by Linux |
| [14:12] | `STATE` | State field (≥ 4.1) | 0 = ACTIVE, 1 = HALT, 2 = SLEEP, 3 = SUSPEND |
| [19:15] | – | – | Reserved |
| 20 | `INACTIVE_COMPUTE` | FW event | Compute endpoints inactive (unused by Linux) |
| 21 | `INACTIVE_FRAGMENT` | FW event | Fragment endpoints inactive (unused) |
| 22 | `INACTIVE_TILER` | FW event | Tiler endpoints inactive (unused) |
| 23 | `PROTM_EXIT` | FW event | GPU left protected mode (unused) |
| 24 | `PRFCNT_THRESHOLD` | FW event | Performance-counter buffer reached its threshold (unused) |
| 25 | `PRFCNT_OVERFLOW` | FW event | Performance-counter buffer overflowed (unused) |
| 26 | `IDLE` | FW event | GPU has been idle for the `GLB_IDLE_TIMER` period |
| [29:27] | – | – | Reserved |
| 30 | `DEBUG_CSF` | – | Debug request from firmware (unused) |
| 31 | `DEBUG_HOST` | – | Debug request from host (unused) |

Groupings: bits [10:0] are host-initiated request bits, and bits [26:20] are
firmware-initiated events.

> OPEN QUESTION: Whether `IDLE_ENABLE`, `COUNTER_ENABLE` and `PRFCNT_ENABLE`
> are pure level bits, or level bits that firmware also echoes into `GLB_ACK`,
> is not proven. Linux writes them as levels (sets 1 regardless of ACK), yet
> also enables `IDLE_ENABLE` in `GLB_ACK_IRQ_MASK`, which suggests firmware
> mirrors it into ACK. The same bits in REQ and ACK define the "done" state for
> the 4.1+ `STATE` field. Whether `SLEEP` (bit 12) had any meaning before 4.1 is
> unknown.

### 6.4.3 GLB input block (host → FW)

| Offset | Width | Name | Meaning |
|---|---|---|---|
| `0x00` | 32 | `GLB_REQ` | Request word (§6.4.2) |
| `0x04` | 32 | `GLB_ACK_IRQ_MASK` | Bit *n* = 1: a change of `GLB_ACK` bit *n* raises JOB interrupt bit 31 |
| `0x08` | 32 | `GLB_DB_REQ` | Toggle bit *i* to make firmware re-read CSG slot *i* |
| `0x0C` | 32 | – | Reserved |
| `0x10` | 32 | `GLB_PROGRESS_TIMER` | Progress timeout (§6.4.4) |
| `0x14` | 32 | `GLB_PWROFF_TIMER` | Shader-core power-off hysteresis (§6.4.4) |
| `0x18` | 64 | `GLB_ALLOC_EN` | Bitmask of shader cores firmware may power and use. Applied by `CFG_ALLOC_EN` |
| `0x20` | 32 | – | Reserved |
| `0x24` | 32 | `GLB_PRFCNT_AS` | MMU address space of the perf-counter buffer (unused) |
| `0x28` | 64 | `GLB_PRFCNT_BASE` | GPU VA of the perf-counter ring buffer (unused) |
| `0x30` | 32 | `GLB_PRFCNT_EXTRACT` | Host read index into the perf-counter buffer (unused) |
| `0x34`–`0x3F` | 3×32 | – | Reserved |
| `0x40` | 32 | `GLB_PRFCNT_CONFIG` | Perf-counter configuration (unused) |
| `0x44` | 32 | `GLB_PRFCNT_CSG_SELECT` | CSG selection for perf counters (unused) |
| `0x48` | 32 | `GLB_PRFCNT_FW_EN` | Firmware counter enable mask (unused) |
| `0x4C` | 32 | `GLB_PRFCNT_CSG_EN` | CSG counter enable mask (unused) |
| `0x50` | 32 | `GLB_PRFCNT_CSF_EN` | CSF counter enable mask (unused) |
| `0x54` | 32 | `GLB_PRFCNT_SHADER_EN` | Shader counter enable mask (unused) |
| `0x58` | 32 | `GLB_PRFCNT_TILER_EN` | Tiler counter enable mask (unused) |
| `0x5C` | 32 | `GLB_PRFCNT_MMU_L2_EN` | MMU/L2 counter enable mask (unused) |
| `0x60`–`0x7F` | 8×32 | – | Reserved |
| `0x80` | 32 | `GLB_IDLE_TIMER` | Global idle hysteresis (§6.4.4) |

Minimum block size the host relies on: `0x84` bytes.

> OPEN QUESTION: The encodings of the perf-counter fields are not given in the
> source. Linux does not use firmware perf counters.

### 6.4.4 Timer encodings

**`GLB_PWROFF_TIMER` and `GLB_IDLE_TIMER`** (**[HW]**):

| Bits | Field | Meaning |
|---|---|---|
| [30:0] | `TIMEOUT` | Timeout in units of **1024 ticks** of the selected source |
| 31 | `TIMER_SOURCE` | 0 = system timestamp (the GPU timestamp counter, which runs at the SoC system-timer frequency, the same as the Arm generic timer `CNTFRQ`). 1 = GPU cycle counter (GPU core clock) |

Encoding a timeout of *t* µs with source frequency *f* Hz:
`TIMEOUT = ceil(t × f / (1 000 000 × 1024))`, clamped to `0x7FFFFFFF`.

- **[POLICY]** Linux uses the system timestamp source when the CPU's generic
  timer frequency is known, and otherwise the GPU cycle counter at the current
  core clock rate. If neither rate is known, it writes `0xFFFFFFFF` (maximum
  value, cycle-counter source) so firmware still manages core power.
- **[POLICY]** Linux values: power-off hysteresis 10 000 µs, idle hysteresis
  800 µs. With the 24 MHz system timer on RK3588 this gives `TIMEOUT` = 235
  (`0x000000EB`) and 19 (`0x00000013`), both with bit 31 clear.
- **[HW]** `GLB_PWROFF_TIMER` takes effect when `CFG_PWROFF_TIMER` is
  toggled. `GLB_IDLE_TIMER` is used while `IDLE_ENABLE` is set.

> OPEN QUESTION: Whether changing `GLB_IDLE_TIMER` while `IDLE_ENABLE` is
> already 1 takes effect without clearing and setting `IDLE_ENABLE` again.
> Linux only writes it before setting `IDLE_ENABLE`.

**`GLB_PROGRESS_TIMER`** (**[HW]**): a plain 32-bit count, in units of **1024
GPU cycles**, with no source bit. If a resident CSG's streams make no forward
progress for this long, firmware raises `CSG_ACK.PROGRESS_TIMER_EVENT` for
that slot. It takes effect when `CFG_PROGRESS_TIMER` is toggled.

- **[POLICY]** Linux writes `2 560 000` (`0x00271000`), that is
  5 × 500 × 2^20 cycles ÷ 1024, about 5 s at 500 MHz.

> OPEN QUESTION: Whether a value of 0 disables the progress timer, and exactly
> which clock "GPU cycles" refers to here (core clock assumed).

### 6.4.5 Halt and state transitions

- **[HW]** **Interface < 4.1:** to halt, set `GLB_REQ.HALT` = 1 and ring
  doorbell 0. Completion is not acknowledged via `GLB_ACK`. The host polls
  `MCU_STATUS` (02-register-map §2.4) until it reads 2 (HALT). To leave halt
  before the MCU is restarted, clear `GLB_REQ.HALT` to 0.
- **[HW]** **Interface ≥ 4.1:** to halt, write `GLB_REQ.STATE` = 1 (HALT)
  and ring doorbell 0. Halt is complete when *both* `MCU_STATUS` = 2 *and*
  `GLB_ACK.STATE` = 1. To return to active before restarting the MCU, write
  `GLB_REQ.STATE` = 0 (ACTIVE).
- **[HW]** A cleanly halted MCU can be restarted without reloading the
  firmware's writable sections ("fast reset", see 05-firmware). In that case
  the host must set the halt request back to ACTIVE *before* re-enabling the
  MCU, or the firmware halts again straight away.
- **[POLICY]** Linux waits up to 1 s for the halt.

`GLB_HALT_STATUS` (GLB output `+0x10`) values:

| Value | Meaning |
|---|---|
| `0x00` | Halted normally |
| `0x4E` | Halted because of a firmware panic |
| `0x4F` | Halted because the firmware watchdog expired |

> OPEN QUESTION: When `GLB_HALT_STATUS` is valid (after any halt, or only
> after an MCU-initiated halt) is not shown. Linux never reads it.

### 6.4.6 Global initialization (after every MCU boot)

**[HW]** The global input settings must be programmed after *every* MCU start,
including a fast reset:

1. Write `GLB_ALLOC_EN` = the full shader-present mask (from
   `SHADER_PRESENT`, or from the PWR block on arch 14+).
2. Write `GLB_PWROFF_TIMER`, `GLB_PROGRESS_TIMER` and `GLB_IDLE_TIMER`
   (§6.4.4).
3. Write `GLB_ACK_IRQ_MASK`. **[POLICY]** Linux enables `CFG_ALLOC_EN`,
   `PING`, `CFG_PROGRESS_TIMER`, `CFG_PWROFF_TIMER`, `IDLE_ENABLE` and `IDLE`,
   plus the `STATE` field bits [14:12] on interface ≥ 4.1. That is
   bits 1, 2, 3, 8, 10 and 26 = `0x0400050E`, or `0x0400750E` with `STATE`.
4. In `GLB_REQ`, set the level bits `IDLE_ENABLE` = 1 and `COUNTER_ENABLE` =
   1, and toggle `CFG_ALLOC_EN`, `CFG_PWROFF_TIMER` and `CFG_PROGRESS_TIMER`.
5. Ring doorbell 0.

- **[POLICY]** Linux does not wait for the configuration acks.

### 6.4.7 Ping / watchdog

- **[HW]** Toggling `GLB_REQ.PING` and ringing doorbell 0 makes a live
  firmware toggle `GLB_ACK.PING` to match. No other side effect.
- **[POLICY]** Linux pings every 12 s and waits 100 ms for the ack. A missed
  ack is treated as a firmware hang and triggers a full GPU reset.

> OPEN QUESTION: Whether the firmware's own watchdog (halt status `0x4F`)
> depends on host pings is not shown.

### 6.4.8 Global idle event

- **[HW]** With `IDLE_ENABLE` = 1, firmware toggles `GLB_ACK.IDLE` (bit 26)
  when the GPU has had no work for the `GLB_IDLE_TIMER` period. The host
  acknowledges it by copying `GLB_ACK.IDLE` into `GLB_REQ.IDLE`.
- **[POLICY]** Linux responds by running its scheduler right away, which may
  suspend groups and let the device runtime-suspend (07-scheduler).

### 6.4.9 GLB output block (FW → host)

| Offset | Width | Name | Meaning |
|---|---|---|---|
| `0x00` | 32 | `GLB_ACK` | Acknowledge word (§6.4.2) |
| `0x04` | 32 | – | Reserved |
| `0x08` | 32 | `GLB_DB_ACK` | Bit *i* matches `GLB_DB_REQ` bit *i* once firmware has seen the CSG *i* doorbell |
| `0x0C` | 32 | – | Reserved |
| `0x10` | 32 | `GLB_HALT_STATUS` | Halt reason (§6.4.5) |
| `0x14` | 32 | `GLB_PRFCNT_STATUS` | Perf-counter status (unused) |
| `0x18` | 32 | `GLB_PRFCNT_INSERT` | Firmware write index into the perf-counter buffer (unused) |

Block size: `0x1C` bytes.

## 6.5 Command Stream Group (CSG) interface

### 6.5.1 CSG control block (shared-section offset `0x1000 + i × GLB_GROUP_STRIDE`, read-only)

| Offset | Width | Name | Meaning |
|---|---|---|---|
| `0x00` | 32 | `CSG_FEATURES` | CSG feature bits (not decoded by Linux) |
| `0x04` | 32 | `CSG_INPUT_VA` | MCU VA of this slot's CSG input block |
| `0x08` | 32 | `CSG_OUTPUT_VA` | MCU VA of this slot's CSG output block |
| `0x0C` | 32 | `CSG_SUSPEND_SIZE` | Bytes needed for a normal suspend buffer |
| `0x10` | 32 | `CSG_PROTM_SUSPEND_SIZE` | Bytes needed for a protected-mode suspend buffer |
| `0x14` | 32 | `CSG_STREAM_NUM` | Number of CS slots in this CSG slot |
| `0x18` | 32 | `CSG_STREAM_STRIDE` | Byte stride between CS control blocks |

The CS control blocks for this slot start at `+0x40` from this block.

### 6.5.2 `CSG_REQ` / `CSG_ACK` bits

| Bit(s) | Name | Kind | Meaning |
|---|---|---|---|
| [2:0] | `STATE` | State field | 0 = TERMINATE, 1 = START, 2 = SUSPEND, 3 = RESUME. Other values are invalid |
| 3 | – | – | Reserved |
| 4 | `ENDPOINT_CONFIG` | Toggle | Apply the endpoint request (`CSG_EP_REQ`), which includes priority and core counts |
| 5 | `STATUS_UPDATE` | Toggle | Ask firmware to refresh every `CS_STATUS_*` output field of this group |
| [27:6] | – | – | Reserved |
| 28 | `SYNC_UPDATE` | FW event | A stream in this group updated a synchronization object (job completion signal) |
| 29 | `IDLE` | FW event | All streams in this group are idle |
| 30 | `DOORBELL` | – | Defined but unused by Linux |
| 31 | `PROGRESS_TIMER_EVENT` | FW event | A stream in this group exceeded the progress timeout |

Groupings: bits [5:0] are host-initiated requests, and bits 28, 29 and 31 are
firmware events.

**State semantics** (**[HW]**):

| Written to `CSG_REQ.STATE` | Firmware action | `CSG_ACK.STATE` when done |
|---|---|---|
| START (1) | Start the group from scratch using the current CSG and CS inputs | 1 |
| RESUME (3) | Restore the group from `CSG_SUSPEND_BUF` and continue | 3 |
| SUSPEND (2) | Stop all streams, save state to `CSG_SUSPEND_BUF` and free the slot | 2 |
| TERMINATE (0) | Stop all streams and discard their state, freeing the slot | 0 |

- **[HW]** `CSG_ACK.STATE` of 1 or 3 means the group is running. 2 means
  suspended (contents of the suspend buffer are valid). 0 means terminated.
- **[POLICY]** Linux treats any other ACK value (4–7) as firmware corruption,
  marks the group unusable and schedules a GPU reset.
- **[POLICY]** Linux does not modify a slot's CSG or CS input fields while a
  START or RESUME request for that slot is still unacknowledged. Doing the
  same is the safe choice, because it is not documented when firmware samples
  them.

> OPEN QUESTION: `CSG_ACK.DOORBELL` (bit 30) and `CSG_STATUS_STATE.IDLE` are
> defined but their exact use (for example as a doorbell-on-idle request) is
> not shown.

### 6.5.3 CSG input block (host → FW)

| Offset | Width | Name | Meaning |
|---|---|---|---|
| `0x00` | 32 | `CSG_REQ` | Request word (§6.5.2) |
| `0x04` | 32 | `CSG_ACK_IRQ_MASK` | Bit *n* = 1: a change of `CSG_ACK` bit *n* raises JOB interrupt bit *i*. **[POLICY]** Linux writes `0xFFFFFFFF` |
| `0x08` | 32 | `CSG_DB_REQ` | Toggle bit *j* to make firmware re-read CS slot *j* |
| `0x0C` | 32 | `CSG_CS_IRQ_ACK` | Host acknowledge of `CSG_CS_IRQ_REQ` (§6.3.2) |
| `0x10`–`0x1F` | 4×32 | – | Reserved |
| `0x20` | 64 | `CSG_ALLOW_COMPUTE` | Bitmask of shader cores this group may use for compute |
| `0x28` | 64 | `CSG_ALLOW_FRAGMENT` | Bitmask of shader cores this group may use for fragment |
| `0x30` | 32 | `CSG_ALLOW_OTHER` | Bitmask of tiler units this group may use |
| `0x34` | 32 | `CSG_EP_REQ` | Endpoint request, interface < 4.0 (§6.5.4) |
| `0x38` | 64 | `CSG_EP_REQ_64` | Endpoint request, interface ≥ 4.0 (§6.5.4) |
| `0x40` | 64 | `CSG_SUSPEND_BUF` | MCU VA of the normal suspend buffer (`CSG_SUSPEND_SIZE` bytes). 0 = none |
| `0x48` | 64 | `CSG_PROTM_SUSPEND_BUF` | MCU VA of the protected-mode suspend buffer (`CSG_PROTM_SUSPEND_SIZE` bytes). 0 = none |
| `0x50` | 32 | `CSG_CONFIG` | MMU address-space slot index used by every stream in this group |
| `0x54` | 32 | `CSG_ITER_TRACE_CONFIG` | Iterator trace configuration (unused by Linux) |

Size the host relies on: `0x58` bytes.

- **[HW]** The suspend buffers must be mapped in the MCU address space
  (address space 0), not in the group's address space. **[POLICY]** Linux
  allocates both for every group, mapped non-executable and not accessible to
  userspace.
- **[HW]** `CSG_CONFIG` must hold the index of an MMU AS slot that is bound to
  the group's page tables for as long as the group is resident. AS 0 belongs
  to the MCU (04-mmu).

> OPEN QUESTION: Whether `CSG_CONFIG` has fields beyond the AS index (Linux
> writes the plain index), and the meaning of `CSG_ITER_TRACE_CONFIG`.

### 6.5.4 Endpoint request encoding

The same bit layout is used in the low 32 bits of both endpoint request
forms:

| Bits | Field | Meaning |
|---|---|---|
| [7:0] | `COMPUTE_EP` | Maximum number of shader cores for compute |
| [15:8] | `FRAGMENT_EP` | Maximum number of shader cores for fragment |
| [19:16] | `TILER_EP` | Maximum number of tiler units |
| 20 | `EXCLUSIVE_COMPUTE` | Ask for exclusive use of compute endpoints (unused by Linux) |
| 21 | `EXCLUSIVE_FRAGMENT` | Ask for exclusive use of fragment endpoints (unused by Linux) |
| [27:22] | – | Reserved |
| [31:28] | `PRIORITY` | CSG priority, 0–15 |

- **[HW]** On interface ≥ 4.0, write the 64-bit field at `+0x38`. On older
  interfaces, write the 32-bit field at `+0x34`.
- **[HW]** Changes take effect when `CSG_REQ.ENDPOINT_CONFIG` is toggled (and
  the CSG doorbell is rung). They also take effect on START or RESUME.
- **[HW]** Higher `PRIORITY` values mean higher priority. Linux gives 15 to
  the most important resident group. (This direction is inferred from how
  Linux assigns priorities.)
- **[POLICY]** Firmware may deadlock if two resident groups have the same
  priority and compete for overlapping endpoints. Linux therefore gives every
  resident group a distinct priority and uses at most 16 CSG slots.
- **[ABI]** The core masks and maximum counts come from the group-creation
  ioctl (10-uapi).

> OPEN QUESTION: The meaning of bits [63:32] of the 4.0+ 64-bit endpoint
> request. Linux leaves them 0.

### 6.5.5 CSG output block (FW → host)

| Offset | Width | Name | Meaning |
|---|---|---|---|
| `0x00` | 32 | `CSG_ACK` | Acknowledge word (§6.5.2) |
| `0x04` | 32 | – | Reserved |
| `0x08` | 32 | `CSG_DB_ACK` | Bit *j* matches `CSG_DB_REQ` bit *j* once firmware has seen it |
| `0x0C` | 32 | `CSG_CS_IRQ_REQ` | Firmware toggles bit *j* when CS slot *j* has events |
| `0x10` | 32 | `CSG_STATUS_EP_CURRENT` | Endpoints currently allocated (unused by Linux) |
| `0x14` | 32 | `CSG_STATUS_EP_REQ` | Endpoints currently requested (unused by Linux) |
| `0x18` | 32 | `CSG_STATUS_STATE` | Bit 0 = group idle. Other bits reserved (unused by Linux) |
| `0x1C` | 32 | `CSG_RESOURCE_DEP` | Resource dependency information (unused by Linux) |

Block size: `0x20` bytes.

### 6.5.6 CSG sequences

**Start or resume a group in slot *i*** (**[HW]** semantics. **[POLICY]**
batching across slots is a Linux choice):

1. For each stream *j* the group uses, program the CS slot (§6.6.6), then
   collect bit *j* into a mask *Q*.
2. Write `CSG_ALLOW_COMPUTE`, `CSG_ALLOW_FRAGMENT`, `CSG_ALLOW_OTHER`, the
   endpoint request with the chosen priority, `CSG_CONFIG` (AS index),
   `CSG_SUSPEND_BUF`, `CSG_PROTM_SUSPEND_BUF` and `CSG_ACK_IRQ_MASK`.
3. Toggle the bits *Q* in `CSG_DB_REQ`.
4. In `CSG_REQ`, write `STATE` = START (fresh group) or RESUME (group whose
   last state was SUSPEND), and toggle `ENDPOINT_CONFIG`.
5. Toggle bit *i* in `GLB_DB_REQ` and ring doorbell 0.
6. Wait until `CSG_ACK` matches `CSG_REQ` on `STATE` and `ENDPOINT_CONFIG`.

**Suspend or terminate:** write `CSG_REQ.STATE` = SUSPEND or TERMINATE, toggle
`GLB_DB_REQ` bit *i*, ring doorbell 0, and wait for the `STATE` match.

- **[HW]** After a successful SUSPEND, the `CS_STATUS_*` fields of the
  group's streams are up to date, and the host may read them to decide
  whether each stream is idle or blocked (§6.6.4).
- **[HW]** After a group leaves the running state, the host should set each
  of its streams' `CS_REQ.STATE` back to STOP. No doorbell or wait is needed.
  This takes effect the next time the slot is started.
- **[POLICY]** If a suspend times out, Linux sends TERMINATE. If that also
  times out, it relies on the GPU reset to clear the slot.

**Refresh stream status:** toggle `CSG_REQ.STATUS_UPDATE`, ring as above,
wait for the match, then read the `CS_STATUS_*` fields.

**Change priority of a running group:** update `PRIORITY` in the endpoint
request (read-modify-write, keeping the other fields), toggle
`CSG_REQ.ENDPOINT_CONFIG`, ring, and wait. **[HW]** Firmware applies the new
priority only once the toggle is acknowledged. The host should treat the old
priority as current until then.

**CSG events:**
- `SYNC_UPDATE`: a stream wrote a synchronization object. The host should
  re-check its completion sequence numbers (07-scheduler) and re-evaluate any
  host-side waits on blocked streams (§6.6.4).
- `IDLE`: every stream in the group is idle or blocked. **[POLICY]** Linux
  runs the scheduler.
- `PROGRESS_TIMER_EVENT`: a stream made no progress within
  `GLB_PROGRESS_TIMER`. **[POLICY]** Linux marks the group timed out and
  evicts it.

## 6.6 Command Stream (CS) interface

### 6.6.1 CS control block (read-only)

Location: CSG *i* control block `+ 0x40 + j × CSG_STREAM_STRIDE`.

| Offset | Width | Name | Meaning |
|---|---|---|---|
| `0x00` | 32 | `CS_FEATURES` | See below |
| `0x04` | 32 | `CS_INPUT_VA` | MCU VA of this CS slot's input block |
| `0x08` | 32 | `CS_OUTPUT_VA` | MCU VA of this CS slot's output block |

`CS_FEATURES`:

| Bits | Field | Meaning |
|---|---|---|
| [7:0] | `WORK_REGS` | Number of CS work registers **minus one**. Register count = value + 1 |
| [15:8] | `SCOREBOARDS` | Number of scoreboard slots |
| 16 | `COMPUTE` | Stream can issue compute work |
| 17 | `FRAGMENT` | Stream can issue fragment work |
| 18 | `TILER` | Stream can issue tiler (vertex/tiling) work |
| [31:19] | – | Reserved |

- **[ABI]** The register count and scoreboard count are reported to
  userspace. Linux also reserves the top 4 work registers for its own
  ring-buffer prologue and reports that as the "unpreserved" register count
  (10-uapi).

### 6.6.2 `CS_REQ` / `CS_ACK` bits

| Bit(s) | Name | Kind | Meaning |
|---|---|---|---|
| [2:0] | `STATE` | State field | 0 = STOP, 1 = START. Other values reserved |
| 3 | – | – | Reserved |
| 4 | `EXTRACT_EVENT` | Toggle | Ask firmware to report the current extract position (unused by Linux) |
| [7:5] | – | – | Reserved |
| 8 | `IDLE_SYNC_WAIT` | Level | 1 = a stream blocked on a sync wait counts as idle |
| 9 | `IDLE_PROTM_PENDING` | Level | 1 = a stream waiting for protected mode counts as idle (unused by Linux) |
| 10 | `IDLE_EMPTY` | Level | 1 = a stream with an empty ring buffer counts as idle |
| 11 | `IDLE_RESOURCE_REQ` | Level | 1 = a stream waiting for resources counts as idle (unused by Linux) |
| [25:12] | – | – | Reserved |
| 26 | `TILER_OOM` | FW event | Tiler heap out of memory. The host must supply a chunk (§6.6.5) |
| 27 | `PROTM_PENDING` | FW event | Stream wants protected mode (unused by Linux) |
| [29:28] | – | – | Reserved |
| 30 | `FATAL` | FW event | Unrecoverable stream error. See `CS_FATAL` |
| 31 | `FAULT` | FW event | Recoverable stream fault. See `CS_FAULT` |

Groupings: bits [2:0], 4 and [11:8] are host-written, and bits 26, 27, 30 and
31 are firmware events.

> OPEN QUESTION: Linux never waits for `CS_ACK.STATE`, because the CSG
> START/RESUME/SUSPEND ack covers it. Whether firmware updates
> `CS_ACK.STATE` independently is not shown.

### 6.6.3 CS input block (host → FW)

| Offset | Width | Name | Meaning |
|---|---|---|---|
| `0x00` | 32 | `CS_REQ` | Request word (§6.6.2) |
| `0x04` | 32 | `CS_CONFIG` | [3:0] = stream priority (0–15). [15:8] = doorbell ID. Other bits reserved |
| `0x08` | 32 | – | Reserved |
| `0x0C` | 32 | `CS_ACK_IRQ_MASK` | Bit *n* = 1: a change of `CS_ACK` bit *n* is reported via `CSG_CS_IRQ_REQ`. **[POLICY]** Linux writes `0xFFFFFFFF` |
| `0x10` | 64 | `CS_BASE` | GPU VA (in the group's address space) of the ring buffer |
| `0x18` | 32 | `CS_SIZE` | Ring buffer size in bytes |
| `0x1C` | 32 | – | Reserved |
| `0x20` | 64 | `CS_TILER_HEAP_START` | Tiler heap chunk pointer, written in reply to `TILER_OOM` (§6.6.5) |
| `0x28` | 64 | `CS_TILER_HEAP_END` | Tiler heap chunk pointer, written in reply to `TILER_OOM` (§6.6.5) |
| `0x30` | 64 | `CS_USER_INPUT` | MCU VA of the ring-buffer input page (§6.7) |
| `0x38` | 64 | `CS_USER_OUTPUT` | MCU VA of the ring-buffer output page (§6.7) |
| `0x40` | 32 | `CS_INSTR_CONFIG` | Instrumentation configuration (unused by Linux) |
| `0x44` | 32 | `CS_INSTR_BUFFER_SIZE` | Instrumentation buffer size (unused) |
| `0x48` | 64 | `CS_INSTR_BUFFER_BASE` | Instrumentation buffer address (unused) |
| `0x50` | 64 | `CS_INSTR_BUFFER_OFFSET_POINTER` | Address of the instrumentation offset word (unused) |

Size the host relies on: `0x58` bytes.

- **[HW]** `CS_BASE` is a GPU VA in the address space named by
  `CSG_CONFIG`. `CS_USER_INPUT` and `CS_USER_OUTPUT` are MCU VAs (address
  space 0).
- **[HW]** The ring buffer size must be a power of two, because positions
  are reduced modulo the size. **[ABI]** Linux accepts 4 KiB to 64 KiB, power
  of two (10-uapi).
- **[HW]** `CS_CONFIG` priority ranks streams within a group. **[ABI]** It
  comes from the queue-creation parameters (0–15).

> OPEN QUESTION: The encodings of the `CS_INSTR_*` fields are not given.
> Linux leaves them at whatever value the zeroed shared section holds.

### 6.6.4 CS output block (FW → host)

| Offset | Width | Name | Meaning |
|---|---|---|---|
| `0x00` | 32 | `CS_ACK` | Acknowledge word (§6.6.2) |
| `0x04`–`0x3F` | 15×32 | – | Reserved |
| `0x40` | 64 | `CS_STATUS_CMD_PTR` | GPU VA of the current CS instruction (unused by Linux) |
| `0x48` | 32 | `CS_STATUS_WAIT` | Wait status (below) |
| `0x4C` | 32 | `CS_STATUS_REQ_RESOURCE` | Resources the stream is waiting for (unused by Linux) |
| `0x50` | 64 | `CS_STATUS_WAIT_SYNC_POINTER` | GPU VA (group address space) of the sync object being waited on |
| `0x58` | 32 | `CS_STATUS_WAIT_SYNC_VALUE` | Reference value, low 32 bits |
| `0x5C` | 32 | `CS_STATUS_SCOREBOARDS` | Bitmask of scoreboard entries with outstanding (deferred) work. 0 = none |
| `0x60` | 32 | `CS_STATUS_BLOCKED_REASON` | [3:0] blocked reason (below) |
| `0x64` | 32 | `CS_STATUS_WAIT_SYNC_VALUE_HI` | Reference value, high 32 bits (valid for 64-bit waits) |
| `0x68`–`0x7F` | 6×32 | – | Reserved |
| `0x80` | 32 | `CS_FAULT` | [7:0] exception type, [31:8] exception data |
| `0x84` | 32 | `CS_FATAL` | [7:0] exception type, [31:8] exception data |
| `0x88` | 64 | `CS_FAULT_INFO` | Extra fault data |
| `0x90` | 64 | `CS_FATAL_INFO` | Extra fatal-error data |
| `0x98`–`0xBF` | 10×32 | – | Reserved |
| `0xC0` | 32 | `CS_HEAP_VT_START` | Count of tiler (vertex/tiling) passes started on this heap |
| `0xC4` | 32 | `CS_HEAP_VT_END` | Count of tiler passes finished |
| `0xC8` | 32 | – | Reserved |
| `0xCC` | 32 | `CS_HEAP_FRAG_END` | Count of fragment passes finished |
| `0xD0` | 64 | `CS_HEAP_ADDRESS` | GPU VA (group address space) of the tiler heap context that ran out of memory |

Block size: `0xD8` bytes.

**`CS_STATUS_WAIT`:**

| Bits | Field | Meaning |
|---|---|---|
| [15:0] | `SB_MASK` | Scoreboard entries the stream is waiting on |
| [19:16] | `SB_SOURCE` | 0 = none, 8 = wait instruction. Other values not documented |
| [23:20] | – | Reserved |
| [27:24] | `SYNC_WAIT_CONDITION` | 0 = LE, 1 = GT (see below). Other values not documented |
| 28 | `PROGRESS_WAIT` | Waiting for forward progress |
| 29 | `PROTM_PEND` | Waiting to enter protected mode |
| 30 | `SYNC_WAIT_SIZE` | 1 = the sync object is 64-bit. 0 = 32-bit |
| 31 | `SYNC_WAIT` | Stream is blocked in a sync-object wait |

**`CS_STATUS_BLOCKED_REASON[3:0]`:**

| Value | Meaning |
|---|---|
| 0 | Unblocked |
| 1 | Scoreboard wait |
| 2 | Progress wait |
| 3 | Sync-object wait |
| 4 | Deferred |
| 5 | Waiting for resources |
| 6 | Waiting for a flush |
| 7–15 | Reserved |

**Interpreting status after SUSPEND or `STATUS_UPDATE`** (**[HW]**
semantics. What to do about them is policy, see 07-scheduler):

- **Idle stream**: blocked reason 0, *and* ring-buffer `INSERT` equals output
  `EXTRACT` (§6.7), *and* `CS_STATUS_SCOREBOARDS` = 0.
- **Stream blocked on a sync object**: blocked reason 3. It is truly blocked
  only if `CS_STATUS_SCOREBOARDS` = 0. If deferred work is outstanding, the
  stream may still make progress by itself and should be treated as runnable.
- To evaluate a sync wait on the host while the group is not resident:
  1. Read `CS_STATUS_WAIT_SYNC_POINTER`.
  2. Form the reference value from `CS_STATUS_WAIT_SYNC_VALUE`, OR'ing in
     `CS_STATUS_WAIT_SYNC_VALUE_HI << 32` if `SYNC_WAIT_SIZE` = 1.
  3. Read the sync object's sequence number (§6.8) at that GPU VA through
     the group's memory.
  4. The wait is satisfied when, for condition GT, *seq > ref*, or for
     condition LE, *seq ≤ ref*.
  Once it is satisfied the group can be made resident again.
- All other blocked reasons count as runnable.

> OPEN QUESTION: Which side of the condition counts as "satisfied" is taken
> from how Linux unblocks queues (unblocked when the condition holds). The
> authoritative meaning belongs to the CS instruction set used by Mesa.

### 6.6.5 CS event handling

All three events below are reported through `CSG_CS_IRQ_REQ` (§6.3.7).

**`FAULT`** (recoverable):
- **[HW]** Read `CS_FAULT` and `CS_FAULT_INFO`. The exception type uses the
  codes in 02-register-map §2.8. The output-page `EXTRACT` (§6.7) shows how
  far the ring had got. Linux uses it to find the submissions that were in
  flight and fails them.
- **[HW]** Acknowledge by copying `CS_ACK.FAULT` into `CS_REQ.FAULT`. Then
  toggle the CS bit in `CSG_DB_REQ` and ring the CSG (GLB DB bit and doorbell
  0) so that firmware acts on the acknowledgement.

> OPEN QUESTION: Whether the stream resumes execution after a fault ack, or
> stays halted until terminated, is not shown. Linux relies on its scheduler
> and job timeouts either way.

**`FATAL`**:
- **[HW]** Read `CS_FATAL` and `CS_FATAL_INFO`. Acknowledge by copying the
  ACK bit into REQ. The stream cannot continue and the group must be
  terminated.
- **[POLICY]** If the exception type is `CS_UNRECOVERABLE` (`0x41`), Linux
  schedules a full GPU reset and stops scheduling until it has happened.
  Otherwise it only evicts and terminates the group.

**`TILER_OOM`** (heap growth, full policy in 08-heap):
1. **[HW]** Read `CS_HEAP_ADDRESS`, `CS_HEAP_VT_START`, `CS_HEAP_VT_END` and
   `CS_HEAP_FRAG_END`. The number of render passes in flight is
   `VT_START − FRAG_END`. The number of passes waiting for fragment work is
   `VT_END − FRAG_END`.
2. **[HW]** A consistent report satisfies `FRAG_END ≤ VT_END < VT_START`.
   **[POLICY]** Linux treats anything else as a fatal stream error.
3. Allocate a chunk, if policy allows, and write the **same** chunk pointer to
   both `CS_TILER_HEAP_START` and `CS_TILER_HEAP_END`. **[HW]** Pointer
   encoding: bits [63:12] = chunk GPU VA (4 KiB aligned), bits [11:0] = chunk
   size ÷ 4096. **[HW]** Write 0 to both to say "no memory available now". The
   firmware then waits for in-flight fragment work to free memory, or runs the
   stream's out-of-memory exception handler (incremental rendering).
4. **[HW]** Acknowledge by copying `CS_ACK.TILER_OOM` into `CS_REQ.TILER_OOM`.
   Toggle the CS bit in `CSG_DB_REQ`, toggle the CSG bit in `GLB_DB_REQ`, and
   ring doorbell 0.

- **[POLICY]** Linux allocates outside interrupt context and leaves
  `TILER_OOM` unacknowledged until then.
- **[HW]** If the group is suspended before the reply, the request lapses.
  Firmware raises `TILER_OOM` again after the group is resumed.

### 6.6.6 Programming a CS slot before CSG start/resume

1. In the ring-buffer input page, set `EXTRACT` := the output page's
   `EXTRACT`. **[HW]** `INSERT` must be ≥ `EXTRACT`.
2. Write `CS_BASE`, `CS_SIZE`, `CS_USER_INPUT`, `CS_USER_OUTPUT`,
   `CS_CONFIG` (priority and doorbell) and `CS_ACK_IRQ_MASK`.
3. In `CS_REQ`, set `STATE` = START, and set the idle-policy bits.
   **[POLICY]** Linux sets `IDLE_SYNC_WAIT` = 1 and `IDLE_EMPTY` = 1. It
   writes the value `0x00000501` under the mask `0x00000507` (bits [2:0], 8
   and 10).
4. Collect the CS bit for the group's `CSG_DB_REQ` toggle (§6.5.6 step 3).

> Note: Linux's read-modify-write in step 3 touches only bits [2:0], 8 and 10.
> Bits 9 and 11 keep their previous value, which is 0 because nothing else
> sets them.

## 6.7 Ring-buffer interface (per CS)

### 6.7.1 Memory

- **[HW]** Each stream has a ring buffer of `CS_SIZE` bytes at `CS_BASE` in
  the group's GPU address space, holding 64-bit CS instructions.
- **[HW]** Each stream also has an **input page** and an **output page**,
  both at MCU VAs (`CS_USER_INPUT` and `CS_USER_OUTPUT`).
- **[POLICY]** Linux allocates one 8 KiB buffer per queue, mapped in the MCU
  address space uncached and non-executable. The input page is at offset 0
  and the output page at offset 4096. The buffer is zeroed at allocation.

**Input page** (written by the host):

| Offset | Width | Name | Meaning |
|---|---|---|---|
| `0x00` | 64 | `INSERT` | Byte position just past the last valid instruction |
| `0x08` | 64 | `EXTRACT` | Host copy of the extract position (refreshed from the output page) |

**Output page** (written by firmware):

| Offset | Width | Name | Meaning |
|---|---|---|---|
| `0x00` | 64 | `EXTRACT` | Byte position of the next instruction firmware will fetch |
| `0x08` | 32 | `ACTIVE` | Non-zero while the stream is executing (unused by Linux) |

### 6.7.2 Position semantics

- **[HW]** `INSERT` and `EXTRACT` are monotonically increasing 64-bit byte
  counts. They are **not** wrapped to the ring size. The ring offset of
  position *p* is `p mod CS_SIZE`, which is `p AND (CS_SIZE − 1)`.
- **[HW]** The ring holds work when `INSERT > EXTRACT` (output page). It is
  empty when they are equal.
- **[HW]** The host must not let `INSERT − EXTRACT` exceed `CS_SIZE`.
- **[POLICY]** Linux writes each submission as a whole number of 64-byte
  cache lines (multiples of 8 instructions), zero-padding the tail, and
  splits a write that crosses the end of the ring.

> OPEN QUESTION: Whether firmware *requires* cache-line-granular submissions,
> or Linux only pads for efficiency, is not stated.

### 6.7.3 Submitting work

1. Write the instructions into the ring at `INSERT mod CS_SIZE` (wrapping as
   needed).
2. **[HW]** Issue a write memory barrier so the ring contents are visible
   before the pointer update.
3. Write input-page `EXTRACT` := output-page `EXTRACT`, then input-page
   `INSERT` := old `INSERT` + bytes written.
4. If the group is resident in a CSG slot, ring the stream's doorbell
   (`CS_CONFIG.DOORBELL`) with a device-ordered MMIO write. **[HW]** No
   `CS_REQ` or `GLB_DB_REQ` change is needed for new ring-buffer work. If the
   group is not resident, nothing is rung, and firmware sees the new `INSERT`
   when the group is next started or resumed.

### 6.7.4 Userspace visibility

- **[ABI]** In this kernel baseline the ring buffer, input page and output
  page are **kernel-owned** and are not mappable by userspace. Mesa does not
  write `INSERT`/`EXTRACT` directly. It submits through the group-submit
  ioctl. The kernel then writes a short prologue into the ring that calls the
  userspace command stream (10-uapi). A clean-room driver that keeps
  compatibility with existing Mesa must do the same.

## 6.8 Synchronization objects (host interpretation only)

The host reads these objects in two places: job-completion tracking and
evaluating sync waits for non-resident groups. The memory is ordinary GPU
memory in the group's address space.

**32-bit sync object** (8 bytes):

| Offset | Width | Field |
|---|---|---|
| `0x0` | 32 | Sequence number |
| `0x4` | 32 | Status. Non-zero = error |

**64-bit sync object** (16 bytes):

| Offset | Width | Field |
|---|---|---|
| `0x0` | 64 | Sequence number |
| `0x8` | 32 | Status. Non-zero = error |
| `0xC` | 32 | Padding, must be zero |

- **[HW]** The `SYNC_WAIT_SIZE` bit of `CS_STATUS_WAIT` tells which layout a
  blocked stream is waiting on.
- **[POLICY]** Linux tracks completion with one 64-bit sync object per queue,
  in a kernel buffer, that its ring prologue increments. A job is complete
  when the sequence number is ≥ the job's sequence number. `CSG_ACK.SYNC_UPDATE`
  prompts the check (07-scheduler).

## 6.9 Reset and reinitialization

- **[HW]** After any MCU (re)start the host must redo §6.4.6. It must not
  assume any CSG slot is still resident. Groups that were running are lost
  unless they were suspended to their suspend buffers first.
- **[HW]** Before a reset that is not a hang, the host can halt cleanly
  (§6.4.5) to allow a fast restart. **[POLICY]** Linux suspends or
  terminates every resident group before it halts (07-scheduler).
- **[POLICY]** On a slow reset Linux reloads all firmware sections, which
  re-zeroes the shared section and so every input block. On a fast reset it
  keeps them and only switches the halt request back to ACTIVE.

## 6.10 Summary of fixed constants

| Constant | Value | Tag |
|---|---|---|
| Shared section MCU VA | `0x04000000` | [HW] |
| MCU shared region size | `0x04000000` | [HW] |
| First CSG control block offset | `0x1000` | [HW] |
| First CS control block offset within a CSG control block | `0x40` | [HW] |
| Global doorbell ID | 0 | [HW] |
| Doorbell address | `0x80000 + id × 0x10000` | [HW] |
| JOB IRQ global bit | 31 | [HW] |
| Max usable CSG slots | 31 | [HW] |
| Max CS per CSG | 32 | [HW] |
| Timer tick unit (power-off, idle, progress) | 1024 ticks | [HW] |
| Timer source select | `GLB_*_TIMER` bit 31 (1 = GPU cycle counter) | [HW] |
| CSG and CS priority range | 0–15 | [HW] |
| Interface versions with layout changes | 1.1.0, 4.0.0, 4.1.0 | [HW] |
| Linux CSG/ping ack timeout | 100 ms | [POLICY] |
| Linux ping period | 12 s | [POLICY] |
| Linux halt timeout | 1 s | [POLICY] |
| Linux idle / power-off hysteresis | 800 µs / 10 ms | [POLICY] |
| Linux progress timeout | 2 560 000 × 1024 cycles | [POLICY] |
| Linux minimum CSG / CS counts | 3 / 8 | [POLICY] |
| Linux maximum CSG slots used | 16 | [POLICY] |
| Linux unpreserved CS registers | 4 | [ABI] |
