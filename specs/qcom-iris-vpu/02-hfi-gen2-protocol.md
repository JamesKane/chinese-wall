# Host Firmware Interface Generation 2 (HFI Gen2) Protocol

## 1. Shared Memory Layout (`UC_REGION` and `SFR`)

### 1.1 Overview of Shared Memory Allocations
- **[HW]** Host software allocates two 4 KiB-aligned coherent DMA buffers mapped into the VPU's non-protected IOVA window (`0x25800000`–`0xdfffffff`):
  1. **Interface Queue Table and Circular Queues (`UC_REGION`):** Total raw size `456 + (3 × 1,048,576) = 3,146,184` bytes, rounded up to a 4 KiB page boundary = **`0x00301000` (3,149,824 bytes)**. Programmed into `UC_REGION_ADDR` (`0x000a0064`), `UC_REGION_SIZE` (`0x000a0068`), and `QTBL_ADDR` (`0x000a0054`).
  2. **Subsystem Failure Reason (`SFR`) Buffer:** **`0x00001000` (4,096 bytes)**, zero-initialized. Programmed into `SFR_ADDR` (`0x000a005c`) as `sfr_iova + 1` (where `1` is `VIDEO_ARCH_LX`).

### 1.2 Queue Table Header Layout (288 Bytes at Offset `0x0000`)
- **[HW]** At byte offset `0x0000` of the `UC_REGION` buffer lies the 288-byte (`0x120`-byte) Queue Table Header, immediately followed by three 56-byte (`0x38`-byte) Queue Headers (for a combined header size of `288 + 168 = 456` bytes = `0x1c8` bytes):

| Byte Offset | Size | Type | Field Name | Initial Value & Description | Tag |
|---|---|---|---|---|---|
| `0x0000` | `4` | `u32` | `version` | `0x00000000` | **[HW]** |
| `0x0004` | `4` | `u32` | `size` | `456` (`0x000001c8`): Total byte size of the table header plus all 3 queue headers | **[HW]** |
| `0x0008` | `4` | `u32` | `qhdr0_offset` | `288` (`0x00000120`): Byte offset from start of table header to Queue Header 0 | **[HW]** |
| `0x000c` | `4` | `u32` | `qhdr_size` | `56` (`0x00000038`): Byte size of each individual Queue Header | **[HW]** |
| `0x0010` | `4` | `u32` | `num_q` | `3`: Total number of queues (`0` = Host-to-Firmware Command, `1` = Firmware-to-Host Message, `2` = Firmware-to-Host Debug) | **[HW]** |
| `0x0014` | `4` | `u32` | `num_active_q` | `3`: Number of active queues | **[HW]** |
| `0x0018` | `4` | `u32` | `device_addr` | `0x00000000` (unused / NULL) | **[HW]** |
| `0x001c` | `256` | `char[256]` | `name` | Zero-filled (`"N/A"` or `\0`) | **[HW]** |

### 1.3 Individual Queue Header Layout (3 × 56 Bytes at Offsets `0x0120`, `0x0158`, `0x0190`)
- **[HW]** Each of the 3 queues has a 56-byte (`0x38`-byte) header describing a 1 MiB (`0x00100000` = `1,048,576` bytes = `262,144` dwords) circular ring buffer:
  - **Queue 0 (`CMD_Q` at `0x0120`):** Host writes commands, Firmware reads. Ring buffer at offset `0x000001c8`, size `0x00100000`.
  - **Queue 1 (`MSG_Q` at `0x0158`):** Firmware writes responses/events, Host reads. Ring buffer at offset `0x001001c8`, size `0x00100000`.
  - **Queue 2 (`DBG_Q` at `0x0190`):** Firmware writes debug trace strings, Host reads. Ring buffer at offset `0x002001c8`, size `0x00100000`.

| Relative Offset | Size | Type | Field Name | Initial Value & Description | Tag |
|---|---|---|---|---|---|
| `+0x00` | `4` | `u32` | `status` | `0x00000001` (`1` = Active) | **[HW]** |
| `+0x04` | `4` | `u32` | `start_addr` | 32-bit device IOVA of the circular ring buffer (`qtable_iova + 0x1c8 + (queue_id * 0x100000)`) | **[HW]** |
| `+0x08` | `4` | `u32` | `type` | Queue type bitfield: `CMD_Q` (`0`) = `0x01000000`, `MSG_Q` (`1`) = `0x02010000`, `DBG_Q` (`2`) = `0x00020000` (Bits `[7:0]` = priority `0`, Bits `[15:8]` = Basic type `0`, Bits `[23:16]` = Queue ID `0/1/2`, Bits `[31:24]` = Direction `1` TX for Q0, `2` RX for Q1, `0` for Q2) | **[HW]** |
| `+0x0c` | `4` | `u32` | `q_size` | `262144` (`0x00040000`): Queue size in **32-bit dwords** (`1,048,576 / 4`) | **[HW]** |
| `+0x10` | `4` | `u32` | `pkt_size` | `0x00000000`: Variable packet size (`0` means every HFI message begins with a 32-bit byte-length field) | **[HW]** |
| `+0x14` | `4` | `u32` | `pkt_drop_cnt` | `0x00000000`: Dropped packet counter | **[HW]** |
| `+0x18` | `4` | `u32` | `rx_wm` | `0x00000001`: Receive watermark | **[HW]** |
| `+0x1c` | `4` | `u32` | `tx_wm` | `0x00000001`: Transmit watermark | **[HW]** |
| `+0x20` | `4` | `u32` | `rx_req` | `1` for `CMD_Q` and `MSG_Q`; `0` for `DBG_Q`. Indicates receiver wants an interrupt when new data is written | **[HW]** |
| `+0x24` | `4` | `u32` | `tx_req` | `0x00000000`: Set to `1` by writer if ring is full and writer wants an interrupt when space frees up | **[HW]** |
| `+0x28` | `4` | `u32` | `rx_irq_status` | `0x00000000`: Receiver interrupt status | **[HW]** |
| `+0x2c` | `4` | `u32` | `tx_irq_status` | `0x00000000`: Transmitter interrupt status | **[HW]** |
| `+0x30` | `4` | `u32` | `read_idx` | `0x00000000`: Read index in **32-bit dwords** (`0`..`262143`) | **[HW]** |
| `+0x34` | `4` | `u32` | `write_idx` | `0x00000000`: Write index in **32-bit dwords** (`0`..`262143`) | **[HW]** |

