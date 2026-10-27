# Adreno 690 GMU, Host Firmware Interface (HFI), and Devfreq Specification

## 1. GMU Subsystem Overview and Register Windows

**[HW]** The Graphics Management Unit (GMU) on the Adreno 690 (`chip-id = 0x06090000`, `ADRENO_6XX_GEN4` / A660 family) is an embedded ARM Cortex-M3 microcontroller that manages GPU power rails (`GX` and `CX`), clock switching, RPMh resource voting, inter-frame power collapse (`IFPC`), and dynamic voltage and frequency scaling (DVFS).

**[HW]** The GMU subsystem spans four physical MMIO regions on SC8280XP:

| Region Name | Physical Base | Size | Description |
|---|---|---|---|
| **GMU Core MMIO** | `0x03d6a000` | `0x00034000` (`0x31000` active) | GMU Cortex-M3 ITCM/DTCM, control, HFI, interrupt, and Always-On (`AO`) registers. Hardware dword offsets (`0x1a800`–`0x26801`) are specified relative to the GPU 3D base (`0x03d00000`); the byte offset relative to `0x03d6a000` is `(dword_offset * 4) - 0x6a000`. |
| **GMU RSCC** | `0x03de0000` (DT) / `0x03d8c000` (alias) | `0x00010000` | Resource State Coordinator Controller (RSCC) for GPU RPMh sleep/wake sequences. Register offsets are 32-bit dword indices (`byte_offset = dword_offset * 4`). Note: `0x03de0000` is ACPI `\_SB.GPU0` Resource 7; `0x03d8c000` is the alias inside the combined `0x03d60000` window (`0x03d6a000 + 0x22000`). |
| **GMU PDC** | `0x0b290000` (GPU0) / `0x0b2a0000` | `0x00010000` | Power Domain Controller (PDC) GPU register bank. Register offsets are 32-bit dword indices (`byte_offset = dword_offset * 4`). On A650/A660/A690 families, PDC TCS and sequence memory are pre-configured by the Always-On Processor (AOP), so only `PDC_GPU_SEQ_START_ADDR` and `PDC_GPU_ENABLE_PDC` are written by the host. |
| **GMU PDC SEQ** | `0x0b490000` / `0x0b4a0000` | `0x00010000` | PDC sequencer microcode window (unused by host software on A690 because `pdc_in_aop` is true for the A650/A660/A690 family). |

---

## 2. GMU Register Map

### 2.1 GMU Core Registers (`0x03d6a000`)

**[HW]** All GMU core registers are 32-bit little-endian words. Both the hardware dword offset (from `0x03d00000`) and the byte offset from `0x03d6a000` are listed below:

