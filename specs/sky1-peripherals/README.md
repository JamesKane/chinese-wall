# CIX Sky1 Board Peripherals Functional Specification

Target: the **CIX Sky1** SoC (CD8180) on the Radxa Orion O6 and the
Orange Pi 6 Plus, booted with **ACPI** (the UEFI default) or devicetree.
These sections cover the low-speed peripherals a board needs early: GPIO,
I²C, SPI and the serial-flash controller, the USB-C power-delivery
controller, and the pin controllers.

The GPIO, I²C and SPI blocks are Cadence IP. The specs describe the IP
behaviour Sky1 needs and the Sky1 integration (addresses, interrupts,
clocks, resets, pins, quirks). They build on sky1-soc (SCMI clock IDs,
CLKT/RSTL tables, reset indices, GSIV numbering) and do not repeat it.

Sources: the mainline Sky1 devicetree, bindings, pad-function header and
drivers; the Sky1-Linux kernel patch set and the CIX vendor devicetree;
and the MIT-licensed Sky1-Linux ACPI-to-DT notes and ACPI-derived
devicetrees. No DSDT was available, so ACPI facts come from what drivers
match and parse and from the ACPI-derived tables. Each statement is
tagged:

- **[HW]**: required by the hardware or firmware
- **[POLICY]**: a Linux design choice, informative only
- **[ABI]**: needed for compatibility with existing firmware tables or
  userspace

Facts seen only in vendor or community patches are marked *(vendor)*.

| # | Section | Status |
|---|---------|--------|
| 01 | [GPIO controllers (Cadence GPIO r1p02): instances, registers, edge/level IRQs, wake SMC, PM](01-gpio.md) | draft: pad-to-line map partly unknown |
| 02 | [I²C controllers (Cadence I²C r1p14): registers, divider, transfer sequences, errors](02-i2c.md) | draft |
| 03 | [SPI controllers (Cadence SPI r1p6) and the XSPI flash controller](03-spi-qspi.md) | draft: XSPI IP identity unresolved |
| 04 | [Realtek RTS5453H USB-C PD controller: status protocol, Type-C/PHY/DP reporting](04-usb-pd-rts5453.md) | draft: ACPI properties unknown |
| 05 | [Pin controllers: pad register, drive codes, ACPI pin groups, pad tables (Annex A)](05-pinctrl.md) | draft |

## ACPI hardware IDs used here

| `_HID` | Device | Section |
|---|---|---|
| `CIXH1002`, `CIXH1003` | GPIO controller (same programming model) | 01 |
| `CIXH200B` | I²C controller | 02 |
| `CIXH2001` | SPI controller | 03 |
| `CIXH2002` | Serial-flash (XSPI) controller | 03 |
| `CIXH200D` | RTS5453H PD controller (I²C child) | 04 |
| `CIXHA016`, `CIXHA017` | FCH and S5 pin controllers | 05 |

The public ACPI-to-DT notes get several of these wrong. They list
`CIXH2011` as SPI (it is PWM), `CIXH2001` as watchdog (it is SPI; the
watchdog is `CIXH1005`) and `CIXH1006`/`CIXH1007` as pin controllers
(they are a DMA controller and the GPT timer). The table above follows
the driver match tables.
