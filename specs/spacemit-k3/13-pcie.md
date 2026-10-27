# 13: PCIe Host Controllers and PCIe PHY

**Status: not started.** *(vendor)*: mainline has no driver or devicetree
node for this block at the baseline.

## Scope

SoC integration of the five Synopsys DesignWare PCIe root ports, A–E, at `0x80000000`, `0x80400000`, `0x80800000`, `0x80C00000` and `0x81000000`. Covers register windows (DBI, DBI2, iATU), link bring-up, the SoC-level control and status registers, PERST# and clock-request handling, MSI routing to the IMSIC, and the PCIe PHYs (`spacemit,k3-pcie-phy` at `0x81D00000`–`0x82200000`). The first PHY is the same block as mainline's USB3/PCIe combo PHY (07). The DesignWare core itself is published IP and is not respecified.

On the CoM260 carrier, PCIe carries the Wi-Fi module.

Clock and reset IDs for these blocks are in [01](01-clocks-resets.md).