| Register Name | Dword Offset | Byte Offset (`0x03d6a000 +`) | Access | Description |
|---|---|---|---|---|
| `GPU_GMU_GX_SPTPRAC_CLOCK_CONTROL` | `0x1a880` | `0x00200` | R/W | Bit 0: SP clock enable (`0` = disable while programming HWCG, `1` = enable). |
| `GMU_GX_SPTPRAC_POWER_CONTROL` | `0x1a881` | `0x00204` | R/W | Legacy SPTPRAC power control (`0x778000` = power on, `0x778001` = power off). |
| `GMU_CM3_ITCM_START` | `0x1b400` | `0x03000` | W | Start of 16 KiB Cortex-M3 Instruction Tightly Coupled Memory (`0x1b400`–`0x1c3ff`, 4096 dwords). |
| `GMU_CM3_DTCM_START` | `0x1c400` | `0x07000` | R/W | Start of 16 KiB Cortex-M3 Data Tightly Coupled Memory (`0x1c400`–`0x1d3ff`, 4096 dwords). |
| `GMU_NMI_CONTROL_STATUS` | `0x1cbf0` | `0x08fc0` | R/W | NMI control/status inside DTCM. |
| `GMU_BOOT_SLUMBER_OPTION` | `0x1cbf8` | `0x08fe0` | R/W | Legacy OOB boot/slumber selector (`0` = boot, `1` = slumber). |
| `GMU_GX_VOTE_IDX` | `0x1cbf9` | `0x08fe4` | R/W | Legacy GX ARC primary vote index. |
| `GMU_MX_VOTE_IDX` | `0x1cbfa` | `0x08fe8` | R/W | Legacy MX ARC secondary vote index. |
| `GMU_DCVS_ACK_OPTION` | `0x1cbfc` | `0x08ff0` | R/W | Legacy DCVS ack mode (`0`). |
| `GMU_DCVS_PERF_SETTING` | `0x1cbfd` | `0x08ff4` | R/W | Legacy DCVS performance index (`(3 << 28) \| perf_index`). |
| `GMU_DCVS_BW_SETTING` | `0x1cbfe` | `0x08ff8` | R/W | Legacy DCVS bandwidth index (`0xff`). |
| `GMU_DCVS_RETURN` | `0x1cbff` | `0x08ffc` | R | Legacy DCVS return status (`0` = success). |
| `GMU_CORE_FW_VERSION` | `0x1d3f8` | `0x0afe0` | R | Loaded GMU firmware version in DTCM (`MAJOR` = Bits 31:28, `MINOR` = Bits 27:16, `STEP` = Bits 15:0; `DTCM + 0xff8` is dword `0x1c400 + 0x3fe = 0x1c7fe`, byte `+0x07ff8`). |
| `GMU_ICACHE_CONFIG` | `0x1f400` | `0x13000` | R/W | Write `0x1` to disable GMU instruction cache WB/RB buffer. |
| `GMU_DCACHE_CONFIG` | `0x1f401` | `0x13004` | R/W | Write `0x1` to disable GMU data cache WB/RB buffer. |
| `GMU_SYS_BUS_CONFIG` | `0x1f40f` | `0x1303c` | R/W | Write `0x1` to disable GMU system bus WB/RB buffer. |
| `GMU_CM3_SYSRESET` | `0x1f800` | `0x14000` | R/W | Bit 0: Cortex-M3 reset (`1` = assert reset / halt CPU, `0` = release reset / run). |
| `GMU_CM3_BOOT_CONFIG` | `0x1f801` | `0x14004` | R/W | Write `0x02` before releasing `GMU_CM3_SYSRESET`. |
| `GMU_CM3_FW_BUSY` | `0x1f81a` | `0x14068` | R | Firmware busy status. |
| `GMU_CM3_FW_INIT_RESULT` | `0x1f81c` | `0x14070` | R/W | Cleared to `0` before boot; polled after releasing `GMU_CM3_SYSRESET` until `(val & mask) == reset_val` (`0xbabeface` or `0x100`). |
| `GMU_CM3_CFG` | `0x1f82d` | `0x140b4` | R/W | Write `0x4052` to prevent snapshot NMI toggle from jumping to NMI handler instead of `__main`. |
| `GMU_CX_GMU_POWER_COUNTER_ENABLE` | `0x1f840` | `0x14100` | R/W | Write `1` to enable CX power counters, `0` before slumber. |
| `GMU_CX_GMU_POWER_COUNTER_SELECT_0` | `0x1f841` | `0x14104` | R/W | Bits 7:0: Counter 0 countable selector (`0x20` = Bit 5 busy cycles). |
| `GMU_CX_GMU_POWER_COUNTER_SELECT_1` | `0x1f842` | `0x14108` | R/W | Counter 1 countable selector. |
| `GMU_CX_GMU_POWER_COUNTER_XOCLK_0_L` | `0x1f844` | `0x14110` | R | Low 32 bits of 19.2 MHz XO busy cycle counter (used by devfreq). |
| `GMU_CX_GMU_POWER_COUNTER_XOCLK_0_H` | `0x1f845` | `0x14114` | R | High 32 bits of 19.2 MHz XO busy cycle counter. |
| `GMU_ALWAYS_ON_COUNTER_L` | `0x1f888` | `0x14220` | R | Low 32 bits of 19.2 MHz Always-On timestamp counter. |
| `GMU_ALWAYS_ON_COUNTER_H` | `0x1f889` | `0x14224` | R | High 32 bits of 19.2 MHz Always-On timestamp counter. |
| `GMU_PWR_COL_INTER_FRAME_CTRL` | `0x1f8c0` | `0x14300` | R/W | Bit 0: `IFPC_ENABLE`, Bit 1: `HM_POWER_COLLAPSE_ENABLE`, Bit 2: `SPTPRAC_POWER_CONTROL_ENABLE`, Bits 13:10: `NUM_PASS_SKIPS`, Bits 31:14: `MIN_PASS_LENGTH`. Base write: `0x09c40400`. |
| `GMU_PWR_COL_INTER_FRAME_HYST` | `0x1f8c1` | `0x14304` | R/W | IFPC hysteresis in XO cycles (`0x000a1680`: low 16 bits `0x1680` = 300 us main hysteresis, high 16 bits `0x000a` = 0.5 us short hysteresis). |
| `GMU_PWR_COL_SPTPRAC_HYST` | `0x1f8c2` | `0x14308` | R/W | SPTPRAC hysteresis (`0x000a1680`). |
| `GMU_GMU_PWR_COL_KEEPALIVE` | `0x1f8c3` | `0x1430c` | R/W | Bit 0: Set to `1` during 3D IRQ handling to prevent GMU power collapse while reading `RBBM_INT_0_STATUS`, then `0` when done. |
| `GMU_PWR_COL_PREEMPT_KEEPALIVE` | `0x1f8c4` | `0x14310` | R/W | Bit 0: Set to `1` while ringbuffer preemption is in flight, `0` upon completion. |
| `GMU_SPTPRAC_PWR_CLK_STATUS` | `0x1f8d0` | `0x14340` | R | Bit 0: `SPTPRAC_GDSC_POWERING_OFF`, Bit 1: `SPTPRAC_GDSC_POWERING_ON`, Bit 2: `SPTPRAC_GDSC_POWER_OFF`, Bit 3: `SPTPRAC_GDSC_POWER_ON`, Bit 4: `SP_CLOCK_OFF`, Bit 5: `GMU_UP_POWER_STATE`, Bit 6: `GX_HM_GDSC_POWER_OFF`, Bit 7: `GX_HM_CLK_OFF`. GX is on when Bits 7:6 are both `0`. |
| `GMU_RPMH_CTRL` | `0x1f8e8` | `0x143a0` | R/W | Bit 0: `RPMH_INTERFACE_ENABLE`, Bit 4: `LLC_VOTE_ENABLE`, Bit 8: `DDR_VOTE_ENABLE`, Bit 9: `MX_VOTE_ENABLE`, Bit 10: `CX_VOTE_ENABLE`, Bit 11: `GFX_VOTE_ENABLE` (enable mask `0x0f11`). |
| `GPU_GMU_CX_GMU_RPMH_POWER_STATE` | `0x1f8ec` | `0x143b0` | R | Current GMU idle/power state (`0` = ACTIVE, `2` = SPTP, `3` = IFPC, `0x0f` = SLUMBER). |
| `GPU_GMU_CX_GMU_CX_FAL_INTF` | `0x1f8f0` | `0x143c0` | R/W | Write `0x1` on A650/A660/A690 family to veto FAL10 during GMU boot. |
| `GPU_GMU_CX_GMU_CX_FALNEXT_INTF` | `0x1f8f1` | `0x143c4` | R/W | Write `0x1` on A650/A660/A690 family to veto FAL10 and assert Qactive during power-down. |
| `GPU_GMU_CX_GMU_PWR_COL_CP_MSG` | `0x1f900` | `0x14400` | R/W | GMU log buffer descriptor: `(log_iova & 0xfffff000) \| ((log_size / 4096 - 1) & 0xff)`. |
| `GPU_GMU_CX_GMU_PWR_COL_CP_RESP` | `0x1f901` | `0x14404` | R/W | GMU log write-pointer index (write `0` before releasing `GMU_CM3_SYSRESET`). |
| `GMU_HFI_CTRL_STATUS` | `0x1f980` | `0x14600` | R | Bit 0: HFI queues ready (`1` = ready). |
| `GMU_HFI_VERSION_INFO` | `0x1f981` | `0x14604` | R | HFI hardware version. |
| `GMU_HFI_SFR_ADDR` | `0x1f982` | `0x14608` | R/W | Massaged GMU `chipid`: `(chip_id & 0xffff0000) \| ((chip_id << 4) & 0xf000) \| ((chip_id << 8) & 0x0f00)`. For A690 (`0x06090000`), write `0x06090000` (or `0x06090100` if `patchid = 1`). |
| `GMU_HFI_QTBL_INFO` | `0x1f984` | `0x14610` | R/W | Write `0x1` when `GMU_HFI_QTBL_ADDR` is valid. |
| `GMU_HFI_QTBL_ADDR` | `0x1f985` | `0x14614` | R/W | 32-bit GMU IOVA of the HFI queue table buffer. |
| `GMU_HFI_CTRL_INIT` | `0x1f986` | `0x14618` | R/W | Write `0x1` to start HFI queues after GMU firmware boot. |
| `GMU_GMU2HOST_INTR_SET` | `0x1f990` | `0x14640` | W | Force GMU-to-host interrupt bits. |
| `GMU_GMU2HOST_INTR_CLR` | `0x1f991` | `0x14644` | W | Write-1-to-clear GMU-to-host interrupt status bits. |
| `GMU_GMU2HOST_INTR_INFO` | `0x1f992` | `0x14648` | R | GMU-to-host interrupt status: Bit 0 = `MSGQ`, Bit 1 = `DSGQ`, Bit 2 = `BLOCKED_MSG`, Bits 22:16 = `GMU_ERR`, Bit 23 = `CM3_FAULT`, Bits 31:24 = `OOB_ACK`. |
| `GMU_GMU2HOST_INTR_MASK` | `0x1f993` | `0x1464c` | R/W | Interrupt mask (`0` = unmasked, `1` = masked). Unmask Bit 23 (`~(1 << 23) = 0xff7fffff`) after HFI start. |
| `GMU_HOST2GMU_INTR_SET` | `0x1f994` | `0x14650` | W | Host-to-GMU interrupt trigger: Bit 0 = HFI queue doorbell (`0x1`), Bits 31:16 = OOB request/clear bits. |
| `GMU_HOST2GMU_INTR_CLR` | `0x1f995` | `0x14654` | W | Host-to-GMU interrupt clear. |
| `GMU_GENERAL_7` | `0x1f9cc` | `0x14730` | R/W | Write `0x1` to enable TCM (Tightly Coupled Memory) retention. |
| `GMU_AO_INTERRUPT_EN` | `0x23b03` | `0x24c0c` | R/W | Always-On interrupt enable. |
| `GMU_AO_HOST_INTERRUPT_CLR` | `0x23b04` | `0x24c10` | W | Write-1-to-clear `GMU_AO_HOST_INTERRUPT_STATUS`. |
| `GMU_AO_HOST_INTERRUPT_STATUS` | `0x23b05` | `0x24c14` | R | Bit 0 = `WDOG_BITE`, Bit 1 = `RSCC_COMP`, Bit 2 = `VDROOP`, Bit 3 = `FENCE_ERR`, Bit 4 = `DBD_WAKEUP`, Bit 5 = `HOST_AHB_BUS_ERROR`. |
| `GMU_AO_HOST_INTERRUPT_MASK` | `0x23b06` | `0x24c18` | R/W | Mask register (`0` = enabled). Unmask Bits 0, 3, 5 (`~0x29 = 0xffffffd6`). |
| `GMU_RSCC_CONTROL_REQ` | `0x23b07` | `0x24c1c` | R/W | Bit 0: Request PDC/RSCC sleep (`1`), Bit 1: Request RSCC wake (`2`). Clear to `0` after ack. |
| `GMU_RSCC_CONTROL_ACK` | `0x23b08` | `0x24c20` | R | Bit 1: RSCC wake ack (`2`). |
| `GPU_GMU_AO_GMU_CGC_MODE_CNTL` | `0x23b09` | `0x24c24` | R/W | GMU clock gating mode (`0x00020200` on A690 when HWCG enabled, `0` when disabled). |
| `GPU_GMU_AO_GMU_CGC_DELAY_CNTL` | `0x23b0a` | `0x24c28` | R/W | GMU CGC delay (`0x00010111` on A690). |
| `GPU_GMU_AO_GMU_CGC_HYST_CNTL` | `0x23b0b` | `0x24c2c` | R/W | GMU CGC hysteresis (`0x00005555` on A690). |
| `GPU_GMU_AO_GPU_CX_BUSY_STATUS` | `0x23b0c` | `0x24c30` | R | Bit 23 (`0x00800000`): `GPUBUSYIGNAHB` (`0` = GPU/GMU idle, `1` = busy). |
| `GPU_GMU_AO_GPU_CX_BUSY_STATUS2` | `0x23b0d` | `0x24c34` | R | Secondary CX busy status. |
| `GPU_GMU_AO_GPU_CX_BUSY_MASK` | `0x23b0e` | `0x24c38` | R/W | Write `0xff000000` during 3D hardware init. |
| `GMU_AO_AHB_FENCE_CTRL` | `0x23b10` | `0x24c40` | R/W | AHB fence mode (`0` = allow all writes; non-zero = fenced/dropped mode during power transitions). |
| `GMU_AHB_FENCE_RANGE_0` | `0x23b11` | `0x24c44` | R/W | Bit 31 = Enable, Bits 30:18 = Upper range (`0xa`), Bits 17:0 = Lower range (`0xa0`). Write `0x802800a0` on A6xx. |
| `GMU_AHB_FENCE_STATUS` | `0x23b13` | `0x24c4c` | R | Bit 0: `WRITEDROPPED0` (CP_RB_WPTR fence), Bit 1: `WRITEDROPPED1` (preempt register fence). |
| `GMU_AHB_FENCE_STATUS_CLR` | `0x23b14` | `0x24c50` | W | Write `0x7` to clear dropped-write sticky status bits. |
| `GMU_RBBM_INT_UNMASKED_STATUS` | `0x23b15` | `0x24c54` | R | Unmasked mirror of `RBBM_INT_0_STATUS`. |
| `GPU_CC_GX_GDSCR` | `0x24403` | `0x2700c` | R/W | Alias of `GPUCC + 0x100c` (`gpu_cc_gx_gdsc`). |

