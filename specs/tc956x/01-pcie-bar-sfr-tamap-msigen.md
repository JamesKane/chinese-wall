# 01 — PCIe Topology, BAR Layout, SFR, TAMAP, GPIO, and MSIGEN

## 1. PCIe Bus Hierarchy and Device Identification

**[HW]** On the Radxa Dragon Q8B (Qualcomm SC8280XP), the Toshiba TC9563 package is attached to PCIe Controller 4 (`1c10000.pcie`, ACPI/PCI segment `0004`) via a PCIe Gen3 x2 physical link (8 GT/s per lane). Inside the TC9563 package, an integrated PCIe switch fans out to external downstream ports and an internal Gen3 x4 downstream port (Port 3) that connects to the dual-port Ethernet Endpoint (`0004:03:00.0` and `0004:03:00.1`).

**[HW]** The PCIe hierarchy and device identifiers are:

| BDF (Segment `0004`) | Role | Vendor ID | Device ID | Subvendor ID | Subdevice ID | Class Code | Link Width / Speed |
|---|---|---|---|---|---|---|---|
| `0004:00:00.0` | SC8280XP Root Port (`1c10000.pcie`) | `0x17CB` | `0x0110` | `0x17CB` | `0x0110` | `0x060400` (PCI Bridge) | Gen3 (8 GT/s) x2 |
| `0004:01:00.0` | TC9563 Switch Upstream Port | `0x1179` | `0x0623` | — | — | `0x060400` (PCI Bridge) | Gen3 (8 GT/s) x2 |
| `0004:02:01.0` | TC9563 Switch Downstream Port 1 (M.2 Key M) | `0x1179` | `0x0623` | — | — | `0x060400` (PCI Bridge) | Gen3 (8 GT/s) x2 |
| `0004:02:02.0` | TC9563 Switch Downstream Port 2 (Unused) | `0x1179` | `0x0623` | — | — | `0x060400` (PCI Bridge) | Gen3 (8 GT/s) x1 (Down) |
| `0004:02:03.0` | TC9563 Switch Downstream Port 3 (Internal GbE) | `0x1179` | `0x0623` | — | — | `0x060400` (PCI Bridge) | Gen3 (8 GT/s) x4 |
| `0004:03:00.0` | TC956x Ethernet Port 0 (Physical Function 0) | `0x1179` | `0x0220` | `0x1179` | `0x0001` | `0x020000` (Ethernet) | Gen3 (8 GT/s) x4 |
| `0004:03:00.1` | TC956x Ethernet Port 1 (Physical Function 1) | `0x1179` | `0x0220` | `0x1179` | `0x0001` | `0x020000` (Ethernet) | Gen3 (8 GT/s) x4 |

**[HW]** Each Ethernet Physical Function (`0004:03:00.0` and `0004:03:00.1`) exposes Revision ID `0x01` and the following PCI Configuration Space capabilities:
- Power Management (`0x40`): version 3, supports D0, D1, D2, D3hot, D3cold, PME from D0/D1/D2/D3hot/D3cold.
- MSI (`0x50`): 64-bit address capable, Multiple Message Capable up to 32 vectors (`MMC = 5`), per-vector masking not implemented in standard PCI MSI capability (per-vector masking is handled in the BAR4 MSIGEN block).
- MSI-X (`0x70`): 8 table entries located in BAR4 at offset `0x00008080`, Pending Bit Array (PBA) in BAR4 at offset `0x00008100`.
- PCI Express Endpoint (`0x90`): version 2, MaxPayload 512 bytes supported (configured to 128 bytes when parent switch uses 128B MPS), MaxReadReq 512 bytes, L0s and L1 ASPM supported (`L1 Substates` capability at `0x220`).

---

## 2. PCIe Base Address Registers (BARs) and Memory Map

**[HW]** Both Physical Function 0 (`0004:03:00.0`) and Physical Function 1 (`0004:03:00.1`) implement three 64-bit non-prefetchable memory BARs at BAR0 (PCI config offset `0x10–0x14`), BAR2 (`0x18–0x1C`), and BAR4 (`0x20–0x24`):

| BAR | Type | Size | Function 0 Q8B Physical Base | Function 1 Q8B Physical Base | Functional Purpose |
|---|---|---|---|---|---|
| `BAR0` | 64-bit, non-prefetchable | `16 KB` (`0x00004000`) | `0x9f281000` | `0x9f285000` | PCIe Bridge Configuration Space: AXI4 Slave Translation Address Map (`TAMAP` at `+0x0800`) and Resource Manager ID (`RSCMNG_ID` at `+0x2000`). |
| `BAR2` | 64-bit, non-prefetchable | `512 KB` (`0x00080000`) | `0x9f200000` | `0x9f000000` | On-chip Cortex-M3 SRAM (`0x00000–0x7FFFF`). Unused when the host CPU drives XGMAC DMA directly (`ENABLE_NC` mode). |
| `BAR4` | 64-bit, non-prefetchable | `2 MB` (`0x00200000`) | `0x9ec00000` | `0x9ee00000` | Special Function Registers (`SFR`), Interrupt Controller (`INTC`), MSI Generator (`MSIGEN`), XGMAC2 Core/MTL/DMA, XPCS, and SerDes PMA. |

### 2.1 BAR0 Internal Sub-Map (`16 KB`)

**[HW]** Each Physical Function has its own independent BAR0 window:

| BAR0 Offset | Register / Block Name | Access | Description |
|---|---|---|---|
| `0x0800–0x087F` | `AXI4_SLV0_TAMAP` | R/W | 4-entry Translation Address Map for AXI4 Slave 0 (`0x20` bytes per entry, `tid = 0..3`). Translates internal XGMAC DMA AXI4 transactions into upstream PCIe Memory Read/Write TLPs. Must be programmed in **each** function's BAR0 independently. |
| `0x2000` | `RSCMNG_ID_REG` | RO | Resource Manager ID register. Bits `[3:0]` (`RSCMNG_PF_ID`) return `0x0` on Physical Function 0 (`0004:03:00.0`) and `0x1` on Physical Function 1 (`0004:03:00.1`). |

