# 05: SpacemiT P1 PMIC

**Status: not started.**

## Scope

Register map of the P1 PMIC: six buck regulators, the ALDO/DLDO regulators, voltage encodings and ramp settings, the RTC, and the interrupt line. It sits on i2c8 at address `0x41` and uses SoC interrupt 64.

Low priority: the CoM260 devicetree leaves every rail it uses always-on.

Clock and reset IDs for these blocks are in [01](01-clocks-resets.md).
