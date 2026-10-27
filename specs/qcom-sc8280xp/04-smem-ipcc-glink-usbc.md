# Qualcomm SC8280XP: TCSR Mutex, SMEM, IPCC, GLINK, QRTR, PDR, PMIC GLINK USB-C & QMP Combo PHY

## 1. Overview & Inter-Processor Architecture

**[HW]** On the Qualcomm SC8280XP SoC (including the Radxa Dragon Q8B), multiple co-processors—most notably the Low-Power Audio Subsystem / Audio DSP (`ADSP` / `LPASS`), Modem (`MPSS`), Compute DSPs (`CDSP` / `NSP0` and `NSP1`), and System Low-Power Island (`SLPI` / `SSC`)—share system DRAM and coordinate services with the Application Processor (`APSS` / Apps) through a layered inter-processor communication stack:

1. **TCSR Hardware Mutex (`0x01f40000`)**: Hardware spinlocks arbitrating concurrent multi-processor updates to shared memory structures.
2. **SMEM v12 (`0x80900000`)**: A 2 MB reserved DRAM region containing a global directory and per-host-pair private partitions (`$PRT`) that store variable-sized items (such as `socinfo` and GLINK ring buffers).
3. **IPCC (`0x00408000`)**: The Inter-Processor Communication Controller hardware block that generates and receives hardware doorbell interrupts across processor domains and protocols.
4. **GLINK Native SMEM Transport**: A bidirectional packetized virtual channel protocol operating over SMEM ring buffers (`478`, `479`, `480`) and IPCC MPROC doorbell signals.
5. **QRTR (`IPCRTR`) & PDR**: The Qualcomm IPC Router packet protocol running over the `"IPCRTR"` GLINK channel, used to discover QMI services (such as Protection Domain Restart `SERVREG_LOCATOR` at service ID `64` and `SERVREG_NOTIF` at service ID `66`).
6. **PMIC GLINK (`PMIC_RTR_ADSP_APPS`) & QMP Combo PHY (`0x088eb000`, `0x08903000`)**: ADSP-hosted USB Type-C Power Delivery and DisplayPort Altmode state notifications that direct the Application Processor to orient and reset the USB3/DP QMP Combo PHYs and DWC3 controllers.

---

## 2. TCSR Hardware Mutex (`0x01f40000`)

### 2.1 Base Address and Lock Stride

**[HW]** The Top Control and Status Register (TCSR) hardware mutex block on SC8280XP is mapped in the Application Processor physical address space as follows:

| Parameter | Value | Description |
| :--- | :--- | :--- |
| Physical Base Address | `0x01f40000` | TCSR Mutex MMIO base |
| Region Size | `0x00020000` (128 KB) | Total MMIO aperture |
| Number of Locks | `32` (`0` .. `31`) | Independent hardware spinlock registers |
| Lock Register Offset | `lock_index * 0x1000` | 32-bit lock register for `lock_index` (`0` .. `31`) |
| SMEM Allocation Lock | `3` (offset `0x00003000`, physical `0x01f43000`) | Hardware mutex guarding SMEM item allocation |
| APSS Processor Lock Value | `1` | Value written by the Application Processor (`host_id + 1`) |

### 2.2 Lock Acquisition and Release Protocol

**[HW]** Each TCSR mutex register is a 32-bit read/write register with hardware compare-and-set semantics:
- Writing a non-zero 32-bit processor token (`1` for the Application Processor) attempts to claim the lock. If the register currently holds `0` (unlocked), the hardware latches the written value (`1`). If the register already holds a non-zero value written by another processor, the write is ignored and the existing owner value remains latched.
- Writing `0` unconditionally releases the lock.

**[HW]** Step-by-step hardware sequence to acquire lock `N` (specifically `N = 3` at `0x01f43000` for SMEM):
1. Write the 32-bit value `0x00000001` to `0x01f40000 + (N * 0x1000)`.
2. Issue a memory barrier (`dsb sy`) or strongly ordered device read back from `0x01f40000 + (N * 0x1000)`.
3. Compare the read-back value against `0x00000001`.
4. If the read-back value equals `0x00000001`, the Application Processor holds the lock. If not, wait and retry from step 1.

**[HW]** Step-by-step hardware sequence to release lock `N`:
1. Ensure all prior writes to shared memory (SMEM) have completed and are visible to other observers (`dmb sy` / `dsb sy`).
2. Write `0x00000000` to `0x01f40000 + (N * 0x1000)`.

---

## 3. Shared Memory (SMEM v12)

### 3.1 Reserved Memory Region and Host Identifiers

**[HW]** On SC8280XP, SMEM resides in normal uncached / write-combined system DRAM initialized by early boot firmware prior to OS entry:

| Parameter | Value | Description |
| :--- | :--- | :--- |
| Physical Base Address | `0x80900000` | SMEM region base address |
| Region Size | `0x00200000` (2 MB) | Total SMEM size (`0x80900000` .. `0x80afffff`) |
| Partition Table Offset | `0x001ff000` (physical `0x80aff000`) | Always at `smem_base + smem_size - 0x1000` (final 4 KB page) |
| Expected SMEM Version | `12` (`0x000c`) | Version field in `(version[7] >> 16)` (SBL) and `(version[0] >> 16)` (Apps) |
| Hardware Spinlock | TCSR Mutex `3` (`0x01f43000`) | Must be held while allocating any SMEM item |

**[ABI]** Processor host identifiers (`u16`) used in SMEM partition descriptors and APIs:

| Host ID | Mnemonic | Subsystem Description |
| :--- | :--- | :--- |
| `0` | `APPS` | Application Processor Subsystem (`APSS`) |
| `1` | `MODEM` | Modem Processor Subsystem (`MPSS`) |
| `2` | `ADSP` | Low-Power Audio Subsystem (`LPASS` / `ADSP`) |
| `3` | `SSC` | Sensor Subsystem / System Low-Power Island (`SLPI`) |
| `4` | `WCNSS` | Wireless Connectivity Subsystem |
| `5` | `CDSP` | Compute DSP 0 (`NSP0`) |
| `6` | `RPM` | Resource Power Manager |
| `7` | `TZ` | TrustZone Secure World |
| `8` | `SPSS` | Secure Processor Subsystem |
| `9` | `HYP` | Hypervisor |
| `10` | `NPU` | Neural Processing Unit |
| `12` | `NSP1` | Compute DSP 1 (`NSP1` on SC8280XP) |
| `0xfffe` | `GLOBAL` | Common/global partition shared across all processors |

### 3.2 SMEM Global Header (`0x80900000`)

**[ABI]** At byte offset `0x0000` of the SMEM region (`0x80900000`) sits the SMEM global header and legacy 512-entry global allocation table:

| Byte Offset | Size (Bytes) | Type | Field Name | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x0000` | `64` | `4 × 16B` | `proc_comm[4]` | Legacy proc_comm command/status/data words (`4 × 4 × u32`) |
| `0x0040` | `128` | `u32[32]` | `version[32]` | Subsystem version array. Index `7` (`0x005c`) is the bootloader (`MODEM_SBL`) version; bits `[31:16]` must equal `12` (`0x000c`). Index `0` (`0x0040`) is the Apps version |
| `0x00c0` | `4` | `u32` | `initialized` | Must equal `1` when SMEM is initialized |
| `0x00c4` | `4` | `u32` | `free_offset` | Byte offset of next free space in the legacy global heap |
| `0x00c8` | `4` | `u32` | `available` | Remaining free bytes in the legacy global heap |
| `0x00cc` | `4` | `u32` | `reserved` | Must equal `0` (used as spinlock on legacy platforms; unused with TCSR mutex) |
| `0x00d0` | `8192` | `512 × 16B` | `toc[512]` | Global item table (`512` entries of `16` bytes each, `0x00d0` .. `0x20cf`) |

**[ABI]** Each 16-byte entry in `toc[0..511]` has the following layout:

| Byte Offset | Size (Bytes) | Type | Field Name | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x00` | `4` | `u32` | `allocated` | Non-zero (`1`) if the item is allocated; `0` if free |
| `0x04` | `4` | `u32` | `offset` | Byte offset of the item payload relative to the base region |
| `0x08` | `4` | `u32` | `size` | Size in bytes of the allocated item |
| `0x0c` | `4` | `u32` | `aux_base` | Physical base address mask (`aux_base & 0xfffffffc`). If `0` or matching `0x80900000`, offset is relative to `0x80900000` |

