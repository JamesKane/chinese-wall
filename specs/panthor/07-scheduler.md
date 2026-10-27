# 07: Command-Stream Group Scheduling

This section describes how the host driver turns userspace work into
firmware-executed command streams. It covers group and queue resources,
binding groups to firmware slots, the job ring format, completion
tracking, time-multiplexing groups onto a limited number of slots, and
error, reset and power-management handling.

The byte layout of the global, CSG and CS interfaces and the generic
request/acknowledge protocol are in 06-csf-interface. This section refers to
those fields by name only (for example `CSG_REQ.STATE`,
`CS_STATUS_BLOCKED_REASON`). The tiler heap allocator is in 08-heap. The
ioctl structures are in 10-uapi.

Notation: "toggle bit B of X_REQ" means making bit B of the host-written
request word differ from the same bit of the firmware-written acknowledge word.
"Acknowledge event B" means copying bit B of the acknowledge word into the
request word so the two match again (06-csf-interface).

## 7.1 Concepts

| Term | Meaning |
|---|---|
| Queue | A single command stream: a ring buffer of CS instructions plus a small input/output control area. Userspace never writes the ring. The kernel writes it. |
| Group | A set of 1..N queues that share one GPU address space (VM), one priority, and one set of core masks. The firmware schedules groups as a unit. |
| CSG slot | A firmware-side group slot. A group must be bound to a CSG slot to execute. |
| CS slot | A firmware-side stream slot inside a CSG slot. Queue *i* of a group is always placed in CS slot *i* of the group's CSG slot. |
| Sync object | A small memory record (sequence number + status) that the GPU updates and waits on. |

- **[HW]** The firmware reports the number of CSG slots (global control
  `GROUP_NUM`), the number of CS slots per CSG (CSG control `STREAM_NUM`),
  the number of scoreboard slots (CS control `FEATURES` bits [15:8]) and the
  number of CS work registers (CS control `FEATURES` bits [7:0], plus one). All
  CS slots are expected to report identical `FEATURES`.
- **[HW]** At most 31 CSG slots can be used, because the JOB interrupt
  register has 31 per-CSG bits plus the global-interface bit 31 (02-register-map).
- **[HW]** At most 32 CS slots per CSG.
- **[POLICY]** Linux further clamps the usable CSG slot count to 16 so that
  every resident group can be given a distinct firmware CSG priority (0–15,
  see 7.6.3). The reason is that the firmware scheduler can deadlock if two
  groups at the same CSG priority each hold part of a shared resource set and
  both wait for the rest. Giving each resident group a unique priority avoids
  that.
- **[HW]** MMU address space 0 belongs to the MCU (firmware). The address
  spaces available to groups are the remaining bits of `AS_PRESENT`. At least
  one is needed.
- **[HW]** Because there may be more groups than CSG slots, and more VMs than
  address spaces, the host must time-multiplex groups: suspend some, then start
  or resume others.

## 7.2 Per-group and per-queue resources

### 7.2.1 Allocated when the group is created

| Resource | Size | Mapped in | Attributes | Purpose |
|---|---|---|---|---|
| Normal suspend buffer | CSG control `SUSPEND_SIZE` | Firmware (MCU) VM | no-exec | Firmware saves group state here on SUSPEND |
| Protected suspend buffer | CSG control `PROTM_SUSPEND_SIZE` | Firmware VM | no-exec | Firmware saves state of a group suspended in protected mode |
| Sync-object array | 16 bytes × queue count | Group VM | no-exec, uncached, not user-mappable | One 64-bit sync object per queue, at index = queue index. Zero-filled at creation |

Per queue:

| Resource | Size | Mapped in | Attributes | Purpose |
|---|---|---|---|---|
| Ring buffer | `ringbuf_size` from userspace (power of 2, 4 KiB–64 KiB) | Group VM | no-exec, uncached, not user-mappable | CS instructions written by the kernel |
| Queue control area | 8 KiB | Firmware VM | no-exec, uncached, zero-filled | Input page at offset 0, output page at offset 4 KiB |
| Profiling slots | 32 bytes × slot count (7.4.4) | Group VM | no-exec, uncached | Timestamp/cycle samples written by the GPU |

- **[HW]** The suspend buffer sizes are taken from CSG slot 0's control
  block. Linux assumes every CSG slot reports the same sizes.
- **[HW]** The suspend buffers and the queue control area must be reachable
  through the firmware's address space, because the firmware addresses them
  directly. The ring buffer, sync objects and profiling slots are accessed by
  the command-stream engine through the *group's* address space, so they must
  be mapped in the group VM.
- **[HW]** The CS input/output control-area addresses that the host programs
  are 32-bit firmware virtual addresses.
- **[POLICY]** Linux allocates both suspend buffers for every group, even
  though protected mode is never used (7.12).

### 7.2.2 Queue control area layout

| Page | Offset | Width | Field | Written by |
|---|---|---|---|---|
| Input | `+0x0` | 64 | `INSERT`: byte offset one past the last valid instruction | Host |
| Input | `+0x8` | 64 | `EXTRACT`: host copy of the extract pointer | Host |
| Output | `+0x0` | 64 | `EXTRACT`: byte offset of the next instruction the CS will fetch | Firmware |
| Output | `+0x8` | 32 | `ACTIVE` | Firmware |

- **[HW]** `INSERT` and `EXTRACT` increase monotonically and never wrap. The
  ring position is `offset mod ringbuf_size` (the size is a power of two). The
  ring holds unread work while `INSERT > EXTRACT`. `INSERT − EXTRACT` must
  never exceed `ringbuf_size`.
- **[POLICY]** Before programming a CS slot, and again each time it publishes a
  new `INSERT`, Linux copies output `EXTRACT` into input `EXTRACT`.

> OPEN QUESTION: Whether the firmware reads input `EXTRACT` at all (for
> example to seed its fetch pointer when a CS starts) is not shown. The safe
> approach is to keep it equal to the last observed output `EXTRACT`.

