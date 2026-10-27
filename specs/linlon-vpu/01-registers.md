# 01: Host-Visible Register Map

All registers are 32 bits wide and little-endian, and are accessed with
32-bit loads and stores. Offsets are from the start of the VPU register
window (resource 1, `0x1424_0000` on Sky1). RCSU registers are in
`sky1-soc/02` §2.4.3 and are listed again in §1.5 only as a cross-reference.

Mnemonics are the register names used by the vendor driver's register
dump. Where a meaning is marked "inferred", it comes only from how the
vendor driver uses the register.

## 1.1 Global registers

| Offset | Name | Access | Meaning |
|---|---|---|---|
| `0x00` | `HARDWARE_ID` | RO | `[31:16]` product (00 §0.1.1), `[15:8]` revision, `[7:0]` patch |
| `0x04` | `ENABLE` | RW | Global job-scheduler enable. 1 = the hardware scheduler takes jobs from `JOBQUEUE`. Write 0 before changing `JOBQUEUE` and 1 afterwards |
| `0x08` | `NCORES` | RO | Number of implemented cores |
| `0x0C` | `NLSID` | RO | Number of LSIDs |
| `0x10` | `CORELSID` | RO | Which LSID each core is running: 4 bits per core, core *c* at `[4c+3:4c]`. An LSID number means busy. `0xF` means idle (inferred) |
| `0x14` | `JOBQUEUE` | RW | Hardware job queue, 4 entries of 8 bits (§1.3) |
| `0x18` | `IRQVE` | RO | Interrupt summary: bit *n* = LSID *n* has a pending engine-to-host interrupt |
| `0x24` | `CLKFORCE` | RW | `[7:0]` force core clocks on (one bit per core), `[8]` force scheduler clock on, `[9]` force core power on. Resets to 0. Must be 0 in normal operation |
| `0x30` | `SVNREV` | RO | Build/variant magic value (00 §0.1.1) |
| `0x34` | `FUSE` | RO | Feature fuses: `[0]` AFBC disabled, `[1]` RealVideo disabled, `[2]` VP8/VP9 disabled, `[3]` HEVC disabled |
| `0x40` | `PROTCTRL` | RW | Protection control (inferred). Not written by the vendor driver |
| `0x44` | `BUSCTRL` | RW | Bus control (§1.2) |
| `0x50` | `RESET` | WO | Write 1 for an engine soft reset |

- **[HW]** When a `FUSE` bit is set, the matching formats must not be
  offered. AFBC covers all AFBC frame formats. VPx covers VP8 and VP9.
- **[HW]** The same feature-disable bits appear as straps in RCSU
  `STRAP_PIN0[3:0]`.

## 1.2 `BUSCTRL` (`0x44`)

| Bits | Field | Values |
|---|---|---|
| `[1:0]` | AXI burst split | 0 = 128, 1 = 256, 2 = 512 (unit inferred to be bytes) |
| `[2]` | Reference-fetch mode | 0 = default, 1 = "mix" |

- **[POLICY]** The vendor driver writes `0x2` (split 512, default reference
  mode) on every power-up. Both fields are module parameters.

> OPEN QUESTION: The meaning of the "mix" reference mode and the exact unit
> of the split field are not documented.

## 1.3 `JOBQUEUE` (`0x14`) format

| Bits of entry *k* (`[8k+7:8k]`) | Meaning |
|---|---|
| `[3:0]` | LSID to run. `0xF` = empty entry |
| `[7:4]` | Number of cores to use, minus 1 |

- **[HW]** Entry 0 (`[7:0]`) is the head of the queue, the next job the
  hardware takes. Fill entries from 0 upwards, with no gaps. An empty queue
  reads `0x0F0F0F0F`.
- **[HW]** The vendor code also recognises an entry value `0xFF` as
  "obsoleted" and skips it.
- **[HW]** Procedure to add LSID *L* with *n* cores:
  1. Write `ENABLE` = 0.
  2. Read `JOBQUEUE`. If *L* is already in an entry, go to step 4.
  3. Put `(n−1) << 4 | L` in the first empty entry. You may reorder the
     entries (for example, by priority) and then write `JOBQUEUE`. If no
     entry is empty, the add fails.
  4. Write `ENABLE` = 1.
- **[HW]** To remove LSID *L*: write `ENABLE` = 0, rewrite `JOBQUEUE` with
  *L*'s entries taken out and the remaining entries packed towards entry 0.
  Fill the freed entries with `0x0F`. Then write `ENABLE` = 1.
- **[HW]** An LSID is **idle** when it appears in no `JOBQUEUE` entry and in
  no `CORELSID` field (check the `NCORES` lowest fields).
- **[POLICY]** Priority insertion: each session has a priority 0–3 (0
  highest). A newly added job moves ahead of queued jobs with a numerically
  larger priority. Each job it passes has its effective priority raised by
  one step, but never above 1, so that low-priority jobs cannot starve.