### 3.3 Partition Table of Contents (`$TOC` at `0x80aff000`) and `smem_info` (`SIII`)

**[ABI]** On SMEM v12, the final 4 KB of the primary SMEM region (`0x80aff000`) contains the Partition Table of Contents (`ptable`):

| Byte Offset | Size (Bytes) | Type | Field Name | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x00` | `4` | `u8[4]` / `u32` | `magic` | ASCII `"$TOC"` (`0x24, 0x54, 0x4f, 0x43`, or `0x434f5424` little-endian) |
| `0x04` | `4` | `u32` | `version` | Table version; must equal `1` |
| `0x08` | `4` | `u32` | `num_entries` | Number of 48-byte partition entries immediately following at `0x20` |
| `0x0c` | `20` | `u32[5]` | `reserved[5]` | Reserved (`0`) |
| `0x20` | `48 × num_entries` | `ptable_entry[]` | `entry[]` | Array of partition table entries |

**[ABI]** Each 48-byte `ptable_entry` has the following layout:

| Byte Offset | Size (Bytes) | Type | Field Name | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x00` | `4` | `u32` | `offset` | Byte offset of the partition within the SMEM region (`0x80900000 + offset`) |
| `0x04` | `4` | `u32` | `size` | Total size of the partition in bytes |
| `0x08` | `4` | `u32` | `flags` | Partition flags |
| `0x0c` | `2` | `u16` | `host0` | First endpoint Host ID (`0xffff` if unused entry; `0xfffe` for the global partition) |
| `0x0e` | `2` | `u16` | `host1` | Second endpoint Host ID (`0xffff` if unused entry; `0xfffe` for the global partition) |
| `0x10` | `4` | `u32` | `cacheline` | Cacheline alignment size in bytes (typically `32` or `64`) |
| `0x14` | `28` | `u32[7]` | `reserved[7]` | Reserved (`0`) |

**[ABI]** Immediately following `entry[num_entries - 1]` at byte offset `0x20 + (48 * num_entries)` inside `0x80aff000` is the 16-byte `smem_info` descriptor:

| Byte Offset | Size (Bytes) | Type | Field Name | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x00` | `4` | `u8[4]` / `u32` | `magic` | ASCII `"SIII"` (`0x53, 0x49, 0x49, 0x49`, or `0x49494953` little-endian) |
| `0x04` | `4` | `u32` | `size` | Total SMEM size in bytes (`0x00200000`) |
| `0x08` | `4` | `u32` | `base_addr` | SMEM physical base address (`0x80900000`) |
| `0x0c` | `2` | `u16` | `reserved` | Reserved (`0`) |
| `0x0e` | `2` | `u16` | `num_items` | Maximum dynamic item ID count supported beyond the fixed 512 items |

### 3.4 Private Partition Header (`$PRT`) and Dual-Heap Entry Layout

**[ABI]** Every valid partition referenced by a `ptable_entry` (where `{host0, host1}` is either `{0xfffe, 0xfffe}` for the global partition or `{0, remote_host}` / `{remote_host, 0}` for an Apps-to-coprocessor private partition) begins with a 32-byte `$PRT` partition header:

| Byte Offset | Size (Bytes) | Type | Field Name | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x00` | `4` | `u8[4]` / `u32` | `magic` | ASCII `"$PRT"` (`0x24, 0x50, 0x52, 0x54`, or `0x54525024` little-endian) |
| `0x04` | `2` | `u16` | `host0` | First Host ID (must match `ptable_entry.host0`) |
| `0x06` | `2` | `u16` | `host1` | Second Host ID (must match `ptable_entry.host1`) |
| `0x08` | `4` | `u32` | `size` | Usable partition size in bytes (less than or equal to `ptable_entry.size`) |
| `0x0c` | `4` | `u32` | `offset_free_uncached` | Byte offset from partition start of the next free uncached item (grows upward from `0x20`) |
| `0x10` | `4` | `u32` | `offset_free_cached` | Byte offset from partition start of the next free cached space (grows downward from `size`) |
| `0x14` | `12` | `u32[3]` | `reserved[3]` | Reserved (`0`) |

**[ABI]** Each item stored inside a `$PRT` partition is preceded by a 16-byte private entry header (`private_entry`):

| Byte Offset | Size (Bytes) | Type | Field Name | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x00` | `2` | `u16` | `canary` | Must equal `0xa5a5`; any other value indicates memory corruption |
| `0x02` | `2` | `u16` | `item` | SMEM item ID (e.g., `137` for `socinfo`, `478`..`480` for GLINK) |
| `0x04` | `4` | `u32` | `size` | Total padded data size in bytes (includes `padding_data`) |
| `0x08` | `2` | `u16` | `padding_data` | Number of alignment padding bytes included in `size` (`actual_size = size - padding_data`) |
| `0x0a` | `2` | `u16` | `padding_hdr` | Extra padding bytes between this 16-byte header and the item payload |
| `0x0c` | `4` | `u32` | `reserved` | Reserved (`0`) |

**[HW]** **Uncached vs. Cached Partition Walk & Allocation**:
1. **Uncached Heap (grows upward from `partition_base + 0x20` to `partition_base + offset_free_uncached`)**:
   - The first `private_entry` is at `partition_base + 0x20`.
   - For an uncached entry at `hdr`, `padding_hdr` is `0`, the item payload starts at `hdr + 16`, the actual data length is `hdr.size - hdr.padding_data`, and the next uncached `private_entry` is at `hdr + 16 + hdr.size` (`hdr.size` is rounded up to an 8-byte boundary).
   - To allocate an uncached item of `req_size` bytes while holding TCSR Mutex `3`:
     1. Compute `alloc_size = align_up(req_size, 8)`.
     2. Verify that `partition_base + offset_free_uncached + 16 + alloc_size <= partition_base + offset_free_cached`.
     3. Write the 16-byte `private_entry` at `partition_base + offset_free_uncached` with `canary = 0xa5a5`, `item = item_id`, `size = alloc_size`, `padding_data = alloc_size - req_size`, `padding_hdr = 0`, `reserved = 0`.
     4. Execute a write memory barrier (`dmb ishst` / `dsb sy`), then update `offset_free_uncached += 16 + alloc_size`.
2. **Cached Heap (grows downward from `partition_base + size` to `partition_base + offset_free_cached`)**:
   - Let `cacheline` be `ptable_entry.cacheline` (typically `32` or `64`).
   - The first cached `private_entry` is located at `partition_base + size - align_up(16, cacheline)`.
   - For a cached entry at `hdr`, the item payload is placed **below** `hdr` at `hdr - hdr.size` (where `hdr.size` is aligned up to `cacheline`), and the next cached `private_entry` is located at `hdr - hdr.size - align_up(16, cacheline)`.
   - The walk terminates while `hdr > partition_base + offset_free_cached`.
   - To allocate a cached item of `req_size` bytes while holding TCSR Mutex `3`:
     1. Compute `hdr_padded = align_up(16, cacheline)` and `data_padded = align_up(req_size, cacheline)`.
     2. Write the 16-byte `private_entry` at `partition_base + offset_free_cached - hdr_padded` with `canary = 0xa5a5`, `item = item_id`, `size = data_padded`, `padding_data = data_padded - req_size`, `padding_hdr = hdr_padded - 16`, `reserved = 0`.
     3. The item data resides at `partition_base + offset_free_cached - hdr_padded - data_padded`.
     4. Execute a write memory barrier, then update `offset_free_cached -= (hdr_padded + data_padded)`.

**[HW]** **Item Lookup Precedence**:
To look up SMEM item `item_id` for `remote_host`:
1. If `remote_host` is a specific host (`< 15`) and a private partition exists in `$TOC` for `{APPS (0), remote_host}`, search that partition's uncached and cached heaps.
2. If `remote_host` is `GLOBAL` (`0xfffe`) or no private partition exists:
   - If a global `$PRT` partition (`{0xfffe, 0xfffe}`) exists in `$TOC`, search its uncached and cached heaps first.
   - Otherwise (or for legacy fixed items `< 512`), check `toc[item_id]` in the global header at `0x809000d0 + (item_id * 16)`.

### 3.5 Important Global and Private SMEM Item IDs

**[ABI]** Standard SMEM item IDs used on SC8280XP:

| Item ID | Hex | Partition | Description |
| :--- | :--- | :--- | :--- |
| `137` | `0x0089` | Global (`0xfffe`) | `SMEM_HW_SW_BUILD_ID` (`socinfo` descriptor) |
| `478` | `0x01de` | Private (`APPS` ↔ `remote`) | GLINK SMEM Native FIFO Indices (`tx_tail`, `tx_head`, `rx_tail`, `rx_head`) |
| `479` | `0x01df` | Private (`APPS` ↔ `remote`) | GLINK SMEM Native Apps TX Ring Buffer (`16384` bytes) |
| `480` | `0x01e0` | Private (`APPS` ↔ `remote`) | GLINK SMEM Native Apps RX Ring Buffer (`16384` bytes) |

**[ABI]** **`socinfo` Structure (SMEM Item `137` in Global Partition)**:

| Byte Offset | Size (Bytes) | Type | Field Name | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x00` | `4` | `u32` | `fmt` | Structure format (`0x0000` major, `0x0001`..`0x0016` minor) |
| `0x04` | `4` | `u32` | `id` | Qualcomm SoC ID: **`449` (`0x01c1`)** for `SC8280XP` |
| `0x08` | `4` | `u32` | `ver` | SoC hardware version (`major = ver >> 16`, `minor = ver & 0xffff`) |
| `0x0c` | `32` | `char[32]` | `build_id` | Null-terminated firmware build string |
| `0x2c` | `4` | `u32` | `raw_id` | Raw JTAG/foundry ID (v0.2+) |
| `0x30` | `4` | `u32` | `raw_ver` | Raw silicon version (v0.2+) |
| `0x34` | `4` | `u32` | `hw_plat` | Hardware platform type (v0.3+) |
| `0x38` | `4` | `u32` | `plat_ver` | Platform version (v0.4+) |
| `0x3c` | `4` | `u32` | `accessory_chip` | Accessory chip ID (v0.5+) |
| `0x40` | `4` | `u32` | `hw_plat_subtype` | Platform subtype (v0.6+) |
| `0x44` | `4` | `u32` | `pmic_model` | Primary PMIC model (v0.7+) |
| `0x48` | `4` | `u32` | `pmic_die_rev` | Primary PMIC die revision (v0.7+) |
| `0x54` | `4` | `u32` | `serial_num` | Unique 32-bit SoC serial number (v0.10+) |