### 7.2.3 Sync object format

| Form | Offset | Width | Field |
|---|---|---|---|
| 64-bit | `+0x0` | 64 | Sequence number |
| 64-bit | `+0x8` | 32 | Status (non-zero = error) |
| 64-bit | `+0xC` | 32 | Padding, must be zero |
| 32-bit | `+0x0` | 32 | Sequence number |
| 32-bit | `+0x4` | 32 | Status |

The kernel-owned per-queue sync objects always use the 64-bit form. Userspace
sync objects that a queue waits on may use either form (7.8).

## 7.3 Group lifecycle

### 7.3.1 States

| State | Meaning |
|---|---|
| Created | Resources allocated. Never started on firmware |
| Active | Bound to a CSG slot. Firmware acknowledged START or RESUME |
| Suspended | Firmware acknowledged SUSPEND. State is in the suspend buffer. May or may not still be bound |
| Terminated | Firmware acknowledged TERMINATE, or the host forced this state. Cannot run again |
| Unknown | Firmware reported a `CSG_ACK.STATE` value outside 0–3. Cannot run again. A full GPU reset is scheduled |

The host also tracks these flags, independent of the state:
- *destroyed*: userspace has released its handle.
- *timed-out*: a job timeout or a firmware progress timeout happened, or a suspend failed.
- *fatal queues*: a bitmask of queues that took a CS fatal error.
- *innocent*: the group was killed only because suspension failed during a reset or power-down.

A group is **runnable** only if its state is neither Terminated nor Unknown,
it is not destroyed, it has no fatal queues, and it is not timed out.

```
             bind+START              SUSPEND ack
  Created ───────────────▶ Active ───────────────▶ Suspended
                            ▲  │                       │
                            │  │ TERMINATE ack         │ bind+RESUME
                            │  ▼                       │
                            │ Terminated ◀─────────────┤ (forced on reset
                            │                          │  failure / flush failure)
                            └──────────────────────────┘
  any CSG_ACK.STATE not in {0..3}  ──▶ Unknown (+ full GPU reset)
```

- **[HW]** `CSG_REQ.STATE` encodings: 0 = TERMINATE, 1 = START, 2 = SUSPEND,
  3 = RESUME. The firmware echoes the value into `CSG_ACK.STATE` once the
  transition is done.
- **[HW]** A group that was started at least once and then suspended must be
  brought back with RESUME, not START, so that the firmware reloads it from
  the suspend buffer. A never-run group uses START.
- **[POLICY]** A STATE acknowledgement only counts as complete if all three
  state bits match. A partial match is treated as "not acknowledged".

### 7.3.2 Create

1. **[ABI]** Validate the arguments (7.13).
2. Resolve the VM. Allocate the suspend buffers, the sync-object array and,
   for each queue, its ring buffer, control area and profiling slots.
3. **[POLICY]** Mark every queue idle. Put the group on the idle list for its
   priority. It does not take a CSG slot until it has work.

### 7.3.3 Bind and program (making a group resident)

This is done for a group that the scheduler chose and that has no slot.

1. **[POLICY]** Choose the lowest-numbered free CSG slot.
2. **[HW]** Make the group's VM active in an MMU address space (04-mmu). The
   AS number is needed for `CSG_CONFIG`.
3. **[POLICY]** Doorbell assignment: every queue of the group uses doorbell
   `CSG slot index + 1`. Doorbell 0 is the global-interface doorbell.
4. **[HW]** Program each CS slot *i* (one per queue), **before** the CSG
   fields:
   1. Copy the queue's output `EXTRACT` into its input `EXTRACT`.
   2. `CS_RINGBUF_BASE` = GPU VA of the ring buffer (group VM).
   3. `CS_RINGBUF_SIZE` = ring size in bytes.
   4. `CS_RINGBUF_INPUT` = firmware VA of the input page.
   5. `CS_RINGBUF_OUTPUT` = firmware VA of the output page.
   6. `CS_CONFIG` = queue priority in bits [3:0] | doorbell ID in bits [15:8].
   7. `CS_ACK_IRQ_MASK` = `0xFFFFFFFF`.
   8. In `CS_REQ`, set `IDLE_SYNC_WAIT` (bit 8) = 1, `IDLE_EMPTY` (bit 10) = 1,
      and `STATE` (bits [2:0]) = 1 (START). Change only those bits.
5. **[HW]** Program the CSG slot:
   1. `CSG_ALLOW_COMPUTE` = compute core mask. `CSG_ALLOW_FRAGMENT` =
      fragment core mask. `CSG_ALLOW_OTHER` = tiler core mask.
   2. `CSG_EP_REQ` = max compute cores in bits [7:0] | max fragment cores in
      bits [15:8] | max tiler cores in bits [19:16] | CSG priority in bits
      [31:28]. On firmware interface version ≥ 4.0.0 the 64-bit variant of
      the endpoint request field is written instead of the 32-bit one, with
      the same low bits (06-csf-interface).
   3. `CSG_CONFIG` = MMU address-space number of the group VM.
   4. `CSG_SUSPEND_BUF` = firmware VA of the normal suspend buffer (0 if none).
   5. `CSG_PROTM_SUSPEND_BUF` = firmware VA of the protected suspend buffer
      (0 if none).
   6. `CSG_ACK_IRQ_MASK` = `0xFFFFFFFF`.
   7. Toggle the bit in `CSG_DB_REQ` for every programmed CS slot (relative to
      `CSG_DB_ACK`).
6. **[HW]** Request the transition: in `CSG_REQ`, set `STATE` = START (if
   Created) or RESUME (if Suspended), and toggle `ENDPOINT_CONFIG` (bit 4).
7. **[HW]** Toggle bit *slot* of `GLB_DB_REQ` (relative to `GLB_DB_ACK`) and
   ring doorbell 0.
