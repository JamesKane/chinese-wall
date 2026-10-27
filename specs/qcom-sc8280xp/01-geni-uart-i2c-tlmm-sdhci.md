# Qualcomm SC8280XP Core Peripherals Specification: GENI UART/I²C, TLMM GPIO, SDHCI-MSM v5, and DWC3 USB3

## 1. Overview and Platform Topology

**[HW]** On the Qualcomm SC8280XP System-on-Chip (Radxa Dragon Q8B), low-speed serial peripherals (UART, I²C, SPI) are implemented using Qualcomm Universal Peripheral v3 (QUPv3) Generic Interface (GENI) Serial Engines (SEs). General-purpose input/output and pin multiplexing are managed by the Top-Level Mode Multiplexer (TLMM). Removable SD/MMC storage is controlled by an SDHCI-MSM v5 host controller, and USB 3.1/2.0 connectivity is provided by Synopsys DesignWare USB3 (`DWC_usb31`) controllers wrapped in Qualcomm QScratch glue logic.

**[HW]** All memory-mapped I/O (MMIO) registers described in this specification are 32-bit little-endian registers and must be accessed with 32-bit aligned reads and writes unless explicitly stated otherwise.

| Peripheral Block | Physical Base Address | Region Size | ACPI Device Path / `_HID` | GICv3 Interrupt (GSIV) | Notes |
|---|---|---|---|---|---|
| QUP 2 Wrapper (`geniqup@8c0000`) | `0x008C0000` | `0x4000` | `\_SB.QUP2` | — | Hosts SEs at `0x00880000`–`0x0089C000` |
| GENI UART 17 (`serial@884000`) | `0x00884000` | `0x4000` | `\_SB.UARD` (`QCOM0616`, `_UID` 18) | GSIV 615 (SPI 583, level-high) | Serial console (115200 8N1) |
| QUP 1 Wrapper (`geniqup@ac0000`) | `0x00AC0000` | `0x4000` | `\_SB.QUP1` | — | Hosts SEs at `0x00A80000`–`0x00A9C000` |
| GENI I²C 12 (`i2c@a90000`) | `0x00A90000` | `0x4000` | `\_SB.IC13` (`QCOM0610`) | GSIV 604 (SPI 572, level-high) | QUP 1 SE 4; hosts RTC (`0x68`) and MAC EEPROM (`0x50`) |
| TLMM GPIO (`pinctrl@f100000`) | `0x0F100000` | `0x300000` | `\_SB.GIO0` (`QCOM060C`) | GSIV 240 (SPI 208, level-high) | 228 GPIO pins (`0`–`227`), `0x1000` stride per pin |
| SDHCI-MSM v5 (`mmc@8804000`) | `0x08804000` | `0x1000` | `\_SB.SDC2` (`QCOM2466`) | GSIV 290 (PWRCTL), SDHCI IRQ | MicroSD slot; Apps SMMU stream `0x4e0` |
| USB 0 DWC3 (`usb@a600000`) | `0x0A600000` | `0x100000` | `\_SB.URS0` (`QCOM068B`, `_CID PNP0CA1`) | GSIV 835 (`0x343`, level-high) | Dual-role USB-C port 0 (`USB0` child) |
| USB 1 DWC3 (`usb@a800000`) | `0x0A800000` | `0x100000` | `\_SB.URS1` (`QCOM068B`, `_CID PNP0CA1`) | Level-high SPI | Dual-role USB-C port 1 (`USB0` child) |
| USB 2 Multiport DWC3 (`usb@a400000`) | `0x0A400000` | `0x100000` | `\_SB.USB2` (`QCOM0826`, `_CID PNP0D10`) | Level-high SPI | Host-only 4-port controller (Type-A ports, M.2, Wi-Fi/BT) |

---

## 2. QUPv3 GENI Serial Engine Common Architecture

### 2.1 Common GENI Register Map

**[HW]** Each QUPv3 Serial Engine (SE) occupies a `0x4000` (16 KiB) MMIO aperture. The registers below are common across UART, I²C, and SPI protocols loaded into a GENI SE.