### 2.2 BAR4 Internal Sub-Map (`2 MB`) and Cross-Function Ownership

**[HW]** BAR4 spans `0x000000` to `0x1FFFFF` (2 MB). Although both Physical Function 0 and Physical Function 1 expose a 2 MB BAR4 aperture, the chip-wide Special Function Registers (SFR: clocks, resets, eMAC mode control, and GPIOs) are decoded through **Function 0's BAR4** (`0004:03:00.0`). Meanwhile, each function's BAR4 decodes the per-port blocks at fixed port-specific offsets:

| BAR4 Offset | Size | Block Name | Owner / Access Rule | Description |
|---|---|---|---|---|
| `0x000000–0x000107` | `264 B` | Chip ID & System Status | Fn0 BAR4 | `NCID` (`0x0000`), `NMODESTS` (`0x0004`), `NFUNCEN0` (`0x0008`), `NPCIEBOOT` (`0x0018`), `NSYSDATA0/1` (`0x0100/0x0104`). |
| `0x001000–0x00107F` | `128 B` | Clock, Reset & eMAC Control | Fn0 BAR4 | `NCTLSTS` (`0x1000`), `NCLKCTRL0` (`0x1004`), `NRSTCTRL0` (`0x1008`), `NCLKCTRL1` (`0x100C`), `NRSTCTRL1` (`0x1010`), `NBUSCTRL` (`0x1014`), `NEMACTXCDLY` (`0x1050`), `NEMAC0CTL` (`0x1070`), `NEMAC1CTL` (`0x1074`), `NEMACSTS` (`0x1078`), `NEMACIOCTL` (`0x107C`). |
| `0x001200–0x001217` | `24 B` | GPIO Controller | Fn0 BAR4 | 37-pin GPIO block (`GPIOI0/1`, `GPIOE0/1`, `GPIOO0/1`). Controls external QCA8081 PHY reset lines (`GPIO0` = Port 0 PHY reset, `GPIO1` = Port 1 PHY reset). |
| `0x008000–0x0081FF` | `512 B` | Interrupt Controller (`INTC`) & MSI-X Table | Fn0 / Fn1 BAR4 | `INTMCUMASK0` (`0x8020`), `INTMCUMASK1` (`0x8024`), `INTMCUMASK2` (`0x8028`), MSI-X Table (`0x8080`), MSI-X PBA (`0x8100`). |
| `0x00F000–0x00F07F` | `128 B` | `MSIGEN0` (Port 0 MSI Generator) | Fn0 BAR4 | MSI interrupt multiplexer and status/mask registers for Port 0 (`0004:03:00.0`). |
| `0x00F100–0x00F17F` | `128 B` | `MSIGEN1` (Port 1 MSI Generator) | Fn1 BAR4 (or Fn0 BAR4) | MSI interrupt multiplexer and status/mask registers for Port 1 (`0004:03:00.1`). |
| `0x040000–0x0437FF` | `14 KB` | `XGMAC0` (Port 0 MAC / MTL / DMA) | Fn0 BAR4 | Synopsys DesignWare XGMAC2 v3.01a Core (`+0x0000`), MTL (`+0x1000`), and DMA (`+0x3000`) for Port 0. |
| `0x043A00–0x043DFF` | `1 KB` | `XPCS0` (Port 0 XPCS) | Fn0 BAR4 | Indirect viewport (`+0x3A00` from `XGMAC0`) for Port 0 DesignWare XPCS. |
| `0x044000–0x045FFF` | `8 KB` | `PMA0` (Port 0 SerDes PMA) | Fn0 BAR4 | Toshiba SerDes Physical Medium Attachment registers (`+0x4000` from `XGMAC0`) for Port 0. |
| `0x048000–0x04B7FF` | `14 KB` | `XGMAC1` (Port 1 MAC / MTL / DMA) | Fn1 BAR4 (or Fn0 BAR4) | Synopsys DesignWare XGMAC2 v3.01a Core (`+0x0000`), MTL (`+0x1000`), and DMA (`+0x3000`) for Port 1. |
| `0x04BA00–0x04BDFF` | `1 KB` | `XPCS1` (Port 1 XPCS) | Fn1 BAR4 (or Fn0 BAR4) | Indirect viewport (`+0x3A00` from `XGMAC1`) for Port 1 DesignWare XPCS. |
| `0x04C000–0x04DFFF` | `8 KB` | `PMA1` (Port 1 SerDes PMA) | Fn1 BAR4 (or Fn0 BAR4) | Toshiba SerDes Physical Medium Attachment registers (`+0x4000` from `XGMAC1`) for Port 1. |

**[HW]** **Fatal Unclocked / In-Reset MMIO Access Hazard**: Any PCIe Memory Read or Memory Write targeting `MSIGEN0/1` (`0xF000`/`0xF100`), `XGMAC0/1` (`0x40000`/`0x48000`), `XPCS0/1` (`0x43A00`/`0x4BA00`), or `PMA0/1` (`0x44000`/`0x4C000`) while that block's clock enable bit in `NCLKCTRL0`/`NCLKCTRL1` is `0` or its reset bit in `NRSTCTRL0`/`NRSTCTRL1` is `1` will hang on the internal AXI fabric, terminate with a PCIe Unsupported Request / Completer Abort, and raise an immediate fatal asynchronous SError exception on Qualcomm SC8280XP ARM64 CPUs. Clocks must always be enabled and resets deasserted via Function 0's BAR4 (`0x1004–0x1010`) before touching any downstream sub-block.

---

## 3. Special Function Registers (SFR) in BAR4

### 3.1 Chip Identification and Mode Status (`0x0000–0x0104`)

**[HW]** All offsets below are relative to BAR4 of Function 0:

| BAR4 Offset | Register Name | Bits | Field Name | Description / Observed Q8B Value |
|---|---|---|---|---|
| `0x0000` | `NCID` | `[7:0]` | `REV_ID` | Silicon revision ID (`0x01` on Radxa Dragon Q8B). |
| | | `[31:16]` | `CHIP_ID` | Chip ID (`0x0000` on TC9563; full register reads `0x00000001`). |
| `0x0004` | `NMODESTS` | `[8:0]` | `STRAP_PINS` | Bootstrap pin status (`0x00000201` on Radxa Dragon Q8B). |
| | | `[9]` | `MODE` | Operation mode bit (`1` on Q8B). |
| | | `[10]` | `MODE2` | PCIe switch lane bifurcation mode (`0` = Setting A: x4 upstream + x1 + x1 downstream; `1` = Setting B: x2 upstream + x2 + x1 downstream). |
| `0x0008` | `NFUNCEN0` | `[31:0]` | `FUNCEN0` | Pin / function enable strap status (`0x00000120` on Q8B). |
| `0x0018` | `NPCIEBOOT` | `[31:0]` | `PCIEBOOT` | PCIe boot status (`0x00000001` on Q8B). |
| `0x0100` | `NSYSDATA0` | `[31:0]` | `SYSDATA0` | System scratch data 0 (`0x00000000`). |
| `0x0104` | `NSYSDATA1` | `[31:0]` | `SYSDATA1` | System scratch data 1 (`0x00000000`). |

### 3.2 Clock Enable and Reset Control Registers (`0x1000–0x1014`)

**[HW]** In `NCLKCTRL0` (`0x1004`) and `NCLKCTRL1` (`0x100C`), writing `1` to a bit **enables** the corresponding clock and writing `0` **gates (disables)** it.
**[HW]** In `NRSTCTRL0` (`0x1008`) and `NRSTCTRL1` (`0x1010`), writing `1` to a bit **asserts** reset (holds the block in reset) and writing `0` **deasserts** reset (releases the block to run).

#### `NCLKCTRL0` (BAR4 `+0x1004`) — Chip-Wide and Port 0 Clock Control

**[HW]** Bit definitions for `NCLKCTRL0`:

| Bit | Field Name | Mask | Description | Host Driver Setting |
|---|---|---|---|---|
| `0` | `MCUCEN` | `0x00000001` | Embedded Cortex-M3 MCU clock enable | `0` (disabled when host drives DMA) |
| `4` | `INTCEN` | `0x00000010` | Legacy Interrupt Controller (`INTC`) clock enable | `1` while masking `INTMCUMASK0..2`, or `0` |
| `7` | `MAC0TXCEN` | `0x00000080` | Port 0 XGMAC TX clock enable | `1` (enabled when Port 0 active) |
| `9` | `PCIECEN` | `0x00000200` | PCIe Endpoint bridge clock enable | `1` (must remain enabled: `0x00000200`) |
| `13` | `SRMCEN` | `0x00002000` | Embedded Cortex-M3 SRAM clock enable | `0` (disabled when host drives DMA) |
| `14` | `MAC0RXCEN` | `0x00004000` | Port 0 XGMAC RX clock enable | `1` (enabled when Port 0 active) |
| `15` | `MAC0RMCEN` | `0x00008000` | Port 0 RMII clock enable | Preserved / `1` if RMII clock path used |
| `16` | `UARTOCEN` | `0x00010000` | On-chip UART0 clock enable | `0` (disabled) |
| `18` | `MSIGENCEN` | `0x00040000` | MSI Generator (`MSIGEN0` & `MSIGEN1`) clock enable | `1` (mandatory for MSI / MSI-X interrupts) |
| `24` | `POEPLLCEN` | `0x01000000` | PoE PLL clock enable | `0` (disabled) |
| `25` | `SGMPCIEN` | `0x02000000` | SGMII PCIe clock enable | `0` (disabled) |
| `26` | `REFCLKOCEN` | `0x04000000` | Reference clock output enable | `0` (disabled) |
| `29` | `MAC0125CLKEN` | `0x20000000` | Port 0 125 MHz clock enable | `0` (disabled; `MAC0ALLCLKEN` is used) |
| `30` | `MAC0312CLKEN` | `0x40000000` | Port 0 312.5 MHz clock enable | `0` (disabled; `MAC0ALLCLKEN` is used) |
| `31` | `MAC0ALLCLKEN` | `0x80000000` | Port 0 master clock enable | `1` (enabled when Port 0 active) |

**[HW]** On the Radxa Dragon Q8B, the live active value of `NCLKCTRL0` with Port 0 running is `0x80044280` (`MAC0ALLCLKEN | MSIGENCEN | MAC0RXCEN | PCIECEN | MAC0TXCEN`).
**[POLICY]** During chip initialization on Function 0, unused clocks (`MCUCEN`, `INTCEN`, `SRMCEN`, `UARTOCEN`, `POEPLLCEN`, `SGMPCIEN`, `REFCLKOCEN`, `MAC0125CLKEN`, `MAC0312CLKEN`, combined mask `0x67012011`) may be cleared to save power while enabling `MSIGENCEN` (`0x00040000`) and Port 0's `MAC0TXCEN | MAC0RXCEN | MAC0ALLCLKEN` (`0x80004080`).

#### `NRSTCTRL0` (BAR4 `+0x1008`) — Chip-Wide and Port 0 Reset Control

**[HW]** Bit definitions for `NRSTCTRL0` (`1` = reset asserted, `0` = reset deasserted):

| Bit | Field Name | Mask | Description | Host Driver Setting |
|---|---|---|---|---|
| `0` | `MCURST` | `0x00000001` | Embedded Cortex-M3 MCU reset | `1` (held in reset when unused) |
| `1` | `MCU1RST` | `0x00000002` | Embedded MCU1 reset | `1` (held in reset when unused) |
| `4` | `INTRST` | `0x00000010` | Interrupt Controller (`INTC`) reset | `0` if `INTC` accessed, `1` if gated |
| `7` | `MAC0RST` | `0x00000080` | Port 0 XGMAC2 core & DMA reset | `0` (deasserted when Port 0 active) |
| `9` | `PCIERST` | `0x00000200` | PCIe Endpoint reset | `0` (never assert from host!) |
| `16` | `UART0RST` | `0x00010000` | UART0 reset | `1` (held in reset when unused) |
| `18` | `MSIGENRST` | `0x00040000` | MSI Generator (`MSIGEN0` & `MSIGEN1`) reset | `0` (deasserted for MSI operation) |
| `30` | `MAC0PMARST` | `0x40000000` | Port 0 SerDes PMA reset | Pulsed `1` -> `0` during PMA init |
| `31` | `MAC0PONRST` | `0x80000000` | Port 0 XPCS power-on reset | `0` (deasserted before XPCS/PMA init) |