### 2.2 RSCC (`0x03de0000` / `0x03d8c000`) and PDC (`0x0b290000`) Registers

**[HW]** During one-time GMU initialization, software programs the RSCC and PDC to handle autonomous sleep/wakeup handshakes with RPMh. Because A690 belongs to the `ADRENO_6XX_GEN4` (A650/A660) family, the PDC sequence microcode and TCS registers are pre-programmed by the AOP (`pdc_in_aop = true`), so host software only programs RSCC and the two PDC enable registers:

| Block | Dword Offset | Byte Offset | Value Written on A690 | Description |
|---|---|---|---|---|
| **RSCC** | `0x0004` | `0x0010` | `0x01000000` (Bit 24) | `GPU_RSCC_RSC_STATUS0_DRV0`: Disable SDE clock gating. During `rpmh_stop`, poll Bit 16 (`0x00010000`) until `1`. |
| **RSCC** | `0x0008` | `0x0020` | `0x00004520` | `RSCC_PDC_SEQ_START_ADDR`: PDC sequence start address. |
| **RSCC** | `0x0009` | `0x0024` | `0x00004510` | `RSCC_PDC_MATCH_VALUE_LO`: PDC match value low. |
| **RSCC** | `0x000a` | `0x0028` | `0x00004514` | `RSCC_PDC_MATCH_VALUE_HI`: PDC match value high. |
| **RSCC** | `0x000b` | `0x002c` | `0x00000001` | `RSCC_PDC_SLAVE_ID_DRV0`: PDC slave ID (`1`). |
| **RSCC** | `0x000d` | `0x0034` | `0x00000000` | `RSCC_HIDDEN_TCS_CMD0_ADDR`. |
| **RSCC** | `0x000e` | `0x0038` | `0x00000000` | `RSCC_HIDDEN_TCS_CMD0_DATA`. |
| **RSCC** | `0x000f` | `0x003c` | `0x00000000` | `RSCC_HIDDEN_TCS_CMD0_ADDR + 2`. |
| **RSCC** | `0x0010` | `0x0040` | `0x00000000` | `RSCC_HIDDEN_TCS_CMD0_DATA + 2`. |
| **RSCC** | `0x0011` | `0x0044` | `0x00000000` | `RSCC_HIDDEN_TCS_CMD0_ADDR + 4`. |
| **RSCC** | `0x0012` | `0x0048` | `0x80000000` | `RSCC_HIDDEN_TCS_CMD0_DATA + 4`. |
| **RSCC** | `0x0100` | `0x0400` | `0x00000000` | `RSCC_OVERRIDE_START_ADDR`. |
| **RSCC** | `0x0101` | `0x0404` | *(Read-only)* | `RSCC_SEQ_BUSY_DRV0`: Poll until `0` during `rpmh_start`. |
| **RSCC** | `0x0180` | `0x0600` | `0xeaaae5a0` | `RSCC_SEQ_MEM_0_DRV0 + 0`: RSC sequencer uCode word 0 (A650/A660/A690 family). |
| **RSCC** | `0x0181` | `0x0604` | `0xe1a1ebab` | `RSCC_SEQ_MEM_0_DRV0 + 1`: RSC sequencer uCode word 1. |
| **RSCC** | `0x0182` | `0x0608` | `0xa2e0a581` | `RSCC_SEQ_MEM_0_DRV0 + 2`: RSC sequencer uCode word 2. |
| **RSCC** | `0x0183` | `0x060c` | `0xecac82e2` | `RSCC_SEQ_MEM_0_DRV0 + 3`: RSC sequencer uCode word 3. |
| **RSCC** | `0x0184` | `0x0610` | `0x0020edad` | `RSCC_SEQ_MEM_0_DRV0 + 4`: RSC sequencer uCode word 4. |
| **RSCC** | `0x0346` | `0x0d18` | *(Read-only)* | `RSCC_TCS0_DRV0_STATUS`: Poll Bit 0 (`1` = idle) before forcing GMU off. |
| **RSCC** | `0x03ee` | `0x0fb8` | *(Read-only)* | `RSCC_TCS1_DRV0_STATUS`: Poll Bit 0 (`1` = idle). |
| **RSCC** | `0x0496` | `0x1258` | *(Read-only)* | `RSCC_TCS2_DRV0_STATUS`: Poll Bit 0 (`1` = idle). |
| **RSCC** | `0x053e` | `0x14f8` | *(Read-only)* | `RSCC_TCS3_DRV0_STATUS`: Poll Bit 0 (`1` = idle). |
| **PDC** | `0x0007` | `0x001c` | `0x00000000` | `PDC_GPU_SEQ_START_ADDR`: Sequence start address (`0`). |
| **PDC** | `0x0000` | `0x0000` | `0x80000001` | `PDC_GPU_ENABLE_PDC`: Enable GPU PDC (`0x80000001`), followed by a write memory barrier (`wmb`). |