| Offset | Register Name | Access | Description |
|---|---|---|---|
| `0x0000` | `SE_GENI_INIT_CFG_REVISION` | RO | Initial configuration revision |
| `0x0004` | `SE_GENI_S_INIT_CFG_REVISION` | RO | Secondary engine initial configuration revision |
| `0x0010` | `SE_GENI_GENERAL_CFG` | RW | General configuration register |
| `0x0020` | `SE_GENI_RX_PACKING_CFG0` | RW | RX FIFO byte packing vectors 0 and 1 |
| `0x0024` | `SE_GENI_RX_PACKING_CFG1` | RW | RX FIFO byte packing vectors 2 and 3 |
| `0x0028` | `SE_GENI_TX_PACKING_CFG0` | RW | TX FIFO byte packing vectors 0 and 1 |
| `0x002C` | `SE_GENI_TX_PACKING_CFG1` | RW | TX FIFO byte packing vectors 2 and 3 |
| `0x0040` | `SE_GENI_STATUS` | RO | Engine activity status (`M_GENI_CMD_ACTIVE`, `S_GENI_CMD_ACTIVE`) |
| `0x0048` | `SE_GENI_SER_M_CLK_CFG` | RW | Primary serial clock divider and enable |
| `0x004C` | `SE_GENI_SER_S_CLK_CFG` | RW | Secondary serial clock divider and enable |
| `0x0050` | `SE_GENI_CLK_CTRL_RO` | RO | Read-only clock control status |
| `0x0060` | `SE_GENI_IF_DISABLE_RO` | RO | Interface disable status |
| `0x0068` | `SE_GENI_FW_REVISION_RO` | RO | Loaded firmware protocol and version (`[15:8]` = protocol, `[7:0]` = version) |
| `0x006C` | `SE_GENI_S_FW_REVISION_RO` | RO | Secondary firmware protocol and version |
| `0x0070` | `SE_GENI_CLK_SEL` | RW | Clock source selection (`[2:0]`) |
| `0x0074` | `SE_GENI_CFG_SEQ_START` | WO | Configuration sequence start trigger |
| `0x0078` | `SE_GENI_BYTE_GRANULARITY` | RW | Byte granularity configuration |
| `0x007C` | `SE_GENI_DMA_MODE_EN` | RW | DMA mode enable (`[0]`: 0 = FIFO mode, 1 = DMA/GSI mode) |
| `0x0080` | `SE_GENI_TX_RFR_WATERMARK_REG` | RW | Hardware Ready-For-Receiving (RFR) watermark |
| `0x0100` | `SE_GENI_CFG_REG0` | RW | Protocol-specific configuration register 0 (`0x100`–`0x2AC` range) |
| `0x0254` | `SE_GENI_RX_RFR_WATERMARK_REG` | RW | UART hardware flow-control RX RFR watermark |
| `0x0600` | `SE_GENI_M_CMD0` | RW | Primary engine command register (`[31:27]` = opcode, `[26:0]` = params) |
| `0x0604` | `SE_GENI_M_CMD_CTRL_REG` | RW | Primary command control (`[2]` = `ABORT`, `[1]` = `CANCEL`, `[0]` = `DISABLE`) |
| `0x0610` | `SE_GENI_M_IRQ_STATUS` | RO | Primary engine interrupt status |
| `0x0614` | `SE_GENI_M_IRQ_EN` | RW | Primary engine interrupt enable mask |
| `0x0618` | `SE_GENI_M_IRQ_CLEAR` | WO | Primary engine interrupt clear (write-1-to-clear) |
| `0x061C` | `SE_GENI_M_IRQ_EN_SET` | WO | Primary engine interrupt enable set (QUPv3 HW >= 2.5) |
| `0x0620` | `SE_GENI_M_IRQ_EN_CLEAR` | WO | Primary engine interrupt enable clear (QUPv3 HW >= 2.5) |
| `0x0630` | `SE_GENI_S_CMD0` | RW | Secondary engine command register (`[31:27]` = opcode, `[26:0]` = params) |
| `0x0634` | `SE_GENI_S_CMD_CTRL_REG` | RW | Secondary command control (`[2]` = `ABORT`, `[1]` = `CANCEL`, `[0]` = `DISABLE`) |
| `0x0640` | `SE_GENI_S_IRQ_STATUS` | RO | Secondary engine interrupt status |
| `0x0644` | `SE_GENI_S_IRQ_EN` | RW | Secondary engine interrupt enable mask |
| `0x0648` | `SE_GENI_S_IRQ_CLEAR` | WO | Secondary engine interrupt clear (write-1-to-clear) |
| `0x064C` | `SE_GENI_S_IRQ_EN_SET` | WO | Secondary engine interrupt enable set |
| `0x0650` | `SE_GENI_S_IRQ_EN_CLEAR` | WO | Secondary engine interrupt enable clear |
| `0x0700` | `SE_GENI_TX_FIFOn` | WO | Transmit FIFO data port (`n = 0` at `0x0700`) |
| `0x0780` | `SE_GENI_RX_FIFOn` | RO | Receive FIFO data port (`n = 0` at `0x0780`) |
| `0x0800` | `SE_GENI_TX_FIFO_STATUS` | RO | Transmit FIFO word count (`[27:0]`) |
| `0x0804` | `SE_GENI_RX_FIFO_STATUS` | RO | Receive FIFO status (word count, last-word valid bytes, parity/frame error) |
| `0x080C` | `SE_GENI_TX_WATERMARK_REG` | RW | Transmit FIFO watermark level (`[5:0]`, in FIFO words) |
| `0x0810` | `SE_GENI_RX_WATERMARK_REG` | RW | Receive FIFO watermark level (`[5:0]`, in FIFO words) |
| `0x0814` | `SE_GENI_RX_RFR_WATERMARK_REG_STD` | RW | Standard RX RFR watermark (`[5:0]`, in FIFO words) |
| `0x0900` | `SE_GENI_IOS` | RO | Serial I/O line state (`[1]` = RX_DATA_IN, `[0]` = IO2_DATA_IN) |
| `0x0908` | `SE_GENI_M_GP_LENGTH` | RW | Primary general-purpose transfer length |
| `0x0910` | `SE_GENI_S_GP_LENGTH` | RW | Secondary general-purpose transfer length |
| `0x0914` | `SE_DMA_TX_PTR_L` | RW | SE DMA TX buffer physical address low 32 bits |
| `0x0918` | `SE_DMA_TX_PTR_H` | RW | SE DMA TX buffer physical address high 32 bits |
| `0x091C` | `SE_DMA_TX_ATTR` | RW | SE DMA TX attributes |
| `0x0920` | `SE_DMA_TX_LEN` | RW | SE DMA TX transfer length in bytes |
| `0x0924` | `SE_DMA_TX_IRQ_STAT` | RO | SE DMA TX interrupt status (`[0]` = `TX_DMA_DONE`, `[2]` = `TX_RESET_DONE`) |
| `0x0928` | `SE_DMA_TX_IRQ_CLR` | WO | SE DMA TX interrupt clear |
| `0x092C` | `SE_DMA_TX_IRQ_EN` | RW | SE DMA TX interrupt enable |
| `0x0930` | `SE_DMA_TX_IRQ_EN_SET` | WO | SE DMA TX interrupt enable set |
| `0x0934` | `SE_DMA_TX_IRQ_EN_CLR` | WO | SE DMA TX interrupt enable clear |
| `0x0938` | `SE_DMA_TX_LEN_IN` | RO | SE DMA TX bytes transferred |
| `0x093C` | `SE_DMA_TX_FSM_RST` | WO | SE DMA TX state machine reset |
| `0x0940` | `SE_DMA_TX_MAX_BURST` | RW | SE DMA TX maximum burst size |
| `0x0954` | `SE_DMA_RX_PTR_L` | RW | SE DMA RX buffer physical address low 32 bits |
| `0x0958` | `SE_DMA_RX_PTR_H` | RW | SE DMA RX buffer physical address high 32 bits |
| `0x095C` | `SE_DMA_RX_ATTR` | RW | SE DMA RX attributes |
| `0x0960` | `SE_DMA_RX_LEN` | RW | SE DMA RX transfer length in bytes |
| `0x0964` | `SE_DMA_RX_IRQ_STAT` | RO | SE DMA RX interrupt status (`[0]` = `RX_DMA_DONE`, `[1]` = `RX_EOT`, `[2]` = `RX_RESET_DONE`, `[3]` = `FLUSH_DONE`, `[10:5]` = GP_ERR) |
| `0x0968` | `SE_DMA_RX_IRQ_CLR` | WO | SE DMA RX interrupt clear |
| `0x096C` | `SE_DMA_RX_IRQ_EN` | RW | SE DMA RX interrupt enable |
| `0x0970` | `SE_DMA_RX_IRQ_EN_SET` | WO | SE DMA RX interrupt enable set |
| `0x0974` | `SE_DMA_RX_IRQ_EN_CLR` | WO | SE DMA RX interrupt enable clear |
| `0x0978` | `SE_DMA_RX_LEN_IN` | RO | SE DMA RX bytes transferred |
| `0x097C` | `SE_DMA_RX_FSM_RST` | WO | SE DMA RX state machine reset |
| `0x0980` | `SE_DMA_RX_MAX_BURST` | RW | SE DMA RX maximum burst size |
| `0x0984` | `SE_DMA_RX_FLUSH` | WO | SE DMA RX flush trigger |
| `0x0E14` | `SE_IRQ_EN` | RW | Top-level SE interrupt routing mask (`[0]` = DMA_RX, `[1]` = DMA_TX, `[2]` = GENI_M, `[3]` = GENI_S) |
| `0x0E18` | `SE_HW_PARAM_0` | RO | TX FIFO parameters (`[31:24]` = width in bits, `[21:16]` = depth in words, `[15:8]` = async depth) |
| `0x0E1C` | `SE_HW_PARAM_1` | RO | RX FIFO parameters (`[31:24]` = width in bits, `[21:16]` = depth in words, `[15:8]` = async depth) |
| `0x0E20` | `SE_GSI_EVENT_EN` | RW | Mode select (`[2:0]`: `0` = FIFO mode, `1` = GSI DMA mode, `2` = SE DMA mode) |
| `0x0E30` | `SE_DMA_GENERAL_CFG` | RW | SE DMA AHB master and endianness configuration |
| `0x1000` | `QUPV3_SE_AHB_M_CFG` | RW | AHB master clock gate (`[0]` = `AHB_M_CLK_CGC_ON`) |
| `0x1004` | `SE_GENI_CGC_CTRL` | RW | Clock gating control (`0x7F` = default dynamic clock gating enabled) |
| `0x1008` | `SE_GENI_CLK_CTRL` | RW | Serial clock source (`[0]` = `SER_CLK_SEL`: 0 = DFS, 1 = root clock) |
| `0x100C` | `SE_FIFO_IF_DISABLE` | RW | Disable FIFO interface (`[0]` = `FIFO_IF_DISABLE`) |
| `0x1014` | `SE_GENI_FW_MULTILOCK_MSA_RO` | RO | Firmware multi-lock MSA status |

**[HW]** Each QUPv3 wrapper (`0x009C0000`, `0x00AC0000`, `0x008C0000`) exposes hardware version register `QUPV3_HW_VER_REG` at wrapper offset `+0x0004`:
- Bits `[31:28]`: Major version.
- Bits `[27:16]`: Minor version.
- Bits `[15:0]`: Step version.
On SC8280XP, the QUPv3 wrapper hardware version is `>= 2.5`, which means the `SE_HW_PARAM_0` (`0x0E18`) and `SE_HW_PARAM_1` (`0x0E1C`) FIFO depth fields occupy bits `[21:16]` (6-bit field, mask `0x3F0000`, shift 16) rather than bits `[23:16]` on older QUPv3 revisions.

### 2.2 Bitfield Definitions for Common Registers

**[HW]** **`SE_GENI_FW_REVISION_RO` (`0x0068`) and `SE_GENI_S_FW_REVISION_RO` (`0x006C`)**:

| Bits | Name | Description |
|---|---|---|
| `15:8` | `FW_REV_PROTOCOL` | Protocol ID loaded into the SE: `0` = None, `1` = SPI, `2` = UART, `3` = I²C, `4` = I3C, `5` = SPI Slave |
| `7:0` | `FW_REV_VERSION` | Firmware revision number |

**[HW]** **`SE_GENI_STATUS` (`0x0040`)**:

| Bits | Name | Description |
|---|---|---|
| `12` | `S_GENI_CMD_ACTIVE` | `1` when the secondary engine has an active command running |
| `8:4` | `M_GENI_CMD_STATE` | Primary engine state machine status |
| `0` | `M_GENI_CMD_ACTIVE` | `1` when the primary engine has an active command running |

**[HW]** **`SE_GENI_SER_M_CLK_CFG` (`0x0048`) and `SE_GENI_SER_S_CLK_CFG` (`0x004C`)**:

| Bits | Name | Description |
|---|---|---|
| `15:4` | `CLK_DIV_VALUE` | 12-bit clock divider factor (`1`..`4095`) applied to the SE root clock |
| `0` | `SER_CLK_EN` | `1` = enable serial clock output |

