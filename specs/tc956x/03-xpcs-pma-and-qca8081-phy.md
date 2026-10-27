# 03 — XPCS, SerDes PMA, and QCA8081 2.5G PHY

## 1. Sub-Block Architecture and Data Path

**[HW]** Between each XGMAC2 MAC core and its external Qualcomm Atheros QCA8081 Copper PHY sits a two-stage on-chip serial link subsystem inside BAR4:

$$\text{XGMAC2 Core} \xrightarrow{\text{GMII / XGMII}} \text{DesignWare XPCS} \xrightarrow{\text{10-bit TBI}} \text{Toshiba SerDes PMA} \xrightarrow{\text{2500BASE-X / SGMII Differential Pair}} \text{QCA8081 SerDes (`0x1d`) \& Copper PHY (`0x1c`)}$$

**[HW]** BAR4 base offsets for the two ports are:

| Port / Physical Function | XGMAC2 Base | XPCS Base (`XGMAC + 0x3A00`) | SerDes PMA Base (`XGMAC + 0x4000`) | External PHY MDIO Addr | QCA8081 SerDes Companion MDIO Addr |
|---|---|---|---|---|---|
| Port 0 (`0004:03:00.0`) | `BAR4 + 0x040000` | `BAR4 + 0x043A00` | `BAR4 + 0x044000` | `0x1c` (`28`) | `0x1d` (`29`) |
| Port 1 (`0004:03:00.1`) | `BAR4 + 0x048000` | `BAR4 + 0x04BA00` | `BAR4 + 0x04C000` | `0x1c` (`28`, on Port 1 MDIO bus) | `0x1d` (`29`, on Port 1 MDIO bus) |

---

## 2. Synopsys DesignWare XPCS (`XGMAC + 0x3A00`)

### 2.1 Indirect 21-Bit Clause 45 Viewport Mechanism

**[HW]** The Synopsys DesignWare XPCS block is organized internally as a Clause 45 MDIO-style register space addressed by a 5-bit MMD (Device Address `mmd`, bits `[20:16]`) and a 16-bit Register Offset (`reg`, bits `[15:0]`), forming a 21-bit internal CSR index:

$$\text{csr} = ((\text{mmd} \mathbin{\&} \text{0x1F}) \ll 16) \mathbin{|} (\text{reg} \mathbin{\&} \text{0xFFFF})$$

**[HW]** Because the XPCS aperture in BAR4 (`XGMAC + 0x3A00`) is only 1 KB (`0x400` bytes = 256 32-bit words), the hardware uses a **paged indirect access protocol** where word `0xFF` (byte offset `XPCS + 0x03FC`) is the **Page Select Register**:

1. **Compute Page and Offset**:
   - $\text{page} = (\text{csr} \gg 8) \mathbin{\&} \text{0x1FFF}$ (upper 13 bits `[20:8]`)
   - $\text{offset} = (\text{csr} \mathbin{\&} \text{0xFF}) \times 4$ (lower 8 bits `[7:0]` scaled to 32-bit word stride, in `0x000..0x3F8`)
2. **XPCS Indirect Read(`mmd`, `reg`)**:
   - Write `page` (32-bit write) to `XPCS_BASE + 0x03FC`.
   - Read and return the 32-bit (16-bit significant `[15:0]`) value from `XPCS_BASE + offset`.
3. **XPCS Indirect Write(`mmd`, `reg`, `val`)**:
   - Write `page` (32-bit write) to `XPCS_BASE + 0x03FC`.
   - Write `val` (32-bit write) to `XPCS_BASE + offset`.

**[HW]** Callers must serialize access to `XPCS_BASE + 0x03FC` with a per-port lock so concurrent link polling and reconfiguration cannot interleave page writes.

### 2.2 Complete XPCS Register Map Used on TC956x

**[HW]** All XPCS registers used for identification, SGMII (10M/100M/1000M), and 2500BASE-X (2500M) operation:

| MMD (`devad`) | Reg (`16-bit`) | 21-Bit `csr` | Page (`+0x3FC`) | Offset (`+0x...`) | Register Name | Bits & Description |
|---|---|---|---|---|---|---|
| `0x03` (PCS) | `0x0000` | `0x030000` | `0x0300` | `+0x000` | `SR_XS_PCS_CTRL1` | `[15]` `RST` (`0x8000` soft reset), `[11]` `LPM` (`0x0800` low power). |
| `0x03` (PCS) | `0x0001` | `0x030001` | `0x0300` | `+0x004` | `SR_XS_PCS_STS1` | `[2]` `LINK_STAT` (`0x0004`: `1` = PCS link up). |
| `0x03` (PCS) | `0x0002` | `0x030002` | `0x0300` | `+0x008` | `SR_XS_PCS_DEV_ID1` | `[15:0]` Upper 16 bits of XPCS Synopsys ID (`0x7996`). |
| `0x03` (PCS) | `0x0003` | `0x030003` | `0x0300` | `+0x00C` | `SR_XS_PCS_DEV_ID2` | `[15:0]` Lower 16 bits of XPCS Synopsys ID (`0xCED0`; full 32-bit ID = `0x7996CED0`). |
| `0x03` (PCS) | `0x0007` | `0x030007` | `0x0300` | `+0x01C` | `SR_XS_PCS_CTRL2` | `[3:0]` `PCS_TYPE_SEL` (`0` = 10GBASE-R, `1` = 10GBASE-X). |
| `0x03` (PCS) | `0x8000` | `0x038000` | `0x0380` | `+0x000` | `VR_XS_PCS_DIG_CTRL1` | `[15]` `VR_RST` (`0x8000`: Vendor-specific soft reset, self-clearing), `[9]` `USXG_EN` (`0x0200`: USXGMII enable; **must be `0` for 2500BASE-X and SGMII**), `[8]` `INIT` (`0x0100`), `[2]` `EN_2_5G_MODE` (`0x0004`: **`1` = enable 2.5G GMII/2500BASE-X mode, `0` = 1G/100M/10M SGMII mode**), `[1]` `BYP_PWRUP` (`0x0002`). |
| `0x03` (PCS) | `0x8007` | `0x038007` | `0x0380` | `+0x01C` | `VR_XS_PCS_KR_CTRL` | `[12:10]` `USXG_MODE` (unused when `USXG_EN = 0`). |
| `0x1E` (VEND1) | `0x0002` | `0x1E0002` | `0x1E00` | `+0x008` | `SR_VEND1_ID1` | `[15:0]` Vendor1 ID1 (`0x7996`). |
| `0x1E` (VEND1) | `0x0003` | `0x1E0003` | `0x1E00` | `+0x00C` | `SR_VEND1_ID2` | `[15:0]` Vendor1 ID2 (`0xCED0`). |
| `0x1F` (VEND2 / MII) | `0x0000` | `0x1F0000` | `0x1F00` | `+0x000` | `SR_MII_CTRL` | **Clause 37 / SGMII MII Control**:<br>- `[15]` `RST` (`0x8000`)<br>- `[14]` `LOOPBACK` (`0x4000`)<br>- **`[13]` `SS13` (`0x2000`: Speed Select LSB: `1` = 100M, `0` = 10M/1000M)**<br>- **`[12]` `AN_ENABLE` (`0x1000`: In-band Clause 37 / SGMII Autonegotiation Enable; `0` on TC956x!)**<br>- `[9]` `RESTART_AN` (`0x0200`)<br>- **`[8]` `DUPLEX_MODE` (`0x0100`: `1` = Full Duplex, `0` = Half Duplex)**<br>- **`[6]` `SS6` (`0x0040`: Speed Select MSB: `1` = 1000M/2500M, `0` = 10M/100M)**<br>- `[5]` `SS5` (`0x0020`). |
| `0x1F` (VEND2 / MII) | `0x0001` | `0x1F0001` | `0x1F00` | `+0x004` | `SR_MII_STS` | `[5]` `AN_COMPLETE` (`0x0020`), **`[2]` `LINK_UP` (`0x0004`: `1` = XPCS SGMII/2500BASE-X link up, observed `0x000001AD` on Q8B)**. |
| `0x1F` (VEND2 / MII) | `0x0004` | `0x1F0004` | `0x1F00` | `+0x010` | `SR_MII_AN_ADV` | `[6:5]` `FD` / `HD` capability advertisement. |
| `0x1F` (VEND2 / MII) | `0x8000` | `0x1F8000` | `0x1F80` | `+0x000` | `VR_MII_DIG_CTRL1` | `[15]` `VR_RST` (`0x8000`), `[9]` `MAC_AUTO_SW` (`0x0200`: automatic speed switch from SGMII in-band AN; **`0` when `AN_ENABLE = 0`**), **`[2]` `EN_2_5G_MODE` (`0x0004`: `1` = 2500BASE-X 2.5G mode, `0` = SGMII 10M/100M/1000M)**. |
| `0x1F` (VEND2 / MII) | `0x8001` | `0x1F8001` | `0x1F80` | `+0x004` | `VR_MII_AN_CTRL` | `[8]` `MII_CTRL` (`0x0100`: `0` = 4-bit MII, `1` = 8-bit GMII), `[4]` `SGMII_LINK_STS` (`0x0010`), `[3]` `TX_CONFIG` (`0x0008`: `0` = MAC side SGMII), **`[2:1]` `PCS_MODE` (`0x0006`: `0` = 1000BASE-X / 2500BASE-X, `2` (`0x0004`) = SGMII)**, `[0]` `MII_AN_INTR_EN` (`0x0001`). |
| `0x1F` (VEND2 / MII) | `0x8002` | `0x1F8002` | `0x1F80` | `+0x008` | `VR_MII_AN_INTR_STS` | `[4]` `CL37_ANCMPLT_INTR`, `[3:2]` `LP_SPEED` (`0` = 10M, `1` = 100M, `2` = 1000M), `[1]` `LP_DUPLEX` (`1` = Full), `[0]` `LP_LINK`. |