---

## 3. GMU Virtual Memory Map and Firmware Loading

### 3.1 GMU SMMU Context Bank 1 Allocations

**[HW]** The GMU Cortex-M3 accesses system RAM through GPU SMMUv2 Context Bank 1 (`0x03da9000`, Stream ID `0x01` / `0x05`) using a 32-bit IOVA space (`0x00000000`–`0x7fffffff`). All GMU shared buffers are allocated with Write-Combining (`WC`) CPU mapping attributes. Fixed-address buffers (`icache`, `dummy`, `debug`) additionally set the privileged SMMU mapping attribute (`IOMMU_PRIV`).

**[HW]** On the Adreno 690 (`A660` family), the following five buffers must be allocated and pinned in the GMU IOVA space before loading firmware:

| Buffer Name | Size (Bytes) | Required GMU IOVA | Privileged (`IOMMU_PRIV`) | Purpose |
|---|---|---|---|---|
| **`icache`** | `0x00ffc000` (16 MiB - 16 KiB) | Fixed at `0x00004000` (`0x00004000`–`0x00ffffff`) | Yes | Extended GMU instruction cache region for firmware blocks outside 16 KiB ITCM. |
| **`dummy`** | `0x00004000` (16 KiB) | Fixed at `0x60000000` (`0x60000000`–`0x60003fff`) | Yes | Preallocated GMU firmware scratch/dummy region at base of uncached window. |
| **`debug`** | `0x00007000` (28 KiB, `7 * 4 KiB`) | Fixed at `0x60400000` (`0x60400000`–`0x60406fff`) | Yes | Preallocated A660/A690 GMU firmware data/debug region. |
| **`log`** | `0x00004000` (16 KiB) | Dynamic in `0x60001000`–`0x7fffffff` | No | GMU firmware log ringbuffer, passed in `GPU_GMU_CX_GMU_PWR_COL_CP_MSG`. |
| **`hfi`** | `0x00004000` (16 KiB) | Dynamic in `0x60001000`–`0x7fffffff` | No | Host Firmware Interface queue table and three 4 KiB circular queues, passed in `GMU_HFI_QTBL_ADDR`. |

### 3.2 GMU Firmware Binary Format and Loading (`qcom/a660_gmu.bin`)

**[HW]** The Adreno 690 uses the `qcom/a660_gmu.bin` firmware image (sometimes symlinked or referred to as `qcom/a690_gmu.bin`). Unlike legacy A630 GMU firmware (which is a flat binary), A650/A660/A690 GMU firmware is a sequence of contiguous variable-length blocks.

**[HW]** Each block in `qcom/a660_gmu.bin` begins with a 16-byte little-endian header followed immediately by `size` bytes of payload data:

| Byte Offset | Type | Field Name | Description |
|---|---|---|---|
| `0x00` | `uint32` | `addr` | Target GMU address (ITCM, DTCM, or SMMU IOVA). |
| `0x04` | `uint32` | `size` | Payload size in bytes (always a multiple of 4). Next block is at `current_block + 16 + size`. |
| `0x08` | `uint32` | `type` | Block type metadata. |
| `0x0c` | `uint32` | `value` | Block value metadata. |
| `0x10` | `uint32[]` | `data` | `size` bytes of firmware payload. |

