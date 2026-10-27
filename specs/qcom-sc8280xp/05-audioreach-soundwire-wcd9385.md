# Qualcomm SC8280XP Audio Subsystem: GPR, AudioReach (APM/PRM), LPASS Macros, SoundWire, and WCD9385 Codec

## 1. Overview & Audio Topology

**[HW]** On the Qualcomm SC8280XP (Radxa Dragon Q8B), audio processing is split between the Hexagon Low Power Audio Subsystem (`LPASS` / `ADSP`, remote processor ID `2`) running the Signal Processing Framework (`SPF` / `AudioReach`) firmware (`qcom/sc8280xp/qcadsp8280.mbn`) and host-accessible MMIO blocks in the `LPASS` peripheral region (`0x03200000`–`0x03560000`).

**[HW]** The end-to-end audio hardware topology consists of:
1. **Generic Packet Router (`GPR`)**: An IPC framing layer over the `adsp_apps` GLINK channel (`IPCC_CLIENT_LPASS = 3`, `IPCC_MPROC_SIGNAL_GLINK_QMP = 0`) used to exchange command, response, and data packets with `SPF` services on the ADSP (`domain_id = 2`).
2. **Audio Process Manager (`APM`, service ID `1`) & Proxy Resource Manager (`PRM`, service ID `2`)**: Firmware services inside `SPF` that manage dynamic audio processing graphs, shared DMA buffer mappings, LPASS hardware core power votes, and LPASS root clocks.
3. **Apps SMMU Audio Context Bank (`SID = 0x0c01`)**: Translates host DRAM PCM buffers for the ADSP DMA engine; the ADSP requires the lower 4 bits of the Stream ID (`0x1`) to be ORed into bits `[35:32]` of every 64-bit IOVA passed in `APM` shared-memory map and data commands (`tagged_iova = iova | (1 << 32)`).
4. **LPASS Digital Codec Macros (`RX Macro`, `TX Macro`, `WSA Macro`, `VA Macro`) & LPI TLMM (`0x033c0000`)**: Host-mapped digital decimator/interpolator and clock-divider blocks that bridge `SPF` hardware endpoints (`CODEC_DMA_SINK` / `CODEC_DMA_SOURCE`) to the SoundWire master controllers.
5. **Qualcomm SoundWire Master Controllers (`SWR0` WSA at `0x03250000`, `SWR1` RX at `0x03210000`, `SWR2` TX at `0x03330000`)**: SoundWire v1.6.0 (`0x01060000`) controllers driving the 2-wire SoundWire links (`CLK` + `DATA`) to external codecs and amplifiers.
6. **WCD9385 Analog Audio Codec**: Dual-link SoundWire peripheral (RX slave on `SWR1`, TX slave on `SWR2`) providing headphone (`HPHL`/`HPHR`) Class-H DACs, microphone ADCs (`ADC1`–`ADC4`), and Multi-Button Headset Control (`MBHC`) jack/button/impedance detection.

---

## 2. Generic Packet Router (GPR) Framing Protocol

**[HW]** `GPR` packets are transported over the native GLINK channel named `"adsp_apps"` on the LPASS GLINK edge. The GLINK channel must be pre-populated with at least `20` RX intents of `512` bytes each so the ADSP can push asynchronous command responses and buffer completion events.

### 2.1 GPR Packet Header (`20` bytes, Little-Endian)

**[HW]** Every GPR packet begins with a 20-byte (5 × 32-bit word) header followed immediately by `pkt_size - 20` bytes of command or response payload:

| Byte Offset | Bits | Field | Description |
|---|---|---|---|
| `0x00` | `3:0` | `version` | Protocol version (`0x0`) |
| `0x00` | `7:4` | `hdr_size` | Header size in 32-bit words (`5` = 20 bytes) |
| `0x00` | `31:8` | `pkt_size` | Total packet length in bytes, including the 20-byte header |
| `0x04` | `7:0` | `dest_domain` | Destination domain ID (`0x02` = `GPR_DOMAIN_ID_ADSP`, `0x03` = `GPR_DOMAIN_ID_APPS`) |
| `0x04` | `15:8` | `src_domain` | Source domain ID (`0x03` = `GPR_DOMAIN_ID_APPS` when sent by host) |
| `0x04` | `31:16` | `reserved` | Reserved (`0x0000`) |
| `0x08` | `31:0` | `src_port` | Source instance/port ID (host client port or graph ID) |
| `0x0C` | `31:0` | `dest_port` | Destination module/service instance ID on the ADSP |
| `0x10` | `31:0` | `token` | Caller-supplied transaction token echoed in the response packet |
| `0x14` | `31:0` | `opcode` | 32-bit GPR/APM/PRM command, response, data, or event opcode |

### 2.2 Static Service Instance IDs and Basic Responses

**[HW]** Well-known static module instance IDs (`dest_port`) on the ADSP (`domain_id = 2`):

| Instance ID | Name | Role |
|---|---|---|
| `0x00000001` | `APM_MODULE_INSTANCE_ID` | Audio Process Manager (graph lifecycle, memory mapping, SPF state) |
| `0x00000002` | `PRM_MODULE_INSTANCE_ID` | Proxy Resource Manager (LPASS core power & audio clock voting) |
| `0x00000003` | `AMDB_MODULE_INSTANCE_ID` | Audio Module Database |
| `0x00000004` | `VCPM_MODULE_INSTANCE_ID` | Voice Call Process Manager |
| `0x00004000`+ | `AR_SG_INSTANCE_ID_START` | Sub-graph instance IDs |
| `0x00005000`–`0x00005FFF` | `AR_CONT_INSTANCE_ID` | Container instance IDs |
| `0x00006000`–`0x00006FFF` | `AR_MODULE_INSTANCE_ID` | Static module instance IDs within graphs |

