# Qualcomm SC8280XP Firmware & Power Fabric Specification: SCM SMC ABI, Command DB, RPMh RSC, and PAS Remoteproc

## 1. Overview and System Architecture

**[HW]** On the Qualcomm SC8280XP (Radxa Dragon Q8B), power rails (ARC), voltage regulators (VRM), bus clock managers (BCM), and crystal oscillator clocks are controlled through the Resource Power Manager Hardened (**RPMh**) subsystem. Resource addresses and auxiliary metadata for RPMh accelerators are discovered at runtime from a static in-memory database called **Command DB**, populated by boot firmware at physical address `0x80860000`. High-level OS requests are transmitted to RPMh via the Apps **Resource State Coordinator (RSC)** at `0x18200000` (Direct Resource Voter `DRV 2` at `0x18220000`).

**[HW]** Co-processor firmware loading and authentication for the Audio DSP (**ADSP**, `0x03000000`), Compute DSPs (**NSP0**, `0x1B300000`, and **NSP1**), and Adreno 690 GPU zap shader are mediated by TrustZone through the **Qualcomm Secure Channel Manager (SCM)** ARM SMCCC `SMC64` interface and the **Peripheral Authentication Service (PAS)**.

| Component | Physical Base Address | Size | Identifier / Version | Role |
|---|---|---|---|---|
| Command DB (`cmd-db-region@80860000`) | `0x80860000` | `0x20000` (128 KiB) | Magic `0x0C0330DB`, Version `1` | Maps 8-char resource names to 20-bit RPMh addresses and aux data |
| Apps RSC (`rsc@18200000`) | `0x18200000` (DRV 0)<br>`0x18210000` (DRV 1)<br>`0x18220000` (DRV 2) | `0x10000` each | RSC v3.0 (`DRV_ID = 0x00030000`) | Apps processor uses **DRV 2 (`0x18220000`)**, TCS base `+0xD00` (`0x18220D00`), GIC SPI 5 |
| SMEM (`smem-region@80900000`) | `0x80900000` | `0x200000` (2 MiB) | `qcom,smem` (hwlock 3) | Shared memory for SMP2P, GLINK, and crash reasons |
| ADSP PAS (`remoteproc@3000000`) | `0x03000000` | `0x10000` | PAS ID `1` (`0x01`) | Audio DSP (`qcadsp8280.mbn`); carveout `0x86C00000` (`0x2000000` = 32 MiB) |
| NSP0 / CDSP0 PAS (`remoteproc@1b300000`) | `0x1B300000` | `0x10000` | PAS ID `18` (`0x12`) | Neural/Compute DSP 0 (`qccdsp8280.mbn`); carveout `0x8A100000` (`0x1E00000` = 30 MiB) |
| NSP1 / CDSP1 PAS | — | — | PAS ID `30` (`0x1E`) | Neural/Compute DSP 1 (disabled on Dragon Q8B) |
| GPU Zap Shader (`gpu-mem@8bf00000`) | `0x8BF00000` | `0x2000` (8 KiB) | PAS ID `13` (`0x0D`) | Adreno 690 GPU secure microcode (`qcdxkmuc8280.mbn`) |

---

## 2. Qualcomm SCM SMC64 Calling Convention

### 2.1 SMCCC Function ID (`x0`) and Argument Encoding (`x1`–`x5`)

**[ABI]** **[HW]** On SC8280XP, all SCM calls use the **ARM SMCCC SMC64** convention. Invoking `smc #0` from EL1 traps to TrustZone EL3 monitor firmware.
- **Critical EL2 Requirement**: If the OS boots at EL2 (as UEFI leaves the CPU on Radxa Dragon Q8B when `ExitBootServices` is called), `HCR_EL2.TSC` (bit 19) is `0` by default, so `smc #0` traps directly to EL3 even from EL2. However, if the OS drops to EL1 or enables a hypervisor, `HCR_EL2.TSC` must remain `0`.

**[ABI]** **Register `x0`: 32-Bit SMCCC Function Identifier (`FNID`)**:

| Bits | Field Name | Value / Encoding |
|---|---|---|
| `31` | `CALL_TYPE` | `1` = Atomic / Fast Call (`ARM_SMCCC_FAST_CALL`), `0` = Yielding / Standard Call (`ARM_SMCCC_STD_CALL`) |
| `30` | `CONVENTION` | `1` = 64-bit (`ARM_SMCCC_SMC_64`), `0` = 32-bit (`ARM_SMCCC_SMC_32`) |
| `29:24` | `OWNER` | `0x02` = SiP Service (`ARM_SMCCC_OWNER_SIP`), `0x32` = Trusted OS (`ARM_SMCCC_OWNER_TRUSTED_OS`), `0x33` = TZ Apps |
| `23:16` | `RESERVED` | `0x00` |
| `15:10` | `SVC_ID` | 6-bit Qualcomm SCM Service ID (`svc`) |
| `9:0` | `CMD_ID` | 10-bit Qualcomm SCM Command ID (`cmd`) |

- Formula for 64-bit SiP SCM calls:
  $$\text{x0} = (\text{atomic} \ll 31) \mid (1 \ll 30) \mid (2 \ll 24) \mid ((\text{svc} \ \&\ 0x3\text{F}) \ll 10) \mid (\text{cmd} \ \&\ 0x3\text{FF})$$
  - Standard (yielding) SMC64 SiP call base: `0x42000000 | (svc << 10) | cmd`.
  - Fast (atomic) SMC64 SiP call base: `0xC2000000 | (svc << 10) | cmd`.

