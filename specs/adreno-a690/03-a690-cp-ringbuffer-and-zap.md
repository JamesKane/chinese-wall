# Adreno 690 Command Processor, Ringbuffers, Preemption, and Zap Shader

## 1. Architecture Parameters & Hardware Schema References

**[HW]** The Adreno 690 GPU on the Qualcomm SC8280XP SoC belongs to the Generation 4 Adreno 6xx (`A660`-class) architecture family and reports a hardware Chip ID of `0x06090000` (or `0x06090001` on Patch Revision 1 silicon).

**[ABI]** Please note that the upstream `drivers/gpu/drm/msm/registers/` XML hardware schema files (`a6xx.xml`, `adreno_pm4.xml`, `adreno_common.xml`, and `a6xx_enums.xml`) are licensed under MIT/BSD-2-Clause and may be referenced directly by clean-room implementers for exhaustive 3D pipeline register and enum definitions. Unless otherwise specified, all register offsets in this specification are **32-bit dword indices** relative to the GPU 3D `core` aperture (`0x03d00000`); multiply any dword index by `4` to obtain its byte offset.

| Parameter | Value | Tag | Description |
| :--- | :--- | :--- | :--- |
| Chip ID (`revn = 0`) | `0x06090000` (`0x06090001` rev 1) | **[HW]** | 32-bit hardware revision identifier (`core = 6`, `major = 9`, `minor = 0`, `patchid = 0` or `1`). |
| Architecture Family | Adreno 6xx Gen 4 (A660 family) | **[HW]** | Uses `a660_sqe.fw` microcode, A650/A660-style UCHE, and UBWC 4.0. |
| Hardware Feature Flags | `HAS_CACHED_COHERENT` (`0x10`), `HAS_HW_APRIV` (`0x08`) | **[HW]** | Supports hardware-enforced privileged memory accesses (`CP_APRIV_CNTL`) and I/O-coherent (`IOMMU_CACHE`) CPU/GPU buffers. |
| On-Chip GMEM Size | `0x00400000` (`4 MiB`) / `0x001b9000` (`1764 KiB`) | **[HW]** | The hardware catalog reports `4 MiB` (`0x400000` bytes) across 4 shader processor slices (`GMEM_BASE = 0x0`), while some empirical board documentation notes `1764 KiB` (`0x1b9000` bytes) across 4 slices. |
| Primitive FIFO Threshold | `0x00800200` | **[HW]** | Programmed into `PC_DBG_ECO_CNTL` (`0x9e00`). |
| GMU CGC Mode | `0x00020200` | **[HW]** | Programmed into `GPU_GMU_AO_GMU_CGC_MODE_CNTL` (`0x23b09`). |
| UCHE Trap Base | `0x1fffffffff000` | **[HW]** | Programmed into `UCHE_TRAP_BASE` (`0x0e09` LO, `0x0e0a` HI) and `UCHE_WRITE_THRU_BASE` (`0x0e07` LO, `0x0e08` HI). |
| UBWC / DDR Configuration | UBWC 4.0, `highest_bank_bit = 16`, Macrotile Mode `1` (8-channel) | **[HW]** | Memory controller bank swizzling and Universal Bandwidth Compression parameters on SC8280XP. |

---

## 2. CP Microcode (`qcom/a660_sqe.fw`) & Kernel Shared Buffers

### 2.1 SQE Microcode Loading

**[HW]** The Adreno 690 Command Processor (CP) executes instructions on its Sequencer Engine (SQE) from a firmware image named `qcom/a660_sqe.fw`:
1. **[ABI]** The first 4 bytes (`offset 0x00..0x03`) of `qcom/a660_sqe.fw` are a legacy metadata header and **must be skipped**.
2. **[HW]** Allocate a write-combining (`WC`), GPU-read-only buffer of `firmware_byte_size - 4` bytes mapped into GPU SMMU Context Bank 0 (`kernel_vm`), copy the firmware bytes starting at file offset `4` into the buffer, and program its 64-bit IOVA into `CP_SQE_INSTR_BASE_LO` (`0x0830`) and `CP_SQE_INSTR_BASE_HI` (`0x0831`).

### 2.2 Hardware RPTR Shadow Buffer (`shadow`)

**[HW]** Because Adreno 690 implements hardware privileged access control (`HAS_HW_APRIV`), the Command Processor automatically writes back each ringbuffer's current read pointer (`RPTR`) to a privileged shadow array in GPU virtual memory whenever `CP_RB_CNTL.NO_UPDATE` (Bit 27) is `0`.
- **[HW]** Allocate a privileged (`WC | MAP_PRIV`) buffer of `4 * nr_rings` bytes (`16` bytes for 4 ringbuffers). Ring `i` (`0..3`) uses the 32-bit word at `shadow_iova + (i * 4)`, programmed into `CP_RB_RPTR_ADDR_LO` (`0x0804`) and `CP_RB_RPTR_ADDR_HI` (`0x0805`).

### 2.3 Per-Ring Shared Memory Pointers (`memptrs`)

**[POLICY]** For each ringbuffer (`0..3`), the host driver allocates a write-combining (`WC`) shared tracking buffer (`2080` bytes = `0x820` bytes) mapped into the kernel GPU address space at `memptrs_iova`:

| Byte Offset | Size | Type | Name | Tag | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `+0x000` | 4 | `uint32` | `rptr` | **[POLICY]** | Software-tracked read pointer (unused when hardware `shadow` is active). |
| `+0x004` | 4 | `uint32` | `fence` | **[HW]** | Monotonically increasing completed submission sequence number written by `CP_EVENT_WRITE(CACHE_FLUSH_TS)`. |
| `+0x008` | 4 | `uint32` | `bv_rptr` | **[HW]** | Concurrent binning visibility pass read pointer written by hardware during preemption (`bv_rptr_addr`). |
| `+0x00c` | 4 | `uint32` | `bv_fence` | **[HW]** | Binning visibility pass fence word. |
| `+0x010` | 2048 | `uint64[256]` | `stats[64]` | **[POLICY]** | Array of 64 per-submission profiling records (`32` bytes each): `+0x00` `cpcycles_start` (`uint64`), `+0x08` `cpcycles_end` (`uint64`), `+0x10` `alwayson_start` (`uint64`), `+0x18` `alwayson_end` (`uint64`). |
| `+0x810` | 8 | `uint64` | `ttbr0` | **[POLICY]** | Saved copy of the active user `TTBR0` value written via `CP_MEM_WRITE` on page-table switches for crash diagnostics. |
| `+0x818` | 4 | `uint32` | `context_idr` | **[POLICY]** | Saved copy of the active user `CONTEXTIDR` written via `CP_MEM_WRITE`. |
| `+0x81c` | 4 | `uint32` | `perfcntr_fence` | **[POLICY]** | Sequence number for performance counter sample synchronization. |

