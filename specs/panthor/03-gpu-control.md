# 03: GPU Control (Identification, Reset, Power, Caches, Interrupts)

This section describes the sequences a driver runs on the GPU-control block
(arch 10–14) and on the PWR block (arch 14). Register names, offsets and bit
encodings are in [02-register-map](02-register-map.md) and are not repeated
here. Platform clocks, supplies and power domains are in
[01-platform](01-platform.md). Firmware (MCU) boot and the global interface
are in 05 and 06.

All waits below have two forms: "poll" (read a register repeatedly with a
delay) and "wait for IRQ" (sleep until the interrupt handler reports an
event). Delay and timeout values are Linux's choices and are **[POLICY]**
unless marked otherwise.

## 3.1 Identification and feature probe

Run with the GPU powered and clocked (01 §1.4.2), before any other GPU access.

1. Read `GPU_ID`. **[HW]** A value of 0 means the GPU is unpowered, unclocked
   or absent. Fail probe.
2. Pick the hardware generation from `GPU_ID.ARCH_MAJOR`:
   - 10–13: legacy power control through the GPU-control `*_PWRON`/`*_PWROFF`
     registers.
   - 14: PWR block power control with delegation to firmware.
   - Anything else: unsupported. **[POLICY]** Linux fails probe.
3. Read and keep these registers for userspace queries and internal use
   (02 §2.3): `CSF_ID`, `REVID`, `CORE_FEATURES`, `L2_FEATURES`,
   `TILER_FEATURES`, `MEM_FEATURES`, `MMU_FEATURES`, `THREAD_FEATURES`,
   `THREAD_MAX_THREADS`, `THREAD_MAX_WORKGROUP_SIZE`,
   `THREAD_MAX_BARRIER_SIZE`, `COHERENCY_FEATURES`, `TEXTURE_FEATURES[0..3]`,
   `AS_PRESENT`, and the 64-bit `GPU_FEATURES`.
   **[ABI]** These values are passed to userspace unchanged (10).
4. Read the present masks:
   - Arch 10–13: `SHADER_PRESENT`, `TILER_PRESENT`, `L2_PRESENT` from the
     GPU-control block.
   - Arch 14: the same three from the PWR block (02 §2.5).
5. Apply the `shader-present` nvmem override if present (01 §1.7).
6. **[POLICY]** Log a marketing name using the product table in 02 §2.3.1.
   It has no functional effect.

Derived values used elsewhere:
- PA bits = `MMU_FEATURES[15:8]`: the DMA addressing limit (01 §1.4.1).
- VA bits = `MMU_FEATURES[7:0]`: used by 04.
- Ray-traversal support = `GPU_FEATURES` bit 5: decides whether the shader
  domain's RTU subdomain is included in arch 14 power commands (3.5).

## 3.2 Driver-side request tracking

The GPU-control and PWR blocks each signal command completion through an
interrupt bit. Linux keeps a per-block "pending requests" bitmask, protected
by a spinlock that the interrupt handler also takes:

- Before issuing a command, set the bit for the expected completion event.
- The interrupt handler clears pending bits that match the events it saw and
  wakes waiters.
- **[HW]** If the wait times out, check `INT_RAWSTAT` directly before reporting
  failure. The event may have fired while the interrupt was masked or still
  being handled. Linux treats "raw status shows completion" as success.

This is **[POLICY]** in structure. The fallback check of `INT_RAWSTAT` is
recommended, because interrupt masking during processing (3.8) can delay
delivery.

## 3.3 Soft reset

The GPU is reset by register command. No external reset line is used
(01 §1.1.3).

### 3.3.1 Arch 10–13

1. Mark `RESET_COMPLETED` as pending.
2. Write `RESET_COMPLETED` (bit 8) to the GPU `INT_CLEAR`. This removes a
   stale completion.
3. Write `GPU_CMD` = `0x101` (soft reset).
4. Wait for `RESET_COMPLETED`. **[POLICY]** 100 ms timeout, then the raw-status
   check (3.2).
5. On success, clear all pending GPU requests. Any cache flush in flight is
   lost in the reset.

- The GPU-control interrupt stays enabled during the reset, with
  `RESET_COMPLETED` in its mask (3.8.2).
- Hard reset (`GPU_CMD` = `0x201`) exists but Linux never uses it.