**[ABI]** **Register `x1`: Argument Info (`arginfo`)**:
- Bits `[3:0]`: Total number of arguments `N` (`0`..`10`).
- Bits `[5 + 2*i : 4 + 2*i]` (for `i = 0 .. N-1`): 2-bit type of argument `i`:
  - `0` (`SCM_VAL`): Immediate scalar value.
  - `1` (`SCM_RO`): Read-only physical memory buffer pointer.
  - `2` (`SCM_RW`): Read-write physical memory buffer pointer.
  - `3` (`SCM_BUFVAL`): Buffer value.

**[ABI]** **Registers `x2`–`x5` and Extended Argument Buffer**:
- Arguments `0`, `1`, and `2` are passed in `x2`, `x3`, and `x4`.
- If `N <= 4`, argument `3` is passed directly in `x5`.
- If `N > 4` (5 to 10 arguments), `x5` holds the **64-bit physical address** of a cache-coherent, physically contiguous 64-byte buffer containing arguments `3` through `9` as seven 64-bit little-endian words (at offsets `+0x00` through `+0x30`). This buffer must be flushed to the Point of Coherency (`dc civac`) before `smc #0` when not allocated from a non-cacheable/write-combining TrustZone pool.

**[ABI]** **Return Registers (`x0`–`x3`) and Return Status Codes**:
- On return from `smc #0`:
  - `x0`: Signed 64-bit SCM status code:
    - `> 0` (`1` = `EBUSY`): Secure world busy; retry call.
    - `0` (`SUCCESS`): Success.
    - `-1` (`ERROR`): General failure.
    - `-2` (`INTERRUPTED`): Interrupted by IRQ; re-issue the exact same `smc #0` call with the original `x0`–`x5` values!
    - `-3` (`INVALID_ARG`): Invalid argument.
    - `-4` (`ADDR_ERROR`): Invalid physical address.
    - `-5` (`NOMEM`): Out of memory.
    - `-6` (`NOT_SUPPORTED`): Call not supported.
    - `6` (`WAITQ_SLEEP`): Asynchronous wait-queue sleep; `x1` holds `wq_ctx`. The caller must wait for the SCM wait-queue interrupt (or poll `WAITQ_RESUME`, `svc = 0x24, cmd = 0x02` with `x2 = wq_ctx`) until completion.
  - `x1`, `x2`, `x3`: Result words 0, 1, and 2.

### 2.2 Complete SCM Service & Command Table

**[ABI]** **[HW]** The following SCM services and commands are used on SC8280XP:

| Service (`svc`) | Command (`cmd`) | Name | `x0` (`SMC64`) | `x1` (`arginfo`) | Arguments (`x2`, `x3`, `x4`, `x5`) | Return Values (`x1`–`x3`) |
|---|---|---|---|---|---|---|
| `0x01` (`BOOT`) | `0x0A` | `DLOAD_MODE` | `0xC200040A` | `0x002` (2 VAL) | `x2` = dload mode flags (`0` = off, `0x10` = full, `0x20` = mini), `x3` = `0` | — |
| `0x02` (`PIL`) | `0x01` | `PAS_INIT_IMAGE` | `0x42000801` | `0x022` (VAL, RW) | `x2` = `pas_id`, `x3` = phys addr of ELF+Hash metadata buffer (4 KiB aligned) | `x1` = `0` on success |
| `0x02` (`PIL`) | `0x02` | `PAS_MEM_SETUP` | `0x42000802` | `0x003` (3 VAL) | `x2` = `pas_id`, `x3` = carveout `mem_phys`, `x4` = `mem_size` | `x1` = `0` on success |
| `0x02` (`PIL`) | `0x05` | `PAS_AUTH_AND_RESET` | `0x42000805` | `0x001` (1 VAL) | `x2` = `pas_id` | `x1` = `0` on success |
| `0x02` (`PIL`) | `0x06` | `PAS_SHUTDOWN` | `0x42000806` | `0x001` (1 VAL) | `x2` = `pas_id` | `x1` = `0` on success |
| `0x02` (`PIL`) | `0x07` | `PAS_IS_SUPPORTED` | `0x42000807` | `0x001` (1 VAL) | `x2` = `pas_id` | `x1` = `1` if supported, `0` if not |
| `0x02` (`PIL`) | `0x0A` | `PAS_MSS_RESET` | `0x4200080A` | `0x002` (2 VAL) | `x2` = reset flag, `x3` = `0` | — |
| `0x06` (`INFO`) | `0x01` | `IS_CALL_AVAIL` | `0xC2001801` | `0x001` (1 VAL) | `x2` = target `FNID` (`x0` value with bit 30 set) | `x1` = `1` if available, `0` if not |
| `0x06` (`INFO`) | `0x03` | `GET_FEAT_VERSION` | `0xC2001803` | `0x001` (1 VAL) | `x2` = `feat_id` | `x1` = feature version word |
| `0x0C` (`MP`) | `0x02` | `MEM_PROTECT_VIDEO` | `0x42003002` | `0x002` (2 VAL) | `x2` = state, `x3` = spare | `x1` = status |
| `0x0C` (`MP`) | `0x16` | `ASSIGN_MEM` | `0x42003016` | `0x557` (RO,VAL,RO,VAL,RO,VAL,VAL) | `x2` = phys(`mem_to_map`), `x3` = sz, `x4` = phys(`src_vm`), `x5` = phys(extra args: `src_sz`, `dest_vm` phys, `dest_sz`, `0`) | `x1` = `0` on success |
| `0x0C` (`MP`) | `0x1C` | `SHM_BRIDGE_ENABLE` | `0x4200301C` | `0x000` | None | `x1` = status |
| `0x0C` (`MP`) | `0x1D` | `SHM_BRIDGE_DELETE` | `0x4200301D` | `0x001` (1 VAL) | `x2` = `handle` | — |
| `0x0C` (`MP`) | `0x1E` | `SHM_BRIDGE_CREATE` | `0x4200301E` | `0x004` (4 VAL) | `x2` = `pfn_and_ns_perm`, `x3` = `ipfn_and_s_perm`, `x4` = `size_and_flags`, `x5` = `ns_vmids` | `x1` = status, `x2` = `handle` |
| `0x15` (`SMMU`) | `0x01` | `SMMU_PT_FORMAT` | `0x42005401` | `0x003` (3 VAL) | `x2` = `sec_id`, `x3` = `ctx_num`, `x4` = `1` (AArch64) | — |
| `0x15` (`SMMU`) | `0x03` | `SMMU_ERRATA1` | `0xC2005403` | `0x002` (2 VAL) | `x2` = `0x2` (`CLIENT_ALL`), `x3` = enable (`0` or `1`) | — |
| `0x24` (`WAITQ`) | `0x01` | `GET_WQ_CTX` | `0xC2009001` | `0x000` | None | `x1` = `wq_ctx`, `x2` = `flags`, `x3` = `more_pending` |
| `0x24` (`WAITQ`) | `0x02` | `WAITQ_RESUME` | `0x42009002` | `0x001` (1 VAL) | `x2` = `wq_ctx` | — |
| `0x24` (`WAITQ`) | `0x03` | `WAITQ_GET_INFO` | `0xC2009003` | `0x000` | None | `x1[7:0]` = `wq_cnt`, `x2[15:0]` = `hwirq` |
| `0x28` (`GPU`) | `0x01` | `GPU_INIT_REGS` | `0x4200A001` | `0x001` (1 VAL) | `x2` = `gpu_req` bitmask | — |