**[HW]** **GMU Firmware Loading Procedure** (performed on cold boot):
1. Set `itcm_base = 0x00000000` (size `16 KiB` = `0x4000`) and `dtcm_base = 0x10004000` (size `16 KiB` = `0x4000` for A650/A660/A690 family).
2. Iterate through all blocks in `qcom/a660_gmu.bin` from byte 0 to `fw_size`:
   - If `blk.size == 0`, skip to the next block (`offset += 16`).
   - If `blk.addr` falls within `[0x00000000, 0x00004000)`: copy `blk.size` bytes from `blk.data` to GMU MMIO starting at dword `GMU_CM3_ITCM_START + ((blk.addr - 0x00000000) >> 2)` (byte offset `0x03000 + blk.addr`), followed by a write memory barrier.
   - Else if `blk.addr` falls within `[0x10004000, 0x10008000)`: copy `blk.size` bytes from `blk.data` to GMU MMIO starting at dword `GMU_CM3_DTCM_START + ((blk.addr - 0x10004000) >> 2)` (byte offset `0x07000 + (blk.addr - 0x10004000)`), followed by a write memory barrier.
   - Else if `blk.addr` falls within the `icache` (`[0x00004000, 0x01000000)`), `dummy` (`[0x60000000, 0x60004000)`), or `debug` (`[0x60400000, 0x60407000)`) buffer IOVA ranges: copy `blk.size` bytes into the corresponding buffer at `virt_base + (blk.addr - bo.iova)`.
3. After loading all blocks, `GMU_CORE_FW_VERSION` (dword `0x1d3f8`, byte `0x0afe0`) contains the loaded firmware version (`MAJOR` in bits 31:28, `MINOR` in bits 27:16, `STEP` in bits 15:0).

---

## 4. GMU Cold Boot, Power Config, and Shutdown Sequences

### 4.1 GMU Cold Boot Sequence (`a6xx_gmu_fw_start`)

**[HW]** On A690, warm boot is not used because `icache` and `dcache` regions must be restored; every resume executes the `GMU_COLD_BOOT` (`1`) sequence:
1. Enable GMU clocks (`core_clk` = `200 MHz`, `hub_clk` = `150 MHz`).
2. Clear and unmask GMU AO interrupts: write `0xffffffff` to `GMU_AO_HOST_INTERRUPT_CLR` (`0x23b04`) and `~0x29` (`0xffffffd6`) to `GMU_AO_HOST_INTERRUPT_MASK` (`0x23b06`).
3. Vote FAL10 veto: write `0x1` to `GPU_GMU_CX_GMU_CX_FALNEXT_INTF` (`0x1f8f1`) and `0x1` to `GPU_GMU_CX_GMU_CX_FAL_INTF` (`0x1f8f0`).
4. Enable TCM retention: write `0x1` to `GMU_GENERAL_7` (`0x1f9cc`).
5. If waking from a previous PDC sleep (`GMU_STATUS_PDC_SLEEP` set):
   - Write `0x2` (Bit 1) to `GMU_RSCC_CONTROL_REQ` (`0x23b07`).
   - Poll `GMU_RSCC_CONTROL_ACK` (`0x23b08`) until Bit 1 (`0x2`) is set (timeout 10 ms).
   - Poll RSCC `RSCC_SEQ_BUSY_DRV0` (`0x0101`) until `0` (timeout 10 ms).
   - Write `0x0` to `GMU_RSCC_CONTROL_REQ` (`0x23b07`).
6. Load the GMU firmware blocks into ITCM, DTCM, and GMU BOs as specified in Section 3.2.
7. Write `0x0` to `GMU_CM3_FW_INIT_RESULT` (`0x1f81c`) and `0x02` to `GMU_CM3_BOOT_CONFIG` (`0x1f801`).
8. Write the 32-bit IOVA of the `hfi` buffer to `GMU_HFI_QTBL_ADDR` (`0x1f985`) and `0x1` to `GMU_HFI_QTBL_INFO` (`0x1f984`).
9. Configure AHB fence range 0: write `0x802800a0` (`(1 << 31) | (0xa << 18) | 0xa0`) to `GMU_AHB_FENCE_RANGE_0` (`0x23b11`).
10. Write `0x4052` to `GMU_CM3_CFG` (`0x1f82d`).
11. Write the GMU `chipid` (`0x06090000`) to `GMU_HFI_SFR_ADDR` (`0x1f982`) and `(log.iova | (log.size / 4096 - 1))` to `GPU_GMU_CX_GMU_PWR_COL_CP_MSG` (`0x1f900`).
12. Program CX GBIF QSB sideband registers in the 3D GPU MMIO (`0x03d00000`):
    - `GBIF_QSB_SIDE0` (dword `0x3c03`) = `0x00071620`
    - `GBIF_QSB_SIDE1` (dword `0x3c04`) = `0x00071620`
    - `GBIF_QSB_SIDE2` (dword `0x3c05`) = `0x00071620`
    - `GBIF_QSB_SIDE3` (dword `0x3c06`) = `0x00071620`
13. Configure GMU power and caches:
    - Write `0x1` to `GMU_SYS_BUS_CONFIG` (`0x1f40f`), `GMU_ICACHE_CONFIG` (`0x1f400`), and `GMU_DCACHE_CONFIG` (`0x1f401`).
    - Write `0x09c40400` to `GMU_PWR_COL_INTER_FRAME_CTRL` (`0x1f8c0`). If `IFPC` is enabled (`idle_level == 3`), write `0x000a1680` to `GMU_PWR_COL_INTER_FRAME_HYST` (`0x1f8c1`) and `GMU_PWR_COL_SPTPRAC_HYST` (`0x1f8c2`), and set Bits 0, 1, and 2 (`0x7`) in `GMU_PWR_COL_INTER_FRAME_CTRL`.
    - Set Bits 0, 4, 8, 9, 10, 11 (`0x0f11`) in `GMU_RPMH_CTRL` (`0x1f8e8`).
14. Read DTCM word at `GMU_CM3_DTCM_START + 0x3fe` (byte offset `+0x07ff8`, i.e. `REG_A6XX_GMU_CM3_DTCM_START + 0xff8` in Linux where `+0xff8` is added to the dword index, which actually reads dword `0x1c400 + 0xff8 = 0x1d3f8` = `GMU_CORE_FW_VERSION`!):
    - If value `<= 0x20010004`, set `mask = 0xffffffff` and `reset_val = 0xbabeface`.
    - Else, set `mask = 0x000001ff` and `reset_val = 0x00000100`.
