# 03: Device Control

## 3.1 Identification

Precondition: power domain on, `aclk` running (01 §1.6).

1. Read `GLB_CORE_ID` (GCU `0x004`). Accept product IDs `0x0071`, `0x0032`
   and `0x0060` (02 §2.3). **[HW]** Any other value means a different
   register model.
2. Record `GLB_ARCH_ID` and `GLB_CORE_INFO` for diagnostics.

- **[HW]** The AXI data bus width used for alignment rules is **16 bytes**
  on all three products (04 §4.6).

## 3.2 Soft reset and block discovery

Steps, **[HW]** order:

1. **Soft reset.** Write `GCU_CONTROL` = `0x00010000` (only `SRST` set; this
   also requests mode 0). Poll `GCU_CONTROL` until bit 16 reads 0.
   - **[POLICY]** Linux polls up to 100 times with a 1–10 ms sleep between
     polls and fails probe on timeout.
2. **Global capabilities.** Read `GLB_CORE_INFO`: slot count N = `[7:0]`,
   pipeline count P = `[10:8]`. **[HW]** P ≤ 2 for this register model.
   Reject larger values.
3. **Legacy or new layout.** Read `BLOCK_INFO` at offset `0xFE00`. If its
   type field (`[15:8]`) is `0x08`, the part has a PERIPH block:
   - line size = 4096 if `PERIPH_CONFIGURATION_ID[0]`, otherwise 2048
   - max lines = 4096
   - rich layers = 2 if `[4]`, otherwise 1
   - dual-link = `[8]`, integrated TBU = `[12]`

   Otherwise read `GCU_CONFIGURATION_ID0/1` for the same facts (02 §2.3).
4. **Walk the slots.** For k = 1 … N−1, at offset `k × 0x200`:
   1. Read `BLOCK_INFO`. If the type is `0xFF`, skip the slot.
   2. Read `PIPELINE_INFO`, then the valid-input IDs and output IDs it
      announces.
   3. Classify by type (02 §2.2.1) and record the block address under
      (pipeline, role). The pipeline and local index come from the instance
      field:
      - read layer: pipeline = instance ÷ 4, layer = instance mod 4
      - scaler: pipeline = instance ÷ 2, scaler = instance mod 2
      - every other per-pipeline block: pipeline = instance
5. **Per-block capabilities.** These are needed for validation (04, 05):

| Block | Capability | Source |
|---|---|---|
| Read layer | rich or simple | `LAYER_INFO[0]` |
| Read layer | max line (RGB) | No PERIPH: block `MAX_LINE_SIZE`. PERIPH with 4096-pixel lines: 4096. PERIPH with 2048-pixel lines: rich = 4096, simple = 2048 |
| Read layer | max line (YUV) | No PERIPH: `LAYER_INFO[31:16]`. PERIPH: half the RGB max line (0 for simple 2K layers) |
| Write-back, CU, splitter | max line | Block `MAX_LINE_SIZE` (no PERIPH) or the global line size |
| Scaler | max line | Block `MAX_LINE_SIZE` (no PERIPH) or 2048 |
| Merger | max merged width | Block `MAX_LINE_SIZE` (no PERIPH) or 4032 |
| IPS | YCbCr 4:2:0 output | `IPS_INFO[10]` |
| BS | dual-link | the global dual-link flag |

6. **Initial register state** written during discovery:
   - every read layer: `L_PALPHA` = `0xFFAA5500` (04 §4.4)
   - every scaler: `SC_CONTROL` = 0

- **[HW]** Each layer's and scaler's local index must be dense: layers
  0, 1, 2, 3 in order. Linux rejects a hole.
- **[HW]** Valid-input lists decide the topology. For example, a compositor
  lists the layers, scalers and merger of its own pipeline and, on a
  master, the other pipeline's compositor (00 §0.3). An input whose block
  was not found should be dropped from the list.

## 3.3 Operating mode changes

The GCU mode (02 §2.3.1) must match the set of running timing generators.

To change mode to M:

1. Read-modify-write `GCU_CONTROL[2:0]` = M, keeping the other bits.
2. Poll until `GCU_CONTROL[2:0]` reads back M.
   - **[POLICY]** 100 polls, 1–10 ms apart.

Mode transitions used:

| Event | From | To |
|---|---|---|
| First display on pipeline p turns on | 0 | 3 (p = 0) or 4 (p = 1) |
| Second display turns on | 3 or 4 | 5 |
| One of two displays turns off | 5 | 4 or 3 (the one still on) |
| Last display turns off | 3 or 4 | 0 |
| TBU connect / disconnect | 0 | 1 / 2, then back to 0 (§3.7) |