### 1.4 Circular Queue Read and Write Rules
- **[HW]** Both `read_idx` and `write_idx` are measured in **32-bit words (4-byte units)**, modulo `q_size` (`262,144` dwords). All HFI messages are a multiple of 4 bytes in size.
- **[HW]** **Enqueuing to `CMD_Q` (Queue 0):**
  1. Compute `write_size_dwords = msg_byte_size >> 2`.
  2. Read `read_idx` and `write_idx`. Available space in dwords is `(read_idx - write_idx - 1) mod q_size` (or `q_size + read_idx - write_idx` when wrapped). If insufficient space, set `tx_req = 1` and return error.
  3. Copy the message bytes to `ring_base + (write_idx << 2)`, wrapping around the end of the 1 MiB ring buffer if `write_idx + write_size_dwords > q_size`.
  4. Execute a DMA write memory barrier (`dmb ishst`).
  5. Update `write_idx = (write_idx + write_size_dwords) % q_size`.
  6. Execute a full memory barrier (`dmb ish`) and, if `rx_req != 0`, ring the VPU doorbell by writing `0x00010000` (`1 << 16`) to `CPU_IC_SOFTINT` (`0x000a0150`).
- **[HW]** **Dequeuing from `MSG_Q` (Queue 1) and `DBG_Q` (Queue 2):**
  1. Read `read_idx` and `write_idx`. If `read_idx == write_idx`, clear `tx_req = 0` (for `MSG_Q`) and stop.
  2. Read the first 32-bit word at `ring_base + (read_idx << 2)` as `pkt_byte_size`. Validate that `(pkt_byte_size & 3) == 0` and `pkt_byte_size <= 4096`.
  3. Copy `pkt_byte_size` bytes from `ring_base + (read_idx << 2)` into a contiguous local buffer, wrapping around `q_size` if needed.
  4. Advance `read_idx = (read_idx + (pkt_byte_size >> 2)) % q_size`.
  5. For `MSG_Q`, if the ring transitions from empty to non-empty or at the end of reading, ensure `rx_req = 1`; if `tx_req` was `1`, clear `tx_req = 0` and ring `CPU_IC_SOFTINT` (`0x000a0150` = `0x00010000`).

---

## 2. HFI Gen2 Message and Packet Binary Formats

### 2.1 HFI Gen2 Message Header (`32` Bytes)
- **[HW]** Every HFI Gen2 message written to `CMD_Q` or read from `MSG_Q` starts with a **32-byte (`0x20`-byte) Message Header**, followed immediately by `num_packets` back-to-back HFI Packets:

| Byte Offset | Size | Type | Field Name | Description | Tag |
|---|---|---|---|---|---|
| `+0x00` | `4` | `u32` | `size` | Total byte size of this HFI message including the 32-byte header and all `num_packets` packets | **[HW]** |
| `+0x04` | `4` | `u32` | `session_id` | `0x00000000` for system-level messages (`SYSTEM_INIT`, `IMAGE_VERSION`, `SYSTEM_ERROR`); non-zero unique session identifier (e.g., `1`, `2`, ...) for instance-level messages | **[HW]** |
| `+0x08` | `4` | `u32` | `header_id` | Monotonically incrementing 32-bit sequence counter per message | **[HW]** |
| `+0x0c` | `16` | `u32[4]` | `reserved` | Must be zero (`0`) | **[HW]** |
| `+0x1c` | `4` | `u32` | `num_packets` | Number of `HFI_PACKET` structures immediately following this header (`>= 1`) | **[HW]** |

### 2.2 HFI Gen2 Packet Header (`32` Bytes + Variable Payload)
- **[HW]** Each packet inside an HFI Gen2 message consists of a **32-byte (`0x20`-byte) Packet Header** followed by `(size - 32)` bytes of payload (padded/aligned to 4-byte dwords):

| Byte Offset | Size | Type | Field Name | Description | Tag |
|---|---|---|---|---|---|
| `+0x00` | `4` | `u32` | `size` | Total byte size of this packet (`32 + payload_byte_size`) | **[HW]** |
| `+0x04` | `4` | `u32` | `type` | Command ID (`0x01000000`..), Property ID (`0x03000000`..), or System Error (`0x08000001`) | **[HW]** |
| `+0x08` | `4` | `u32` | `flags` | Bitmask of `HFI_FW_FLAGS_*` or `HFI_HOST_FLAGS_*` (see Section 2.3) | **[HW]** |
| `+0x0c` | `4` | `u32` | `payload_info` | Payload data type (`HFI_PAYLOAD_*`, see Section 2.3) | **[HW]** |
| `+0x10` | `4` | `u32` | `port` | Target port (`HFI_PORT_NONE = 0`, `HFI_PORT_BITSTREAM = 1`, `HFI_PORT_RAW = 2`) | **[HW]** |
| `+0x14` | `4` | `u32` | `packet_id` | Monotonically incrementing 32-bit packet sequence number | **[HW]** |
| `+0x18` | `8` | `u32[2]` | `reserved` | Must be zero (`0`) | **[HW]** |
| `+0x20` | `N` | `u32[]` | `payload` | Packet payload (`N = size - 32` bytes) | **[HW]** |

### 2.3 Packet Flags, Ports, and Payload Types
- **[HW]** Packet header constants:

| Constant Name | Hex Value | Description | Tag |
|---|---|---|---|
| `HFI_FW_FLAGS_SUCCESS` | `0x00000001` | Set in response packets when command/property succeeded | **[HW]** |
| `HFI_FW_FLAGS_INFORMATION` | `0x00000002` | Informational message from firmware | **[HW]** |
| `HFI_FW_FLAGS_SESSION_ERROR` | `0x00000004` | Session-level error (`type` contains `HFI_SYSTEM_ERROR_*` or `HFI_SESSION_ERROR_*`) | **[HW]** |
| `HFI_FW_FLAGS_SYSTEM_ERROR` | `0x00000008` | Fatal system-level firmware error | **[HW]** |
| `HFI_HOST_FLAGS_RESPONSE_REQUIRED` | `0x00000001` | Host requests an explicit acknowledgement response packet | **[HW]** |
| `HFI_HOST_FLAGS_INTR_REQUIRED` | `0x00000002` | Host requests an interrupt upon completion | **[HW]** |
| `HFI_HOST_FLAGS_NON_DISCARDABLE` | `0x00000004` | Packet cannot be dropped by firmware | **[HW]** |
| `HFI_HOST_FLAGS_GET_PROPERTY` | `0x00000008` | Query property value | **[HW]** |
| `HFI_PORT_NONE` | `0x00000000` | Global or session-wide (non-port-specific) | **[HW]** |
| `HFI_PORT_BITSTREAM` | `0x00000001` | Compressed bitstream port (Decoder INPUT / Encoder OUTPUT) | **[HW]** |
| `HFI_PORT_RAW` | `0x00000002` | Uncompressed raw pixel port (Decoder OUTPUT / Encoder INPUT) | **[HW]** |
| `HFI_PAYLOAD_NONE` | `0x00000000` | No payload (`size = 32`) | **[HW]** |
| `HFI_PAYLOAD_U32` | `0x00000001` | 4-byte unsigned integer (`size = 36`) | **[HW]** |
| `HFI_PAYLOAD_S32` | `0x00000002` | 4-byte signed integer (`size = 36`) | **[HW]** |
| `HFI_PAYLOAD_U64` | `0x00000003` | 8-byte unsigned integer (`size = 40`) | **[HW]** |
| `HFI_PAYLOAD_S64` | `0x00000004` | 8-byte signed integer (`size = 40`) | **[HW]** |
| `HFI_PAYLOAD_STRUCTURE` | `0x00000005` | Binary structure (e.g., 64-byte `HFI_BUFFER`, `size = 96`) | **[HW]** |
| `HFI_PAYLOAD_BLOB` | `0x00000006` | Variable-length byte blob | **[HW]** |
| `HFI_PAYLOAD_STRING` | `0x00000007` | Null-terminated ASCII string | **[HW]** |
| `HFI_PAYLOAD_Q16` | `0x00000008` | 32-bit fixed-point Q16 (`(int_val << 16) \| frac_val`, `size = 36`) | **[HW]** |
| `HFI_PAYLOAD_U32_ENUM` | `0x00000009` | 4-byte enumeration value (`size = 36`) | **[HW]** |
| `HFI_PAYLOAD_32_PACKED` | `0x0000000a` | Packed 32-bit fields (`size = 36`) | **[HW]** |
| `HFI_PAYLOAD_U32_ARRAY` | `0x0000000b` | Array of 32-bit unsigned integers | **[HW]** |
| `HFI_PAYLOAD_64_PACKED` | `0x0000000d` | Packed 64-bit fields (`size = 40`) | **[HW]** |

### 2.4 HFI Gen2 Buffer Descriptor (`64` Bytes, Payload of `HFI_CMD_BUFFER`)
- **[HW]** All buffer operations (queuing input/output frames, registering/releasing internal scratch buffers, or returning decoded/encoded frames from firmware) use `HFI_CMD_BUFFER` (`0x01000009`) with `payload_info = HFI_PAYLOAD_STRUCTURE` (`0x5`) and a **64-byte (`0x40`-byte) `HFI_BUFFER` payload** (total packet size = `32 + 64 = 96` bytes):

| Payload Offset | Size | Type | Field Name | Description | Tag |
|---|---|---|---|---|---|
| `+0x00` | `4` | `u32` | `type` | Buffer type (`HFI_BUFFER_BITSTREAM = 1`, `HFI_BUFFER_RAW = 2`, `HFI_BUFFER_BIN = 4`, `HFI_BUFFER_ARP = 5`, `HFI_BUFFER_COMV = 6`, `HFI_BUFFER_NON_COMV = 7`, `HFI_BUFFER_LINE = 8`, `HFI_BUFFER_DPB = 9`, `HFI_BUFFER_PERSIST = 10`, `HFI_BUFFER_VPSS = 11`) | **[HW]** |
| `+0x04` | `4` | `u32` | `index` | Host buffer index (`0` for internal buffers) | **[HW]** |
| `+0x08` | `8` | `u64` | `base_address` | Base IOVA in the SMMU domain (`0x25800000`–`0xdfffffff`) | **[HW]** |
| `+0x10` | `4` | `u32` | `addr_offset` | Byte offset from `base_address` (`0`) | **[HW]** |
| `+0x14` | `4` | `u32` | `buffer_size` | Total allocated buffer size in bytes (for raw decode output buffers with HFI linear stride, set to `stride * scanline * 3 / 2` or UBWC buffer size) | **[HW]** |
| `+0x18` | `4` | `u32` | `data_offset` | Byte offset to valid payload data within the buffer | **[HW]** |
| `+0x1c` | `4` | `u32` | `data_size` | Valid data byte length (`bytesused - data_offset` for input; `0` when queuing capture buffers to firmware; filled byte count when returned by firmware) | **[HW]** |
| `+0x20` | `8` | `u64` | `timestamp` | Presentation timestamp in **microseconds** (`v4l2_timestamp_ns / 1000`) | **[HW]** |
| `+0x28` | `4` | `u32` | `flags` | Bitmask of `HFI_BUF_HOST_FLAG_*` (host-to-firmware) and `HFI_BUF_FW_FLAG_*` (firmware-to-host) | **[HW]** |
| `+0x2c` | `20` | `u32[5]` | `reserved` | Zero-filled (`0`) | **[HW]** |

- **[HW]** Buffer Flags (`HFI_BUFFER.flags` at `+0x28`):