---

## 3. Priority Ringbuffers & Fenced Register Writes

### 3.1 Ringbuffer Geometry & Control Registers

**[HW]** Adreno 690 supports up to **4 priority ringbuffers** (`nr_rings = 4`, where Ring 0 is highest priority and Ring 3 is lowest priority). Each ringbuffer is a circular buffer of **`32 KiB` (`0x8000` bytes = `8192` 32-bit dwords)** allocated with write-combining and privileged attributes (`WC | MAP_PRIV`) and aligned to a `32 KiB` boundary.

| Dword Offset | Byte Offset | Register Name | Tag | Bitfields & Programming Value |
| :--- | :--- | :--- | :--- | :--- |
| `0x0800` | `0x02000` | `CP_RB_BASE_LO` | **[HW]** | Lower 32 bits of the active ringbuffer's 64-bit IOVA (`32 KiB` aligned). |
| `0x0801` | `0x02004` | `CP_RB_BASE_HI` | **[HW]** | Upper 32 bits of the active ringbuffer's 64-bit IOVA. |
| `0x0802` | `0x02008` | `CP_RB_CNTL` | **[HW]** | Bits 5:0 = `BUFSZ` ($\log_2(\text{size in qwords}) = \log_2(32768 / 8) = 12 = \text{0x0c}$); Bits 13:8 = `BLKSZ` ($\log_2(32 / 8) = 2 = \text{0x02}$); Bit 27 = `NO_UPDATE` (`0` on A690 so HW updates `shadow`); Bit 31 = `RPTR_WR_EN`. Write **`0x0000020c`**. |
| `0x0804` | `0x02010` | `CP_RB_RPTR_ADDR_LO` | **[HW]** | Lower 32 bits of `shadow_iova + (ring_id * 4)`. |
| `0x0805` | `0x02014` | `CP_RB_RPTR_ADDR_HI` | **[HW]** | Upper 32 bits of `shadow_iova + (ring_id * 4)`. |
| `0x0806` | `0x02018` | `CP_RB_RPTR` | **[HW]** | Current read pointer in 32-bit dwords (`0..8191`). |
| `0x0807` | `0x0201c` | `CP_RB_WPTR` | **[HW]** | Current write pointer in 32-bit dwords (`0..8191`). Must be updated using a fenced write. |
| `0x0808` | `0x02020` | `CP_SQE_CNTL` | **[HW]** | Write `0x00000001` to enable the SQE and begin fetching from the ringbuffer (`0x00000003` halts the SQE on hang recovery). |

### 3.2 Fenced Register Writes (`GMU_AHB_FENCE_STATUS`)

**[HW]** When GMU Inter-Frame Power Collapse (IFPC) is enabled, the GMU may power-collapse the GX 3D domain while the host CPU is writing to 3D registers over the AHB bus. Any write to `CP_RB_WPTR` (`0x0807`) or the context-switch preemption registers (`0x08a0..0x08a4`) must use the GMU AHB fence handshake to guarantee the write reaches the 3D hardware:

1. **[HW]** Determine the fence status bitmask:
   - For `CP_RB_WPTR` (`0x0807`): `fence_mask = 0x00000001` (Bit 0, `WRITEDROPPED0`).
   - For `CP_CONTEXT_SWITCH_*` (`0x08a0..0x08a4`): `fence_mask = 0x00000002` (Bit 1, `WRITEDROPPED1`).
2. **[HW]** Write the target 32-bit value to the 3D MMIO register and execute a full CPU memory barrier (`dsb sy`).
3. **[HW]** Poll `GMU_AHB_FENCE_STATUS` (dword `0x23b13`, byte offset `0x8ec4c` from `0x03d00000`) for up to `1 ms` (retrying once for an additional `1 ms` on timeout):
   - If `(GMU_AHB_FENCE_STATUS & fence_mask) == 0`, the write succeeded; return.
   - If `(GMU_AHB_FENCE_STATUS & fence_mask) != 0`, the write was dropped while GX was collapsing: delay `10 us`, re-write the target 32-bit value to the 3D register, execute a full memory barrier (`dsb sy`), and re-check `GMU_AHB_FENCE_STATUS`.

---

## 4. PM4 Packet Formats & Command Processor Opcodes

### 4.1 Type-4 (`PKT4`) and Type-7 (`PKT7`) Packet Headers

**[HW]** All commands in the ringbuffer and indirect buffers use 32-bit PM4 Type-4 (register write) or Type-7 (CP opcode) headers protected by an **odd parity bit**:
- **[HW]** **Odd Parity Calculation**: The parity bit (`PARITY`) is set to `1` if the number of `1` bits across the remaining 31 bits of the header word is **even**, and `0` if the number of `1` bits across the remaining 31 bits is **odd** (ensuring the complete 32-bit header always contains an odd number of set bits).

| Packet Type | Bits 31:28 (`TYPE`) | Bits 27:16 | Bit 15 | Bits 14:8 | Bit 7 | Bits 6:0 | Description |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Type-4 (`PKT4`)** | `0x4` | `00` (Bits 27:26), `BASE_INDEX[17:8]` (Bits 25:16) | `BASE_INDEX[7]` | `BASE_INDEX[6:0]` | `PARITY` (odd parity) | `COUNT[6:0]` (`0..127`) | Writes `COUNT` sequential 32-bit dwords starting at dword register offset `BASE_INDEX[17:0]`. |
| **Type-7 (`PKT7`)** | `0x7` | `00000` (Bits 27:23), `OPCODE[6:0]` (Bits 22:16) | `PARITY` (odd parity) | `COUNT[14:8]` | `COUNT[7]` | `COUNT[6:0]` | Executes CP command `OPCODE[6:0]` consuming `COUNT[14:0]` following payload dwords. |

### 4.2 Essential Type-7 Opcodes (`PKT7`)

**[HW]** The host kernel driver emits the following Type-7 packets in ringbuffers for initialization, per-process page table switching, indirect buffer dispatch, cache flushing, fence signaling, and preemption:

| Opcode | Mnemonic | Payload Dwords | Tag | Payload Layout & Behavior |
| :--- | :--- | :--- | :--- | :--- |
| `0x10` | `CP_NOP` | `N` | **[HW]** | Skips `N` payload dwords. |
| `0x12` | `CP_WAIT_MEM_WRITES` | `0` | **[HW]** | Stalls until all outstanding GPU memory writes have completed. |
| `0x13` | `CP_WAIT_FOR_ME` | `0` | **[HW]** | Stalls the Prefetch Parser (PFP) until the PFP-to-Micro-Engine (ME) FIFO drains. |
| `0x14` | `CP_WAIT_MEM_GTE` | `4` | **[HW]** | Dword 0: `0x00000000`; Dword 1: `POLL_ADDR_LO`; Dword 2: `POLL_ADDR_HI`; Dword 3: `REF`. Stalls until the signed 32-bit value at `POLL_ADDR >= REF`. |
| `0x21` | `CP_REG_RMW` | `3` | **[HW]** | Read-modify-write register: Dword 0 = `DST_REG[17:0] \| (ROTATE << 24) \| (SRC1_ADD << 29) \| (SRC1_IS_REG << 30) \| (SRC0_IS_REG << 31)`; Dword 1 = `SRC0` (AND mask/reg); Dword 2 = `SRC1` (OR/ADD value/reg). |
| `0x26` | `CP_WAIT_FOR_IDLE` | `0` | **[HW]** | Stalls until the entire 3D pipeline is idle. |
| `0x3c` | `CP_WAIT_REG_MEM` | `6` | **[HW]** | Polls register or memory until condition holds: Dword 0 = `FUNCTION[2:0]` (`3` = `EQ`, `4` = `NE`, `5` = `GE`) `\| (SIGNED << 3) \| (POLL_TYPE[1:0] << 4)` (`0` = reg, `1` = mem, `2` = scratch) `\| (WRITE_MEMORY << 8)`; Dword 1 = `POLL_ADDR_LO` (or reg offset); Dword 2 = `POLL_ADDR_HI`; Dword 3 = `REF`; Dword 4 = `MASK`; Dword 5 = `DELAY_LOOP_CYCLES`. |
| `0x3d` | `CP_MEM_WRITE` | `2 + N` | **[HW]** | Writes `N` 32-bit dwords to GPU memory: Dword 0 = `ADDR_LO`; Dword 1 = `ADDR_HI`; Dwords `2..N+1` = sequential 32-bit values. |
| `0x3e` | `CP_REG_TO_MEM` | `3` | **[HW]** | Copies register(s) to memory: Dword 0 = `REG[17:0] \| (CNT[11:0] << 18) \| (IS_64B << 30) \| (ACCUMULATE << 31)`; Dword 1 = `DEST_ADDR_LO`; Dword 2 = `DEST_ADDR_HI`. |
| `0x3f` | `CP_INDIRECT_BUFFER` | `3` | **[HW]** | Dispatches an indirect command buffer (`IB1`): Dword 0 = `IB_BASE_LO`; Dword 1 = `IB_BASE_HI`; Dword 2 = `IB_SIZE_DWORDS[19:0]`. |
| `0x46` | `CP_EVENT_WRITE` | `1` or `4` | **[HW]** | Triggers a hardware event: Dword 0 = `EVENT[7:0] \| (TIMESTAMP << 30) \| (IRQ << 31)`. Key events: `CACHE_FLUSH_TS` (`0x04`, 4 dwords: Dword 0 = `0x80000004` with `IRQ`, Dwords 1..2 = 64-bit `fence` IOVA, Dword 3 = `seqno`), `PC_CCU_INVALIDATE_DEPTH` (`0x18`), `PC_CCU_INVALIDATE_COLOR` (`0x19`), `PC_CCU_FLUSH_DEPTH_TS` (`0x1c`), `PC_CCU_FLUSH_COLOR_TS` (`0x1d`), `CCU_RESOLVE` (`0x1e`), `LRZ_CLEAR` (`0x25`), `LRZ_FLUSH_INVALIDATE` (`0x26`), `CACHE_INVALIDATE` (`0x31`). |
| `0x48` | `CP_ME_INIT` | `8` | **[HW]** | Initializes the CP Micro-Engine after cold boot or resume: Dword 0 = `0x0000002f` (ordinal mask); Dword 1 = `0x00000003` (multiple hardware contexts); Dword 2 = `0x20000000` (error detection); Dwords 3..7 = `0x00000000`. |
| `0x53` | `CP_SMMU_TABLE_UPDATE` | `4` | **[HW]** | In-band per-process SMMU Context Bank 0 page table switch: Dword 0 = `TTBR0_LO`; Dword 1 = `TTBR0_HI[15:0] \| (ASID[15:0] << 16)`; Dword 2 = `CONTEXTIDR` (`0x00000000`); Dword 3 = `CONTEXTBANK` (`0x00000000`). |
| `0x55` | `CP_SET_AMBLE` | `3` | **[HW]** | Registers a preemption save/restore command buffer: Dword 0 = `ADDR_LO`; Dword 1 = `ADDR_HI`; Dword 2 = `DWORDS[19:0] \| (TYPE[1:0] << 20)` (`0` = preamble, `1` = bin preamble, `2` = postamble, `3` = KMD postamble). |
| `0x56` | `CP_SET_PSEUDO_REG` | `3 * N` | **[HW]** | Sets internal CP preemption pointers using `N` 3-dword triplets `(REG_ID, ADDR_LO, ADDR_HI)`, where `REG_ID` is `0` (`SMMU_INFO`), `1` (`NON_SECURE_SAVE_ADDR`), `2` (`SECURE_SAVE_ADDR`), `3` (`NON_PRIV_SAVE_ADDR`), or `4` (`COUNTER`). |
| `0x5f` | `CP_SET_PROTECTED_MODE` | `1` | **[HW]** | Dword 0 = `0x00000000` disables register protection (allowing ringbuffer writes to protected registers); `0x00000001` re-enables protection. |
| `0x65` | `CP_SET_MARKER` | `1` | **[HW]** | Sets render pass or IFPC marker: when Bit 8 (`MARKER_MODE`) is `1`, Bit 0 controls IFPC (`0x00000101` disables IFPC during a submit; `0x00000100` re-enables IFPC after completion). When Bit 8 is `0`, Bits 3:0 denote pass mode (`13` = `IB1LIST_START`, `14` = `IB1LIST_END`). |
| `0x66` | `CP_SET_SECURE_MODE` | `1` | **[HW]** | Dword 0 = `0x00000000` transitions the GPU from secure mode to non-secure mode after Zap shader authentication (packet words: `0x70668001`, `0x00000000`). |
| `0x6b` | `CP_CONTEXT_SWITCH_YIELD` | `4` | **[HW]** | Cooperative preemption yield point emitted at the end of each submission: Dword 0 = `0x00000000` (`ADDR_LO`); Dword 1 = `0x00000000` (`ADDR_HI`); Dword 2 = `0x00000001` (`DATA`); Dword 3 = `0x00000000` (`INTERRUPT`). |

