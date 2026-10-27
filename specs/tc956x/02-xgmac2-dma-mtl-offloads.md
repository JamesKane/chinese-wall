# 02 — XGMAC2 Core, MTL, DMA Engine, Descriptors, and Offloads

## 1. XGMAC2 Base Offsets, Version, and Hardware Capabilities

**[HW]** Each TC956x Ethernet port contains an independent Synopsys DesignWare XGMAC2 v3.01a controller mapped inside BAR4:
- **Port 0 (`0004:03:00.0`)**: `BAR4 + 0x040000`
- **Port 1 (`0004:03:00.1`)**: `BAR4 + 0x048000`

**[HW]** All register offsets in this document are relative to the port's XGMAC base (`0x40000` or `0x48000`).

### 1.1 Version and Synthesized Feature Registers (`+0x0110` to `+0x0128`)

**[HW]** Reading the version and hardware feature registers on the Radxa Dragon Q8B returns:

| Offset | Register Name | Observed Q8B Value | Decoded Hardware Capabilities |
|---|---|---|---|
| `+0x0110` | `MAC_VERSION` | `0x00107630` | `SNPSVER[7:0] = 0x30` (DesignWare XGMAC2 v3.01a), `USERVER[15:8] = 0x76` (Toshiba user revision). |
| `+0x011C` | `MAC_HW_FEATURE0` | `0xCA7D79F3` | `GMIISEL` (`bit 1 = 1`: 1G GMII), `VLHASH` (`bit 4 = 1`: VLAN hash filter), `SMASEL` (`bit 5 = 1`: MDIO master), `RWKSEL` (`bit 6 = 1`: PMT remote wake-up), `MGKSEL` (`bit 7 = 1`: Magic Packet), `MMCSEL` (`bit 8 = 1`: RMON counters), `ARPOFFSEL` (`bit 9 = 1`: ARP offload), `AVSEL` (`bit 11 = 1`: AVB), `TSSEL` (`bit 12 = 1`: IEEE 1588 PTP), `EEESEL` (`bit 13 = 1`: 802.3az EEE), `TXCOESEL` (`bit 14 = 1`: TX checksum offload), `RXCOESEL` (`bit 16 = 1`: RX checksum offload), `ADDMACADRSEL[22:18] = 31` (32 total unicast MAC address slots `0..31`), `SAVLANINS` (`bit 27 = 1`), `VXN` (`bit 29 = 1`: VXLAN/NVGRE), `EDIFFC` (`bit 30 = 1`), `EDMA` (`bit 31 = 1`: Enhanced DMA). |
| `+0x0120` | `MAC_HW_FEATURE1` | `0x01857A69` | `RXFIFOSIZE[4:0] = 9` ($128 \ll 9 = 65536$ nominal encoding; **actual physical shared MTL RX SRAM on TC956x is 46 KB / 47,104 bytes**), `OSTEN` (`bit 5 = 1`: One-Step Timestamp), `TXFIFOSIZE[10:6] = 9` (**actual physical shared MTL TX SRAM on TC956x is 46 KB / 47,104 bytes**), `PTOEN` (`bit 11 = 1`), `ADVTHWORD` (`bit 13 = 1`), `ADDR64[15:14] = 1` (40-bit internal DMA addressing), `DCBEN` (`bit 16 = 1`), `SPHEN` (`bit 17 = 1`: Split Header), `TSOEN` (`bit 18 = 1`: TCP Segmentation Offload), `DBGMEMA` (`bit 19 = 1`), **`RSSEN` (`bit 20 = 0`: Receive Side Scaling is NOT synthesized in hardware!)**, `NUMTC[23:21] = 4` (5 Traffic Classes `0..4`), `HASHTBLSZ[25:24] = 1` (64-bit multicast hash table: two 32-bit registers at `+0x0010` and `+0x0014`), `L3L4FNUM[30:27] = 0` (no L3/L4 filter slots). |
| `+0x0124` | `MAC_HW_FEATURE2` | `0x131C71C7` | `RXQCNT[3:0] = 7` (8 MTL RX queues `0..7`), `TXQCNT[9:6] = 7` (8 MTL TX queues `0..7`), `RXCHCNT[15:12] = 7` (8 RX DMA channels `0..7`), `TXCHCNT[21:18] = 7` (8 TX DMA channels `0..7`), `PPSOUTNUM[26:24] = 3` (4 PPS outputs), `AUXSNAPNUM[30:28] = 1` (2 aux timestamp inputs). |
| `+0x0128` | `MAC_HW_FEATURE3` | `0x3DAC099B` | `NRVF[2:0] = 3` (4 VLAN filters), `CBTISEL` (`bit 4 = 1`), `FRPSEL` (`bit 10 = 1`: Flexible Receive Parser), `FRPPB[12:11] = 0` (64 bytes), `FRPES[14:13] = 1` (128 FRP entries), `ASP[19:18] = 3` (Automotive Safety Package), `ESTSEL` (`bit 20 = 1`: 802.1Qbv Enhancements for Scheduled Traffic), `FPESEL` (`bit 26 = 1`: 802.1Qbu Frame Preemption), `TBSSEL` (`bit 27 = 1`: Time-Based Scheduling), `TBSCH[31:28] = 3` (4 TBS channels). |

**[HW]** **Lack of Hardware RSS (`RSSEN = 0`)**: Because `MAC_HW_FEATURE1` bit 20 (`RSSEN`) is `0`, the TC956x does not implement Toeplitz hash distribution across multiple RX queues. All normal unicast/multicast/broadcast traffic defaults to RX Queue 0 (`DMA_RX_CH0`) unless explicitly steered by VLAN priority (`MAC_RXQ_CTRL2`/`3`), packet type (`MAC_RXQ_CTRL1`), or Flexible Receive Parser (`FRP`) instructions. A single-queue or multi-queue clean-room driver that enables RX Queue 0 and maps it to RX DMA Channel 0 receives all frames on Channel 0.

---

## 2. MAC Core Register Map and Configuration (`+0x0000` to `+0x03FC`)

### 2.1 MAC Transmit and Receive Configuration (`+0x0000`, `+0x0004`, `+0x0040`)

**[HW]** Key MAC Core configuration registers:

| Offset | Register Name | Bits | Field Name | Description / Required Setting |
|---|---|---|---|---|
| `+0x0000` | `MAC_TX_CONFIG` | `[0]` | `TE` | Transmitter Enable (`1` = enabled, `0` = disabled). |
| | | `[16]` | `JD` | Jabber Disable (`1` = allow frames up to 16,383 bytes for TSO / jumbo). |
| | | `[31:29]` | `SS` | **Speed Select** (3-bit field; see speed table below). |
| `+0x0004` | `MAC_RX_CONFIG` | `[0]` | `RE` | Receiver Enable (`1` = enabled, `0` = disabled). |
| | | `[1]` | `ACS` | Automatic Pad or CRC Stripping (`1` = strip FCS on length-field frames). |
| | | `[2]` | `CST` | CRC Stripping for Type frames (`1` = strip 4-byte Ethernet FCS before DMA write). |
| | | `[3]` | `DCRCC` | Disable CRC Check (`0` = hardware checks FCS). |
| | | `[6]` | `GPSLCE` | Giant Packet Size Limit Control Enable (`1` when MTU > 1500). |
| | | `[7]` | `WD` | Watchdog Disable (`1` when frame size > 2048). |
| | | `[8]` | `JE` | Jumbo Packet Enable (`1` when frame size > 1518 / 1522). |
| | | `[9]` | `IPC` | Checksum Offload Enable (`1` = enable hardware IPv4 header + TCP/UDP/ICMP payload checksum verification on RX; `0` = disable). |
| | | `[29:16]` | `GPSL` | Giant Packet Size Limit (e.g., `9018` when jumbo frames are enabled). |
| `+0x0040` | `MAC_EXT_CONFIG` | `[13:0]` | `GPSL` | Extended Giant Packet Size Limit. |
| | | `[16]` | `DCRCC` | Extended CRC check control. |
| | | `[24]` | `EIPG` | Extended Inter-Packet Gap (`0x01000000` observed on Q8B). |

**[HW]** **`MAC_TX_CONFIG` Speed Select (`SS`, bits `[31:29]`) Encoding**:

| `SS` Value (`bits [31:29]`) | Hex Mask (`SS << 29`) | MAC Interface & Rate | Used for QCA8081 Link Speed |
|---|---|---|---|
| `0` (`0b000`) | `0x00000000` | 10 Gbps XGMII | Unused on Q8B |
| `2` (`0b010`) | `0x40000000` | **2.5 Gbps GMII** | **2500BASE-T (`2500BASE-X` SerDes)** |
| `3` (`0b011`) | `0x60000000` | **1000 Mbps GMII** | **1000BASE-T (`SGMII` SerDes)** |
| `4` (`0b100`) | `0x80000000` | **100 Mbps MII** | **100BASE-TX (`SGMII` SerDes)** |
| `5` (`0b101`) | `0xA0000000` | 5 Gbps XGMII | Unused on Q8B |
| `6` (`0b110`) | `0xC0000000` | 2.5 Gbps XGMII | Unused on Q8B (USXGMII only; 2500BASE-X uses `SS = 2`!) |
| `7` (`0b111`) | `0xE0000000` | **10 Mbps MII** | **10BASE-T (`SGMII` SerDes)** |

**[HW]** Note: At 2500 Mbps over 2500BASE-X / SGMII 2.5G (`NEMACxCTL.SP_SEL = 4`), `MAC_TX_CONFIG` bits `[31:29]` (`SS`) must be set to **`2` (`0x40000000`, 2.5G GMII)**, **not** `6` (`0xC0000000`, 2.5G XGMII, which is only for USXGMII). At 1000 Mbps (`SP_SEL = 5`), `SS` is `3` (`0x60000000`, observed `MAC_TX_CONFIG = 0x60010001` on Q8B with `JD | TE`).

### 2.2 Station MAC Address Registers (`+0x0300`, `+0x0304`) and Reset Preservation

**[HW]** The XGMAC2 supports 32 unicast MAC address filter entries (`i = 0..31`) at:
- `MAC_ADDRi_HIGH` (`+0x0300 + i * 8`):
  - Bits `[15:0]`: Upper 2 bytes of MAC address (`addr[4]` in `[7:0]`, `addr[5]` in `[15:8]`).
  - Bit `31` (`AE`): Address Enable (`1` = active; always `1` for slot 0).
  - Bits `[29:24]` (`DCS`): DMA Channel Select bitmap/index (set to `0` to route to RX DMA Channel 0).
- `MAC_ADDRi_LOW` (`+0x0304 + i * 8`):
  - Bits `[31:0]`: Lower 4 bytes of MAC address (`addr[0]` in `[7:0]`, `addr[1]` in `[15:8]`, `addr[2]` in `[23:16]`, `addr[3]` in `[31:24]`).

**[ABI]** **Factory MAC Address Sources on Radxa Dragon Q8B**:
1. **Pre-programmed in `MAC_ADDR0_HIGH`/`LOW` by UEFI**: At boot, UEFI reads the board EEPROM and writes Port 0's MAC address (e.g., `88:12:4e:00:02:00`, where `MAC_ADDR0_HIGH = 0x8000024E`, `MAC_ADDR0_LOW = 0x004E1288`) and Port 1's MAC address (`88:12:4e:00:02:01`) into each port's XGMAC registers.
2. **Board I²C EEPROM**: Stored on QUPv3 I²C bus 12 (`a88000.i2c`, `/dev/iic4`) at 7-bit address `0x50` (8-bit offset addressing):
   - Offset `0x9E` (6 bytes): Port 0 MAC address (`88:12:4e:00:02:00`).
   - Offset `0xA4` (6 bytes): Port 1 MAC address (`88:12:4e:00:02:01`).

**[HW]** **Mandatory MAC Address Save-Before-Reset Rule**: Asserting `NRSTCTRL0` bit 7 (`MAC0RST`), `NRSTCTRL1` bit 7 (`MAC1RST1`), or `DMA_MODE` bit 0 (`SWR` at `+0x3000`) resets `MAC_ADDR0_HIGH` to `0x8000FFFF` and `MAC_ADDR0_LOW` to `0xFFFFFFFF` (`ff:ff:ff:ff:ff:ff`). Therefore, during driver attach:
1. Enable the port's clocks in `NCLKCTRL0` / `NCLKCTRL1` and ensure `MACxRST` in `NRSTCTRL0` / `NRSTCTRL1` is `0` (deasserted).
2. Read `MAC_ADDR0_HIGH` (`+0x0300`) and `MAC_ADDR0_LOW` (`+0x0304`). If the 6-byte address is a valid non-broadcast, non-zero unicast MAC address, save it in driver state.
3. Only **after** saving the MAC address may the driver pulse `MACxRST` (`1` -> `0`) or set `DMA_MODE.SWR` (`+0x3000` bit 0).
4. After `DMA_MODE.SWR` clears, write the saved MAC address back into `MAC_ADDR0_HIGH` (with bit 31 `AE = 1`) and `MAC_ADDR0_LOW`.

### 2.3 Packet Filter and 64-Bit Multicast Hash Table (`+0x0008`, `+0x0010`, `+0x0014`)

**[HW]** `MAC_PACKET_FILTER` (`+0x0008`) controls address filtering:

| Bit | Field Name | Mask | Description |
|---|---|---|---|
| `0` | `PR` | `0x00000001` | Promiscuous Mode (`1` = pass all frames regardless of destination address). |
| `1` | `HUC` | `0x00000002` | Hash Unicast (`1` = filter unicast via hash table). |
| `2` | `HMC` | `0x00000004` | Hash Multicast (`1` = filter multicast frames using the 64-bit hash table at `+0x0010..+0x0014`). |
| `4` | `PM` | `0x00000010` | Pass All Multicast (`1` = pass all multicast frames without checking the hash table). |
| `5` | `DBF` | `0x00000020` | Disable Broadcast Frames (`0` = accept broadcast). |
| `10` | `HPF` | `0x00000400` | Hash or Perfect Filter (`1` = accept if either hash or perfect table matches). |
| `16` | `VTFE` | `0x00010000` | VLAN Tag Filter Enable (`1` = drop frames not matching VLAN filter). |
| `31` | `RA` | `0x80000000` | Receive All (`1` = pass all frames to DMA, marking filter status in RX descriptor). |

**[HW]** **Multicast Hash Calculation (`HASHTBLSZ = 1`, 64-Bit Table)**:
 Because `MAC_HW_FEATURE1` bits `[25:24]` (`HASHTBLSZ`) equal `1`, the multicast hash table consists of two 32-bit registers:
- `MAC_HASH_TAB_0_31` (`+0x0010`): Hash bins `0..31`
- `MAC_HASH_TAB_32_63` (`+0x0014`): Hash bins `32..63`

To compute the 6-bit hash index `bin` ($0 \le \text{bin} \le 63$) for a 6-byte multicast MAC address `addr[0..5]`:
1. Compute the standard IEEE 802.3 little-endian CRC-32 (polynomial `0xEDB88320`, initial value `0xFFFFFFFF`, without final XOR inversion—or equivalently, bitwise-invert the standard finalized CRC-32):
   $$\text{crc}_{\text{raw}} = \text{CRC32\_LE}(\text{seed}=\text{0xFFFFFFFF},\ \text{addr}[0..5])$$
2. Bit-reverse the 32-bit value $\sim\text{crc}_{\text{raw}}$ (i.e., bit 0 becomes bit 31, bit 31 becomes bit 0) and take the upper 6 bits (`[31:26]` of the bit-reversed inverted CRC, which is identical to taking the lower 6 bits `[5:0]` of $\sim\text{crc}_{\text{raw}}$ and reversing those 6 bits):
   $$\text{bin} = \text{BitReverse32}(\sim\text{crc}_{\text{raw}}) \gg 26$$
3. Set bit $(\text{bin} \mathbin{\&} 31)$ in register `+0x0010` (if $\text{bin} < 32$) or register `+0x0014` (if $\text{bin} \ge 32$).
4. Set `HMC` (bit 2, `0x00000004`) in `MAC_PACKET_FILTER` (`+0x0008`).

### 2.4 VLAN Tag Offload and Filtering (`+0x0050` to `+0x0064`)

**[HW]** VLAN tag stripping, insertion, and filtering registers:
- `MAC_VLAN_TAG` (`+0x0050`):
  - Bit `18` (`ESVL`): Enable S-VLAN (`0x88A8`) processing.
  - Bits `[22:21]` (`EVLS`): Enable VLAN Tag Stripping on Receive (`0` = do not strip, `1` = strip if filter passes, `2` = strip if filter fails, `3` = **always strip outer VLAN tag on RX** and place it in RX write-back descriptor `RDES0[15:0]`).
  - Bit `24` (`EVLRXS`): Enable VLAN Tag in RX Status (`1` = report stripped VLAN tag in `RDES0[15:0]` and set `RDES3.RS0V` + `RDES2.LT = 0b100` or `0b101`).
  - Bit `25` (`VTHM`): VLAN Tag Hash Table Match Enable.
  - Bit `26` (`EDVLP`): Enable Double VLAN Processing.