**[HW]** On the Radxa Dragon Q8B, the live active value of `NRSTCTRL0` with Port 0 running is `0x00010013` (`UART0RST | INTRST | MCU1RST | MCURST` asserted; `MAC0PONRST`, `MAC0PMARST`, `MSIGENRST`, `PCIERST`, and `MAC0RST` all `0` / deasserted).

#### `NCLKCTRL1` (BAR4 `+0x100C`) — Port 1 Clock Control

**[HW]** Bit definitions for `NCLKCTRL1`:

| Bit | Field Name | Mask | Description | Host Driver Setting |
|---|---|---|---|---|
| `7` | `MAC1TXCEN` | `0x00000080` | Port 1 XGMAC TX clock enable | `1` (enabled when Port 1 active) |
| `14` | `MAC1RXCEN` | `0x00004000` | Port 1 XGMAC RX clock enable | `1` (enabled when Port 1 active) |
| `15` | `MAC1RMCEN` | `0x00008000` | Port 1 RMII / internal reference clock enable | `1` (enabled when Port 1 active) |
| `29` | `MAC1125CLKEN1` | `0x20000000` | Port 1 125 MHz clock enable | `0` (disabled; `MAC1ALLCLKEN1` used) |
| `30` | `MAC1312CLKEN1` | `0x40000000` | Port 1 312.5 MHz clock enable | `0` (disabled; `MAC1ALLCLKEN1` used) |
| `31` | `MAC1ALLCLKEN1` | `0x80000000` | Port 1 master clock enable | `1` (enabled when Port 1 active) |

**[HW]** On the Radxa Dragon Q8B, the active value of `NCLKCTRL1` with Port 1 enabled is `0x8000C080` (`MAC1ALLCLKEN1 | MAC1RMCEN | MAC1RXCEN | MAC1TXCEN`).

#### `NRSTCTRL1` (BAR4 `+0x1010`) — Port 1 Reset Control

**[HW]** Bit definitions for `NRSTCTRL1` (`1` = reset asserted, `0` = reset deasserted):

| Bit | Field Name | Mask | Description | Host Driver Setting |
|---|---|---|---|---|
| `7` | `MAC1RST1` | `0x00000080` | Port 1 XGMAC2 core & DMA reset | `0` (deasserted when Port 1 active) |
| `30` | `MAC1PMARST1` | `0x40000000` | Port 1 SerDes PMA reset | Pulsed `1` -> `0` during PMA init |
| `31` | `MAC1PONRST1` | `0x80000000` | Port 1 XPCS power-on reset | `0` (deasserted before XPCS/PMA init) |

**[HW]** When both Port 1 XPCS, PMA, and MAC are out of reset, `NRSTCTRL1` reads `0x00000000`.

### 3.3 Bus and RGMII Control Registers (`0x1014–0x1050`)

**[HW]** Additional bus and delay registers in BAR4:
- `NBUSCTRL` (`+0x1014`): Internal bus control register (power-on/default value `0x00010000`).
- `NEMACTXCDLY` (`+0x1050`): RGMII TX clock delay control (`0x00000000`; unused in SerDes SGMII/2500BASE-X mode).

### 3.4 eMAC Mode and Speed Control: `NEMAC0CTL` (`0x1070`) and `NEMAC1CTL` (`0x1074`)

**[HW]** `NEMAC0CTL` (`+0x1070`, for Port 0) and `NEMAC1CTL` (`+0x1074`, for Port 1) configure the SerDes/PCS interface speed, clock source, signal detect polarity, and LPI clock gating, and report PMA initialization completion:

| Bits | Field Name | Mask | Access | Description |
|---|---|---|---|---|
| `[3:0]` | `SP_SEL` | `0x0000000F` | R/W | Interface and speed selection (see encoding table below). **Resets to `8` (USXGMII 10G/10G)!** |
| `[5:4]` | `PHY_INF_SEL` | `0x00000030` | R/W | PCS clock source selection (`0` = internal, `1` = clock from PHY, i.e. value `0x10`). |
| `[6]` | `INV_SGM_SIG_DET` | `0x00000040` | R/W | SGMII signal detect polarity (`0` = normal, `1` = inverted). Set to `0` on Q8B. |
| `[8]` | `LPIHWCLKEN` | `0x00000100` | R/W | LPI hardware clock gating enable (`0` = disabled, `1` = enabled). |
| `[21]` | `INIT_DONE` | `0x00200000` | RO | PMA and eMAC wrapper initialization complete (`1` = ready, `0` = initializing). |

**[HW]** Complete `SP_SEL` (bits `[3:0]`) encoding table for `NEMAC0CTL` and `NEMAC1CTL`:

| `SP_SEL` (Decimal) | `SP_SEL` (Hex) | Interface Mode | Line / Data Rate | Used on Radxa Dragon Q8B? |
|---|---|---|---|---|
| `0` | `0x0` | RGMII | 1000 Mbps | No |
| `1` | `0x1` | RGMII | 100 Mbps | No |
| `2` | `0x2` | RGMII | 10 Mbps | No |
| `4` | `0x4` | 2500BASE-X / SGMII 2.5G | 2500 Mbps (3.125 Gbaud SerDes) | **Yes — 2500BASE-T link** |
| `5` | `0x5` | SGMII | 1000 Mbps (1.25 Gbaud SerDes) | **Yes — 1000BASE-T link** |
| `6` | `0x6` | SGMII | 100 Mbps (1.25 Gbaud SerDes) | **Yes — 100BASE-TX link** |
| `7` | `0x7` | SGMII | 10 Mbps (1.25 Gbaud SerDes) | **Yes — 10BASE-T link** |
| `8` | `0x8` | USXGMII | 10G rate / 10G line (**Power-on Default**) | No (Invalid for QCA8081!) |
| `9` | `0x9` | USXGMII | 5G rate / 10G line | No |
| `10` | `0xA` | USXGMII | 5G rate / 5G line | No |
| `11` | `0xB` | USXGMII | 2.5G rate / 10G line | No |
| `12` | `0xC` | USXGMII | 2.5G rate / 5G line | No |
| `13` | `0xD` | USXGMII | 2.5G rate / 2.5G line | No |