| Flag Name | Hex Value | Direction | Description | Tag |
|---|---|---|---|---|
| `HFI_BUF_HOST_FLAG_RELEASE` | `0x00000001` | Host → FW | Instructs firmware to unmap/release a previously registered buffer | **[HW]** |
| `HFI_BUF_HOST_FLAG_READONLY` | `0x00000010` | Host → FW | Buffer is held as a read-only reference frame | **[HW]** |
| `HFI_BUF_HOST_FLAG_CODEC_CONFIG` | `0x00000100` | Host → FW | Buffer contains out-of-band codec header (SPS/PPS/VPS) | **[HW]** |
| `HFI_BUF_FW_FLAG_RELEASE_DONE` | `0x00000200` | FW → Host | Firmware has finished releasing the buffer; host may unmap/free it | **[HW]** |
| `HFI_BUF_FW_FLAG_READONLY` | `0x00001000` | FW → Host | Decoded frame is still referenced internally by DPB | **[HW]** |
| `HFI_BUF_FW_FLAG_LAST` | `0x01000000` | Host ↔ FW | Last buffer of a drain (`CMD_STOP`) or stream (`V4L2_BUF_FLAG_LAST`) | **[HW]** |
| `HFI_BUF_FW_FLAG_PSC_LAST` | `0x02000000` | FW → Host | Last decoded frame before a Port Settings Change (dynamic resolution change) | **[HW]** |
| `HFI_BUF_FW_FLAG_CODEC_CONFIG` | `0x10000000` | FW → Host | Encoded output buffer contains SPS/PPS/VPS header bytes | **[HW]** |
| `HFI_BUF_FW_FLAG_CORRUPT` | `0x20000000` | FW → Host | Decoded frame contains bitstream errors (`V4L2_BUF_FLAG_ERROR`) | **[HW]** |
| `HFI_BUF_FW_FLAG_OVERFLOW` | `0x40000000` | FW → Host | Buffer overflowed | **[HW]** |

---

## 3. HFI Gen2 Commands, Properties, and Enumerations

### 3.1 HFI Gen2 Command IDs (`0x01000000`–`0x01ffffff`)

| Command Name | Hex Value | Port | Payload | Description | Tag |
|---|---|---|---|---|---|
| `HFI_CMD_SYSTEM_INIT` | `0x01000001` | `NONE` | `U32` (`1` = `VIDEO_ARCH_LX`) | Initializes the HFI firmware after boot. Sent with `flags = RESPONSE_REQUIRED \| INTR_REQUIRED \| NON_DISCARDABLE`, `session_id = 0`. Accompanied in the same message by UBWC configuration property packets | **[HW]** |
| `HFI_CMD_SESSION_OPEN` | `0x01000002` | `NONE` | `U32_ENUM` (`1` = Decoder, `2` = Encoder) | Opens a codec session (`session_id != 0`). Sent in a multi-packet message accompanied by `HFI_PROP_CODEC` and `HFI_PROP_COLOR_FORMAT` | **[HW]** |
| `HFI_CMD_SESSION_CLOSE` | `0x01000003` | `NONE` | `NONE` | Closes a codec session and frees firmware session state | **[HW]** |
| `HFI_CMD_START` | `0x01000004` | `BITSTREAM` or `RAW` | `NONE` | Starts streaming on the specified port | **[HW]** |
| `HFI_CMD_STOP` | `0x01000005` | `BITSTREAM` or `RAW` | `NONE` | Stops streaming on the specified port and returns all queued buffers | **[HW]** |
| `HFI_CMD_DRAIN` | `0x01000006` | `BITSTREAM` (dec) or `RAW` (enc) | `NONE` | Initiates a stateful drain; firmware processes all pending inputs and marks the final capture buffer with `HFI_BUF_FW_FLAG_LAST` | **[HW]** |
| `HFI_CMD_CLEAR_IRQ` | `0x01000007` | `NONE` | `NONE` | Clear IRQ command | **[HW]** |
| `HFI_CMD_RESUME` | `0x01000008` | `BITSTREAM` or `RAW` | `NONE` | Resumes streaming after a Port Settings Change (`DRC`) | **[HW]** |
| `HFI_CMD_BUFFER` | `0x01000009` | `BITSTREAM` or `RAW` | `STRUCTURE` (`64` B) | Queues, registers, releases, or returns a DMA buffer | **[HW]** |
| `HFI_CMD_SUBSCRIBE_MODE` | `0x0100000a` | `BITSTREAM` or `RAW` | `U32_ARRAY` | Subscribes to port settings change notifications (`0x01` = `CMD_SETTINGS_CHANGE`, or `0x02` = `PROPERTY` delivery) | **[HW]** |
| `HFI_CMD_SETTINGS_CHANGE` | `0x0100000b` | `BITSTREAM` (FW→Host) | `U32_ARRAY` | Sent by firmware when bitstream headers change resolution, bit-depth, crop, or DPB requirements | **[HW]** |
| `HFI_CMD_DELIVERY_MODE` | `0x0100000c` | `BITSTREAM` or `RAW` | `U32_ARRAY` | Configures property delivery mode per buffer | **[HW]** |
| `HFI_CMD_PWR_COLLAPSE` | `0x0100000d` | `NONE` | `NONE` | Prepares firmware for power collapse (`session_id = 0`) | **[HW]** |

### 3.2 System and UBWC Configuration Packet (`HFI_CMD_SYSTEM_INIT`)
- **[HW]** The `HFI_CMD_SYSTEM_INIT` message (`session_id = 0`, `num_packets = 8`) contains:
  1. Packet 0: `type = HFI_CMD_SYSTEM_INIT` (`0x01000001`), `flags = 0x7`, `payload_info = HFI_PAYLOAD_U32` (`1`), `port = 0`, `payload[0] = 1` (`VIDEO_ARCH_LX`).
  2. Packets 1..7 (UBWC system properties):
     - `HFI_PROP_UBWC_MAX_CHANNELS` (`0x03000003`), `HFI_PAYLOAD_U32`: `8` on SC8280XP.
     - `HFI_PROP_UBWC_MAL_LENGTH` (`0x03000004`), `HFI_PAYLOAD_U32`: `32` (Min Access Length = 32 bytes).
     - `HFI_PROP_UBWC_HBB` (`0x03000005`), `HFI_PAYLOAD_U32`: `16` (Highest Bank Bit = 16).
     - `HFI_PROP_UBWC_BANK_SWZL_LEVEL1` (`0x03000006`), `HFI_PAYLOAD_U32`: `0` (Bank Swizzle Level 1 disabled).
     - `HFI_PROP_UBWC_BANK_SWZL_LEVEL2` (`0x03000007`), `HFI_PAYLOAD_U32`: `1` (Bank Swizzle Level 2 enabled).
     - `HFI_PROP_UBWC_BANK_SWZL_LEVEL3` (`0x03000008`), `HFI_PAYLOAD_U32`: `1` (Bank Swizzle Level 3 enabled).
     - `HFI_PROP_UBWC_BANK_SPREADING` (`0x03000009`), `HFI_PAYLOAD_U32`: `1` (Bank Spreading enabled).

