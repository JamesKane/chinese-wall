# Display Clock Controller (`DISP_CC`) and QMP V5 Combo PHY Specification (`SC8280XP`)

## 1. Display Clock Controller (`DISP_CC_0` at `0x0af00000`, `DISP_CC_1` at `0x22100000`)

**[HW]** Each MDSS instance on SC8280XP is clocked by a dedicated Display Clock Controller (`DISP_CC_0` at `0x0af00000` for `MDSS0`, `DISP_CC_1` at `0x22100000` for `MDSS1`, size `0x20000`). Both instances have an identical register layout relative to their `DISP_CC_BASE`.

### 1.1 Global Distributed Switch Controllers (`GDSCs`) and Block Resets (`BCRs`)

**[HW]** Power domains and block-level resets inside `DISP_CC`:

| Offset | Register Name | Type | Description |
|---|---|---|---|
| `0x02000` | `DISP_CC_MDSS_CORE_BCR` | Reset | Bit `0` `BLK_ARES`: write `1` to assert full MDSS core reset, `0` to release |
| `0x03000` | `DISP_CC_MDSS_GDSC` | GDSC | Primary MDSS power switch: bit `0` `SW_COLLAPSE`, bit `11` `RETAIN_FF_ENABLE`, bit `31` `PWR_ON` |
| `0x03004` | `DISP_CC_MDSS_GDSC_CFG_STATUS` | RO | GDSC state machine status |
| `0x04000` | `DISP_CC_MDSS_RSCC_BCR` | Reset | Bit `0` `BLK_ARES`: MDSS RSCC block reset |
| `0x0a000` | `DISP_CC_MDSS_INT2_GDSC` | GDSC | Secondary internal MDSS power switch: bit `0` `SW_COLLAPSE`, bit `11` `RETAIN_FF_ENABLE`, bit `31` `PWR_ON` |

**[HW]** To power on `MDSS_GDSC` (`0x3000`):
1. Read `0x3000`. If bit `31` (`PWR_ON`) is already `1` and bit `0` (`SW_COLLAPSE`) is `0`, ensure bit `11` (`RETAIN_FF_ENABLE`) is set and return.
2. Clear bit `0` (`SW_COLLAPSE`) in `0x3000`, then poll `0x3000` bit `31` (`PWR_ON`, up to `1000 us`) until it reads `1`.
3. Set bit `11` (`RETAIN_FF_ENABLE` = `0x00000800`) in `0x3000` so clock-branch flip-flops retain state across power transitions.

### 1.2 Lucid 5LPE PLLs (`DISP_CC_PLL0`, `DISP_CC_PLL1`, `DISP_CC_PLL2`)

**[HW]** Each `DISP_CC` contains three Lucid 5LPE Alpha PLLs fed by the `19.2 MHz` `bi_tcxo` reference clock:

| PLL Name | Base Offset | Default VCO Rate | Purpose |
|---|---|---|---|
| `DISP_CC_PLL0` | `0x00000` | `1380 MHz` (`L = 0x47`, `ALPHA = 0xe000`) | Clocks `mdp_clk_src` and `rot_clk_src` |
| `DISP_CC_PLL1` | `0x01000` | `600 MHz` (`L = 0x1f`, `ALPHA = 0x4000`) | Clocks `ahb_clk_src` (`DISP_CC_PLL1_OUT_MAIN`) |
| `DISP_CC_PLL2` | `0x09000` | `540 MHz` (`L = 0x1c`, `ALPHA = 0x2000`) | Clocks `mdss_edp_pixel_clk_src` if internal PLL used |

**[HW]** Lucid 5LPE PLL register offsets relative to `PLL_BASE` (`0x0000`, `0x1000`, or `0x9000`):
- `+0x00` `PLL_MODE`: bit `0` `OUTCTRL`, bit `1` `BYPASSNL`, bit `2` `RESET_N`, bit `29` `LOCK_DET` (RO: `1` = locked), bit `30` `ACTIVE_FLAG` (RO: `1` = active), bit `31` `ACK_LATCH`.
- `+0x04` `PLL_L_VAL`: 16-bit integer multiplier `L` (`f_vco = 19.2 MHz * (L + ALPHA / 2^16)`).
- `+0x08` `PLL_CAL_L_VAL`: calibration `L` value (`0x00000044`).
- `+0x0c` `PLL_USER_CTL`: bit `0` `PLLOUT_MAIN_EN`, bit `1` `PLLOUT_EVEN_EN`, bits `[14:13]` `VCO_SEL` (`2` = VCO2).
- `+0x10` `PLL_USER_CTL_U`: `0x00000805`.
- `+0x14` `PLL_USER_CTL_U1`: `0x00000000`.
- `+0x18` `PLL_CONFIG_CTL`: `0x20485699`.
- `+0x1c` `PLL_CONFIG_CTL_U`: `0x00002261`.
- `+0x20` `PLL_CONFIG_CTL_U1`: `0x329a299c`.
- `+0x24` `PLL_TEST_CTL`: `0x00000000`.
- `+0x28` `PLL_TEST_CTL_U`: `0x00000000`.
- `+0x2c` `PLL_TEST_CTL_U1`: `0x00000000`.
- `+0x30` `PLL_STATUS`: status register.
- `+0x40` `PLL_ALPHA_VAL`: 16-bit fractional multiplier `ALPHA`.
- `+0x54` `PLL_OPMODE`: `0x00000001` (`RUN`).

### 1.3 Root Clock Generators (`RCGs`)