> OPEN QUESTION: Linux does not rewrite the GPU-control `INT_MASK` after a
> soft reset done by the recovery worker (3.9), and it does not reprogram
> `COHERENCY_PROTOCOL`/`L2_CONFIG` except at the following L2 power-up. Whether
> soft reset clears `INT_MASK` is not established. A cautious driver
> rewrites `INT_CLEAR` and `INT_MASK` for every block after a soft reset.

### 3.3.2 Arch 14

1. Read `PWR_STATUS`. If bit 33 (soft reset allowed) is clear, fail and do not
   issue the command. **[HW]** Firmware or a security monitor may forbid host
   resets.
2. Mark PWR `RESET_COMPLETED` (PWR interrupt bit 3) as pending and write that
   bit to the PWR `INT_CLEAR`.
3. Write `PWR_COMMAND` = `0x00000031` (soft reset, domain 0, no subdomain).
   `PWR_CMDARG` is not written.
4. Wait for PWR `RESET_COMPLETED`. **[POLICY]** 500 ms timeout, then the
   raw-status check.

- Fast reset (`0x32`, allowed when `PWR_STATUS` bit 34 is set) exists but Linux
  never uses it.

## 3.4 L2 and core power, arch 10–13

**[HW]** The host powers only the **L2**. Shader cores and tilers are powered
on and off by the MCU firmware according to the core masks it is given
through the global interface (06). The host never writes `SHADER_PWRON`,
`TILER_PWRON` or the matching `PWROFF` registers on these GPUs.

### 3.4.1 L2 power-up

1. Write `COHERENCY_PROTOCOL` with the selected protocol (01 §1.6: 0 for
   ACE-Lite, 31 for none). **[HW]** This must be programmed before the L2
   powers up.
2. If the SoC needs a custom address hash (01 §1.3) and ARCH_MAJOR ≥ 11:
   write `ASN_HASH[0]`, `ASN_HASH[1]`, `ASN_HASH[2]`. Then read-modify-write
   `L2_CONFIG` to set bit 24. **[HW]** This must also be done before the L2
   powers up.
3. Poll `L2_PWRTRANS` until bit 0 is clear. **[POLICY]** 100 µs interval,
   20 ms timeout.
4. Write `L2_PWRON` = `0x1` (64-bit write: low word 1, high word 0).
5. Poll `L2_READY` until the L2 reports ready. **[POLICY]** 100 µs interval,
   20 ms timeout.

- Only bit 0 of the L2 mask (the first core group) is powered. **[POLICY]**
  If `L2_PRESENT` has more than one bit set, Linux uses only the first core
  group and logs that. Multi-core-group GPUs work at reduced width.
- Interrupts are not used for L2 power transitions. The
  `POWER_CHANGED`/`POWER_CHANGED_ALL` events are enabled only for optional
  tracing.

> OPEN QUESTION: In step 5 Linux's poll condition accepts any `L2_READY` value
> whose set bits are a subset of the requested mask. That includes `0`, so it
> can return before the L2 is actually ready. The intended hardware condition
> is presumably "`L2_READY` bit 0 is set". A reimplementation should wait for
> `(L2_READY & 1) == 1`. Also, Linux polls each register as a 64-bit pair but
> compares only the low 32 bits.

### 3.4.2 L2 power-down

1. mask = `L2_PRESENT` (the full mask, not just bit 0).
2. Poll `L2_PWRTRANS` until `(value & mask) == 0`. **[POLICY]** 100 µs
   interval, 20 ms timeout.
3. Write `L2_PWROFF` = mask (64-bit).
4. Poll `L2_PWRTRANS` again until `(value & mask) == 0` (same timing).

- **[HW]** The MCU must already be halted or disabled, with its shader and
  tiler cores off, before the L2 goes down (05). Linux does not check
  `SHADER_READY`/`TILER_READY` first.
- **[POLICY]** Linux does not check that `L2_READY` reads 0 afterwards.
- A timeout is logged but not propagated.

## 3.5 L2 and core power, arch 14 (PWR block)

On arch 14 every power domain (L2, TILER, SHADER, BASE, STACK) is either
under host control ("allowed" bit in `PWR_STATUS`) or delegated to the MCU
("delegated" bit). The host powers the L2 itself, then **delegates** shader
and tiler control to firmware. **[HW]** The L2 domain is never delegated.

Register shorthand: for domain *d*, `READY(d)` and `PWRTRANS(d)` are that
domain's 64-bit ready and transition registers in the PWR block (02 §2.5).
`ALLOW(d)` = `PWR_STATUS` bit *d*. `DELEG(d)` = `PWR_STATUS` bit 16+*d*.

