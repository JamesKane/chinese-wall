# 01: GPIO Controllers

Sky1 uses the Cadence GPIO IP (revision r1p02) for its general-purpose
I/O. Each instance drives up to 32 lines and raises one interrupt into the
GIC. This section gives the register map, the interrupt and wake model the
Sky1 platform needs, and the ACPI and devicetree description of the
instances.

Interrupt numbers below are GIC shared peripheral interrupt (SPI) numbers.
In ACPI `_CRS` the same interrupt appears as GSIV = SPI + 32
(sky1-soc 01 §1.1.1).

## 1.1 Instances

**[HW]** Seven controllers: four in the FCH (peripheral, S0) domain, three
in the S5 (always-on) domain. Each register window is 4 KiB in devicetree.
The ACPI `_CRS` of the FCH instances declares 64 KiB; the S5 instances
declare 4 KiB. Only the first 0x30 bytes are used.

| Controller | Base | IRQ (GIC SPI / GSIV) | Lines (`ngpios`) | Clock | Reset |
|---|---|---|---|---|---|
| FCH GPIO 0 | `0x04120000` | 304 / 336 | 32 | SCMI `FCH_GPIO_APB` (262) | FCH index 26 (`apb_reset`) |
| FCH GPIO 1 | `0x04130000` | 305 / 337 | 32 | same | same |
| FCH GPIO 2 | `0x04140000` | 306 / 338 | 32 | same | same |
| FCH GPIO 3 | `0x04150000` | 307 / 339 | 17 | same | same |
| S5 GPIO 0 | `0x16004000` | 372 / 404 | 32 | 100 MHz fixed (DT); CLKT 262 under ACPI | none |
| S5 GPIO 1 | `0x16005000` | 373 / 405 | 10 | same | none |
| S5 GPIO 2 | `0x16006000` | 374 / 406 | 10 | same | none |

- **[HW]** The GIC interrupt is level-high. It is the logical OR of all
  unmasked pending line interrupts of that controller.
- **[HW]** All four FCH controllers share **one** APB clock (SCMI clock
  262) and **one** reset line (FCH reset controller index 26, sky1-soc 02
  §2.1.2). Asserting that reset resets all four. Treat both as shared
  resources with reference counting.
- **[HW]** The S5 controllers sit in the always-on domain and keep their
  state across system suspend. They are the controllers used for wake
  sources (PD controllers, EC, RTC, power button on the reference boards).
- **[HW]** The vendor devicetree gives the FCH GPIO clock as 200 MHz
  and the S5 GPIO clock as 100 MHz. Under ACPI, CLKT lists clock 262 for
  both FCH and S5 instances. The controller has no internal debounce or
  timing that depends on the clock rate. The clock only needs to be running.

> OPEN QUESTION: The ACPI-derived tables list clock 262 (`FCH_GPIO_APB`)
> for the S5 instances too. Whether gating SCMI clock 262 affects the S5
> controllers, or whether the entry is only informative, is not known. Do
> not gate clock 262 while any S5 GPIO is in use.

### 1.1.2 ACPI identification

| `_HID` | Instances in the DSDT (from the ACPI-derived tables) |
|---|---|
| `CIXH1003` | Six: `0x04130000`, `0x04140000`, `0x04150000`, `0x16004000`, `0x16005000`, `0x16006000` |
| `CIXH1002` | One: `0x04120000` (IRQ GSIV 336) |

- **[ABI]** Both HIDs select the same programming model. A driver must
  match both. The HID does not tell you the power domain; use the base
  address.
- **[ABI]** The public mapping notes call `CIXH1002` the "S5 domain"
  controller, but the extracted DSDT gives it the FCH GPIO 0 address. The
  address is what counts.
- **[ABI]** The DSDT device names seem to be `GPI0` … `GPI6`. Consumers
  refer to a controller by its ACPI path in `GpioIo` / `GpioInt`.

> OPEN QUESTION: The mapping of the names `GPI4`/`GPI5`/`GPI6` to the three
> S5 base addresses is not proven. The public conversion tool assumes
> `GPI4` = `0x16004000`, `GPI5` = `0x16006000`, `GPI6` = `0x16005000`,
> which disagrees with the vendor devicetree for at least two board GPIOs.
> Resolve it from the DSDT (`_CRS` of each `GPIn`).

> OPEN QUESTION: Whether the DSDT supplies an `ngpios` `_DSD` property is
> not known. The Sky1-Linux driver uses 32 lines if the property is
> missing. That would expose 15 lines that do not exist on FCH GPIO 3 and
> 22 on S5 GPIO 1/2. Use the line counts in §1.1 if the property is
> absent.