**[HW]** **`SE_GENI_M_IRQ_STATUS` (`0x0610`), `SE_GENI_M_IRQ_EN` (`0x0614`), `SE_GENI_M_IRQ_CLEAR` (`0x0618`)** (and corresponding `S_IRQ_*` registers at `0x0640`–`0x0650`):

| Bit | Name | Description |
|---|---|---|
| `31` | `SEC_IRQ` | Secondary engine interrupt summary bit (only in `M_IRQ_STATUS`: indicates `S_IRQ_STATUS` has pending bits) |
| `30` | `TX_FIFO_WR_ERR` | Write attempted to full TX FIFO |
| `29` | `TX_FIFO_RD_ERR` | Invalid read from TX FIFO |
| `28` | `TX_FIFO_WATERMARK` | TX FIFO word count dropped below `SE_GENI_TX_WATERMARK_REG` (`0x080C`). Cleared by disabling in `M_IRQ_EN`, not by `M_IRQ_CLEAR` |
| `27` | `RX_FIFO_LAST` | Last partial/complete word of an RX command is present in RX FIFO |
| `26` | `RX_FIFO_WATERMARK` | RX FIFO word count reached or exceeded `SE_GENI_RX_WATERMARK_REG` (`0x0810`). Cleared by reading RX FIFO words |
| `25` | `RX_FIFO_WR_ERR` | RX FIFO overflow |
| `24` | `RX_FIFO_RD_ERR` | Read attempted from empty RX FIFO |
| `23` | `IO_DATA_ASSERT` | Serial I/O data assertion |
| `22` | `IO_DATA_DEASSERT` | Serial I/O data deassertion |
| `21` | `M_GP_SYNC_IRQ_0` | General-purpose sync interrupt |
| `20` | `M_RX_BREAK_END` | UART RX break condition ended |
| `19` | `M_RX_BREAK_START` | UART RX break condition started |
| `11:5` | `M_GP_IRQ` | Protocol-specific general-purpose status/error bits (see UART and I²C tables below) |
| `4` | `M_CMD_ABORT` | Command abort completed |
| `3` | `M_CMD_CANCEL` | Command cancel completed |
| `2` | `M_CMD_FAILURE` | Command execution failure |
| `1` | `M_CMD_OVERRUN` | Command written while a previous command was still active |
| `0` | `M_CMD_DONE` | Command completed |

**[HW]** **`SE_GENI_RX_FIFO_STATUS` (`0x0804`)**:

| Bits | Name | Description |
|---|---|---|
| `31` | `RX_LAST` | `1` if the last word of the current transfer is in the RX FIFO |
| `30:28` | `RX_LAST_BYTE_VALID` | Number of valid bytes (`1`..`3`) in the last word when `RX_LAST` is `1`. If `0` while `RX_LAST` is `1`, all 4 bytes of the last word are valid |
| `27:26` | `RX_AFIFO_ERR` | Async FIFO parity/framing error indicator |
| `24:0` | `RX_FIFO_WC` | Number of 32-bit words currently readable in the RX FIFO |

### 2.3 FIFO Byte Packing (`SE_GENI_TX_PACKING_CFG0/1` and `SE_GENI_RX_PACKING_CFG0/1`)

**[HW]** Each 32-bit FIFO word is packed or unpacked into serial bytes according to up to four 10-bit packing vectors:
- Vector 0: `CFG0` bits `[9:0]`
- Vector 1: `CFG0` bits `[19:10]`
- Vector 2: `CFG1` bits `[9:0]`
- Vector 3: `CFG1` bits `[19:10]`

**[HW]** Each 10-bit packing vector has the following layout:

| Vector Bits | Field Name | Description |
|---|---|---|
| `9:5` | `START` | Starting bit index (`0`..`31`) within the 32-bit FIFO word for this unit |
| `4` | `DIR` | Shift direction within the unit: `0` = LSB-to-MSB (`START` is the LSB of the unit), `1` = MSB-to-LSB (`START` is the MSB of the unit) |
| `3:1` | `LEN` | Unit bit length minus 1 (`7` for an 8-bit byte) |
| `0` | `STOP` | `1` marks the final active vector in the 32-bit FIFO word; `0` continues to the next vector |

**[HW]** **Critical Protocol Difference Between UART and I²C Packing**:
- **I²C / SPI (`DIR = 1`, MSB-to-LSB)**: I²C shifts the most significant bit of each byte onto SDA first. Therefore, each 8-bit byte vector must set `DIR = 1`, `LEN = 7`, and `START` to the MSB bit index of that byte within the 32-bit FIFO word (`7`, `15`, `23`, `31`).
  - For 4 bytes per 32-bit FIFO word (`4 x 8`, MSB-first):
    - Vector 0: `START = 7`, `DIR = 1`, `LEN = 7`, `STOP = 0` -> `(7 << 5) | (1 << 4) | (7 << 1) | 0 = 0x0FE`
    - Vector 1: `START = 15`, `DIR = 1`, `LEN = 7`, `STOP = 0` -> `(15 << 5) | (1 << 4) | (7 << 1) | 0 = 0x1FE`
    - Vector 2: `START = 23`, `DIR = 1`, `LEN = 7`, `STOP = 0` -> `(23 << 5) | (1 << 4) | (7 << 1) | 0 = 0x2FE`
    - Vector 3: `START = 31`, `DIR = 1`, `LEN = 7`, `STOP = 1` -> `(31 << 5) | (1 << 4) | (7 << 1) | 1 = 0x3FF`
    - Resulting register values: `CFG0 = 0x0007F8FE`, `CFG1 = 0x000FFEFE`.
- **UART (`DIR = 0`, LSB-to-MSB)**: UART shifts the least significant bit of each character first. Each 8-bit character vector sets `DIR = 0`, `LEN = 7`, and `START` to the LSB bit index of that byte (`0`, `8`, `16`, `24`).
  - For 1 byte per 32-bit FIFO word (`1 x 8`, LSB-first, as left by UEFI on the console UART):
    - Vector 0: `START = 0`, `DIR = 0`, `LEN = 7`, `STOP = 1` -> `0x00F`
    - Resulting register values: `CFG0 = 0x0000000F`, `CFG1 = 0x00000000`.
  - For 4 bytes per 32-bit FIFO word (`4 x 8`, LSB-first):
    - Vector 0: `START = 0`, `DIR = 0`, `LEN = 7`, `STOP = 0` -> `0x00E`
    - Vector 1: `START = 8`, `DIR = 0`, `LEN = 7`, `STOP = 0` -> `0x10E`
    - Vector 2: `START = 16`, `DIR = 0`, `LEN = 7`, `STOP = 0` -> `0x20E`
    - Vector 3: `START = 24`, `DIR = 0`, `LEN = 7`, `STOP = 1` -> `0x30F`
    - Resulting register values: `CFG0 = 0x0004380E`, `CFG1 = 0x000C3E0E`.
- **[HW]** Writing `0x0F` (LSB-first) to an I²C SE causes the hardware to bit-reverse every transmitted and received byte on the bus. In addition, after writing `SE_GENI_TX_PACKING_CFG0/1` and `SE_GENI_RX_PACKING_CFG0/1`, if `1 x 8` packing is used, bit `0` (`BYTE_GRANULARITY`) of `SE_GENI_BYTE_GRANULARITY` (`0x0078`) must be set to `1`; for `4 x 8` packing it must be `0`.

---

## 3. QUPv3 GENI UART (`0x00884000`, `\_SB.UARD`)

### 3.1 ACPI Discovery and SPCR Quirks

**[ABI]** On the Radxa Dragon Q8B, the serial console is QUP 2 SE 1 (`serial@884000`) at physical base `0x00884000`.
1. **SPCR Table Quirks**:
   - The ACPI `SPCR` table reports Interface Type `0x13`.
   - Its Generic Address Structure (GAS) specifies Address `0x0000000000884000`, Bit Width `0x20` (32), Bit Offset `0x00`, and Encoded Access Width `0x20` (`32` instead of the ACPI-standard `0x03` for 32-bit access). An OS early console parser must tolerate `AccessWidth == 0x20` on `0x13` SPCR entries and treat the register space as 32-bit MMIO.
   - The `SPCR` table reports Global System Interrupt `0x00000182` (386), which is **incorrect**.
2. **DSDT Device Node (`\_SB.UARD`)**:
   - `_HID`: `"QCOM0616"`, `_UID`: `18`.
   - MMIO resource: `0x00884000` length `0x4000`.
   - Interrupt resource: **GSIV 615** (GIC SPI 583, level-triggered, active-high). Interrupt-driven UART operation must use GSIV 615 from `\_SB.UARD`, not 386 from `SPCR`.