- `MAC_VLAN_INCL` (`+0x0060`):
  - Bits `[15:0]` (`VLT`): Default VLAN tag value.
  - Bits `[17:16]` (`VLC`): VLAN Command (`2` = insert VLAN tag).
  - Bit `18` (`VLP`): VLAN Priority control.
  - Bit `19` (`CSVL`): C-VLAN (`0`) or S-VLAN (`1`).
  - Bit `20` (`VLTI`): **VLAN Tag Input from TX Context Descriptor** (`1` = enable per-packet hardware VLAN insertion using the `VT[15:0]` field of a preceding TX Context Descriptor plus `TDES2.VTIR = 2` on the TX first descriptor).

### 2.5 RX Queue Enable and DMA Channel Routing (`+0x00A0` to `+0x00AC`, `+0x1030`)

**[HW]** Even when only RX Queue 0 is used, it must be explicitly enabled and mapped to RX DMA Channel 0:
1. `MAC_RXQ_CTRL0` (`+0x00A0`): Each RX queue `q` (`0..7`) uses a 2-bit field at bits `[2q+1 : 2q]`:
   - `0` (`0b00`): Queue disabled
   - `1` (`0b01`): Queue enabled for AV traffic
   - `2` (`0b10`): **Queue enabled for Generic / Data Center Bridging (DCB) traffic**
   - For a single-queue driver (Queue 0 only), write `0x00000002` to `MAC_RXQ_CTRL0` (`+0x00A0`).
2. `MAC_RXQ_CTRL1` (`+0x00A4`), `MAC_RXQ_CTRL2` (`+0x00A8`), `MAC_RXQ_CTRL3` (`+0x00AC`): Set to `0x00000000` so untagged, multicast, broadcast, and all VLAN priorities route to RX Queue 0.
3. `MTL_RXQ_DMA_MAP0` (`+0x1030`, covering RX queues `0..3`) and `MTL_RXQ_DMA_MAP1` (`+0x1034`, covering RX queues `4..7`):
   - Each queue $q \in [0..3]$ occupies a byte at bits `[8q+7 : 8q]` in `MTL_RXQ_DMA_MAP0`:
     - Bits `[8q+2 : 8q]` (`QxMDMACH`): Target RX DMA channel number (`0..7`).
     - Bit `8q+4` (`QxDDMACH`): Dynamic DMA channel selection (`0` = static mapping).
   - For static 1:1 mapping (or single-queue Queue 0 -> DMA Channel 0), write `0x03020100` (or `0x00000000` for Queue 0 only) to `MTL_RXQ_DMA_MAP0` (`+0x1030`).

---

## 3. MDIO Master Controller (`+0x0200` to `+0x0220`): Clause 22 and Clause 45

**[HW]** Each port's XGMAC2 has an independent MDIO (Station Management Agent) master connected to that port's external QCA8081 PHY (which responds at MDIO addresses `0x1c` / 28 for the PHY core and `0x1d` / 29 for the SerDes PCS/PMA block).

### 3.1 MDIO Registers (`+0x0200`, `+0x0204`, `+0x0220`)

| Offset | Register Name | Bits | Field Name | Description |
|---|---|---|---|---|
| `+0x0200` | `MAC_MDIO_ADDR` | `[15:0]` | `RA` | Register Address: 5-bit register (`[4:0]`) in Clause 22 mode, or 16-bit register (`[15:0]`) in Clause 45 mode. |
| | | `[20:16]` | `PA` | 5-bit PHY / Port Address (`0..31`; `0x1c` = 28 for QCA8081, `0x1d` = 29 for QCA8081 SerDes). |
| | | `[25:21]` | `DA` | 5-bit Device Address / MMD (`0..31`) in Clause 45 mode; `0` in Clause 22 mode. |
| `+0x0204` | `MAC_MDIO_DATA` | `[15:0]` | `GD` | 16-bit MDIO Read / Write Data. |
| | | `[17:16]` | `CMD` | Operation command: **`1` (`0x00010000`) = Write**, **`3` (`0x00030000`) = Read**. |
| | | `[18]` | `SADDR` | Skip Address Frame (`0x00040000`; set on Clause 22 writes, `0` on Clause 45). |
| | | `[21:19]` | `CR` | MDC Clock Range divider (`0..7`; `0` observed on Q8B). |
| | | `[22]` | `SBUSY` | Start / Busy (`0x00400000`). Set to `1` to start transaction; hardware clears to `0` when complete. |
| `+0x0220` | `MAC_MDIO_C22P` | `[31:0]` | `C22P` | Clause 22 Port Select bitmask: bit `pa` (`1 << pa`) = `1` selects **Clause 22** framing for PHY address `pa`; bit `pa` = `0` selects **Clause 45** framing for PHY address `pa`. |

**[HW]** **Note on `MAC_MDIO_C22P` (`+0x0220`) Bit Width on XGMAC2 v3.01a**:
While XGMAC2 cores older than v2.20 (`SNPSVER < 0x22`) only implemented bits `[3:0]` of `MAC_MDIO_C22P` (`phyaddr <= 3`), the TC956x integrates XGMAC2 v3.01a (`SNPSVER = 0x30`), which implements all 32 bits `[31:0]` of `MAC_MDIO_C22P`. On the Radxa Dragon Q8B, `MAC_MDIO_C22P` reads `0x10000000` (`1 << 28`) after a Clause 22 transaction to PHY address `0x1c` (28).

### 3.2 Step-by-Step MDIO Read and Write Sequences

#### Clause 22 Read (`phyaddr` $0..31$, `reg` $0..31$)
1. **[HW]** Poll `MAC_MDIO_DATA` (`+0x0204`) until bit 22 (`SBUSY`, `0x00400000`) is `0` (timeout `10 ms`).
2. **[HW]** Read `MAC_MDIO_C22P` (`+0x0220`), set bit `(1 << phyaddr)`, and write back to `+0x0220`.
3. **[HW]** Write `((phyaddr & 0x1F) << 16) | (reg & 0x1F)` to `MAC_MDIO_ADDR` (`+0x0200`).
4. **[HW]** Write `0x00430000 | (cr << 19)` (`SBUSY | CMD_READ`) to `MAC_MDIO_DATA` (`+0x0204`).
5. **[HW]** Poll `MAC_MDIO_DATA` (`+0x0204`) until bit 22 (`SBUSY`, `0x00400000`) is `0`.
6. **[HW]** Return `MAC_MDIO_DATA & 0xFFFF`.

#### Clause 22 Write (`phyaddr` $0..31$, `reg` $0..31$, `val` $16\text{-bit}$)
1. **[HW]** Poll `MAC_MDIO_DATA` (`+0x0204`) until bit 22 (`SBUSY`, `0x00400000`) is `0`.
2. **[HW]** Read `MAC_MDIO_C22P` (`+0x0220`), set bit `(1 << phyaddr)`, and write back to `+0x0220`.
3. **[HW]** Write `((phyaddr & 0x1F) << 16) | (reg & 0x1F)` to `MAC_MDIO_ADDR` (`+0x0200`).
4. **[HW]** Write `0x00450000 | (cr << 19) | (val & 0xFFFF)` (`SBUSY | SADDR | CMD_WRITE | val`) to `MAC_MDIO_DATA` (`+0x0204`).
5. **[HW]** Poll `MAC_MDIO_DATA` (`+0x0204`) until bit 22 (`SBUSY`, `0x00400000`) is `0`.

#### Clause 45 Read (`phyaddr` $0..31$, `devad` $0..31$, `reg` $16\text{-bit}$)
1. **[HW]** Poll `MAC_MDIO_DATA` (`+0x0204`) until bit 22 (`SBUSY`, `0x00400000`) is `0`.
2. **[HW]** Read `MAC_MDIO_C22P` (`+0x0220`), **clear** bit `~(1 << phyaddr)`, and write back to `+0x0220`.
3. **[HW]** Write `((devad & 0x1F) << 21) | ((phyaddr & 0x1F) << 16) | (reg & 0xFFFF)` to `MAC_MDIO_ADDR` (`+0x0200`).
4. **[HW]** Write `0x00430000 | (cr << 19)` (`SBUSY | CMD_READ`) to `MAC_MDIO_DATA` (`+0x0204`).
5. **[HW]** Poll `MAC_MDIO_DATA` (`+0x0204`) until bit 22 (`SBUSY`, `0x00400000`) is `0`.
6. **[HW]** Return `MAC_MDIO_DATA & 0xFFFF`.