> OPEN QUESTION: Whether the hardware removes an entry from `JOBQUEUE` when
> it dispatches it to cores, and when `0xFF` entries appear, is implied by
> the code but not shown.

## 1.4 Per-LSID registers

LSID *n* (0–3) has a 64-byte block at **`0x200 + 0x40 × n`**.

| Offset | Name | Access | Meaning |
|---|---|---|---|
| `+0x00` | `CTRL` | RW | `[7:0]` core disallow mask (bit *c* = 1: core *c* must not run this LSID). `[11:8]` maximum number of cores for this LSID |
| `+0x04` | `MMU_CTRL` | RW | Level-1 page-table descriptor for this LSID (02 §2.2.4) |
| `+0x08` | `NPROT` | RW | Non-protected flag (inferred: 1 = normal, 0 = protected). Read back and saved, but never written, by the vendor driver |
| `+0x0C` | `ALLOC` | RW | LSID allocation: 0 = free, 1 = allocated non-protected, 2 = allocated protected |
| `+0x10` | `FLUSH_ALL` | WO | Any write (the vendor driver writes 0) invalidates this LSID's MMU translation caches |
| `+0x14` | `SCHED` | RW | 1 = the LSID may be scheduled, 0 = not |
| `+0x18` | `TERMINATE` | RW | Write 1 to terminate everything running on the LSID. Reads non-zero until termination is complete |
| `+0x1C` | `LIRQVE` | RW | Engine-to-host interrupt for this LSID. The firmware sets it. The host writes 0 to acknowledge, or 1 to raise it by software |
| `+0x20` | `IRQHOST` | RW | Host-to-engine doorbell. The host writes 1 to tell the firmware that new messages or buffers are queued |
| `+0x24` | `INTSIG` | ? | Unknown. Not used |
| `+0x2C` | `STREAMID` | ? | Unknown (00 §0.3). Not used |
| `+0x30` | `BUSATTR_0` | RW | AXI bus attribute word 0 |
| `+0x34` | `BUSATTR_1` | RW | AXI bus attribute word 1 |
| `+0x38` | `BUSATTR_2` | RW | AXI bus attribute word 2 |
| `+0x3C` | `BUSATTR_3` | RW | AXI bus attribute word 3 |

- **[POLICY]** `BUSATTR_0..3` default to `0x33`, `0x22`, `0x33`, `0x33`. The
  firmware `busattr` property overrides them.

> OPEN QUESTION: The bit layout of `BUSATTR_*`, for example the AXI
> cache/QoS fields and which traffic class each word controls, is not
> documented. The default values are in the vendor source with no
> explanation. `INTSIG` (`+0x24`) and offset `+0x28` are unexplained.

### 1.4.1 Attaching a session to an LSID

1. Read `ALLOC`. If it is non-zero, the LSID is in use. Stop.
2. Write `ALLOC` = 1 (or 2 for a protected session).
3. Write `TERMINATE` = 1 and poll until it reads 0.
4. Write `CTRL` = `((~core_mask & 0xFF) << 0) | ((ncores & 0xF) << 8)`.
5. Write `MMU_CTRL` = the session's level-1 descriptor, then write
   `FLUSH_ALL`.
6. Write `BUSATTR_0..3`.
7. Write `LIRQVE` = 0. Write `IRQHOST` = the saved doorbell state: 1 if the
   host rang the doorbell while the session had no LSID, else 0.
8. Issue a write barrier so that all of the above is visible, then write
   `SCHED` = 1.
9. Add the LSID to `JOBQUEUE` (§1.3).

### 1.4.2 Detaching a session from an LSID

1. If `ALLOC` = 0, there is nothing to do.
2. Write `SCHED` = 0.
3. Save `CTRL`, `MMU_CTRL`, `IRQHOST` and `NPROT` for a later re-attach.
4. Write `ALLOC` = 0.

- **[HW]** Detach a session only when its LSID is idle (§1.3), or after
  terminating it (§1.4.3).

### 1.4.3 Terminating a session

1. Remove the LSID from `JOBQUEUE`.
2. Write `TERMINATE` = 1. Wait until it reads 0.
3. Detach (§1.4.2).

- **[POLICY]** The wait for `TERMINATE` is bounded at 100 ms. Attach
  (§1.4.1 step 3) polls without a bound.

## 1.5 RCSU registers used (cross-reference)

| RCSU offset | Use in this spec |
|---|---|
| `0x21C` `PGCTRL` bit 0 | Set after reset on power-up. Cleared before power-down (00 §0.7) |
| `0x300` `STRAP_PIN0` `[31:28]` | ENPWOFF per-core auto power-off (00 §0.7.3). `[27:4]` power-switch timings: preserve them |
| `0x308` `STRAP_PIN2` `[7:4]` | Harvested cores (00 §0.7.1) |

> OPEN QUESTION: RCSU `STRAP_PIN1` (`0x304`) is named but never accessed.
