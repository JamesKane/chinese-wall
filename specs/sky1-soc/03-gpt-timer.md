# 03: General-Purpose Timer (GPT)

The Sky1 GPT is a memory-mapped timer in the FCH (peripheral) domain. Each
instance is a **pair** of 32-bit timers that can be chained into one 64-bit
free-running counter with a 64-bit compare. It can serve as the system's
**broadcast clock-event device**: when CPUs enter deep idle states in which
their per-core Arm generic timers stop, a global timer must wake them.

## 3.1 Instances and attachment

| Instance | Base | Size | Interrupt (GIC SPI / ACPI GSIV) |
|---|---|---|---|
| GPT 0 | `0x04110000` | 8 KiB | 467 / 499 |
| GPT 1 | `0x04112000` | 8 KiB | 468 / 500 |
| GPT 2 | `0x04114000` | 8 KiB | 469 / 501 |
| GPT 3 | `0x04116000` | 8 KiB | 470 / 502 |

- **[HW]** ACPI `_HID` `CIXH1007`. DT compatible `cix,sky1-gpt`. Interrupts
  are level-high.
- **[HW]** Clocks (SCMI clock IDs, section 01 §1.2.4): APB clock
  `CLK_TREE_FCH_TIMER_APB` (261) and counter (functional) clock
  `CLK_TREE_FCH_TIMER_FUN` (242). The counter ticks at the functional
  clock's rate.
- **[HW]** Clock names: the vendor DT and the DSDT CLKT use
  `fch_timer_apb_clk` and `fch_timer_func_clk`. (The Sky1-Linux driver
  first asks for `apb` and `func`, then falls back to the vendor names.)
  The APB clock is optional (may already be on); the functional clock is
  required and must report a non-zero rate.
- **[HW]** Reset: FCH controller index 8 (`0x04160008` bit 20, active-low,
  section 02 §2.1.2), shared by all four instances. It must be released.
- All four instances are disabled in the Orion O6 devicetree; nothing in
  the vendor tree uses them for timekeeping.

## 3.2 Register map

Each instance has two timers: **even** at instance base + `0x0000` (low 32
bits) and **odd** at instance base + `0x1000` (high 32 bits). Both have the
same register layout:

| Offset | Name | Access | Description |
|---|---|---|---|
| `0x00` | `LDINIT` | RW | Initial (reload) value (not used in free-run mode) |
| `0x1C` | `FREE_CNT` | RO | Free-running counter value |
| `0x20` | `COMP` | RW | Compare value |
| `0x24` | `TCTL` | RW | Control (§3.2.1) |
| `0x30` | `INTSTAT` | RO | Interrupt status |
| `0x34` | `INTCLR` | WO | Interrupt clear: write `0x20` (bit 5) to clear the free-run compare interrupt |

### 3.2.1 `TCTL`

| Bits | Meaning |
|---|---|
| `[1:0]` (width inferred) | Mode: `2` = free-running |
| `[6]` | Interrupt enable |
| `[12]` | 64-bit chaining: set on the **odd** timer so it counts carries from the even timer |
| `[15]` | Free-run enable |
| `[17]` | Timer enable |

> OPEN QUESTION: Other mode encodings, the meaning of the remaining
> `INTSTAT` bits, and whether `LDINIT` matters in free-run mode are not
> visible. Only free-run mode is used.

## 3.3 Initialisation

**[HW]** order:

1. Release the reset, enable the APB clock, enable the functional clock and
   read its rate.
2. Even timer: write `TCTL` = free-run enable | mode 2 | interrupt enable
   (`0x8042`).
3. Odd timer: write `TCTL` = free-run enable | 64-bit | mode 2 | interrupt
   enable (`0x9042`).
4. Even timer: write `TCTL` again with timer enable added (`0x28042`).
5. Odd timer: write `TCTL` again with timer enable added (`0x29042`).

The enable bit must be set in a separate, final write for each timer.

## 3.4 Reading the 64-bit counter

**[HW]** The two halves are read separately, so a carry can happen between
the reads:

1. Read the high word (odd `FREE_CNT`).
2. Read the low word (even `FREE_CNT`).
3. Read the high word again. If it changed, repeat from step 2 using the
   new high word.
4. Counter = high × 2³² + low.

The counter is 64 bits wide, never wraps in practice, and counts up.

## 3.5 One-shot events (clock-event use)

- **[HW]** To schedule an interrupt at counter value T: write the **high**
  word of T to the odd timer's `COMP`, then the **low** word to the even
  timer's `COMP`. (High first, so the 64-bit compare does not briefly
  match a wrong value.)
- **[HW]** The interrupt is level and stays asserted until cleared: write
  `0x20` to the **odd** timer's `INTCLR`. Interrupt enable is the odd
  timer's `TCTL[6]`.
- **[HW]** If T is already in the past when the write lands, assume no
  interrupt is generated for it (inferred: the driver reports such an event
  as missed). After programming, re-read the counter: if it is
  already ≥ T, treat the event as missed and handle it in software.
- **[POLICY]** To stop or park the timer, Linux disables the odd timer's
  interrupt, sets the compare to (now − 3) so it cannot match soon, and
  clears any pending interrupt. It re-enables the interrupt when one-shot
  mode is selected.
- **[POLICY]** Linux accepts deltas from 1 to 2³² − 1 ticks, rates the
  device below the architected timer, and allows the interrupt to be
  routed to any CPU.
- **[HW]** On resume from a state that lost the timer's context, repeat
  §3.3, then park it as above.

## 3.6 Use as an idle-broadcast timer (ACPI `_LPI`)

- **[HW]** When the ACPI processor `_LPI` states declare that the per-CPU
  architected timer stops (the state's flags in `_LPI` / the generic timer
  being in the power domain that is turned off), the OS must program a
  global wake-up timer before a CPU enters that state. The GPT is a
  suitable global timer: it is in the FCH domain, not in a CPU cluster.
- **[HW]** The broadcast interrupt must be routable to a CPU that stays
  awake, or able to wake a CPU from the idle state through the GIC.

> OPEN QUESTION: Whether the FCH domain and the GPT functional clock stay
> on in Sky1's deepest CPU idle states (and in system suspend) is not
> stated. Sky1-Linux registers the GPT as an ordinary clock-event device
> and does not mark it as always-on or test it under `_LPI`. Verify on
> hardware before relying on it for broadcast.

> OPEN QUESTION: Whether the GTDT also describes a memory-mapped Arm
> generic timer frame on Sky1 (which would be the conventional broadcast
> source) could not be checked; no ACPI tables were available to this
> extraction.