15. Assert Cortex-M3 reset (`GMU_CM3_SYSRESET` = `1`), write `0` to `GPU_GMU_CX_GMU_PWR_COL_CP_RESP` (`0x1f901`), and deassert Cortex-M3 reset (`GMU_CM3_SYSRESET` = `0`).
16. Poll `GMU_CM3_FW_INIT_RESULT` (`0x1f81c`) every 100 us (up to 100 ms) until `(val & mask) == reset_val`.
17. Start HFI queues: write `0x1` to `GMU_HFI_CTRL_INIT` (`0x1f986`) and poll `GMU_HFI_CTRL_STATUS` (`0x1f980`) until Bit 0 is `1` (timeout 10 ms).

---

## 5. GMU Out-Of-Band (OOB) Handshake Protocol

**[HW]** Out-Of-Band (OOB) requests allow the host CPU to wake the GX power domain or lock power state directly via interrupt registers without sending HFI packets. On A690 (`legacy == false`, HFI v2), the `new` bit positions are used in `GMU_HOST2GMU_INTR_SET` (`0x1f994`), `GMU_GMU2HOST_INTR_INFO` (`0x1f992`), and `GMU_GMU2HOST_INTR_CLR` (`0x1f991`):

| OOB State | Purpose | HFI v2 `set` Bit (A690) | HFI v2 `ack` Bit (A690) | HFI v2 `clear` Bit (A690) | Legacy `set` / `ack` / `clear` Bits |
|---|---|---|---|---|---|
| **`OOB_GPU_SET`** (`1`) | Forces GMU to power on GX and prevent power collapse while CPU initializes 3D registers. | **Bit 30** (`0x40000000`) | **Bit 31** (`0x80000000`) | **Bit 31** (`0x80000000`) | `16` / `24` / `24` |
| **`OOB_PERFCOUNTER_SET`** (`3`) | Keeps GX powered so CPU/profiler can read performance counters without IFPC loss. | **Bit 28** (`0x10000000`) | **Bit 30** (`0x40000000`) | **Bit 29** (`0x20000000`) | `17` / `25` / `25` |
| **`OOB_BOOT_SLUMBER`** (`0`) | Legacy boot/slumber handshake (unused on A690; replaced by HFI messages). | *N/A* | *N/A* | *N/A* | `22` / `30` / `30` |
| **`OOB_DCVS_SET`** (`2`) | Legacy frequency change handshake (unused on A690; replaced by `HFI_H2F_MSG_GX_BW_PERF_VOTE`). | *N/A* | *N/A* | *N/A* | `23` / `31` / `31` |

**[HW]** **OOB Set Sequence**:
1. Write `1 << set_bit` to `GMU_HOST2GMU_INTR_SET` (`0x1f994`).
2. Poll `GMU_GMU2HOST_INTR_INFO` (`0x1f992`) every 100 us (up to 10 ms) until `val & (1 << ack_bit)` is non-zero.
3. Write `1 << ack_bit` to `GMU_GMU2HOST_INTR_CLR` (`0x1f991`) to clear the acknowledge interrupt.

**[HW]** **OOB Clear Sequence**:
1. Write `1 << clear_bit` to `GMU_HOST2GMU_INTR_SET` (`0x1f994`). No acknowledgment poll is required when clearing an OOB vote.

---

## 6. Host Firmware Interface (HFI) Queues and Protocol

### 6.1 HFI Shared Memory Layout (`hfi` Buffer, 16 KiB)

**[HW]** The 16 KiB `hfi` buffer mapped into GMU Context Bank 1 at `hfi.iova` (`GMU_HFI_QTBL_ADDR`) is partitioned as follows:

| Offset in `hfi` BO | Size (Bytes) | Content |
|---|---|---|
| `0x0000` | `0x0018` (24 B) | **Queue Table Header** (`version = 0`, `size = 168`, `qhdr0_offset = 6` dwords, `qhdr_size = 12` dwords, `num_queues = 3`, `active_queues = 3`). |
| `0x0018` | `0x0030` (48 B) | **Queue 0 Header (Command Queue, Host -> GMU)**: `iova = hfi.iova + 0x1000`, `type = 0x0a00` (`(10 << 8) \| 0`), `status = 1`, `size = 1024` dwords (`4096 B`). |
| `0x0048` | `0x0030` (48 B) | **Queue 1 Header (Response Queue, GMU -> Host)**: `iova = hfi.iova + 0x2000`, `type = 0x0a01` (`(10 << 8) \| 1`), `status = 1`, `size = 1024` dwords (`4096 B`). |
| `0x0078` | `0x0030` (48 B) | **Queue 2 Header (Debug Queue, GMU -> Host)**: `iova = hfi.iova + 0x3000`, `type = 0x0a02` (`(10 << 8) \| 2`), `status = 1`, `size = 1024` dwords (`4096 B`). |
| `0x1000` | `0x1000` (4 KiB) | **Queue 0 Data Ring** (1024 `uint32` dwords). |
| `0x2000` | `0x1000` (4 KiB) | **Queue 1 Data Ring** (1024 `uint32` dwords). |
| `0x3000` | `0x1000` (4 KiB) | **Queue 2 Data Ring** (1024 `uint32` dwords). |

**[HW]** Each 48-byte **HFI Queue Header** has the following exact structure:

| Byte Offset | Type | Field Name | Initial Value | Description |
|---|---|---|---|---|
| `0x00` | `uint32` | `status` | `1` | `1` = Queue active. |
| `0x04` | `uint32` | `iova` | `hfi.iova + 0x1000 * (idx + 1)` | 32-bit GMU IOVA of the 4 KiB queue data ring. |
| `0x08` | `uint32` | `type` | `(10 << 8) \| id` | Queue ID (`0` = Command, `1` = Response, `2` = Debug on HFI v2). |
| `0x0c` | `uint32` | `size` | `1024` (`0x400`) | Ring capacity in 32-bit dwords (`4096 / 4`). |
| `0x10` | `uint32` | `msg_size` | `0` | Fixed message size (`0` = variable length). |
| `0x14` | `uint32` | `dropped` | `0` | Count of dropped messages. |
| `0x18` | `uint32` | `rx_watermark` | `1` | Receive watermark (`1`). |
| `0x1c` | `uint32` | `tx_watermark` | `1` | Transmit watermark (`1`). |
| `0x20` | `uint32` | `rx_request` | `1` | Receive interrupt request flag (`1`). |
| `0x24` | `uint32` | `tx_request` | `0` | Transmit interrupt request flag (`0`). |
| `0x28` | `uint32` | `read_index` | `0` | Consumer dword index (`0`..`1023`). |
| `0x2c` | `uint32` | `write_index` | `0` | Producer dword index (`0`..`1023`). |

### 6.2 HFI Packet Header Encoding and 4-Dword Alignment