### 3.2 UART-Specific Registers and Opcodes

**[HW]** When protocol `2` (UART) is loaded (`SE_GENI_FW_REVISION_RO[15:8] == 2`), the following protocol-specific registers and command opcodes apply:

| Offset | Register Name | Access | Description |
|---|---|---|---|
| `0x0254` | `SE_GENI_RX_RFR_WATERMARK_REG` | RW | Hardware flow control RFR threshold |
| `0x025C` | `SE_UART_TX_TRANS_CFG` | RW | `[0]` = `UART_CTS_MASK` (`1` = ignore CTS input during TX) |
| `0x0260` | `SE_UART_TX_WORD_LEN` | RW | Bits per TX character (`8` for 8N1) |
| `0x0264` | `SE_UART_TX_STOP_BIT_LEN` | RW | Stop bit length: `0` = 1 stop bit (`TX_STOP_BIT_LEN_1`), `1` = 1.5 stop bits, `2` = 2 stop bits (`TX_STOP_BIT_LEN_2`) |
| `0x0268` | `SE_UART_TX_PARITY_CFG` | RW | `[0]` = `TX_PAR_CALC_EN`, `[2:1]` = parity mode (`0` = even, `1` = odd, `2` = mark, `3` = space) |
| `0x0270` | `SE_UART_TX_TRANS_LEN` | RW | Number of characters to transmit in the current primary command (`[23:0]`) |
| `0x0280` | `SE_UART_RX_TRANS_CFG` | RW | `[3]` = `UART_RX_PAR_EN` (`1` = enable RX parity check), `[2]` = `UART_RX_INSERT_STATUS_BIT` |
| `0x0284` | `SE_UART_RX_WORD_LEN` | RW | Bits per RX character (`8` for 8N1) |
| `0x0288` | `SE_UART_RX_STALE_CNT` | RW | Stale timeout in bit periods (`[23:0]`; e.g., `16 * 10 = 160` bit periods) |
| `0x028C` | `SE_UART_RX_PARITY_CFG` | RW | `[0]` = `RX_PAR_CALC_EN`, `[2:1]` = parity mode |
| `0x0290` | `SE_UART_LOOPBACK_CFG` | RW | `[1:0]`: `0` = normal, `1` = internal TX-to-RX loopback, `2` = CTS-to-RFR loopback, `3` = both |
| `0x02A4` | `SE_UART_MANUAL_RFR` | RW | `[31]` = `UART_MANUAL_RFR_EN`, `[1]` = `UART_RFR_NOT_READY`, `[0]` = `UART_RFR_READY` |

**[HW]** **Primary Engine (`M_CMD0`, `0x0600`) UART Opcodes** (placed in bits `[31:27]`):
- `1` (`UART_START_TX`, `0x08000000`): Start transmitting `SE_UART_TX_TRANS_LEN` characters from the TX FIFO.
- `4` (`UART_START_BREAK`, `0x20000000`): Assert TX break condition.
- `5` (`UART_STOP_BREAK`, `0x28000000`): Deassert TX break condition.

**[HW]** **Secondary Engine (`S_CMD0`, `0x0630`) UART Opcodes** (placed in bits `[31:27]`):
- `1` (`UART_START_READ`, `0x08000000`): Start continuous reception into the RX FIFO.

**[HW]** **Secondary Engine (`S_IRQ_STATUS`, `0x0640`) UART GP_IRQ Bits**:
- Bit `5` (`S_GP_IRQ_0`): `UART_PAR_FRAME_ERR_IRQ` (parity or framing error).
- Bit `6` (`S_GP_IRQ_1`): `UART_BREAK_IRQ` (RX break detected).
- Bit `7` (`S_GP_IRQ_2`): `UART_OVERSAMPLING_ERR` (oversampling error).
- Bit `8` (`S_GP_IRQ_3`): `UART_RX_STALE_IRQ` (RX stale timer expired with data in RX FIFO).

### 3.3 Handover from UEFI and Operating Sequences

**[HW]** UEFI initializes `serial@884000` at 115200 8N1 before OS handoff with the following live state:
- `SE_HW_PARAM_0` (`0x0E18`): TX FIFO depth is 16 words.
- `SE_HW_PARAM_1` (`0x0E1C`): RX FIFO depth is 16 words.
- `SE_GENI_TX_PACKING_CFG0` (`0x0028`) and `SE_GENI_RX_PACKING_CFG0` (`0x0020`): `0x0000000F` (`1 x 8` packing, 1 byte per 32-bit FIFO word).
- `SE_UART_TX_TRANS_CFG` (`0x025C`): `0x2` (`UART_CTS_MASK` set, hardware flow control ignored).
- Secondary engine (`S_CMD0`) is already running `UART_START_READ` (`0x08000000`).

**[POLICY]** If the OS driver preserves UEFI's baud rate, clock configuration, and `1 x 8` byte packing (`0x0F`), it does not need to reprogram `SE_GENI_SER_M_CLK_CFG` or `SE_GENI_SER_S_CLK_CFG`. If the OS driver reconfigures packing to `4 x 8` (`CFG0 = 0x4380E`, `CFG1 = 0xC3E0E`), it must first cancel any active primary/secondary commands.

**[HW]** **Polled / Interrupt Transmit Sequence**:
1. Wait until `SE_GENI_STATUS` (`0x0040`) bit `0` (`M_GENI_CMD_ACTIVE`) is `0` and `SE_GENI_TX_FIFO_STATUS` (`0x0800`) bits `[27:0]` are `0`.
2. Write the number of bytes to transmit (`N`) to `SE_UART_TX_TRANS_LEN` (`0x0270`).
3. Write `0x08000000` (`UART_START_TX`) to `SE_GENI_M_CMD0` (`0x0600`).
4. Write the character data to `SE_GENI_TX_FIFOn` (`0x0700`):
   - If `1 x 8` packing (`0x0F`) is active, write 1 byte per 32-bit word into `0x0700`, checking `SE_GENI_TX_FIFO_STATUS` (`0x0800`) to ensure the word count stays below the TX FIFO depth (16 words).
   - If `4 x 8` packing (`0x4380E` / `0xC3E0E`) is active, pack up to 4 bytes in little-endian order per 32-bit word written to `0x0700`.
5. For interrupt-driven TX when `N` exceeds the FIFO depth, set `SE_GENI_TX_WATERMARK_REG` (`0x080C`) to a threshold (e.g., `2` or `4`) and enable `TX_FIFO_WATERMARK` (bit `28`) and `M_CMD_DONE` (bit `0`) in `SE_GENI_M_IRQ_EN` (`0x0614`). When all bytes have been written to the FIFO, clear `TX_FIFO_WATERMARK` (bit `28`) in `SE_GENI_M_IRQ_EN` (`0x0614`), wait for `M_CMD_DONE` (bit `0`), and clear `M_CMD_DONE` by writing `1` to `SE_GENI_M_IRQ_CLEAR` (`0x0618`).

**[HW]** **Receive Sequence**:
1. To start reception (if `SE_GENI_STATUS` bit `12` `S_GENI_CMD_ACTIVE` is `0`), write `RX_FIFO_DEPTH - 2` (`14`) to `SE_GENI_RX_WATERMARK_REG` (`0x0810`), write `0x08000000` (`UART_START_READ`) to `SE_GENI_S_CMD0` (`0x0630`), and enable `RX_FIFO_WATERMARK` (bit `26`), `RX_FIFO_LAST` (bit `27`), and `SEC_IRQ` (bit `31`) in `SE_GENI_M_IRQ_EN` (`0x0614`) as well as `RX_FIFO_WATERMARK` (bit `26`) and `RX_FIFO_LAST` (bit `27`) in `SE_GENI_S_IRQ_EN` (`0x0644`).
2. When an RX interrupt arrives:
   - Read `s_stat = SE_GENI_S_IRQ_STATUS` (`0x0640`) and `m_stat = SE_GENI_M_IRQ_STATUS` (`0x0610`).
   - Clear non-watermark bits by writing `s_stat` to `SE_GENI_S_IRQ_CLEAR` (`0x0648`) and `m_stat` to `SE_GENI_M_IRQ_CLEAR` (`0x0618`).
   - If `s_stat` has `RX_FIFO_WATERMARK` (bit `26`) or `RX_FIFO_LAST` (bit `27`) set:
     - Read `rx_fifo_stat = SE_GENI_RX_FIFO_STATUS` (`0x0804`).
     - Extract `wc = rx_fifo_stat & 0x01FFFFFF` (number of 32-bit words in FIFO).
     - If `1 x 8` packing (`0x0F`) is configured, each of the `wc` reads from `SE_GENI_RX_FIFOn` (`0x0780`) yields 1 character in bits `[7:0]`.
     - If `4 x 8` packing is configured, the first `wc - 1` reads yield 4 characters each (little-endian). For the last word, if `rx_fifo_stat` bit `31` (`RX_LAST`) is `1` and bits `[30:28]` (`RX_LAST_BYTE_VALID`) are non-zero (`1`..`3`), extract only that many bytes from the final word; otherwise extract all 4 bytes.