### 3.5.1 Generic domain power transition

Used for power-up (`0x10`) and power-down (`0x11`) of a domain with a core
mask *m*:

1. target = *m* for power-up, 0 for power-down.
2. Poll `PWRTRANS(d)` until it reads 0 (all 64 bits). **[POLICY]** 100 µs
   interval, 2 s timeout.
3. If `(READY(d) & m) == target`, stop: the domain is already there.
4. Write `PWR_CMDARG` = *m* (64-bit, skipped if *m* is 0), then write
   `PWR_COMMAND` = `(sub << 16) | (d << 8) | cmd`. Here *sub* = 1 (RTU) for
   the SHADER domain when `GPU_FEATURES` bit 5 is set, otherwise 0.
5. Poll `READY(d)` until `(READY(d) & m) == target`. **[POLICY]** 100 µs
   interval, 2 s timeout.

### 3.5.2 Delegate a domain to firmware

1. Read `PWR_STATUS`. If `DELEG(d)` is set, stop (already delegated).
2. If `ALLOW(d)` is clear, fail: the host does not control it and may not
   delegate it.
3. Poll `PWRTRANS(d)` until 0 (2 s).
4. Write `PWR_COMMAND` = `0x20 | (d << 8)` (no subdomain, no argument).
5. Poll `PWR_STATUS` until `ALLOW(d)` = 0 **and** `DELEG(d)` = 1. **[POLICY]**
   10 µs interval, 2 s timeout.

### 3.5.3 Retract a domain from firmware

1. Poll `PWR_STATUS` until bit 43 (retract pending) is clear. **[POLICY]**
   Continuous polling, 2 ms timeout.
2. If `DELEG(d)` was clear (in the `PWR_STATUS` value read on entry), stop.
3. Write `PWR_COMMAND` = `0x21 | (d << 8)`.
4. Poll `PWR_STATUS` until `DELEG(d)` = 0 **and** `ALLOW(d)` = 1. **[POLICY]**
   10 µs interval, 2 s timeout.

### 3.5.4 L2 power-up (arch 14)

1. Read `PWR_STATUS`. If `ALLOW(L2)` (bit 0) is clear, fail.
2. Power up domain L2 (*d* = 0) with *m* = `L2_PRESENT` (3.5.1). Unlike arch
   10–13, **all** L2 slices are powered.
3. Delegate SHADER (*d* = 8), then TILER (*d* = 1), to firmware (3.5.2). If
   the TILER delegation fails, retract SHADER (3.5.3) and report failure.

> OPEN QUESTION: Linux writes neither `COHERENCY_PROTOCOL` nor the ASN hash
> registers on the arch 14 path. It is not shown whether arch 14 hardware
> still needs them before L2 power-up, or whether they moved elsewhere. A
> coherent arch 14 platform may need step 3.4.1(1) added.

### 3.5.5 L2 power-down (arch 14)

1. Read `PWR_STATUS`. If `ALLOW(L2)` is clear, log and give up.
2. Force TILER off, then force SHADER off. For each domain *d*:
   1. r = `READY(d)`. If r is 0, skip this domain.
   2. Retract *d* (3.5.3). On failure, stop the whole sequence and leave the
      L2 on.
   3. Power down *d* with *m* = r (3.5.1).
   4. On failure, stop the whole sequence. The domain stays under host
      control.
3. Power down L2 with *m* = `L2_PRESENT` (3.5.1).

- **[HW]** A halted MCU should already have powered down its delegated
  domains. The retract-and-force-off step covers a hung MCU that did not.
  Order: tiler, shader, then L2.
- After a reset or the next L2 power-up, shader and tiler are delegated again
  by 3.5.4.

## 3.6 Cache flush and clean

Command and payload encoding: 02 §2.3.3. Completion: GPU interrupt bit 17
`CLEAN_CACHES_COMPLETED`.

### 3.6.1 Sequence

1. **[POLICY]** Serialize flushes. Only one may be outstanding.
2. If a flush is still marked pending (an earlier one never completed), fail
   at once without writing the command.
3. Mark `CLEAN_CACHES_COMPLETED` as pending. Write `GPU_CMD` =
   `0x04 | (payload << 8)`.
   Linux does **not** clear `CLEAN_CACHES_COMPLETED` in `INT_CLEAR` first. It
   relies on the interrupt handler having acknowledged earlier ones.
