# 01: Platform Integration (Devicetree, Clocks, Supplies, Power, DVFS)

This section covers what a driver must get from the platform before it can
touch the GPU registers, and how those platform resources are sequenced
across probe, runtime suspend/resume, system suspend and removal. Register
names follow [02-register-map](02-register-map.md). GPU-internal sequences
(reset, L2 power, interrupts) are in [03-gpu-control](03-gpu-control.md).

## 1.1 Devicetree binding summary

Source: the public binding `arm,mali-valhall-csf.yaml` (GPL-2.0-only OR
BSD-2-Clause).

### 1.1.1 Properties

| Property | Required | Meaning |
|---|---|---|
| node name | – | `gpu@<unit-address>` |
| `compatible` | yes | One SoC string followed by `"arm,mali-valhall-csf"`. SoC strings in the binding: `mediatek,mt8196-mali`, `nxp,imx95-mali` (Mali-G310), `rockchip,rk3588-mali`. The GPU model and revision are discoverable from `GPU_ID`, so the generic string is enough to drive the hardware |
| `reg` | yes | Exactly one MMIO region (the register window in 02) |
| `interrupts` | yes | Three interrupts, in this order: job, MMU, GPU |
| `interrupt-names` | yes | `"job"`, `"mmu"`, `"gpu"` (same order) |
| `clocks` | yes | 1 to 3 clocks |
| `clock-names` | – | First is `"core"`. Optional second is `"coregroup"` or `"stacks"`. Optional third is `"stacks"` |
| `nvmem-cells`, `nvmem-cell-names` | no | One cell named `"shader-present"`: bitmask of functional shader cores (1.7) |
| `mali-supply` | no | Main GPU supply (scaled with frequency) |
| `sram-supply` | no | GPU SRAM supply |
| `operating-points-v2`, `opp-table` | no | Frequency/voltage table (1.5) |
| `power-domains`, `power-domain-names` | no | 1 to 5 power domains |
| `#cooling-cells` | no | Must be 2. The GPU can act as a thermal cooling device |
| `dynamic-power-coefficient` | no | u32 in µW/MHz/V². Dynamic power = coefficient × V² × f(MHz). Used by an energy-model-based thermal governor |
| `dma-coherent` | no | GPU accesses are I/O-coherent with CPU caches (1.6) |

### 1.1.2 Per-SoC constraints in the binding

| SoC | Constraints |
|---|---|
| `rockchip,rk3588-mali` | Exactly 3 clocks. `mali-supply` required. At most 1 power domain, no `power-domain-names`. No nvmem cells |
| `mediatek,mt8196-mali` | Clocks are exactly `"core"`, `"stacks"`. `nvmem-cells`/`nvmem-cell-names` and one `power-domains` entry required. No `mali-supply`, `sram-supply` or `operating-points-v2` (DVFS is done by the power-domain provider) |

### 1.1.3 What the driver actually consumes

- **[HW]** The `"job"`, `"mmu"` and `"gpu"` interrupts are looked up **by name**,
  not by index.
- **[HW]** On arch 14+ the PWR interrupt block (02 §2.5) has no interrupt of its
  own. It shares the `"gpu"` line with the GPU-control interrupt block. Both
  handlers must be installed as shared handlers on that line, and each must
  check its own `INT_STAT` to decide whether the interrupt is its own (03 §3.8).
- **[HW]** The first clock (`"core"`, or the unnamed first entry) is mandatory.
  `"stacks"` and `"coregroup"` are optional. A missing optional clock is not an
  error.
- **[POLICY]** No `resets` property exists in the binding. Linux does not use a
  reset controller for the GPU. Reset is done through GPU registers only (03).

## 1.2 RK3588 / RK3588S node

From the RK3588 base dtsi. Board files enable the node and add supplies.

| Item | Value |
|---|---|
| `reg` | `0xFB000000`, size `0x200000` (2 MiB) |
| `interrupts` | GIC SPI 92 (job), SPI 93 (MMU), SPI 94 (GPU). All level-high |
| `clocks` | CRU `CLK_GPU` → `"core"`; CRU `CLK_GPU_COREGROUP` → `"coregroup"`; CRU `CLK_GPU_STACKS` → `"stacks"` |
| `assigned-clocks` | SCMI clock `SCMI_CLK_GPU` assigned 200 MHz at boot |
| `power-domains` | `RK3588_PD_GPU` (a single domain) |
| `#cooling-cells` | 2 |
| `dynamic-power-coefficient` | 2982 |
| `dma-coherent` | absent (the GPU is **not** I/O-coherent on RK3588) |
| `status` | `"disabled"` in the SoC file. Boards set `"okay"` |
| `mali-supply` | Board-supplied, usually the PMIC rail `vdd_gpu_s0` |
| `sram-supply` | Usually absent. A few boards set it to `vdd_gpu_mem_s0`. On several boards this is the same physical regulator as `vdd_gpu_s0` (two labels on one PMIC output) |

