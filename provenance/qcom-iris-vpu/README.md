# Provenance: `specs/qcom-iris-vpu/README.md`

## Baseline Commits and Empirical Sources
- **Mainline Linux:** `d135ab453a7400a34930bdda1ef4bdc8acfac80e` (`/home/jkane/Projects/OS/linux`)
- **Radxa Linux Kernel (`linux-7.0.11`):** `radxa/kernel@c00e1e17e65104c8143a8f3d559ef723ae5824d3` (`/home/jkane/.gemini/antigravity-cli/brain/2761ccec-ec33-4e2c-9581-a942f4d0f19e/scratch/radxa-kernel`)
- **Empirical Board Documentation (AbyssBSD):** `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/`
- **Extraction Date:** `2026-10-27`

## Source Files Read
1. `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/vpu.md` (lines 1–194)
   - Extracted SC8280XP VPU 2.0 MMIO base (`0x0aa00000`), `videocc` base (`0x0abf0000`), wrapper hardware version `0x60100608`, GSIV `206` (SPI `174`), SMMU stream IDs (`0x2a00` and `0x2a07`, mask `0x0400`), PAS ID `9`, carve-out `0x8c600000` (size `0x700000`), firmware path `qcom/vpu/vpu20_p4_gen2_s6.mbn`, and IOVA reservation `0x0`–`0x25800000`.
2. `/home/jkane/.gemini/antigravity-cli/brain/2761ccec-ec33-4e2c-9581-a942f4d0f19e/scratch/radxa-kernel/arch/arm64/boot/dts/qcom/sc8280xp.dtsi` (lines 4666–4757)
   - Extracted `iris: video-codec@aa00000` and `videocc: clock-controller@abf0000` device tree definitions, clocks, power domains, resets, interconnects, OPP table, and IOMMU stream IDs.
3. `/home/jkane/.gemini/antigravity-cli/brain/2761ccec-ec33-4e2c-9581-a942f4d0f19e/scratch/radxa-kernel/arch/arm64/boot/dts/qcom/sc8280xp-radxa-dragon-q8b.dts` (lines 68–84, 909–914)
   - Extracted board-specific `pil_video_mem` (`0x8c600000`, size `0x700000`), `iris_iova` (`0x0`, size `0x25800000`), and `firmware-name = "qcom/vpu/vpu20_p4_gen2_s6.mbn"`.
4. `/home/jkane/.gemini/antigravity-cli/brain/2761ccec-ec33-4e2c-9581-a942f4d0f19e/scratch/radxa-kernel/drivers/media/platform/qcom/iris/iris_platform_vpu2.c` (lines 1–168)
   - Extracted SC8280XP platform configuration, capabilities, PAS ID `9`, and DMA mask `0xe0000000 - 1`.
5. `/home/jkane/.gemini/antigravity-cli/brain/2761ccec-ec33-4e2c-9581-a942f4d0f19e/scratch/radxa-kernel/drivers/media/platform/qcom/iris/iris_vidc.c` (lines 1–727)
   - Extracted V4L2 M2M video device names (`qcom-iris-decoder`, `qcom-iris-encoder`) and device capabilities.