8. **[HW]** Wait until the requested `CSG_ACK` bits match. Linux waits up to
   100 ms per slot **[POLICY]**, spinning for about 10 µs and then sleeping
   on the JOB interrupt.
9. When `ENDPOINT_CONFIG` is acknowledged, the new priority is in effect.
   When `STATE` is acknowledged, the group becomes Active.
10. **[POLICY]** A timeout in step 8 schedules a full GPU reset (7.11).

- **[HW]** Several slots may be batched: write every slot's requests, toggle
  all of their bits in `GLB_DB_REQ` at once, ring doorbell 0 once, then wait
  on each slot.
- **[POLICY]** Core masks and max core counts come straight from userspace at
  group creation. Linux sets neither of the exclusive-endpoint bits
  (`CSG_EP_REQ` bits 20 and 21).

### 7.3.4 Changing the priority of a resident group

**[HW]** Rewrite only the `CSG_EP_REQ` priority field (bits [31:28]), toggle
`CSG_REQ.ENDPOINT_CONFIG`, then ring and wait as in steps 7 and 8. The
priority is in effect once it is acknowledged.

### 7.3.5 Suspend / terminate (evicting a group)

1. **[HW]** Set `CSG_REQ.STATE` = SUSPEND (group still runnable) or TERMINATE
   (group not runnable). Ring through `GLB_DB_REQ` and doorbell 0, then wait
   for the acknowledgement.
2. **[HW]** When the acknowledgement arrives and the previous state was
   Active, put each CS slot back to STOP by writing `CS_REQ.STATE` = 0. No
   doorbell or wait is needed. The CS slot is reused only when a later CSG
   START or RESUME happens.
3. **[HW]** When the new state is Suspended, read each CS slot's status
   fields to work out whether each queue is idle or blocked (7.7.2).
4. Unbind:
   1. Process any pending CSG and CS events for the slot first (7.9), so that
      nothing is lost once the slot changes owner.
   2. Release the address-space reference (04-mmu).
   3. Discard any pending tiler-OOM work for the group. **[HW]** The firmware
      raises the OOM event again after the group is resumed.
   4. Invalidate the queue doorbell IDs.

### 7.3.6 Destroy

1. **[ABI]** The handle is removed at once. Later submissions fail.
2. Mark the group destroyed. If it is resident, schedule an immediate tick,
   which sends TERMINATE (7.6). Otherwise remove it from all scheduler lists
   and run the termination clean-up (7.10.5) straight away.
3. **[HW]** The firmware may write internal state (for example heap context)
   into the group VM during SUSPEND or TERMINATE. All groups that use a VM
   must therefore be evicted before that VM's page tables are torn down.
4. Free resources after the last reference is dropped.

## 7.4 Job submission and the kernel-written ring sequence

### 7.4.1 What a job is

**[ABI]** A job is the tuple (queue index, user command-stream GPU VA, user
stream size in bytes, flush ID). The user stream must be 64-byte aligned, and
its size must be a multiple of 8. A size of 0 means "no GPU work": the job
completes when every earlier job on the same queue has completed, and the ring
is left untouched.

**[POLICY]** Each queue has its own job queue with dependency fences. A job is
written into the ring only when all its input dependencies have signaled.
Jobs on the same queue are written in submission order.

### 7.4.2 Ring-space accounting

- **[HW]** The host must not overwrite ring bytes that the CS has not yet
  consumed.
- **[POLICY]** Linux treats ring capacity as `ringbuf_size / 8` instruction
  credits. Each job uses the padded length of its kernel sequence (7.4.4): 16
  instructions (128 bytes) with no profiling or one profiling kind, 24 (192
  bytes) with both. Credits come back when the job's completion fence signals.

### 7.4.3 Writing a job

1. **[POLICY]** Assign the job the queue's next fence sequence number. The
   numbers start at 1 for each queue and increase by one per non-empty job.
2. Build the instruction sequence (7.4.4) and pad it with zero words to a
   multiple of 8 instructions (one 64-byte cache line).
3. Copy it into the ring at `INSERT mod size`, wrapping to offset 0 if it
   reaches the end of the ring.
   - **[HW]** Each job's sequence starts on a 64-byte boundary and fills whole
     cache lines. The CS instruction prefetcher needs this. Since every
     sequence is a multiple of 64 bytes and the ring size is a power of two of
     at least 4 KiB, this holds automatically.
4. **[HW]** Issue a write memory barrier so the instructions are visible
   before `INSERT` changes.
5. Write input `EXTRACT` = output `EXTRACT`, then input `INSERT` = old
   `INSERT` + padded byte length.
6. If the group is resident:
   - **[HW]** Ring the queue's doorbell (write 1 to the doorbell page, see
     02-register-map).
   - **[POLICY]** Clear the queue's idle bit, start or resume the queue's
     timeout timer, and mark the device busy for power management.
   - **[POLICY]** If the tick was stopped and the group was idle, restart
     ticking (7.6.5).
7. If the group is not resident, tell the scheduler the group has new work
   (7.6.5). The doorbell is not rung, because the CS will see the new `INSERT`
   when the group is next programmed.

### 7.4.4 Kernel instruction sequence

Notation: `RA` = first register of a 64-bit pair holding an address. `RV` =
`RA + 2`. **[HW]** CS work registers are 32 bits wide, and a 64-bit operand
uses an even/odd pair. The firmware does not preserve the top 4 work
registers across a userspace call. Linux uses that scratch range:
`RA = work_register_count − 4`. For example, with 96 work registers,
`RA` = r92 and `RV` = r94.

The sequence, in order. Rows marked *cycles* or *time* are emitted only when
that profiling kind is enabled for the job (7.14):