### 1.2.1 RK3588 GPU power domain

The SoC power-domain node for `RK3588_PD_GPU`:

- lists the same three CRU clocks (`CLK_GPU`, `CLK_GPU_COREGROUP`,
  `CLK_GPU_STACKS`). The power-domain controller needs these running while
  it switches the domain.
- has four bus QoS register blocks (GPU master ports M0–M3) attached. The
  power-domain controller must save them before power-off and restore them
  after power-on.
- has `domain-supply = <vdd_gpu_s0>` on boards. **[HW]** The GPU rail must be
  enabled before the GPU power domain is switched on.

> OPEN QUESTION: The exact RK3588 PMU register sequence for the GPU power
> domain (idle request, ack, QoS save/restore) belongs to the Rockchip
> power-domain controller, not to this GPU driver, and is not covered here.
> A BSD port needs an equivalent power-domain driver.

### 1.2.2 RK3588 OPP tables

Standard RK3588 / RK3588S (`opp-microvolt` is target / min / max in µV):

| Frequency | Voltage target | min | max |
|---|---|---|---|
| 300 MHz | 675000 | 675000 | 850000 |
| 400 MHz | 675000 | 675000 | 850000 |
| 500 MHz | 675000 | 675000 | 850000 |
| 600 MHz | 675000 | 675000 | 850000 |
| 700 MHz | 700000 | 700000 | 850000 |
| 800 MHz | 750000 | 750000 | 850000 |
| 900 MHz | 800000 | 800000 | 850000 |
| 1000 MHz | 850000 | 850000 | 850000 |

RK3588J (industrial variant): 300, 400, 500, 600, 700 MHz, all at
750000 / 750000 / 850000 µV. The maximum is 700 MHz.

### 1.2.3 RK3588 thermal zone

The `gpu-thermal` zone uses TSADC channel 5 with passive polling every 100 ms.
Trips: passive at 85 °C (2 °C hysteresis), mapped to the GPU cooling device
with no limits. Critical at 115 °C.

## 1.3 SoC match table and per-SoC data

| Compatible | Per-SoC data |
|---|---|
| `mediatek,mt8196-mali` | Custom L2 address hash: enable = yes, `ASN_HASH[0..2]` = `0x0000000B`, `0x0000000E`, `0x00000000` |
| `rockchip,rk3588-mali` | none |
| `arm,mali-valhall-csf` | none (generic fallback, so i.MX95 matches here) |

- **[HW]** When the SoC needs a custom ASN hash, the driver writes the three
  `ASN_HASH` registers and sets `L2_CONFIG` bit 24 before every L2 power-up
  (03 §3.4.1). The hash affects how the L2 slices interleave physical
  addresses, which must match the SoC interconnect.
- **[POLICY]** Linux rejects (logs and skips) a custom ASN hash on arch major
  < 11.
- No RK3588-specific quirks exist.

> OPEN QUESTION: The meaning of the MT8196 hash values `0xB`, `0xE`, `0x0` is
> not documented in the source. Treat them as opaque constants.

## 1.4 Resource bring-up and tear-down order

### 1.4.1 Probe

The order below is what Linux does. Where a step is **[HW]**, the ordering
matters to the hardware. Other orderings are **[POLICY]**.

1. **Get clocks.** `"core"` is required. `"stacks"` and `"coregroup"` are
   optional. Clocks are not enabled yet.
2. **Attach power domains.** If the platform has not already attached a single
   power domain to the device, attach all listed domains and link them so they
   all turn on and off together with the device.
3. **Set up DVFS (1.5)**:
   1. If the power-domain provider already supplies an operating-point table
      (a performance domain, as on MT8196), use it. Do not attach regulators
      or parse the DT table.
   2. Otherwise attach `mali-supply` as the voltage regulator for the
      operating points (absent is allowed), then load the DT OPP table.
   3. Get and **enable `sram-supply`** if present. It stays enabled until the
      device is removed.
   4. Read the current `"core"` clock rate, choose the OPP that the governor
      recommends for it, and apply that OPP. This sets the rate and voltage and
      **enables `mali-supply`**. **[HW]** The GPU rail must be on before the
      GPU is accessed. **[POLICY]** Linux also does this so a boot-time
      "unused regulator" cleanup does not switch the rail off.
   5. Record the highest OPP frequency as the device's maximum rate (reported
      to userspace through per-file statistics).