### 4.3 Standard Ringbuffer Command Submission Sequence

**[HW]** To execute a user command submission (`IB1` list) on a given ringbuffer:
1. **[HW]** Emit `PKT7(CP_SET_MARKER, 1)` with payload `0x00000101` (disable GMU IFPC while the command stream is in flight).
2. **[HW]** **Page Table Switch**: If switching user address spaces (`TTBR0` / `ASID`):
   - Emit `CP_WAIT_FOR_ME` (`0x13`), `CP_WAIT_FOR_IDLE` (`0x26`), `CP_EVENT_WRITE(PC_CCU_FLUSH_DEPTH_TS)`, `CP_EVENT_WRITE(PC_CCU_FLUSH_COLOR_TS)`, `CP_EVENT_WRITE(CACHE_INVALIDATE)`, `CP_SET_PROTECTED_MODE(0)`, `PKT4(CP_CHICKEN_DBG, 1)` = `0x00028805` (or set `stabilize_rp`), `CP_SMMU_TABLE_UPDATE` (`0x53`, 4 dwords: `ttbr0_lo`, `ttbr0_hi | (asid << 16)`, `contextidr`, `0`), `CP_MEM_WRITE` saving `ttbr0` and `contextidr` to `memptrs`, `CP_WAIT_MEM_WRITES`, `CP_WAIT_FOR_IDLE`, `CP_WAIT_FOR_ME`, `PKT4(CP_CHICKEN_DBG, 1)` = `0x00028801`, and `CP_SET_PROTECTED_MODE(1)`.
3. **[HW]** Emit `CP_EVENT_WRITE(PC_CCU_INVALIDATE_DEPTH)` (`0x18`) and `CP_EVENT_WRITE(PC_CCU_INVALIDATE_COLOR)` (`0x19`).
4. **[HW]** Emit `CP_SET_PSEUDO_REG` (`0x56`, 3 dwords: `REG_ID = 1` (`NON_SECURE_SAVE_ADDR`), `preempt_bo_iova_lo`, `preempt_bo_iova_hi`).
5. **[HW]** Emit `CP_REG_TO_MEM` reading `RBBM_PERFCTR_CP(0)` (`0x0370`, 64-bit) and `CP_ALWAYS_ON_COUNTER` (`0x0980`, 64-bit) into `memptrs->stats[idx].cpcycles_start` and `alwayson_start`.
6. **[HW]** Emit `CP_SET_MARKER(IB1LIST_START)` (`0x0d`), followed by `CP_INDIRECT_BUFFER` (`0x3f`) for each user `IB1` command stream, followed by `CP_SET_MARKER(IB1LIST_END)` (`0x0e`).
7. **[HW]** Emit `CP_SET_AMBLE` (`0x55`) clearing any user postamble (`DWORDS = 0, TYPE = 2`).
8. **[HW]** Emit `CP_REG_TO_MEM` reading ending `RBBM_PERFCTR_CP(0)` and `CP_ALWAYS_ON_COUNTER` into `memptrs->stats[idx].cpcycles_end` and `alwayson_end`.
9. **[HW]** Emit `CP_WAIT_FOR_ME`, `CP_EVENT_WRITE(CCU_RESOLVE)` (`0x1e`), `CP_EVENT_WRITE(PC_CCU_FLUSH_DEPTH_TS)` (`0x1c`), `CP_EVENT_WRITE(PC_CCU_FLUSH_COLOR_TS)` (`0x1d`), and `CP_EVENT_WRITE(CACHE_FLUSH_TS)` (`0x04` with Bit 31 `IRQ` set, writing the 32-bit `seqno` to `memptrs->fence` and firing `RBBM_INT_0_STATUS.CACHE_FLUSH_TS` Bit 20).
10. **[HW]** Emit `PKT7(CP_SET_MARKER, 1)` with payload `0x00000100` (re-enable GMU IFPC) and `CP_CONTEXT_SWITCH_YIELD` (`0x6b`, 4 dwords: `0`, `0`, `1`, `0`).
11. **[HW]** Update `CP_RB_WPTR` (`0x0807`) using the fenced write procedure (Section 3.2).

---

## 5. Ringbuffer Preemption & Context Switching

### 5.1 Preemption Memory Structures

**[HW]** To support hardware preemption across the 4 priority ringbuffers, each ringbuffer `i` requires two privileged buffers mapped in the kernel GPU VM:
1. **[HW]** **Preemption Record Buffer (`preempt_bo[i]`)**: Size **`4,292,608` bytes (`4192 KiB` = `0x418000` bytes)**, mapped with `WC | MAP_PRIV`. The first 64 bytes hold the hardware context header:

| Byte Offset | Size | Type | Field Name | Tag | Initial / Runtime Value |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `+0x00` | 4 | `uint32` | `magic` | **[HW]** | Must be **`0xae399d6e`**. |
| `+0x04` | 4 | `uint32` | `info` | **[HW]** | Initialize to `0x00000000`. |
| `+0x08` | 4 | `uint32` | `errno` | **[HW]** | Initialize to `0x00000000`. |
| `+0x0c` | 4 | `uint32` | `data` | **[HW]** | Initialize to `0x00000000`. |
| `+0x10` | 4 | `uint32` | `cntl` | **[HW]** | Initialize to **`0x0000020c`** (`CP_RB_CNTL` value). |
| `+0x14` | 4 | `uint32` | `rptr` | **[HW]** | Initialize to `0x00000000` (updated by CP on save). |
| `+0x18` | 4 | `uint32` | `wptr` | **[HW]** | Updated by host CPU before preemption trigger to the target ring's current `wptr`. |
| `+0x1c` | 4 | `uint32` | `_pad1c` | **[HW]** | Initialize to `0x00000000`. |
| `+0x20` | 8 | `uint64` | `rptr_addr` | **[HW]** | 64-bit IOVA of `shadow_iova + (i * 4)`. |
| `+0x28` | 8 | `uint64` | `rbase` | **[HW]** | 64-bit IOVA of ringbuffer `i`. |
| `+0x30` | 8 | `uint64` | `counter` | **[HW]** | 64-bit IOVA of performance counter save area (or `0` when unused). |
| `+0x38` | 8 | `uint64` | `bv_rptr_addr` | **[HW]** | 64-bit IOVA of `memptrs[i].bv_rptr` (`memptrs_iova + 0x008`). |