4. Wait for `CLEAN_CACHES_COMPLETED`. **[POLICY]** 100 ms timeout, then the
   raw-status check (3.2).
5. **[POLICY]** On timeout, schedule a full GPU reset (3.9) and return an
   error.

### 3.6.2 Flush types Linux uses

| Use | L2 | LSC | Other | `GPU_CMD` value |
|---|---|---|---|---|
| After page-table changes / before unlocking an MMU region / when disabling an address space (04) | clean + invalidate | clean + invalidate | invalidate | `0x00023304` |
| After command-stream groups are suspended (07), so suspend buffers in memory are up to date | clean | clean | none | `0x00001104` |

- **[HW]** The flush must complete before an MMU region is unlocked after
  page-table updates. Otherwise the GPU can use stale data or translations
  (04).
- `LATEST_FLUSH_ID` (02 §2.9) increments as flushes complete. Userspace uses it
  to skip redundant flushes in command streams (3.11).

## 3.7 Suspend and resume of the GPU-control layer

The platform-level sequence is in 01 §1.4.2–§1.4.3. This is the GPU-level
part.

### 3.7.1 Suspend (after scheduler and MMU are quiesced)

1. Firmware suspend (05/06): request an MCU halt through the global
   interface and poll for `MCU_STATUS` = 2 (halted). **[POLICY]** 10 µs
   interval, 1 s timeout. On firmware versions that have it, the global
   state must also report halted.
   - Halt confirmed: the next restart may be a **fast** restart (firmware
     sections are not reloaded).
   - Halt failed: the next restart is **slow** (all sections reloaded).
2. Mask the `"job"` interrupt block (3.8.3). Write `MCU_CONTROL` = 0, then
   poll until `MCU_STATUS` = 0. **[POLICY]** 10 µs interval, 100 ms timeout.
3. MMU suspend (04).
4. GPU suspend:
   - Fast restart possible: power down the L2 only (3.4.2 or 3.5.5).
   - Otherwise: soft reset (3.3).
5. Mask the GPU-control interrupt block (3.8.3), then the PWR block on
   arch 14.

### 3.7.2 Resume

1. If a recovery reset was pending when the device suspended, force a **slow**
   restart and clear the pending flag.
2. Arch 14: unmask the PWR interrupt block (3.8.3).
3. Unmask the GPU-control interrupt block. Power up the L2 (3.4.1 or 3.5.4).
4. MMU resume (04).
5. Firmware restart (05): make the MCU's address space active. On a slow
   restart reload every firmware section. On a fast restart, put the global
   halt request back to "active". Unmask the `"job"` interrupt, write
   `MCU_CONTROL` = 2 (auto), and wait for the global-interface bit (JOB
   interrupt bit 31). **[POLICY]** 1 s timeout. Re-initialize the global
   interface (even on a fast restart).
6. **[POLICY]** If a fast restart fails: undo steps 2–4 (MMU suspend, then L2
   power-down since the mode is still "fast", then interrupt masking), switch
   to slow, and run steps 2–5 again.
7. Scheduler resume (07).

## 3.8 Interrupt handling model

Applies to all four interrupt blocks (GPU, PWR, JOB, MMU). Register layout:
02 §2.1.1.

### 3.8.1 Line properties

- **[HW]** Lines are level-triggered. A line stays asserted while
  `INT_STAT` = `INT_RAWSTAT & INT_MASK` is non-zero. Acknowledge events by
  writing 1s to `INT_CLEAR`.
- **[HW]** On arch 14 the GPU-control and PWR blocks share the `"gpu"` line.
  Linux installs all handlers as shared. A handler must return "not mine"
  when its own `INT_STAT` is 0.

### 3.8.2 Handling (Linux structure, behavior is what matters)

For each block the driver keeps an *enabled-events mask* and a state
(active, processing, suspending, suspended).

Hard-interrupt part:
1. If the block is not *active*, return "not mine".
2. If `INT_STAT` is 0, return "not mine".
3. Write `INT_MASK` = 0, so the level line drops and other blocks on the line
   are not starved. Set state *processing* and defer to a thread.

Threaded part:
1. Loop: read `INT_RAWSTAT & enabled-events`. If 0, leave the loop. Otherwise
   run the block's handler on those bits. Each handler starts by writing the
   bits to `INT_CLEAR`. **[HW]** Reading `INT_RAWSTAT` (not `INT_STAT`) is
   needed here because `INT_MASK` is 0 while processing.
2. If the state is still *processing*, set *active* and write
   `INT_MASK` = enabled-events.