**[HW]** **Power-On Reset `SP_SEL` Trap**: At chip reset, `NEMAC0CTL` and `NEMAC1CTL` default to `SP_SEL = 8` (`0x8`, USXGMII 10G). On a cold port (such as Port 1 when not initialized by UEFI, or either port after a chip reset), leaving `SP_SEL = 8` prevents the SerDes PMA from locking at 1.25 Gbaud (SGMII) or 3.125 Gbaud (2500BASE-X). The driver must write `SP_SEL = 4` (for 2.5G) or `SP_SEL = 5` (for 1G) into `NEMACxCTL` **before** executing the 5-step PMA reset and initialization sequence. On the Radxa Dragon Q8B at 1000BASE-T, `NEMAC0CTL` reads `0x00200115` (`INIT_DONE = 1`, `LPIHWCLKEN = 1`, `PHY_INF_SEL = 1`, `SP_SEL = 5`).

**[HW]** Companion status and I/O control registers:
- `NEMACSTS` (`+0x1078`): eMAC status register (`0x00001800` on Q8B).
- `NEMACIOCTL` (`+0x107C`): eMAC I/O pad control (`0xF300F300` on Q8B).

---

## 4. On-Chip GPIO Controller (`0x1200–0x1214`) and External PHY Reset

**[HW]** The TC956x provides 37 GPIO pins (pins `0..31` in Bank 0 and pins `32..36` in Bank 1) controlled through Function 0's BAR4 at offsets `0x1200–0x1214`:

| BAR4 Offset | Register Name | Bank / Pins | Access | Polarity / Semantics |
|---|---|---|---|---|
| `0x1200` | `GPIOI0` (`GLV0`) | Bank 0 (`0..31`) | RO | Input pin logic level (`1` = high, `0` = low). |
| `0x1204` | `GPIOI1` (`GLV1`) | Bank 1 (`32..36`) | RO | Input pin logic level for bits `[4:0]` (`1` = high, `0` = low). |
| `0x1208` | `GPIOE0` (`GDIR0`) | Bank 0 (`0..31`) | R/W | Direction / Output-Disable control: **`0` = Output enabled, `1` = Input (high-Z)**. |
| `0x120C` | `GPIOE1` (`GDIR1`) | Bank 1 (`32..36`) | R/W | Direction / Output-Disable control for bits `[4:0]`: **`0` = Output enabled, `1` = Input**. |
| `0x1210` | `GPIOO0` (`GOUT0`) | Bank 0 (`0..31`) | R/W | Output drive level when `GPIOE0` bit is `0` (`1` = drive high, `0` = drive low). |
| `0x1214` | `GPIOO1` (`GOUT1`) | Bank 1 (`32..36`) | R/W | Output drive level for bits `[4:0]` when `GPIOE1` bit is `0`. |

**[HW]** **Inverted Direction Bit Convention**: Unlike controllers where `1` means output, in `GPIOE0` (`0x1208`) and `GPIOE1` (`0x120C`), clearing a bit to `0` configures the pin as an **output**, and setting a bit to `1` configures the pin as an **input**. On the Radxa Dragon Q8B, `GPIOE0` reads `0xFFFFFFF0` and `GPIOO0` reads `0x0000000F` (pins `0..3` configured as outputs driven high).

**[HW]** **Board-Level PHY Reset Pin Assignment (Radxa Dragon Q8B)**:
- `GPIO0` (bit 0 of `GPIOE0` / `GPIOO0`): Port 0 QCA8081 hardware reset (`PHY0_RST_N`, active-low).
- `GPIO1` (bit 1 of `GPIOE0` / `GPIOO0`): Port 1 QCA8081 hardware reset (`PHY1_RST_N`, active-low).

**[HW]** **External PHY Hardware Reset Sequence**:
1. Clear bit `p` (`0` for Port 0, `1` for Port 1) in `GPIOE0` (`BAR4 + 0x1208`) so the pin is configured as an output.
2. To assert PHY reset (if performing a cold reset), clear bit `p` in `GPIOO0` (`BAR4 + 0x1210`) to `0` and wait at least `10 ms` (`[POLICY]`: Toshiba vendor driver uses `10 ms` assert / `50 ms` deassert; QCA8081 hardware requires at least `10 ms` reset assertion and `50 ms` post-reset stabilization before MDIO access).
3. To deassert PHY reset, set bit `p` in `GPIOO0` (`BAR4 + 0x1210`) to `1` and wait at least `50 ms` before issuing MDIO transactions to `0x1c`.

---

## 5. BAR0 Translation Address Map (TAMAP) and DMA Address Offset

**[HW]** The two XGMAC2 DMA engines (`XGMAC0` and `XGMAC1`) are masters on the TC956x internal 64-bit AXI4 interconnect. In the internal AXI4 address map, low physical addresses (`0x0000_0000_0000_0000` upward) decode to internal Cortex-M3 SRAM, SFRs, and peripheral blocks. To reach host DRAM across the PCIe link, the XGMAC2 DMA engine must issue AXI4 read/write transactions into an AXI4 Slave 0 address window that is translated by the PCIe Bridge's **Translation Address Map (TAMAP)** at `BAR0 + 0x0800`.

### 5.1 TAMAP Register Layout (`BAR0 + 0x0800`)

**[HW]** Each Physical Function has 4 TAMAP table entries (`tid = 0, 1, 2, 3`) in its own BAR0 at base offset `0x0800 + (tid * 0x20)`:

| Entry Offset | Register Name | Bits | Field Name | Description |
|---|---|---|---|---|
| `+0x00` | `TAMAP_SRC_ADDR_LO` | `[0]` | `ATR_IMPL` | Entry enable (`1` = translation window active, `0` = disabled). |
| | | `[6:1]` | `ATR_SIZE` (`OW`) | Window size exponent minus 1: window covers $2^{\text{ATR\_SIZE} + 1}$ bytes. Minimum valid value is `11` ($2^{12} = 4\text{ KB}$); maximum for 64 GB ($2^{36}$ bytes) is `35` (`0x23`). |
| | | `[11:7]` | Reserved | Write `0`. |
| | | `[31:12]` | `SRC_ADDR_LO` | Low 20 bits `[31:12]` of the 64-bit internal AXI4 source base address (must be aligned to the window size). |
| `+0x04` | `TAMAP_SRC_ADDR_HI` | `[31:0]` | `SRC_ADDR_HI` | Upper 32 bits `[63:32]` of the 64-bit internal AXI4 source base address. |
| `+0x08` | `TAMAP_TRSL_ADDR_LO` | `[11:0]` | Reserved | Write `0`. |
| | | `[31:12]` | `TRSL_ADDR_LO` | Low 20 bits `[31:12]` of the translated 64-bit upstream PCIe target base address. |
| `+0x0C` | `TAMAP_TRSL_ADDR_HI` | `[31:0]` | `TRSL_ADDR_HI` | Upper 32 bits `[63:32]` of the translated 64-bit upstream PCIe target base address. |
| `+0x10` | `TAMAP_TRSL_PARAM` | `[3:0]` | `TRSL_ID` | Target interface ID (`0x0` = PCIe TX/RX memory transaction). |
| | | `[27:16]` | `TRSL_PARAM` | Transaction attribute parameters (`0x000`). |
| `+0x18` | `TAMAP_TRSL_MASK1` | `[31:0]` | `TRSL_MASK1` | Translation mask low (read-only diagnostic: `0xFFFFF000` when `ATR_SIZE = 35`). |
| `+0x1C` | `TAMAP_TRSL_MASK2` | `[31:0]` | `TRSL_MASK2` | Translation mask high (read-only diagnostic: `0x0000000F` when `ATR_SIZE = 35`). |

### 5.2 Programming TAMAP Entry 0 and Disabling Entries 1–3

**[HW]** During initialization of **each** Physical Function (`0004:03:00.0` and `0004:03:00.1`), the driver must program its own `BAR0 + 0x0800` table as follows:

1. **Program Entry 0 (`BAR0 + 0x0800`)** to map the 64 GB ($2^{36}$-byte) AXI4 window `[0x0000_0010_0000_0000 .. 0x0000_001F_FFFF_FFFF]` to upstream PCIe bus addresses `[0x0000_0000_0000_0000 .. 0x0000_000F_FFFF_FFFF]`:
   - Write `0x00000047` to `BAR0 + 0x0800` (`TAMAP_SRC_ADDR_LO`: `SRC_ADDR_LO[31:12] = 0`, `ATR_SIZE[6:1] = 35` (`35 << 1 = 0x46`), `ATR_IMPL[0] = 1`).
   - Write `0x00000010` to `BAR0 + 0x0804` (`TAMAP_SRC_ADDR_HI`).
   - Write `0x00000000` to `BAR0 + 0x0808` (`TAMAP_TRSL_ADDR_LO`).
   - Write `0x00000000` to `BAR0 + 0x080C` (`TAMAP_TRSL_ADDR_HI`).
   - Write `0x00000000` to `BAR0 + 0x0810` (`TAMAP_TRSL_PARAM`).
2. **Disable Entries 1, 2, and 3 (`tid = 1..3`, offsets `BAR0 + 0x0820`, `0x0840`, `0x0860`)**:
   - For each `tid` in `1..3`, write `0x00000000` to `+0x00` (`SRC_ADDR_LO`), `+0x04` (`SRC_ADDR_HI`), `+0x08` (`TRSL_ADDR_LO`), `+0x0C` (`TRSL_ADDR_HI`), and `+0x10` (`TRSL_PARAM`).

### 5.3 Mandatory DMA Address Translation Formula (`+0x10_0000_0000`)

**[HW]** Because TAMAP Entry 0 covers a 36-bit ($64\text{ GB}$) aperture starting at AXI address `0x0000_0010_0000_0000`, the host OS must restrict DMA allocations (or IOMMU IOVAs) for the TC956x to a **36-bit DMA mask** (`0x0000_000F_FFFF_FFFF`), and **every** host physical/IOVA address $A_{\text{PCIe}} \in [0, 2^{36}-1]$ given to the XGMAC2 hardware must be converted to an internal AXI4 address $A_{\text{AXI}}$ by adding `0x0000_0010_0000_0000`:

$$A_{\text{AXI}} = (A_{\text{PCIe}} \mathbin{\&} \text{0x0000\_000F\_FFFF\_FFFF}) \mathbin{|} \text{0x0000\_0010\_0000\_0000}$$

Equivalently, in 32-bit high/low register pairs:
- $\text{Low 32 bits} = A_{\text{PCIe}}[31:0]$
- $\text{High 32 bits} = \text{0x00000010} \mathbin{|} (A_{\text{PCIe}}[35:32])$

**[HW]** This `+0x10_0000_0000` translation applies to:
1. TX and RX descriptor ring base high/low registers (`DMA_CH_TxDESC_HADDR` / `LADDR` at `+0x10` / `+0x14`, `DMA_CH_RxDESC_HADDR` / `LADDR` at `+0x18` / `+0x1C`).
2. TX and RX descriptor buffer pointers (`TDES0`/`TDES1` and `RDES0`/`RDES1`, plus split-header `RDES2`/`RDES3` if used).
3. **4 GB Boundary Constraint on Descriptor Rings**: Because `DMA_CH_TxDESC_TAIL_LPTR` (`+0x24`) and `DMA_CH_RxDESC_TAIL_LPTR` (`+0x2C`) are 32-bit registers that only accept the low 32 bits of the tail descriptor address while the DMA hardware reuses the upper 32 bits from `TxDESC_HADDR` / `RxDESC_HADDR`, a descriptor ring must **never** cross a 4 GB ($2^{32}$-byte) physical/IOVA boundary. Aligning each descriptor ring to its size (e.g., 4 KB or 16 KB alignment) guarantees it never crosses a 4 GB boundary.