### 2.3 Exact XPCS Configuration Sequences by Link Speed

**[HW]** On the Radxa Dragon Q8B, the QCA8081 PHY uses **2500BASE-X without in-band Clause 37 autonegotiation** at 2500 Mbps, and **SGMII without in-band autonegotiation reliance (manual speed programming)** at 1000 / 100 / 10 Mbps:

#### Mode A: 2500BASE-T (`2500BASE-X`, 2500 Mbps Full Duplex)
1. **[HW]** In `MMD 0x03`, `VR_XS_PCS_DIG_CTRL1` (`reg 0x8000`, `csr 0x038000`):
   - Clear `USXG_EN` (bit 9, `0x0200`) and set `EN_2_5G_MODE` (bit 2, `0x0004`).
   - Set `VR_RST` (bit 15, `0x8000`) to apply the mode change, then poll `VR_XS_PCS_DIG_CTRL1` until bit 15 (`VR_RST`) self-clears to `0` (timeout `50 ms`).
2. **[HW]** In `MMD 0x1F`, `VR_MII_AN_CTRL` (`reg 0x8001`, `csr 0x1F8001`):
   - Set `PCS_MODE[2:1]` to `0` (`0x0000`, 1000BASE-X / 2500BASE-X PCS framing; clear bits `[2:1]`).
3. **[HW]** In `MMD 0x1F`, `VR_MII_DIG_CTRL1` (`reg 0x8000`, `csr 0x1F8000`):
   - Set `EN_2_5G_MODE` (bit 2, `0x0004`) and clear `MAC_AUTO_SW` (bit 9, `0x0200`).
4. **[HW]** In `MMD 0x1F`, `SR_MII_CTRL` (`reg 0x0000`, `csr 0x1F0000`):
   - Clear `AN_ENABLE` (bit 12, `0x1000`), clear `SS13` (bit 13, `0x2000`), set `SS6` (bit 6, `0x0040`), and set `DUPLEX_MODE` (bit 8, `0x0100`) $\Rightarrow$ write **`0x0140`**.

#### Mode B: 1000BASE-T / 100BASE-TX / 10BASE-T (`SGMII`)
1. **[HW]** In `MMD 0x03`, `VR_XS_PCS_DIG_CTRL1` (`reg 0x8000`, `csr 0x038000`):
   - If switching down from 2500BASE-X (`EN_2_5G_MODE` was `1`), clear `USXG_EN` (bit 9) and clear `EN_2_5G_MODE` (bit 2), set `VR_RST` (bit 15, `0x8000`), and poll until bit 15 (`VR_RST`) self-clears to `0`.
2. **[HW]** In `MMD 0x1F`, `VR_MII_AN_CTRL` (`reg 0x8001`, `csr 0x1F8001`):
   - Set `PCS_MODE[2:1]` to `2` (`0x0004`, SGMII mode) and clear `TX_CONFIG` (bit 3, MAC-side SGMII).
3. **[HW]** In `MMD 0x1F`, `VR_MII_DIG_CTRL1` (`reg 0x8000`, `csr 0x1F8000`):
   - Clear `EN_2_5G_MODE` (bit 2, `0x0004`) and clear `MAC_AUTO_SW` (bit 9, `0x0200`) $\Rightarrow$ `0x0000`.