4. **Map the register window** (`reg` index 0) and remember its physical base.
   The physical base is needed for the user flush-ID mapping (03 §3.11).
5. **Power up** through the runtime-resume path (1.4.2). This powers the
   domain(s) and enables the clocks.
6. **GPU identification and feature probe** (03 §3.1), including the
   `shader-present` override (1.7).
7. Arch 14+: set up the PWR interrupt block on the `"gpu"` line (03 §3.8).
8. Set up the GPU-control interrupt block on the `"gpu"` line. Set the DMA
   addressing limit to the PA width in `MMU_FEATURES[15:8]` and the maximum
   DMA segment size to unlimited. **[HW]** The GPU cannot address memory above
   that PA width.
9. Choose the coherency protocol (1.6).
10. MMU, firmware and scheduler setup (04–07). Firmware setup installs the
    `"job"` interrupt and powers the L2 (03 §3.4) before loading the MCU
    firmware.
11. Enable the reset/recovery worker (03 §3.9).
12. **[POLICY]** Enable runtime autosuspend with a 50 ms delay (about 3
    frames at 60 Hz), register the device, and drop the probe-time power
    reference.

- **[POLICY]** Linux issues no GPU soft reset during probe. It relies on the
  GPU being in its reset state after the power domain turns on.

> OPEN QUESTION: Whether a GPU left powered and running by a bootloader needs
> a soft reset before the first L2 power-up is not covered. A defensive driver
> may issue one (03 §3.3) before step 10.

### 1.4.2 Runtime resume (power up)

1. **[HW]** The power domain(s) must be on first. On Linux the power-domain
   framework turns the domain on before the driver's resume step runs. On
   RK3588 this needs `vdd_gpu_s0` on and the three CRU clocks available to the
   power-domain controller.
2. Enable clocks in this order: `"core"`, then `"stacks"`, then `"coregroup"`
   (each if present).
   > OPEN QUESTION: Whether this clock order is a hardware requirement is
   > unknown. Disable uses the reverse order.
3. Resume the DVFS governor and reset its busy/idle accounting.
4. If the device was fully initialized, restore the GPU (03 §3.7.2): PWR and
   GPU interrupt blocks, L2 power-up, MMU, then firmware boot, then the
   scheduler.
5. Re-enable the user flush-ID mapping (03 §3.11).

Supplies are **not** touched on resume. They stay on for the device lifetime
(1.5.3).

On failure, clocks are disabled in reverse order, the device stays
suspended, and a "recovery needed" flag is set. **[POLICY]** On the next
power-up request, Linux clears the runtime-PM error state so a later resume
can be tried again.

### 1.4.3 Runtime suspend (power down)

1. Mark the device as suspending and revoke all user flush-ID mappings
   (03 §3.11). **[HW]** This must happen before clocks or power go away. A CPU
   access to the GPU register window while the GPU is unclocked or unpowered
   raises an external abort.
2. If the device was fully initialized: cancel any pending reset work, then
   suspend the scheduler, firmware, MMU, GPU (soft reset or L2 off) and PWR
   blocks, in that order (03 §3.7.1).
3. Suspend the DVFS governor.
4. Disable clocks in this order: `"coregroup"`, `"stacks"`, `"core"`.
5. **[HW]** The power domain turns off after the clocks are disabled. On Linux
   the framework does this after the driver's suspend step returns.

### 1.4.4 System suspend / resume

- **[POLICY]** Linux has no separate system-sleep logic. System suspend forces
  the runtime-suspend path (1.4.3) if the device is active. System resume
  forces the runtime-resume path (1.4.2) if the device was active before
  suspend.
- On RK3588 the PMIC switches off `vdd_gpu_s0` in system suspend
  (`regulator-off-in-suspend`). All GPU state is lost. Resume must therefore
  take the full path, including the L2 power-up and firmware boot.

### 1.4.5 Removal

1. Take a power reference so the GPU is up.
2. Mark the device unplugged, so later hardware access through user paths is
   refused.
3. Shut down the scheduler and firmware (MCU disabled, `"job"` interrupt
   masked, L2 powered off), then the MMU, the GPU-control interrupt and the
   PWR interrupt (03).
4. Disable autosuspend and power down synchronously (1.4.3). Hardware steps
   are skipped because the device is already unplugged. Only the governor
   suspend and the clock disable run.
5. Release resources. Removing the OPP table **disables `mali-supply`**.
   `sram-supply` is disabled when the device is released.