### 3.3 HFI Gen2 Property IDs (`0x03000000`–`0x03ffffff`)

| Property Name | Hex Value | Payload Type | Port | Description & Encoding | Tag |
|---|---|---|---|---|---|
| `HFI_PROP_IMAGE_VERSION` | `0x03000001` | `STRING` | `NONE` | Sent by FW on `SYSTEM_INIT` response with firmware version string | **[HW]** |
| `HFI_PROP_UBWC_MAX_CHANNELS` | `0x03000003` | `U32` | `NONE` | DDR channels (`8` on SC8280XP) | **[HW]** |
| `HFI_PROP_UBWC_MAL_LENGTH` | `0x03000004` | `U32` | `NONE` | Minimum Access Length (`32`) | **[HW]** |
| `HFI_PROP_UBWC_HBB` | `0x03000005` | `U32` | `NONE` | Highest Bank Bit (`16`) | **[HW]** |
| `HFI_PROP_UBWC_BANK_SWZL_LEVEL1` | `0x03000006` | `U32` | `NONE` | Bank swizzle level 1 (`0`) | **[HW]** |
| `HFI_PROP_UBWC_BANK_SWZL_LEVEL2` | `0x03000007` | `U32` | `NONE` | Bank swizzle level 2 (`1`) | **[HW]** |
| `HFI_PROP_UBWC_BANK_SWZL_LEVEL3` | `0x03000008` | `U32` | `NONE` | Bank swizzle level 3 (`1`) | **[HW]** |
| `HFI_PROP_UBWC_BANK_SPREADING` | `0x03000009` | `U32` | `NONE` | Bank spreading (`1`) | **[HW]** |
| `HFI_PROP_CODEC` | `0x03000100` | `U32_ENUM` | `NONE` | `1` = `HFI_CODEC_DEC_AVC` (H.264 Dec), `2` = `HFI_CODEC_ENC_AVC` (H.264 Enc), `3` = `HFI_CODEC_DEC_HEVC` (HEVC Dec), `4` = `HFI_CODEC_ENC_HEVC` (HEVC Enc), `5` = `HFI_CODEC_DEC_VP9` (VP9 Dec) | **[HW]** |
| `HFI_PROP_COLOR_FORMAT` | `0x03000101` | `U32_ENUM` | `RAW` | `1` = `NV12` (linear), `2` = `NV12_UBWC` (`QC08C`), `3` = `NV12_TP10_UBWC` (`QC10C` 10-bit), `4` = `P010` (linear 10-bit), `5` = `NV21` | **[HW]** |
| `HFI_PROP_BITSTREAM_RESOLUTION` | `0x03000103` | `32_PACKED` | `BITSTREAM` (dec) / `RAW` (enc) | `(width << 16) \| height` | **[HW]** |
| `HFI_PROP_LINEAR_STRIDE_SCANLINE` | `0x03000104` | `64_PACKED` | `RAW` | `((u64)y_stride << 48) \| ((u64)y_scanline << 32) \| ((u64)uv_stride << 16) \| (u64)uv_scanline` | **[HW]** |
| `HFI_PROP_CROP_OFFSETS` | `0x03000105` | `64_PACKED` | `BITSTREAM` / `RAW` | `payload[0] = (left << 16) \| top`; `payload[1] = (right_offset << 16) \| bottom_offset` (where `right_offset = aligned_width - left - crop_width`, `bottom_offset = aligned_height - top - crop_height`) | **[HW]** |
| `HFI_PROP_PROFILE` | `0x03000107` | `U32_ENUM` | `BITSTREAM` | Codec profile (H.264 / HEVC / VP9) | **[HW]** |
| `HFI_PROP_LEVEL` | `0x03000108` | `U32_ENUM` | `BITSTREAM` | Codec level | **[HW]** |
| `HFI_PROP_STAGE` | `0x0300010a` | `U32_ENUM` | `NONE` | Pipeline work stage: `1` = `1_STAGE` (widths `<= 1920`), `2` = `2_STAGE` (widths `> 1920`) | **[HW]** |
| `HFI_PROP_PIPE` | `0x0300010b` | `U32` | `NONE` | Hardware pipes: `1` (1 pipe), `2` (2 pipes), `4` (4 pipes; default on SC8280XP VPU2 is `4`) | **[HW]** |
| `HFI_PROP_FRAME_RATE` | `0x0300010c` | `Q16` | `RAW` or `BITSTREAM` | Frame rate in Q16 (`fps << 16`) | **[HW]** |
| `HFI_PROP_PIC_STRUCT` | `0x0300010e` | `U32` | `BITSTREAM` | Picture structure (`0` = progressive, `1` = interlaced) | **[HW]** |
| `HFI_PROP_SIGNAL_COLOR_INFO` | `0x03000115` | `32_PACKED` | `BITSTREAM` / `RAW` | Packed VUI colorimetry: `(matrix_coeff & 0xff) \| ((transfer_char & 0xff) << 8) \| ((primaries & 0xff) << 16) \| ((colour_desc_present & 0x1) << 24) \| ((full_range & 0x1) << 25) \| ((video_format & 0x7) << 26) \| ((video_signal_present & 0x1) << 29)` | **[HW]** |
| `HFI_PROP_PICTURE_TYPE` | `0x03000116` | `U32` | `RAW` / `BITSTREAM` | Delivered per buffer (`0x1` = `I`, `0x2` = `P`, `0x4` = `B`, `0x10` = `IDR`, `0x1000` = `CRA`) or written (`0x10` = force IDR on encoder) | **[HW]** |
| `HFI_PROP_DEC_DEFAULT_HEADER` | `0x03000117` | `U32` | `BITSTREAM` | `0` = default header disabled | **[HW]** |
| `HFI_PROP_DEC_START_FROM_RAP_FRAME` | `0x03000118` | `U32` | `BITSTREAM` | `1` = start decoding from Random Access Point (I/IDR), `0` = decode immediately | **[HW]** |
| `HFI_PROP_NO_OUTPUT` | `0x03000119` | `NONE` | `BITSTREAM` | Delivered by FW when an input bitstream buffer produced no displayable output frame | **[HW]** |
| `HFI_PROP_BUFFER_MARK` | `0x0300011b` | `U32` | `RAW` | Buffer mark identifier | **[HW]** |
| `HFI_PROP_WORST_COMPRESSION_RATIO` | `0x03000124` | `U32` | `BITSTREAM` | Reported by FW for worst-case UBWC compression ratio | **[HW]** |
| `HFI_PROP_MIN_OUTPUT_BUF_COUNT` | `0x03000125` | `U32` | `BITSTREAM` | Minimum required output/DPB buffers reported by FW | **[HW]** |
| `HFI_PROP_QUALITY_MODE` | `0x03000128` | `U32_ENUM` | `NONE` | Encoder quality mode (`1` = `MAX_QUALITY_MODE`, `2` = `POWER_SAVE_MODE`) | **[HW]** |
| `HFI_PROP_SEQ_HEADER_MODE` | `0x03000149` | `U32_ENUM` | `BITSTREAM` | `1` = Separate sequence header buffer, `2` = Joined with first frame | **[HW]** |
| `HFI_PROP_DPB_LIST` | `0x03000157` | `BLOB` | `RAW` | List of active DPB buffer addresses | **[HW]** |
| `HFI_PROP_TOTAL_BITRATE` | `0x0300015c` | `U32` | `BITSTREAM` | Target encoder bitrate in bits per second | **[HW]** |
| `HFI_PROP_BITRATE_CONTROL` | `0x0300015d` | `U32_ENUM` | `BITSTREAM` | Rate control mode (`0` = `RC_OFF`, `1` = `VBR_CFR`, `2` = `CBR_CFR`, `3` = `CBR_VFR`, `4` = `CQ`) | **[HW]** |
| `HFI_PROP_MAX_GOP_FRAMES` | `0x03000161` | `U32` | `BITSTREAM` | Maximum frames per GOP (`(b_frames + 1) * (p_frames + 1) - 1`) | **[HW]** |
| `HFI_PROP_MAX_B_FRAMES` | `0x03000162` | `U32` | `BITSTREAM` | Maximum consecutive B-frames (`0`..`2`) | **[HW]** |
| `HFI_PROP_REQUEST_SYNC_FRAME` | `0x03000163` | `NONE` | `BITSTREAM` | Force next encoded frame to be an IDR keyframe | **[HW]** |
| `HFI_PROP_CABAC_SESSION` | `0x03000171` | `U32` | `BITSTREAM` | H.264 entropy mode (`0` = CAVLC, `1` = CABAC) | **[HW]** |
| `HFI_PROP_LUMA_CHROMA_BIT_DEPTH` | `0x03000175` | `32_PACKED` | `BITSTREAM` | `(luma_bit_depth << 16) \| chroma_bit_depth`: `0x00080008` = 8-bit, `0x000a000a` = 10-bit | **[HW]** |
| `HFI_PROP_CODED_FRAMES` | `0x03000176` | `U32` | `BITSTREAM` | `1` = Progressive (`FRAME_MBS_ONLY_FLAG`), `2` = Interlaced | **[HW]** |
| `HFI_PROP_BUFFER_HOST_MAX_COUNT` | `0x03000179` | `U32` | `BITSTREAM` or `RAW` | Maximum host buffers (`64`) | **[HW]** |
| `HFI_PROP_BUFFER_FW_MIN_OUTPUT_COUNT` | `0x0300017a` | `U32` | `BITSTREAM` or `RAW` | Minimum output buffers required by firmware | **[HW]** |
| `HFI_PROP_PIC_ORDER_CNT_TYPE` | `0x0300017e` | `U32` | `BITSTREAM` | H.264 POC type | **[HW]** |
| `HFI_PROP_QUALITY_BOOST` | `0x03000185` | `U32` | `NONE` | Encoder quality boost (`1` = enabled) | **[HW]** |
| `HFI_PROP_QP_PACKED` | `0x03000188` | `32_PACKED` | `BITSTREAM` | `(b_qp) \| (p_qp << 8) \| (i_qp << 16) \| (enable_mask << 24)` | **[HW]** |
| `HFI_PROP_MIN_QP_PACKED` | `0x03000189` | `32_PACKED` | `BITSTREAM` | `(min_b_qp) \| (min_p_qp << 8) \| (min_i_qp << 16)` | **[HW]** |
| `HFI_PROP_MAX_QP_PACKED` | `0x0300018a` | `32_PACKED` | `BITSTREAM` | `(max_b_qp) \| (max_p_qp << 8) \| (max_i_qp << 16)` | **[HW]** |
| `HFI_PROP_IR_RANDOM_PERIOD` | `0x0300018c` | `U32` | `BITSTREAM` | Intra refresh random period | **[HW]** |
| `HFI_PROP_IR_CYCLIC_PERIOD` | `0x0300018e` | `U32` | `BITSTREAM` | Intra refresh cyclic period | **[HW]** |
| `HFI_PROP_HEVC_TIER` | `0x03000190` | `U32_ENUM` | `BITSTREAM` | `0` = Main Tier, `1` = High Tier | **[HW]** |
| `HFI_PROP_SYNC_FRAME_SEQUENCE_HEADER` | `0x03000192` | `U32` | `BITSTREAM` | `1` = Prepend SPS/PPS before every IDR frame | **[HW]** |
| `HFI_PROP_LTR_COUNT` | `0x03000193` | `U32` | `BITSTREAM` | Long-Term Reference frame count (`0`..`2`) | **[HW]** |
| `HFI_PROP_LTR_MARK` | `0x03000194` | `U32` | `RAW` | Mark current frame as LTR index | **[HW]** |
| `HFI_PROP_LTR_USE` | `0x03000195` | `U32` | `RAW` | Use LTR frame bitmask | **[HW]** |
| `HFI_PROP_SLICE_DEBLOCKING_IDC` | `0x0300019b` | `U32` | `BITSTREAM` | Loop filter mode (`0` = enabled, `1` = disabled, `2` = disabled at slice boundary) | **[HW]** |
| `HFI_PROP_SLICE_BYTE_COUNT` | `0x0300019c` | `U32` | `BITSTREAM` | Max bytes per slice | **[HW]** |
| `HFI_PROP_SLICE_MB_COUNT` | `0x0300019d` | `U32` | `BITSTREAM` | Max macroblocks/CTUs per slice | **[HW]** |
| `HFI_PROP_COMV_BUFFER_COUNT` | `0x0300019e` | `U32` | `BITSTREAM` | Number of co-located motion vector buffers | **[HW]** |
| `HFI_PROP_RAW_RESOLUTION` | `0x030001a2` | `32_PACKED` | `RAW` | `(width << 16) \| height` for raw input/output frames | **[HW]** |
| `HFI_PROP_TOTAL_PEAK_BITRATE` | `0x030001a5` | `U32` | `BITSTREAM` | Peak bitrate in bps | **[HW]** |
| `HFI_PROP_8X8_TRANSFORM` | `0x030001a9` | `U32` | `BITSTREAM` | H.264 8×8 transform enable (`0` or `1`) | **[HW]** |

