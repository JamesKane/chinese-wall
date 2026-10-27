# Provenance: `specs/qcom-fastrpc/README.md`

## Baseline Commits and Empirical Sources
- **Mainline Linux:** `d135ab453a7400a34930bdda1ef4bdc8acfac80e` (`/home/jkane/Projects/OS/linux`)
- **Radxa Linux Kernel (`linux-7.0.11`):** `radxa/kernel@c00e1e17e65104c8143a8f3d559ef723ae5824d3` (`/home/jkane/.gemini/antigravity-cli/brain/2761ccec-ec33-4e2c-9581-a942f4d0f19e/scratch/radxa-kernel`)
- **Empirical Board Documentation (AbyssBSD):** `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/`
- **Extraction Date:** `2026-10-27`

## Source Files Read
1. `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/npu.md` (lines 1–147)
   - Extracted NSP0/CDSP architecture (Hexagon v68, 4 HVX threads + HMX), PAS ID `18` (`0x12`), carve-out `0x8a600000` (size `0x1e00000` = 30 MiB), NSP1 fused-off status, SMP2P SMEM IDs (`94`, `432`), IPCC client `6`, remote PID `5`, GLINK channel `"fastrpcglink-apps-dsp"`, and SMMU stream IDs (`0x3181`..`0x318f`, mask `0x0420`).
2. `/home/jkane/.gemini/antigravity-cli/brain/2761ccec-ec33-4e2c-9581-a942f4d0f19e/scratch/radxa-kernel/arch/arm64/boot/dts/qcom/sc8280xp.dtsi` (lines 688–755, 3357–3386, 7481–7630)
   - Extracted `remoteproc_nsp0` (`0x1b300000`), `fastrpc` child node under `glink-edge`, and `compute-cb@1`..`compute-cb@14` SMMU stream IDs.
3. `/home/jkane/.gemini/antigravity-cli/brain/2761ccec-ec33-4e2c-9581-a942f4d0f19e/scratch/radxa-kernel/arch/arm64/boot/dts/qcom/sc8280xp-radxa-dragon-q8b.dts` (lines 80–90, 1467–1471)
   - Extracted Dragon Q8B `pil_nsp0_mem` (`0x8a600000`, size `0x1e00000`) and `firmware-name = "qcom/sc8280xp/qccdsp8280.mbn"`.
4. `/home/jkane/Projects/OS/linux/drivers/misc/fastrpc.c` (lines 1–2832)
   - Extracted FastRPC domain IDs, device node registration (`fastrpc-cdsp`, `fastrpc-cdsp-secure`), `default_soc_data` (`sid_pos = 32`, `dma_addr_bits = 32`), and GLINK/SMMU architecture.