| # | When | Instruction | Purpose |
|---|---|---|---|
| 1 | always | MOV32 `RV` ← flush ID from userspace | Flush-reduction token |
| 2 | always | FLUSH_CACHE2 clean+invalidate L2 and LSC, invalidate other caches, no wait, signal scoreboard 0, flush-ID register `RV` | Make prior memory writes visible to the user stream |
| 3 | cycles | MOV48 `RA` ← profiling slot address + 0 | |
| 4 | cycles | STORE_STATE cycle counter → [`RA`] | "before" cycles |
| 5 | time | MOV48 `RA` ← profiling slot address + 16 | |
| 6 | time | STORE_STATE timestamp → [`RA`] | "before" timestamp |
| 7 | always | MOV48 `RA` ← user stream VA | |
| 8 | always | MOV32 `RV` ← user stream size (bytes) | |
| 9 | always | WAIT scoreboard mask `0x0001` | Wait for the flush in #2 |
| 10 | always | CALL address `RA`, size `RV` | Run the user stream |
| 11 | cycles | MOV48 `RA` ← profiling slot address + 8 | |
| 12 | cycles | STORE_STATE cycle counter → [`RA`] | "after" cycles |
| 13 | time | MOV48 `RA` ← profiling slot address + 24 | |
| 14 | time | STORE_STATE timestamp → [`RA`] | "after" timestamp |
| 15 | always | MOV48 `RA` ← GPU VA of this queue's kernel sync object | |
| 16 | always | MOV48 `RV` ← 1 | Increment value (64-bit pair) |
| 17 | always | WAIT scoreboard mask = all scoreboard slots (`(1 << sb_count) − 1`) | Drain every deferred operation of the user stream |
| 18 | always | SYNC_ADD64 [`RA`] += `RV`, system scope, propagate error, no wait | Signal completion |
| 19 | always | ERROR_BARRIER | Confine faults to job boundaries |
| … | | zero words to the next multiple of 8 | |

Counts: 11 instructions unprofiled → padded to 16. Either profiling kind alone
gives 15 → 16. Both give 19 → 24.

### 7.4.5 CS instruction encodings used by the kernel

**[HW]** Every CS instruction is one 64-bit little-endian word. Bits [63:56]
hold the opcode.

| Instruction | Opcode | Operand fields (bit ranges of the 64-bit word) |
|---|---|---|
| MOV48 | 1 | [55:48] destination register (pair base), [47:0] 48-bit immediate |
| MOV32 | 2 | [55:48] destination register, [31:0] 32-bit immediate |
| WAIT | 3 | [31:16] scoreboard slot mask to wait on, other bits 0 |
| CALL | 32 | [47:40] address register pair, [39:32] size register |
| FLUSH_CACHE2 | 36 | [55:48] = 0, [47:40] flush-ID register, [23:16] scoreboard slot to signal (0 here), [15:0] flush mode = `0x233` |
| STORE_STATE | 40 | [47:40] address register pair, [39:32] state selector: 0 = timestamp, 1 = cycle counter |
| ERROR_BARRIER | 47 | all operand bits 0 |
| SYNC_ADD64 | 51 | [55:48] = 0, [47:40] address register pair, [39:32] value register pair, [23:16] = 0, [15:0] = `0x0001` |

Literal words, with `RA`/`RV` substituted, as the kernel emits them:

| Instruction | Value |
|---|---|
| MOV32 r, imm | `0x02 << 56 \| r << 48 \| imm32` |
| FLUSH_CACHE2 | `0x24 << 56 \| RV << 40 \| 0x233` |
| MOV48 r, imm | `0x01 << 56 \| r << 48 \| imm48` |
| STORE_STATE cycles | `0x28 << 56 \| RA << 40 \| 1 << 32` |
| STORE_STATE timestamp | `0x28 << 56 \| RA << 40` |
| WAIT sb0 | `0x03 << 56 \| 0x0001 << 16` |
| WAIT all | `0x03 << 56 \| ((1 << sb_count) − 1) << 16` |
| CALL | `0x20 << 56 \| RA << 40 \| RV << 32` |
| SYNC_ADD64 | `0x33 << 56 \| RA << 40 \| RV << 32 \| 0x1` |
| ERROR_BARRIER | `0x2F << 56` |

> OPEN QUESTION: The flush-mode field `0x233` is documented only as "clean and
> invalidate all". It probably mirrors the `GPU_CMD` flush payload (L2 = 3,
> LSC = 3, other = 2 = invalidate only), but the exact sub-field boundaries
> are not proven.

> OPEN QUESTION: The meaning of the individual bits in SYNC_ADD64's low
> modifier field (`0x0001`) and in bits [55:48] is described only as "system
> scope, propagate error, no wait". Which bit selects which property is not
> shown.

> OPEN QUESTION: An all-zero instruction word (used as padding) is assumed to
> be a NOP. The source relies on it but does not say so explicitly.

> OPEN QUESTION: How FLUSH_CACHE2 uses the flush-ID register. Presumably the
> firmware skips cache maintenance already covered by a flush completed since
> that `LATEST_FLUSH_ID` value (02-register-map, 2.9). This is not stated.

### 7.4.6 Completion detection

- **[HW]** After all of a job's work drains, SYNC_ADD64 increments the queue's
  kernel sync object by 1, so its sequence number equals the number of
  completed non-empty jobs. If the user stream raised an error, "propagate
  error" makes the CS write a non-zero status into the sync object.
- **[HW]** The firmware raises the CSG `SYNC_UPDATE` event (`CSG_REQ`/`CSG_ACK`
  bit 28) after a sync operation in the group.
- On `SYNC_UPDATE` for a slot, for each queue of the bound group, the host
  reads the queue's sync-object sequence number and signals, in order, every
  in-flight job whose fence number ≤ that value. It then re-evaluates blocked
  queues (7.8).
- **[POLICY]** A queue's timeout timer is reset whenever at least one job
  completes, and stopped when no job is left in flight.
- **[POLICY]** Fence error codes are set as follows: CS fault inside the job's
  ring range = invalid; queue fatal = invalid; group timed out = timed out;
  any other termination = cancelled.

## 7.5 Priorities

| Userspace priority | Value | Who may request it |
|---|---|---|
| Low | 0 | anyone |
| Medium | 1 | anyone |
| High | 2 | privileged (nice capability or display master) |
| Realtime | 3 | privileged |