**[HW]** Every RCG in `DISP_CC` has the following register offsets relative to its `CMD_RCGR` base:
- `+0x00` `CMD_RCGR`: bit `0` `UPDATE` (write `1` to latch new configuration; poll until bit `0` clears to `0`), bit `1` `ROOT_EN`, bit `31` `ROOT_OFF` (RO: `0` = running).
- `+0x04` `CFG_RCGR`: bits `[4:0]` `SRC_DIV` (`2 * div - 1`, where `0` or `1` = bypass/div-1), bits `[10:8]` `SRC_SEL` (parent clock mux index), bits `[13:12]` `MODE` (`0` = bypass/divider-only, `2` = dual-edge M/N counter).
- `+0x08` `M` (only on M/N RCGs): 16-bit numerator `M`.
- `+0x0c` `N` (only on M/N RCGs): 16-bit bitwise-inverted `(N - M)`, i.e., `(~(N - M)) & 0xffff`.
- `+0x10` `D` (only on M/N RCGs): 16-bit bitwise-inverted duty-cycle value, programmed to `(~N) & 0xffff` for 50% duty cycle.

| RCG Clock Name | `CMD_RCGR` Offset | M/N Width | Parent `SRC_SEL` Map | Notes |
|---|---|---|---|---|
| `mdp_clk_src` | `0x020c8` | `0` (Div only) | `0`=`bi_tcxo`, `1`=`DISP_CC_PLL0` | `1380 MHz / 2.5 = 552 MHz` or `/ 3 = 460 MHz` (`SRC_SEL=1`) |
| `ahb_clk_src` | `0x02170` | `0` (Div only) | `0`=`bi_tcxo`, `2`=`DISP_CC_PLL1` | `19.2 MHz` (`SRC_SEL=0`), `37.5 MHz` (`SRC_SEL=2, DIV=31`), `75 MHz` (`DIV=15`) |
| `vsync_clk_src` | `0x02128` | `0` (Div only) | `0`=`bi_tcxo` (`19.2 MHz`) | Fixed `19.2 MHz` (`CFG_RCGR = 0x0000`) |
| `dptx0_aux_clk_src` | `0x02188` | `0` (Div only) | `0`=`bi_tcxo` (`19.2 MHz`) | Fixed `19.2 MHz` (`CFG_RCGR = 0x0000`) |
| `dptx0_link_clk_src` | `0x02140` | `0` (Div only) | `0`=`bi_tcxo`, `1`=`dp0_phy_pll_link` | Driven by PHY 0 `link_clk` (`SRC_SEL=1, SRC_DIV=1` -> `CFG_RCGR = 0x0101`) |
| `dptx0_pixel0_clk_src` | `0x02158` | `16-bit M/N` | `0`=`bi_tcxo`, `2`=`dp0_phy_pll_vco_div` | Driven by PHY 0 `vco_div_clk` (`SRC_SEL=2, MODE=2, SRC_DIV=1` -> `CFG = 0x2201`) |
| `dptx0_pixel1_clk_src` | `0x021a0` | `16-bit M/N` | `0`=`bi_tcxo`, `2`=`dp0_phy_pll_vco_div` | MST Stream 1 pixel clock for `DP0` |
| `dptx1_aux_clk_src` | `0x02200` | `0` (Div only) | `0`=`bi_tcxo` (`19.2 MHz`) | Fixed `19.2 MHz` |
| `dptx1_link_clk_src` | `0x021d0` | `0` (Div only) | `0`=`bi_tcxo`, `1`=`dp1_phy_pll_link` | Driven by PHY 1 `link_clk` (`CFG_RCGR = 0x0101`) |
| `dptx1_pixel0_clk_src` | `0x021b8` | `16-bit M/N` | `0`=`bi_tcxo`, `2`=`dp1_phy_pll_vco_div` | Driven by PHY 1 `vco_div_clk` (`CFG_RCGR = 0x2201`) |
| `dptx1_pixel1_clk_src` | `0x021e8` | `16-bit M/N` | `0`=`bi_tcxo`, `2`=`dp1_phy_pll_vco_div` | MST Stream 1 pixel clock for `DP1` |
| `dptx2_aux_clk_src` | `0x02260` | `0` (Div only) | `0`=`bi_tcxo` (`19.2 MHz`) | Fixed `19.2 MHz` |
| `dptx2_link_clk_src` | `0x02230` | `0` (Div only) | `0`=`bi_tcxo`, `1`=`dp2_phy_pll_link` | Driven by PHY 2 `link_clk` (`CFG_RCGR = 0x0101`) |
| `dptx2_pixel0_clk_src` | `0x02218` | `16-bit M/N` | `0`=`bi_tcxo`, `2`=`dp2_phy_pll_vco_div` | Driven by PHY 2 `vco_div_clk` (`CFG_RCGR = 0x2201`; Dragon Q8B HDMI) |
| `dptx2_pixel1_clk_src` | `0x02248` | `16-bit M/N` | `0`=`bi_tcxo`, `2`=`dp2_phy_pll_vco_div` | MST Stream 1 pixel clock for `DP2` |
| `dptx3_aux_clk_src` | `0x022a8` | `0` (Div only) | `0`=`bi_tcxo` (`19.2 MHz`) | Fixed `19.2 MHz` |
| `dptx3_link_clk_src` | `0x02278` | `0` (Div only) | `0`=`bi_tcxo`, `1`=`dp3_phy_pll_link` | Driven by PHY 3 `link_clk` (`CFG_RCGR = 0x0101`) |
| `dptx3_pixel0_clk_src` | `0x02290` | `16-bit M/N` | `0`=`bi_tcxo`, `2`=`dp3_phy_pll_vco_div` | Driven by PHY 3 `vco_div_clk` (`CFG_RCGR = 0x2201`) |