4. **[HW]** In `MMD 0x1F`, `SR_MII_CTRL` (`reg 0x0000`, `csr 0x1F0000`):
   - Clear `AN_ENABLE` (bit 12, `0x1000`) and program manual speed (`SS6` bit 6, `SS13` bit 13) and duplex (`DUPLEX_MODE` bit 8):
     - **1000 Mbps Full Duplex**: `SS6 = 1, SS13 = 0, DUPLEX = 1` $\Rightarrow$ **`0x0140`** (observed Q8B live value at 1000BASE-T).
     - **100 Mbps Full Duplex**: `SS6 = 0, SS13 = 1, DUPLEX = 1` $\Rightarrow$ **`0x2100`** (`0x2000` for Half Duplex).
     - **10 Mbps Full Duplex**: `SS6 = 0, SS13 = 0, DUPLEX = 1` $\Rightarrow$ **`0x0100`** (`0x0000` for Half Duplex).

---

## 3. Toshiba SerDes PMA (`XGMAC + 0x4000`)

**[HW]** Each port has a Toshiba SerDes Physical Medium Attachment (PMA) wrapper mapped at `XGMAC + 0x4000`:
- **Port 0 (`0004:03:00.0`)**: `BAR4 + 0x044000`
- **Port 1 (`0004:03:00.1`)**: `BAR4 + 0x04C000`

### 3.1 PMA Register Map (Relative to Port's PMA Base `0x44000` or `0x4C000`)

**[HW]** The PMA contains a global Current Mode Logic (CML) power configuration register and 5 per-slice reference clock / common configuration register sets (`i = 0, 1, 2, 3, 4`):

| PMA Offset | Register Name | Slice Index `i` | Required Value | Description |
|---|---|---|---|---|
| `+0x01B8` | `PMA_CML_GL_PM_CFG0` | Global | `0x00000000` | CML Buffer Power Management Configuration (`0` = power on CML buffer, `1` = power down). |
| `+0x1080 + i * 0x14` | `PMA_HWT_REFCK_R_EN(i)` | `i = 0..4` (`+0x1080`, `+0x1094`, `+0x10A8`, `+0x10BC`, `+0x10D0`) | `0x00000000` | Hardware Test Reference Clock Receiver Enable (`0` = disable external test receiver). |
| `+0x1090 + i * 0x14` | `PMA_HWT_REFCK_TERM_EN(i)` | `i = 0..4` (`+0x1090`, `+0x10A4`, `+0x10B8`, `+0x10CC`, `+0x10E0`) | `0x00000000` | Hardware Test Reference Clock Termination Enable (`0` = disable termination). |
| `+0x1888 + i * 0x08` | `PMA_COMM_CFG_0_1(i)` | `i = 0..4` (`+0x1888`, `+0x1890`, `+0x1898`, `+0x18A0`, `+0x18A8`) | **`0x0001EF04`** | Common Reference Clock Source Configuration:<br>- Bits `[7:0]` `WRITE_DATA = 0x04` (select internal `CLK_REF_I` instead of external `C0_REFCK`)<br>- Bit `[8]` `ENABLE = 1` (`0x00000100`)<br>- Bits `[16:9]` `WRITE_MASK = 0xF7` (`0x0001EE00`, unmasks bit 3 of internal config byte). |

### 3.2 Mandatory 5-Step PMA Initialization Sequence

**[HW]** Whenever a port is initialized at driver attach, or whenever the SerDes line rate changes between 2500BASE-X (`NEMACxCTL.SP_SEL = 4`) and SGMII (`NEMACxCTL.SP_SEL = 5, 6, 7`), the driver **must** execute the following 5-step sequence in exact order:

1. **Ensure `NEMACxCTL` Has Valid `SP_SEL` and Assert PMA Reset**:
   - Ensure `NEMAC0CTL` (`BAR4 + 0x1070`, for Port 0) or `NEMAC1CTL` (`BAR4 + 0x1074`, for Port 1) has `SP_SEL` (`bits [3:0]`) set to the target mode (`4` for 2.5G, `5` for 1G, `6` for 100M, `7` for 10M), `PHY_INF_SEL` (`bits [5:4]`) = `1` (`0x10`), and `LPIHWCLKEN` (`bit 8`) = `1` (`0x100`).
   - Assert the port's PMA reset by setting bit 30 (`0x40000000`) in Function 0's `NRSTCTRL0` (`BAR4 + 0x1008`, `MAC0PMARST` for Port 0) or `NRSTCTRL1` (`BAR4 + 0x1010`, `MAC1PMARST1` for Port 1) to **`1`**.
2. **Power On CML Buffer**:
   - Write `0x00000000` to `PMA_BASE + 0x01B8` (`PMA_CML_GL_PM_CFG0`).