**[HW]** Standard GPR response opcodes returned by the ADSP:
- `GPR_BASIC_RSP_RESULT` (`0x02001005`): 8-byte payload containing:
  - Offset `0x00` (`u32`): `opcode` (the original command opcode being acknowledged)
  - Offset `0x04` (`u32`): `status` (`0` = success, non-zero = ADSP error code)
- `GPR_BASIC_EVT_ACCEPTED` (`0x02001006`): 4-byte payload containing `opcode` (`u32`).

### 2.3 Common APM Command & Parameter Envelopes

**[HW]** Commands that carry configuration blocks (`APM_CMD_GRAPH_OPEN`, `APM_CMD_SET_CFG`, `PRM_CMD_REQUEST_HW_RSC`, etc.) append a 16-byte `apm_cmd_header` immediately after the 20-byte `gpr_hdr`, followed by one or more 8-byte-aligned `apm_module_param_data` blocks:

**[HW]** **APM Command Header (`16` bytes at packet offset `0x14`)**:

| Offset | Type | Field | Description |
|---|---|---|---|
| `0x00` | `u32` | `payload_address_lsw` | `0` for in-band payload (carried inside the GPR packet) |
| `0x04` | `u32` | `payload_address_msw` | `0` for in-band payload |
| `0x08` | `u32` | `mem_map_handle` | `0` for in-band payload |
| `0x0C` | `u32` | `payload_size` | Total byte length of all parameter blocks following this header |

**[HW]** **Module Parameter Data Header (`16` bytes)**:

| Offset | Type | Field | Description |
|---|---|---|---|
| `0x00` | `u32` | `module_instance_id` | Target module instance ID (`0x1` for APM, `0x2` for PRM, or `0x6000`+ for graph modules) |
| `0x04` | `u32` | `param_id` | Parameter ID (`0x0800xxxx`) |
| `0x08` | `u32` | `param_size` | Size in bytes of the parameter payload following this 16-byte header |
| `0x0C` | `u32` | `error_code` | `0` on request; filled by ADSP on error |

---

## 3. Proxy Resource Manager (PRM) Protocol (`dest_port = 0x00000002`)

**[HW]** Before accessing host-mapped LPASS digital macro registers (`0x03200000`–`0x033c0000`) or starting SoundWire links, the host must vote for LPASS hardware cores and clocks via `PRM` over GPR.

### 3.1 PRM Opcodes and Parameter IDs

| Symbol | Value | Description |
|---|---|---|
| `PRM_CMD_REQUEST_HW_RSC` | `0x0100100F` | Request/enable an LPASS hardware core or clock |
| `PRM_CMD_RSP_REQUEST_HW_RSC` | `0x02001002` | Response to `PRM_CMD_REQUEST_HW_RSC` |
| `PRM_CMD_RELEASE_HW_RSC` | `0x01001010` | Release/disable an LPASS hardware core or clock |
| `PRM_CMD_RSP_RELEASE_HW_RSC` | `0x02001003` | Response to `PRM_CMD_RELEASE_HW_RSC` |
| `PARAM_ID_RSC_LPASS_CORE` | `0x0800102B` | LPASS core GDSC vote |
| `PARAM_ID_RSC_AUDIO_HW_CLK` | `0x0800102C` | LPASS audio clock frequency & attribute vote |
| `PARAM_ID_RSC_HW_CORE` | `0x08001032` | LPASS macro (`LPASS_HW_MACRO_VOTE = 100`) or digital codec (`LPASS_HW_DCODEC_VOTE = 101`) block vote |
| `PARAM_ID_RSC_CPU_LPR` | `0x08001A6E` | CPU Low-Power Resource sleep disable vote (`0x1`) |

### 3.2 Hardware Core Vote Payload (`PARAM_ID_RSC_HW_CORE` = `0x08001032`)

**[HW]** Follows `gpr_hdr` (20B) + `apm_cmd_header` (16B) + `apm_module_param_data` (16B, `param_size = 4`):
- Offset `0x00` (`u32`): `hw_clk_id` (`100` = `LPASS_HW_MACRO_VOTE`, `101` = `LPASS_HW_DCODEC_VOTE`).

### 3.3 Audio Clock Request/Release Payloads (`PARAM_ID_RSC_AUDIO_HW_CLK` = `0x0800102C`)

**[HW]** **Clock Request (`PRM_CMD_REQUEST_HW_RSC`, `param_size = 20`)**:

| Offset | Type | Field | Description |
|---|---|---|---|
| `0x00` | `u32` | `num_clk_id` | Number of clock descriptors (`1`) |
| `0x04` | `u32` | `clock_id` | Clock ID: `0x307` = `RX_CORE_TX_MCLK`, `0x308` = `RX_CORE_TX_2X_MCLK`, `0x305` = `WSA_CORE_TX_MCLK`, `0x306` = `WSA_CORE_TX_2X_MCLK`, `0x309` = `TX_CORE_MCLK`, `0x30a` = `TX_CORE_NPL_MCLK`, `0x30b` = `VA_CORE_MCLK`, `0x30c` = `VA_CORE_2X_MCLK` |
| `0x08` | `u32` | `clock_freq` | Requested frequency in Hz (typically `19200000` for LPASS macro MCLK/NPL) |
| `0x0C` | `u32` | `clock_attri` | Clock coupling attribute (`1` = `LPASS_CLK_ATTRIBUTE_COUPLE_NO`) |
| `0x10` | `u32` | `clock_root` | Clock root source (`0` = default) |

**[HW]** **Clock Release (`PRM_CMD_RELEASE_HW_RSC`, `param_size = 8`)**:
- Offset `0x00` (`u32`): `num_clk_id` (`1`)
- Offset `0x04` (`u32`): `clock_id`

---

## 4. AudioReach Graph Management & Shared-Memory DMA (`APM`)