**[HW]** **Command Cancel and Abort Sequence (Primary or Secondary)**:
1. To cancel an active command on the primary engine, write `0x2` (`M_GENI_CMD_CANCEL`) to `SE_GENI_M_CMD_CTRL_REG` (`0x0604`) and poll `SE_GENI_M_IRQ_STATUS` (`0x0610`) until bit `3` (`M_CMD_CANCEL`) is `1`. Write `0x8` to `SE_GENI_M_IRQ_CLEAR` (`0x0618`).
2. If `M_CMD_CANCEL` does not assert within the timeout, write `0x4` (`M_GENI_CMD_ABORT`) to `SE_GENI_M_CMD_CTRL_REG` (`0x0604`), poll `SE_GENI_M_IRQ_STATUS` for bit `4` (`M_CMD_ABORT`), and clear it via `SE_GENI_M_IRQ_CLEAR`.
3. The identical procedure applies to the secondary engine using `SE_GENI_S_CMD_CTRL_REG` (`0x0634`), `SE_GENI_S_IRQ_STATUS` (`0x0640`), and `SE_GENI_S_IRQ_CLEAR` (`0x0648`).

---

## 4. QUPv3 GENI I²C (`0x00A90000`, `\_SB.IC13`)

### 4.1 Platform Configuration and UEFI Handover State

**[HW]** On the Radxa Dragon Q8B, QUP 1 SE 4 (`i2c@a90000`, ACPI `\_SB.IC13`, `_HID QCOM0610`, GSIV 604) is pre-initialized by UEFI firmware and left powered and clocked at OS entry:
- GCC clock vote register `0x00152008` has bit `12` (`qup1_wrap1_s4`) set.
- `SE_GENI_FW_REVISION_RO` (`0x0068`) reads `0x00000303`: protocol `3` (I²C) in bits `[15:8]`, version `3` in bits `[7:0]`.
- `SE_GSI_EVENT_EN` (`0x0E20`) reads `0x00000000` (FIFO mode; GSI DMA is not claimed).
- `SE_HW_PARAM_0` (`0x0E18`) and `SE_HW_PARAM_1` (`0x0E1C`): TX and RX FIFO depths are 16 words (`(val >> 16) & 0x3F`).
- Clock configuration: 19.2 MHz SE root clock with `SE_GENI_SER_M_CLK_CFG` (`0x0048`) divider `= 2`, enabled (`0x00000021`), and `SE_I2C_SCL_COUNTERS` (`0x0278`) `= 0x00503018` (400 kHz Fast Mode).

**[HW]** **On-Board Devices on `i2c@a90000`**:
1. **ST M41T11 Real-Time Clock** at 7-bit I²C address `0x68`:
   - Register-compatible with Dallas/Maxim DS1307 BCD timekeeping registers:
     - `0x00`: Seconds (`[7]` = `STOP` oscillator halt bit, `[6:4]` = 10-seconds, `[3:0]` = seconds `00`–`59`).
     - `0x01`: Minutes (`[6:4]` = 10-minutes, `[3:0]` = minutes `00`–`59`).
     - `0x02`: Hours (`[7:6]` = Century enable/value, `[5:4]` = 10-hours, `[3:0]` = hours `00`–`23`).
     - `0x03`: Day of week (`[2:0]` = `1`–`7`).
     - `0x04`: Day of month (`[5:4]` = 10-days, `[3:0]` = day `01`–`31`).
     - `0x05`: Month (`[4]` = 10-months, `[3:0]` = month `01`–`12`).
     - `0x06`: Year (`[7:4]` = 10-years, `[3:0]` = year `00`–`99`).
     - `0x07`: Control/calibration register (`[7]` = `OUT`, `[6]` = `FT`, `[5]` = `S`, `[4:0]` = calibration).
2. **24C02-Compatible NVMEM EEPROM** at 7-bit I²C address `0x50`:
   - Uses 1-byte internal byte offset addressing (`0x00`–`0xFF`).
   - Offset `0x9E` (6 bytes): Ethernet Port 0 MAC address (`88:12:4e:00:02:00` on reference board).
   - Offset `0xA4` (6 bytes): Ethernet Port 1 MAC address (`88:12:4e:00:02:01` on reference board).

### 4.2 I²C-Specific Registers, Opcodes, and Bitfields

**[HW]** The following registers and fields are specific to the I²C protocol (`FW_REV_PROTOCOL == 3`):

| Offset | Register Name | Access | Description |
|---|---|---|---|
| `0x026C` | `SE_I2C_TX_TRANS_LEN` | RW | Number of bytes to transmit in an `I2C_WRITE` command (`[23:0]`) |
| `0x0270` | `SE_I2C_RX_TRANS_LEN` | RW | Number of bytes to receive in an `I2C_READ` or `I2C_WRITE_READ` command (`[23:0]`) |
| `0x0274` | `SE_I2C_DELAY_COUNTER` | RW | Inter-frame delay counter |
| `0x0278` | `SE_I2C_SCL_COUNTERS` | RW | SCL timing counters: `[29:20]` = `HIGH_TIME`, `[19:10]` = `LOW_TIME`, `[9:0]` = `CYCLE_TIME` |

**[HW]** **I²C SCL Timing Table (at 19.2 MHz SE Source Clock)**:

| Bus Speed | `CLK_DIV` (`SE_GENI_SER_M_CLK_CFG[15:4]`) | `HIGH_TIME` (`[29:20]`) | `LOW_TIME` (`[19:10]`) | `CYCLE_TIME` (`[9:0]`) |
|---|---|---|---|---|
| 100 kHz | `4` | `14` | `18` | `40` |
| 400 kHz | `2` | `5` | `11` | `22` (or UEFI `0x00503018`) |
| 1000 kHz | `1` | `3` | `9` | `18` |

**[HW]** **Primary Engine (`M_CMD0`, `0x0600`) Layout for I²C**:

| Bits | Field Name | Description |
|---|---|---|
| `31:27` | `OPCODE` | `1` = `I2C_WRITE`, `2` = `I2C_READ`, `3` = `I2C_WRITE_READ`, `4` = `I2C_ADDR_ONLY`, `6` = `I2C_BUS_CLEAR`, `7` = `I2C_STOP_ON_BUS` |
| `15:9` | `SLV_ADDR` | 7-bit I²C target address |
| `8` | `PRE_CMD_DELAY` | Insert `SE_I2C_DELAY_COUNTER` delay before command |
| `6` | `TIMESTAMP_BEFORE` | Capture timestamp before command |
| `4` | `STOP_STRETCH` | **Hold bus at end of transfer without issuing a STOP condition** (`1` = stretch/hold SCL low so the next command issues a Repeated START; `0` = issue STOP at end of transfer) |
| `2` | `TIMESTAMP_AFTER` | Capture timestamp after command |
| `1` | `POST_COMMAND_DELAY` | Insert delay after command |
| `0` | `READ_FINISH_WITH_ACK` | Finish read with ACK instead of NACK |

**[HW]** **Primary Engine (`M_IRQ_STATUS`, `0x0610`) I²C Error Bits (`M_GP_IRQ`)**:

| Bit | Name | Description |
|---|---|---|
| `5` | `NACK` (`M_GP_IRQ_0`) | Target did not acknowledge address or data byte |
| `6` | `BUS_PROTO` (`M_GP_IRQ_1`) | Bus protocol error (unexpected START/STOP) |
| `7` | `ARB_LOST` (`M_GP_IRQ_2`) | Arbitration lost on multi-master bus |

### 4.3 Step-by-Step I²C Initialization and Transfer Sequence

**[HW]** **Initialization Sequence**:
1. Read `SE_GENI_FW_REVISION_RO` (`0x0068`) and verify bits `[15:8]` equal `3` (I²C).
2. Read `SE_HW_PARAM_0` (`0x0E18`) and extract TX FIFO depth `= (val >> 16) & 0x3F` (`16` words on SC8280XP).
3. Program MSB-first `4 x 8` byte packing:
   - Write `0x0007F8FE` to `SE_GENI_TX_PACKING_CFG0` (`0x0028`) and `SE_GENI_RX_PACKING_CFG0` (`0x0020`).
   - Write `0x000FFEFE` to `SE_GENI_TX_PACKING_CFG1` (`0x002C`) and `SE_GENI_RX_PACKING_CFG1` (`0x0024`).
   - Write `0x00000000` to `SE_GENI_BYTE_GRANULARITY` (`0x0078`).