**[HW]** **Empirical Probe on Radxa Dragon Q8B**:
- `IS_CALL_AVAIL` (`0xC2001801`): returns `x0 = 0, x1 = 1`.
- `PAS_IS_SUPPORTED` (`0x42000807`):
  - `pas_id = 1` (**ADSP**): returns `x0 = 0, x1 = 1` (**Supported**).
  - `pas_id = 13` (`0x0D`, **GPU Zap**): returns `x0 = 0, x1 = 1` (**Supported**).
  - `pas_id = 18` (`0x12`, **NSP0 / CDSP0**): returns `x0 = 0, x1 = 1` (**Supported**).
  - `pas_id = 30` (`0x1E`, **NSP1 / CDSP1**): returns `x0 = 0, x1 = 0` (**Not supported on this SKU**).
  - `pas_id = 4` (MPSS) and `pas_id = 12` (`0x0C`, SLPI): return `x0 = 0, x1 = 0` (**Not supported**).

---

## 3. Command DB (`0x80860000`)

### 3.1 Binary Database Layout

**[HW]** Command DB is a read-only binary database placed by boot firmware in reserved memory at physical address `0x80860000` (size `0x20000` = 128 KiB). Map this region as Normal Non-Cacheable or Write-Combining memory.

**[HW]** **1. Top-Level Database Header (at `0x80860000`, 144 bytes)**:

| Byte Offset | Size (Bytes) | Type | Field Name | Expected / Empirical Value on Dragon Q8B |
|---|---|---|---|---|
| `0x00` | `4` | 32-bit LE uint | `version` | `0x00000001` (Version 1) |
| `0x04` | `4` | 4-byte array | `magic` | `0xDB, 0x30, 0x03, 0x0C` (`0x0C0330DB` as 32-bit LE uint) |
| `0x08` | `128` (`8 * 16`) | 8 x 16-byte records | `header[0..7]` | Up to 8 slave resource headers (16 bytes each). Terminate iteration when `slv_id == 0` |
| `0x88` | `4` | 32-bit LE uint | `checksum` | Checksum (unused by OS) |
| `0x8C` | `4` | 32-bit LE uint | `reserved` | `0x00000000` |
| `0x90` | Variable | byte array | `data` | Base of entry headers and auxiliary data (`data_base = 0x80860090`) |

**[HW]** **2. Slave Resource Header (16 bytes each at `0x08 + 16 * i`)**:

| Byte Offset | Size (Bytes) | Type | Field Name | Description |
|---|---|---|---|---|
| `0x00` | `2` | 16-bit LE uint | `slv_id` | Hardware accelerator slave ID: `3` = **ARC** (Power Domains), `4` = **VRM** (Voltage Regulators), `5` = **BCM** (Bus Clock Managers / Clocks) |
| `0x02` | `2` | 16-bit LE uint | `header_offset` | Byte offset from `data` (`0x80860090`) to the first resource entry header for this slave |
| `0x04` | `2` | 16-bit LE uint | `data_offset` | Byte offset from `data` (`0x80860090`) to the start of the auxiliary data pool for this slave |
| `0x06` | `2` | 16-bit LE uint | `cnt` | Number of resource entry header records for this slave |
| `0x08` | `2` | 16-bit LE uint | `version` | Slave table version (`[15:8]` = major, `[7:0]` = minor) |
| `0x0A` | `6` | 3 x 16-bit LE uint | `reserved` | Reserved (`0`) |

**[HW]** **3. Resource Entry Header (24 bytes each at `0x80860090 + header_offset + 24 * j`)**:

| Byte Offset | Size (Bytes) | Type | Field Name | Description |
|---|---|---|---|---|
| `0x00` | `8` | 8-byte ASCII | `id` | 8-byte ASCII resource name, zero-padded on the right (NOT NUL-terminated if 8 characters long) |
| `0x08` | `8` | 2 x 32-bit LE uint | `priority` | Priority words |
| `0x10` | `4` | 32-bit LE uint | `addr` | **20-bit RPMh hardware address**: `[19:16]` = `slv_id` (`3` = ARC, `4` = VRM, `5` = BCM), `[15:0]` = accelerator register offset |
| `0x14` | `2` | 16-bit LE uint | `len` | Byte length of auxiliary data for this resource (`0` if none) |
| `0x16` | `2` | 16-bit LE uint | `offset` | Byte offset of this entry's auxiliary data relative to `0x80860090 + data_offset` |

### 3.2 Empirical Command DB Contents on Radxa Dragon Q8B

**[HW]** On the Radxa Dragon Q8B, Command DB contains three slave tables:
1. **Slave 3 (ARC, v2.0)**: **15 entries** (power domain rails). Each entry has `len = 32` bytes of auxiliary data consisting of sixteen 16-bit little-endian virtual level (`vlvl`) values indexed by hardware corner (`hlvl = 0..15`).
2. **Slave 4 (VRM, v2.0)**: **77 entries** (PMIC SMPS, LDO, and BOB regulators). Note that each VRM resource has 4 contiguous 4-byte aligned addresses (`+0x0` = voltage, `+0x4` = enable, `+0x8` = mode); two addresses refer to the same VRM resource if `addr1[19:16] == 4` and `addr1[19:4] == addr2[19:4]`.
3. **Slave 5 (BCM, v2.0)**: **51 entries** (interconnect BCMs and RPMh clocks).

| Resource Name (`id`) | Slave Type | RPMh Address (`addr`) | Aux Data Length (`len`) | Aux Data Format & Empirical Values |
|---|---|---|---|---|
| `cx.lvl` | ARC (`3`) | `0x30010` | `32` bytes | 16 x 16-bit LE `vlvl`: `[0]=0x000 (0), [1]=0x010 (16, RET), [2]=0x030 (48, MIN_SVS), [3]=0x040 (64, LOW_SVS), [4]=0x080 (128, SVS), [5]=0x0C0 (192, SVS_L1), [6]=0x100 (256, NOM), [7]=0x140 (320, NOM_L1), [8]=0x150 (336, NOM_L2), [9]=0x180 (384, TURBO), [10]=0x190 (400, TURBO_L0), [11]=0x1A0 (416, TURBO_L1), [12..15]=0` |
| `mx.lvl` | ARC (`3`) | `0x30000` | `32` bytes | 16 x 16-bit LE `vlvl`: `[0]=0, [1]=16, [2]=48, [3]=64, [4]=128, [5]=192, [6]=256, [7]=320, [8]=336, [9]=384, [10]=400, [11]=416, [12]=464, [13..15]=0` |
| `ebi.lvl` | ARC (`3`) | `0x3000C` | `32` bytes | 16 x 16-bit LE `vlvl`: `[0]=0, [1]=16, [2]=48, [3]=64, [4]=128, [5]=192, [6]=256, [7..15]=0` |
| `lcx.lvl` | ARC (`3`) | `0x30024` | `32` bytes | 16 x 16-bit LE `vlvl`: `[0]=0, [1]=16, [2]=48, [3]=64, [4]=128, [5]=192, [6]=256, [7]=320, [8]=384, [9]=416, [10..15]=0` |
| `lmx.lvl` | ARC (`3`) | `0x30028` | `32` bytes | 16 x 16-bit LE `vlvl`: `[0]=0, [1]=16, [2]=48, [3]=64, [4]=128, [5]=192, [6]=256, [7]=320, [8]=384, [9]=416, [10..15]=0` |
| `gfx.lvl` | ARC (`3`) | `0x3001C` | `32` bytes | 16 x 16-bit LE `vlvl`: `[0]=0, [1]=16, [2]=48, [3]=64, [4]=80, [5]=128, [6]=192, [7]=224, [8]=256, [9]=320, [10]=384, [11]=400, [12]=416, [13..15]=0` |
| `mmcx.lvl` | ARC (`3`) | `0x30020` | `32` bytes | 16 x 16-bit LE `vlvl`: `[0]=0, [1]=16, [2]=48, [3]=64, [4]=128, [5]=192, [6]=256, [7]=320, [8]=336, [9]=384, [10..15]=0` |
| `mxc.lvl` | ARC (`3`) | `0x30014` | `32` bytes | 16 x 16-bit LE `vlvl`: `[0]=0, [1]=16, [2]=48, [3]=128, [4]=192, [5]=256, [6]=320, [7]=336, [8]=384, [9]=400, [10]=416, [11]=448, [12]=452, [13]=456, [14]=464, [15]=0` |
| `nsp.lvl` | ARC (`3`) | `0x30018` | `32` bytes | 16 x 16-bit LE `vlvl`: `[0]=0, [1]=16, [2]=48, [3]=64, [4]=128, [5]=192, [6]=256, [7]=320, [8]=384, [9]=416, [10..15]=0` |
| `qphy.lvl` | ARC (`3`) | `0x30038` | `32` bytes | 16 x 16-bit LE `vlvl` |
| `xo.lvl` | ARC (`3`) | `0x30004` | `32` bytes | 16 x 16-bit LE `vlvl`: `[0]=0, [1]=192, [2]=512, [3..15]=0` |
| `ddr.lvl` | ARC (`3`) | `0x30008` | `32` bytes | 16 x 16-bit LE `vlvl` |
| `MC0` | BCM (`5`) | `0x50000` | `16` bytes | BCM aux record: `unit = 0x3D0900` (4,000,000), `width = 4`, `vcd = 0` |
| `SH0` | BCM (`5`) | `0x50004` | `16` bytes | BCM aux record: `unit = 0x3D0900`, `width = 16`, `vcd = 1` |
| `CN0` | BCM (`5`) | `0x50030` | `16` bytes | BCM aux record: `unit = 0x0F4240` (1,000,000), `width = 16`, `vcd = 4` |
| `MM0` | BCM (`5`) | `0x50010` | `16` bytes | BCM aux record: `unit = 0xF42400` (16,000,000), `width = 32`, `vcd = 2` |
| `SN0` | BCM (`5`) | `0x50054` | `16` bytes | BCM aux record: `unit = 0x3D0900` (4,000,000), `width = 32`, `vcd = 6` |
| `QUP0` | BCM (`5`) | `0x50044` | `16` bytes | BCM aux record: `unit = 0x002710` (10,000), `width = 4`, `vcd = 5` |