### 4.1 Apps SMMU Stream `0x0c01` High-Nibble Address Tagging

**[HW]** The host PCM/compressed audio frontend (`q6apm-dais`) is attached to the Apps SMMU (`0x15000000`) with Stream ID `0x0c01` (`mask = 0x0`).
- When the ADSP initiates DMA reads or writes to host DRAM through the LPASS/Apps SMMU bridge, the hardware uses bits `[35:32]` of the 64-bit address on the ADSP bus to select the SMMU Stream ID low nibble (`sid = 0x0c01 & 0xF = 0x1`).
- Consequently, every DMA IOVA (`iova`, allocated in the 32-bit range `0x00000000`–`0xFFFFFFFF`) passed to the ADSP in `APM_CMD_SHARED_MEM_MAP_REGIONS`, `DATA_CMD_WR_SH_MEM_EP_DATA_BUFFER_V2`, or `DATA_CMD_RD_SH_MEM_EP_DATA_BUFFER_V2` MUST be tagged as:
  - `dsp_addr = iova | (1 << 32)` (i.e., `shm_addr_lsw = iova & 0xFFFFFFFF`, `shm_addr_msw = 0x00000001`).
- Omitting `shm_addr_msw = 0x00000001` causes the ADSP DMA access to fault on Stream ID `0x0c00` instead of `0x0c01`.

### 4.2 APM Opcodes (`dest_port = 0x00000001`)

| Opcode | Name | Direction | Description |
|---|---|---|---|
| `0x01001021` | `APM_CMD_GET_SPF_STATE` | Host -> ADSP | Query whether SPF firmware is ready |
| `0x02001007` | `APM_CMD_RSP_GET_SPF_STATE` | ADSP -> Host | 4-byte response: `state` (`1` = ready, `0` = not ready) |
| `0x01001000` | `APM_CMD_GRAPH_OPEN` | Host -> ADSP | Instantiate sub-graphs, containers, modules, and connections |
| `0x01001001` | `APM_CMD_GRAPH_PREPARE` | Host -> ADSP | Prepare sub-graphs for execution |
| `0x01001002` | `APM_CMD_GRAPH_START` | Host -> ADSP | Start streaming on prepared sub-graphs |
| `0x01001003` | `APM_CMD_GRAPH_STOP` | Host -> ADSP | Stop streaming on sub-graphs |
| `0x01001004` | `APM_CMD_GRAPH_CLOSE` | Host -> ADSP | Tear down sub-graphs and free ADSP resources |
| `0x01001005` | `APM_CMD_GRAPH_FLUSH` | Host -> ADSP | Flush queued buffers in sub-graphs |
| `0x01001006` | `APM_CMD_SET_CFG` | Host -> ADSP | Configure module parameters (media format, gain, DMA interface) |
| `0x0100100C` | `APM_CMD_SHARED_MEM_MAP_REGIONS` | Host -> ADSP | Map 4 KB-aligned host IOVA region into ADSP shared memory pool |
| `0x02001001` | `APM_CMD_RSP_SHARED_MEM_MAP_REGIONS` | ADSP -> Host | 4-byte response: `mem_map_handle` (`u32`) |
| `0x0100100D` | `APM_CMD_SHARED_MEM_UNMAP_REGIONS` | Host -> ADSP | Unmap previously returned `mem_map_handle` (`u32`) |

### 4.3 Graph Open Topology (`APM_CMD_GRAPH_OPEN` = `0x01001000`)

**[HW]** `APM_CMD_GRAPH_OPEN` sends an `apm_cmd_header` followed by 8-byte-aligned `apm_module_param_data` blocks targeting `module_instance_id = 1` (`APM`) with the following parameter IDs in order:
1. `APM_PARAM_ID_SUB_GRAPH_CONFIG` (`0x08001001`): Declares `num_sub_graphs` and for each `sub_graph_id` its 3 properties:
   - `APM_SUB_GRAPH_PROP_ID_PERF_MODE` (`0x0800100E`): `1` (Low Power) or `2` (Low Latency)
   - `APM_SUB_GRAPH_PROP_ID_DIRECTION` (`0x0800100F`): `1` (RX/Playback) or `2` (TX/Capture)
   - `APM_SUB_GRAPH_PROP_ID_SCENARIO_ID` (`0x08001010`): `1` (Audio Playback), `2` (Audio Record), `3` (Voice Call)
2. `APM_PARAM_ID_CONTAINER_CONFIG` (`0x08001000`): Declares `num_containers` and for each `container_id` its properties:
   - `APM_CONTAINER_PROP_ID_CAPABILITY_LIST` (`0x08001011`): capability ID (`1` = `PP`)
   - `APM_CONTAINER_PROP_ID_GRAPH_POS` (`0x08001012`): stream/per-stream/global position
   - `APM_CONTAINER_PROP_ID_STACK_SIZE` (`0x08001013`): container thread stack size in bytes (e.g., `2048` or `4096`)
   - `APM_CONTAINER_PROP_ID_PROC_DOMAIN` (`0x08001014`): `1` (`ADSP`)