**[HW]** Note on Dragon Q8B empirical offset: `0x0af00000 + 0x02218 = 0x0af02218` (`MDSS0 + 0x102218` is `dptx2_pixel0_clk_src`, with `M` at `+0x2220`, `N` at `+0x2224`, `D` at `+0x2228`).

### 1.4 Rational `M`/`N`/`D` Calculation for `dptxN_pixel0_clk_src`

**[HW]** Because widebus (`DATABUS_WIDEN`) is enabled on all SC8280XP DP interfaces, the target RCG output frequency is half the mode pixel clock:
$$f_{\text{rcg\_khz}} = \frac{\text{pclk\_khz}}{2}$$
The parent clock `dpN_phy_pll_vco_div` (`SRC_SEL = 2`) has frequency $f_{\text{parent\_khz}}$ determined by the DP link rate and the PHY's `QSERDES_V5_DP_PHY_VCO_DIV` setting:
- **RBR** (`162000 kHz`): VCO is `4860 MHz`, `VCO_DIV = 1` (`/6`) -> $f_{\text{parent\_khz}} = 810000\text{ kHz}$.
- **HBR** (`270000 kHz`): VCO is `5400 MHz`, `VCO_DIV = 2` (`/4`) -> $f_{\text{parent\_khz}} = 1350000\text{ kHz}$.
- **HBR2** (`540000 kHz`): VCO is `5400 MHz`, `VCO_DIV = 1` (`/6` or `/2` = `1350` / `2700 MHz`) -> $f_{\text{parent\_khz}} = 1350000\text{ kHz}$.
- **HBR3** (`810000 kHz`): VCO is `8100 MHz`, `VCO_DIV = 0` (`/6`) -> $f_{\text{parent\_khz}} = 1350000\text{ kHz}$ (`1350 MHz`).

**[HW]** To program `dptxN_pixel0_clk_src` (`CMD_RCGR` at `rcg_base`):
1. Compute the best rational approximation $M / N \approx f_{\text{rcg\_khz}} / f_{\text{parent\_khz}}$ with $1 \le M < N \le 32767$ (`0x7fff`) using continued-fraction convergent search.
   - *Example (`1920x1080@60` at `pclk = 148500 kHz`, widebus $f_{\text{rcg}} = 74250\text{ kHz}$, parent `1350000 kHz`)*: $74250 / 1350000 = 11 / 200$, so $M = 11$ (`0x000b`), $N = 200$ (`0x00c8`).
2. Write `M` (`rcg_base + 0x08`) = `M & 0xffff` (`0x000b`).
3. Write `N` (`rcg_base + 0x0c`) = `(~(N - M)) & 0xffff` (`~189 & 0xffff = 0xff42`).
4. Write `D` (`rcg_base + 0x10`) = `(~N) & 0xffff` (`~200 & 0xffff = 0xff37`).
5. Write `CFG_RCGR` (`rcg_base + 0x04`) = `0x00002201` (`MODE = 2` dual-edge M/N, `SRC_SEL = 2` `dpN_phy_pll_vco_div`, `SRC_DIV = 1` div-1).
6. Set bit `0` (`UPDATE`) in `CMD_RCGR` (`rcg_base + 0x00`) and poll `CMD_RCGR` bit `0` (up to `500 us`) until it clears to `0`.

### 1.5 Post-Dividers (`CDIVR`) and Clock Branch Control Registers (`CBCRs`)

**[HW]** Each `dptxN_link_intf_clk` is divided from `dptxN_link_clk_src` by a 4-bit post-divider (`CDIVR` bits `[3:0]` = `div - 1`, set to `1` for divide-by-2 so `link_intf_clk = link_clk / 2`), and each clock branch is enabled via bit `0` (`CLK_ENABLE`) of its `CBCR` (polling bit `31` `CLK_OFF` until it clears to `0`):