---

## 4. RPMh Resource State Coordinator (`apps_rsc`, `0x18200000`)

### 4.1 Apps RSC Geometry and Version 3.0 Register Offsets

**[HW]** The Apps RSC (`rsc@18200000`) has three DRV apertures (`drv-0` at `0x18200000`, `drv-1` at `0x18210000`, `drv-2` at `0x18220000`). The high-level OS runs on **`drv-2` (`qcom,drv-id = 2`)** at base `0x18220000`, with GICv3 interrupt **SPI 5** (GSIV 37, level-high).
- Reading `RSC_DRV_ID` (`0x18220000 + 0x00`) returns `0x00030000`: Major Version `3` (`[23:16]`), Minor Version `0` (`[15:8]`). Because `major >= 3`, the controller uses the **RSC v3.0 register offset table**.
- Reading `DRV_SOLVER_CONFIG` (`0x18220004`) returns `0x00000001` (bit `24` `DRV_HW_SOLVER` = `0`, meaning software solver mode is active; the OS flushes SLEEP and WAKE TCS sets before cluster power collapse).
- Reading `DRV_PRNT_CHLD_CONFIG` (`0x1822000C`) returns `0x481041C7`:
  - `ncpt` (Number of Commands Per TCS) = `(val >> 27) & 0x1F` = **`9` commands per TCS**.
  - `max_tcs` for `drv-2` = `(val >> (6 * 2)) & 0x3F` = **`7` TCS blocks** (`TCS 0` through `TCS 6`).
- The TCS register bank starts at `tcs_base = drv_base + 0x0D00 = 0x18220D00`.

**[HW]** **TCS Group Assignment on `apps_rsc` (`qcom,tcs-config`)**:
- `TCS 0`–`TCS 1` (2 TCSes, mask `0x03`, offset `0`): **`ACTIVE_TCS`** (type `2`) — used for immediate runtime requests (`RPMH_ACTIVE_ONLY_STATE`).
- `TCS 2`–`TCS 4` (3 TCSes, mask `0x1C`, offset `2`): **`SLEEP_TCS`** (type `0`) — triggered by hardware upon entering cluster sleep (`RPMH_SLEEP_STATE`).
- `TCS 5`–`TCS 6` (2 TCSes*, mask `0x60`, offset `5`): **`WAKE_TCS`** (type `1`) — triggered by hardware upon waking from cluster sleep (`RPMH_WAKE_ONLY_STATE`). *(Note: DT specifies 3 wake TCSes if `max_tcs >= 8`, or 2 on 7-TCS configurations; check `DRV_PRNT_CHLD_CONFIG`.)*
- `CONTROL_TCS` (type `3`, 1 control TCS): Next wakeup timer programmed at `drv_base + 0x40` (`LO`) and `drv_base + 0x38` (`HI`, with bit `31` = `VALID`).

**[HW]** **RSC v3.0 TCS Register Map (`tcs_base = 0x18220D00`, TCS Stride = `672` (`0x2A0`), Command Stride = `24` (`0x18`))**:

| Offset from `tcs_base` | Register Name | Access | Description |
|---|---|---|---|
| `0x0000` | `RSC_DRV_IRQ_ENABLE` | RW | Bitmask of TCS IDs (`1 << tcs_id`) enabled to raise GIC SPI 5 on completion (`0x03` enables `TCS 0` and `TCS 1`) |
| `0x0004` | `RSC_DRV_IRQ_STATUS` | RO | Bitmask of TCS IDs with completed transfers (`1 << tcs_id`) |
| `0x0008` | `RSC_DRV_IRQ_CLEAR` | WO | Write `1 << tcs_id` to clear the completion interrupt for `tcs_id` |
| `0x02A0 * m + 0x0020` | `RSC_DRV_CMD_WAIT_FOR_CMPL(m)` | RW | Per-command chain dependency mask for TCS `m` |
| `0x02A0 * m + 0x0024` | `RSC_DRV_CONTROL(m)` | RW | TCS `m` control: Bit `16` = `TCS_AMC_MODE_ENABLE`, Bit `24` = `TCS_AMC_MODE_TRIGGER` |
| `0x02A0 * m + 0x0028` | `RSC_DRV_STATUS(m)` | RO | TCS `m` idle/busy status (`1` = idle, `0` = busy in hardware) |
| `0x02A0 * m + 0x002C` | `RSC_DRV_CMD_ENABLE(m)` | RW | 16-bit bitmask (`1 << j`) of valid command slots (`j = 0..8`) in TCS `m` |
| `0x02A0 * m + 0x0034 + 0x18 * j` | `RSC_DRV_CMD_MSGID(m, j)` | RW | Command `j` message header: `[7:0]` = `LEN` (`8`), `[8]` = `RESP_REQ` (`1` = wait for ack), `[16]` = `WRITE` (`1` = write, `0` = read) |
| `0x02A0 * m + 0x0038 + 0x18 * j` | `RSC_DRV_CMD_ADDR(m, j)` | RW | Command `j` 20-bit RPMh address (from Command DB) |
| `0x02A0 * m + 0x003C + 0x18 * j` | `RSC_DRV_CMD_DATA(m, j)` | RW | Command `j` 32-bit payload data |
| `0x02A0 * m + 0x0040 + 0x18 * j` | `RSC_DRV_CMD_STATUS(m, j)` | RO | Command `j` status: `[8]` = `CMD_STATUS_ISSUED`, `[16]` = `CMD_STATUS_COMPL` |
| `0x02A0 * m + 0x0044 + 0x18 * j` | `RSC_DRV_CMD_RESP_DATA(m, j)` | RO | Command `j` 32-bit read response data (for read requests) |