- **[ABI]** The priority values and the permission rule are visible to
  userspace, which can query an allowed-priorities bitmask (10-uapi).
  Requesting a disallowed priority fails with "permission denied". A value
  above 3 fails with "invalid".
- **[HW]** The firmware arbitrates between resident groups using the 4-bit
  CSG priority in `CSG_EP_REQ[31:28]` (15 = highest), and between the queues
  of one group using the 4-bit CS priority in `CS_CONFIG[3:0]`.
- **[POLICY]** The CSG priority is derived from the scheduling order. On each
  tick, the selected groups are ordered by userspace priority (realtime
  first), then by selection order. The first gets CSG priority 15, the next
  14, and so on. Already-resident groups whose priority changed are updated
  through `ENDPOINT_CONFIG`.
- **[POLICY]** The queue priority (0–15) comes straight from userspace.

## 7.6 The tick-based scheduler [POLICY unless marked]

### 7.6.1 Bookkeeping

For each userspace priority the scheduler keeps a *runnable* list (groups with
at least one queue that is neither idle nor blocked) and an *idle* list. There
is one *waiting* list of groups with at least one queue blocked on a sync
object. A group is **idle** when every queue is idle or blocked.

### 7.6.2 Triggers

A scheduling pass ("tick") runs:
- periodically, every **10 ms**, while ticking is enabled;
- immediately when:
  - a CSG `IDLE` event or the global `GLB_IDLE` event arrives;
  - a CS fatal error or a progress-timer event arrives;
  - an MMU fault is reported;
  - a resident group is destroyed or times out;
  - a realtime group gets work or is unblocked;
  - new work arrives while some resident group may be idle, or while slots are free;
  - the device resumes or a reset finishes.

No tick is queued while a reset is pending or in progress.

### 7.6.3 Selection algorithm

A tick is *full* if the time since the last tick is at least one tick period.
Otherwise it is a *mid-slice* tick.

1. **Refresh.** For every occupied CSG slot:
   - **[HW]** If the group's VM has unhandled MMU faults, process the slot's
     pending events. If no CS fatal was reported, mark every queue of that
     group fatal.
   - Record the group as "previously resident", sorted within its userspace
     priority by current CSG priority (highest first).
   - **[HW]** Toggle `CSG_REQ.STATUS_UPDATE` (bit 5) on all occupied slots,
     ring, and wait. When acknowledged, the CS status fields are current. Use
     them to recompute each queue's idle and blocked bits (7.7.2). A timeout
     schedules a GPU reset and aborts the tick.
2. **Mid-slice only.** Going from highest to lowest priority, keep the
   previously resident non-idle groups. At realtime priority, also add
   runnable realtime groups. In effect, only realtime work can preempt in the
   middle of a timeslice, and resident realtime groups are never preempted.
3. **Non-idle pass.** For each priority from realtime down to low:
   1. Take out the previously resident group with the highest CSG priority
      (the rotation candidate).
   2. Add the other previously resident non-idle groups.
   3. Add non-idle groups from the runnable list in FIFO order.
   4. Put the rotation candidate back at the tail and add it if room is left.
      This gives a lower CSG priority to whichever group had the highest one
      last time, which rotates groups of equal userspace priority.
4. **Idle pass.** If slots remain, for each priority add idle groups:
   previously resident ones first (this avoids reprogramming a slot), then
   ones from the idle list.
5. **Constraints on every add:**
   - the group must be runnable;
   - **[HW]** a group whose VM is not already among the selected VMs may be
     added only if the number of distinct VMs stays within the number of
     group address spaces;
   - selection stops when the number of selected groups equals the CSG slot
     count.
6. **Tick continuation.** Ticking stops after this pass unless at least two
   selected **non-idle** groups share a userspace priority, because only then
   is there anything to rotate. If ticking continues, the next tick is due at
   `last tick + 10 ms`.
7. **Apply** (7.6.4).
8. **Power accounting.** If every selected group is idle, the device is
   reported idle and the scheduler drops its runtime-PM reference. Otherwise
   it is reported busy and the reference is held.

### 7.6.4 Applying a selection

1. Previously resident groups that were not selected: request SUSPEND if the
   group is runnable, TERMINATE if not (7.3.5).
2. Selected groups that are already resident: if their computed CSG priority
   differs from the current one, update it (7.3.4).
3. Send all of steps 1–2 as one batch (7.3.3) and wait. On timeout, schedule
   a GPU reset and stop.
4. Unbind the evicted groups.
5. For each selected group that is not resident, from highest priority down:
   bind it to the lowest free slot, program it, and request START or RESUME
   plus `ENDPOINT_CONFIG` (7.3.3). Send as one batch and wait. On timeout or
   bind failure, schedule a GPU reset.
6. Return evicted runnable groups to their runnable or idle list. Queue
   termination clean-up (7.10.5) for evicted groups that cannot run.

### 7.6.5 Waking the scheduler on new work

When a job is written to a non-resident group:
- if every updated queue is blocked on a sync wait, do nothing;
- otherwise clear the idle bits, and if the group was idle, move it to the
  runnable list;
- if the group is realtime, run a tick now;
- else if some resident group may be idle, run a tick now;
- else if ticking is enabled and some CSG slots are free, run a tick now;
- else if ticking is stopped, restart it: the next tick is at
  `last tick + 10 ms` if all slots are used, otherwise now.

### 7.6.6 What the firmware does on its own [HW]

- Once resident, a group runs without host involvement. The firmware picks
  among resident groups by CSG priority and core availability.
- The firmware raises CSG `IDLE` (`CSG_REQ`/`CSG_ACK` bit 29) when a resident
  group has no runnable stream, and `GLB_IDLE` (global bit 26) when the whole
  GPU goes idle. `GLB_IDLE` requires `GLB_IDLE_EN` and an idle hysteresis
  timer configured at boot (05-firmware and 06-csf-interface).