4. Write `RX_FIFO_DEPTH - 2` (`14`) to `SE_GENI_RX_WATERMARK_REG` (`0x0810`), and write `RX_FIFO_DEPTH - 3` (`13`) to `SE_GENI_RX_RFR_WATERMARK_REG_STD` (`0x0814`).
5. Ensure FIFO mode is selected by writing `0` to `SE_GENI_DMA_MODE_EN` (`0x007C`) and `0` to `SE_GSI_EVENT_EN` (`0x0E20`).
6. Enable dynamic clock gating by writing `0x7F` to `SE_GENI_CGC_CTRL` (`0x1004`), and route primary/DMA interrupts to the SE IRQ line by writing `0x0F` to `SE_IRQ_EN` (`0x0E14`).

**[HW]** **Multi-Message I²C Transaction Sequence (FIFO Mode)**:
For a sequence of `M` messages (`m = 0 .. M-1`), where each message has 7-bit address `addr`, length `len`, and direction (`READ` or `WRITE`):
1. Clear any stale status bits by writing `0xFFFFFFFF` to `SE_GENI_M_IRQ_CLEAR` (`0x0618`).
2. Compute the command parameters `params = (addr & 0x7F) << 9`.
3. If `m < M - 1` (there is a subsequent message in the transaction and no explicit STOP flag is requested on message `m`), set bit `4` (`STOP_STRETCH`, `0x10`) in `params`: `params |= (1 << 4)`. This causes the GENI engine to hold the bus at the end of message `m` and generate a Repeated START at the beginning of message `m + 1`.
4. **If message `m` is a READ**:
   - Write `len` to `SE_I2C_RX_TRANS_LEN` (`0x0270`).
   - Enable `RX_FIFO_WATERMARK` (bit `26`), `RX_FIFO_LAST` (bit `27`), `M_CMD_DONE` (bit `0`), `M_CMD_CANCEL` (bit `3`), `M_CMD_ABORT` (bit `4`), and `NACK | BUS_PROTO | ARB_LOST` (bits `7:5`) in `SE_GENI_M_IRQ_EN` (`0x0614`).
   - Write `(2 << 27) | params` (`I2C_READ`) to `SE_GENI_M_CMD0` (`0x0600`).
   - As `RX_FIFO_WATERMARK` or `RX_FIFO_LAST` asserts in `SE_GENI_M_IRQ_STATUS` (`0x0610`), read `SE_GENI_RX_FIFO_STATUS` (`0x0804`), get `wc = status & 0x01FFFFFF`, and read `wc` 32-bit words from `SE_GENI_RX_FIFOn` (`0x0780`), unpacking up to 4 bytes per word in little-endian byte order (`byte 0 = word[7:0]`, `byte 1 = word[15:8]`, `byte 2 = word[23:16]`, `byte 3 = word[31:24]`).
5. **If message `m` is a WRITE**:
   - Write `len` to `SE_I2C_TX_TRANS_LEN` (`0x026C`).
   - Write `1` to `SE_GENI_TX_WATERMARK_REG` (`0x080C`).
   - Enable `TX_FIFO_WATERMARK` (bit `28`), `M_CMD_DONE` (bit `0`), `M_CMD_CANCEL` (bit `3`), `M_CMD_ABORT` (bit `4`), and `NACK | BUS_PROTO | ARB_LOST` (bits `7:5`) in `SE_GENI_M_IRQ_EN` (`0x0614`).
   - Write `(1 << 27) | params` (`I2C_WRITE`) to `SE_GENI_M_CMD0` (`0x0600`).
   - Whenever `TX_FIFO_WATERMARK` asserts, pack up to 4 bytes in little-endian byte order per 32-bit word and write them to `SE_GENI_TX_FIFOn` (`0x0700`) up to the available TX FIFO capacity (`16 - TX_FIFO_STATUS`). Once all `len` bytes have been written to the TX FIFO, clear `TX_FIFO_WATERMARK` (bit `28`) in `SE_GENI_M_IRQ_EN` (`0x0614`) and write `0` to `SE_GENI_TX_WATERMARK_REG` (`0x080C`).
6. Wait for `M_CMD_DONE` (bit `0`) in `SE_GENI_M_IRQ_STATUS` (`0x0610`).
7. **Error Recovery**: If `NACK` (bit `5`), `BUS_PROTO` (bit `6`), `ARB_LOST` (bit `7`), or a timeout (e.g., 1000 ms **[POLICY]**) occurs:
   - Cancel and/or abort the command via `SE_GENI_M_CMD_CTRL_REG` (`0x0604`) as described in Section 3.3.
   - If the error was `ARB_LOST` or a timeout/bus hang, issue an `I2C_BUS_CLEAR` command by writing `(6 << 27)` (`0x30000000`) to `SE_GENI_M_CMD0` (`0x0600`) and waiting for `M_CMD_DONE`.

---

## 5. Top-Level Mode Multiplexer (TLMM GPIO, `0x0F100000`, `\_SB.GIO0`)

### 5.1 Overview and Pin Map

**[HW]** The SC8280XP TLMM controller lies at physical base `0x0F100000` (size `0x300000`, ACPI `\_SB.GIO0`, `_HID QCOM060C`) with a summary interrupt on **GSIV 240** (GIC SPI 208, level-high).
- Total GPIO pins: **228** (`gpio0` through `gpio227`).
- Each GPIO pin `N` (`0 <= N <= 227`) has its own `0x1000` (4 KiB) register block at offset `0x1000 * N` from `0x0F100000`.
- **[HW]** **TrustZone-Reserved Pins (Access Fault Hazard)**: The following GPIO pins are owned by TrustZone / XPU-protected and **must never be read or written** by the non-secure OS:
  - **Pins `74`–`79`** (`0x0F14A000`–`0x0F14F000`)
  - **Pins `83`–`86`** (`0x0F153000`–`0x0F156000`)
  - **Pins `125`–`126`** (`0x0F17D000`–`0x0F17E000`)
  - **Pins `128`–`129`** (`0x0F180000`–`0x0F181000`)
  - Valid accessible pin ranges are `0`–`73`, `80`–`82`, `87`–`124`, `127`, and `130`–`227`.

### 5.2 Per-Pin Register Layout (`Base + 0x1000 * N`)

**[HW]** Every accessible GPIO pin `N` exposes four 32-bit registers:

| Offset from Pin Base | Register Name | Access | Description |
|---|---|---|---|
| `+0x0000` | `TLMM_GPIO_CFGn` | RW | Pin mux function, drive strength, pull configuration, output enable, and Ego/I2C pull |
| `+0x0004` | `TLMM_GPIO_IN_OUTn` | RW | Input state (`[0]`) and output value (`[1]`) |
| `+0x0008` | `TLMM_GPIO_INTR_CFGn` | RW | Interrupt enable, polarity/edge detection, raw status enable, and target processor routing |
| `+0x000C` | `TLMM_GPIO_INTR_STATUSn` | RW | Interrupt pending status (`[0]`); write `0` to clear |

**[HW]** **`TLMM_GPIO_CFGn` (`+0x0000`) Bitfields**:

| Bits | Name | Description |
|---|---|---|
| `13` | `EGPIO_ENABLE` | `1` = eGPIO mode enabled (controlled by remote processor) |
| `12` | `EGPIO_PRESENT` | RO: `1` = pin supports eGPIO |
| `11` | `I2C_PULL` | Optional strong internal pull-up for I²C lines |
| `9` | `GPIO_OE` | Output enable (`0` = input, `1` = output) |
| `8:6` | `DRV_STRENGTH` | Drive strength in mA: `val = (mA / 2) - 1` (`0` = 2 mA, `1` = 4 mA, `2` = 6 mA, `3` = 8 mA, `4` = 10 mA, `5` = 12 mA, `6` = 14 mA, `7` = 16 mA) |
| `5:2` | `FUNC_SEL` | Pinmux function select (`0` = general-purpose GPIO; `1`..`9` = alternate peripheral functions) |
| `1:0` | `GPIO_PULL` | Internal pull resistor: `0` = No Pull, `1` = Pull Down, `2` = Keeper, `3` = Pull Up |

**[HW]** **`TLMM_GPIO_IN_OUTn` (`+0x0004`) Bitfields**:

| Bits | Name | Description |
|---|---|---|
| `1` | `GPIO_OUT` | Output logic level driven when `GPIO_OE == 1` (`0` = low, `1` = high) |
| `0` | `GPIO_IN` | RO: Sampled input logic level on the pin (`0` = low, `1` = high) |