---

## 4. Inter-Processor Communication Controller (IPCC — `0x00408000`)

### 4.1 Base Address, Interrupt, and Register Map

**[HW]** The IPCC hardware block provides low-latency hardware interrupts between subsystems without polling shared memory. On SC8280XP, the Application Processor accesses the `MPROC` (Protocol `0`, APSS Client `8`) register frame at `0x00408000`:

| Parameter | Value | Description |
| :--- | :--- | :--- |
| Physical Base Address | `0x00408000` | APSS MPROC Protocol frame base |
| Region Size | `0x00001000` (4 KB) | MMIO frame size |
| ACPI HID | `QCOM06C2` | ACPI device identifier on SC8280XP |
| GICv3 Interrupt | SPI `229` (GSIV `261`) | Level-high (`IRQ_TYPE_LEVEL_HIGH`) summary interrupt to APSS |

**[HW]** Register map of the IPCC protocol frame relative to `0x00408000` (note that some ACPI/firmware tables describe the window starting at `0x0040800c`; both offsets are listed below):

| Offset from `0x00408000` | Offset from `0x0040800c` | Access | Register Name | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x0000` | N/A | RO | `IPC_VERSION` | Hardware version (`major = [31:16]`, `minor = [15:8]`, `step = [7:0]`) |
| `0x0008` | N/A | RW | `IPC_CONFIG` | Configuration register. Bit `0` (`CLEAR_ON_RECV_RD`): when `1`, reading `IPC_RECV_ID` automatically clears the signal; when `0`, software must write `IPC_RECV_SIGNAL_CLEAR` |
| `0x000c` | `+0x00` | WO | `IPC_SEND_ID` | Triggers an outgoing doorbell interrupt to `(client_id << 16) \| signal_id` |
| `0x0010` | `+0x04` | RO | `IPC_RECV_ID` | Returns the highest-priority pending `(client_id << 16) \| signal_id`, or `0xffffffff` (`IPC_RECV_ID_NONE`) when no interrupt is pending |
| `0x0014` | `+0x08` | WO | `IPC_RECV_SIGNAL_ENABLE` | Unmasks incoming interrupt for `(client_id << 16) \| signal_id` |
| `0x0018` | `+0x0c` | WO | `IPC_RECV_SIGNAL_DISABLE` | Masks incoming interrupt for `(client_id << 16) \| signal_id` |
| `0x001c` | `+0x10` | WO | `IPC_RECV_SIGNAL_CLEAR` | Clears pending incoming interrupt for `(client_id << 16) \| signal_id` |
| `0x0038` | `+0x2c` | WO | `IPC_CLIENT_CLEAR` | Resets client state |

**[HW]** All signal registers (`IPC_SEND_ID`, `IPC_RECV_ID`, `IPC_RECV_SIGNAL_ENABLE`, `IPC_RECV_SIGNAL_DISABLE`, `IPC_RECV_SIGNAL_CLEAR`) share the same 32-bit bitfield encoding:
- **Bits `[31:16]` (`CLIENT_ID`)**: Target (for `SEND_ID`) or source (for `RECV_*`) processor client ID (`0` .. `65535`).
- **Bits `[15:0]` (`SIGNAL_ID`)**: Signal number within that client (`0` .. `65535`).

### 4.2 Client IDs and MPROC Signal IDs

**[ABI]** IPCC Client IDs on SC8280XP:

| Client ID | Mnemonic | Subsystem Description |
| :--- | :--- | :--- |
| `0` | `IPCC_CLIENT_AOP` | Always-On Processor |
| `1` | `IPCC_CLIENT_TZ` | TrustZone |
| `2` | `IPCC_CLIENT_MPSS` | Modem Processor Subsystem |
| `3` | `IPCC_CLIENT_LPASS` | Low-Power Audio Subsystem (`ADSP`) |
| `4` | `IPCC_CLIENT_SLPI` | Sensor Low-Power Island (`SSC`) |
| `5` | `IPCC_CLIENT_SDC` | Sensor Dedicated Controller |
| `6` | `IPCC_CLIENT_CDSP` | Compute DSP 0 (`NSP0`) |
| `7` | `IPCC_CLIENT_NPU` | Neural Processing Unit |
| `8` | `IPCC_CLIENT_APSS` | Application Processor Subsystem |
| `9` | `IPCC_CLIENT_GPU` | Graphics Processing Unit |
| `10` | `IPCC_CLIENT_CVP` | Computer Vision Processor |
| `11` | `IPCC_CLIENT_CAM` | Camera Subsystem |
| `12` | `IPCC_CLIENT_VPU` | Video Processing Unit |
| `13` | `IPCC_CLIENT_PCIE0` | PCIe Controller 0 |
| `14` | `IPCC_CLIENT_PCIE1` | PCIe Controller 1 |
| `15` | `IPCC_CLIENT_PCIE2` | PCIe Controller 2 |
| `16` | `IPCC_CLIENT_SPSS` | Secure Processor Subsystem |
| `18` | `IPCC_CLIENT_NSP1` | Compute DSP 1 (`NSP1`) |

**[ABI]** Standard MPROC Protocol Signal IDs (`[15:0]`):

| Signal ID | Mnemonic | Purpose |
| :--- | :--- | :--- |
| `0` | `IPCC_MPROC_SIGNAL_GLINK_QMP` | GLINK SMEM Native FIFO notification / QMP mailbox |
| `1` | `IPCC_MPROC_SIGNAL_TZ` | TrustZone notification |
| `2` | `IPCC_MPROC_SIGNAL_SMP2P` | Shared Memory Point-to-Point (`SMP2P`) state change |
| `3` | `IPCC_MPROC_SIGNAL_PING` | Diagnostic ping |

### 4.3 Initialization, Doorbell Send, and Interrupt Handling Sequences

**[HW]** **Initialization Sequence**:
1. Read `IPC_CONFIG` (`0x00408008`), clear bit `0` (`0x00000001`), and write the updated value back to `IPC_CONFIG` so that reading `IPC_RECV_ID` does not implicitly clear signals before software dispatches them.
2. For each `(client_id, signal_id)` pair that the OS listens to (for example, `LPASS` GLINK at `client_id = 3, signal_id = 0` → `0x00030000`), write `(client_id << 16) | signal_id` to `IPC_RECV_SIGNAL_ENABLE` (`0x00408014`).

**[HW]** **Doorbell Send Sequence**:
1. Ensure all prior SMEM ring buffer payload and head-index writes have completed (`dsb sy`).
2. Write `(client_id << 16) | signal_id` (e.g., `0x00030000` for LPASS GLINK) to `IPC_SEND_ID` (`0x0040800c`).

**[HW]** **Interrupt Service Routine (GIC SPI 229 / GSIV 261)**:
1. Loop: read `val = read32(0x00408010)` (`IPC_RECV_ID`).
2. If `val == 0xffffffff`, break out of the loop and return from the interrupt handler.
3. Write `val` to `IPC_RECV_SIGNAL_CLEAR` (`0x0040801c`) to acknowledge this `(client_id, signal_id)` source.
4. Extract `client_id = (val >> 16) & 0xffff` and `signal_id = val & 0xffff` and invoke the registered handler (e.g., GLINK RX ring processing for `client_id = 3, signal_id = 0`).
5. Repeat from step 1 until `IPC_RECV_ID` reads `0xffffffff`.

---

## 5. GLINK SMEM Native Transport

### 5.1 SMEM FIFO Allocation and Descriptor Layout

**[HW]** For each remote processor (such as `ADSP`/`LPASS` with SMEM Host ID `2` and IPCC Client ID `3`), the GLINK SMEM native transport uses three items inside the private SMEM partition (`$PRT`) shared between `APPS` (`0`) and that remote host:

| SMEM Item ID | Allocation Size | Direction (from APSS perspective) | Description |
| :--- | :--- | :--- | :--- |
| `478` (`0x1de`) | `32` bytes (`8 × u32`) | Bidirectional shared | FIFO read/write index descriptors |
| `479` (`0x1df`) | `16384` bytes (`0x4000`) | APSS TX (`tx_pipe`) | Circular byte buffer written by APSS, read by Remote |
| `480` (`0x1e0`) | `16384` bytes (`0x4000`) | APSS RX (`rx_pipe`) | Circular byte buffer written by Remote, read by APSS |

**[HW]** Item `478` (`32` bytes) contains two 16-byte descriptor blocks:

| Byte Offset | Size (Bytes) | Type | Field Name | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x00` | `4` | `u32` | `tx_tail` | APSS TX read index (updated by Remote as it consumes item `479`) |
| `0x04` | `4` | `u32` | `tx_head` | APSS TX write index (updated by APSS as it produces into item `479`) |
| `0x08` | `8` | `u32[2]` | `tx_reserved` | Reserved (`0`) |
| `0x10` | `4` | `u32` | `rx_tail` | APSS RX read index (updated by APSS as it consumes item `480`) |
| `0x14` | `4` | `u32` | `rx_head` | APSS RX write index (updated by Remote as it produces into item `480`) |
| `0x18` | `8` | `u32[2]` | `rx_reserved` | Reserved (`0`) |