2. **[HW]** **Preemption SMMU Info Buffer (`preempt_smmu_bo[i]`)**: Size **`4096` bytes**, mapped with `WC | MAP_PRIV | GPU_READONLY`. The first 28 bytes hold the SMMU page-table descriptor:

| Byte Offset | Size | Type | Field Name | Tag | Initial / Runtime Value |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `+0x00` | 4 | `uint32` | `magic` | **[HW]** | Must be **`0x241350d5`**. |
| `+0x04` | 4 | `uint32` | `_pad4` | **[HW]** | Initialize to `0x00000000`. |
| `+0x08` | 8 | `uint64` | `ttbr0` | **[HW]** | Updated before preemption trigger to the target ring's active `ttbr0 | (asid << 48)` (from `memptrs->ttbr0`). |
| `+0x10` | 4 | `uint32` | `asid` | **[HW]** | Must be **`0xdecafbad`** (sentinel instructing CP to extract the 16-bit ASID from bits 63:48 of `ttbr0`). |
| `+0x14` | 4 | `uint32` | `context_idr` | **[HW]** | Updated before preemption trigger to `memptrs->context_idr` (or `0`). |
| `+0x18` | 4 | `uint32` | `context_bank` | **[HW]** | Initialize to `0x00000000` (Context Bank 0). |

### 5.2 Preemption Trigger Sequence

**[HW]** During hardware initialization, write `0x00000001` to `RB_CONTEXT_SWITCH_GMEM_SAVE_RESTORE_ENABLE` (`0x8870`). To preempt the current ringbuffer and switch to target ring `next`:
1. **[HW]** Update `preempt_bo[next]->wptr` with ring `next`'s latest `wptr`, and update `preempt_smmu_bo[next]->ttbr0` and `context_idr` from `memptrs[next]`. Execute a memory barrier (`dsb sy`).
2. **[HW]** Using fenced writes (fence mask `0x2`), program:
   - `CP_CONTEXT_SWITCH_SMMU_INFO_LO` (`0x08a1`) and `_HI` (`0x08a2`) = 64-bit IOVA of `preempt_smmu_bo[next]`.
   - `CP_CONTEXT_SWITCH_PRIV_NON_SECURE_RESTORE_ADDR_LO` (`0x08a3`) and `_HI` (`0x08a4`) = 64-bit IOVA of `preempt_bo[next]`.
   - `CP_CONTEXT_SWITCH_CNTL` (`0x08a0`) = **`0x00000341`** (`STOP` Bit 0 = `1`, `LEVEL` Bits 7:6 = `1`, `USES_GMEM` Bit 8 = `1`, `SKIP_SAVE_RESTORE` Bit 9 = `1`).
3. **[HW]** When the switch finishes, the hardware clears `CP_CONTEXT_SWITCH_CNTL.STOP` (Bit 0) to `0` and asserts `RBBM_INT_0_STATUS.CP_SW` (Bit 8).

---

## 6. Complete 3D Hardware Initialization Sequence

**[HW]** After the GMU boot and HFI handshake complete, the host driver initializes the Adreno 690 3D core while holding the GMU `OOB_GPU_SET` wake vote:

### Step 1: Clear GBIF Halt & Untrusted Secure Video Mode
1. **[HW]** Write `0x00000000` to `GBIF_HALT` (`0x3c45`) and `0x00000000` to `RBBM_GBIF_HALT` (`0x0016`).
2. **[HW]** Write `0x00000000` to `RBBM_SECVID_TSB_CNTL` (`0xf803`), `RBBM_SECVID_TSB_TRUSTED_BASE_LO` (`0xf800`), `RBBM_SECVID_TSB_TRUSTED_BASE_HI` (`0xf801`), and `RBBM_SECVID_TSB_TRUSTED_SIZE` (`0xf802`).

### Step 2: Enable 64-Bit Addressing Across All 12 Pipeline Blocks
**[HW]** Write `0x00000001` to each of the following 12 registers:
- `CP_ADDR_MODE_CNTL` (`0x0842`), `VSC_ADDR_MODE_CNTL` (`0x0c01`), `GRAS_ADDR_MODE_CNTL` (`0x8601`), `RB_ADDR_MODE_CNTL` (`0x8e05`), `PC_ADDR_MODE_CNTL` (`0x9e01`), `HLSQ_ADDR_MODE_CNTL` (`0xbe05`), `VFD_ADDR_MODE_CNTL` (`0xa601`), `VPC_ADDR_MODE_CNTL` (`0x9601`), `UCHE_ADDR_MODE_CNTL` (`0x0e01`), `SP_ADDR_MODE_CNTL` (`0xae01`), `TPL1_ADDR_MODE_CNTL` (`0xb601`), and `RBBM_SECVID_TSB_ADDR_MODE_CNTL` (`0xf810`).

### Step 3: Program Adreno 690 Hardware Clock Gating (`HWCG`)
1. **[HW]** Program GMU clock gating registers:
   - `GPU_GMU_AO_GMU_CGC_MODE_CNTL` (`0x23b09`) = `0x00020200`
   - `GPU_GMU_AO_GMU_CGC_DELAY_CNTL` (`0x23b0a`) = `0x00010111`
   - `GPU_GMU_AO_GMU_CGC_HYST_CNTL` (`0x23b0b`) = `0x00005555`
2. **[HW]** Read `GPU_GMU_GX_SPTPRAC_CLOCK_CONTROL` (`0x1a880`), clear Bit 0 (`0xfffffffe`), and write it back.
3. **[HW]** Write the 50 Adreno 690 RBBM clock gating register values:

| Dword Offset | Register Name | Value | Dword Offset | Register Name | Value |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `0x00b0` | `RBBM_CLOCK_CNTL_SP0` | `0x02222222` | `0x00c2` | `RBBM_CLOCK_DELAY_RAC` | `0x00000011` |
| `0x00b4` | `RBBM_CLOCK_CNTL2_SP0` | `0x02222220` | `0x00c3` | `RBBM_CLOCK_HYST_RAC` | `0x00445044` |
| `0x00b8` | `RBBM_CLOCK_DELAY_SP0` | `0x00000080` | `0x00c4` | `RBBM_CLOCK_CNTL_TSE_RAS_RBBM` | `0x04222222` |
| `0x00bc` | `RBBM_CLOCK_HYST_SP0` | `0x0000f3cf` | `0x00c5` | `RBBM_CLOCK_MODE_VFD` | `0x00002222` |
| `0x00b0` | `RBBM_CLOCK_CNTL_TP0` | `0x22222222` | `0x00c6` | `RBBM_CLOCK_MODE_GPC` | `0x02222222` |
| `0x00b4` | `RBBM_CLOCK_CNTL2_TP0` | `0x22222222` | `0x00c7` | `RBBM_CLOCK_DELAY_HLSQ_2` | `0x00000002` |
| `0x00b8` | `RBBM_CLOCK_CNTL3_TP0` | `0x22222222` | `0x00c8` | `RBBM_CLOCK_MODE_HLSQ` | `0x00002222` |
| `0x00bc` | `RBBM_CLOCK_CNTL4_TP0` | `0x00022222` | `0x00c9` | `RBBM_CLOCK_DELAY_TSE_RAS_RBBM` | `0x00004000` |
| `0x00c0` | `RBBM_CLOCK_DELAY_TP0` | `0x11111111` | `0x00ca` | `RBBM_CLOCK_DELAY_VFD` | `0x00002222` |
| `0x00c4` | `RBBM_CLOCK_DELAY2_TP0` | `0x11111111` | `0x00cb` | `RBBM_CLOCK_DELAY_GPC` | `0x00000200` |
| `0x00c8` | `RBBM_CLOCK_DELAY3_TP0` | `0x11111111` | `0x00cc` | `RBBM_CLOCK_DELAY_HLSQ` | `0x00000000` |
| `0x00cc` | `RBBM_CLOCK_DELAY4_TP0` | `0x00011111` | `0x00cd` | `RBBM_CLOCK_HYST_TSE_RAS_RBBM` | `0x00000000` |
| `0x00d0` | `RBBM_CLOCK_HYST_TP0` | `0x77777777` | `0x00ce` | `RBBM_CLOCK_HYST_VFD` | `0x00000000` |
| `0x00d4` | `RBBM_CLOCK_HYST2_TP0` | `0x77777777` | `0x00cf` | `RBBM_CLOCK_HYST_GPC` | `0x04104004` |
| `0x00d8` | `RBBM_CLOCK_HYST3_TP0` | `0x77777777` | `0x00d0` | `RBBM_CLOCK_HYST_HLSQ` | `0x00000000` |
| `0x00dc` | `RBBM_CLOCK_HYST4_TP0` | `0x00077777` | `0x00d1` | `RBBM_CLOCK_CNTL_TEX_FCHE` | `0x00000222` |
| `0x00f0` | `RBBM_CLOCK_CNTL_RB0` | `0x22222222` | `0x00d2` | `RBBM_CLOCK_DELAY_TEX_FCHE` | `0x00000111` |
| `0x00f4` | `RBBM_CLOCK_CNTL2_RB0` | `0x01002222` | `0x00d3` | `RBBM_CLOCK_HYST_TEX_FCHE` | `0x00000777` |
| `0x00f8` | `RBBM_CLOCK_CNTL_CCU0` | `0x00002220` | `0x00d4` | `RBBM_CLOCK_CNTL_UCHE` | `0x22222222` |
| `0x0100` | `RBBM_CLOCK_HYST_RB_CCU0` | `0x00040f00` | `0x00d5` | `RBBM_CLOCK_HYST_UCHE` | `0x00000004` |
| `0x00c0` | `RBBM_CLOCK_CNTL_RAC` | `0x25222222` | `0x00d6` | `RBBM_CLOCK_DELAY_UCHE` | `0x00000002` |
| `0x00c1` | `RBBM_CLOCK_CNTL2_RAC` | `0x00005555` | `0x00d7` | `RBBM_CLOCK_CNTL_GMU_GX` | `0x00000222` |
| `0x00d8` | `RBBM_CLOCK_DELAY_GMU_GX` | `0x00000111` | `0x00d9` | `RBBM_CLOCK_HYST_GMU_GX` | `0x00000555` |
| `0x0118` | `RBBM_CLOCK_CNTL_FCHE` | `0x00000222` | `0x0119` | `RBBM_CLOCK_DELAY_FCHE` | `0x00000111` |
| `0x011a` | `RBBM_CLOCK_HYST_FCHE` | `0x00000777` | `0x00ae` | `RBBM_CLOCK_CNTL` | `0x8aa8aa82` |

*(Note: After programming the table entries above, set Bit 0 of `GPU_GMU_GX_SPTPRAC_CLOCK_CONTROL` (`0x1a880`) back to `1`, and write `0x8aa8aa82` to `RBBM_CLOCK_CNTL` (`0x00ae`).)*

### Step 4: Configure UCHE, ROQ, UBWC 4.0, APRIV, and Bicubic Weights
**[HW]** Write the following configuration registers:

| Dword Offset | Register Name | Value | Tag | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x050b` | `RBBM_PERFCTR_GPU_BUSY_MASKED` | `0xffffffff` | **[HW]** | Unmask all blocks for the GMU busy counter. |
| `0x0e0c`..`0x0e0d` | `UCHE_WRITE_RANGE_MAX_LO` / `_HI` | `0xfffffffc0` / `0x0000001f` | **[HW]** | 64-bit value `0x0000001fffffffffffc0`. |
| `0x0e09`..`0x0e0a` | `UCHE_TRAP_BASE_LO` / `_HI` | `0xfffff000` / `0x0000001f` | **[HW]** | 64-bit value `0x0000001fffffffff000`. |
| `0x0e07`..`0x0e08` | `UCHE_WRITE_THRU_BASE_LO` / `_HI` | `0xfffff000` / `0x0000001f` | **[HW]** | 64-bit value `0x0000001fffffffff000` (note: do **not** write `UCHE_GMEM_RANGE_MIN/MAX` on A690). |
| `0x0e18` | `UCHE_FILTER_CNTL` | `0x00000804` | **[HW]** | UCHE filter control. |
| `0x0e17` | `UCHE_CACHE_WAYS` | `0x00000004` | **[HW]** | UCHE cache ways. |
| `0x08c2` | `CP_ROQ_THRESHOLDS_2` | `0x02000140` | **[HW]** | ROQ threshold 2 (`0x02000140` on A660/A690 family). |
| `0x08c1` | `CP_ROQ_THRESHOLDS_1` | `0x8040362c` | **[HW]** | ROQ threshold 1. |
| `0x0b34` | `CP_LPAC_PROG_FIFO_SIZE` | `0x00000020` | **[HW]** | LPAC FIFO size. |
| `0x08c3` | `CP_MEM_POOL_SIZE` | `0x00000080` | **[HW]** | CP memory pool size (`128`). |
| `0x9e00` | `PC_DBG_ECO_CNTL` | `0x00800200` | **[HW]** | Primitive FIFO threshold for A690. |
| `0x098d` | `CP_AHB_CNTL` | `0x00000001` | **[HW]** | Do not halt AHB on error. |
| `0x0500` | `RBBM_PERFCTR_CNTL` | `0x00000001` | **[HW]** | Enable performance counters. |
| `0x08d0` | `CP_PERFCTR_CP_SEL(0)` | `0x00000000` | **[HW]** | Select `PERF_CP_ALWAYS_COUNT` (`0`) on counter 0. |
| `0x8e08` | `RB_NC_MODE_CNTL` | `0x00000816` | **[HW]** | UBWC 4.0 (`highest_bank_bit = 16 -> hbb = 3`, `rgb565_predicator = 1`, `amsbc = 1`, `macrotile_mode = 1`). |
| `0xb604` | `TPL1_NC_MODE_CNTL` | `0x00000006` | **[HW]** | UBWC 4.0 (`hbb = 3`, `macrotile_mode = 1`). |
| `0xae02` | `SP_NC_MODE_CNTL` | `0x00000026` | **[HW]** | UBWC 4.0 (`uavflagprd_inv = 2`, `hbb = 3`, `macrotile_mode = 1`). |
| `0x0e00` | `UCHE_MODE_CNTL` | `0x00600000` | **[HW]** | UBWC 4.0 (`hbb_lo = 3` at bits 22:21). |
| `0x0534` | `RBBM_NC_MODE_CNTL` | `0x00000001` | **[HW]** | `MACROTILE_MODE = 1` (8-channel). |
| `0x001f` | `RBBM_INTERFACE_HANG_INT_CNTL` | `0x404fffff` | **[HW]** | Enable hang detection (`Bit 30`) with timeout `0x4fffff` on A690. |
| `0x0e19` | `UCHE_CLIENT_PF` | `0x00000081` | **[HW]** | `(1 << 7) \| 0x1` (`PERFSEL = 1`). |
| `0xb608`..`0xb60c` | `TPL1_BICUBIC_WEIGHTS_TABLE_0..4` | `0x00000000`, `0x3fe05ff4`, `0x3fa0ebee`, `0x3f5193ed`, `0x3f0243f0` | **[HW]** | Default bicubic texture filter weights. |
| `0x0841` | `CP_CHICKEN_DBG` | `0x00028801` | **[HW]** | Adreno 690-specific CP chicken bits. |
| `0x0011` | `RBBM_GBIF_CLIENT_QOS_CNTL` | `0x00000000` | **[HW]** | Client QoS control (`0x0` on A660/A690 family). |
| `0x0e1c` | `UCHE_CMDQ_CONFIG` | `0x00000090` | **[HW]** | Set dual-queue command mode (`0x90` on A690). |
| `0x0844` | `CP_APRIV_CNTL` | `0x0000006e` | **[HW]** | Enable privileged protection for crashdump, RPTR writeback, and RB fetch (`Bits 6, 5, 3, 2, 1`). |
| `0x0038` | `RBBM_INT_0_MASK` | `0x03d0e342` | **[HW]** | Enable RBBM interrupts (`CP_AHB_ERROR`, `ATB_ASYNCFIFO_OVERFLOW`, `GPC_ERR`, `CP_SW`, `CP_HW_ERROR`, `CCU_CLEAN_*`, `CACHE_FLUSH_TS`, `UCHE_OOB_ACCESS`, `UCHE_TRAP_INTR`, `DEBBUS_INTR_0/1`). |

### Step 5: Program CP Register Protection (`CP_PROTECT`)
**[HW]** Write `0x0000000b` (`ACCESS_PROT_EN | ACCESS_FAULT_ON_VIOL_EN | LAST_SPAN_INF_RANGE`) to `CP_PROTECT_CNTL` (`0x084f`), then program the 35 Adreno 690 protection spans into `CP_PROTECT(0..33)` (`0x0850..0x0871`) and `CP_PROTECT(47)` (`0x087f`). Each entry encodes `(NORDWR << 31) | ((dword_len & 0x3fff) << 18) | (start_dword_reg & 0x3ffff)`:

| Index (`Reg`) | Value | Span Covered | Index (`Reg`) | Value | Span Covered |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `0` (`0x0850`) | `0x60000001` | `NORDWR(0x00001, 0x1800)` | `18` (`0x0862`) | `0x80048e50` | `NORDWR(0x08e50, 0x0001)` |
| `1` (`0x0851`) | `0x000c0050` | `RDONLY(0x00050, 0x0003)` | `19` (`0x0863`) | `0x81cc9624` | `NORDWR(0x09624, 0x0073)` |
| `2` (`0x0852`) | `0x885c04fc` | `NORDWR(0x004fc, 0x0217)` | `20` (`0x0864`) | `0x80009e70` | `NORDWR(0x09e70, 0x0000)` |
| `3` (`0x0853`) | `0x00640806` | `RDONLY(0x00806, 0x0019)` | `21` (`0x0865`) | `0x80209e78` | `NORDWR(0x09e78, 0x0008)` |
| `4` (`0x0854`) | `0x80bc0820` | `NORDWR(0x00820, 0x002f)` | `22` (`0x0866`) | `0x86f89f00` | `NORDWR(0x09f00, 0x01be)` |
| `5` (`0x0855`) | `0x84000882` | `NORDWR(0x00882, 0x0100)` | `23` (`0x0867`) | `0x8004a630` | `NORDWR(0x0a630, 0x0001)` |
| `6` (`0x0856`) | `0x003c0980` | `RDONLY(0x00980, 0x000f)` | `24` (`0x0868`) | `0x8000ae50` | `NORDWR(0x0ae50, 0x0000)` |
| `7` (`0x0857`) | `0x00200b31` | `RDONLY(0x00b31, 0x0008)` | `25` (`0x0869`) | `0x8168ae80` | `NORDWR(0x0ae80, 0x005a)` |
| `8` (`0x0858`) | `0x81fc0b00` | `NORDWR(0x00b00, 0x007f)` | `26` (`0x086a`) | `0x80c0af00` | `NORDWR(0x0af00, 0x0030)` |
| `9` (`0x0859`) | `0x80240c01` | `NORDWR(0x00c01, 0x0009)` | `27` (`0x086b`) | `0x8000b604` | `NORDWR(0x0b604, 0x0000)` |
| `10` (`0x085a`) | `0x00080ce0` | `RDONLY(0x00ce0, 0x0002)` | `28` (`0x086c`) | `0x800cb608` | `NORDWR(0x0b608, 0x0003)` |
| `11` (`0x085b`) | `0x801c0df8` | `NORDWR(0x00df8, 0x0007)` | `29` (`0x086d`) | `0x80fcb620` | `NORDWR(0x0b620, 0x003f)` |
| `12` (`0x085c`) | `0x807c0e01` | `NORDWR(0x00e01, 0x001f)` | `30` (`0x086e`) | `0xbfffbe20` | `NORDWR(0x0be20, 0x0fff)` |
| `13` (`0x085d`) | `0xbfff3c00` | `NORDWR(0x03c00, 0x0fff)` | `31` (`0x086f`) | `0xbffff000` | `NORDWR(0x0f000, 0x0fff)` |
| `14` (`0x085e`) | `0x81cc8600` | `NORDWR(0x08600, 0x0073)` | `32` (`0x0870`) | `0x0030fc00` | `RDONLY(0x0fc00, 0x000c)` |
| `15` (`0x085f`) | `0x80a08e00` | `NORDWR(0x08e00, 0x0028)` | `33` (`0x0871`) | `0x80010000` | `NORDWR(0x10000, 0x0000)` |
| `16` (`0x0860`) | `0x80088e08` | `NORDWR(0x08e08, 0x0002)` | `47` (`0x087f`) | `0x80011c00` | `NORDWR(0x11c00, inf)` |
| `17` (`0x0861`) | `0x80708e10` | `NORDWR(0x08e10, 0x001c)` | | | |

### Step 6: Activate LLCC System Cache Slices (`gpu` and `gpuhtw`)
**[HW]** Activate SC8280XP LLCC system cache slice `11` (`LLCC_GPU`) and slice `12` (`LLCC_GPUHTW`). On Adreno 690 (which pairs with an `arm,mmu-500` SMMU):
- Write `(gpu_scid) | (gpu_scid << 5) | (gpu_scid << 10) | (gpu_scid << 15) | (gpu_scid << 20)` to `GBIF_SCACHE_CNTL1` (`0x3c02`).
- Write `(gpu_scid << 10) | (1 << 8)` to `GBIF_SCACHE_CNTL0` (`0x3c01`).

### Step 7: Configure SMMU Aperture, Start SQE, and Run `CP_ME_INIT`
1. **[HW]** Invoke the TrustZone SCM call `QCOM_SCM_MP_CP_SMMU_APERTURE_ID` (service `0x0c`, command `0x1b`, SMC function ID `0x42000c1b`) with argument `0x00800000` (`CP_APERTURE_REG | (0 << 16)`) to bind Context Bank 0 to `CP_SMMU_TABLE_UPDATE`.
2. **[HW]** Program `CP_SQE_INSTR_BASE_LO`/`_HI` (`0x0830`/`0x0831`), Ring 0 `CP_RB_BASE_LO`/`_HI` (`0x0800`/`0x0801`), `CP_RB_CNTL` (`0x0802` = `0x0000020c`), `CP_RB_RPTR_ADDR_LO`/`_HI` (`0x0804`/`0x0805`), and set `CP_SQE_CNTL` (`0x0808`) = `0x00000001`.
3. **[HW]** Emit `PKT7(CP_ME_INIT, 8)` into Ring 0, update `CP_RB_WPTR`, and wait for Ring 0 to drain.

---

## 7. Zap Shader Authentication (`PAS ID 13`) & Non-Secure Mode Transition

**[HW]** Following cold reset, the Adreno 690 3D engine starts in hardware **Secure Mode**, blocking non-secure GPU memory accesses until the signed **Zap Shader** (`qcom/sc8280xp/LENOVO/21BX/qcdxkmsuc8280.mbn`, catalog fallback `a690_zap.mdt`) is authenticated by the TrustZone Peripheral Authentication Service (PAS) and switches the GPU to non-secure mode:

1. **[HW]** **PAS Image Initialization (`PAS ID = 13` / `0x0d`)**: Parse the ELF32 header and hash table segment from `qcdxkmsuc8280.mbn`, copy them into a physically contiguous 4 KiB-aligned metadata buffer, and invoke TrustZone SCM `PAS_INIT_IMAGE` (`owner = 2`, `svc = 0x02`, `cmd = 0x01`, SMC `0x42000202`) with `pas_id = 13` and `metadata_phys`.
2. **[HW]** **Carveout Segment Load**: Copy all loadable `PT_LOAD` ELF segments into the reserved `pil_gpu_mem` physical carveout at **`0x8bf00000`** (`size >= 0x2000` bytes).
3. **[HW]** **PAS Memory Setup**: Invoke TrustZone SCM `PAS_MEM_SETUP` (`svc = 0x02`, `cmd = 0x02`, SMC `0x42000203`) with `pas_id = 13`, `phys_addr = 0x8bf00000`, and the relocated region byte size.
4. **[HW]** **PAS Authenticate & Reset**: Invoke TrustZone SCM `PAS_AUTH_AND_RESET` (`svc = 0x02`, `cmd = 0x05`, SMC `0x42000201`) with `pas_id = 13`.
5. **[HW]** **In-Band Transition to Non-Secure Mode**:
   - Emit `PKT7(CP_SET_SECURE_MODE, 1)` into Ring 0:
     - Dword 0: **`0x70668001`** (`TYPE = 7`, `OPCODE = 0x66`, `PARITY = 1`, `COUNT = 1`)
     - Dword 1: **`0x00000000`** (exit secure mode)
   - Emit a `CACHE_FLUSH_TS` fence packet, perform a fenced write to `CP_RB_WPTR`, and wait for Ring 0 completion.
   - *(Note: Only on unpacked/development fused boards where TrustZone reports PAS unsupported may software write `0x00000000` directly to `RBBM_SECVID_TRUST_CNTL` (`0xf400`).)*
6. **[HW]** Release the GMU `OOB_GPU_SET` wake vote by writing `0x80000000` (Bit 31) to `GMU_HOST2GMU_INTR_SET` (`0x23194`).