**[HW]** Every HFI message begins with a 32-bit header dword:
- **Bits 7:0 (`id`)**: Message identifier (`HFI_H2F_MSG_*` or `HFI_F2H_MSG_*`).
- **Bits 15:8 (`size`)**: Total message size in 32-bit dwords, including the header dword itself.
- **Bits 19:16 (`type`)**: Message type (`0` = `HFI_MSG_CMD`, `1` = `HFI_MSG_ACK`, `2` = `HFI_MSG_ACK_V1`).
- **Bits 31:20 (`seqnum`)**: 12-bit monotonically incrementing sequence number (`0`..`0xffe`).

**[HW]** **HFI v2 4-Dword Alignment Rule (Required on A690)**:
- **Writing to Command Queue (Queue 0)**: After writing `dwords` words starting at `write_index`, if the new `index` is not a multiple of 4 (`index % 4 != 0`), software **must** pad the remaining words up to the next 4-dword boundary with the magic value `0xfafafafa`, advancing `index = (index + 1) % size`. Execute a DMA memory barrier (`dma_mb()`), update `header.write_index = index`, and write `0x01` to `GMU_HOST2GMU_INTR_SET` (`0x1f994`).
- **Reading from Response Queue (Queue 1)**: Read `hdr = data[read_index]` and extract `msg_dwords = (hdr >> 8) & 0xff`. Copy `msg_dwords` words from the ring, then round `index` up to the next multiple of 4 modulo `size`: `index = ((index + 3) & ~3) % size`. Execute `dma_mb()` and update `header.read_index = index`.
- **Waiting for Response**: Poll `GMU_GMU2HOST_INTR_INFO` (`0x1f992`) until Bit 0 (`MSGQ = 0x1`) is set (timeout 1 s), clear it by writing `0x1` to `GMU_GMU2HOST_INTR_CLR` (`0x1f991`), and read packets from Queue 1 until a response (`id = 126` `HFI_F2H_MSG_ACK`) whose `ret_header` sequence number (`(ret_header >> 20) & 0xfff`) matches the command's `seqnum` is received.

### 6.3 HFI Message Catalog and Structures

**[HW]** The HFI message formats used on Adreno 6xx are:

| Message Name | ID | Direction | Size (Bytes / Dwords) | Payload Layout (after 4-byte `header` at offset `0x00`) |
|---|---|---|---|---|
| `HFI_H2F_MSG_INIT` | `0` | H2F | `20 B` (`5` dw) | `+0x04`: `seg_id` (`0`), `+0x08`: `dbg_buffer_addr`, `+0x0c`: `dbg_buffer_size`, `+0x10`: `boot_state` (Legacy HFI v1 only). |
| `HFI_H2F_MSG_FW_VERSION` | `1` | H2F | `8 B` (`2` dw) | `+0x04`: `supported_version` (`0x100a0000`, Legacy HFI v1 only). |
| `HFI_H2F_MSG_BW_TABLE` | `3` | H2F | `624 B` (`156` dw) | `+0x04`: `bw_level_num`, `+0x08`: `cnoc_cmds_num`, `+0x0c`: `ddr_cmds_num`, `+0x10`: `cnoc_wait_bitmask`, `+0x14`: `ddr_wait_bitmask`, `+0x18`: `cnoc_cmds_addrs[6]` (`24 B`), `+0x30`: `cnoc_cmds_data[2][6]` (`48 B`), `+0x60`: `ddr_cmds_addrs[8]` (`32 B`), `+0x80`: `ddr_cmds_data[16][8]` (`512 B`). |
| `HFI_H2F_MSG_PERF_TABLE` | `4` | H2F | `236 B` (`59` dw) | `+0x04`: `num_gpu_levels`, `+0x08`: `num_gmu_levels`, `+0x0c`: `gx_votes[16]` (each 12 B: `vote`, `acd = 0xffffffff`, `freq` in kHz), `+0xcc`: `cx_votes[4]` (each 8 B: `vote`, `freq` in kHz). |
| `HFI_H2F_MSG_TEST` | `5` | H2F | `4 B` (`1` dw) | Header only (Legacy HFI v1 only). |
| `HFI_H2F_MSG_ACD` | `7` | H2F | `148 B` (`37` dw) | `+0x04`: `version` (`1`), `+0x08`: `enable_by_level` (bitmask), `+0x0c`: `stride` (`1`), `+0x10`: `num_levels`, `+0x14`: `data[32]` (`128 B`). |
| `HFI_H2F_MSG_START` | `10` | H2F | `4 B` (`1` dw) | Header only. Signals GMU that host initialization is ready. |
| `HFI_H2F_FEATURE_CTRL` | `11` | H2F | `16 B` (`4` dw) | `+0x04`: `feature` (`9` = `HFI_FEATURE_IFPC`, `12` = `HFI_FEATURE_ACD`, `21` = `HFI_FEATURE_CLX`, `38` = `HFI_FEATURE_THINMEM_CFG`), `+0x08`: `enable` (`1`), `+0x0c`: `data` (`0x1680` for `IFPC`, `0` for `ACD`). |
| `HFI_H2F_MSG_CORE_FW_START` | `14` | H2F | `8 B` (`2` dw) | `+0x04`: `handle` (`0`). Starts core GMU state machine. |
| `HFI_H2F_MSG_GX_BW_PERF_VOTE` | `30` | H2F | `16 B` (`4` dw) | `+0x04`: `ack_type` (`1` = blocking), `+0x08`: `freq` (`perf_index`), `+0x0c`: `bw` (`bw_index`). |
| `HFI_H2F_MSG_PREPARE_SLUMBER` | `33` | H2F | `12 B` (`3` dw) | `+0x04`: `bw` (`0`), `+0x08`: `freq` (`0`). Instructs GMU to prepare for power-off slumber. |
| `HFI_F2H_MSG_ERROR` | `100` | F2H | `16 B` (`4` dw) | `+0x04`: `code`, `+0x08`: `payload[2]`. |
| `HFI_F2H_MSG_ACK` | `126` | F2H | `76 B` (`19` dw) | `+0x04`: `ret_header` (echoes command header), `+0x08`: `error` (`0` = success), `+0x0c`: `payload[16]`. |

### 6.4 A690 HFI Startup Sequence (`a6xx_hfi_start`)

