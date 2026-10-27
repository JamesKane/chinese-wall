# 16: Display: DPU, MIPI DSI, DisplayPort/eDP

**Status: not started.** *(vendor)*: mainline has no driver or devicetree
node for this block at the baseline.

## Scope

The "Saturn" display processor (`spacemit,dpu-saturn`), its MMU and command lists, the MIPI DSI host and D-PHY, and the two Innosilicon DisplayPort/eDP transmitters (`spacemit,inno-dp0`, `spacemit,inno-dp1`). Clocks are in 01 §1.8.6.

Lower priority than storage and networking.

Clock and reset IDs for these blocks are in [01](01-clocks-resets.md).