Mask updates while processing or suspended are recorded only in the
enabled-events mask. They are written to `INT_MASK` on the return to
*active* (end of processing, or resume).

### 3.8.3 Initialization, suspend, resume

- Before installing the handler: write `INT_MASK` = 0 and start in
  *suspended*.
- Suspend: set *suspending*, write `INT_MASK` = 0, wait for any running
  handler instance (hard and threaded) to finish, then set *suspended*.
- Resume: set *active*, write `INT_CLEAR` = enabled-events (drop anything
  latched while suspended), then write `INT_MASK` = enabled-events.
- **[HW]** All interrupt-block register accesses need the GPU clocked and
  powered. Mask everything and wait for handlers to finish before clocks
  stop.

### 3.8.4 Enabled events per block

| Block | Enabled events (Linux) |
|---|---|
| GPU control | `FAULT`, `PROTM_FAULT`, `RESET_COMPLETED`, `CLEAN_CACHES_COMPLETED` (bits 0, 1, 8, 17). `POWER_CHANGED`/`POWER_CHANGED_ALL` (bits 9, 10) only while power tracing is on (arch 10–13) |
| PWR (arch 14) | Bits 0, 1, 2, 3, 4, 5, 30, 31 (all defined events) |
| JOB | All 32 bits (05/06) |
| MMU | Per-address-space fault bits only (04) |

### 3.8.5 GPU-control event handling

| Event | Action |
|---|---|
| `FAULT` | Read `GPU_FAULT_STATUS` (bits [7:0] = exception code, 02 §2.8) and `GPU_FAULT_ADDR`. **[POLICY]** Log only. No reset is started |
| `PROTM_FAULT` | **[POLICY]** Log only |
| `RESET_COMPLETED`, `CLEAN_CACHES_COMPLETED` | Clear the matching pending-request bits and wake waiters (3.2) |
| `POWER_CHANGED*` | **[POLICY]** Optional tracing of `SHADER_READY`, `TILER_READY`, `L2_READY` |

PWR events: `COMMAND_NOT_ALLOWED` (bit 30) and `COMMAND_INVALID` (bit 31) are
logged. `RESET_COMPLETED` wakes the reset waiter. Power-change, delegation,
retract and inspect events are acknowledged but not waited on, because Linux
polls the status registers instead.

## 3.9 Device reset and recovery

### 3.9.1 Triggers ([POLICY])

A recovery reset is requested when:
- a cache flush times out (3.6),
- an MMU address space stays busy (`AS_STATUS` bit 0) longer than 100 ms, or a
  region unlock fails (04),
- the firmware stops answering the periodic global ping (every 12 s, with a
  100 ms acknowledgement timeout) (06),
- the scheduler sees a command-stream-group update request time out, a group
  fail to bind to a slot, a group reach an unknown state, or a command
  stream report the `CS_UNRECOVERABLE` exception (07).

Requests are merged. A "reset pending" flag is set atomically, and the work is
queued only if the flag was clear and the device is active. If the device is
suspending or suspended, the work is not queued. Instead, the next resume
does a slow restart (3.7.2).

### 3.9.2 Recovery sequence

Runs in an ordered worker so recoveries never overlap. Skip it if the device
is not active or has been unplugged.

1. Stop the scheduler: no new submissions, resident groups marked for
   recovery (07).
2. Firmware pre-reset: stop the ping. Do **not** try a graceful halt (the
   restart is always **slow**). Mask the `"job"` interrupt. Write
   `MCU_CONTROL` = 0 and poll `MCU_STATUS` = 0 (100 ms).
3. MMU pre-reset: mask the MMU interrupt and stop the VM-bind queues. Address
   spaces are deliberately not shut down cleanly, because the reset may have
   been caused by a stuck one (04).
4. Soft reset (3.3).
5. L2 power-up (3.4.1 or 3.5.4).
6. MMU post-reset: treat every address-space slot as unprogrammed, clear fault
   state, unmask the MMU interrupt (04).
7. Firmware post-reset: make the MCU address space active, reload **all**
   firmware sections (including read-only ones), set `MCU_CONTROL` = 2, wait
   for boot (1 s), re-initialize the global interface (05/06).
8. Clear "reset pending". Resume the scheduler, telling it whether the restart
   failed.
9. If the firmware did not boot: run the device-removal sequence
   (01 §1.4.5). The device becomes permanently unusable until the driver is
   reloaded.