3. `APM_PARAM_ID_MODULE_LIST` (`0x08001002`): Maps `(module_id, instance_id)` pairs to each `(sub_graph_id, container_id)`:
   - `0x07001000` (`MODULE_ID_WR_SHARED_MEM_EP`): Host -> ADSP shared-memory playback endpoint
   - `0x07001001` (`MODULE_ID_RD_SHARED_MEM_EP`): ADSP -> Host shared-memory capture endpoint
   - `0x07001002` (`MODULE_ID_GAIN`): Digital gain module (`APM_PARAM_ID_GAIN` = `0x08001006`)
   - `0x07001003` (`MODULE_ID_PCM_CNV`): PCM format converter (`PARAM_ID_PCM_OUTPUT_FORMAT_CFG` = `0x08001008`)
   - `0x07001004` / `0x07001005` (`MODULE_ID_PCM_ENC` / `MODULE_ID_PCM_DEC`): PCM encoder/decoder
   - `0x07001015` (`MODULE_ID_MFC`): Media Format Converter (sample-rate/channel converter, `0x08001024`)
   - `0x07001023` (`MODULE_ID_CODEC_DMA_SINK`): LPASS Codec DMA playback sink (`PARAM_ID_CODEC_DMA_INTF_CFG` = `0x08001063`, `lpaif_type`: `1` = RXTX, `2` = WSA, `3` = VA)
   - `0x07001024` (`MODULE_ID_CODEC_DMA_SOURCE`): LPASS Codec DMA capture source
   - `0x07001069` (`MODULE_ID_DISPLAY_PORT_SINK`): DisplayPort audio sink (`PARAM_ID_DISPLAY_PORT_INTF_CFG` = `0x08001154`)
4. `APM_PARAM_ID_MODULE_PROP` (`0x08001003`): Sets `APM_MODULE_PROP_ID_PORT_INFO` (`0x08001015`, `max_ip_port` and `max_op_port`) for each module instance.
5. `APM_PARAM_ID_MODULE_CONN` (`0x08001004`): Declares directed edges `(src_mod_inst_id, src_mod_op_port_id, dst_mod_inst_id, dst_mod_ip_port_id)`.

### 4.4 Shared Memory Mapping & Data Buffer Streaming

**[HW]** **Memory Map Command (`APM_CMD_SHARED_MEM_MAP_REGIONS` = `0x0100100C`)**:
- Sent directly after `gpr_hdr` (no `apm_cmd_header`):
  - `mem_pool_id` (`u16` at `0x00`): `3` (`APM_MEMORY_MAP_SHMEM8_4K_POOL`)
  - `num_regions` (`u16` at `0x02`): `1`
  - `property_flag` (`u32` at `0x04`): `0x0` for normal DMA buffer (`0x2` for position buffer)
  - `shm_addr_lsw` (`u32` at `0x08`): `iova & 0xFFFFFFFF` (must be 4 KB aligned)
  - `shm_addr_msw` (`u32` at `0x0C`): `0x00000001` (Stream ID `0x0c01` low nibble)
  - `mem_size_bytes` (`u32` at `0x10`): Buffer size aligned up to a multiple of `4096` bytes
- Response `APM_CMD_RSP_SHARED_MEM_MAP_REGIONS` (`0x02001001`) returns `mem_map_handle` (`u32`).

**[HW]** **Playback Data Write (`DATA_CMD_WR_SH_MEM_EP_DATA_BUFFER_V2` = `0x0400100A`)**:
- Sent to `dest_port = wr_sh_mem_ep_instance_id` (44-byte payload after `gpr_hdr`):
  - `buf_addr_lsw` (`u32` at `0x00`): Period IOVA low 32 bits
  - `buf_addr_msw` (`u32` at `0x04`): `0x00000001`
  - `mem_map_handle` (`u32` at `0x08`): Handle from `APM_CMD_RSP_SHARED_MEM_MAP_REGIONS`
  - `buf_size` (`u32` at `0x0C`): Period length in bytes
  - `timestamp_lsw` (`u32` at `0x10`), `timestamp_msw` (`u32` at `0x14`), `flags` (`u32` at `0x18`), `md_addr_lsw` (`0`), `md_addr_msw` (`0`), `md_map_handle` (`0`), `md_buf_size` (`0`).
- ADSP replies with `DATA_CMD_RSP_WR_SH_MEM_EP_DATA_BUFFER_DONE_V2` (`0x05001004`) when the period has been consumed.

**[HW]** **Capture Data Read (`DATA_CMD_RD_SH_MEM_EP_DATA_BUFFER_V2` = `0x0400100B`)**:
- Sent to `dest_port = rd_sh_mem_ep_instance_id` (32-byte payload after `gpr_hdr`):
  - `buf_addr_lsw` (`u32` at `0x00`), `buf_addr_msw` (`u32` at `0x04` = `0x00000001`), `mem_map_handle` (`u32` at `0x08`), `buf_size` (`u32` at `0x0C`), followed by 4 zero words for metadata.
- ADSP replies with `DATA_CMD_RSP_RD_SH_MEM_EP_DATA_BUFFER_V2` (`0x05001005`) when the capture period is filled.

---

## 5. LPASS Digital Codec Macros (RX, TX, WSA, VA) & LPI TLMM

**[HW]** Physical base addresses of the SC8280XP LPASS blocks:

| Block | Base Address | Size | Clocks / Votes (`q6prmcc`) | Description |
|---|---|---|---|---|
| `rxmacro` | `0x03200000` | `0x1000` | `RX_CORE_TX_MCLK` (`19.2 MHz`), `RX_CORE_TX_2X_MCLK` (`19.2 MHz`), `MACRO_VOTE`, `DCODEC_VOTE`, `fsgen` | Headphone/earpiece digital interpolators (`RX0`..`RX2`), companders (`COMP0`/`COMP1`), Class-H controller, sidetone |
| `swr1` (RX) | `0x03210000` | `0x2000` | `rxmacro` MCLK, reset `LPASS_AUDIO_SWR_RX_CGCR`, GIC SPI `155` | SoundWire v1.6.0 RX Master (`5` DOUT ports, `0` DIN ports) |
| `txmacro` | `0x03220000` | `0x1000` | `TX_CORE_MCLK` (`19.2 MHz`), `TX_CORE_NPL_MCLK` (`19.2 MHz`), `MACRO_VOTE`, `DCODEC_VOTE`, `fsgen` | Microphone digital decimators (`DEC0`..`DEC7`) for SoundWire TX and DMICs |
| `wsamacro` | `0x03240000` | `0x1000` | `WSA_CORE_TX_MCLK` (`19.2 MHz`), `WSA_CORE_TX_2X_MCLK` (`19.2 MHz`), `MACRO_VOTE`, `DCODEC_VOTE`, `fsgen` | Speaker amplifier interpolators and VI-sense feedback decimators |
| `swr0` (WSA) | `0x03250000` | `0x2000` | `wsamacro` MCLK, reset `LPASS_AUDIO_SWR_WSA_CGCR`, GIC SPI `170` | SoundWire v1.6.0 WSA Master (`6` DOUT ports, `2` DIN ports) |
| `lpass_audiocc` | `0x032a9000` | `0x1000` | Host MMIO clock/reset controller | Provides `LPASS_AUDIO_SWR_RX_CGCR` and `LPASS_AUDIO_SWR_WSA_CGCR` resets |
| `swr2` (TX) | `0x03330000` | `0x2000` | `txmacro` MCLK, reset `LPASS_AUDIO_SWR_TX_CGCR`, GIC SPI `959` (core) / `520` (wakeup) | SoundWire v1.6.0 TX Master (`0` DOUT ports, `4` DIN ports) |
| `vamacro` | `0x03370000` | `0x1000` | `TX_CORE_MCLK` (`19.2 MHz`), `TX_CORE_NPL_MCLK`, `MACRO_VOTE`, `DCODEC_VOTE` | Voice-Activation / DMIC decimators (`DEC0`..`DEC3`) and shared `fsgen` frame-sync generator |
| `lpass_tlmm` | `0x033c0000` (CFG), `0x03550000` (SLEW) | `0x20000`, `0x10000` | `MACRO_VOTE`, `DCODEC_VOTE` | 19-pin LPI pinctrl (`gpio0`=`swr_tx_clk`, `gpio1..2`=`swr_tx_data`, `gpio3`=`swr_rx_clk`, `gpio4..5`=`swr_rx_data`, `gpio6..9`=`dmic1..2`, `gpio10..12`=`wsa_swr`) |

### 5.1 RX Macro (`0x03200000`, LPASS Codec v2.5) Key Registers

**[HW]** On SC8280XP (LPASS Codec version 2.5), the RX Macro must enable its MCLK and SoundWire clock before `swr1` (`0x03210000`) is accessed:

| Offset | Register | Key Bits & Description |
|---|---|---|
| `0x0000` | `CDC_RX_TOP_TOP_CFG0` | Top configuration |
| `0x0008` | `CDC_RX_TOP_SWR_CTRL` | SoundWire interface control |
| `0x0100` | `CDC_RX_CLK_RST_CTRL_MCLK_CONTROL` | Bit `0` = `MCLK_ENABLE` (`1` = enable 9.6/19.2 MHz MCLK), Bit `1` = `MCLK2_ENABLE` |
| `0x0104` | `CDC_RX_CLK_RST_CTRL_FS_CNT_CONTROL` | Bit `0` = `FS_MCLK_CNT_ENABLE`, Bit `1` = `FS_MCLK_CNT_CLR` |
| `0x0108` | `CDC_RX_CLK_RST_CTRL_SWR_CONTROL` | Bit `0` = `SWR_CLK_EN` (supplies `iface` clock to `swr1`), Bit `1` = `SWR_RESET` |
| `0x0180` | `CDC_RX_INP_MUX_RX_INT0_CFG0` | Bits `[3:0]` = `INT0_1_MIX_INP0_SEL`, Bits `[7:4]` = `INT0_1_MIX_INP1_SEL` (`RX0`..`RX2` interpolator input mux from `AIF1_PB`..`AIF4_PB`) |
| `0x0188` | `CDC_RX_INP_MUX_RX_INT1_CFG0` | `INT1` interpolator input mux (`HPHR` channel) |
| `0x0200` | `CDC_RX_CLSH_CRC` | Bit `0` = `CLSH_CLK_EN` (Class-H power supply tracker clock enable) |
| `0x0400` | `CDC_RX_RX0_RX_PATH_CTL` | Bit `6` = `RESET_EN`, Bit `5` = `CLK_EN`, Bit `4` = `PGA_MUTE`, Bits `[3:0]` = `PCM_RATE` (`4` = 48 kHz) |
| `0x0404` | `CDC_RX_RX0_RX_PATH_CFG0` | Bit `6` = `CLSH_EN`, Bit `3` = `DLY_ZN_EN`, Bit `2` = `HD2_EN`, Bit `1` = `COMP_EN` (Compander 0 enable) |
| `0x0414` | `CDC_RX_RX0_RX_VOL_CTL` | 8-bit signed digital volume (`0x00` = 0 dB, 1 dB/step) |
| `0x04C0` | `CDC_2_5_RX_RX1_RX_PATH_CTL` | `RX1` (`HPHR`) path control (note v2.5 stride `0xC0` between `RX0` at `0x400`, `RX1` at `0x4C0`, `RX2` at `0x580`) |
| `0x0800` | `CDC_RX_COMPANDER0_CTL0` | Bit `0` = `CLK_EN`, Bit `1` = `SOFT_RST`, Bit `2` = `HALT` (`HPHL` compander; `COMPANDER1` at `0x0840` for `HPHR`) |

### 5.2 VA Macro (`0x03370000`) & TX Macro (`0x03220000`) Key Registers

**[HW]** `vamacro` (`0x03370000`) provides the master frame-sync clock (`fsgen`) to `rxmacro`, `txmacro`, and `wsamacro`, as well as DMIC/SoundWire TX decimators:

| Offset | Register | Key Bits & Description |
|---|---|---|
| `0x0000` | `CDC_VA_CLK_RST_CTRL_MCLK_CONTROL` | Bit `0` = `MCLK_CONTROL_EN` (must be set to `1` to enable `fsgen`) |
| `0x0004` | `CDC_VA_CLK_RST_CTRL_FS_CNT_CONTROL` | Bit `0` = `FS_CONTROL_EN` (enables `fsgen` output), Bit `1` = `FS_COUNTER_CLR` |
| `0x0008` | `CDC_VA_CLK_RST_CTRL_SWR_CONTROL` | Bit `0` = `SWR_CLK_EN`, Bit `1` = `SWR_RESET` (in `txmacro` at `0x03220008`, supplies `iface` clock to `swr2`) |
| `0x0080` | `CDC_VA_TOP_CSR_TOP_CFG0` | Bit `1` = `FS_BROADCAST_EN` |
| `0x0084`–`0x0090` | `CDC_VA_TOP_CSR_DMIC0_CTL`..`DMIC3_CTL` | Bit `0` = `DMIC_EN`, Bits `[3:1]` = `DMIC_CLK_SEL` divider |
| `0x0100`+`8*n` | `CDC_VA_INP_MUX_ADC_MUXn_CFG0/1` | Selects DMIC or SoundWire TX ADC input for decimator `n` (`0..3`) |
| `0x0400`+`0x80*n` | `CDC_VA_TXn_TX_PATH_CTL` | Bit `5` = `CLK_EN`, Bit `4` = `PGA_MUTE_EN`, Bits `[3:0]` = `PCM_RATE` |
| `0x040C`+`0x80*n` | `CDC_VA_TXn_TX_VOL_CTL` | Decimator `n` digital gain |

---

## 6. Qualcomm SoundWire Master Controller (`v1.6.0`)

**[HW]** All three SoundWire controllers (`swr0` at `0x03250000`, `swr1` at `0x03210000`, `swr2` at `0x03330000`) implement `qcom,soundwire-v1.6.0` (using the v1.3 register layout with `default_rows = 50`, `default_cols = 16`, and software clock gating required).

### 6.1 SoundWire Master Register Map (Offsets from Controller Base)

| Offset | Register | Access | Description |
|---|---|---|---|
| `0x0000` | `SWRM_COMP_HW_VERSION` | RO | Hardware version (`0x01060000` / v1.6.0) |
| `0x0004` | `SWRM_COMP_CFG_ADDR` | RW | Bit `0` = `ENABLE`, Bit `1` = `IRQ_LEVEL_OR_PULSE` |
| `0x0008` | `SWRM_COMP_SW_RESET` | WO | Write `0x1` to soft-reset the controller |
| `0x0014` | `SWRM_COMP_STATUS` | RO | Bit `0` = `FRM_GEN_ENABLED` (`1` when frame generator is running) |
| `0x0018` | `SWRM_LINK_MANAGER_EE` | RW | Execution environment (`1` = `SWRM_EE_CPU`) |
| `0x0100` | `SWRM_COMP_PARAMS` | RO | `[4:0]` = `DOUT_PORTS`, `[9:5]` = `DIN_PORTS`, `[14:10]` = `WR_FIFO_DEPTH`, `[19:15]` = `RD_FIFO_DEPTH` |
| `0x0200` | `SWRM_V1_3_INTERRUPT_STATUS` | RO | 17-bit interrupt status (`[0]` = `SLAVE_PEND_IRQ`, `[1]` = `NEW_SLAVE_ATTACHED`, `[2]` = `CHANGE_ENUM_SLAVE_STATUS`, `[10]` = `SPECIAL_CMD_ID_FINISHED`) |
| `0x0204` | `SWRM_V1_3_INTERRUPT_MASK_ADDR` | RW | Interrupt mask register |
| `0x0208` | `SWRM_V1_3_INTERRUPT_CLEAR` | WO | Write-1-to-clear interrupt status bits |
| `0x0210` | `SWRM_V1_3_INTERRUPT_CPU_EN` | RW | Enable delivery of unmasked interrupts to CPU GIC SPI |
| `0x0300` | `SWRM_V1_3_CMD_FIFO_WR_CMD` | WO | Packed 32-bit SoundWire register write command: `(data << 24) \| (dev_num << 20) \| (cmd_id << 16) \| reg_addr` |
| `0x0304` | `SWRM_V1_3_CMD_FIFO_RD_CMD` | WO | Packed 32-bit SoundWire register read command: `(dev_num << 20) \| (cmd_id << 16) \| reg_addr` |
| `0x0308` | `SWRM_CMD_FIFO_CMD` | WO | Write `0x1` to flush command FIFO |
| `0x030C` | `SWRM_V1_3_CMD_FIFO_STATUS` | RO | `[12:8]` = `WR_CMD_FIFO_CNT`, `[20:16]` = `RD_CMD_FIFO_CNT` |
| `0x0314` | `SWRM_CMD_FIFO_CFG_ADDR` | RW | Bit `31` = `CONTINUE_EXEC_ON_CMD_IGNORE`, Bits `[2:0]` = `RD_WR_CMD_RETRIES` (`0x7`) |
| `0x0318` | `SWRM_V1_3_CMD_FIFO_RD_FIFO_ADDR` | RO | `[7:0]` = read data byte, `[11:8]` = `CMD_ID` |
| `0x0500` | `SWRM_ENUMERATOR_CFG_ADDR` | RW | Auto-enumeration control (`0x1` = enable auto-enumeration) |
| `0x0530`+`8*m` | `SWRM_ENUMERATOR_SLAVE_DEV_ID_1(m)` | RO | Lower 32 bits of 48-bit SoundWire Device ID for logical device `m` (`1..11`) |
| `0x0534`+`8*m` | `SWRM_ENUMERATOR_SLAVE_DEV_ID_2(m)` | RO | Upper 16 bits of 48-bit SoundWire Device ID for logical device `m` |
| `0x101C`+`0x40*b` | `SWRM_MCP_FRAME_CTRL_BANK_ADDR(b)` | RW | Bank `b` (`0` or `1`) frame shape: `[2:0]` = `COL_CTRL`, `[7:3]` = `ROW_CTRL` |
| `0x1044` | `SWRM_MCP_BUS_CTRL` | RW | Bit `1` = `BUS_CLK_START` |
| `0x1048` | `SWRM_MCP_CFG_ADDR` | RW | `[21:17]` = `MAX_NUM_OF_CMD_NO_PINGS` (`0x1f`) |
| `0x104C` | `SWRM_MCP_STATUS` | RO | Bit `0` = active bank number (`0` or `1`) |
| `0x1090` | `SWRM_MCP_SLV_STATUS` | RO | 2 bits per slave (`m = 0..11`): `0` = not present, `1` = attached ok, `2` = alert, `3` = reserved |
| `0x1124`+`0x100*(n-1)`+`0x40*b` | `SWRM_DPn_PORT_CTRL_BANK(n, b)` | RW | Data Port `n` (`1..14`), Bank `b`: `[7:0]` = `SINTERVAL_LOW`, `[15:8]` = `OFFSET1`, `[23:16]` = `OFFSET2`, `[31:24]` = `EN_CHAN` mask |
| `0x112C`+`0x100*(n-1)` | `SWRM_DPn_BLOCK_CTRL_1(n)` | RW | Data Port `n` word length (`bits_per_sample - 1`) |
| `0x1134`+`0x100*(n-1)`+`0x40*b` | `SWRM_DPn_PORT_HCTRL_BANK(n, b)` | RW | `[3:0]` = `HSTART`, `[7:4]` = `HSTOP` |