#### Clause 45 Write (`phyaddr` $0..31$, `devad` $0..31$, `reg` $16\text{-bit}$, `val` $16\text{-bit}$)
1. **[HW]** Poll `MAC_MDIO_DATA` (`+0x0204`) until bit 22 (`SBUSY`, `0x00400000`) is `0`.
2. **[HW]** Read `MAC_MDIO_C22P` (`+0x0220`), **clear** bit `~(1 << phyaddr)`, and write back to `+0x0220`.
3. **[HW]** Write `((devad & 0x1F) << 21) | ((phyaddr & 0x1F) << 16) | (reg & 0xFFFF)` to `MAC_MDIO_ADDR` (`+0x0200`).
4. **[HW]** Write `0x00410000 | (cr << 19) | (val & 0xFFFF)` (`SBUSY | CMD_WRITE | val`) to `MAC_MDIO_DATA` (`+0x0204`).
5. **[HW]** Poll `MAC_MDIO_DATA` (`+0x0204`) until bit 22 (`SBUSY`, `0x00400000`) is `0`.

---

## 4. MTL (MAC Transaction Layer) FIFO Sizing and 802.3x Flow Control

**[HW]** Each port has **46 KB (47,104 bytes)** of physical on-chip TX FIFO SRAM and **46 KB (47,104 bytes)** of physical on-chip RX FIFO SRAM shared across its enabled MTL queues. FIFO size fields (`TQS` and `RQS`) are programmed in units of 256 bytes minus 1:

$$\text{TQS or RQS} = \frac{\text{Queue\_Bytes}}{256} - 1$$

### 4.1 MTL Per-Queue Registers (`+0x1100 + q * 0x80`)

**[HW]** For Queue $q \in [0..7]$, base offset is `+0x1100 + (q * 0x80)`:

| Offset (Queue $q$) | Register Name | Bits | Field Name | Description / Recommended Single-Queue Setting |
|---|---|---|---|---|
| `+0x1100 + q*0x80` | `MTL_TXQq_OP_MODE` | `[1]` | `TSF` | Transmit Store and Forward (`1` = required for TX checksum offload and TSO). |
| | | `[3:2]` | `TXQEN` | TX Queue Enable (`2` (`0b10`) = enabled, `0` = disabled). |
| | | `[25:16]` | `TQS` | Transmit Queue Size in 256B blocks minus 1. For single TX queue using 32 KB: `TQS = 127` (`0x7F`, register value `0x007F000A`). Up to 46 KB (`TQS = 183` / `0xB7`) is available if only Queue 0 is enabled. |
| `+0x1118 + q*0x80` | `MTL_TXQq_QUANTUM_WEIGHT` | `[20:0]` | `ISCQW` | Quantum / weight for WRR/DWRR (`0x00000010` default). |
| `+0x1140 + q*0x80` | `MTL_RXQq_OP_MODE` | `[5]` | `RSF` | Receive Store and Forward (`1` = drop/mark errored frames and enable RX COE). |
| | | `[7]` | `EHFC` | Enable Hardware Flow Control (`1` = assert 802.3x Pause based on `RFA`/`RFD` thresholds when FIFO size $\ge 4\text{ KB}$). |
| | | `[25:16]` | `RQS` | Receive Queue Size in 256B blocks minus 1. For single RX queue using 32 KB: `RQS = 127` (`0x7F`, observed Q8B value `0x007F00A0` with `EHFC | RSF`). |
| `+0x1150 + q*0x80` | `MTL_RXQq_FLOW_CONTROL` | `[15:1]` | `RFA` | Threshold for Activating Flow Control (in 512B units below full minus 1 KB: $\text{Headroom} = (1 + 0.5 \times \text{RFA})\text{ KB}$). Set `RFA = 14` (`14 << 1 = 0x1C`, 8 KB headroom). |
| | | `[31:17]` | `RFD` | Threshold for Deactivating Flow Control ($\text{Headroom} = (1 + 0.5 \times \text{RFD})\text{ KB}$). Set `RFD = 22` (`22 << 17 = 0x002C0000`, 12 KB headroom; combined register value `0x002C001C`). |
| `+0x1170 + q*0x80` | `MTL_Qq_INT_ENABLE` | `[16]` | `RXOIE` | RX Queue Overflow Interrupt Enable. |
| `+0x1174 + q*0x80` | `MTL_Qq_INT_CTRL_STATUS` | `[16]` | `RXOVFIS` | RX Queue Overflow Interrupt Status (write `1` to clear). |

**[POLICY]** **Multi-Queue FIFO Partitioning (Optional)**: When all 8 queues are enabled (as in the Toshiba vendor reference configuration), the 46 KB SRAM is partitioned as:
- TX Queues (`46 KB` total): `TXQ0 = 20 KB` (`TQS = 79`), `TXQ1 = 2 KB` (`TQS = 7`), `TXQ2..7 = 4 KB` each (`TQS = 15`).
- RX Queues (`46 KB` total): `RXQ0..2 = 8 KB` each (`RQS = 31`), `RXQ3..6 = 4 KB` each (`RQS = 15`), `RXQ7 = 6 KB` (`RQS = 23`).
For a single-queue driver (Queue 0 only), assigning `32 KB` (`TQS = 127`, `RQS = 127`) or the full `46 KB` (`TQS = 183`, `RQS = 183`) to Queue 0 is optimal.

### 4.2 MAC 802.3x Pause Frame Registers (`+0x0070`, `+0x0090`)

**[HW]** To enable symmetric 802.3x Pause flow control in full-duplex mode:
1. `MAC_Q0_TX_FLOW_CTRL` (`+0x0070`): Write `0xFFFF0002` (`PT[31:16] = 0xFFFF` pause time quanta, `TFE[1] = 1` Transmit Flow Control Enable).Wait until bit 0 (`FCB` / Flow Control Busy) is `0` before modifying.
2. `MAC_RX_FLOW_CTRL` (`+0x0090`): Write `0x00000001` (`RFE[0] = 1` Receive Flow Control Enable).

---

## 5. XGMAC2 DMA Engine (`+0x3000` to `+0x34FC`) and v3.01a Hardware Errata

### 5.1 Global DMA Registers (`+0x3000` to `+0x3050`)

**[HW]** Global DMA control and status registers:

| Offset | Register Name | Bits | Field Name | Description / Required Setting |
|---|---|---|---|---|
| `+0x3000` | `DMA_MODE` | `[0]` | `SWR` | Software Reset (`1` = reset MAC, MTL, and DMA; hardware self-clears to `0` when complete, poll every 10 ms up to 100 ms). **Warning: clears `MAC_ADDR0_HIGH/LOW`!** |
| | | `[8]` | `DSPW` | Descriptor Posted Write. **MUST BE `0` on XGMAC 3.01a! (See Erratum 1 below).** |
| | | `[13:12]` | `INTM` | Interrupt Mode: **`1` (`0x00001000`)** required for TC956x `MSIGEN` operation (observed Q8B value `0x00001000`). |
| `+0x3004` | `DMA_SYSBUS_MODE` | `[0]` | `UNDEF` | Undefined Burst Length (`1` = allow arbitrary burst lengths). |
| | | `[7:1]` | `BLEN4..256` | Allowed AXI burst lengths: bit 1 = `BLEN4`, bit 2 = `BLEN8`, bit 3 = `BLEN16`, bit 4 = `BLEN32`, bit 5 = `BLEN64`, bit 6 = `BLEN128`, bit 7 = `BLEN256`. |
| | | `[11]` | `EAME` | **Enhanced Address Mode Enable (`1` = mandatory to enable 40-bit DMA addressing with `HADDR` registers)**. |
| | | `[12]` | `AAL` | Address-Aligned Beats (`1` = align bursts). |
| | | `[21:16]` | `RD_OSR_LMT` | AXI Maximum Read Outstanding Requests minus 1 (`31` = 32 outstanding reads). |
| | | `[29:24]` | `WR_OSR_LMT` | AXI Maximum Write Outstanding Requests minus 1 (`31` = 32 outstanding writes). |
| | | — | — | **Recommended / Observed Q8B Value**: `0x1F1F087F` (`WR_OSR_LMT = 31`, `RD_OSR_LMT = 31`, `EAME = 1`, `BLEN256..4 = 0x7F`, `UNDEF = 1`). |
| `+0x3008` | `DMA_INT_STATUS` | `[7:0]` | `DCnIS` | Per-channel DMA interrupt pending (`bit n = 1` when Channel $n$ has an active interrupt). |
| | | `[16]` | `MTLIS` | MTL interrupt pending. |
| | | `[17]` | `MACIS` | MAC Core interrupt pending. |
| `+0x3040` | `DMA_AXI_TX_AR_ACE_CTRL` | `[31:0]` | — | AXI TX Read cache control (`0x00000000`). |
| `+0x3044` | `DMA_AXI_RX_AW_ACE_CTRL` | `[31:0]` | — | AXI RX Write cache control (`0x00000000`). |
| `+0x3048` | `DMA_AXI_TXRX_AWAR_ACE_CTRL` | `[31:0]` | — | AXI descriptor Read/Write cache control (`0x00000000`). |