- The CS `REQ` bits `IDLE_SYNC_WAIT` (8) and `IDLE_EMPTY` (10) are set when a
  CS starts. They appear to choose which conditions count as idle for a
  stream: blocked on a sync wait, and ring empty.

> OPEN QUESTION: The precise semantics of `CS_REQ` bits 8–11 (including
> `IDLE_PROTM_PENDING` bit 9 and `IDLE_RESOURCE_REQ` bit 11, which Linux
> leaves clear) are not documented in the source.

> OPEN QUESTION: `CSG_STATUS_STATE` bit 0 ("group is idle") exists, but Linux
> does not read it. It relies on the `IDLE` event plus its own per-queue
> analysis instead.

## 7.7 Queue state analysis

### 7.7.1 When it is done

The host re-derives the idle and blocked bits of each queue after a CSG
`STATUS_UPDATE` acknowledgement and after a group reaches Suspended.

### 7.7.2 Rules

For each CS slot, read `CS_STATUS_BLOCKED_REASON` (low 4 bits):

| Value | Reason | Host interpretation |
|---|---|---|
| 0 | Unblocked | **Idle** if input `INSERT` = output `EXTRACT` **and** `CS_STATUS_SCOREBOARDS` = 0. Otherwise runnable |
| 1 | Scoreboard wait | Runnable |
| 2 | Progress wait | Runnable |
| 3 | Sync wait | Add the group to the waiting list. The queue is **blocked** if `CS_STATUS_SCOREBOARDS` = 0. Record the wait parameters (below) |
| 4 | Deferred | Runnable |
| 5 | Resource | Runnable |
| 6 | Flush | Runnable |

For a sync wait, the host records:
- the sync object GPU VA from `CS_STATUS_WAIT_SYNC_POINTER`;
- the reference value from `CS_STATUS_WAIT_SYNC_VALUE`, plus
  `CS_STATUS_WAIT_SYNC_VALUE_HI` as bits [63:32] when the wait is 64-bit;
- the condition from `CS_STATUS_WAIT` bits [27:24]: 0 = wait until
  value ≤ reference, 1 = wait until value > reference;
- the width from `CS_STATUS_WAIT` bit 30: 1 = 64-bit sync object, 0 = 32-bit.

- **[HW]** A queue in sync wait that still has outstanding scoreboard entries
  (deferred operations in flight) is not really blocked. It will make
  progress by itself.

> OPEN QUESTION: The other `CS_STATUS_WAIT` fields (scoreboard mask bits
> [15:0], scoreboard source bits [19:16], `PROGRESS` bit 28, `PROTM` bit 29,
> and `SYNC` bit 31) are defined but not used.

## 7.8 Synchronization waits

1. **[HW]** A queue executing a sync-wait instruction stops until the condition
   holds. The firmware re-evaluates the wait for resident groups by itself.
2. **[POLICY]** For **non-resident** groups the host does the evaluation, so it
   knows when to give such a group a slot again. On every CSG `SYNC_UPDATE`
   event (from any slot), after a group termination, and after a reset, the
   host runs a sync-evaluation pass:
   1. For each group on the waiting list and each blocked queue, find the
      buffer object that backs the recorded GPU VA in the group's VM, map it,
      make the CPU cache coherent for the 16-byte (64-bit) or 8-byte (32-bit)
      record, and read the sequence number.
   2. Apply the recorded condition: unblocked if (value > reference) for GT,
      or (value ≤ reference) for LE.
   3. Clear the blocked bit of each unblocked queue. If the group is not
      resident, move it to the runnable list. A realtime group triggers an
      immediate tick.
   4. Take the group off the waiting list once no queue is blocked.
3. **[POLICY]** Termination writes the kernel sync object (7.10.5) and then
   runs the pass, so cross-queue waiters are released.

> OPEN QUESTION: The host sees updates made by the GPU only through
> `SYNC_UPDATE` events from resident groups. A non-resident group waiting on
> a sync object written only by the CPU or by another device is not
> re-evaluated until some other event occurs. The queue's job timer keeps
> running in that case (7.10.3), so such a wait is time-limited.

## 7.9 Event (interrupt) processing

**[HW]** The JOB interrupt bit *n* means that CSG slot *n* has events. Bit 31
means the global interface has events. After clearing `JOB_INT`, for each
flagged CSG slot:

1. Read `CSG_REQ`, `CSG_ACK`, `CSG_IRQ_REQ` (written by firmware) and
   `CSG_IRQ_ACK` (written by host). If `CSG_REQ` = `CSG_ACK` and the two IRQ
   words match, there is nothing to do.
2. **[HW]** Write `CSG_IRQ_ACK` := `CSG_IRQ_REQ` *before* looking at any CS
   slot. Otherwise a CS event raised while the host is working could be lost.
3. Acknowledge the CSG events `SYNC_UPDATE` (bit 28), `IDLE` (bit 29) and
   `PROGRESS_TIMER_EVENT` (bit 31).
4. `IDLE`: schedule an immediate tick.
5. `PROGRESS_TIMER_EVENT`: mark the group timed out and schedule an immediate
   tick (7.10.4).
6. For each CS slot whose bit differs between `CSG_IRQ_REQ` and the old
   `CSG_IRQ_ACK`, compute `(CS_REQ ^ CS_ACK)` restricted to the event bits
   `TILER_OOM` (26), `FATAL` (30) and `FAULT` (31):
   - `FATAL`: see 7.10.1.
   - `FAULT`: see 7.10.2.
   - `TILER_OOM`: see 7.10.6. This event is **not** acknowledged here.
   - Acknowledge `FATAL` and `FAULT`.
   - If `FAULT` or `TILER_OOM` was present, toggle this CS slot's bit in
     `CSG_DB_REQ`.
7. `SYNC_UPDATE`: run completion detection (7.4.6) and queue the sync
   evaluation pass (7.8).