### 6.2 Bank Switch Sequence

**[HW]** All SoundWire data port (`DPn`) and frame-control changes are staged in the inactive bank (`next_bank = !(read(SWRM_MCP_STATUS) & 1)`). Once programmed, the host issues a broadcast write (`dev_num = 0x0F`, `cmd_id = 0x0F`) to standard SoundWire `SDW_SCP_FRAMECTRL_B0`/`B1` (`0x0060` / `0x0070`) via `SWRM_V1_3_CMD_FIFO_WR_CMD` (`0x0300`) and waits for `SPECIAL_CMD_ID_FINISHED` (bit 10 of `SWRM_V1_3_INTERRUPT_STATUS`).

---

## 7. WCD9385 Codec & MBHC Jack-Detection Sequence

**[HW]** The WCD9385 (`CHIPID_WCD9385 = 0x5`) exposes two SoundWire slave interfaces (`48-bit` SoundWire Enumeration IDs with Manufacturer `0x0217`, Part `0x010d`):
1. **WCD9385 RX Slave** on `swr1` (`0x03210000`): Receives playback audio streams AND hosts the AHB register bridge used to access all WCD9385 analog/digital control registers.
2. **WCD9385 TX Slave** on `swr2` (`0x03330000`): Transmits microphone (`ADC1`–`ADC4`) and MBHC capture streams.

### 7.1 WCD9385 Register Access via RX SoundWire AHB Bridge

**[HW]** Host register reads and writes to the WCD9385 16-bit paged register space (`page = reg >> 8`, `offset = reg & 0xFF`) are tunneled over the `swr1` RX SoundWire bus using four 4-byte SoundWire slave registers on the WCD9385 RX device:
- **Write (`reg`, `val`)**:
  1. Write 4 bytes (`val` little-endian) to SoundWire slave register `SWRM_AHB_BRIDGE_WR_DATA_0` (`0x0C85`).
  2. Write 4 bytes (`reg` little-endian) to SoundWire slave register `SWRM_AHB_BRIDGE_WR_ADDR_0` (`0x0C89`).
- **Read (`reg` -> `val`)**:
  1. Write 4 bytes (`reg` little-endian) to SoundWire slave register `SWRM_AHB_BRIDGE_RD_ADDR_0` (`0x0C8D`).
  2. Read 4 bytes (`val` little-endian) from SoundWire slave register `SWRM_AHB_BRIDGE_RD_DATA_0` (`0x0C91`).

### 7.2 WCD9385 Power Supplies, Reset, and Analog DAC/ADC Controls

**[HW]** **Power & Reset Sequence**:
1. Enable the four regulator rails: `vdd-rxtx` (`1.8 V`), `vdd-io` (`1.8 V`), `vdd-buck` (`1.8 V`), and `vdd-mic-bias` (`3.5 V`).
2. Assert the WCD9385 reset GPIO low, wait at least `20 ms`, drive reset GPIO high, and wait at least `20 ms` for internal OTP load and SoundWire slave attachment on `swr1` and `swr2`.

**[HW]** **Key WCD9385 Paged Registers**:

| Register | Name | Description |
|---|---|---|
| `0x3009` | `WCD938X_ANA_BIAS` | Bit `7` = `BG_EN` (Bandgap reference enable), Bit `6` = `PRECHRG_EN` |
| `0x300E` | `WCD938X_ANA_HPH` | Bit `7` = `HPHL_PA_EN`, Bit `6` = `HPHR_PA_EN`, Bits `[3:2]` = `PWR_LEVEL` (`HIFI` / `LP` / `AB`) |
| `0x301B` | `WCD938X_ANA_MICB1` | Bits `[7:6]` = `MICB1_CTL` (`1` = enable), Bits `[5:0]` = `VOUT` (`1.6 V`..`2.85 V` in `50 mV` steps) |
| `0x301C` | `WCD938X_ANA_MICB2` | Headset `MICBIAS2` control (`[7:6]` = enable/pullup, `[5:0]` = `VOUT`, typically `1800 mV` -> `0x04`) |
| `0x3022`–`0x3024` | `WCD938X_ANA_TX_CH1`..`CH4` | Analog microphone `ADC1`–`ADC4` enable and PGA gain (`0` to `+30 dB` in `1.5 dB` steps) |
| `0x3409`–`0x340B` | `WCD938X_DIGITAL_INTR_MASK_0..2` | 20-source interrupt mask registers (3 bytes) |
| `0x340F`–`0x3411` | `WCD938X_DIGITAL_INTR_STATUS_0..2` | 20-source interrupt status registers |
| `0x3412`–`0x3414` | `WCD938X_DIGITAL_INTR_CLEAR_0..2` | 20-source interrupt acknowledge/clear registers |