- **[HW]** Mode is per *timing generator*, keyed by the master pipeline's
  index. A slave pipeline's index is never set in the mode word when it only
  feeds the master's compositor.
- **[HW]** The `GCU_IRQ.MODE` interrupt signals completion. Linux enables
  it but polls the register instead.

## 3.4 Flush (config valid) and flip completion

- **[HW]** To commit pipeline p's programmed state, write `0x1` to
  `GCU_CONFIG_VALID0` (p = 0) or `GCU_CONFIG_VALID1` (p = 1).
- **[HW]** Flush the **master** pipeline's register. The same write commits
  the slave pipeline's blocks used by that display (Linux writes only the
  master's register for a display that uses both pipelines).
- **[HW]** When the hardware has taken the new state, it raises
  `GCU_IRQ.CVAL0` or `CVAL1`. The previous framebuffers may then be released
  and the page-flip event delivered.
- **[POLICY]** Linux flushes after every commit, even with no change, and
  waits up to 1 s for `CVAL`. On timeout it logs an error and continues.

> OPEN QUESTION: The source does not say whether a second `CONFIG_VALID`
> write before `CVAL` fires is safe. Linux always waits for completion
> before the next commit.

> OPEN QUESTION: `CVAL1` is decoded, but with a master/slave pair it is
> unclear whether a flush of pipeline 0 also raises `CVAL1`. Treat `CVALp`
> as belonging to the display whose master is pipeline p.

## 3.5 Interrupts

### 3.5.1 Decoding

On interrupt:

1. Read `GLB_IRQ_STATUS` (GCU `0x010`). If it is 0, the interrupt was not
   ours.
2. If bit 0 (GCU) is set:
   1. Read `GCU_IRQ_RAW_STATUS`.
   2. `CVAL0` → flip done for display 0. `CVAL1` → flip done for display 1.
   3. If `ERR`: read `GCU_STATUS`. If `MERR` is set, report a mode error and
      clear `MERR` (02 §2.2 note).
   4. Write the raw value read in step 1 to `GCU_IRQ_CLEAR`.
3. For each pipeline p whose LPU, ATU, CU or DOU summary bit is set:
   - **LPU** (bit 8 + p):
     1. Read `LPU IRQ_RAW_STATUS`. `EOW` → write-back frame finished.
        `IBSY` and `OVR` → error events.
     2. If any of `ERR`, `IBSY` or `OVR` is set: read `LPU_STATUS` and clear
        whichever of `AXIE`, `ACE0–3`, `FEMPTY` and `FFULL` are set. Then read
        `LPU_TBU_STATUS` and clear whichever of `TCF`, `TTNG`, `TITR`,
        `TEMR` and `TTF` are set.
     3. Write the raw value to `LPU IRQ_CLEAR`.
   - **CU** (bit 16 + p):
     1. Read `CU IRQ_RAW_STATUS`. `OVR` → overrun.
     2. If `ERR` or `OVR`: read `CU_STATUS`, drop bit 31, report `CPE`,
        `ZME` and `CFGE`, and write zeros over the bits that were set.
     3. Write the raw value to `CU IRQ_CLEAR`.
   - **DOU** (bit 24 + p):
     1. Read `DOU IRQ_RAW_STATUS`. `PL0` → vblank. `UND` → underrun.
     2. If `ERR` or `UND`: read `DOU_STATUS` and clear whichever of
        `DRIFTTO`, `FRAMETO`, `TETO` and `CSCE` are set.
     3. Write the raw value to `DOU IRQ_CLEAR`.

- **[HW]** Read the *raw* status, not the masked status: Linux takes events
  from `IRQ_RAW_STATUS`, so an unmasked raw bit (for example `PL0` with
  vblank off) can still appear and is cleared with the rest.
- **[POLICY]** All error events are only logged. No recovery is attempted.

### 3.5.2 Event → action

| Event | Source | Action |
|---|---|---|
| Vblank | DOU `PL0` of the master pipeline | Vblank counter and timestamp |
| Flip done | GCU `CVALp` | Complete the pending commit for the display mastered by pipeline p |
| Write-back done | LPU `EOW` of the master pipeline | Signal the write-back job's completion fence |
| Underrun | DOU `UND` | Error. The display missed pixels |
| Others | | Error log |