---

## 4. Internal Scratch and Persist Buffer Sizing Requirements

### 4.1 Internal Buffers Overview
- **[HW]** Depending on the codec, work stage (`1_STAGE` vs `2_STAGE`), pipe count (`4` on SC8280XP), and resolution, the host must allocate, DMA-map into the SMMU domain (`0x25800000`–`0xdfffffff`), and register internal scratch/persist buffers with the firmware via `HFI_CMD_BUFFER` (`index = 0`) before calling `HFI_CMD_START`.
- **[HW]** All internal buffer sizes are rounded up to a **256-byte (`DMA_ALIGNMENT = 256`)** boundary (and allocated in 4 KiB pages).
- **[HW]** Which internal buffer types are required per port:
  - **Decoder Input (`BITSTREAM`) Port:** `BIN`, `COMV`, `NON_COMV`, `LINE`.
  - **Decoder Output (`RAW`) Port:** `DPB` (only required when the host selects linear `V4L2_PIX_FMT_NV12` output; when the host selects native `QC08C`/`QC10C` UBWC output, the capture buffers themselves serve as the DPB and no separate `HFI_BUFFER_DPB` allocation is needed).
  - **Decoder Persistent (Session-lifetime):** `PERSIST`.
  - **Encoder Input (`RAW`) Port:** `VPSS` (only when scaling or rotation is active; `0` bytes otherwise).
  - **Encoder Output (`BITSTREAM`) Port:** `BIN`, `COMV`, `NON_COMV`, `LINE`, `DPB` (recon buffers).
  - **Encoder Persistent (Session-lifetime):** `ARP`.