### 7.3 MBHC v2 Jack, Button, and Impedance Detection State Machine

**[HW]** The WCD9385 Multi-Button Headset Controller (`MBHC v2`) uses the following registers and interrupts (`INTR_STATUS_0` bits `[4:0]`):
- `0x3014` (`WCD938X_ANA_MBHC_MECH`):
  - Bit `7`: `L_DET_EN` (enable mechanical HPHL insertion comparator)
  - Bit `6`: `GND_DET_EN` (enable ground jack sense comparator)
  - Bit `5`: `MECH_DETECTION_TYPE` (`1` = active-low / normally-open, `0` = normally-closed)
  - Bit `4`: `HPHL_PLUG_TYPE`, Bit `3`: `GND_PLUG_TYPE`
- `0x3015` (`WCD938X_ANA_MBHC_ELECT`):
  - Bit `7`: `FSM_EN` (enable MBHC ADC/button state machine)
  - Bits `[6:4]`: `BTN_ISRC_CTL` (current source for button detection when `MICBIAS2` is off)
  - Bit `3`: `ELECT_DETECTION_TYPE`, Bits `[2:1]`: `ELECT_SCHMT_ISRC`
- `0x3016` (`WCD938X_ANA_MBHC_ZDET`): Bit `1` = `ELECT_ISRC_EN`
- `0x3019` (`WCD938X_ANA_MBHC_RESULT_3`):
  - Bit `7`: `HPHL_SCHMT_RESULT`, Bit `6`: `HPHR_SCHMT_RESULT`, Bit `5`: `MIC_SCHMT_RESULT`
  - Bit `4`: `IN2P_CLAMP_STATE` / `SWCH_LEVEL_REMOVE`, Bit `3`: `HS_COMP_RESULT`, Bits `[2:0]`: `BTN_RESULT` (`0..7`)
- `0x33DE` (`WCD938X_MBHC_NEW_CTL_1`): Bit `4` = `ADC_MODE`, Bit `3` = `ADC_EN`, Bit `2` = `DETECTION_DONE`, Bits `[1:0]` = `BTN_DBNC`
- `0x33DF` (`WCD938X_MBHC_NEW_CTL_2`): Bits `[6:4]` = `MUX_CTL` (`1` = `IN2P`, `4` = `HPH_L`, `5` = `HPH_R`), Bits `[1:0]` = `HS_VREF` (`(v_hs_max - 1400) / 100`)
- `0x33E0` (`WCD938X_MBHC_NEW_PLUG_DETECT_CTL`): Bits `[5:4]` = `MIC_CLAMP_CTL`, Bits `[3:0]` = `INSREM_DBNC`
- `0x33E1` (`WCD938X_MBHC_NEW_FSM_STATUS`): Bit `7` = `ADC_TIMEOUT`, Bit `6` = `ADC_COMPLETE`, Bit `5` = `MOISTURE_STATUS`, Bit `0` = `FSM_STATUS`
- `0x33E2` (`WCD938X_MBHC_NEW_ADC_RESULT`): 8-bit MBHC ADC voltage reading (`ADC_RESULT`)

**[POLICY]** **Plug Insertion, Classification, and Impedance Sequence**:
1. **Mechanical Insertion IRQ (`WCD938X_IRQ_MBHC_SW_DET`, IRQ `4`)**:
   - Read `SWCH_LEVEL_REMOVE` (`0x3019` bit `4`) to determine whether a plug was inserted or removed.
   - Flip `MECH_DETECTION_TYPE` (`0x3014` bit `5`) to arm the comparator for the opposite edge.
2. **Moisture & Cross-Connection Check**:
   - Measure HPHL voltage via the MBHC ADC (`MUX_CTL = 4`). If moisture or a ground-mic swap (CTIA vs OMTP) is detected (`< 100 mV` cross-connection threshold), toggle the `us_euro_gpio` / crossbar switch if present.
3. **Accessory Classification (3-pole Headphone vs 4-pole Headset vs Line-Out)**:
   - Enable `MICBIAS2` (`1800 mV` in `0x301C`), select `MUX_CTL = 1` (`IN2P`), trigger an MBHC ADC conversion (`ADC_EN = 1`, `FSM_EN = 1`), and poll `ADC_COMPLETE` (`0x33E1` bit `6`):
     - If voltage `< 75 mV` (`WCD_MBHC_ADC_HPH_THRESHOLD_MV`): classify as **3-pole Headphone** (`SND_JACK_HEADPHONE`) and disable `MICBIAS2`.
     - If `75 mV <= voltage <= 1700 mV` (`WCD_MBHC_ADC_HS_THRESHOLD_MV`): classify as **4-pole Headset** (`SND_JACK_HEADSET`) and arm button press/release interrupts (`IRQ 0` / `IRQ 1`).
     - If voltage `> 1700 mV`: classify as **Line-Out / High-Impedance Extension Cable** (`SND_JACK_LINEOUT`) and arm electrical insertion/removal detection (`IRQ 2`).
4. **Headphone Load Impedance Measurement (`zl`, `zr`)**:
   - With `HPHL`/`HPHR` PAs muted, apply the internal current ramp on `HPH_L` and `HPH_R`, read the 16-bit ramp codes (`c1 = (val >> 14) & 0x3`, `x1 = val & 0x3FFF`), and compute impedance in milliohms (`ZDET_CONST = 86 * 16384 = 1409024`) to scale the Class-H headroom and volume compensation tables.