8. Toggle bit *slot* of `GLB_DB_REQ` and ring doorbell 0, so the firmware
   sees the acknowledgements and any CS doorbell requests.

For the global interface: if `GLB_IDLE` (bit 26) differs between `GLB_REQ` and
`GLB_ACK`, acknowledge it and schedule an immediate tick.

> OPEN QUESTION: Linux rings the CSG doorbell in step 8 every time, even when
> only CSG events were acknowledged. It is not shown whether the firmware
> needs this to notice acknowledgements or whether it is merely harmless.

## 7.10 Faults, timeouts and termination

### 7.10.1 CS fatal

- **[HW]** Read `CS_FATAL` (bits [7:0] = exception type, [31:8] = exception
  data) and `CS_FATAL_INFO` (64-bit). Exception codes are in 02-register-map
  section 2.8.
- **[HW]** A queue with a fatal error cannot continue. Mark the queue fatal.
  The group becomes non-runnable.
- **[HW]** If the exception type is `CS_UNRECOVERABLE` (`0x41`), the firmware
  state cannot be trusted. Schedule a full GPU reset and stop scheduling
  ticks until the reset is done.
- Otherwise, schedule an immediate tick. That tick finds the group
  non-runnable and sends TERMINATE.

### 7.10.2 CS fault (recoverable)

- **[HW]** Read `CS_FAULT` (same layout as `CS_FATAL`) and `CS_FAULT_INFO`.
- **[POLICY]** Find the in-flight job whose ring range
  `[start, end)` contains the current output `EXTRACT`, and set an "invalid"
  error on its completion fence. The fence still signals when the sync object
  advances.
- **[HW]** Acknowledge `FAULT` and toggle the CS bit in `CSG_DB_REQ` (then ring
  the CSG doorbell) so the stream resumes. The ERROR_BARRIER at the end of
  every kernel sequence keeps the error from leaking into the next job.

> OPEN QUESTION: The exact firmware semantics of resuming after a CS fault
> (whether the faulting user stream is abandoned up to the next ERROR_BARRIER)
> are implied by the instruction name but not documented.

### 7.10.3 Job timeout [POLICY]

- Each queue has a 5-second timer. It runs only while the group is resident
  and the queue has in-flight jobs. When the group is evicted, the timer is
  paused and the remaining time is saved.
- Exception: if the queue is blocked on a sync wait and its whole group is
  idle, the timer keeps running even when the group is evicted, because the
  unblocking event must come from outside.
- The timer is reset to 5 s every time a job on the queue completes.
- On expiry, first check for completions. If none happened, the timeout is
  real: mark the group timed out. If the group is resident, schedule an
  immediate tick, which terminates it. Otherwise terminate it directly.
  Timed-out groups never run again.

### 7.10.4 Firmware progress timeout

- **[HW]** The firmware raises `PROGRESS_TIMER_EVENT` for a group whose
  stream makes no forward progress within the global progress-timer budget
  configured at boot (05-firmware).
- **[POLICY]** Linux configures that budget to 2,560,000 units of 1024 GPU
  cycles (about 2.6 × 10⁹ cycles), treats the event like a job timeout, and
  terminates the group.

### 7.10.5 Termination clean-up

Once a non-runnable group is off its slot (TERMINATE acknowledged, or it was
never resident):
1. Signal every in-flight job of every queue with the error described in
   7.4.6.
2. **[POLICY]** For each queue, write the kernel sync object: status =
   `0xFFFFFFFF`, sequence number = the last fence number issued on that queue.
   Because GPU sync waits observe this memory, this releases any GPU or host
   waiter that targets it.
3. Run the sync evaluation pass (7.8).

**[ABI]** The group-state query reports bit 0 = timed out, bit 1 = fatal fault
(plus the fatal-queue bitmask), and bit 2 = innocent (10-uapi).

### 7.10.6 Tiler out-of-memory

1. **[HW]** On `CS_TILER_OOM`, read `CS_HEAP_ADDRESS` (heap context GPU VA),
   `CS_HEAP_VT_START`, `CS_HEAP_VT_END` and `CS_HEAP_FRAG_END`.
2. **[HW]** Sanity check: require `FRAG_END ≤ VT_END < VT_START`. If this
   fails, treat the event as a fatal heap error.
3. Derive:
   - render passes in flight = `VT_START − FRAG_END`;
   - pending fragment jobs = `VT_END − FRAG_END`.

   Ask the heap allocator for a new chunk (08-heap).
   - **[POLICY]** This runs outside the interrupt context and outside the main
     scheduler lock, on a separate worker, because the allocation may block
     on memory reclaim.
4. Outcomes:
   - Chunk allocated: its GPU VA is the new value.
   - Allocator reports "out of memory" or "limit reached": the new value is 0.
     **[HW]** A zero chunk tells the firmware to recover by itself, either by
     waiting for in-flight fragment work to free chunks or by running the
     userspace tiler-OOM exception handler (incremental rendering).
   - Any other failure: mark the queue fatal and schedule an immediate tick.
5. **[HW]** If the group is still resident, write `CS_HEAP_START` =
   `CS_HEAP_END` = new value, acknowledge `TILER_OOM` in `CS_REQ`, toggle the
   CS bit in `CSG_DB_REQ`, then toggle the slot bit in `GLB_DB_REQ` and ring
   doorbell 0.
6. If the group was evicted while the chunk was being allocated, give the
   chunk back to the heap. **[HW]** The firmware will raise `TILER_OOM` again
   after the group is resumed.

### 7.10.7 MMU faults

**[POLICY]** An MMU fault on a group's address space schedules an immediate
tick. The tick processes the slot's events and marks the group fatal if the
firmware did not report a CS fatal, then terminates it (7.6.3 step 1).

## 7.11 Full GPU reset

Triggered by any of: an unrecoverable CS fatal; an Unknown CSG state; a CSG
request acknowledgement timeout; a firmware ping timeout (05-firmware).