## 3.10 Timestamps and cycle counter

- **[HW]** `TIMESTAMP` (64-bit) counts at the system architected-timer
  frequency. On Arm64 this is the CPU generic timer frequency (`CNTFRQ_EL0`).
  `CYCLE_COUNT` (64-bit) counts GPU core-clock cycles.
- **[HW]** Read both with the high/low/high rule from 02 §2.1.
- **[HW]** `TIMESTAMP_OFFSET` is added by hardware to the timestamp.
  **[POLICY]** Linux never writes `TIMESTAMP_OFFSET`. It leaves whatever value
  is there after reset or boot firmware. It only reads the register to report
  it.
  > OPEN QUESTION: Whether `TIMESTAMP_OFFSET` resets to 0 on soft reset or
  > power-domain cycling is not shown. If a driver wants GPU timestamps
  > aligned with the CPU counter, it could program `TIMESTAMP_OFFSET`, but
  > Linux does not, and userspace gets the offset through the query instead.
- **[ABI]** The timestamp query (10) powers the GPU for its duration and returns,
  as requested by flag: the timer frequency (architected-timer frequency, or 0
  if none), the `TIMESTAMP_OFFSET` value, the current `TIMESTAMP`, the current
  `CYCLE_COUNT`, a CPU monotonic or monotonic-raw time, and how long the
  sampling took. With no flags, it returns timestamp, offset and frequency.
  **[POLICY]** When more than one source or the duration is requested, local
  interrupts and preemption are disabled while sampling so the values are
  close together.
- Firmware timeouts in the global interface are given in timer ticks. The
  source is either the architected timer or, if its frequency is unknown, the
  GPU cycle counter at the `"core"` clock rate. A flag in the timeout word
  says which (06).
- **[ABI]** Driver interface version 1.6 and later enable the firmware's
  counter (cycle/timestamp sampling) at global-interface init (06).
  **[POLICY]** Per-job cycle and timestamp sampling is switched on by an
  administrator through a device attribute bitmask (bit 0 = cycles,
  bit 1 = timestamps). The scheduler then adds sampling instructions to
  jobs (07).

## 3.11 User mapping of the flush-ID page

**[ABI]** Userspace (Mesa) maps the `LATEST_FLUSH_ID` page (register window
offset `0x10000`) read-only. It compares the flush ID with the one recorded
at submission time so it can skip redundant cache flushes in command streams.

### 3.11.1 mmap offsets

| Item | Value |
|---|---|
| User-MMIO base, 64-bit processes | `1 << 56` |
| User-MMIO base, 32-bit processes | `1 << 43` (32-bit `mmap` page-offset limits) |
| Flush-ID page | user-MMIO base + 0 |

- The base is chosen per open file: 32-bit for 32-bit tasks on a 64-bit
  kernel, 64-bit otherwise. A per-file ioctl can set it to either of the two
  values (for CPU emulators). Any other value is rejected.
- mmap offsets at or above the file's base are user-MMIO. They are re-based to
  the kernel's canonical base before dispatch. Offsets below the base are
  buffer-object mappings (09).

### 3.11.2 Mapping rules

- The mapping must be shared and exactly one page long, with no write or exec
  permission. Write permission cannot be added later. The mapping is not
  inherited across fork, cannot be expanded, and is excluded from core dumps.
  Other user-MMIO offsets are rejected.
- Pages are filled in lazily, on fault, under a lock that also protects the
  power-state changes:
  - Device **active**: map the physical page at register base + `0x10000`,
    **uncached** (device memory).
  - Device **not active**: map a driver-owned, zero-filled RAM page whose
    first 32-bit word is **1**, with normal caching. **[ABI]** Userspace takes
    an unchanged flush ID as "no flush needed". The value 0 means "always
    flush", so the dummy page must not hold 0.
- On suspend (01 §1.4.3): under the lock, set the state to suspending and
  revoke **all** user-MMIO page mappings of the device **before** clocks or
  power go off. **[HW]** A later CPU access to an unpowered register page
  faults with an external abort.
- On resume: under the lock, revoke all user-MMIO mappings again (removing
  dummy-page mappings) and set the state to active, so the next access maps
  the real register page.

> OPEN QUESTION: With CPU page sizes larger than 4 KiB, the "one page"
> mapping at `0x10000` also covers the following registers (up to the
> doorbells at `0x80000` on 64 KiB pages). Linux does not handle this
> specially. It is not shown whether the hardware leaves that range empty.