**[HW]** **`TLMM_GPIO_INTR_CFGn` (`+0x0008`) Bitfields**:

| Bits | Name | Description |
|---|---|---|
| `8` | `WAKEUP_ENABLE` | `1` = enable wakeup to Power Domain Controller (PDC) |
| `7:5` | `INTR_TARGET_PROC` | Target processor routing for this pin's interrupt. **Note**: On SC8280XP, the Apps processor (APSS/KPSS) target value is `3` (`011b`) or `4` (`100b` on QCOMv4 TLMM designs; UEFI leaves `4` on active pins, while value `7` routes to "none" and suppresses the interrupt). A driver should preserve the firmware-programmed `INTR_TARGET_PROC` field if non-zero (not `7`), or program `3` / `4` as required by the platform |
| `4` | `INTR_RAW_STATUS_EN` | `1` = enable raw status latching into `TLMM_GPIO_INTR_STATUSn` (must be `1` for interrupt detection) |
| `3:2` | `INTR_DECT_CTL` | Edge/level detection mode: `0` = Level-triggered, `1` = Positive-edge (rising), `2` = Negative-edge (falling), `3` = Dual-edge (both edges) |
| `1` | `INTR_POL_CTL` | Polarity control: for level-triggered (`INTR_DECT_CTL == 0`), `0` = active-low, `1` = active-high. For single-edge (`INTR_DECT_CTL == 1` or `2`), must be set to `1`. For dual-edge (`INTR_DECT_CTL == 3`), set to `0` |
| `0` | `INTR_ENABLE` | `1` = unmask delivery of `TLMM_GPIO_INTR_STATUSn` to the summary IRQ (GSIV 240) |

**[HW]** **`TLMM_GPIO_INTR_STATUSn` (`+0x000C`) Bitfields**:

| Bits | Name | Description |
|---|---|---|
| `0` | `INTR_STATUS` | `1` = interrupt pending. **Write `0` to clear** (note: this register is write-0-to-clear, NOT write-1-to-clear!) |

### 5.3 Board-Specific GPIO Assignments and ACPI Pitfalls

**[HW]** Key GPIO assignments on the Radxa Dragon Q8B:

| GPIO Pin | Offset | Function / Board Role | Active Level / Configuration |
|---|---|---|---|
| `2` | `0x002000` | ACPI `_AEI` event pin | Listed in `\_SB.GIO0._AEI` with `0x2c0` |
| `4` | `0x004000` | User status LED (`user-led`) | Active-high output |
| `47` (`0x2f`) | `0x02F000` | USB1 SBU mux select (`select-gpios`) | Active-high output |
| `48` (`0x30`) | `0x030000` | USB1 SBU mux enable (`enable-gpios`) | Active-low output |
| `106` (`0x6a`) | `0x06A000` | WCD9385 audio codec reset (`reset-gpios`) | Active-low output |
| `131` (`0x83`) | `0x083000` | MicroSD (`SDC2`) card detect (`cd-gpios`) | Active-low input (`0` = card inserted, `1` = card removed) |
| `158` (`0x9e`) | `0x09E000` | NVMe2 3.3V regulator enable (`vcc_nvme2_3v3`) | Active-high output |

**[ABI]** **ACPI `\_SB.GIO0` Pitfalls**:
1. `\_SB.GIO0` contains a `PSUB` object evaluating to `"QRD08280"` and an `_AEI` resource template referencing pin 2 (`0x2c0`).
2. **Do NOT evaluate or program `_AEI` pins during GPIO controller bus attach** unless a full ACPI Generic Event Device (GED) handler is registered and ready to service them; enabling pin 2's interrupt at attach without a handler causes an immediate interrupt storm or hang.

---

## 6. SDHCI-MSM v5 (`0x08804000`, `\_SB.SDC2`)

### 6.1 Overview and Hardware Capabilities

**[HW]** The removable MicroSD card slot is driven by the SDHCI-MSM v5 controller at physical address `0x08804000` (ACPI `\_SB.SDC2`, `_HID QCOM2466`).
- **Registers**: Standard SDHCI v4.20 register set at `0x000`–`0x0FF`, followed by Qualcomm v5 vendor-specific registers starting at `0x200` (in the unified single-aperture v5 layout where `MCI` registers are merged into the HC aperture).
- **Capabilities (`SDHCI_CAPABILITIES` at `0x040`)**: Reads `0x3629C8B2`:
  - `SDHCI_CAN_DO_ADMA2` (bit 19) = `1`.
  - `SDHCI_CAN_DO_SDMA` (bit 22) = `0` (**SDMA is NOT supported**; DMA transfers must use ADMA2 or 64-bit ADMA2 descriptors).
  - `SDHCI_CAN_64BIT` (bit 28) = `1`, `SDHCI_CAN_64BIT_V4` (bit 27) = `1`.
- **DMA & IOMMU**: Routes through the Apps SMMU (`0x15000000`) with Stream ID `0x4e0` (mask `0x0`). UEFI pre-programs SMR 2 (`0x15000808 = 0x800004e0`) and S2CR 2 (`0x15000c08 = 0x00010000`, `TYPE=BYPASS`).
- **Clocks & Power**: UEFI leaves the `SDC2` clocks enabled and card power on at OS handoff.

### 6.2 SDHCI-MSM v5 Vendor-Specific Register Map

**[HW]** On SDHCI-MSM v5 (single-aperture layout without a separate MCI block), all vendor-specific registers reside within the main SDHCI aperture at `0x08804000`:

| Offset | Register Name | Access | Description |
|---|---|---|---|
| `0x0200` | `CORE_DLL_CONFIG` | RW | Delay-Locked Loop (DLL) configuration |
| `0x0204` | `CORE_DLL_STATUS` | RO | DLL lock and CDR status |
| `0x020C` | `CORE_VENDOR_SPEC` | RW | Qualcomm vendor-specific control (`POR_VAL = 0xA1C`) |
| `0x0210` | `CORE_VENDOR_SPEC_ADMA_ERR_ADDR0` | RO | ADMA error address low |
| `0x0214` | `CORE_VENDOR_SPEC_ADMA_ERR_ADDR1` | RO | ADMA error address high |
| `0x0218` | `CORE_VENDOR_SPEC_FUNC2` | RW | Vendor function 2 control |
| `0x021C` | `CORE_VENDOR_SPEC_CAPABILITIES0` | RW | Vendor capabilities override |
| `0x0240` | `CORE_PWRCTL_STATUS` | RO | Power/bus control request status from controller (`[0]` = `BUS_OFF`, `[1]` = `BUS_ON`, `[2]` = `IO_LOW`, `[3]` = `IO_HIGH`) |
| `0x0244` | `CORE_PWRCTL_MASK` | RW | Interrupt enable mask for `CORE_PWRCTL_STATUS` (`0x0F` enables all 4 bits) |
| `0x0248` | `CORE_PWRCTL_CLEAR` | WO | Write-1-to-clear bits in `CORE_PWRCTL_STATUS` |
| `0x024C` | `CORE_PWRCTL_CTL` | RW | Handshake response to controller (`[0]` = `BUS_PWR_SUCCESS`, `[1]` = `BUS_PWR_FAIL`, `[2]` = `IO_SIG_SWITCH_SUCCESS`, `[3]` = `IO_SIG_SWITCH_FAIL`) |
| `0x0250` | `CORE_DDR_200_CFG` | RW | DDR200 mode configuration |
| `0x0254` | `CORE_DLL_CONFIG_2` | RW | Secondary DLL configuration (`[18]` = `DLL_CLOCK_DISABLE`, `[21]` = `DDR_CAL_EN`, `[23:10]` = `MCLK_FREQ_CALC`) |
| `0x0258` | `CORE_VENDOR_SPEC3` | RW | Vendor control 3 (`POR_VAL = 0x022C4098`) |
| `0x0264` | `CORE_DLL_CONFIG_3` | RW | Tertiary DLL configuration (`POR_VAL = 0x10`) |
| `0x03E8` | `CORE_DDR_CONFIG` | RW | DDR timing configuration (`0x80040873`) |

**[HW]** **`CORE_VENDOR_SPEC` (`0x020C`) Bitfields**:

| Bits | Name | Description |
|---|---|---|
| `19:16` | `SELECT_IN` | Tuning/CDR sampling select: `0` = non-CDR, `2` = feedback clock (`HC_SELECT_IN_FB_CLK`), `4` = SDR50, `6` = HS400 |
| `18` | `SELECT_IN_EN` | `1` = enable `SELECT_IN` mux |
| `15` | `MCLK_REMOVABLE` | `1` = allow MCLK gating |
| `14:13` | `CDR_SELEXT` | CDR external phase selection |
| `12` | `CDR_EXT_EN` | `1` = enable external CDR phase |
| `11` | `CDR_EN` | `1` = enable Clock and Data Recovery |
| `9` | `IO_PAD_PWR_SWITCH` | `0` = 3.3V I/O pad signaling, `1` = 1.8V I/O pad signaling |
| `8` | `IO_PAD_PWR_SWITCH_EN` | `1` = enable software control of `IO_PAD_PWR_SWITCH` |
| `0` | `PWRSAVE` | `1` = gate SDCLK when bus is idle |

### 6.3 Required `PWRCTL` Handshake Sequence

**[HW]** Whenever software writes to `SDHCI_POWER_CONTROL` (`0x029`), `SDHCI_SOFTWARE_RESET` (`0x02F`), or the 1.8V signaling bit in `SDHCI_HOST_CONTROL2` (`0x03E`), the SDHCI-MSM hardware blocks internal state progression and asserts bits in `CORE_PWRCTL_STATUS` (`0x0240`) (and raises GSIV 290 if enabled in `CORE_PWRCTL_MASK`). Software **must** complete the `PWRCTL` handshake or subsequent SDHCI commands will time out:

1. Read `status = CORE_PWRCTL_STATUS` (`0x0240`) and mask with `0x0F`.
2. Write `status` to `CORE_PWRCTL_CLEAR` (`0x0248`), then poll `CORE_PWRCTL_STATUS` (`0x0240`) until the cleared bits read back as `0`.
3. Handle the requested power/voltage transition:
   - If `status & 0x03` (`BUS_OFF = bit 0` or `BUS_ON = bit 1`), configure the card VMMC regulator if managed by the OS, and prepare response bit `0` (`REQ_BUS_PWR_SUCCESS = 0x01`).
   - If `status & 0x0C` (`IO_LOW = bit 2` for 1.8V or `IO_HIGH = bit 3` for 3.3V), update `CORE_VENDOR_SPEC` (`0x020C`):
     - If `IO_LOW` (bit `2`, 1.8V), set bit `9` (`IO_PAD_PWR_SWITCH = 1`).
     - If `IO_HIGH` (bit `3`, 3.3V), clear bit `9` (`IO_PAD_PWR_SWITCH = 0`).
     - Prepare response bit `2` (`REQ_IO_SIG_SWITCH_SUCCESS = 0x04`).
4. Write the response (`0x01`, `0x04`, or `0x05`) to `CORE_PWRCTL_CTL` (`0x024C`).
5. Execute a memory barrier or read back `CORE_PWRCTL_CTL` (`0x024C`) to ensure the handshake write completes before issuing further SDHCI register accesses.

---

## 7. Synopsys DWC3 USB3 Controllers (`0x0A600000`, `0x0A800000`, `0x0A400000`)

### 7.1 Overview and ACPI Hierarchy

**[HW]** **[ABI]** The SC8280XP provides three Synopsys DesignWare USB3 (`DWC_usb31`) controllers:
1. **USB 0 (`0x0A600000`)**: Dual-role controller under ACPI `\_SB.URS0` (`_HID "QCOM068B"`, `_CID "PNP0CA1"`), with child device `USB0` (`_ADR 0`, GSIV `0x343` / 835). Wrapper QScratch registers are at `0x0A6F8800`.
2. **USB 1 (`0x0A800000`)**: Dual-role controller under ACPI `\_SB.URS1` (`_HID "QCOM068B"`, `_CID "PNP0CA1"`), with child device `USB0` (`_ADR 0`). Wrapper QScratch registers are at `0x0A8F8800`.
3. **USB 2 Multiport (`0x0A400000`)**: 4-port host-only XHCI controller under ACPI `\_SB.USB2` (`_HID "QCOM0826"`, `_CID "PNP0D10"`). Wrapper QScratch registers are at `0x0A4F8800`.

**[HW]** In each DWC3 `0x100000` aperture:
- Offset `0x0000`–`0x7FFF`: Standard XHCI Host Controller registers (`CAPLENGTH`, `HCIVERSION`, `HCSPARAMS1..3`, `HCCPARAMS1..2`, operational, runtime, and doorbell registers).
- Offset `0xC100`–`0xCCFF`: Synopsys DWC3 Global Registers.

### 7.2 DWC3 Global Registers and SuperSpeed RX Threshold Quirk

**[HW]** Key DWC3 global registers relative to the controller base (`0x0A600000`, `0x0A800000`, `0x0A400000`):

| Offset | Register Name | Access | Description |
|---|---|---|---|
| `0xC100` | `GSBUSCFG0` | RW | Global SoC Bus Configuration 0 (burst lengths, cacheability) |
| `0xC104` | `GSBUSCFG1` | RW | Global SoC Bus Configuration 1 (`[11:8]` = `PIPETRANSLIMIT`) |
| `0xC108` | `GTXTHRCFG` | RW | Global TX Threshold Configuration |
| `0xC10C` | `GRXTHRCFG` | RW | Global RX Threshold Configuration |
| `0xC110` | `GCTL` | RW | Global Core Control (`[13:12]` = `PRTCAPDIR`: `1` = Host, `2` = Device, `3` = OTG; `[11]` = `CORESOFTRESET`) |
| `0xC118` | `GSTS` | RW | Global Status Register |
| `0xC11C` | `GUCTL1` | RW | Global User Control 1 |
| `0xC120` | `GSNPSID` | RO | Synopsys Core Identification (`[31:16]` = core ID, `[15:0]` = release number; `0x5533xxxx` = `DWC_usb3`, `0x3331xxxx` = `DWC_usb31`, `0x3332xxxx` = `DWC_usb32`) |
| `0xC12C` | `GUID` | RW | Global User ID |
| `0xC130` | `GUCTL` | RW | Global User Control |
| `0xC140` | `GHWPARAMS0` | RO | Global Hardware Parameters 0 (`[2:0]` = mode, `[15:8]` = data bus width) |
| `0xC144` | `GHWPARAMS1` | RO | Global Hardware Parameters 1 |
| `0xC148` | `GHWPARAMS2` | RO | Global Hardware Parameters 2 |
| `0xC14C` | `GHWPARAMS3` | RO | Global Hardware Parameters 3 (`[1:0]` = SS PHY ifc, `[3:2]` = HS PHY ifc) |
| `0xC200` | `GUSB2PHYCFG0` | RW | Global USB2 PHY Configuration 0 (`[31]` = `PHYSOFTRST`, `[8]` = `ENBLSLPM`, `[6]` = `SUSPHY`) |
| `0xC2C0` | `GUSB3PIPECTL0` | RW | Global USB3 PIPE Control 0 (`[31]` = `PHYSoftRst`, `[17]` = `SUSPHY`) |

**[HW]** **`GRXTHRCFG` (`0xC10C`) Bitfields and Mandatory SuperSpeed Throughput Fix**:

| Core Type (`GSNPSID[31:16]`) | `PKTCNTSEL` Bit | `RXPKTCNT` Bits | `MAXRXBURSTSIZE` Bits |
|---|---|---|---|
| `DWC_usb3` (`0x5533`) | Bit `29` (`0x20000000`) | `[27:24]` | `[23:19]` |
| `DWC_usb31` (`0x3331`) / `DWC_usb32` (`0x3332`) | **Bit `26` (`0x04000000`)** | `[24:20]` | `[19:16]` |

**[HW]** **Empirical UEFI Quirk on SC8280XP (`DWC_usb31`)**:
- UEFI firmware leaves `GRXTHRCFG` (`0xC10C`) set to `0x04F30000` on the USB3 controllers:
  - Bit `26` (`DWC31_GRXTHRCFG_PKTCNTSEL = 0x04000000`) = `1` (RX packet count thresholding enabled).
  - Bits `[24:20]` (`RXPKTCNT`) = `15`.
  - Bits `[19:16]` (`MAXRXBURSTSIZE`) = `3`.
- When bit `26` (`PKTCNTSEL`) is set in host mode on SC8280XP without matching periodic ESS RX threshold tuning, SuperSpeed (5 Gbps / 10 Gbps) bulk IN transfers are throttled to ~35 MB/s (USB 2.0 speeds).
- **Required Fix**: During XHCI/DWC3 host initialization, inspect `GSNPSID` (`0xC120`) and clear `GRXTHRCFG` (`0xC10C`) (or clear bit `26` on `DWC_usb31`/`DWC_usb32` and bit `29` on `DWC_usb3`) to `0x00000000`. Clearing `GRXTHRCFG` immediately restores full SuperSpeed bulk throughput (~112 MB/s on 1 GbE USB3 adapters and >400 MB/s on USB3 storage).
