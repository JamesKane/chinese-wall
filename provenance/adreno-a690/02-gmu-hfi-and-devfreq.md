# Provenance: `specs/adreno-a690/02-gmu-hfi-and-devfreq.md`

- **Baseline commits**:
  - `mainline`: `d135ab453a7400a34930bdda1ef4bdc8acfac80e`
  - `AbyssBSD board docs`: `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/`
- **Extracted date**: `2026-10-27`

## Source Files Read

| Source File | Line Range | Extracted Facts |
|---|---|---|
| `/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/adreno/a6xx_gmu.c` | 1–2569 | GMU interrupt handlers (`a6xx_gmu_irq`, `a6xx_hfi_irq`), frequency setting (`a6xx_gmu_set_freq`), Cortex-M3 boot (`a6xx_gmu_start`, `a6xx_gmu_fw_start`), OOB bit tables and handshakes (`a6xx_gmu_oob_bits`, `a6xx_gmu_set_oob`, `a6xx_gmu_clear_oob`), RPMh/RSCC/PDC initialization (`a6xx_gmu_rpmh_init`, `a6xx_rpmh_start`, `a6xx_rpmh_stop`), GMU firmware block loader (`a6xx_gmu_fw_load`), GMU SMMU memory allocations (`icache`, `dummy`, `debug`, `log`, `hfi`), and RPMh ARC vote construction (`a6xx_gmu_rpmh_arc_votes_init`). |
| `/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/adreno/a6xx_gmu.h` | 1–246 | GMU constants, `GMU_BYTE_OFFSET` calculation, OOB state enum, and idle state levels (`GMU_IDLE_STATE_ACTIVE`, `SPTP`, `IFPC`). |
| `/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/adreno/a6xx_hfi.c` | 1–1199 | HFI circular queue read/write with 4-dword alignment and `0xfafafafa` padding, HFI command/ack handshake, A690 bandwidth table (`a690_build_bw_table`), performance table construction (`a6xx_hfi_send_perf_table`), HFI startup sequence (`a6xx_hfi_start`), and queue table initialization (`a6xx_hfi_init`). |
| `/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/adreno/a6xx_hfi.h` | 1–340 | Binary layout of `a6xx_hfi_queue_table_header`, `a6xx_hfi_queue_header`, packet header bitfields (`id`, `size`, `type`, `seqnum`), and all HFI message structures (`HFI_H2F_MSG_*`, `HFI_F2H_MSG_*`). |
| `/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/registers/adreno/a6xx_gmu.xml` | 1–272 | GMU and RSCC register offsets and bitfields (`0x1a880`–`0x26801`, RSCC `0x0004`–`0x053e`). |
| `/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/adreno/a6xx_gpu.c` | 1375–1382, 2337–2352 | GMU XOCLK_0 busy counter setup (`GPU_GMU_AO_GPU_CX_BUSY_MASK`, `GMU_CX_GMU_POWER_COUNTER_SELECT_0`, `GMU_CX_GMU_POWER_COUNTER_XOCLK_0_L`/`_H` at 19.2 MHz). |
| `/home/jkane/Projects/OS/linux/arch/arm64/boot/dts/qcom/sc8280xp.dtsi` | 3393–3513 | SC8280XP `gpu@3d00000` and `gmu@3d6a000` nodes, MMIO ranges, and GPU/GMU OPP tables (`270 MHz`–`690 MHz` and `200 MHz`/`500 MHz`). |
| `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/gpu-display.md` | 142–195 | Empirical Radxa Dragon Q8B ACPI `GPU0` resource mapping, AOP `cmd-db` dictionary (`0xc3f000c` -> `0x80860000`), and `gfx.lvl` (`0x30050`) levels. |