### 4.2 Decoder Internal Buffer Sizing Formulas (SC8280XP: `num_vpp_pipes = 4`)
- **[HW]** Let `W` and `H` be the coded bitstream width and height, `stage = (W <= 1920) ? 1 : 2`, and `pipes = 4`:
  1. **`HFI_BUFFER_PERSIST` (`10`):**
     - **H.264 (`AVC`):** `204,800` bytes (`0x32000`).
     - **HEVC:** `3,522,560` bytes (`0x35c000`).
     - **VP9:** `319,488` bytes (`0x4e000`, `312 * 1024` aligned to `4096`, covering probability tables + segmentation).
  2. **`HFI_BUFFER_BIN` (`4`):**
     - `0` bytes in `1_STAGE` (`W <= 1920`).
     - In `2_STAGE` (`W > 1920`), computed from `aligned_W * aligned_H * 3 / 2` scaled by worst-case bin ratios (`~12–30 MiB` for 4K, aligned to 256 bytes).
  3. **`HFI_BUFFER_COMV` (`6`):**
     - **H.264:** `comv_count * (((((W + 15) >> 4) * ((H + 15) >> 4)) << 8) + 512)`, where `comv_count = max(min_output_bufs, 6)`.
     - **HEVC:** `comv_count * ((((((W + 63) >> 6) * ((H + 63) >> 6)) << 9) + 512))`, rounded up to 256 bytes.
     - **VP9:** `0` bytes (not required).
  4. **`HFI_BUFFER_NON_COMV` (`7`):**
     - **H.264 / HEVC:** `~1–4 MiB` scratch proportional to macroblock/LCU frame dimensions and `pipes = 4`.
     - **VP9:** `0` bytes.
  5. **`HFI_BUFFER_LINE` (`8`):**
     - Line buffer proportional to `W`, `H`, and `pipes = 4` (`~1–4 MiB` for 1080p–4K).
  6. **`HFI_BUFFER_DPB` (`9`):**
     - Only allocated when decoder capture format is linear `NV12`. Each DPB buffer (`count = min_output_buf_count`) has the exact size of a UBWC `QC08C` (8-bit) or `QC10C` (10-bit) frame at `(W, H)` (see `03-v4l2-m2m-abi.md` Section 2.2), because the VPU hardware always reconstructs reference frames in UBWC format and uses its internal OPB hardware to convert to linear `NV12` simultaneously.

---

## 5. Session Lifecycle and HFI Packet Sequences