### 1.1.3 Devicetree identification

| Compatible | Meaning |
|---|---|
| `cdns,gpio-r1p02` | Generic Cadence IP (mainline and vendor DT) |
| `cix,sky1-gpio` | Sky1 variant (Sky1-Linux): same registers plus the firmware wake call of §1.5 |

Properties: `ngpios` (1–32), `gpio-controller`, `#gpio-cells = 2`,
`interrupt-controller`, `#interrupt-cells = 2`, optional `clocks`,
`resets`. **[POLICY]** *(vendor)* Optional `gpio-io-mask`: a 32-bit mask
of lines that must never be switched to output.

## 1.2 Register map

**[HW]** All registers are 32 bits wide. Bit *n* controls line *n*.

| Offset | Name | Access | Meaning per bit |
|---|---|---|---|
| `0x00` | `BYPASS_MODE` | RW | 1 = line is not under GPIO control (passes through to another function); 0 = GPIO controls the line |
| `0x04` | `DIRECTION_MODE` | RW | 1 = input, 0 = output |
| `0x08` | `OUTPUT_EN` | RW | 1 = output driver enabled. Only takes effect while the line is an output |
| `0x0C` | `OUTPUT_VALUE` | RW | Level driven when the line is an output |
| `0x10` | `INPUT_VALUE` | RO | Current pin level |
| `0x14` | `IRQ_MASK` | RO | 1 = line interrupt masked, 0 = enabled |
| `0x18` | `IRQ_EN` | WO | Write 1 = unmask (clears the `IRQ_MASK` bit). 0 bits have no effect |
| `0x1C` | `IRQ_DIS` | WO | Write 1 = mask (sets the `IRQ_MASK` bit). 0 bits have no effect |
| `0x20` | `IRQ_STATUS` | R / W1C | 1 = interrupt pending on the line (see §1.3.3) |
| `0x24` | `IRQ_TYPE` | RW | 1 = level-sensitive, 0 = edge-sensitive |
| `0x28` | `IRQ_VALUE` | RW | Level mode: 1 = high, 0 = low. Edge mode: 1 = rising, 0 = falling |
| `0x2C` | `IRQ_ANY_EDGE` | RW | 1 = interrupt on both edges (edge mode only; overrides `IRQ_VALUE`) |

- **[HW]** `IRQ_EN`/`IRQ_DIS` are set/clear ports for the mask, so you
  never need read-modify-write on the mask. All other writable registers
  are plain RW and need read-modify-write under a lock, because one word
  serves 32 lines.
- **[HW]** Lines at or above the instance's line count read as zero.
  Software must not rely on writes to them.

### 1.2.1 Trigger encoding

| Wanted trigger | `IRQ_TYPE` | `IRQ_VALUE` | `IRQ_ANY_EDGE` |
|---|---|---|---|
| Level high | 1 | 1 | 0 |
| Level low | 1 | 0 | 0 |
| Rising edge | 0 | 1 | 0 |
| Falling edge | 0 | 0 | 0 |
| Both edges | 0 | don't care | 1 |

**[HW]** Program all three registers for a line together. A stale
`IRQ_ANY_EDGE` bit left set from an earlier both-edges setting overrides
a later single-edge setting.

## 1.3 Operation

### 1.3.1 Initialisation

1. **[HW]** Enable the clock and release the reset (FCH instances).
   *(vendor)* Under ACPI the clock comes from CLKT and the reset from RSTL.
2. **[HW]** Mask all line interrupts (write all-ones of the line count to
   `IRQ_DIS`), then clear pending status (§1.3.3). Do this **before**
   unmasking the parent GIC interrupt. A pending line with no handler
   otherwise gives an interrupt storm: the parent is level-triggered and
   stays asserted.
3. **[POLICY]** Make all lines inputs (`DIRECTION_MODE` = all ones of the
   line count).
   *(vendor)* The property `gpio-io-mask` is described as "lines that must
   stay inputs", but the vendor driver clears those bits in the
   all-inputs write, which makes them outputs. Treat the property as
   "do not touch these lines" and leave their direction as firmware set
   it.
4. **[POLICY]** Set `OUTPUT_EN` to all ones and `BYPASS_MODE` to 0, so
   that direction alone decides whether a line drives. Linux records the
   `BYPASS_MODE` value found at probe and restores a line's bypass bit
   when the line is released.

> OPEN QUESTION: Clearing `BYPASS_MODE` for every line at probe hands
> lines that firmware left in bypass over to GPIO control. On Sky1 the pad
> function is chosen by the pin controller (05-pinctrl), so the meaning of
> bypass on this SoC is unclear. A conservative driver clears bypass only
> for lines it is asked to use.