3. **Switch Reference Clock Source from `C0_REFCK` to `CLK_REF_I` Across All 5 Slices (`i = 0..4`)**:
   - For `i = 0, 1, 2, 3, 4`:
     - Write `0x00000000` to `PMA_BASE + 0x1080 + (i * 0x14)` (`PMA_HWT_REFCK_R_EN`).
     - Write `0x00000000` to `PMA_BASE + 0x1090 + (i * 0x14)` (`PMA_HWT_REFCK_TERM_EN`).
     - Write `0x0001EF04` to `PMA_BASE + 0x1888 + (i * 0x08)` (`PMA_COMM_CFG_0_1`).
4. **Deassert PMA Reset**:
   - Clear bit 30 (`~0x40000000`) in Function 0's `NRSTCTRL0` (`BAR4 + 0x1008`, for Port 0) or `NRSTCTRL1` (`BAR4 + 0x1010`, for Port 1) to **`0`**.
5. **Wait for PMA `INIT_DONE`**:
   - Poll `NEMAC0CTL` (`BAR4 + 0x1070`, for Port 0) or `NEMAC1CTL` (`BAR4 + 0x1074`, for Port 1) until bit 21 (`INIT_DONE`, `0x00200000`) reads **`1`** (poll every `10 µs`–`100 µs`, timeout `1000 ms`).

---

## 4. Dynamic Link Speed Reconfiguration Sequence (10M / 100M / 1000M / 2500M)

**[HW]** Whenever the external QCA8081 PHY establishes a copper link (or changes negotiated link speed), all four hardware blocks (`NEMACxCTL`, `PMA`, `XPCS`, and `XGMAC2 MAC_TX_CONFIG`) must be synchronized to the negotiated speed:

| Negotiated Copper Speed | `NEMACxCTL` (`0x1070`/`0x1074`) Value | Re-run 5-Step PMA Init? | XPCS Mode (`VR_XS_PCS_DIG_CTRL1`, `VR_MII_AN_CTRL`, `SR_MII_CTRL`) | `MAC_TX_CONFIG` (`+0x0000`) `SS[31:29]` |
|---|---|---|---|---|
| **2500BASE-T** (`2500 Mbps`) | `0x00000114` (`SP_SEL = 4`) | **Yes** | `2500BASE-X`: `EN_2_5G_MODE = 1` (`0x0004`), `PCS_MODE = 0`, `SR_MII_CTRL = 0x0140` | `2` (`0x40000000`, 2.5G GMII) |
| **1000BASE-T** (`1000 Mbps`) | `0x00000115` (`SP_SEL = 5`) | **Yes** | `SGMII`: `EN_2_5G_MODE = 0`, `PCS_MODE = 2` (`0x0004`), `SR_MII_CTRL = 0x0140` | `3` (`0x60000000`, 1G GMII) |
| **100BASE-TX** (`100 Mbps`) | `0x00000116` (`SP_SEL = 6`) | **Yes** | `SGMII`: `EN_2_5G_MODE = 0`, `PCS_MODE = 2` (`0x0004`), `SR_MII_CTRL = 0x2100` (FD) / `0x2000` (HD) | `4` (`0x80000000`, 100M MII) |
| **10BASE-T** (`10 Mbps`) | `0x00000117` (`SP_SEL = 7`) | **Yes** | `SGMII`: `EN_2_5G_MODE = 0`, `PCS_MODE = 2` (`0x0004`), `SR_MII_CTRL = 0x0100` (FD) / `0x0000` (HD) | `7` (`0xE0000000`, 10M MII) |

**[HW]** Complete step-by-step link-up reconfiguration order:
1. Update `NEMAC0CTL` (`0x1070`) or `NEMAC1CTL` (`0x1074`) bits `[3:0]` (`SP_SEL`) to `4`, `5`, `6`, or `7` (keeping bit 8 `LPIHWCLKEN = 1` and bits `[5:4]` `PHY_INF_SEL = 1`).
2. Execute the **5-step PMA Initialization Sequence** (Section 3.2) and wait for `NEMACxCTL.INIT_DONE` (bit 21) == `1`.
3. Reconfigure the **XPCS** registers (Section 2.3) for 2500BASE-X (`2500 Mbps`) or SGMII (`1000 / 100 / 10 Mbps`).
4. Update `MAC_TX_CONFIG` (`XGMAC + 0x0000`) bits `[31:29]` (`SS`) to match the new speed while preserving `JD` (bit 16) and `TE` (bit 0).
5. Execute the **QCA8081 SerDes Companion (`0x1d`) FIFO Reset** (Section 5.4 below) so the PHY-to-MAC SerDes elastic buffer locks cleanly onto the new clock domain.

---

