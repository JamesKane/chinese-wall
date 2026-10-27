# 15: Thermal Sensor

**Status: not started.** *(vendor)*: mainline has no driver or devicetree
node for this block at the baseline.

## Scope

The K3 temperature sensor (vendor DT compatible `spacemit,k3-tsensor`, base `0xD4018000`): channel layout, conversion, thresholds and interrupt. Needed before running the CPUs at high clock rates. The APBC TSEN clock and reset are in 01.

Clock and reset IDs for these blocks are in [01](01-clocks-resets.md).