| Clock / Divider Name | Offset | Type | Description |
|---|---|---|---|
| `disp_cc_mdss_mdp_clk` | `0x02010` | `CBCR` | Main DPU pixel processing pipeline clock |
| `disp_cc_mdss_mdp_lut_clk` | `0x02020` | `CBCR` | DPU LUT clock |
| `disp_cc_mdss_vsync_clk` | `0x02024` | `CBCR` | DPU VSYNC reference clock (`19.2 MHz`) |
| `disp_cc_mdss_ahb_clk` | `0x02080` | `CBCR` | MDSS/DPU/DP register AHB bus clock |
| `disp_cc_mdss_dptx0_pixel0_clk` | `0x0202c` | `CBCR` | `DP0` stream 0 pixel clock |
| `disp_cc_mdss_dptx0_link_clk` | `0x02030` | `CBCR` | `DP0` main link symbol clock |
| `disp_cc_mdss_dptx0_link_div_clk_src` | `0x02154` | `CDIVR` | `DP0` link intf divider (`bits [3:0] = 1` for `/2`) |
| `disp_cc_mdss_dptx0_link_intf_clk` | `0x02034` | `CBCR` | `DP0` PHY-to-controller link interface clock |
| `disp_cc_mdss_dptx0_usb_router_link_intf_clk` | `0x02038` | `CBCR` | `DP0` USB4/Type-C router link interface clock |
| `disp_cc_mdss_dptx0_pixel1_clk` | `0x0203c` | `CBCR` | `DP0` stream 1 pixel clock |
| `disp_cc_mdss_dptx0_aux_clk` | `0x02040` | `CBCR` | `DP0` AUX channel clock (`19.2 MHz`) |
| `disp_cc_mdss_dptx1_pixel0_clk` | `0x02044` | `CBCR` | `DP1` stream 0 pixel clock |
| `disp_cc_mdss_dptx1_link_clk` | `0x02048` | `CBCR` | `DP1` main link symbol clock |
| `disp_cc_mdss_dptx1_link_div_clk_src` | `0x021e4` | `CDIVR` | `DP1` link intf divider (`bits [3:0] = 1` for `/2`) |
| `disp_cc_mdss_dptx1_link_intf_clk` | `0x0204c` | `CBCR` | `DP1` PHY-to-controller link interface clock |
| `disp_cc_mdss_dptx1_usb_router_link_intf_clk` | `0x02050` | `CBCR` | `DP1` USB4/Type-C router link interface clock |
| `disp_cc_mdss_dptx1_pixel1_clk` | `0x02054` | `CBCR` | `DP1` stream 1 pixel clock |
| `disp_cc_mdss_dptx1_aux_clk` | `0x02058` | `CBCR` | `DP1` AUX channel clock (`19.2 MHz`) |
| `disp_cc_mdss_dptx2_pixel0_clk` | `0x0205c` | `CBCR` | `DP2` stream 0 pixel clock (Dragon Q8B HDMI) |
| `disp_cc_mdss_dptx2_pixel1_clk` | `0x02060` | `CBCR` | `DP2` stream 1 pixel clock |
| `disp_cc_mdss_dptx2_link_clk` | `0x02064` | `CBCR` | `DP2` main link symbol clock |
| `disp_cc_mdss_dptx2_link_div_clk_src` | `0x02244` | `CDIVR` | `DP2` link intf divider (`bits [3:0] = 1` for `/2`) |
| `disp_cc_mdss_dptx2_link_intf_clk` | `0x02068` | `CBCR` | `DP2` PHY-to-controller link interface clock |
| `disp_cc_mdss_dptx2_usb_router_link_intf_clk` | `0x0206c` | `CBCR` | `DP2` router link interface clock |
| `disp_cc_mdss_dptx2_aux_clk` | `0x02070` | `CBCR` | `DP2` AUX channel clock (`19.2 MHz`) |
| `disp_cc_mdss_dptx3_pixel0_clk` | `0x02074` | `CBCR` | `DP3` stream 0 pixel clock |
| `disp_cc_mdss_dptx3_link_clk` | `0x02078` | `CBCR` | `DP3` main link symbol clock |
| `disp_cc_mdss_dptx3_link_div_clk_src` | `0x0228c` | `CDIVR` | `DP3` link intf divider (`bits [3:0] = 1` for `/2`) |
| `disp_cc_mdss_dptx3_link_intf_clk` | `0x0207c` | `CBCR` | `DP3` PHY-to-controller link interface clock |
| `disp_cc_mdss_dptx3_aux_clk` | `0x02084` | `CBCR` | `DP3` AUX channel clock (`19.2 MHz`) |

---

## 2. QMP V5 5nm USB4/USB3/DP Combo PHY (`0x088eb000` & `0x08903000`)

**[HW]** On SC8280XP, USB-C Port 0 (`0x088eb000`, paired with `MDSS0_DP0`) and USB-C Port 1 (`0x08903000`, paired with `MDSS0_DP1`) use the QMP V5 5nm Combo PHY. Sub-block offsets from `PHY_BASE` are:
- `DP_COM`: `+0x0000`
- `TXA` (Lanes 0/1): `+0x0400` (`TXA_LANE0` at `+0x0400`, `TXA_LANE1` at `+0x0600` when DP-Dedicated or `+0x0400` in V5 layout)
- `RXA` (Lanes 0/1): `+0x0600`
- `TXB` (Lanes 2/3): `+0x0a00`
- `RXB` (Lanes 2/3): `+0x0c00`
- `USB3_SERDES`: `+0x1000`
- `USB3_PCS_MISC`: `+0x1200`
- `USB3_PCS`: `+0x1400`
- `USB3_PCS_USB`: `+0x1700`
- `DP_SERDES`: `+0x2000`
- `DP_DP_PHY`: `+0x2200`

*(Note: Dedicated DP2 PHY at `0x0aec2000` and DP3 PHY at `0x0aec5000` have `DP_SERDES` at `+0x0000`, `TXA` at `+0x0200`, `TXB` at `+0x0600`, and `DP_DP_PHY` at `+0x0a00`.)*

### 2.1 `DP_COM` (`PHY_BASE + 0x0000`) Type-C Multiplexing & Orientation Registers

**[HW]** The `DP_COM` block controls whether the 4 SuperSpeed differential pairs on the USB-C receptacle are routed to USB3 only, DisplayPort only (4 lanes: Type-C Pin Assignment C or E), or concurrent USB3 + DisplayPort (2 lanes USB3 + 2 lanes DP: Type-C Pin Assignment D or F), as well as plug orientation (Normal vs Reversed):