**[HW]** **Transport Bringup Sequence**:
1. Acquire TCSR Mutex `3` (`0x01f43000`) via the SMEM allocator and allocate item `478` (`32` bytes) and item `479` (`16384` bytes) in the `{APPS, remote_host}` private partition if not already allocated by the remote processor.
2. Zero-initialize `tx_tail` (`0x00`) and `tx_head` (`0x04`) in item `478` if freshly allocated.
3. Wait until the remote processor has allocated item `480` (RX pipe) in the same private partition.
4. Enable the IPCC incoming signal `(ipcc_client_id << 16) | 0` in `IPC_RECV_SIGNAL_ENABLE` (`0x00408014`).
5. Queue a `CMD_VERSION` (`0`) packet into item `479` (`tx_pipe`) and ring the IPCC doorbell `(ipcc_client_id << 16) | 0`.

### 5.2 Ring Buffer Framing and Alignment Rules

**[HW]** All packets written to `tx_pipe` (item `479`) or read from `rx_pipe` (item `480`) obey strict 8-byte alignment:
1. Both `head` and `tail` byte offsets (`0 <= offset < fifo_length`) are always integer multiples of **`8` bytes**.
2. Available RX bytes:
   - Let `head = read32(rx_head)` and `tail = read32(rx_tail)`.
   - If `head >= tail`, `avail = head - tail`.
   - If `head < tail`, `avail = (fifo_length - tail) + head`.
3. Available TX space:
   - Always leave **`8` bytes** (`FIFO_FULL_RESERVE`) empty so `tx_head == tx_tail` unambiguously means empty:
   - If `tx_tail <= tx_head`, `free_space = (fifo_length - tx_head + tx_tail) - 8`.
   - If `tx_tail > tx_head`, `free_space = (tx_tail - tx_head) - 8`.
4. When writing a command of `hdr_len` bytes followed by `data_len` bytes of payload:
   - Total bytes written to the ring must be `total_aligned = align_up(hdr_len + data_len, 8)`. Any trailing padding bytes (`total_aligned - (hdr_len + data_len)`) are zero-filled.
   - If the write wraps around the end of the `16384`-byte ring buffer, copy the first `fifo_length - tx_head` bytes to the end of the buffer and the remaining bytes starting at offset `0`.
   - Execute a write memory barrier (`dmb ishst` / `dsb sy`), write the new `(tx_head + total_aligned) % fifo_length` to `tx_head`, execute a full barrier (`dsb sy`), and write `(ipcc_client_id << 16) | 0` to `IPC_SEND_ID`.

### 5.3 GLINK Wire Protocol Commands (`0` .. `15`)

**[ABI]** Every GLINK command starts with an 8-byte base header (`glink_msg`), where the 16-bit `cmd` field identifies the command type and two 16-bit parameters (`param1`, `param2`) and optional trailing payload follow:

| Command ID | Mnemonic | Direction | `param1` (`u16` at `+0x02`) | `param2` (`u32` at `+0x04`) | Trailing Payload (after `+0x08`) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `0` | `CMD_VERSION` | Both | `version` (`1`) | `features` (`0x0001` = Intentless, `0x0002` = MIGRATION) | None (`8` bytes total) |
| `1` | `CMD_VERSION_ACK` | Both | `version` (`1`) | Negotiated `features` (`features & supported`) | None (`8` bytes total) |
| `2` | `CMD_OPEN` | Both | `lcid` (sender's channel ID, `>= 1`) | `name_len` (including NUL terminator) | Channel name ASCII string + NUL, padded to 8B |
| `3` | `CMD_CLOSE` | Both | `lcid` (sender's channel ID) | `0` | None (`8` bytes total) |
| `4` | `CMD_OPEN_ACK` | Both | `rcid` (peer's `lcid` being ACKed) | `0` | None (`8` bytes total) |
| `5` | `CMD_INTENT` | Both | `lcid` (sender's channel ID) | `count` (number of intent descriptors) | `count` × 8B `{ u32 size, u32 iid }` descriptors |
| `6` | `CMD_RX_DONE` | Both | `lcid` (sender's channel ID) | `liid` (peer's intent ID consumed and freed) | None (`8` bytes total) |
| `7` | `CMD_RX_INTENT_REQ` | Both | `lcid` (sender's channel ID) | `size` (requested minimum buffer size in bytes) | None (`8` bytes total) |
| `8` | `CMD_RX_INTENT_REQ_ACK` | Both | `lcid` (sender's channel ID) | `granted` (`1` = granted, `0` = denied) | None (`8` bytes total) |
| `9` | `CMD_TX_DATA` | Both | `lcid` (sender's channel ID) | `riid` (receiver's intent ID being filled) | `u32 chunk_size`, `u32 left_size` (`+0x08..+0x0f`), then `chunk_size` bytes of data (padded to 8B) |
| `11` | `CMD_CLOSE_ACK` | Both | `rcid` (peer's `lcid` being ACKed) | `0` | None (`8` bytes total) |
| `12` | `CMD_TX_DATA_CONT` | Both | `lcid` (sender's channel ID) | `riid` (receiver's intent ID being continued) | `u32 chunk_size`, `u32 left_size` (`+0x08..+0x0f`), then `chunk_size` bytes of data (padded to 8B) |
| `13` | `CMD_READ_NOTIF` | Both | `0` | `0` | None (`8` bytes total; requests peer to send doorbell when RX advances) |
| `14` | `CMD_RX_DONE_W_REUSE` | Both | `lcid` (sender's channel ID) | `liid` (peer's intent ID consumed and immediately reusable) | None (`8` bytes total) |
| `15` | `CMD_SIGNALS` | Both | `lcid` (sender's channel ID) | `sigs` (TIOCM flow control bits) | None (`8` bytes total) |

**[ABI]** **Detailed Packet Layouts for Multi-Word GLINK Commands**:

1. **`CMD_INTENT` (`cmd = 5`)**:
   - Offset `0x00`: `u16 cmd = 5`
   - Offset `0x02`: `u16 lcid` (local channel ID advertising receive buffers)
   - Offset `0x04`: `u32 count` (number of intents in this packet, typically `1`)
   - Offset `0x08 + i*8`: `u32 size` (byte capacity of intent `i`), `u32 iid` (local intent ID `>= 1`)

2. **`CMD_TX_DATA` (`cmd = 9`) and `CMD_TX_DATA_CONT` (`cmd = 12`)**:
   - Offset `0x00`: `u16 cmd` (`9` for first chunk, `12` for continuation chunks)
   - Offset `0x02`: `u16 lcid` (sender's local channel ID)
   - Offset `0x04`: `u32 riid` (remote intent ID previously advertised by the receiver via `CMD_INTENT`; `0` on intentless channels)
   - Offset `0x08`: `u32 chunk_size` (number of payload bytes in this packet)
   - Offset `0x0c`: `u32 left_size` (number of payload bytes remaining in subsequent `CMD_TX_DATA_CONT` packets; `0` if the message is complete in this packet)
   - Offset `0x10`: `u8 data[chunk_size]`, followed by `0..7` zero padding bytes so `(16 + chunk_size)` is a multiple of `8`.

### 5.4 Version Negotiation, Channel Handshake, and Intent Flow Control

**[HW]** **1. Version Negotiation**:
- On startup, APSS sends `CMD_VERSION` (`cmd = 0, version = 1, features = 0x0001`).
- When APSS receives `CMD_VERSION` (`cmd = 0`) from the remote subsystem, it verifies `version == 1` and replies with `CMD_VERSION_ACK` (`cmd = 1, version = 1, features = remote_features & 0x0001`).
- When APSS receives `CMD_VERSION_ACK` (`cmd = 1`), if `features == 0x0001`, the negotiated feature mask is committed; otherwise APSS sends a new `CMD_VERSION` with the masked feature set.

**[HW]** **2. Channel Open and Intent Pre-Allocation**:
- Each named GLINK channel (such as `"IPCRTR"`, `"PMIC_RTR_ADSP_APPS"`, or `"adsp_apps"`) requires a symmetric open handshake:
  1. When the remote processor sends `CMD_OPEN` (`cmd = 2, lcid = rcid, name`), APSS records `rcid` for that channel name and immediately replies with `CMD_OPEN_ACK` (`cmd = 4, rcid`).
  2. When an APSS driver binds to that channel, APSS allocates a local channel ID (`lcid >= 1`) and sends `CMD_OPEN` (`cmd = 2, lcid, name`).
  3. The remote processor replies with `CMD_OPEN_ACK` (`cmd = 4, lcid`).
  4. Once **both** `lcid` and `rcid` are open and ACKed, if the channel uses intents (which is true for `"PMIC_RTR_ADSP_APPS"` and `"adsp_apps"`, whereas `"IPCRTR"` on some transports is intentless if negotiated, or uses intents if `features == 0`), APSS **must** immediately pre-populate receive intents by sending `CMD_INTENT` (`cmd = 5`) packets:
     - **[POLICY]** For standard GLINK channels (including `"PMIC_RTR_ADSP_APPS"`), pre-allocate **`5` reusable intents of `1024` bytes each** (`iid = 1..5`).
     - **[POLICY]** For the AudioReach GPR channel (`"adsp_apps"`), pre-allocate **`20` reusable intents of `512` bytes each** (`iid = 1..20`).
  5. Whenever APSS finishes processing a complete received message (`left_size == 0`) on intent `liid`:
     - If `liid` is a reusable intent, APSS sends `CMD_RX_DONE_W_REUSE` (`cmd = 14, lcid, liid`) so the remote processor can immediately reuse `liid` without waiting for another `CMD_INTENT`.
     - If `liid` was dynamically allocated on demand for a large `CMD_RX_INTENT_REQ`, APSS sends `CMD_RX_DONE` (`cmd = 6, lcid, liid`).
  6. When APSS wants to transmit a message of `N` bytes on an intent-based channel:
     - Search the channel's list of available remote intents (`riid` advertised by the remote processor via `CMD_INTENT`) for one with `size >= N` that is not currently in flight.
     - If none is available, send `CMD_RX_INTENT_REQ` (`cmd = 7, lcid, size = N`), wait for the remote processor to reply with `CMD_RX_INTENT_REQ_ACK` (`cmd = 8, granted = 1`) and `CMD_INTENT` (`cmd = 5`), and then send `CMD_TX_DATA` (`cmd = 9, lcid, riid, chunk_size = N, left_size = 0, payload`).

### 5.5 Empirical Hazard: 1-in-3 Boot Hang on Early ADSP Channel Close/Reopen

**[HW]** **Critical Hardware/Firmware Timing Constraint on SC8280XP (Radxa Dragon Q8B)**:
- During system boot, the ADSP (`LPASS`) firmware initializes its internal services (`IPCRTR`, `SERVREG`, `PMIC_RTR_ADSP_APPS`, `adsp_apps`) asynchronously after the initial `CMD_VERSION` / `CMD_VERSION_ACK` exchange.
- Empirically verified on the Radxa Dragon Q8B: if the Application Processor opens and then closes (`CMD_CLOSE`), or rapidly opens and sends traffic on `"PMIC_RTR_ADSP_APPS"` or `"adsp_apps"` before the ADSP has announced its `CMD_OPEN` for that channel (or before the ADSP protection domain `avs/audio` is reported up via PDR), the ADSP GLINK state machine deadlocks roughly **1 in 3 boots**, halting all subsequent USB-C and audio communication until a full board power cycle.
- **Required Sequence**:
  1. Wait for the remote processor (`ADSP`) to send `CMD_OPEN` for a given channel (`"IPCRTR"`, `"PMIC_RTR_ADSP_APPS"`, `"adsp_apps"`) and reply with `CMD_OPEN_ACK` before sending APSS `CMD_OPEN` and application packets, **and**
  2. Never issue `CMD_CLOSE` / reopen cycles during probe; keep the channel open once established, and for audio (`"adsp_apps"`), wait until PDR (`SERVREG_LOCATOR` / `SERVREG_NOTIF`) reports `SERVICE_STATE_UP` (`0x1fffffff`) for `"msm/adsp/audio_pd"` before sending GPR commands.

---

## 6. QRTR (`IPCRTR`) & Protection Domain Restart (PDR)

### 6.1 QRTR Transport on GLINK Channel `"IPCRTR"`

**[ABI]** The Qualcomm IPC Router (`QRTR`) protocol runs over the GLINK channel named `"IPCRTR"` on each remote processor (`ADSP`, `MPSS`, `CDSP`, `SLPI`). The Application Processor uses node ID `1` (`QRTR_LOCAL_NODE_ID = 1`), and each remote processor announces its own node ID via control packets.

**[ABI]** **QRTR Wire Header Formats**:
The first byte of every GLINK `"IPCRTR"` packet contains the protocol version in bits `[7:0]`:

**Version 1 Header (`version = 1`, `32` bytes, little-endian)**:

| Byte Offset | Size (Bytes) | Type | Field Name | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x00` | `4` | `u32` | `version` | Protocol version: `1` |
| `0x04` | `4` | `u32` | `type` | Packet type (`QRTR_TYPE_*`, see table below) |
| `0x08` | `4` | `u32` | `src_node_id` | Source processor node ID (`1` for APSS) |
| `0x0c` | `4` | `u32` | `src_port_id` | Source port ID (`0xfffffffe` = `QRTR_PORT_CTRL` for router control packets) |
| `0x10` | `4` | `u32` | `confirm_rx` | Non-zero (`1`) to request a `QRTR_TYPE_RESUME_TX` confirmation |
| `0x14` | `4` | `u32` | `size` | Exact byte length of payload following this 32-byte header |
| `0x18` | `4` | `u32` | `dst_node_id` | Destination processor node ID (`0xffffffff` = broadcast) |
| `0x1c` | `4` | `u32` | `dst_port_id` | Destination port ID (`0xfffffffe` = `QRTR_PORT_CTRL`) |

**Version 2 Header (`version = 3`, `16` bytes, little-endian)**:

| Byte Offset | Size (Bytes) | Type | Field Name | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x00` | `1` | `u8` | `version` | Protocol version: `3` |
| `0x01` | `1` | `u8` | `type` | Packet type (`QRTR_TYPE_*`) |
| `0x02` | `1` | `u8` | `flags` | Bit `0` (`0x01`): `CONFIRM_RX` |
| `0x03` | `1` | `u8` | `optlen` | Optional metadata length between header and payload (`0`) |
| `0x04` | `4` | `u32` | `size` | Exact byte length of payload |
| `0x08` | `2` | `u16` | `src_node_id` | Source node ID |
| `0x0a` | `2` | `u16` | `src_port_id` | Source port ID (`0xfffe` maps to `0xfffffffe`) |
| `0x0c` | `2` | `u16` | `dst_node_id` | Destination node ID (`0xffff` maps to `0xffffffff`) |
| `0x0e` | `2` | `u16` | `dst_port_id` | Destination port ID (`0xfffe` maps to `0xfffffffe`) |

**[ABI]** **QRTR Packet Types (`type`) and Control Packet Payload (`20` bytes)**:

| Type ID | Mnemonic | Payload Layout (`20` bytes starting after QRTR header) |
| :--- | :--- | :--- |
| `1` | `QRTR_TYPE_DATA` | QMI / application message payload (`size` bytes) |
| `2` | `QRTR_TYPE_HELLO` | `u32 cmd = 2`, `u8 zero[16]` — exchanged immediately when `"IPCRTR"` opens |
| `3` | `QRTR_TYPE_BYE` | `u32 cmd = 3`, `u8 zero[16]` — sent when a remote node disconnects |
| `4` | `QRTR_TYPE_NEW_SERVER` | `u32 cmd = 4`, `u32 service`, `u32 instance` (`version \| (instance << 8)`), `u32 node`, `u32 port` |
| `5` | `QRTR_TYPE_DEL_SERVER` | `u32 cmd = 5`, `u32 service`, `u32 instance`, `u32 node`, `u32 port` |
| `6` | `QRTR_TYPE_DEL_CLIENT` | `u32 cmd = 6`, `u32 node`, `u32 port`, `u8 zero[8]` |
| `7` | `QRTR_TYPE_RESUME_TX` | `u32 cmd = 7`, `u32 node`, `u32 port`, `u8 zero[8]` |
| `8` | `QRTR_TYPE_EXIT` | `u32 cmd = 8`, `u8 zero[16]` |
| `9` | `QRTR_TYPE_PING` | `u32 cmd = 9`, `u8 zero[16]` |
| `10` | `QRTR_TYPE_NEW_LOOKUP` | `u32 cmd = 10`, `u32 service`, `u32 instance`, `u32 node`, `u32 port` |
| `11` | `QRTR_TYPE_DEL_LOOKUP` | `u32 cmd = 11`, `u32 service`, `u32 instance`, `u32 node`, `u32 port` |

**[HW]** **QRTR Name Service Handshake**:
1. As soon as the `"IPCRTR"` GLINK channel opens (`CMD_OPEN` + `CMD_OPEN_ACK` in both directions), APSS sends a `QRTR_TYPE_HELLO` (`cmd = 2`) control packet (`src_node = 1, src_port = 0xfffffffe, dst_node = 0xffffffff, dst_port = 0xfffffffe`).
2. When the remote processor receives `QRTR_TYPE_HELLO` (and sends its own `QRTR_TYPE_HELLO`), it sends `QRTR_TYPE_NEW_SERVER` (`cmd = 4`) packets announcing every QMI service hosted on that DSP (`service`, `instance`, `node`, `port`).

### 6.2 Protection Domain Restart (PDR — `SERVREG_LOCATOR` & `SERVREG_NOTIF`)

**[ABI]** Before communicating with user-PD services on the ADSP (such as AudioReach), APSS queries the Protection Domain Locator (`SERVREG_LOCATOR`) and subscribes to the Protection Domain Notifier (`SERVREG_NOTIF`) over QRTR/QMI:

| Service Name | QRTR Service ID | QRTR Instance | QMI Message IDs | Description |
| :--- | :--- | :--- | :--- | :--- |
| `SERVREG_LOCATOR` | `64` (`0x40`) | `1` (`version = 1, instance = 0`) | `0x0020` (`GET_DOMAIN_LIST_REQ`/`RESP`) | Maps service name (e.g., `"avs/audio"`) to PD path (`"msm/adsp/audio_pd"`) and notifier instance ID |
| `SERVREG_NOTIF` | `66` (`0x42`) | `(instance << 8) \| 1` | `0x0020` (`REGISTER_LISTENER_REQ`/`RESP`), `0x0022` (`STATE_UPDATED_IND`), `0x0023` (`STATE_UPDATED_IND_ACK_REQ`/`RESP`) | Reports and tracks PD state (`DOWN`, `UP`, `EARLY_DOWN`) |

**[ABI]** **QMI Wire Header (`7` bytes) and TLV Format**:
Every QMI message sent in a `QRTR_TYPE_DATA` packet starts with a 7-byte QMI header:
- Offset `0x00`: `u8 cntl_flag` (`0x00` = Request, `0x02` = Response, `0x04` = Indication)
- Offset `0x01`: `u16 txn_id` (incrementing transaction ID, `>= 1`)
- Offset `0x03`: `u16 msg_id` (QMI message ID, e.g., `0x0020`)
- Offset `0x05`: `u16 msg_len` (total byte length of all following TLVs)
- Offset `0x07+`: Sequence of TLVs, each formatted as `{ u8 type, u16 length, u8 value[length] }`.

**[ABI]** **PDR Step-by-Step Sequence for Audio (`"avs/audio"` / `"msm/adsp/audio_pd"`)**:
1. Wait for `QRTR_TYPE_NEW_SERVER` announcing `service = 0x40` (`SERVREG_LOCATOR`), `instance = 1`.
2. Send `SERVREG_GET_DOMAIN_LIST_REQ` (`msg_id = 0x0020`) to the `SERVREG_LOCATOR` `(node, port)`:
   - Mandatory TLV `0x01`: `u16 string_len` (`9`), followed by ASCII `"avs/audio"` (`9` bytes, no NUL in QMI string payload).
   - Optional TLV `0x10`: `u32 domain_offset = 0`.
3. Parse `SERVREG_GET_DOMAIN_LIST_RESP` (`msg_id = 0x0020`):
   - Mandatory TLV `0x02` (`4` bytes): `{ u16 result, u16 error }` (`result == 0` indicates success).
   - Optional TLV `0x10` (`u16 total_domains`).
   - Optional TLV `0x11` (`u16 db_rev_count`).
   - Optional TLV `0x12` (`domain_list`): `u8 num_entries`, followed by entries each containing `{ u8 name_len, char name[name_len] (e.g., "msm/adsp/audio_pd"), u32 instance_id, u8 service_data_valid, u32 service_data }`.
4. Wait for `QRTR_TYPE_NEW_SERVER` announcing `service = 0x42` (`SERVREG_NOTIF`) with `instance = (instance_id << 8) | 1`.
5. Send `SERVREG_REGISTER_LISTENER_REQ` (`msg_id = 0x0020`) to that `SERVREG_NOTIF` `(node, port)`:
   - Mandatory TLV `0x01` (`1` byte): `u8 enable = 1`.
   - Mandatory TLV `0x02`: `u16 string_len`, followed by the domain name string (e.g., `"msm/adsp/audio_pd"`).
6. Parse `SERVREG_REGISTER_LISTENER_RESP` (`msg_id = 0x0020`):
   - Mandatory TLV `0x02` (`4` bytes): `{ u16 result, u16 error }`.
   - Optional TLV `0x10` (`4` bytes): `u32 curr_state`.
     - `0x0fffffff`: `SERVICE_STATE_DOWN`
     - `0x1fffffff`: `SERVICE_STATE_UP`
     - `0x2fffffff`: `SERVICE_STATE_EARLY_DOWN`
7. Whenever a `SERVREG_STATE_UPDATED_IND` (`cntl_flag = 0x04, msg_id = 0x0022`) arrives from `SERVREG_NOTIF`:
   - TLV `0x01` (`4` bytes): `u32 curr_state`.
   - TLV `0x02`: domain string (`u16 len` + chars).
   - TLV `0x03` (`2` bytes): `u16 transaction_id`.
   - Immediately send `SERVREG_STATE_UPDATED_IND_ACK_REQ` (`cntl_flag = 0x00, msg_id = 0x0023`) containing TLV `0x01` (domain string) and TLV `0x02` (`u16 transaction_id`) to acknowledge the state transition.

---

## 7. PMIC GLINK (`PMIC_RTR_ADSP_APPS`) & USB Type-C / Altmode Protocol

### 7.1 PMIC GLINK Channel and Message Header

**[ABI]** On SC8280XP, the PMIC USB Type-C PD controller and charger firmware run on the ADSP (`LPASS`, SMEM Host ID `2`, IPCC Client ID `3`) and communicate with the Application Processor over the GLINK channel named **`"PMIC_RTR_ADSP_APPS"`**.

**[ABI]** Every message sent or received on `"PMIC_RTR_ADSP_APPS"` starts with an 12-byte `pmic_glink_hdr` (three little-endian 32-bit words):

| Byte Offset | Size (Bytes) | Type | Field Name | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x00` | `4` | `u32` | `owner` | Subsystem owner ID (`0x800a`..`0x800c`) |
| `0x04` | `4` | `u32` | `type` | Message direction/type (`1` = Req/Resp, `2` = Notification) |
| `0x08` | `4` | `u32` | `opcode` | Operation code specific to `owner` |

**[ABI]** **Owner IDs (`owner`) and Message Types (`type`)**:

| Constant | Value | Description |
| :--- | :--- | :--- |
| `PMIC_GLINK_OWNER_BATTMGR` | `0x0000800a` (`32778`) | Battery Manager / Power Supply service |
| `PMIC_GLINK_OWNER_USBC` | `0x0000800b` (`32779`) | USB Type-C role and orientation service |
| `PMIC_GLINK_OWNER_USBC_PAN` | `0x0000800c` (`32780`) | USB Type-C Port/Altmode Notification (PAN) service (SC8280XP) |
| `PMIC_GLINK_REQ_RESP` | `0x00000001` | Request / Response message |
| `PMIC_GLINK_NOTIFY` | `0x00000002` | Asynchronous indication from ADSP to APSS |

### 7.2 USBC & Altmode Opcodes and Packet Layouts (SC8280XP)

**[ABI]** On SC8280XP, the USB-C Altmode & Role service uses the following opcodes:

| Opcode | Value | Owner | Type | Direction | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `USBC_CMD_WRITE_REQ` | `0x00000015` | `0x800c` (`USBC_PAN`) or `0x800b` (`USBC`) | `1` (`REQ_RESP`) | APSS ↔ ADSP | Write command (`PAN_EN = 0x10`, `PAN_ACK = 0x11`) and 16-byte response |
| `USBC_NOTIFY_IND` | `0x00000016` | `0x800a` (`BATTMGR`) or `0x800b` (`USBC`) | `2` (`NOTIFY`) | ADSP → APSS | Port state change indication (`32` bytes total on SC8280XP) |

**[ABI]** **1. `USBC_CMD_WRITE_REQ` (`opcode = 0x15`, `24` bytes, APSS → ADSP)**:

| Byte Offset | Size (Bytes) | Type | Field Name | Value / Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x00` | `4` | `u32` | `hdr.owner` | `0x0000800c` (`PMIC_GLINK_OWNER_USBC_PAN`) |
| `0x04` | `4` | `u32` | `hdr.type` | `0x00000001` (`PMIC_GLINK_REQ_RESP`) |
| `0x08` | `4` | `u32` | `hdr.opcode` | `0x00000015` (`USBC_CMD_WRITE_REQ`) |
| `0x0c` | `4` | `u32` | `cmd` | `0x00000010` (`ALTMODE_PAN_EN`) to enable notifications, or `0x00000011` (`ALTMODE_PAN_ACK`) to acknowledge a port notification |
| `0x10` | `4` | `u32` | `arg` | `0` for `ALTMODE_PAN_EN`; `port_index` (`0` or `1`) for `ALTMODE_PAN_ACK` |
| `0x14` | `4` | `u32` | `reserved` | `0` |

**[ABI]** **2. `USBC_CMD_WRITE_RESP` (`opcode = 0x15`, `16` bytes, ADSP → APSS)**:

| Byte Offset | Size (Bytes) | Type | Field Name | Value / Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x00` | `4` | `u32` | `hdr.owner` | `0x0000800c` (`PMIC_GLINK_OWNER_USBC_PAN`) |
| `0x04` | `4` | `u32` | `hdr.type` | `0x00000001` (`PMIC_GLINK_REQ_RESP`) |
| `0x08` | `4` | `u32` | `hdr.opcode` | `0x00000015` (`USBC_CMD_WRITE_REQ`) |
| `0x0c` | `4` | `u32` | `response` | `0` = success |

**[ABI]** **3. `USBC_NOTIFY_IND` on SC8280XP (`opcode = 0x16`, `32` bytes total, ADSP → APSS)**:
Note: On SC8280XP, the ADSP sends `USBC_NOTIFY_IND` (`opcode = 0x16`) with `hdr.owner == 0x800a` (`PMIC_GLINK_OWNER_BATTMGR`) and `hdr.type == 2`. Drivers must accept `opcode == 0x16` under either `0x800a` or `0x800b`.

| Byte Offset | Size (Bytes) | Type | Field Name | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x00` | `4` | `u32` | `hdr.owner` | `0x0000800a` (`BATTMGR`) or `0x0000800b` (`USBC`) |
| `0x04` | `4` | `u32` | `hdr.type` | `0x00000002` (`PMIC_GLINK_NOTIFY`) |
| `0x08` | `4` | `u32` | `hdr.opcode` | `0x00000016` (`USBC_NOTIFY_IND`) |
| `0x0c` | `1` | `u8` | `port_index` | USB-C connector index (`0` for Port 0, `1` for Port 1) |
| `0x0d` | `1` | `u8` | `orientation` | `0` = Normal (`CC1`), `1` = Reverse (`CC2`), `2` = Disconnected / None |
| `0x0e` | `1` | `u8` | `ctrl_messages` | Bitfield:<ul><li>Bits `[3:0]`: Power Role (`0` = Sink, `1` = Source)</li><li>Bits `[7:4]`: Data Role (`0` = None/Disconnected, `1` = Host/DFP, `2` = Device/UFP)</li></ul> |
| `0x0f` | `1` | `u8` | `svid_mode` | Crossbar/Altmode mux mode:<ul><li>`0`: `NONE` (disconnected)</li><li>`1`: `DP` (`4`-lane DisplayPort only, Pin Assignments C/E)</li><li>`2`: `USB` (USB 3.x SuperSpeed only)</li><li>`3`: `USB_DP` (`2`-lane USB3 + `2`-lane DP, Pin Assignments D/F)</li></ul> |
| `0x10` | `1` | `u8` | `dp_lanes` | DP Pin Assignment encoding (`1..6` → Pin Assignments `A..F`: `(1 << (dp_lanes - 1))`) |
| `0x11` | `1` | `u8` | `hpd` | DisplayPort HPD status:<ul><li>Bit `0` (`0x01`): `HPD_STATE` (`1` = HPD asserted high)</li><li>Bit `1` (`0x02`): `HPD_IRQ` (`1` = short HPD IRQ pulse)</li></ul> |
| `0x12` | `2` | `u16` | `svid` | Standard or Vendor ID (`0xff01` for VESA DisplayPort Altmode) |
| `0x14` | `12` | `u8[12]` | `reserved` | Reserved padding to `32` bytes |

### 7.3 Mandatory Asynchronous PAN_ACK Handshake

**[HW]** **Critical Ordering Rule for `ALTMODE_PAN_EN` and `ALTMODE_PAN_ACK`**:
1. When `"PMIC_RTR_ADSP_APPS"` opens (and after `5 × 1 KB` GLINK RX intents are queued), APSS sends `USBC_CMD_WRITE_REQ` (`owner = 0x800c, type = 1, opcode = 0x15, cmd = 0x10 (ALTMODE_PAN_EN), arg = 0`) and waits up to 5 seconds for `USBC_CMD_WRITE_RESP` (`response == 0`).
2. Once `ALTMODE_PAN_EN` is acknowledged, the ADSP immediately emits `USBC_NOTIFY_IND` (`opcode = 0x16`) packets for each active USB-C port (`port_index = 0`, `port_index = 1`).
3. **Never send `ALTMODE_PAN_ACK` synchronously from inside the GLINK RX interrupt handler**: doing so deadlocks waiting for the `USBC_CMD_WRITE_RESP` packet that can only arrive on the same GLINK RX path.
4. Instead, defer handling of each `USBC_NOTIFY_IND` to an asynchronous worker context:
   - First, program the QMP Combo PHY orientation (`DP_COM_TYPEC_CTRL` and `PCS_SW_RESET`) and update the DWC3 USB role switch (`Host` vs. `Device`).
   - Second, send `USBC_CMD_WRITE_REQ` (`owner = 0x800c, type = 1, opcode = 0x15, cmd = 0x11 (ALTMODE_PAN_ACK), arg = port_index`) and wait for `USBC_CMD_WRITE_RESP`. The ADSP will not deliver subsequent notifications for `port_index` until `ALTMODE_PAN_ACK` is sent.

---

## 8. QMP USB3-DP Combo PHY (`0x088eb000` Port 0, `0x08903000` Port 1)

### 8.1 Base Addresses and Sub-Block Offsets (QMP v5)

**[HW]** SC8280XP features two USB3 + DisplayPort QMP v5 Combo PHYs paired with the two primary USB-C ports (`usb_0` at `0x0a600000` and `usb_1` at `0x0a800000`):

| Port | Combo PHY Physical Base | Size | Paired DWC3 Controller | Description |
| :--- | :--- | :--- | :--- | :--- |
| Port 0 | `0x088eb000` | `0x4000` (16 KB) | `0x0a600000` | USB-C Port 0 SuperSpeed + DP Combo PHY |
| Port 1 | `0x08903000` | `0x4000` (16 KB) | `0x0a800000` | USB-C Port 1 SuperSpeed + DP Combo PHY |

**[HW]** Each QMP v5 Combo PHY (`0x088eb000` and `0x08903000`) is divided into the following sub-blocks (`qmp_combo_offsets_v5`):

| Sub-Block | Offset from PHY Base | Description |
| :--- | :--- | :--- |
| `com` (`dp_com`) | `+0x0000` | Common Type-C & DP/USB3 crossbar control registers |
| `txa` | `+0x0400` | Lane A TX Serializer registers |
| `rxa` | `+0x0600` | Lane A RX Deserializer registers |
| `txb` | `+0x0a00` | Lane B TX Serializer registers |
| `rxb` | `+0x0c00` | Lane B RX Deserializer registers |
| `usb3_serdes` | `+0x1000` | USB3 QSERDES PLL registers |
| `usb3_pcs_misc` | `+0x1200` | USB3 PCS Misc registers |
| `usb3_pcs` | `+0x1400` | USB3 PCS v4 control and status registers |
| `usb3_pcs_usb` | `+0x1700` | USB3 PCS USB4/USB3 specific registers |
| `dp_serdes` | `+0x2000` | DisplayPort QSERDES PLL registers |
| `dp_dp_phy` | `+0x2200` | DisplayPort PHY control registers |

### 8.2 Key Type-C Orientation and Reset Registers

**[HW]** Key registers in the `com` (`+0x0000`) and `usb3_pcs` (`+0x1400`) sub-blocks:

| Offset from PHY Base | Register Name | Key Bits & Values |
| :--- | :--- | :--- |
| `+0x0000` | `QPHY_V3_DP_COM_PHY_MODE_CTRL` | Bit `0` (`0x01`): `USB3_MODE`; Bit `1` (`0x02`): `DP_MODE` (`0x03` enables both USB3 and DP) |
| `+0x0004` | `QPHY_V3_DP_COM_SW_RESET` | Bit `0` (`0x01`): `SW_RESET` |
| `+0x0008` | `QPHY_V3_DP_COM_POWER_DOWN_CTRL` | Bit `0` (`0x01`): `SW_PWRDN` (`1` = power up / active) |
| `+0x000c` | `QPHY_V3_DP_COM_SWI_CTRL` | Software interface control |
| `+0x0010` | `QPHY_V3_DP_COM_TYPEC_CTRL` |<ul><li>Bit `0` (`0x01`): `PORTSELECT_VAL` (`0` = Normal / CC1, `1` = Reverse / CC2)</li><li>Bit `1` (`0x02`): `PORTSELECT_MUX` (`1` = override hardware pin with software `PORTSELECT_VAL`)</li><li>Write **`0x02`** for Normal (`CC1`) orientation</li><li>Write **`0x03`** for Reverse (`CC2`) orientation</li></ul> |
| `+0x001c` | `QPHY_V3_DP_COM_RESET_OVRD_CTRL` | Reset override control (`0x00` clears overrides) |
| `+0x1400` | `QPHY_V4_PCS_SW_RESET` | Bit `0` (`0x01`): `SW_RESET` (`1` = assert USB3 PCS reset, `0` = deassert) |
| `+0x1404` | `QPHY_V4_PCS_REVISION_ID0` | PCS revision ID |
| `+0x1414` | `QPHY_V4_PCS_PCS_STATUS1` | Bit `0` (`0x01`): `PHYSTATUS` (`0` = PHY ready, `1` = busy/resetting) |
| `+0x1428` | `QPHY_V4_PCS_POWER_DOWN_CONTROL` | Bit `0` (`0x01`): `SW_PWRDN` (`1` = power up PCS) |
| `+0x142c` | `QPHY_V4_PCS_START_CONTROL` | Bit `0` (`0x01`): `SERDES_START`, Bit `1` (`0x02`): `PCS_START` (`0x03` starts both) |

### 8.3 Runtime Type-C Orientation Switch Sequence

**[HW]** On the Radxa Dragon Q8B (where UEFI firmware leaves the QMP Combo PHY clocks, QSERDES PLL, and calibration tables initialized at boot), switching USB-C SuperSpeed lane orientation in response to a `USBC_NOTIFY_IND` (`orientation = 0` or `1`) requires the following hardware sequence on Port `P` (`base = 0x088eb000` for Port 0, `0x08903000` for Port 1):

1. Determine the target `TYPEC_CTRL` value from `USBC_NOTIFY_IND.orientation`:
   - If `orientation == 0` (Normal / `CC1`), `typec_ctrl = 0x02`.
   - If `orientation == 1` (Reverse / `CC2`), `typec_ctrl = 0x03`.
   - If `orientation == 2` (Disconnected), leave `TYPEC_CTRL` unchanged or set `0x02`.
2. Write `typec_ctrl` (`0x02` or `0x03`) to `base + 0x0010` (`QPHY_V3_DP_COM_TYPEC_CTRL`).
3. Toggle the USB3 PCS software reset (`base + 0x1400`, `QPHY_V4_PCS_SW_RESET`) so the PCS re-locks onto the newly selected lane pair:
   - Read `val = read32(base + 0x1400)`, set bit `0` (`val | 0x01`), and write back to `base + 0x1400`.
   - Wait `100` microseconds (`dsb sy` + delay).
   - Clear bit `0` (`val & ~0x01`) and write back to `base + 0x1400`.
   - Poll `base + 0x1414` (`QPHY_V4_PCS_PCS_STATUS1`) until bit `0` (`PHYSTATUS`) reads `0` (timeout `1000` microseconds).
4. Send `ALTMODE_PAN_ACK` (`cmd = 0x11, arg = port_index`) over `"PMIC_RTR_ADSP_APPS"`.