---

## 6. Interrupt Controller (`INTC`) and MSI Generator (`MSIGEN`)

### 6.1 Masking the Embedded Cortex-M3 Interrupts (`BAR4 + 0x8000`)

**[HW]** The TC956x contains a legacy Interrupt Controller (`INTC` at `BAR4 + 0x8000`) that routes internal interrupts to the embedded Cortex-M3 MCU. When the host CPU drives the Ethernet ports directly, the MCU interrupt masks should be set so internal MAC events do not trigger the unused MCU path:
- `INTMCUMASK0` (`BAR4 + 0x8020`): OR with `0xFFFF1FFF` (or write `0xFFFFFFFF`).
- `INTMCUMASK1` (`BAR4 + 0x8024`): OR with `0xFFFF1F80` (or write `0xFFFFFFFF`).
- `INTMCUMASK2` (`BAR4 + 0x8028`): write `0xFFFFFFFF`.

**[HW]** Note: Only access `0x8020–0x8028` if `NCLKCTRL0` bit 4 (`INTCEN`, `0x10`) is `1` and `NRSTCTRL0` bit 4 (`INTRST`, `0x10`) is `0`. If the driver gates `INTCEN = 0` and asserts `INTRST = 1`, the `INTC` block is powered down and must not be read or written.

### 6.2 MSIGEN Register Map (`0xF000` for Function 0, `0xF100` for Function 1)

**[HW]** Each Physical Function has a dedicated MSI Generator (`MSIGEN`) block in BAR4:
- **Function 0 (`0004:03:00.0`)**: `MSIGEN0` at `BAR4 + 0xF000`
- **Function 1 (`0004:03:00.1`)**: `MSIGEN1` at `BAR4 + 0xF100`

**[HW]** Both `MSIGEN0` and `MSIGEN1` are clocked by the shared `MSIGENCEN` bit (`NCLKCTRL0` bit 18 = `1`) and reset by the shared `MSIGENRST` bit (`NRSTCTRL0` bit 18 = `0`) in Function 0's BAR4.

**[HW]** Register offsets relative to the port's MSIGEN base (`0xF000` or `0xF100`):

| Offset | Register Name | Access | Description / Observed Q8B Value |
|---|---|---|---|
| `+0x00` | `MSI_OUT_EN` | R/W | Per-source interrupt output enable mask (`1` = source enabled to trigger MSI, `0` = source masked). Observed Q8B value: `0x0017FFF8` (`PHY`, `TX0..TX7`, `RX0..RX7` enabled). |
| `+0x08` | `MSI_MASK_SET` | R/W (W1S) | Per-MSI-vector mask set register (`31:0`). Writing `1` to bit $v$ masks MSI vector $v$. In single-MSI mode (vector 0 only), write `0xFFFFFFFE` at init to mask vectors `1..31`, and write `0x00000001` to mask vector 0 during ISR entry if desired. |
| `+0x0C` | `MSI_MASK_CLR` | R/W (W1C) | Per-MSI-vector mask clear / End-Of-Interrupt (EOI) register (`31:0`). Writing `1` to bit $v$ (`0x00000001` for vector 0) clears the mask on vector $v$ and immediately re-triggers an upstream PCIe MSI TLP if any enabled bit in `MSI_INT_STS` is still `1`. |
| `+0x10` | `MSI_INT_STS` | RO | Active raw interrupt status bitmask across all internal sources (see bit table below). Driven directly by the level-high interrupt outputs of XGMAC2, XPCS, and PHY; a bit clears only when the underlying peripheral interrupt status register is cleared. |
| `+0x20` | `MSI_VECT_SET0` | R/W | Vector assignment for sources `0..5` (5 bits per source; `0x00000000` maps all to vector 0). |
| `+0x24` | `MSI_VECT_SET1` | R/W | Vector assignment for sources `6..11` (`0x00000000` maps all to vector 0). |
| `+0x28` | `MSI_VECT_SET2` | R/W | Vector assignment for sources `12..17` (`0x00000000` maps all to vector 0). |
| `+0x2C` | `MSI_VECT_SET3` | R/W | Vector assignment for sources `18..23` (`0x00000000` maps all to vector 0). |
| `+0x30` | `MSI_VECT_SET4` | R/W | Vector assignment for sources `24..25` (`0x00000000` maps all to vector 0). |
| `+0x34` | `MSI_VECT_SET5` | R/W | Reserved vector assignment 5 (`0x00000000`). |
| `+0x38` | `MSI_VECT_SET6` | R/W | Reserved vector assignment 6 (`0x00000000`). |
| `+0x3C` | `MSI_VECT_SET7` | R/W | Reserved vector assignment 7 (`0x00000000`). |
| `+0x68` | `MSI_EVENT` | RO | Pending MSI vector event bitmap (`bit v = 1` when vector $v$ has an active event). |

### 6.3 MSIGEN Source Bit Mapping (`MSI_OUT_EN` `+0x00` and `MSI_INT_STS` `+0x10`)

**[HW]** Both `MSI_OUT_EN` (`+0x00`) and `MSI_INT_STS` (`+0x10`) share the identical bit layout:

| Bit(s) | Source Name | Mask | Underlying Hardware Source |
|---|---|---|---|
| `0` | `LPI` | `0x00000001` | XGMAC2 Energy Efficient Ethernet (EEE) Low-Power Idle interrupt |
| `1` | `PMT` | `0x00000002` | XGMAC2 Power Management (Magic Packet / Remote Wake-Up) interrupt |
| `2` | `EVENT` | `0x00000004` | XGMAC2 MAC Core / MTL / Timestamp / LPI combined event interrupt |
| `3` | `TX0` | `0x00000008` | XGMAC2 TX DMA Channel 0 interrupt (`DMA_CH0_STATUS`) |
| `4` | `TX1` | `0x00000010` | XGMAC2 TX DMA Channel 1 interrupt (`DMA_CH1_STATUS`) |
| `5` | `TX2` | `0x00000020` | XGMAC2 TX DMA Channel 2 interrupt (`DMA_CH2_STATUS`) |
| `6` | `TX3` | `0x00000040` | XGMAC2 TX DMA Channel 3 interrupt (`DMA_CH3_STATUS`) |
| `7` | `TX4` | `0x00000080` | XGMAC2 TX DMA Channel 4 interrupt (`DMA_CH4_STATUS`) |
| `8` | `TX5` | `0x00000100` | XGMAC2 TX DMA Channel 5 interrupt (`DMA_CH5_STATUS`) |
| `9` | `TX6` | `0x00000200` | XGMAC2 TX DMA Channel 6 interrupt (`DMA_CH6_STATUS`) |
| `10` | `TX7` | `0x00000400` | XGMAC2 TX DMA Channel 7 interrupt (`DMA_CH7_STATUS`) |
| `11` | `RX0` | `0x00000800` | XGMAC2 RX DMA Channel 0 interrupt (`DMA_CH0_STATUS`) |
| `12` | `RX1` | `0x00001000` | XGMAC2 RX DMA Channel 1 interrupt (`DMA_CH1_STATUS`) |
| `13` | `RX2` | `0x00002000` | XGMAC2 RX DMA Channel 2 interrupt (`DMA_CH2_STATUS`) |
| `14` | `RX3` | `0x00004000` | XGMAC2 RX DMA Channel 3 interrupt (`DMA_CH3_STATUS`) |
| `15` | `RX4` | `0x00008000` | XGMAC2 RX DMA Channel 4 interrupt (`DMA_CH4_STATUS`) |
| `16` | `RX5` | `0x00010000` | XGMAC2 RX DMA Channel 5 interrupt (`DMA_CH5_STATUS`) |
| `17` | `RX6` | `0x00020000` | XGMAC2 RX DMA Channel 6 interrupt (`DMA_CH6_STATUS`) |
| `18` | `RX7` | `0x00040000` | XGMAC2 RX DMA Channel 7 interrupt (`DMA_CH7_STATUS`) |
| `19` | `XPCS` | `0x00080000` | Synopsys DesignWare XPCS interrupt |
| `20` | `PHY` | `0x00100000` | External QCA8081 PHY interrupt pin |
| `21` | `PFMAILBOX` | `0x00200000` | Inter-PF Mailbox interrupt |
| `24` | `MSIREQ_PLS` | `0x01000000` | MSI pulse request |

### 6.4 Single-Vector MSI Initialization and Interrupt Service Flow

**[HW]** On platforms (such as SC8280XP under OSes without multi-MSI GICv3 ITS allocation or when using 1 MSI vector per PCI function) where 1 MSI vector (vector 0) is allocated per Physical Function:

1. **Initialization**:
   - Ensure `NCLKCTRL0` bit 18 (`MSIGENCEN`) is `1` and `NRSTCTRL0` bit 18 (`MSIGENRST`) is `0` in Function 0's BAR4.
   - Route all sources to MSI vector 0 by writing `0x00000000` to `MSI_VECT_SET0` through `MSI_VECT_SET7` (`MSIGEN + 0x20` through `+0x3C`).
   - Mask unused MSI vectors `1..31` by writing `0xFFFFFFFE` to `MSI_MASK_SET` (`MSIGEN + 0x08`).
   - Unmask MSI vector 0 by writing `0x00000001` to `MSI_MASK_CLR` (`MSIGEN + 0x0C`).
   - Enable desired sources in `MSI_OUT_EN` (`MSIGEN + 0x00`):
     - For a single TX/RX queue driver (Channel 0 only): write `0x00000808` (`TX0 | RX0`, plus `EVENT` `0x4` or `PHY` `0x100000` if desired).
     - For all 8 TX/RX channels + PHY: write `0x0017FFF8`.
   - Program `DMA_MODE` (`XGMAC + 0x3000`) bits `[13:12]` (`INTM`) to `1` (`0x1000`).

2. **Interrupt Service Routine (Level-to-Edge Re-Arm Protocol)**:
   - **[HW]** Read `MSI_INT_STS` (`MSIGEN + 0x10`) and mask with the enabled bits in `MSI_OUT_EN`. If the result is `0`, return (spurious or already serviced).
   - **[HW]** Mask MSI vector 0 during servicing by writing `0x00000001` to `MSI_MASK_SET` (`MSIGEN + 0x08`).
   - **[HW]** For each asserted `TXn` (`bits 3..10`) or `RXn` (`bits 11..18`) source, read `DMA_CHn_STATUS` (`XGMAC + 0x3160 + n * 0x80`) and write back the active status bits (`TI`, `RI`, `TBU`, `RBU`, `NIS`, `AIS`, `FBE`) to clear the level-high DMA interrupt line.
   - **[HW]** If `EVENT` (bit 2) or `LPI` (bit 0) is asserted, read `MAC_INTERRUPT_STATUS` (`XGMAC + 0x00B0`), `MAC_LPI_CTRL_STATUS` (`+0x00D0`, clears on read), and `MTL_Q0_INT_CTRL_STATUS` (`+0x1174`) to clear the underlying MAC/MTL source.
   - **[HW]** Unmask and re-arm MSI vector 0 by writing `0x00000001` to `MSI_MASK_CLR` (`MSIGEN + 0x0C`). Because internal sources are level-high and PCIe MSI is edge-triggered, pulsing `MSI_MASK_SET(1)` followed by `MSI_MASK_CLR(1)` after clearing the peripheral status guarantees that if a new packet arrived while the ISR was running (leaving `MSI_INT_STS != 0`), `MSIGEN` immediately emits a fresh MSI edge.