### 4.2 Step-by-Step Active-Only TCS Transaction Sequence (`AMC` Trigger)

**[HW]** To send an immediate `RPMH_ACTIVE_ONLY_STATE` request containing `K` commands (`1 <= K <= 9`) on an active TCS `m` (`m = 0` or `1`):
1. Ensure no currently in-flight active TCS has an enabled command targeting the same resource address (see Section 3.2 for VRM 4-address resource matching).
2. For each command `j = 0 .. K-1`:
   - Construct `msgid = 8` (`CMD_MSGID_LEN`).
   - If the request is a write, set bit `16` (`CMD_MSGID_WRITE`, `0x10000`).
   - If completion acknowledgment is required, set bit `8` (`CMD_MSGID_RESP_REQ`, `0x100`). (For a synchronous write, `msgid = 0x00010108`; for an async fire-and-forget write, `msgid = 0x00010008`; for a read, `msgid = 0x00000108`.)
   - Write `msgid` to `RSC_DRV_CMD_MSGID(m, j)` (`+0x34 + 0x18 * j`).
   - Write the target 20-bit resource `addr` to `RSC_DRV_CMD_ADDR(m, j)` (`+0x38 + 0x18 * j`).
   - If writing, write the 32-bit `data` payload to `RSC_DRV_CMD_DATA(m, j)` (`+0x3C + 0x18 * j`).
3. Write the bitmask of enabled slots `((1 << K) - 1)` to `RSC_DRV_CMD_ENABLE(m)` (`+0x2C`).
4. **Trigger the TCS (AMC Mode Hardware Sequence)**:
   - Read `ctrl = RSC_DRV_CONTROL(m)` (`+0x24`).
   - Clear bit `24` (`TCS_AMC_MODE_TRIGGER`): `ctrl &= ~(1 << 24)`. Write `ctrl` to `RSC_DRV_CONTROL(m)` and **poll `RSC_DRV_CONTROL(m)` until it reads back equal to `ctrl`**.
   - Clear bit `16` (`TCS_AMC_MODE_ENABLE`): `ctrl &= ~(1 << 16)`. Write `ctrl` to `RSC_DRV_CONTROL(m)` and **poll `RSC_DRV_CONTROL(m)` until it reads back equal to `ctrl`**.
   - Set bit `16` (`ctrl = (1 << 16)`). Write `ctrl` to `RSC_DRV_CONTROL(m)` and **poll `RSC_DRV_CONTROL(m)` until it reads back equal to `(1 << 16)`**.
   - Set bit `24` (`ctrl = (1 << 16) | (1 << 24) = 0x01010000`) and write `ctrl` to `RSC_DRV_CONTROL(m)` to fire the transfer.
5. **Completion Handling**:
   - Wait for bit `m` (`1 << m`) to assert in `RSC_DRV_IRQ_STATUS` (`tcs_base + 0x04`).
   - If the request was a read, read the response from `RSC_DRV_CMD_RESP_DATA(m, 0)` (`+0x44`).
   - Clear `TCS_AMC_MODE_TRIGGER` (bit `24`, with readback sync) and `TCS_AMC_MODE_ENABLE` (bit `16`, with readback sync) in `RSC_DRV_CONTROL(m)` (`+0x24`).
   - Write `0` to `RSC_DRV_CMD_ENABLE(m)` (`+0x2C`).
   - Write `(1 << m)` to `RSC_DRV_IRQ_CLEAR` (`tcs_base + 0x08`).

### 4.3 RPMh Power Domain (ARC) and Bus Clock Manager (BCM) Formats

**[HW]** **1. Power Domain Voting via ARC**:
- Look up the power domain's resource name (e.g., `"cx.lvl"`, `"mx.lvl"`, `"lcx.lvl"`, `"lmx.lvl"`, `"nsp.lvl"`, `"gfx.lvl"`, `"mmcx.lvl"`, `"mxc.lvl"`, `"ebi.lvl"`, `"qphy.lvl"`) in Command DB to obtain its 20-bit `addr` (`0x300xx`) and its 16-entry 16-bit LE `vlvl` array.
- Note that `vlvl` values (e.g., `48 = MIN_SVS`, `64 = LOW_SVS`, `128 = SVS`, `192 = SVS_L1`, `256 = NOM`, `320 = NOM_L1`, `384 = TURBO`, `416 = TURBO_L1`) are **NOT** written directly to the hardware. Instead, find the smallest index `hlvl` (`0`..`15`) where `vlvl[hlvl] >= requested_vlvl` (skipping `vlvl == 16` `RETENTION`), and write the **index `hlvl` (`0`..`15`)** as the 32-bit `data` payload to `addr`.
- To power off a domain, write `data = 0`. To power on a domain at its minimum operational corner, write `data = enable_corner` (the first index `i` with `vlvl[i] > 16`, typically `hlvl = 2` for `MIN_SVS = 48`).

