# Provenance: `specs/qcom-mdss-dpu/02-dp-controller-aux-link-tu.md`

- **Baseline Commits**:
  - `mainline`: `d135ab453a7400a34930bdda1ef4bdc8acfac80e` (`/home/jkane/Projects/OS/linux`)
  - `radxa/kernel`: `c00e1e17e65104c8143a8f3d559ef723ae5824d3` (`linux-7.0.11`)
  - `TC956X/TC9564_Host_Driver`: `3eca71ce0c5708c92e20f14b8e858cd4483d58f3`
- **Extracted Date**: `2026-10-27`

## Source Files and Line Ranges Read

1. **`/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/dp/dp_reg.h` (lines 1–425)**
   - Extracted complete DisplayPort controller register offsets and bit masks across `AHB` (`+0x0000`), `AUX` (`+0x0200`), `LINK` (`+0x0400`), and `P0` (`+0x1000`) sub-regions.
2. **`/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/dp/dp_aux.c` (lines 1–600)**
   - Extracted AUX transaction FIFO header packing (`REG_DP_AUX_DATA` at `0x0224`), `REG_DP_AUX_TRANS_CTRL` (`0x0228`) flags (`GO`, `TRANS_TYPE`, `NO_SEND_ADDR`, `NO_SEND_STOP`), AUX interrupt status handling (`REG_DP_INTR_STATUS` at `0x0020`), PHY AUX error recalibration, and HPD configuration (`0x0200`..`0x021c`).
3. **`/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/dp/dp_ctrl.c` (lines 1–2888)**
   - Extracted Link Training Phase 1 (`TPS1`) and Phase 2 (`TPS2`/`TPS3`/`TPS4`) state machine, `REG_DP_STATE_CTRL` (`0x0404`) and `REG_DP_MAINLINK_READY` (`0x0438`) bit definitions, `REG_DP_CONFIGURATION_CTRL` (`0x0408`) encoding, `MVID`/`NVID` widebus (`* 2`) and HBR2/HBR3 (`* 2` / `* 3`) scaling rules, Transfer Unit (`TU`) & Boundary Moderation calculation algorithm (`_dp_tu_calc_ok`, `msm_dp_ctrl_calc_tu`), and stream enable/disable sequences (`PUSH_IDLE` before stopping `INTF`).
4. **`/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/dp/dp_link.c` (lines 1–800)**
   - Extracted DPCD link training status parsing, voltage swing and pre-emphasis request extraction (`0x00202`..`0x00207`), and link rate/lane count downshift policy.
5. **`/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/dp/dp_panel.c` (lines 1–829)**
   - Extracted timing register programming (`REG_DP_TOTAL_HOR_VER` `0x0418`, `REG_DP_START_HOR_VER_FROM_SYNC` `0x041c`, `REG_DP_HSYNC_VSYNC_WIDTH_POLARITY` `0x0420`, `REG_DP_ACTIVE_HOR_VER` `0x0424`, `REG_DP_MISC1_MISC0` `0x0428`) and widebus `DP_INTF_CONFIG` (`0x1004`).
6. **`/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/dp/dp_display.c` (lines 1–500)**
   - Extracted SC8280XP DP0..DP3 controller descriptor table and HPD event handling.
7. **`/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/gpu-display.md` (lines 30–115)**
   - Extracted empirical Dragon Q8B DP2 (`0x0ae9a000`) -> Chrontel CH7218A behavior, UEFI vs calculated TU comparison (`TU 33, valid 5` vs `TU 41, valid 6` with boundary moderation), and mode-change ordering (`PUSH_IDLE` -> wait `IDLE_PATTERN_SENT` -> stop `INTF` -> link off).