## 5. Qualcomm Atheros QCA8081 2.5G Copper PHY (`0x1c` and `0x1d`)

### 5.1 MDIO Addressing and PHY Identification

**[HW]** Each TC956x port has a dedicated external **Qualcomm Atheros QCA8081** single-port 2.5GBASE-T Ethernet PHY connected to that port's XGMAC2 MDIO bus:
- **Primary PHY MDIO Address**: `0x1c` (`28`). Responds to both **Clause 22** (registers `0x00..0x1F`) and **Clause 45** (`MMD 1` PMA/PMD, `MMD 3` PCS, `MMD 7` Auto-Negotiation).
- **Companion SerDes Responder MDIO Address**: `0x1d` (`29`, i.e., `phy_addr + 1`). Responds to **Clause 45 `MMD 1`** (`MDIO_MMD_PMAPMD = 1`) for the QCA8081 internal SerDes FIFO control register (`0x9072`).
- **32-Bit PHY Identifier**:
  - Clause 22 Register `0x02` (`MII_PHYSID1`): **`0x004D`**
  - Clause 22 Register `0x03` (`MII_PHYSID2`): **`0xD101`** (combined ID `0x004DD101`, mask `0xFFFFFFF0`).

### 5.2 QCA8081 Register Map (Clause 22 & Clause 45)

#### Clause 22 Standard and Vendor-Specific Registers (at MDIO Address `0x1c`)

| C22 Reg | Register Name | Bits | Field Name | Description / Observed Q8B Value |
|---|---|---|---|---|
| `0x00` | `BMCR` | `[15]` | `RESET` | PHY software reset (`0x8000`). |
| | | `[12]` | `ANENABLE` | Auto-Negotiation Enable (`0x1000`). |
| | | `[9]` | `ANRESTART` | Restart Auto-Negotiation (`0x0200`). Observed Q8B value: `0x1040`. |
| `0x01` | `BMSR` | `[5]` | `ANEGCOMPLETE` | Auto-Negotiation Complete (`0x0020`). |
| | | `[2]` | `LSTATUS` | Link Status (`0x0004`, latched low—read twice or read `PHY_SPEC_STATUS` `0x11`). Observed Q8B value: `0x796D`. |
| `0x04` | `ADVERTISE` | `[11:10]` | `PAUSE_CAP` / `ASYM` | 802.3x Pause capability (`0x0C00`). |
| | | `[8:5]` | `100FULL..10HALF` | 100BASE-TX / 10BASE-T advertisements (`0x01E1`). |
| `0x05` | `LPA` | `[15:0]` | — | Link Partner Ability (`0xC5E1` observed on Q8B). |
| `0x09` | `CTRL1000` | `[9:8]` | `1000FULL` / `HALF` | 1000BASE-T capability advertisement (`0x0200` = 1000BASE-T Full Duplex). |
| `0x0A` | `STAT1000` | `[11:10]` | `LPA_1000FULL/HALF` | Link Partner 1000BASE-T capability (`0x3800` observed on Q8B). |
| `0x0B` | `QCA808X_DBG_ADDR` | `[15:0]` | `DBG_ADDR` | Debug register indirect address port. |
| `0x0C` | `QCA808X_DBG_DATA` | `[15:0]` | `DBG_DATA` | Debug register indirect data port. |
| `0x10` | `PHY_SPEC_CONTROL` | `[6:5]` | `MDIX` | MDI / MDI-X crossover mode (`0` = manual MDI, `1` = manual MDI-X, `3` = Auto MDI-X). |
| `0x11` | `PHY_SPEC_STATUS` | `[15:14]` | `SPEED[1:0]` | Legacy speed bits (`0` = 10M, `1` = 100M, `2` = 1000M). |
| | | `[13]` | `DUPLEX` | Resolved Duplex (`0x2000`: **`1` = Full Duplex, `0` = Half Duplex**). |
| | | `[11]` | `SPEED_AND_DUPLEX_RESOLVED` | **`0x0800`: `1` = Speed and Duplex resolved**. |
| | | `[10]` | `LINK` | **`0x0400`: `1` = Real-time Copper Link Up, `0` = Link Down** (unlatched!). |
| | | `[9:7]` | `EXT_SPEED[2:0]` | **3-bit Resolved Speed Field** (`(val & 0x0380) >> 7`):<br>- `0` (`0b000`): **10 Mbps**<br>- `1` (`0b001`): **100 Mbps**<br>- `2` (`0b010`): **1000 Mbps**<br>- `4` (`0b100`): **2500 Mbps (2.5GBASE-T)**<br>Observed Q8B value at 1000BASE-T Full Duplex: `0xAD00` (`LINK = 1`, `RESOLVED = 1`, `DUPLEX = 1`, `EXT_SPEED = 2`). |