| Offset | Register Name | Access | Description |
|---|---|---|---|
| `0x000` | `DP_COM_PHY_MODE_CTRL` | RW | Bit `0` `USB3_MODE` (`0x01`), bit `1` `DP_MODE` (`0x02`); `0x03` = concurrent USB3 + DP |
| `0x004` | `DP_COM_SW_RESET` | RW | Bit `0` `SW_RESET` (`1` = reset combo common block, `0` = release) |
| `0x008` | `DP_COM_POWER_DOWN_CTRL` | RW | Bit `0` `SW_PWRDN` (`1` = power up combo common block) |
| `0x00c` | `DP_COM_SWI_CTRL` | RW | Software interface control (`0x00`) |
| `0x010` | `DP_COM_TYPEC_CTRL` | RW | Bit `0` `SW_PORTSELECT_VAL` (`0` = Normal orientation, `1` = Reversed orientation), bit `1` `SW_PORTSELECT_MUX` (`1` = software controls orientation) -> `0x02` Normal, `0x03` Reversed |
| `0x014` | `DP_COM_TYPEC_PWRDN_CTRL` | RW | Powerdown control during Type-C switch |
| `0x01c` | `DP_COM_RESET_OVRD_CTRL` | RW | Reset override control (`0x00`) |

**[HW]** Type-C Pin Assignment and Orientation Programming in `DP_COM` and `DP_DP_PHY`:
1. Write `DP_COM_POWER_DOWN_CTRL` (`+0x0008`) = `0x01`.
2. Write `DP_COM_PHY_MODE_CTRL` (`+0x0000`):
   - `0x01` for USB3-only mode.
   - `0x02` for 4-lane DP-only mode (Pin Assignment C or E).
   - `0x03` for concurrent 2-lane USB3 + 2-lane DP mode (Pin Assignment D or F).
3. Write `DP_COM_TYPEC_CTRL` (`+0x0010`):
   - `0x02` (`SW_PORTSELECT_MUX`) for Normal plug orientation (`CC1`).
   - `0x03` (`SW_PORTSELECT_MUX | SW_PORTSELECT_VAL`) for Reversed plug orientation (`CC2`).
4. In `DP_DP_PHY` (`PHY_BASE + 0x2200`):
   - Register `QSERDES_DP_PHY_MODE` (`+0x221c`): write `0x4c` for Reversed orientation (or dedicated eDP/DP PHY), or `0x5c` for Normal orientation on Type-C Combo PHY.
   - Register `QSERDES_DP_PHY_PD_CTL` (`+0x2218`):
     - For 4-lane DP (or 1/2-lane in Reversed orientation using `TXA` lanes 0/1): enable both `LANE_0_1_PWRDN` (`0x08`) and `LANE_2_3_PWRDN` (`0x10`) -> write `0x7d` (`PWRDN | AUX_PWRDN | LANE_0_1_PWRDN | LANE_2_3_PWRDN | PLL_PWRDN | DP_CLAMP_EN`).
     - For 1- or 2-lane DP in Normal orientation (which uses physical lanes `3, 2` on `TXB`): enable only `LANE_2_3_PWRDN` (`0x10`) -> write `0x75` (`0x65 | 0x10`), leaving `TXA` lanes `0, 1` to USB3.
     - For 1- or 2-lane DP in Reversed orientation (which uses physical lanes `0, 1` on `TXA`): enable only `LANE_0_1_PWRDN` (`0x08`) -> write `0x6d` (`0x65 | 0x08`), leaving `TXB` lanes `2, 3` to USB3.

### 2.2 QMP V5 DP AUX Initialization Sequence (`DP_DP_PHY` at `PHY_BASE + 0x2200`)

**[HW]** To initialize the DisplayPort AUX transceiver in `DP_DP_PHY` (`+0x2200`):
1. Write `QSERDES_DP_PHY_PD_CTL` (`+0x2218`) = `0x02` (`PSR_PWRDN`), then write `0x7d` (`PWRDN | AUX_PWRDN | LANE_0_1_PWRDN | LANE_2_3_PWRDN | PLL_PWRDN | DP_CLAMP_EN`).
2. Write `QSERDES_V4_COM_BIAS_EN_CLKBUFLR_EN` (`+0x2044`) = `0x17` (`BIAS_EN | BIAS_EN_MUX | CLKBUF_L_EN | EN_SYSCLK_TX_SEL`).
3. Program the 10 AUX configuration registers in `DP_DP_PHY` (`+0x2200`):
   - `+0x020` `QSERDES_DP_PHY_AUX_CFG0` = `0x00`
   - `+0x024` `QSERDES_DP_PHY_AUX_CFG1` = `0x13` (if an AUX error occurs during DPCD reads, cycle through calibration values `{0x20, 0x13, 0x23, 0x1d}`)
   - `+0x028` `QSERDES_DP_PHY_AUX_CFG2` = `0x24` (`0xa4` on V4, `0x24` on V5)
   - `+0x02c` `QSERDES_DP_PHY_AUX_CFG3` = `0x00`
   - `+0x030` `QSERDES_DP_PHY_AUX_CFG4` = `0x0a`
   - `+0x034` `QSERDES_DP_PHY_AUX_CFG5` = `0x26`
   - `+0x038` `QSERDES_DP_PHY_AUX_CFG6` = `0x0a`
   - `+0x03c` `QSERDES_DP_PHY_AUX_CFG7` = `0x03`
   - `+0x040` `QSERDES_DP_PHY_AUX_CFG8` = `0xb7`
   - `+0x044` `QSERDES_DP_PHY_AUX_CFG9` = `0x03`
4. Clear `DP_COM_RESET_OVRD_CTRL` (`+0x001c` = `0x00`).

### 2.3 QMP V5 DP SerDes Common PLL Tables (`DP_SERDES` at `PHY_BASE + 0x2000`)

**[HW]** First program the common 23-entry QMP V5 DP SerDes PLL initialization table into `DP_SERDES` (`PHY_BASE + 0x2000`):