**[HW]** **2. Interconnect Bandwidth Voting via BCM**:
- Look up the BCM name (e.g., `"MC0"`, `"SH0"`, `"CN0"`, `"MM0"`, `"SN0"`) in Command DB to obtain its 20-bit `addr` (`0x500xx`) and 8-byte BCM auxiliary data record (32-bit LE `unit` at `+0x00`, 16-bit LE `width` at `+0x04`, 8-bit `vcd` at `+0x06`, 8-bit `reserved` at `+0x07`).
- Convert requested average bandwidth (`sum_avg` in bytes/sec) and peak bandwidth (`max_peak` in bytes/sec) on a node with `buswidth` and `channels` into 14-bit `vote_x` and `vote_y` values (`0`..`0x3FFF`):
  $$\text{agg\_avg} = \left\lceil \frac{\text{sum\_avg} \times \text{bcm\_width}}{\text{node\_buswidth} \times \text{node\_channels}} \right\rceil, \quad \text{vote\_x} = \left\lceil \frac{\text{agg\_avg} \times \text{vote\_scale}}{\text{bcm\_unit}} \right\rceil$$
  $$\text{agg\_peak} = \left\lceil \frac{\text{max\_peak} \times \text{bcm\_width}}{\text{node\_buswidth}} \right\rceil, \quad \text{vote\_y} = \left\lceil \frac{\text{agg\_peak} \times \text{vote\_scale}}{\text{bcm\_unit}} \right\rceil$$
  (where `vote_scale` is `1000` if bandwidth is in kB/s, or `1` for clock BCMs `QUP0`–`QUP2`). Clamp `vote_x` and `vote_y` to `0x3FFF`.
- Group all BCM commands by ascending `vcd` (Virtual Clock Domain), and set the `COMMIT` bit (`bit 30`) on the **last** BCM command of each `vcd` group:

| Bits | Field Name | Description |
|---|---|---|
| `30` | `COMMIT` | `1` on the final BCM command in a Virtual Clock Domain (`vcd`) batch to latch all votes in that `vcd` |
| `29` | `VALID` | `1` if `vote_x != 0 || vote_y != 0`; `0` if both votes are `0` |
| `27:14` | `VOTE_X` | 14-bit aggregated average bandwidth vote (`0`..`16383`) |
| `13:0` | `VOTE_Y` | 14-bit aggregated peak bandwidth vote (`0`..`16383`) or `enable_mask` (`0x08` / bit `3` for `ACV`) |

---

## 5. ADSP and NSP0 Remoteproc Loading via PAS

### 5.1 Subsystem Parameters on Radxa Dragon Q8B

**[HW]** **[ABI]** The SC8280XP uses the Qualcomm Peripheral Authentication Service (PAS) to load and boot the Audio DSP (`ADSP`) and Neural/Compute DSP (`NSP0`):

| Parameter | ADSP (`remoteproc@3000000`) | NSP0 / CDSP0 (`remoteproc@1b300000`) |
|---|---|---|
| PAS ID (`pas_id`) | **`1`** (`0x01`) | **`18`** (`0x12`) |
| Firmware Path | `qcom/sc8280xp/radxa/dragon-q8b/qcadsp8280.mbn` | `qcom/sc8280xp/qccdsp8280.mbn` |
| Carveout Physical Base (`mem_phys`) | **`0x86C00000`** (`adsp-region@86c00000`) | **`0x8A100000`** (`cdsp0-region@8a100000`) |
| Carveout Size (`mem_size`) | **`0x02000000`** (32 MiB) | **`0x01E00000`** (30 MiB) |
| Proxy Power Domains | `"lcx"` (`0x30024`), `"lmx"` (`0x30028`) | `"nsp"` (`0x30018`), `"cx"` (`0x30010`), `"mxc"` (`0x30014`) |
| Clocks | `"xo"` (19.2 MHz RPMh XO clock) | `"xo"` (19.2 MHz RPMh XO clock) |
| AOSS QMP Load State String | `"adsp"` (`{class: image, res: load_state, name: adsp, val: on}`) | None |
| SMEM Crash Reason ID | `423` | `601` |
| Watchdog IRQ (`wdog`) | GIC SPI 162 (`0xA2`, rising-edge) | GIC SPI 578 (`0x242`, rising-edge) |
| SMP2P Inbound IRQs (`slave-kernel`) | `fatal` = bit 0, `ready` = bit 1, `handover` = bit 2, `stop-ack` = bit 3, `shutdown-ack` = bit 7 (Remote PID `2`, IPCC client `3` sig `2`) | `fatal` = bit 0, `ready` = bit 1, `handover` = bit 2, `stop-ack` = bit 3 (Remote PID `5`, IPCC client `6` sig `2`) |
| SMP2P Outbound State (`master-kernel`) | `stop` = bit 0 (Remote PID `2`, IPCC client `3` sig `2`) | `stop` = bit 0 (Remote PID `5`, IPCC client `6` sig `2`) |
| GLINK Edge | Label `"lpass"`, Remote PID `2`, IPCC client `3` sig `0` | Label `"nsp0"`, Remote PID `5`, IPCC client `6` sig `0` |

### 5.2 MBN / MDT Firmware Format and Step-by-Step PAS Boot Sequence