### 3.5.3 Masks

| Block | Bits enabled at resume |
|---|---|
| GCU | `CVAL0`, `CVAL1`, `MODE`, `ERR` |
| LPU (each pipeline) | `IBSY`, `ERR`, `EOW` |
| CU (each pipeline) | `OVR`, `ERR` |
| DOU (each pipeline) | `UND`, `ERR` |

- **[HW]** `DOU.PL0` (vblank) is masked or unmasked separately, on the
  master pipeline's DOU, when vblank reporting is wanted.
- **[HW]** `PL0` fires at the line in `BS_PROG_LINE` (06 §6.4).
- Masking is done by read-modify-write of `IRQ_MASK`. Suspend masks the
  same bits.

## 3.6 Disabling a display (two-phase)

**[HW]** The DPU does not latch shadow registers once the display output is
off. If the timing generator and the other blocks are disabled in one flush,
the other blocks' disable never takes effect. The disable is therefore done
in two flushes:

1. **Phase 1.** Disable every active block of the slave pipeline (if used),
   then of the master pipeline, **except** the master's IPS and BS. Flush
   the master pipeline (§3.4) and wait for `CVAL`.
2. **Phase 2.** Disable the master's IPS and BS. Flush again and wait for
   `CVAL`.
3. Turn vblank reporting off. Change the operating mode (§3.3). Gate the
   pipeline's `pxclk`, and `aclk` if no display remains.

How each block is disabled:

| Block | Disable action |
|---|---|
| Read layer | Clear `L_CONTROL.EN` (read-modify-write) |
| Write-back layer | `LW_INPUT_ID0` = 0, then clear `LW_CONTROL.EN` |
| Compositor | `CU_CONTROL` = 0. For each input n: `CU_INPUT_ID<n>` = 0 and `CU_INPUT<n>_CONTROL` = `0x0000FF00` (alpha field 0xFF, disabled) |
| Scaler, splitter, merger, IPS | `CONTROL` = 0 and every `INPUT_ID<i>` = 0 |
| BS | Clear `BS_CONTROL.EN` (read-modify-write) |

> OPEN QUESTION: If phase-2 `CVAL` never arrives because the timing has
> stopped, Linux just times out after 1 s. The source does not show whether
> `CVAL` is reliably raised for the phase that turns `BS` off.

## 3.7 Integrated TBU (IOMMU) connect / disconnect

Applies only when the integrated-TBU capability bit is set (02 §2.3, 2.4)
**and** the platform uses an IOMMU for the DPU.

Connect (at resume), **[HW]** order:

1. Read-modify-write `GCU_CONTROL[2:0]` = 1 (TBU connect).
2. Poll `GCU_STATUS` until `TCS0` is set, and `TCS1` too if there are 2
   pipelines.
   - **[POLICY]** 100 polls, 1 ms apart (1–1 ms sleep).
   - On timeout: set mode 0 and report failure.
3. In every pipeline's LPU, set `LPU_TBU_CONTROL.TLBPEN` (bit 16).

Disconnect (at suspend):

1. Read-modify-write `GCU_CONTROL[2:0]` = 2 (TBU disconnect).
2. Poll until `TCS0` (and `TCS1`) read 0. On timeout set mode 0.

> OPEN QUESTION: Linux leaves `GCU_CONTROL[2:0]` at 1 after a successful
> connect and at 2 after a disconnect. It does not return to mode 0. The
> next display enable then changes directly from mode 1 to 3/4. Whether the
> hardware expects an explicit return to mode 0 first is unknown.

## 3.8 Enabling a display and per-frame update

Display enable, **[HW]** order:

1. Runtime-resume the device (01 §1.6).
2. Change the operating mode (§3.3) to include the master pipeline.
3. If this is the first display: set the `aclk` rate (01 §1.2) and enable it.
4. Set the master's `pxclk` to the adjusted pixel rate (06 §6.4) and enable
   it.
5. Enable vblank (`DOU.PL0` mask).
6. Program every block of the display's data path (04–06) and flush (§3.4).

Per-frame update (no mode change):

1. For every block that is active in the new or old state: program it if it
   is active in the new state, otherwise disable it (§3.6 table).
2. If a write-back job exists, arm it.
3. Flush the master pipeline and wait for `CVAL`.

- **[POLICY]** Linux reprograms every block in the data path on every
  commit, but compositor inputs only when they change (05 §5.1). When the
  pipeline was disabled and re-enabled by a modeset, every input is treated
  as changed.