| Offset from `+0x2000` | Register Name | Value | Offset from `+0x2000` | Register Name | Value |
|---|---|---|---|---|---|
| `0x184` | `COM_SVS_MODE_CLK_SEL` | `0x05` | `0x078` | `COM_CP_CTRL_MODE1` | `0x06` |
| `0x094` | `COM_SYSCLK_EN_SEL` | `0x3b` | `0x17c` | `COM_CMN_CONFIG` | `0x02` |
| `0x04c` | `COM_SYS_CLK_CTRL` | `0x02` | `0x0ec` | `COM_INTEGLOOP_GAIN0_MODE0` | `0x3f` |
| `0x048` | `COM_CLK_ENABLE1` | `0x0c` | `0x0f0` | `COM_INTEGLOOP_GAIN1_MODE0` | `0x00` |
| `0x050` | `COM_SYSCLK_BUF_ENABLE` | `0x06` | `0x10c` | `COM_VCO_TUNE_MAP` | `0x02` |
| `0x154` | `COM_CLK_SELECT` | `0x30` | `0x0cc` | `COM_DIV_FRAC_START1_MODE0` | `0x00` |
| `0x058` | `COM_PLL_IVCO` | `0x0f` | `0x00c` | `COM_BG_TIMER` | `0x0a` |
| `0x084` | `COM_PLL_CCTRL_MODE0` | `0x36` | `0x168` | `COM_CORECLK_DIV_MODE0` | `0x0a` |
| `0x088` | `COM_PLL_CCTRL_MODE1` | `0x36` | `0x108` | `COM_VCO_TUNE_CTRL` | `0x00` |
| `0x07c` | `COM_PLL_RCTRL_MODE0` | `0x16` | `0x044` | `COM_BIAS_EN_CLKBUFLR_EN` | `0x17` |
| `0x080` | `COM_PLL_RCTRL_MODE1` | `0x16` | `0x174` | `COM_CORE_CLK_EN` | `0x1f` |
| `0x074` | `COM_CP_CTRL_MODE0` | `0x06` | — | — | — |

**[HW]** Next, program the link-rate-specific SerDes PLL registers in `DP_SERDES` (`+0x2000`) and `QSERDES_V5_DP_PHY_VCO_DIV` (`DP_DP_PHY + 0x070` = `PHY_BASE + 0x2270`) according to the target link rate:

| Register (Offset from `DP_SERDES +0x2000`) | RBR (`1.62 Gbps`) | HBR (`2.70 Gbps`) | HBR2 (`5.40 Gbps`) | HBR3 (`8.10 Gbps`) |
|---|---|---|---|---|
| `0x158` `COM_HSCLK_SEL` | `0x05` | `0x03` | `0x01` | `0x00` |
| `0x0bc` `COM_DEC_START_MODE0` | `0x69` | `0x69` | `0x8c` | `0x69` |
| `0x0d0` `COM_DIV_FRAC_START2_MODE0` | `0x80` | `0x80` | `0x00` | `0x80` |
| `0x0d4` `COM_DIV_FRAC_START3_MODE0` | `0x07` | `0x07` | `0x0a` | `0x07` |
| `0x0ac` `COM_LOCK_CMP1_MODE0` | `0x6f` | `0x0f` | `0x1f` | `0x2f` |
| `0x0b0` `COM_LOCK_CMP2_MODE0` | `0x08` | `0x0e` | `0x1c` | `0x2a` |
| `0x0a4` `COM_LOCK_CMP_EN` | `0x04` | `0x08` | `0x08` | `0x08` |
| `DP_DP_PHY + 0x014` `DP_PHY_CFG_1` | `0x00` | `0x00` | `0x00` | `0x01` |
| `DP_DP_PHY + 0x070` `V5_DP_PHY_VCO_DIV` | `0x01` | `0x02` | `0x01` | `0x00` |

### 2.4 QMP V5 5nm DP TX Lane Initialization Table (`TXA` `+0x0400` & `TXB` `+0x0a00`)

**[HW]** Program the 11-entry QMP V5 5nm DP TX lane initialization table into both `TXA` (`PHY_BASE + 0x0400`) and `TXB` (`PHY_BASE + 0x0a00`):

| Offset from `TX_BASE` | Register Name | Value |
|---|---|---|
| `0x0b0` | `QSERDES_V5_5NM_TX_VMODE_CTRL1` | `0x40` |
| `0x020` | `QSERDES_V5_5NM_TX_PRE_STALL_LDO_BOOST_EN` | `0x30` |
| `0x0e4` | `QSERDES_V5_5NM_TX_INTERFACE_SELECT` | `0x3b` |
| `0x008` | `QSERDES_V5_5NM_TX_CLKBUF_ENABLE` | `0x0f` |
| `0x01c` | `QSERDES_V5_5NM_TX_RESET_TSYNC_EN` | `0x03` |
| `0x0ac` | `QSERDES_V5_5NM_TX_TRAN_DRVR_EMP_EN` | `0x0f` |
| `0x054` | `QSERDES_V5_5NM_TX_PARRATE_REC_DETECT_IDLE_EN` | `0x00` |
| `0x078` | `QSERDES_V5_5NM_TX_LANE_MODE_1` | `0x00` |
| `0x030` | `QSERDES_V5_5NM_TX_RES_CODE_LANE_OFFSET_TX` | `0x11` |
| `0x034` | `QSERDES_V5_5NM_TX_RES_CODE_LANE_OFFSET_RX` | `0x11` |
| `0x0d8` | `QSERDES_V5_5NM_TX_TX_BAND` | `0x04` |