### 5.1 Decoder Session Sequence
- **[HW]** Complete step-by-step HFI Gen2 sequence for a decoding session:
  1. **Open Session:** Send a single HFI message (`session_id = inst_id`) containing 3 packets:
     - `HFI_CMD_SESSION_OPEN` (`0x01000002`, payload `1` = Decoder).
     - `HFI_PROP_CODEC` (`0x03000100`, payload `1` for H.264, `3` for HEVC, or `5` for VP9).
     - `HFI_PROP_COLOR_FORMAT` (`0x03000101`, port `RAW`, payload `1` for `NV12` or `2` for `QC08C`).
     Wait for the `HFI_CMD_SESSION_OPEN` response packet (`HFI_FW_FLAGS_SUCCESS`).
  2. **Configure Input (`BITSTREAM`) Port:** Send a multi-packet HFI message setting:
     - `HFI_PROP_BITSTREAM_RESOLUTION` (`0x03000103`, port `BITSTREAM`, `(width << 16) | height`).
     - `HFI_PROP_CROP_OFFSETS` (`0x03000105`, port `BITSTREAM`).
     - `HFI_PROP_LUMA_CHROMA_BIT_DEPTH` (`0x03000175`, port `BITSTREAM`, `0x00080008`).
     - `HFI_PROP_CODED_FRAMES` (`0x03000176`, port `BITSTREAM`, `1`).
     - `HFI_PROP_BUFFER_FW_MIN_OUTPUT_COUNT` (`0x0300017a`, port `BITSTREAM`).
     - `HFI_PROP_PIC_ORDER_CNT_TYPE` (`0x0300017e`, port `BITSTREAM`, `0`).
     - `HFI_PROP_PROFILE` (`0x03000107`) and `HFI_PROP_LEVEL` (`0x03000108`).
     - `HFI_PROP_SIGNAL_COLOR_INFO` (`0x03000115`, port `BITSTREAM`).
     - `HFI_PROP_STAGE` (`0x0300010a`, port `NONE`, `1` or `2`) and `HFI_PROP_PIPE` (`0x0300010b`, port `NONE`, `4`).
     - `HFI_PROP_DEC_DEFAULT_HEADER` (`0x03000117`, `0`) and `HFI_PROP_DEC_START_FROM_RAP_FRAME` (`0x03000118`, `1` or `0`).
     - `HFI_PROP_BUFFER_HOST_MAX_COUNT` (`0x03000179`, port `BITSTREAM`, `64`).
     - For H.264/HEVC: `HFI_PROP_COMV_BUFFER_COUNT` (`0x0300019e`, port `BITSTREAM`).
  3. **Configure Output (`RAW`) Port:** Send an HFI message setting:
     - `HFI_PROP_COLOR_FORMAT` (`0x03000101`, port `RAW`, `1` = `NV12`, `2` = `QC08C`, or `3` = `QC10C`).
     - `HFI_PROP_LINEAR_STRIDE_SCANLINE` (`0x03000104`, port `RAW`, only when format is linear `NV12`).
     - `HFI_PROP_BUFFER_HOST_MAX_COUNT` (`0x03000179`, port `RAW`, `64`).
  4. **Subscribe to Port Settings Changes and Properties:**
     - Send `HFI_CMD_SUBSCRIBE_MODE` (`0x0100000a`, port `BITSTREAM`, payload `[0x1, ...]` for `CMD_SETTINGS_CHANGE` on port properties).
     - Send `HFI_CMD_SUBSCRIBE_MODE` (`0x0100000a`, port `BITSTREAM` and `RAW`, payload `[0x2, HFI_PROP_PICTURE_TYPE, HFI_PROP_NO_OUTPUT, HFI_PROP_CABAC_SESSION]`).
  5. **Allocate and Register Internal Buffers:**
     - Allocate `PERSIST`, `BIN`, `COMV`, `NON_COMV`, `LINE`, and (if linear `NV12`) `DPB` buffers and send `HFI_CMD_BUFFER` (`0x01000009`) for each.
  6. **Start Streaming (`HFI_CMD_START`):**
     - Send `HFI_CMD_START` (`0x01000004`) on `HFI_PORT_BITSTREAM` (`1`) and `HFI_PORT_RAW` (`2`), waiting for `HFI_FW_FLAGS_SUCCESS` on each.
  7. **Queue Bitstream and Capture Buffers (`HFI_CMD_BUFFER`):**
     - Queue compressed bitstream buffers on `HFI_PORT_BITSTREAM` (`type = HFI_BUFFER_BITSTREAM = 1`, `data_size = payload_bytes`) and empty capture buffers on `HFI_PORT_RAW` (`type = HFI_BUFFER_RAW = 2`, `data_size = 0`).
  8. **Handle Dynamic Resolution Change (`HFI_CMD_SETTINGS_CHANGE`):**
     - When the first bitstream buffer (or an in-stream SPS change) is parsed by the firmware, firmware sends a message containing `HFI_CMD_SETTINGS_CHANGE` (`0x0100000b`) on `HFI_PORT_BITSTREAM` accompanied by updated property packets (`HFI_PROP_BITSTREAM_RESOLUTION`, `HFI_PROP_CROP_OFFSETS`, `HFI_PROP_LUMA_CHROMA_BIT_DEPTH`, `HFI_PROP_CODED_FRAMES`, `HFI_PROP_BUFFER_FW_MIN_OUTPUT_COUNT`, `HFI_PROP_PIC_ORDER_CNT_TYPE`, `HFI_PROP_PROFILE`, `HFI_PROP_LEVEL`, `HFI_PROP_SIGNAL_COLOR_INFO`).
     - Firmware flushes any remaining decoded frames from the old resolution and marks the final pre-change capture buffer with `HFI_BUF_FW_FLAG_PSC_LAST` (`0x02000000`) and `HFI_BUF_FW_FLAG_LAST` (`0x01000000`).
     - Host emits `V4L2_EVENT_SOURCE_CHANGE` (`V4L2_EVENT_SRC_CH_RESOLUTION`) to userspace. When userspace stops the capture queue, reallocates capture buffers for the new resolution, and restarts the capture queue:
       - Host reconfigures output (`RAW`) port properties (`COLOR_FORMAT`, `LINEAR_STRIDE_SCANLINE`), reallocates/updates internal `DPB`/`COMV`/`NON_COMV`/`LINE`/`BIN` buffers if their required sizes grew, sends `HFI_CMD_RESUME` (`0x01000008`) on `HFI_PORT_RAW`, and sends `HFI_CMD_START` (`0x01000004`) on `HFI_PORT_RAW`.
  9. **Stop and Close Session:**
     - Send `HFI_CMD_STOP` (`0x01000005`) on `HFI_PORT_BITSTREAM` and `HFI_PORT_RAW` and wait for response.
     - Release internal buffers via `HFI_CMD_BUFFER` with `HFI_BUF_HOST_FLAG_RELEASE` (`0x1`).
     - Send `HFI_CMD_SESSION_CLOSE` (`0x01000003`) and wait for response.