### 5.2 Per-Channel DMA Registers (`+0x3100 + ch * 0x80`)

**[HW]** Each DMA channel $ch \in [0..7]$ occupies a 128-byte (`0x80`) block starting at `+0x3100 + (ch * 0x80)`:

| Offset (Channel $ch$) | Register Name | Bits | Field Name | Description / Required Setting |
|---|---|---|---|---|
| `+0x3100 + ch*0x80` | `DMA_CH_CONTROL` | `[16]` | `PBLx8` | Multiply Programmable Burst Length (`TxPBL`/`RxPBL`) by 8 (`1` = enabled, `0x00010000`). |
| | | `[24]` | `SPH` | Split Header Enable (`0` for standard contiguous RX buffers). |
| `+0x3104 + ch*0x80` | `DMA_CH_TX_CONTROL` | `[0]` | `TXST` | Start / Stop Transmission (`1` = start TX DMA, `0` = stop). |
| | | `[4]` | `OSP` | Operate on Second Packet (`1` = fetch next frame before previous frame status returns; `0x00000010`). |
| | | `[12]` | `TSE` | TCP Segmentation Enable (`1` = enable hardware TSO on this channel; `0x00001000`). |
| | | `[21:16]` | `TxPBL` | Transmit Programmable Burst Length (`32` = `0x20` when `PBLx8 = 1`, observed Q8B value `0x00201011`). |
| `+0x3108 + ch*0x80` | `DMA_CH_RX_CONTROL` | `[0]` | `RXST` | Start / Stop Receive (`1` = start RX DMA, `0` = stop). |
| | | `[14:1]` | `RBSZ` | **Receive Buffer Size in bytes** (must be a multiple of 8 bytes; bit 0 is `RXST`, so the byte size sits in bits `[14:1]`, i.e., `(buf_size & 0x3FF8)` ORed directly into the register!). E.g., for `1536` (`0x600`) bytes, bits `[14:1]` hold `0x0600`; for `2048` (`0x800`) bytes, bits `[14:1]` hold `0x0800`; for `4096` (`0x1000`), bits `[14:1]` hold `0x1000`. |
| | | `[21:16]` | `RxPBL` | Receive Programmable Burst Length (`32` = `0x20`; observed Q8B value `0x00200601` for 1536B buffers). |
| | | `[31]` | `RPF` | RX Packet Flush. |
| `+0x3110 + ch*0x80` | `DMA_CH_TxDESC_HADDR` | `[7:0]` | `TDESHA` | Upper 32 bits of TX descriptor ring base AXI address: **`0x00000010 | upper_32_bits(tx_ring_phys)`**. |
| `+0x3114 + ch*0x80` | `DMA_CH_TxDESC_LADDR` | `[31:4]` | `TDESLA` | Lower 32 bits of TX descriptor ring base address (must be 16-byte aligned). |
| `+0x3118 + ch*0x80` | `DMA_CH_RxDESC_HADDR` | `[7:0]` | `RDESHA` | Upper 32 bits of RX descriptor ring base AXI address: **`0x00000010 | upper_32_bits(rx_ring_phys)`**. |
| `+0x311C + ch*0x80` | `DMA_CH_RxDESC_LADDR` | `[31:4]` | `RDESLA` | Lower 32 bits of RX descriptor ring base address (must be 16-byte aligned). |
| `+0x3124 + ch*0x80` | `DMA_CH_TxDESC_TAIL_LPTR` | `[31:4]` | `TDT` | Low 32 bits of the TX descriptor address **one slot past** the last valid software-populated descriptor (or pointing to the end of the ring). Writing this register wakes the TX DMA engine. |
| `+0x312C + ch*0x80` | `DMA_CH_RxDESC_TAIL_LPTR` | `[31:4]` | `RDT` | Low 32 bits of the RX descriptor address **one slot past** the last software-refilled descriptor (in a power-of-2 ring of $N$ descriptors at `rx_ring_phys`, writing `lower_32_bits(rx_ring_phys + N * 16)` or the address of the last refilled descriptor wakes the RX DMA engine). |
| `+0x3130 + ch*0x80` | `DMA_CH_TxDESC_RING_LEN` | `[12:0]` | `TDRL` | TX Descriptor Ring Length **minus 1** ($N_{\text{TX}} - 1$, e.g. `511` / `0x01FF` for 512 descriptors). |
| `+0x3134 + ch*0x80` | `DMA_CH_RxDESC_RING_LEN` | `[12:0]` | `RDRL` | RX Descriptor Ring Length **minus 1** ($N_{\text{RX}} - 1$, e.g. `511` / `0x01FF` for 512 descriptors). |
| | | `[25:24]` | `OWRQ` | **Maximum Outstanding Write Requests for RX Descriptor Write-Back minus 1. MUST BE `3` (`0x03000000`) on XGMAC 3.01a! (See Erratum 2 below).** Observed Q8B value: `0x030001FF`. |
| `+0x3138 + ch*0x80` | `DMA_CH_INT_EN` | `[0]` | `TIE` | Transmit Interrupt Enable (`1`). |
| | | `[1]` | `TSE` | Transmit Stopped Enable (`0`). |
| | | `[2]` | `TBUE` | Transmit Buffer Unavailable Enable (`1` or `0`). |
| | | `[6]` | `RIE` | Receive Interrupt Enable (`1`). |
| | | `[7]` | `RBUE` | Receive Buffer Unavailable Enable (`1`). |
| | | `[8]` | `RSE` | Receive Stopped Enable (`0`). |
| | | `[12]` | `FBEE` | Fatal Bus Error Enable (`1`). |
| | | `[13]` | `CDEE` | Context Descriptor Error Enable (`0`). |
| | | `[14]` | `AIE` | Abnormal Interrupt Summary Enable (`1`, enables `RBUE | FBEE`). |
| | | `[15]` | `NIE` | Normal Interrupt Summary Enable (`1`, enables `TIE | RIE | TBUE`). Observed Q8B value: `0x0000D0C1` (`NIE | AIE | FBEE | RBUE | RIE | TIE`). |
| `+0x313C + ch*0x80` | `DMA_CH_Rx_WATCHDOG` | `[7:0]` | `RWT` | Receive Watchdog Timer count in units of $256 \times \text{RWTU}$ DMA clock cycles (`125 MHz` $\Rightarrow 2.048\ \mu\text{s}$ per unit when `RWTU = 0`). Observed Q8B value: `0x00000040` (`64` $\approx 131\ \mu\text{s}$). |
| | | `[17:16]` | `RWTU` | Watchdog unit multiplier (`0` = 256 clocks, `1` = 512, `2` = 1024, `3` = 2048). |
| `+0x3144 + ch*0x80` | `DMA_CH_CUR_TxDESC_LADDR` | `[31:0]` | `CTD` | Current Host TX Descriptor Address Low (read-only diagnostic). |
| `+0x314C + ch*0x80` | `DMA_CH_CUR_RxDESC_LADDR` | `[31:0]` | `CRD` | Current Host RX Descriptor Address Low (read-only diagnostic). |
| `+0x3154 + ch*0x80` | `DMA_CH_CUR_TxBUFF_LADDR` | `[31:0]` | `CTB` | Current Host TX Buffer Address Low (read-only diagnostic). |
| `+0x315C + ch*0x80` | `DMA_CH_CUR_RxBUFF_LADDR` | `[31:0]` | `CRB` | Current Host RX Buffer Address Low (read-only diagnostic). |
| `+0x3160 + ch*0x80` | `DMA_CH_STATUS` | `[0]` | `TI` | Transmit Interrupt (write `1` to clear). |
| | | `[1]` | `TPS` | Transmit Process Stopped (write `1` to clear). |
| | | `[2]` | `TBU` | Transmit Buffer Unavailable (write `1` to clear). |
| | | `[6]` | `RI` | Receive Interrupt (write `1` to clear). |
| | | `[7]` | `RBU` | Receive Buffer Unavailable (write `1` to clear). |
| | | `[8]` | `RPS` | Receive Process Stopped (write `1` to clear). |
| | | `[12]` | `FBE` | Fatal Bus Error (write `1` to clear; bits `[18:16]` `REB` and `[21:19]` `TEB` indicate error type). |
| | | `[14]` | `AIS` | Abnormal Interrupt Summary (sticky OR of `TPS | RBU | RPS | FBE`; write `1` to clear along with the underlying cause bit). |
| | | `[15]` | `NIS` | Normal Interrupt Summary (sticky OR of `TI | TBU | RI`; write `1` to clear along with the underlying cause bit). |