The same unplug sequence also runs if firmware fails to boot after a reset
(03 §3.9). Calls to it must be serialized, and a second caller waits for the
first to finish.

## 1.5 DVFS (frequency and voltage scaling)

### 1.5.1 Mechanism

- **[HW]** Frequency changes go through the operating-point table: pick an OPP,
  then set the `"core"` clock rate and the `mali-supply` voltage. Usual
  ordering: when raising frequency, raise the voltage first. When lowering
  frequency, lower the clock first, then the voltage. The silicon needs at
  least the OPP voltage at a given frequency.
- Only the `"core"` clock rate is set. On RK3588, `"coregroup"` and `"stacks"`
  are only gated on and off, not rate-scaled by this driver.
  > OPEN QUESTION: Whether the RK3588 CRU keeps `CLK_GPU_COREGROUP` and
  > `CLK_GPU_STACKS` in a fixed ratio with `CLK_GPU` is not shown in the GPU
  > driver.
- **[POLICY]** The OPP layer supports only one regulator here. If
  `sram-supply` is a separate regulator, Linux assumes it is coupled to
  `mali-supply` through the regulator framework's coupling logic. The coupling
  keeps the two voltages within the allowed spread. The GPU driver does not
  set the SRAM voltage directly.
- If a power-domain provider owns DVFS (MT8196), the frequency request goes to
  that provider as a performance-state change.

### 1.5.2 Governor (all [POLICY])

| Parameter | Linux value |
|---|---|
| Governor | Simple on-demand utilization governor |
| Sampling period | 50 ms (delayed timer, so it does not wake an idle CPU just to sample) |
| Up-threshold | 45 % busy → go to the maximum frequency |
| Down-differential | 5 % |
| Initial frequency | The OPP recommended for the clock rate at probe |

Utilization accounting:
- Time is split into *busy* and *idle* intervals. Each sample reports
  `busy_time` and `total_time = busy + idle` since the previous sample, then
  resets both counters.
- The scheduler marks the GPU *busy* when a job is submitted to hardware, or
  when a scheduling pass leaves at least one non-idle command-stream group
  resident. It marks the GPU *idle* when every resident group is idle (07).
- The current frequency is read back from the `"core"` clock.
- On resume, accounting is reset.

### 1.5.3 Regulator lifetime

- **[POLICY]** Linux keeps `mali-supply` and `sram-supply` enabled from probe
  until removal. Runtime suspend does not switch them off. Power saving at
  runtime comes from the power domain and clock gating only. A BSD driver may
  switch the supplies off at runtime suspend. If it does, it must turn them
  back on (with the OPP voltage) before the power domain comes up.

### 1.5.4 Thermal

- **[POLICY]** The GPU registers as a thermal cooling device using an energy
  model from `dynamic-power-coefficient`. The cooling device caps the maximum
  OPP. If registration fails, the driver logs it and carries on.

## 1.6 Coherency protocol selection

1. Start with protocol **none** (`COHERENCY_PROTOCOL` value 31).
2. If the firmware description marks the device `dma-coherent`:
   1. Read `COHERENCY_FEATURES`. If bit 0 (ACE-Lite) is set, select ACE-Lite
      (value 0).
   2. Otherwise **fail probe**. **[POLICY]** The platform claims coherency
      but the GPU cannot provide it.
3. **[HW]** Full ACE (value 1) is never selected. CSF GPUs do not support ACE.
4. **[HW]** The selected value is written to `COHERENCY_PROTOCOL` before every
   L2 power-up on arch 10–13 (03 §3.4.1).
5. **[ABI]** The selected value is reported to userspace (0 = ACE-Lite,
   1 = ACE, 31 = none, the same encoding as the register). Coherency also
   decides page-table memory attributes and whether CPU cache maintenance is
   needed for buffer objects (04, 09).

RK3588 is not `dma-coherent`, so it uses protocol 31 and needs explicit CPU
cache maintenance.

## 1.7 `shader-present` nvmem override

- After reading the shader-present mask from hardware (03 §3.1), look for an
  nvmem cell named `"shader-present"`.
- The cell holds a little-endian unsigned integer of variable length, up to
  64 bits. If the cell exists, its value **replaces** the hardware
  shader-present mask completely, not as an AND.
- No cell is not an error. Any other read error (including "try again later"
  while the nvmem provider is not ready) fails probe.
- The resulting mask is the one reported to userspace and used to build the
  shader core masks given to firmware (06). This lets a SoC fuse off broken
  or binned cores that the GPU still reports as present.
- Required on MT8196. Not allowed by the binding on RK3588.