#### Clause 45 Registers (at MDIO Address `0x1c` and Companion SerDes Address `0x1d`)

| MDIO Addr | MMD (`devad`) | C45 Reg (`16-bit`) | Register Name | Bits & Description |
|---|---|---|---|---|
| `0x1c` | `1` (`PMAPMD`) | `0xA100` | `ADC_THRESHOLD` | `[7:0]` = `0xF0` (`ADC_THR_100MV` — set bits `[7:0]` to `0xF0` during PHY init). |
| `0x1c` | `1` (`PMAPMD`) | `0xA101` | `MSE_THRESHOLD_20DB` | `[15:0]` = `0x529` (20 dB Mean Square Error threshold for 2.5G link training). |
| `0x1c` | `1` (`PMAPMD`) | `0xA10C` | `MSE_THRESHOLD_17DB` | `[15:0]` = `0x341` (17 dB MSE threshold). |
| `0x1c` | `1` (`PMAPMD`) | `0xA10E` | `MSE_THRESHOLD_27DB` | `[15:0]` = `0x419` (27 dB MSE threshold). |
| `0x1c` | `1` (`PMAPMD`) | `0xA111` | `MSE_THRESHOLD_28DB` | `[15:0]` = `0x3A5` (28 dB MSE threshold). |
| `0x1c` | `3` (`PCS`) | `0x8062` | `CLD_CTRL3` | `[15:0]` mask `0x7F00` $\Rightarrow$ `0x2000` (`AN_EEE_THRESHOLD`). |
| `0x1c` | `3` (`PCS`) | `0x8063` | `CLD_CTRL5` | `[15:0]` mask `0x0F00` $\Rightarrow$ `0x0C00` (`PHASE_FIFO_THRESHOLD`). |
| `0x1c` | `3` (`PCS`) | `0x8065` | `CLD_CTRL7` | `[15:0]` mask `0x07F8` $\Rightarrow$ `0x0248` (`CABLE_LENGTH_TH`). |
| `0x1c` | `3` (`PCS`) | `0x8078` | `AZ_TRAINING_CTRL` | `[0]` `1G_SLAVE_F_PHASE_INVERT` = `1` (`0x0001`). |
| `0x1c` | `3` (`PCS`) | `0x901D` | `PCS_FIFO_CONTROL` | `[1:0]` `FIFO_RESET`: write `0x0` (assert FIFO reset) then `0x3` (deassert FIFO reset) if copper link drops or when resetting PCS FIFO. |
| `0x1c` | `7` (`AN`) | `0x0020` | `AN_MULTI_GBASE_T_CTRL` | **`[7]` `ADV_2500T` (`0x0080`: Advertise 2.5GBASE-T Full Duplex)**, `[5]` `FAST_RETRAIN_2500T` (`0x0020`), `[0]` `LOOP_TIMING` (`0x0001`). Observed Q8B value: `0x00A1`. |
| `0x1c` | `7` (`AN`) | `0x0021` | `AN_MULTI_GBASE_T_STATUS` | **`[13]` `LP_FAST_RETRAIN_2500T`**, **`[5]` `LP_2500T` (`0x0020`: Link Partner advertises 2.5GBASE-T)**. |
| `0x1c` | `7` (`AN`) | `0x003E` | `AN_EEE_LP_ABILITY2` | `[0]` `EEE_2500T` (`0x0001`). |
| `0x1c` | `7` (`AN`) | `0x801A` | `FAST_RETRAIN_CTRL` | `[0]` `THP_BYPASS` (`0x0001`: enable 2.5G fast retrain THP bypass). |
| `0x1c` | `7` (`AN`) | `0x807E` | `AN_TOP_CFG` | `[0]` `VGA_DECISION` (`0x0001`: enable VGA decision for 2.5G). |
| **`0x1d`** (`phy_addr + 1`) | **`1` (`PMAPMD`)** | **`0x9072`** | **`SERDES_MMD1_FIFO_CTRL`** | **`[11]` `PHY_FIFO_RSTN` (`0x0800`): QCA8081 SerDes Interface FIFO Reset (active-low: `0` = reset asserted, `1` = normal operation). Mandatory on link-up!** |

### 5.3 QCA8081 Initialization and Auto-Negotiation Configuration