### 5.3 Mandatory Synopsys XGMAC2 v3.01a Hardware Errata Workarounds

**[HW]** The DesignWare XGMAC2 v3.01a silicon inside the TC956x has two hardware errata that **must** be respected by any driver:

1. **Erratum 1 — Do Not Enable `DMA_MODE.DSPW` (`+0x3000` bit 8)**:
   - Setting `DSPW` (Descriptor Posted Write, bit 8 of `DMA_MODE` at `+0x3000`) to `1` causes RX descriptor write-back race conditions where the host CPU sees `RDES3.OWN == 0` before all 16 bytes of the write-back descriptor have landed in DRAM. Leave `DMA_MODE` bit 8 cleared to `0` (`DMA_MODE = 0x00001000`).
2. **Erratum 2 — Program `OWRQ = 3` in `DMA_CH_RxDESC_RING_LEN` (`+0x3134` bits `[25:24]`)**:
   - By default, the RX DMA engine issues up to 4 outstanding descriptor write-back requests (`OWRQ = 3` reduces the internal pipelined limit from the unthrottled hardware default to `3` in bits `[25:24]`). On XGMAC 3.01a, whenever writing `DMA_CH_RxDESC_RING_LEN` (`+0x3134 + ch * 0x80`), the driver **must** OR `0x03000000` (`3 << 24`) with `(rx_ring_len - 1)` (for example, `0x030001FF` for a 512-entry RX ring).

---

## 6. 16-Byte Enhanced DMA Descriptor Formats

**[HW]** Each TX and RX descriptor is **16 bytes** (`4 x 32-bit` little-endian words: `DES0`, `DES1`, `DES2`, `DES3`) and must be 16-byte aligned in host memory.
**[HW]** **Memory Ordering Barrier Rule**: On weakly ordered architectures (ARM64), when handing a descriptor to the DMA engine, the driver must write `DES0`, `DES1`, and `DES2` first, execute a write memory barrier (`dmb(oshst)` / `wmb()`), write `DES3` (which contains bit 31 `OWN = 1`), and execute another write barrier before writing the tail pointer register (`DMA_CH_TxDESC_TAIL_LPTR` or `DMA_CH_RxDESC_TAIL_LPTR`). Conversely, when polling completed descriptors where `DES3` bit 31 (`OWN`) has been cleared to `0` by hardware, the driver must execute a read memory barrier (`dmb(oshld)` / `rmb()`) after observing `OWN == 0` before reading `DES0`, `DES1`, `DES2`, or the packet buffer.

### 6.1 Normal Transmit Read Descriptor (`TDES0..TDES3`, Written by Driver)

**[HW]** Prepared by the driver for normal packet transmission (`TDES3` bit 30 `CTXT = 0`):

| Word / Offset | Bits | Field Name | Description |
|---|---|---|---|
| `TDES0` (`+0x00`) | `[31:0]` | `BUF1AP_LO` | Buffer 1 AXI Address Low (`lower_32_bits(buf_phys)`). |
| `TDES1` (`+0x04`) | `[31:0]` | `BUF1AP_HI` | Buffer 1 AXI Address High: **`0x00000010 | upper_32_bits(buf_phys)`**. |
| `TDES2` (`+0x08`) | `[13:0]` | `B1L` | Buffer 1 Byte Length ($1..16383$). |
| | `[15:14]` | `VTIR` | VLAN Tag Insertion / Replacement (`0` = none, `1` = remove tag, **`2` = insert VLAN tag from preceding TX Context Descriptor**, `3` = replace tag). |
| | `[29:16]` | `B2L` / `HL` | Buffer 2 Length (`0` when using 1 buffer per descriptor). |
| | `[30]` | `TTSE` | Transmit Timestamp Enable (`1` = capture IEEE 1588 TX timestamp on `LD = 1`). |
| | `[31]` | `IOC` | **Interrupt on Completion** (`1` = set `DMA_CH_STATUS.TI` when this descriptor completes). |
| `TDES3` (`+0x0C`) | `[14:0]` | `FL` / `TPL` | Frame Length (total byte length of the frame across all segments, when `TSE = 0`) OR TCP Payload Length (`[17:0]`, when `TSE = 1`). Only required on the First Descriptor (`FD = 1`). |
| | `[17:16]` | `CIC` | **Checksum Insertion Control** (when `TSE = 0`, on `FD = 1`):<br>- `0` (`0b00`): Checksum offload disabled.<br>- `1` (`0b01`): Insert IPv4 header checksum only.<br>- `2` (`0b10`): Insert IPv4 header checksum and TCP/UDP/ICMP payload checksum (pseudo-header pre-calculated).<br>- `3` (`0b11`): **Insert IPv4 header checksum AND full TCP/UDP/ICMP payload + pseudo-header checksum calculated by hardware.** |
| | `[18]` | `TSE` | **TCP Segmentation Enable** (`1` = perform TSO using the `MSS` programmed in a preceding TX Context Descriptor; `TDES2.B1L` on the first descriptor must hold the Ethernet + IP + TCP header length `THL`, or `TDES3[22:19]` holds `THL / 4`). |
| | `[22:19]` | `THL` | TCP Header Length in 32-bit words (when `TSE = 1`). |
| | `[28]` | `LD` | **Last Descriptor** (`1` = last fragment of the packet). |
| | `[29]` | `FD` | **First Descriptor** (`1` = first fragment of the packet). |
| | `[30]` | `CTXT` | Context Type (`0` = Normal TX Data Descriptor). |
| | `[31]` | `OWN` | **Ownership** (`1` = owned by XGMAC DMA hardware, `0` = owned by host CPU). |

### 6.2 Transmit Write-Back Descriptor (`TDES0..TDES3`, Written by Hardware on Completion)

**[HW]** When the DMA finishes transmitting a descriptor, it writes back `TDES3` (and `TDES0`/`TDES1` if TX timestamping was requested):

| Word / Offset | Bits | Field Name | Description |
|---|---|---|---|
| `TDES0` (`+0x00`) | `[31:0]` | `TTSL` | Transmit Timestamp Low (nanoseconds, valid when `LD = 1` and `TDES3.TTSS = 1`). |
| `TDES1` (`+0x04`) | `[31:0]` | `TTSH` | Transmit Timestamp High (seconds, valid when `LD = 1` and `TDES3.TTSS = 1`). |
| `TDES2` (`+0x08`) | `[31:0]` | Reserved | Unmodified / reserved. |
| `TDES3` (`+0x0C`) | `[0]` | `IHE` | IP Header Error (`1` = invalid IP header during COE/TSO). |
| | `[1]` | `DB` | Deferred Bit. |
| | `[2]` | `UF` | Underflow Error (`1` = TX FIFO underflowed). |
| | `[3]` | `ED` | Excessive Deferral. |
| | `[7:4]` | `CC` | Collision Count. |
| | `[8]` | `EC` | Excessive Collisions. |
| | `[9]` | `LC` | Late Collision. |
| | `[10]` | `NC` | No Carrier. |
| | `[11]` | `LOC` | Loss of Carrier. |
| | `[12]` | `PCE` | Payload Checksum Error. |
| | `[13]` | `FF` | Packet Flushed. |
| | `[14]` | `JT` | Jabber Timeout. |
| | `[15]` | `ES` | **Error Summary** (`1` = OR of `IHE | UF | ED | EC | LC | NC | LOC | PCE | FF | JT`). |
| | `[17]` | `TTSS` | Transmit Timestamp Status (`1` = valid timestamp in `TDES0`/`TDES1`). |
| | `[28]` | `LD` | Last Descriptor (`1`). |
| | `[29]` | `FD` | First Descriptor. |
| | `[30]` | `CTXT` | Context Type (`0`). |
| | `[31]` | `OWN` | **Ownership** (`0` = returned to host CPU). |