**[HW]** Qualcomm `.mbn` (single-file) and `.mdt` + `.bXX` (split-file) firmware images are 32-bit ELF (`ELFCLASS32`) binaries containing:
- Program Header `0`: The ELF header + program header table (`p_type != PT_LOAD`).
- Hash Segment (where `(p_flags & 0x07000000) == 0x02000000`, `MDT_TYPE_HASH`): Contains cryptographic signatures and segment hashes verified by TrustZone.
- Loadable Segments (where `p_type == PT_LOAD`, `(p_flags & 0x07000000) != 0x02000000`, and `p_memsz > 0`):
  - If bit `27` (`0x08000000`, `MDT_RELOCATABLE`) is set in `p_flags` on any loadable segment, the image is **relocatable**.

**[HW]** **Step-by-Step PAS Firmware Load and Boot Sequence**:
1. **Enable Proxy Resources**:
   - Vote the subsystem's proxy power domains (`lcx` + `lmx` for ADSP; `nsp` + `cx` + `mxc` for NSP0) to their maximum performance corner via RPMh ARC, and enable the `xo` clock.
   - If the subsystem uses AOSS QMP load-state notification (ADSP), send `{class: image, res: load_state, name: adsp, val: on}` to the AOSS QMP mailbox (`0x0C300000`).
2. **Parse ELF Headers and Compute Address Range**:
   - Validate the 32-bit ELF magic (`\x7fELF`) and iterate over all loadable segments (`PT_LOAD` and not `MDT_TYPE_HASH` with `p_memsz > 0`):
     - Compute `min_addr = min(p_paddr)` and `max_addr = ALIGN(max(p_paddr + p_memsz), 4096)`.
     - Check whether any loadable segment has `MDT_RELOCATABLE` (`1 << 27`) set in `p_flags`.
3. **Construct and Authenticate Metadata (`PAS_INIT_IMAGE`)**:
   - Allocate a physically contiguous, 4 KiB-aligned buffer of size `metadata_len = phdr[0].p_filesz + phdr[hash_segment].p_filesz`.
   - Copy `phdr[0].p_filesz` bytes from the start of the `.mbn`/`.mdt` file (the ELF header and program header table) to offset `0` of the metadata buffer.
   - Copy `phdr[hash_segment].p_filesz` bytes of the hash segment immediately after `phdr[0].p_filesz` in the metadata buffer (with **no padding** in between: for single-file `.mbn`, copy from `fw_data + phdr[hash_segment].p_offset`; for split `.mdt`, copy from `fw_data + phdr[0].p_filesz` or `.b01`).
   - Flush the metadata buffer to physical memory (`dc civac`) and invoke SCM `PAS_INIT_IMAGE` (`svc = 0x02, cmd = 0x01`, `x0 = 0x42000801`, `x1 = 0x22`, `x2 = pas_id`, `x3 = metadata_phys`).
   - *Note*: Keep the metadata buffer allocated until `PAS_AUTH_AND_RESET` completes!
4. **Configure Relocation (`PAS_MEM_SETUP`)**:
   - If `relocate` is true, invoke SCM `PAS_MEM_SETUP` (`svc = 0x02, cmd = 0x02`, `x0 = 0x42000802`, `x1 = 0x03`, `x2 = pas_id`, `x3 = mem_phys`, `x4 = max_addr - min_addr`).
5. **Load `PT_LOAD` Segments into Carveout Memory**:
   - Map the carveout `[mem_phys, mem_phys + mem_size)` as write-combining memory.
   - Set `mem_reloc = relocate ? min_addr : mem_phys`.
   - For each loadable segment (`PT_LOAD`, non-hash, `p_memsz > 0`):
     - Compute `offset = p_paddr - mem_reloc`. Verify `offset >= 0` and `offset + p_memsz <= mem_size`.
     - Copy `p_filesz` bytes from `fw_data + p_offset` (or from split file `.bXX`) to `carveout_base + offset`.
     - Zero-fill the trailing BSS region (`carveout_base + offset + p_filesz`) for `p_memsz - p_filesz` bytes.
   - Unmap or flush the write-combining carveout mapping (`dsb sy`).
6. **Authenticate Segments and Release Reset (`PAS_AUTH_AND_RESET`)**:
   - Invoke SCM `PAS_AUTH_AND_RESET` (`svc = 0x02, cmd = 0x05`, `x0 = 0x42000805`, `x1 = 0x01`, `x2 = pas_id`).
   - TrustZone hashes every loaded `PT_LOAD` segment in the carveout against the authenticated hash table, configures the subsystem XPU/MSA locks, and deasserts the Q6V5 processor reset.
7. **Wait for Subsystem Ready and Handover**:
   - Wait up to 5000 ms **[POLICY]** for the SMP2P `ready` interrupt (bit `1` of the subsystem's inbound `slave-kernel` SMP2P entry).
   - Once the `handover` SMP2P interrupt (bit `2`) fires, the remote processor has placed its own votes on RPMh; the host OS may then release its proxy power domain votes (`lcx`/`lmx` or `nsp`/`cx`/`mxc`) and `xo` clock vote.
   - Release the temporary metadata buffer.
8. **Shutdown Sequence**:
   - Set the `stop` bit (bit `0`) in the outbound `master-kernel` SMP2P state entry and wait up to 5000 ms **[POLICY]** for the `stop-ack` SMP2P interrupt (bit `3`). Clear the `stop` bit.
   - Invoke SCM `PAS_SHUTDOWN` (`svc = 0x02, cmd = 0x06`, `x0 = 0x42000806`, `x1 = 0x01`, `x2 = pas_id`).
   - If `handover` never occurred, release proxy power domains and clocks; if AOSS QMP `load_state` was enabled, send `{class: image, res: load_state, name: adsp, val: off}`.
