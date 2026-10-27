# 14: Power Domains

**Status: not started.** *(vendor)*: mainline has no driver or devicetree
node for this block at the baseline.

## Scope

Power-domain on/off sequencing for the K3 power controller (vendor DT compatible `spacemit,power-controller`), and which domains GPU, VPU, display, camera and PCIe need. The mainline MPMU and APMU nodes already declare `#power-domain-cells`, but no behavior is defined.

Clock and reset IDs for these blocks are in [01](01-clocks-resets.md).