### 6.3 Transmit Context Descriptor (`TDES0..TDES3` with `CTXT = 1`)

**[HW]** Placed in the TX ring immediately **before** a TX First Descriptor (`FD = 1`) whenever the TSO Maximum Segment Size (`MSS`) or the TX VLAN tag (`VT`) changes:

| Word / Offset | Bits | Field Name | Description |
|---|---|---|---|
| `TDES0` (`+0x00`) | `[31:0]` | `TTSL` | Launch Time Low (only if TBS is used; `0` otherwise). |
| `TDES1` (`+0x04`) | `[31:0]` | `TTSH` | Launch Time High (`0`). |
| `TDES2` (`+0x08`) | `[13:0]` | `MSS` | **Maximum Segment Size** for TSO ($1..16383$ bytes). |
| | `[31:16]` | `IVT` | Inner VLAN Tag (`0` if single VLAN). |
| `TDES3` (`+0x0C`) | `[15:0]` | `VT` | **Outer 802.1Q VLAN Tag** (`[15:13]` PCP, `[12]` DEI, `[11:0]` VID) to insert when subsequent data descriptor has `TDES2.VTIR = 2`. |
| | `[16]` | `VLTV` | **VLAN Tag Valid** (`1` = update channel's active TX VLAN tag from `VT[15:0]`). |
| | `[17]` | `IVLTV` | Inner VLAN Tag Valid (`0`). |
| | `[23]` | `CDE` | Context Descriptor Error (written back by hardware). |
| | `[26]` | `TCMSSV` | **One-Step Timestamp / MSS Valid** (`1` = update channel's active TSO `MSS` from `TDES2[13:0]`). |
| | `[30]` | `CTXT` | **Context Type (`1` = Context Descriptor)**. |
| | `[31]` | `OWN` | **Ownership (`1` = owned by XGMAC DMA hardware)**. |

### 6.4 Normal Receive Read Descriptor (`RDES0..RDES3`, Written by Driver to Supply RX Buffers)

**[HW]** To post or refill an RX descriptor slot (`i`) with a host buffer at physical/IOVA address `rx_buf_phys`:

| Word / Offset | Bits | Field Name | Required Driver Value |
|---|---|---|---|
| `RDES0` (`+0x00`) | `[31:0]` | `BUF1AP_LO` | `lower_32_bits(rx_buf_phys)` (must be 8-byte aligned). |
| `RDES1` (`+0x04`) | `[31:0]` | `BUF1AP_HI` | **`0x00000010 | upper_32_bits(rx_buf_phys)`**. |
| `RDES2` (`+0x08`) | `[31:0]` | `BUF2AP_LO` | `0x00000000` (when Split Header `SPH = 0`). |
| `RDES3` (`+0x0C`) | `[29:0]` | Reserved | `0`. |
| | `[30]` | `IOC` | **Interrupt on Completion** (`1` = trigger `DMA_CH_STATUS.RI` when a packet is received into this descriptor). |
| | `[31]` | `OWN` | **Ownership** (`1` = owned by XGMAC DMA hardware). Combined `RDES3` value: **`0xC0000000`** (`OWN | IOC`). |

**[HW]** **Critically Important Refill Rule**: When the RX DMA engine writes a received frame into a descriptor, it **overwrites `RDES0`, `RDES1`, `RDES2`, and `RDES3`** with status metadata! Therefore, when recycling an RX descriptor back to hardware, the driver **must rewrite `RDES0` and `RDES1` with the buffer's AXI address (`+0x10_0000_0000`) and clear `RDES2` to `0`** before writing `0xC0000000` to `RDES3`.

### 6.5 Receive Write-Back Descriptor (`RDES0..RDES3`, Written by Hardware on Packet Arrival)

**[HW]** When a frame is received (`RDES3` bit 31 `OWN == 0` and bit 30 `CTXT == 0`):

| Word / Offset | Bits | Field Name | Description |
|---|---|---|---|
| `RDES0` (`+0x00`) | `[15:0]` | `OVT` | **Outer VLAN Tag** stripped from the frame (valid when `RDES3.RS0V == 1` and `RDES2.LT` is `4` (`0b100`, 802.1Q C-VLAN) or `5` (`0b101`, 802.1ad S-VLAN)). |
| | `[31:16]` | `IVT` | Inner VLAN Tag. |
| `RDES1` (`+0x04`) | `[31:0]` | `RSS_HASH` | RSS Hash (`0` on TC956x because `RSSEN = 0`). |
| `RDES2` (`+0x08`) | `[11:10]` | `L3FM` / `L4FM` | L3/L4 filter match. |
| | `[15]` | `MADRM` | MAC Address Match failure (`1` if failed unicast/hash filter in `RA` mode). |
| | `[18:16]` | `LT` | **Length / Type Field** (valid when `RDES3.RS2V == 1`):<br>- `0`: Length packet<br>- `1`: Type packet<br>- `3`: ARP Request<br>- **`4` (`0b100`): Type packet with 802.1Q C-VLAN tag (`0x8100`)**<br>- **`5` (`0b101`): Type packet with 802.1ad S-VLAN tag (`0x88A8`)**<br>- `6`: MAC Control packet<br>- `7`: OAM packet |
| `RDES3` (`+0x0C`) | `[13:0]` | `PL` | **Packet Length** in bytes ($0..16383$, excluding stripped 4-byte FCS when `MAC_RX_CONFIG.CST = 1`). Valid on the Last Descriptor (`LD = 1`). |
| | `[15]` | `ES` | **Error Summary** (`1` if any RX error occurred; check `ET[19:16]`). |
| | `[19:16]` | `ET` | **Error Type / Checksum Status**:<br>- When `ES == 1` (`bit 15 == 1`), `ET` encodes the receive error:<br>  - `0`: No error<br>  - `1`: Watchdog Timeout (`WT`)<br>  - `2`: Receive GMII/MII Error (`RE`)<br>  - **`3`: CRC / FCS Error (`CE`)**<br>  - `4`: Giant Packet (`GP`)<br>  - **`5`: IPv4 Header Checksum Error (`IPC_H`)**<br>  - **`6`: L4 TCP/UDP/ICMP Payload Checksum Error (`IPC_P`)**<br>  - **`7`: RX FIFO Overflow (`OVF`)**<br>  - `8`: Dribble Bit Error<br>  - `9`: Length Error<br>  - `10`: Good Runt Frame<br>  - **`11`: Runt / Damaged Frame (`RUNT`)**<br>- When `ES == 0` (`bit 15 == 0`), `ET` indicates **CSUM Bypass**: if `ET == 5` or `ET == 6` while `ES == 0`, the hardware skipped checksum verification (do not mark L4 checksum verified!). |
| | `[22:20]` | `L34T` | **Layer 3 / Layer 4 Packet Type** (valid on `LD = 1`):<br>- `0` (`0b000`): Non-IP packet (do not mark IP/L4 checksum verified!)<br>- `1` (`0b001`): IPv4 + TCP<br>- `2` (`0b010`): IPv4 + UDP<br>- `3` (`0b011`): IPv4 + ICMP<br>- `4` (`0b100`): IPv4 + IGMP<br>- `5` (`0b101`): IPv4 + Unknown/Other L4<br>- `6` (`0b110`): IPv6 + TCP / UDP / ICMPv6 |
| | `[23]` | `RS0V` | `RDES0` Status Valid (`1` = `RDES0.OVT` is valid). |
| | `[24]` | `RS1V` | `RDES1` Status Valid. |
| | `[25]` | `RS2V` | `RDES2` Status Valid (`1` = `RDES2.LT` is valid). |
| | `[26]` | `CDA` | Context Descriptor Available (`1` = next descriptor in ring is an RX Context Descriptor holding a 64-bit IEEE 1588 RX timestamp). |
| | `[27]` | `TSA` | Timestamp Available (`1` = RX timestamp captured in following RX Context Descriptor). |
| | `[28]` | `LD` | **Last Descriptor** (`1` = end of received frame; if a single-buffer driver sees `FD == 1` and `LD == 0`, the frame exceeded `RBSZ` and spanned multiple descriptors). |
| | `[29]` | `FD` | **First Descriptor** (`1` = start of received frame). |
| | `[30]` | `CTXT` | Context Type (`0` = Normal RX Write-Back Descriptor, `1` = RX Timestamp Context Descriptor). |
| | `[31]` | `OWN` | **Ownership** (`0` = owned by host CPU). |

**[HW]** **RX Hardware Checksum Verification Rule**: When `MAC_RX_CONFIG.IPC` (bit 9) is `1`, a received frame on a descriptor with `FD == 1 && LD == 1` has a **valid hardware-verified IP and TCP/UDP checksum** if and only if:
1. `RDES3` bit 15 (`ES`) is `0`, AND
2. `RDES3` bits `[22:20]` (`L34T`) are non-zero (`1`, `2`, `3`, or `6`), AND
3. `RDES3` bits `[19:16]` (`ET`) are neither `5` nor `6`.
If `ES == 1` and `ET == 5` (IP header checksum error) or `ET == 6` (TCP/UDP payload checksum error), the checksum failed.

---

## 7. Complete Port Bring-Up and Teardown Sequence

**[HW]** To initialize a port (`Port 0` at `0x40000` or `Port 1` at `0x48000`) from cold reset to packet transmission/reception:

1. **Enable Clocks and Deassert Resets (via Function 0 BAR4)**:
   - For Port 0: set `MAC0ALLCLKEN | MSIGENCEN | MAC0RXCEN | PCIECEN | MAC0TXCEN` (`0x80044280`) in `NCLKCTRL0` (`0x1004`), and clear `MAC0PONRST | MAC0PMARST | MSIGENRST | MAC0RST` in `NRSTCTRL0` (`0x1008`).
   - For Port 1: set `MAC1ALLCLKEN1 | MAC1RMCEN | MAC1RXCEN | MAC1TXCEN` (`0x8000C080`) in `NCLKCTRL1` (`0x100C`), and clear `MAC1PONRST1 | MAC1PMARST1 | MAC1RST1` (`0x00000000`) in `NRSTCTRL1` (`0x1010`).
2. **Program BAR0 TAMAP (`BAR0 + 0x0800`)**:
   - Write Entry 0 (`0x0800` = `0x00000047`, `0x0804` = `0x00000010`, `0x0808..0x0810` = `0`) and zero Entries 1–3 (`0x0820..0x0870`).
3. **Save Pre-Programmed Station MAC Address**:
   - Read `MAC_ADDR0_HIGH` (`+0x0300`) and `MAC_ADDR0_LOW` (`+0x0304`). If valid unicast, save in driver memory.
4. **Reset and Configure SerDes Mode, PMA, and XPCS**:
   - Program `NEMAC0CTL` (`0x1070`) or `NEMAC1CTL` (`0x1074`) to `0x00000114` (`SP_SEL = 4` for 2.5G) or `0x00000115` (`SP_SEL = 5` for 1G) and run the 5-step PMA initialization sequence (see Section 03).
5. **Issue XGMAC DMA Software Reset**:
   - Write `0x00000001` (`SWR`) to `DMA_MODE` (`+0x3000`). Poll `DMA_MODE` until bit 0 (`SWR`) clears to `0` (typically `< 1 ms`, timeout `100 ms`).
6. **Configure Global DMA Registers**:
   - Write `0x00001000` (`INTM = 1`, `DSPW = 0`) to `DMA_MODE` (`+0x3000`).
   - Write `0x1F1F087F` (`WR_OSR_LMT = 31`, `RD_OSR_LMT = 31`, `EAME = 1`, `BLEN256..4 = 0x7F`, `UNDEF = 1`) to `DMA_SYSBUS_MODE` (`+0x3004`).
7. **Restore Station MAC Address and Packet Filter**:
   - Write saved `MAC_ADDR0_HIGH | 0x80000000` (`+0x0300`) and `MAC_ADDR0_LOW` (`+0x0304`).
   - Program `MAC_PACKET_FILTER` (`+0x0008`), `MAC_HASH_TAB_0_31` (`+0x0010`), and `MAC_HASH_TAB_32_63` (`+0x0014`).
   - Program `MAC_VLAN_TAG` (`+0x0050`) and `MAC_VLAN_INCL` (`+0x0060`) if VLAN offload is enabled.
8. **Configure MTL Queues and Flow Control**:
   - Write `0x007F000A` (`TQS = 127` [32 KB], `TXQEN = 2`, `TSF = 1`) to `MTL_TXQ0_OP_MODE` (`+0x1100`).
   - Write `0x007F00A0` (`RQS = 127` [32 KB], `EHFC = 1`, `RSF = 1`) to `MTL_RXQ0_OP_MODE` (`+0x1140`).
   - Write `0x002C001C` (`RFD = 22`, `RFA = 14`) to `MTL_RXQ0_FLOW_CONTROL` (`+0x1150`).
   - Write `0x00000000` to `MTL_RXQ_DMA_MAP0` (`+0x1030`) and `0x00000002` to `MAC_RXQ_CTRL0` (`+0x00A0`).
   - Write `0xFFFF0002` to `MAC_Q0_TX_FLOW_CTRL` (`+0x0070`) and `0x00000001` to `MAC_RX_FLOW_CTRL` (`+0x0090`).
9. **Initialize TX and RX Descriptor Rings and DMA Channel 0 (`+0x3100`)**:
   - Zero TX descriptors (`OWN = 0`); populate all $N_{\text{RX}}$ RX descriptors with `RDES0 = lower_32_bits(buf_phys)`, `RDES1 = 0x10 | upper_32_bits(buf_phys)`, `RDES2 = 0`, `RDES3 = 0xC0000000` (`OWN | IOC`).
   - Write `0x00010000` (`PBLx8 = 1`) to `DMA_CH0_CONTROL` (`+0x3100`).
   - Write `0x00000010 | upper_32_bits(tx_ring_phys)` to `DMA_CH0_TxDESC_HADDR` (`+0x3110`) and `lower_32_bits(tx_ring_phys)` to `DMA_CH0_TxDESC_LADDR` (`+0x3114`).
   - Write `0x00000010 | upper_32_bits(rx_ring_phys)` to `DMA_CH0_RxDESC_HADDR` (`+0x3118`) and `lower_32_bits(rx_ring_phys)` to `DMA_CH0_RxDESC_LADDR` (`+0x311C`).
   - Write `N_TX - 1` (e.g., `511` / `0x000001FF`) to `DMA_CH0_TxDESC_RING_LEN` (`+0x3130`).
   - Write `0x03000000 | (N_RX - 1)` (e.g., `0x030001FF`, including mandatory `OWRQ = 3`) to `DMA_CH0_RxDESC_RING_LEN` (`+0x3134`).
   - Write `0x00000040` (`RWT = 64`) to `DMA_CH0_Rx_WATCHDOG` (`+0x313C`).
   - Clear any stale status by writing `0x0000FFFF` to `DMA_CH0_STATUS` (`+0x3160`), then write `0x0000D0C1` (`NIE | AIE | FBEE | RBUE | RIE | TIE`) to `DMA_CH0_INT_EN` (`+0x3138`).
   - Start TX DMA: write `0x00201011` (`TxPBL = 32`, `TSE = 1`, `OSP = 1`, `TXST = 1`) to `DMA_CH0_TX_CONTROL` (`+0x3104`).
   - Start RX DMA: write `0x00200001 | (rx_buf_size & 0x3FF8)` (`RxPBL = 32`, `RBSZ = rx_buf_size`, `RXST = 1`) to `DMA_CH0_RX_CONTROL` (`+0x3108`), then write `lower_32_bits(rx_ring_phys + N_RX * 16)` to `DMA_CH0_RxDESC_TAIL_LPTR` (`+0x312C`).
10. **Enable MAC Transmitter and Receiver**:
    - Write `0x00000207` (`IPC | CST | ACS | RE`, plus `JE | WD | GPSLCE` if jumbo MTU) to `MAC_RX_CONFIG` (`+0x0004`).
    - Write `(SS << 29) | 0x00010001` (`SS`, `JD = 1`, `TE = 1`) to `MAC_TX_CONFIG` (`+0x0000`).
    - Enable MSIGEN interrupts (`MSI_OUT_EN` and `MSI_MASK_CLR`).