1. **Pre-reset.**
   1. Mark a reset as in progress. While the mark is set, no tick or
      sync-evaluation pass is queued, and newly created groups are parked as
      stopped.
   2. Cancel any pending tick and sync-evaluation work.
   3. Suspend all resident groups exactly as for power-down (7.12.1), with
      the same escalation path.
   4. Stop each group's job queue so that no new job is written into any
      ring. Park every group on a "stopped" list.
2. **Hardware/firmware reset** (03-gpu-control, 05-firmware): halt or stop the
   MCU, reset the GPU, power L2, reset the MMU, and reboot the firmware. If
   the MCU halted cleanly a *fast* reset is used. Otherwise all firmware
   sections are reloaded.
3. **Post-reset.**
   1. If the firmware failed to reboot, force every stopped group to
      Terminated (and the device becomes unusable).
   2. Restart each group's job queue. Jobs already written to a ring are
      **not** rewritten, and their completion fences stay attached.
   3. Runnable groups go back to the runnable or idle list. Others get
      termination clean-up.
   4. Clear the in-progress mark. Unless the reset failed, run an immediate
      tick and a sync-evaluation pass.

- **[HW]** In-flight work survives the reset only through the suspend buffer.
  A group that was successfully suspended before the reset is later RESUMEd
  from its suspend buffer and continues from where it stopped. A group whose
  suspension failed is lost.

> OPEN QUESTION: It is assumed, but not proven, that a suspend buffer written
> before a *slow* reset (full firmware reload) is still valid for RESUME
> after it. Linux relies on this.

## 7.12 Power management

### 7.12.1 Before power-off (and before reset)

1. **[HW]** For every occupied CSG slot, request SUSPEND (runnable group) or
   TERMINATE (non-runnable group). Send as one batch and wait (100 ms).
2. **[POLICY]** For slots that time out, mark the group timed out (and
   innocent if it was runnable), then request TERMINATE as a second batch.
3. **[POLICY]** If TERMINATE also times out, force the group to Terminated
   and put its CS slots back to STOP by hand. The coming soft reset kills
   whatever is left on the firmware side.
4. **[HW]** For slots that suspended successfully, clean (write back) the L2
   and LSC caches with `GPU_CMD` `FLUSH_CACHES` (clean only, no invalidate),
   so the suspend buffers and sync objects are in memory before power is
   removed. If the flush fails, mark those groups Terminated. If it
   succeeds, run completion detection on them.
5. Unbind every group. Runnable groups are put on the idle list of their
   priority. The others get termination clean-up.

All groups must be off the firmware before power-off. **[HW]** The firmware
does not keep CSG slot contents across power-down.

### 7.12.2 After power-on

**[POLICY]** Run an immediate tick. Groups come back through the normal
selection path, RESUMEd from their suspend buffers.

### 7.12.3 Runtime PM coupling [POLICY]

The scheduler holds a runtime-PM reference while any selected group is
non-idle, or when a job is written to a resident non-blocked queue. It drops
the reference when a tick finds every selected group idle. The firmware's
`GLB_IDLE` event drives this.

## 7.13 Argument validation [ABI]

Group creation:
- the queue count is 1..`min(32, CS slots per CSG)`;
- priority ≤ 3, and 2 or 3 need privilege (7.5);
- the compute and fragment core masks must be subsets of `SHADER_PRESENT`,
  and the tiler mask a subset of `TILER_PRESENT`;
- for each mask, popcount(mask) ≥ the matching max-cores value;
- the VM handle must be valid;
- padding fields must be zero;
- at most 128 groups per open file. Handles are 1..128.

Queue creation (per queue):
- ring size is a power of two in [4096, 65536];
- queue priority ≤ 15;
- padding is zero.

Job submission (per job):
- padding is zero;
- stream address and size are either both zero or both non-zero;
- address is 64-byte aligned and size is 8-byte aligned;
- flush ID bits [30:24] are zero;
- the queue index exists;
- the group handle is valid **and the group is runnable** (otherwise
  "invalid").

All jobs in one submission target one group, and therefore one VM.

> OPEN QUESTION: Why flush-ID bits [30:24] must be zero while bit 31 is
> allowed is not explained. It probably reflects the width of the hardware
> flush counter.

## 7.14 Profiling

- **[POLICY]** A device-wide profiling mask (bit 0 = cycles, bit 1 =
  timestamps) is sampled when each job is created and decides which optional
  rows of 7.4.4 are emitted for that job. Changing the mask therefore changes
  ring contents and per-job credit usage for later jobs.
- Each queue has a ring of 32-byte profiling slots:

  | Offset | Content |
  |---|---|
  | `+0x00` | cycle count before the user stream |
  | `+0x08` | cycle count after |
  | `+0x10` | timestamp before |
  | `+0x18` | timestamp after |

- **[POLICY]** The slot count is `ceil(ringbuf_size / 128)`, which is the
  most profiled jobs the ring can hold at once (a profiled job is at least 16
  instructions). Each non-empty job takes the next slot, wrapping around.
- **[HW]** STORE_STATE writes a 64-bit value of the GPU cycle counter
  (selector 1) or GPU timestamp (selector 0) to the given address. The values
  are in the same domains as the `CYCLE_COUNT` and `TIMESTAMP` registers
  (02-register-map).
- **[POLICY]** On completion, the host adds `after − before` to per-client
  usage statistics.

## 7.15 Protected mode

**[POLICY]** Linux does not support protected-mode execution. It never sets
`GLB_PROTM_ENTER`, ignores `CS_PROTM_PENDING` (bit 27) and leaves
`CS_REQ.IDLE_PROTM_PENDING` clear. The only protected-mode-related work it
does is allocating and programming the protected suspend buffer for each
group (7.2.1), since the firmware advertises a size for it.

> OPEN QUESTION: The protected-mode entry and exit handshake (request, global
> acknowledge, `GPU_STATUS` bit 7, `PROTM_FAULT`) cannot be derived from this
> source and would need another reference.