### 2.5 DP PHY Voltage Swing and Pre-Emphasis Lookup Tables (`QMP V5`)

**[HW]** During Link Training (or before starting the PHY), write the voltage swing code to `QSERDES_V5_5NM_TX_TX_DRV_LVL` (`TX_BASE + 0x014`) and the pre-emphasis code to `QSERDES_V5_5NM_TX_TX_EMP_POST1_LVL` (`TX_BASE + 0x00c`) in both `TXA` (`+0x0400`) and `TXB` (`+0x0a00`), and write `0x0a` to `QSERDES_V5_5NM_TX_TX_POL_INV` (`TX_BASE + 0x050`) and `0x03` (`TRANSCEIVER_BIAS_EN` at `TX_BASE + 0x048`):

#### For RBR (`1.62 Gbps`) and HBR (`2.70 Gbps`):

| Swing \ Pre-Emphasis | Pre-Emphasis `0` (`DRV` / `EMP`) | Pre-Emphasis `1` (`DRV` / `EMP`) | Pre-Emphasis `2` (`DRV` / `EMP`) | Pre-Emphasis `3` (`DRV` / `EMP`) |
|---|---|---|---|---|
| **Swing `0`** | `0x28` / `0x20` | `0x2f` / `0x2d` | `0x36` / `0x34` | `0x3f` / `0x3a` |
| **Swing `1`** | `0x31` / `0x20` | `0x3e` / `0x2e` | `0x3f` / `0x35` | *(invalid)* |
| **Swing `2`** | `0x36` / `0x20` | `0x3f` / `0x2e` | *(invalid)* | *(invalid)* |
| **Swing `3`** | `0x3f` / `0x24` | *(invalid)* | *(invalid)* | *(invalid)* |

#### For HBR2 (`5.40 Gbps`) and HBR3 (`8.10 Gbps`):

| Swing \ Pre-Emphasis | Pre-Emphasis `0` (`DRV` / `EMP`) | Pre-Emphasis `1` (`DRV` / `EMP`) | Pre-Emphasis `2` (`DRV` / `EMP`) | Pre-Emphasis `3` (`DRV` / `EMP`) |
|---|---|---|---|---|
| **Swing `0`** | `0x22` / `0x20` | `0x32` / `0x2c` | `0x36` / `0x35` | `0x3a` / `0x3b` |
| **Swing `1`** | `0x29` / `0x22` | `0x39` / `0x2e` | `0x3f` / `0x36` | *(invalid)* |
| **Swing `2`** | `0x30` / `0x22` | `0x3f` / `0x31` | *(invalid)* | *(invalid)* |
| **Swing `3`** | `0x3f` / `0x24` | *(invalid)* | *(invalid)* | *(invalid)* |

*(Note: On the dedicated `DP2` / `DP3` eDP PHYs at `0x0aec2000` / `0x0aec5000`, `TX_EMP_POST1_LVL` is at `TX_BASE + 0x004` and `TX_DRV_LVL` is at `TX_BASE + 0x014`.)*

### 2.6 Starting and Locking the DP PHY PLL and TX Lanes

**[HW]** After writing the SerDes, rate, and TX tables:
1. Write the sequence of `QSERDES_DP_PHY_CFG` (`DP_DP_PHY + 0x010` = `PHY_BASE + 0x2210`) transitions to trigger PLL lock and symbol alignment:
   - Write `0x01`, then `0x05`, then `0x01`, then `0x09` to `QSERDES_DP_PHY_CFG` (`+0x2210`).
2. Write `0x20` (`RESETSM_CNTRL`) to `QSERDES_V4_COM_RESETSM_CNTRL` (`DP_SERDES + 0x09c` = `PHY_BASE + 0x209c`).
3. Poll `QSERDES_V4_COM_C_READY_STATUS` (`DP_SERDES + 0x178` = `PHY_BASE + 0x2178`) bit `0` (`C_READY`, up to `10000 us`) until it reads `1`.
4. Poll `QSERDES_V4_COM_CMN_STATUS` (`DP_SERDES + 0x140` = `PHY_BASE + 0x2140`) bit `0` (or bit `1`) for PLL lock.
5. Write `0x19` to `QSERDES_DP_PHY_CFG` (`+0x2210`).
6. Poll `QSERDES_V5_DP_PHY_STATUS` (`DP_DP_PHY + 0x0dc` = `PHY_BASE + 0x22dc`) bit `1` (`PHY_READY`, up to `10000 us`) until it reads `1`.
7. Write `0x18` then `0x19` to `QSERDES_DP_PHY_CFG` (`+0x2210`) and re-verify `QSERDES_V5_DP_PHY_STATUS` bit `1` == `1`.

---

## 3. USB3 SuperSpeed Initialization Tables for Concurrent Mode (`SC8280XP`)

**[HW]** When operating the QMP V5 5nm Combo PHY in USB3-only (`DP_COM_PHY_MODE_CTRL = 0x01`) or concurrent USB3 + 2-lane DP mode (`DP_COM_PHY_MODE_CTRL = 0x03`, Pin Assignment D/F), the USB3 side of the PHY (`USB3_SERDES` at `+0x1000`, `TXA`/`TXB`, `RXA`/`RXB`, and `USB3_PCS` at `+0x1400`) uses the following SC8280XP 5nm register tables:

### 3.1 `USB3_SERDES` (`PHY_BASE + 0x1000`, QMP V5 5nm USB3 SerDes Table)