**[HW]** Immediately after `GMU_HFI_CTRL_STATUS` Bit 0 becomes `1`, the host sends the following HFI messages in order:
1. **`HFI_H2F_MSG_PERF_TABLE` (ID `4`)**: Sends the GX and CX frequency/ARC vote tables (Section 7).
2. **`HFI_H2F_MSG_BW_TABLE` (ID `3`)**: On A690, sends a fixed single-level bandwidth table:
   - `bw_level_num = 1`
   - `ddr_cmds_num = 3`, `ddr_wait_bitmask = 0x01`
   - `ddr_cmds_addrs[0..2] = { 0x00050004, 0x00050000, 0x000500ac }`
   - `ddr_cmds_data[0][0..2] = { 0x40000000, 0x40000000, 0x40000000 }`
   - `cnoc_cmds_num = 1`, `cnoc_wait_bitmask = 0x01`
   - `cnoc_cmds_addrs[0] = 0x0005003c`
   - `cnoc_cmds_data[0][0] = 0x40000000`, `cnoc_cmds_data[1][0] = 0x60000001`
3. **`HFI_H2F_FEATURE_CTRL` (ID `11`)** (if `IFPC` is enabled): `feature = 9` (`HFI_FEATURE_IFPC`), `enable = 1`, `data = 0x1680`.
4. **`HFI_H2F_MSG_CORE_FW_START` (ID `14`)**: `handle = 0`.
5. **`HFI_H2F_MSG_START` (ID `10`)**: Header-only command completing HFI initialization.
6. Clear and unmask the HFI fault interrupt: write `0xffffffff` to `GMU_GMU2HOST_INTR_CLR` (`0x1f991`) and `~(1 << 23)` (`0xff7fffff`) to `GMU_GMU2HOST_INTR_MASK` (`0x1f993`).

---

## 7. SC8280XP OPP Tables, RPMh ARC Votes, and Devfreq

### 7.1 RPMh ARC Vote Construction (`cmd-db`)

**[HW]** The GMU votes on RPMh voltage rails directly using indices into the SoC Command Database (`cmd-db`, located on SC8280XP via the AOP dictionary at `0x0c3f000c` -> physical `0x80860000`, size `0x13f0`).
- For **GX (`gx_arc_votes`)**, the primary rail is `"gfx.lvl"` (`cmd-db` address `0x30050`) and the secondary rail is `"gmxc.lvl"` if present in `cmd-db`, or `"mx.lvl"` otherwise.
- For **CX (`cx_arc_votes`)**, the primary rail is `"cx.lvl"` and the secondary rail is `"mx.lvl"`.
- For each frequency entry `i` (where `i = 0` is the `0 Hz` off state with `level = 0`), software finds the first index `pindex` in the primary rail's 16-bit level array where `pri[pindex] >= level`, and the first index `sindex` in the secondary rail's array where `sec[sindex] >= pri[pindex]` (clamping `sindex` to `sec_count - 1` if none is high enough). The 32-bit vote word sent in `HFI_H2F_MSG_PERF_TABLE` is:
  `vote = ((pri[pindex] & 0xffff) << 16) | (sindex << 8) | pindex`

### 7.2 SC8280XP GPU (`gx_votes`) and GMU (`cx_votes`) OPP Tables

**[HW]** On SC8280XP, the GMU (`cx_votes`) OPP table has 3 entries (including index 0 = `0 Hz`):

| CX Perf Index | Frequency (`Hz`) | HFI `freq` (`kHz`) | RPMh Regulator Level (`opp-level`) |
|---|---|---|---|
| **0** | `0` | `0` | `0` (Off) |
| **1** | `200,000,000` (200 MHz) | `200000` | `48` (`RPMH_REGULATOR_LEVEL_MIN_SVS`) |
| **2** | `500,000,000` (500 MHz) | `500000` | `128` (`RPMH_REGULATOR_LEVEL_SVS`) |

**[HW]** On SC8280XP, the 3D GPU (`gx_votes`) OPP table has 9 standard entries in upstream DT (indices 0–8, from `0 Hz` / `270 MHz` to `690 MHz`), while the silicon characterization table in `gfx.lvl` (`0, 64, 128, 192, 224, 256, 320, 384, 416`) supports up to `900 MHz` (`TURBO_L3`):

| GX Perf Index | Frequency (`Hz`) | HFI `freq` (`kHz`) | RPMh Level (`opp-level`) | Peak Interconnect BW (`kBps`) |
|---|---|---|---|---|
| **0** | `0` | `0` | `0` (Off) | `0` |
| **1** | `270,000,000` (270 MHz) | `270000` | `64` (`LOW_SVS`) | `451,000` |
| **2** | `410,000,000` (410 MHz) | `410000` | `128` (`SVS`) | `1,555,000` |
| **3** | `500,000,000` (500 MHz) | `500000` | `192` (`SVS_L1`) | `1,555,000` |
| **4** | `547,000,000` (547 MHz) | `547000` | `224` (`SVS_L2`) | `1,555,000` |
| **5** | `606,000,000` (606 MHz) | `606000` | `256` (`NOM`) | `2,736,000` |
| **6** | `640,000,000` (640 MHz) | `640000` | `320` (`NOM_L1`) | `2,736,000` |
| **7** | `655,000,000` (655 MHz) | `655000` | `384` (`TURBO`) | `2,736,000` |
| **8** | `690,000,000` (690 MHz) | `690000` | `416` (`TURBO_L1`) | `2,736,000` |
| *(Optional)* | *Up to `900,000,000` (900 MHz)* | *`900000`* | *`416`–`448`* | *`2,736,000`+* |

### 7.3 Dynamic Frequency Scaling (`devfreq`) and Busy Counter

**[HW]** To change the GPU frequency while the GMU is active:
1. Locate the target `perf_index` (`1`..`nr_gpu_freqs - 1`) in `gpu_freqs[]`.
2. Send `HFI_H2F_MSG_GX_BW_PERF_VOTE` (ID `30`) with `ack_type = 1` (blocking), `freq = perf_index`, and `bw = 0`.
3. Update the interconnect bandwidth quota (`icc_set_bw`) for the `gfx-mem` path (`MASTER_GFX3D` -> `SLAVE_EBI1`) according to the OPP's `opp-peak-kBps`.

**[HW]** To sample GPU utilization for `devfreq`:
- During 3D hardware init, software writes `0xff000000` to `GPU_GMU_AO_GPU_CX_BUSY_MASK` (`0x23b0e`), sets bits 7:0 of `GMU_CX_GMU_POWER_COUNTER_SELECT_0` (`0x1f841`) to `0x20` (bit 5), and writes `0x1` to `GMU_CX_GMU_POWER_COUNTER_ENABLE` (`0x1f840`).
- The 64-bit counter at `GMU_CX_GMU_POWER_COUNTER_XOCLK_0_L` (`0x1f844`) and `GMU_CX_GMU_POWER_COUNTER_XOCLK_0_H` (`0x1f845`) increments at the fixed XO clock rate of **`19,200,000` Hz (19.2 MHz)** whenever the GPU is busy.
- Utilization over a sampling interval `dt` is `(delta_busy_cycles / 19200000) / dt`.
