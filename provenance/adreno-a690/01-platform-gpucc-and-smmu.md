# Provenance: `specs/adreno-a690/01-platform-gpucc-and-smmu.md`

- **Baseline commits**:
  - `mainline`: `d135ab453a7400a34930bdda1ef4bdc8acfac80e`
  - `AbyssBSD board docs`: `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/`
- **Extracted date**: `2026-10-27`

## Source Files Read

| Source File | Line Range | Extracted Facts |
|---|---|---|
| `/home/jkane/Projects/OS/linux/drivers/clk/qcom/gpucc-sc8280xp.c` | 1–475 | Full `GPUCC` (`0x03d90000`) register map: `gpu_cc_pll0` (`+0x0000`), `gpu_cc_pll1` (`+0x0100`), RCGs (`gpu_cc_gmu_clk_src` `+0x1120`, `gpu_cc_hub_clk_src` `+0x11bc`, `gpu_cc_xo_clk_src` `+0x1010`), dividers (`+0x102c`, `+0x11ec`, `+0x11f0`), branch CBCRs (`+0x1004`–`+0x8030`), resets (`+0x1000`–`+0x1200`), GDSCs (`gpu_cc_cx_gdsc` `+0x106c`, `gpu_cc_gx_gdsc` `+0x100c`), and always-on probe writes (`+0x109c`, `+0x1204`). |
| `/home/jkane/Projects/OS/linux/drivers/clk/qcom/gdsc.c` | 1–350 | GDSC register layout (`GDSCR` Bit 31 `PWR_ON`, Bit 11 `RETAIN_FF_ENABLE`, Bit 1 `HW_CONTROL`, Bit 0 `SW_COLLAPSE`; `CFG_GDSCR` at `+0x04` Bits 31:30). |
| `/home/jkane/Projects/OS/linux/drivers/gpu/drm/msm/adreno/a6xx_gmu.c` | 1800–2569 | GMU memory mapping (`gmu` at `0x03d6a000`, `rscc` at `0x03d8c000`), GMU/HFI IRQ handling, and GMU SMMU Context Bank 1 IOVA range (`0x20000000`–`0x20040000`). |
| `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/gpu-display.md` | 1–168 | Empirical ACPI `\_SB.GPU0` (`QCOM0636`) 7 memory resources and 4 interrupt resources, Resource 3 (`0x03d60000`) vs Resource 6 (`0x03d90000`) overlap, GICv3 GSIVs 332/336/337, and GPU SMMUv2 (`0x03da0000`) stream IDs (`0x00, 0x01, 0x04, 0x08, 0x09, 0x0c, 0x0e, 0x0f`). |
| `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/dumps/freebsd/gpupwr-out.txt` | 1–50 | Empirical `GPUCC` GDSC transitions on Radxa Dragon Q8B (`CX_GDSC` `+0x106c`: `0x80228000` -> `0x80228800`, `GX_GDSC` `+0x100c`: `0x80220001` -> `0x80220800`). |
| `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/dumps/freebsd/smmu-out.txt` | 1–75 | Empirical `0x03da0000` SMMUv2 `IDR0..2` values, `SMR0..7`, `S2CR0..7`, and `CB0`/`CB1` register dumps on Radxa Dragon Q8B. |
