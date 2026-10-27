# 18: Video codec (Linlon v5) delta

**Status: not started.** *(vendor)*: mainline has no driver or devicetree
node for this block at the baseline.

## Scope

The K3 video codec (`arm china,linlon-v5`, 2 MiB register window at
`0xC0500000`, interrupt 74, one APMU clock and reset, its own power domain)
belongs to the same firmware-driven MVE family that
[`../linlon-vpu`](../linlon-vpu/) specifies for Sky1. This section records
only the differences:

- product ID in `HARDWARE_ID[31:16]`, number of cores and LSIDs
- firmware file names and whether the firmware message protocol version
  differs
- MMU and bus-attribute differences
- clock, reset and power-domain sequencing on K3 (01, 14)
- codec and format support

> OPEN QUESTION: Which product ID the K3 core reports, and whether the
> Sky1 spec's identification table already covers it.

Firmware comes from SpacemiT's published VPU packages.