**[HW]** To initialize a QCA8081 PHY at MDIO address `0x1c` (particularly on a cold port not initialized by UEFI):
1. **Apply Analog / DSP Thresholds and Fast-Retrain Calibration**:
   - In Clause 45 `MMD 7` (`0x1c`):
     - Set bit 0 (`0x0001`) in register `0x807E` (`AN_TOP_CFG`).
     - Set bit 0 (`0x0001`) in register `0x801A` (`FAST_RETRAIN_CTRL`).
     - Set bits `[5]` and `[0]` (`0x0021`) in register `0x0020` (`AN_MULTI_GBASE_T_CTRL`).
   - In Clause 45 `MMD 1` (`0x1c`):
     - Write `0x0529` to `0xA101`, `0x0341` to `0xA10C`, `0x0419` to `0xA10E`, and `0x03A5` to `0xA111`.
     - Modify `0xA100` (`ADC_THRESHOLD`): clear bits `[7:0]` and set `0x00F0`.
   - In Clause 45 `MMD 3` (`0x1c`):
     - Modify `0x8063`: clear bits `[11:8]` and set `0x0C00`.
     - Modify `0x8065`: clear bits `[10:3]` and set `0x0248`.
     - Modify `0x8062`: clear bits `[14:8]` and set `0x2000`.
     - Modify `0x8078`: set bit 0 (`0x0001`).
   - In Debug register `0x3100` (write `0x3100` to Clause 22 reg `0x0B`, then write `0x928E` to Clause 22 reg `0x0C`): configure 2.5G EEE LPI threshold.
2. **Advertise All Speeds (`10M / 100M / 1000M / 2500M`) and Pause**:
   - In Clause 22 `ADVERTISE` (`reg 0x04` at `0x1c`): set `0x0DE1` (`PAUSE_CAP | PAUSE_ASYM | 100FULL | 100HALF | 10FULL | 10HALF | CSMA`).
   - In Clause 22 `CTRL1000` (`reg 0x09` at `0x1c`): set `0x0200` (`1000FULL`).
   - In Clause 45 `MMD 7` register `0x0020` (`AN_MULTI_GBASE_T_CTRL` at `0x1c`): set bit 7 (`0x0080`, `ADV_2500T`) $\Rightarrow$ `0x00A1`.
   - In Clause 22 `BMCR` (`reg 0x00` at `0x1c`): write `0x1200` (`ANENABLE | ANRESTART`) to start auto-negotiation.

### 5.4 Mandatory QCA8081 Companion SerDes (`0x1d`) FIFO Reset on Every Link Change

**[HW]** The QCA8081 contains an internal SerDes block that responds on the same MDIO bus at address **`phy_addr + 1` (`0x1d` / 29)** using **Clause 45 `MMD 1` (`MDIO_MMD_PMAPMD = 1`)**.
**[HW]** When a port comes up cold (such as Port 1 `0004:03:00.1`, which UEFI never initializes, or either port after a speed change between 2500BASE-X and SGMII), the QCA8081 SerDes elastic FIFO between its copper DSP and its SerDes transmitter/receiver will overflow or underflow unless explicitly reset after the SerDes clocks stabilize. Without this reset, the copper link and XPCS link both report "up", TX descriptor counters increment, and zero packets are transmitted or received on the wire.

**[HW]** **Exact SerDes FIFO Reset Sequence (Run on Every Link State Change / Link-Up)**:
1. Perform a **Clause 45 Read** at MDIO address **`0x1d`** (`phy_addr + 1`), `devad = 1` (`MMD 1`), `reg = 0x9072` (`SERDES_MMD1_FIFO_CTRL`).
2. **Assert SerDes FIFO Reset**: Clear bit 11 (`~0x0800`, `PHY_FIFO_RSTN = 0`) in the value read and perform a **Clause 45 Write** back to MDIO address **`0x1d`**, `devad = 1`, `reg = 0x9072`.
3. Wait `1 ms` (`[POLICY]`: even back-to-back MDIO writes work, a `1 ms` delay ensures complete FIFO pointer clearing across clock domains).
4. **Deassert SerDes FIFO Reset**: Set bit 11 (`| 0x0800`, `PHY_FIFO_RSTN = 1`) and perform a **Clause 45 Write** to MDIO address **`0x1d`**, `devad = 1`, `reg = 0x9072`.
5. **[HW]** Additionally, if the copper link transitions to **Down**, reset the QCA8081 internal PCS FIFO at the primary PHY address **`0x1c`**, `devad = 3` (`MMD 3`), `reg = 0x901D`: clear bits `[1:0]` (`~0x0003`) via Clause 45 Write, then set bits `[1:0]` (`| 0x0003`) via Clause 45 Write.