### 1.3.2 Line use

- **[HW]** Input: set the `DIRECTION_MODE` bit. Read `INPUT_VALUE`.
- **[HW]** Output: write `OUTPUT_VALUE` first, then clear the
  `DIRECTION_MODE` bit, so the line does not glitch to a stale level.
  Reading back `OUTPUT_VALUE` gives the level being driven.
- **[HW]** A line must be an input to be used as an interrupt source.
- **[HW]** The pad must also be muxed to its GPIO function in the pin
  controller (05-pinctrl). Under ACPI, firmware has done this already.

### 1.3.3 Interrupt handling

1. **[HW]** On the parent interrupt, read `IRQ_STATUS` and drop bits set in
   `IRQ_MASK`. Every remaining bit is a pending line.
2. **[HW]** Edge lines: acknowledge the line by writing its bit to
   `IRQ_STATUS` (write-1-to-clear) **before** running the consumer's
   handler, so an edge arriving during the handler is not lost.
3. **[HW]** Level lines: mask the line (`IRQ_DIS`), run the consumer's
   handler, acknowledge, then unmask (`IRQ_EN`) once the source has been
   serviced. The status bit comes back while the input stays active.
4. **[HW]** Changing a line's trigger type also changes how it must be
   handled (edge or level flow). A line with no trigger set must stay
   masked.

> OPEN QUESTION: Earlier Linux support for this IP treated `IRQ_STATUS`
> as clear-**on-read**, with no acknowledge register. The Sky1-Linux
> driver treats it as write-1-to-clear.
> If reads clear it, step 1 consumes every pending bit at once and a
> driver must service all of them from that single read. Do not read the
> register twice per interrupt. Test on hardware with two lines firing at
> once.

- **[POLICY]** Linux supports all five trigger types. Edge support is
  needed on the Orange Pi 6 Plus, whose EC interrupt is rising-edge
  (§1.6). The mainline driver at the baseline programs edge triggers but
  has no acknowledge step and handles every line as level. The Sky1 patch
  set adds the acknowledge and the edge flow of step 2.

### 1.3.4 Locking

**[HW]** The single parent interrupt serves all 32 lines. Configuration
registers are shared words. Serialise read-modify-write of
`DIRECTION_MODE`, `OUTPUT_VALUE`, `IRQ_TYPE`, `IRQ_VALUE`,
`IRQ_ANY_EDGE` and `BYPASS_MODE` per controller.

## 1.4 Power management

### 1.4.1 State to keep across suspend

**[HW]** FCH instances lose their register state when the FCH domain is
powered down. *(vendor)* The Sky1 driver saves and restores these
registers on every system suspend, for all instances:

| Saved | Restored as |
|---|---|
| `BYPASS_MODE`, `DIRECTION_MODE`, `OUTPUT_EN`, `OUTPUT_VALUE` | Written back as is |
| `IRQ_TYPE`, `IRQ_VALUE`, `IRQ_ANY_EDGE` | Written back as is |
| `IRQ_MASK` (read) | Inverted and written to `IRQ_EN` |

### 1.4.2 Suspend

1. **[POLICY]** *(vendor)* Run late in suspend, after consumers have
   suspended, and early in resume, before them.
2. **[HW]** Save the registers in §1.4.1.
3. **[HW]** Mask every line that is not a wake source (write the
   complement of the wake mask to `IRQ_DIS`).
4. **[HW]** *(vendor, `cix,sky1-gpio` / ACPI only)* Report the wake
   configuration to firmware (§1.5) with "suspending" = 1.
5. **[POLICY]** Disable the clock.

### 1.4.3 Resume

1. **[POLICY]** Enable the clock.
2. **[POLICY]** *(vendor)* Pulse the reset if one is described. For FCH
   instances this is the shared reset (§1.1), so do the pulse once for the
   group, before restoring any of the four.
3. **[HW]** Mask all lines (`IRQ_DIS` = all ones) so a half-restored
   trigger setup cannot fire.
4. **[HW]** Restore the configuration and output registers, then the
   trigger registers, then unmask (`IRQ_EN`) the lines that were unmasked
   at suspend.

> OPEN QUESTION: The vendor resume does not tell firmware that the system
> is awake again (it never sends "suspending" = 0). Whether firmware
> drops the wake configuration on its own at wake-up is not known.

### 1.4.4 Shutdown