| Offset | Value | Offset | Value | Offset | Value | Offset | Value |
|---|---|---|---|---|---|---|---|
| `0x010` | `0x01` | `0x034` | `0x06` | `0x080` | `0x16` | `0x0c4` | `0x82` |
| `0x01c` | `0x31` | `0x048` | `0x0c` | `0x084` | `0x36` | `0x0d0` | `0x55` |
| `0x020` | `0x01` | `0x074` | `0x06` | `0x088` | `0x36` | `0x0d4` | `0x55` |
| `0x024` | `0xde` | `0x078` | `0x06` | `0x094` | `0x1a` | `0x0dc` | `0x55` |
| `0x028` | `0x07` | `0x07c` | `0x16` | `0x0a8` | `0x14` | `0x0e0` | `0x55` |
| `0x030` | `0x97` | `0x0ac` | `0x14` | `0x0b0` | `0x34` | `0x0b4` | `0x28` |
| `0x0b8` | `0x68` | `0x0bc` | `0x41` | `0x10c` | `0x02` | `0x158` | `0x01` |
| `0x16c` | `0x08` | `0x17c` | `0x16` | `0x1ac` | `0xb9` | `0x1b0` | `0x1e` |
| `0x1b4` | `0xb9` | `0x1b8` | `0x1e` | — | — | — | — |

### 3.2 `USB3_TX` (`TXA` `+0x0400` / `TXB` `+0x0a00`) and `USB3_RX` (`RXA` `+0x0600` / `RXB` `+0x0c00`)

**[HW]** `USB3_TX` initialization values (QMP V5 5nm USB3 TX table):
- `0x030` (`RES_CODE_LANE_OFFSET_TX`) = `0x1f`, `0x034` (`RES_CODE_LANE_OFFSET_RX`) = `0x0b`, `0x078` (`LANE_MODE_1`) = `0xf5`, `0x080` (`LANE_MODE_3`) = `0x3f`, `0x090` (`RCV_DETECT_LVL_2`) = `0x12`, `0x0cc` (`PI_QEC_CTRL`) = `0x21`.

**[HW]** `USB3_RX` initialization values (QMP V5 5nm USB3 RX table):
- `0x000`=`0x2f`, `0x028`=`0xff`, `0x02c`=`0x0f`, `0x050`=`0x08`, `0x054`=`0x08`, `0x060`=`0x08`, `0x064`=`0x08`, `0x070`=`0x00`, `0x074`=`0x00`, `0x080`=`0x04`, `0x084`=`0x04`, `0x09c`=`0x0a`, `0x0a0`=`0x0a`, `0x0ac`=`0x06`, `0x0b0`=`0x06`, `0x0bc`=`0x99`, `0x0d4`=`0xa0`, `0x198`=`0x54`, `0x19c`=`0x0f`, `0x1bc`=`0x13`, `0x1c8`=`0x0f`, `0x1cc`=`0x4b`, `0x1d0`=`0x0a`, `0x1d4`=`0x07`, `0x1e0`=`0x47`, `0x1ec`=`0x04`, `0x1f4`=`0x0e`, `0x228`=`0xdf`, `0x22c`=`0xdf`, `0x230`=`0xff`, `0x234`=`0xa9`, `0x238`=`0xaa`, `0x23c`=`0x74`, `0x240`=`0xff`, `0x248`=`0xdc`, `0x24c`=`0x5c`, `0x250`=`0x9c`, `0x254`=`0x1d`, `0x258`=`0x09`, `0x25c`=`0x3c`, `0x290`=`0x04`, `0x294`=`0x38`, `0x298`=`0x0c`.

### 3.3 `USB3_PCS` (`PHY_BASE + 0x1400`, QMP V5 5nm USB3 PCS Table)

**[HW]** `USB3_PCS` initialization values:
- `0x0c4` (`LOCK_DETECT_CONFIG1`) = `0xd0`, `0x0c8` (`LOCK_DETECT_CONFIG2`) = `0x07`, `0x0cc` (`LOCK_DETECT_CONFIG3`) = `0x20`, `0x0d8` (`LOCK_DETECT_CONFIG6`) = `0x13`, `0x0dc` (`REFGEN_REQ_CONFIG1`) = `0x21`, `0x188` (`RX_SIGDET_LVL`) = `0xaa`, `0x190` (`RCVR_DTCT_DLY_P1U2_L`) = `0xe7`, `0x194` (`RCVR_DTCT_DLY_P1U2_H`) = `0x03`, `0x1b0` (`CDR_RESET_TIME`) = `0x0a`, `0x1c0` (`ALIGN_DETECT_CONFIG1`) = `0x88`, `0x1c4` (`ALIGN_DETECT_CONFIG2`) = `0x13`, `0x1d0` (`PCS_TX_RX_CONFIG`) = `0x0c`, `0x1dc` (`EQ_CONFIG1`) = `0x4b`, `0x1ec` (`EQ_CONFIG5`) = `0x10`, and in `USB3_PCS_USB` (`+0x1700`): `0x018` (`LFPS_DET_HIGH_COUNT_VAL`) = `0xf8`, `0x0ec` (`RXEQTRAINING_DFE_TIME_S2`) = `0x07`.
- To start `USB3_PCS`: write `0x03` to `QPHY_V5_PCS_POWER_DOWN_CONTROL` (`+0x1440`), release `QPHY_V5_PCS_SW_RESET` (`+0x1400` = `0x00`), write `0x03` (`SERDES_START | PCS_START`) to `QPHY_V5_PCS_START_CONTROL` (`+0x1444`), and poll `QPHY_V5_PCS_PCS_STATUS1` (`+0x1414`) bit `0` (`PHYSTATUS`) until it clears to `0`.
