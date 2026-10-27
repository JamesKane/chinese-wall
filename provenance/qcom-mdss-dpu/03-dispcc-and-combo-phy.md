# Provenance: `specs/qcom-mdss-dpu/03-dispcc-and-combo-phy.md`

- **Baseline Commits**:
  - `mainline`: `d135ab453a7400a34930bdda1ef4bdc8acfac80e` (`/home/jkane/Projects/OS/linux`)
  - `radxa/kernel`: `c00e1e17e65104c8143a8f3d559ef723ae5824d3` (`linux-7.0.11`)
  - `TC956X/TC9564_Host_Driver`: `3eca71ce0c5708c92e20f14b8e858cd4483d58f3`
- **Extracted Date**: `2026-10-27`

## Source Files and Line Ranges Read

1. **`/home/jkane/Projects/OS/linux/drivers/clk/qcom/dispcc-sc8280xp.c` (lines 1–3203)**
   - Extracted `DISP_CC_0` (`0x0af00000`) and `DISP_CC_1` (`0x22100000`) Lucid 5LPE PLLs (`DISP_CC_PLL0` `0x0000`, `DISP_CC_PLL1` `0x1000`, `DISP_CC_PLL2` `0x9000`), Root Clock Generators (`mdp_clk_src` `0x20c8`, `ahb_clk_src` `0x2170`, `vsync_clk_src` `0x2128`, `dptx0..3_aux_clk_src`, `dptx0..3_link_clk_src`, `dptx0..3_pixel0/1_clk_src`), post-dividers (`dptx0..3_link_div_clk_src`), branch clocks (`CBCRs` `0x2010`..`0x2084`), resets (`0x2000`, `0x4000`), and GDSCs (`MDSS_GDSC` `0x3000`, `MDSS_INT2_GDSC` `0xa000`).
2. **`/home/jkane/Projects/OS/linux/drivers/phy/qualcomm/phy-qcom-qmp-combo.c` (lines 1–5444)**
   - Extracted `sc8280xp_usb43dpphy_cfg` (`qmp_combo_offsets_v5`, `sc8280xp_usb43dp_serdes_tbl`, `sc8280xp_usb43dp_tx_tbl`, `sc8280xp_usb43dp_rx_tbl`, `sc8280xp_usb43dp_pcs_tbl`, `qmp_v5_dp_serdes_tbl`, `qmp_v4_dp_serdes_tbl_rbr/hbr/hbr2/hbr3`, `qmp_v5_5nm_dp_tx_tbl`), `qmp_dp_v5_voltage_swing_*` and `qmp_dp_v5_pre_emphasis_*` 4x4 lookup tables, `qmp_v4_dp_aux_init()`, `qmp_v4_configure_dp_tx()`, `qmp_v3_configure_dp_clocks()`, `qmp_v4_configure_dp_phy()`, and Type-C mux/orientation handling (`DP_COM_PHY_MODE_CTRL`, `DP_COM_TYPEC_CTRL`, `QSERDES_DP_PHY_MODE`, `QSERDES_DP_PHY_PD_CTL`).
3. **`/home/jkane/Projects/OS/linux/drivers/phy/qualcomm/phy-qcom-qmp-dp-com-v3.h` (lines 1–19)**
   - Extracted `QPHY_V3_DP_COM_*` register offsets (`0x00`..`0x1c`).
4. **`/home/jkane/Projects/OS/linux/drivers/phy/qualcomm/phy-qcom-qmp-dp-phy.h` (lines 1–63) & `phy-qcom-qmp-dp-phy-v5.h` (lines 1–15)**
   - Extracted `QSERDES_DP_PHY_*` (`0x000`..`0x044`) and `QSERDES_V5_DP_PHY_*` (`VCO_DIV` `0x070`, `AUX_INTERRUPT_STATUS` `0x0d8`, `STATUS` `0x0dc`) register offsets and bit definitions.
5. **`/home/jkane/Projects/OS/linux/drivers/phy/qualcomm/phy-qcom-qmp-qserdes-com-v4.h` (lines 1–124) & `phy-qcom-qmp-qserdes-com-v5.h` (lines 1–125)**
   - Extracted `QSERDES_V4_COM_*` and `QSERDES_V5_COM_*` register offsets (`0x000`..`0x1c0`).
6. **`/home/jkane/Projects/OS/linux/drivers/phy/qualcomm/phy-qcom-qmp-qserdes-txrx-v5_5nm.h` (lines 1–329)**
   - Extracted `QSERDES_V5_5NM_TX_*` (`0x00`..`0xfc`) and `QSERDES_V5_5NM_RX_*` (`0x000`..`0x3e8`) register offsets.
7. **`/home/jkane/Projects/OS/linux/drivers/phy/qualcomm/phy-qcom-qmp-pcs-v5.h` (lines 1–35)**
   - Extracted `QPHY_V5_PCS_*` register offsets (`0x000`..`0x1ec`).
8. **`/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/gpu-display.md` (lines 30–115) & `usb.md` (lines 38–108)**
   - Extracted empirical Dragon Q8B RCG M/N calculation (`11/200` for `1080p60` from `1350 MHz` PHY VCO div clock) and USB-C Port 0 (`0x088eb000`) / Port 1 (`0x08903000`) orientation and DP Alt-Mode wiring.