**[POLICY]** *(vendor)* At shutdown or reboot the driver does steps 3–5 of
§1.4.2, so wake lines stay armed for wake-from-off. Firmware decides
whether a GPIO can power the system on.

## 1.5 Firmware wake configuration (SMC)

**[HW]** *(vendor)* Wake from system suspend needs the power-domain
controller (PDC) firmware to be told which GPIO lines may wake the
system. The call is an SMCCC fast call through the SMC conduit:

| Register | Value |
|---|---|
| Function ID | `0xC2000009` (SiP service, 32-bit fast call) |
| Argument 1 | `0x02` (set wake) |
| Argument 2 | The controller's GIC SPI number (GSIV − 32), for example 372 for S5 GPIO 0 |
| Argument 3 | 1 = entering suspend / shutdown, 0 = leaving |
| Argument 4 | Wake-enable mask, bit *n* = line *n* |
| Arguments 5–7 | 0 |

- **[HW]** *(vendor)* One call per controller that has an interrupt. The
  return value is ignored by the vendor driver.
- **[HW]** *(vendor)* Before a bug fix in the patch set, the driver passed
  an OS-internal IRQ number here. The corrected version passes the GIC
  hardware number minus 32. Use the GIC SPI number.

> OPEN QUESTION: Which controllers firmware can actually wake from (S5
> only, or also FCH) and the meaning of the return value are not
> documented. Treat a call on an FCH controller as a hint.

> OPEN QUESTION: The function ID `0xC2000009` sits in the SiP range,
> owned by Trusted Firmware-A on Sky1. Whether TF-A checks the arguments
> is unknown.

## 1.6 ACPI consumers and board usage

**[ABI]** Consumers describe lines with the standard ACPI `GpioIo` and
`GpioInt` resources. The pin number in the resource is the line number
(0–31) within the controller named by the resource source path. No Sky1
extension is needed:

- `GpioInt` edge/level and active-high/low map directly to §1.2.1.
  "ActiveBoth" maps to both edges.
- `GpioInt` with `ExclusiveAndWake` / `SharedAndWake` marks a wake
  source. Set its bit in the wake mask (§1.5).
- **[ABI]** Board SSDTs describe regulators (`PRP0001` +
  `regulator-fixed`), LEDs (`PRP0001` + `gpio-leds`) and buttons
  (`ACPI0011`, `PNP0C0C`) with these resources.

**[HW]** Board GPIO facts (vendor devicetrees, same numbering as ACPI):

| Board | Use | Controller, line | Trigger / polarity |
|---|---|---|---|
| Orion O6 | RTS5453 PD controllers (both, shared) | S5 GPIO 0, line 8 | Level low |
| Orange Pi 6 Plus | RTS5453 PD controllers (both, shared) | S5 GPIO 0, line 7 | Level low |
| Orange Pi 6 Plus | Embedded controller (I²C6, address `0x76`), wake source | S5 GPIO 0, line 6 | Rising edge |
| Orange Pi 6 Plus | RTC (I²C3, address `0x32`) | S5 GPIO 0, line 10 | Level low |
| Orion O6 | Ethernet PHY 1 power | S5 GPIO 0, line 9 | Active high |
| Orion O6 | WLAN / PCIe 3.3 V | S5 GPIO 0, line 12 | Active high |
| Orion O6 | Status LED | S5 GPIO 0, line 7 | Active high |
| Orion O6 | Camera power | FCH GPIO 1, line 6 | Active high |

### 1.6.1 Line numbers and pad names

**[HW]** Pads named `GPIOnnn` (05-pinctrl) connect to lines of the
controllers above. For S5 pads `GPIO001`–`GPIO014` the vendor sources agree:
pad `GPIOnnn` = S5 GPIO 0, line *nnn* − 1.

> OPEN QUESTION: The mapping of the remaining pads to controller lines is
> not consistent across sources. The S5 controllers have 52 lines between
> them but the S5 pin controller has only 42 `GPIOnnn` pads. The FCH
> controllers have 113 lines but the FCH pin controller has 111
> (`GPIO043`–`GPIO153`). Vendor DT comments put `GPIO015`–`GPIO017` on
> S5 GPIO 1 lines 0–2, `GPIO021`/`GPIO022` on S5 GPIO 2 lines 6/7, and
> `GPIO030`/`GPIO031` on S5 GPIO 0 lines 19/20. One reading that fits the
> line counts: S5 GPIO 0 = `GPIO001`–`GPIO014` then `GPIO025`–`GPIO042`;
> S5 GPIO 1 = `GPIO015`–`GPIO024`. This is unconfirmed. Under ACPI a driver
> does not need the mapping, because consumers give controller and line
> directly.
