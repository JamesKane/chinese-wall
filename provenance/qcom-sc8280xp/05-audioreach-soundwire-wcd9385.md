# Provenance: `specs/qcom-sc8280xp/05-audioreach-soundwire-wcd9385.md`

## Baseline Repositories
- **Mainline Linux (`../linux`)**: commit `d135ab453a7400a34930bdda1ef4bdc8acfac80e` (`v7.3-16755-gd135ab453a74`)

## Source Files Consulted

| Specification Section | Source File | Lines | Notes |
|---|---|---|---|
| §1 Overview & Audio Topology | `arch/arm64/boot/dts/qcom/sc8280xp.dtsi` | `2999-3350` | `gpr`, `q6apm`, `q6apmdai` (`iommus = <&apps_smmu 0x0c01 0x0>`), `q6prm`, `rxmacro`, `swr1`, `txmacro`, `wsamacro`, `swr0`, `lpass_audiocc`, `swr2`, `vamacro`, `lpass_tlmm` |
| §2 Generic Packet Router (GPR) Framing Protocol | `include/linux/soc/qcom/apr.h` | `78-120` | `struct gpr_hdr`, `struct gpr_pkt`, `GPR_HDR_SIZE` (`20`), `GPR_BASIC_RSP_RESULT` (`0x02001005`), `GPR_BASIC_EVT_ACCEPTED` (`0x02001006`) |
| §2.2–2.3 Static Service IDs & APM Command Envelopes | `sound/soc/qcom/qdsp6/audioreach.h` | `42-76`, `413-440` | `APM_MODULE_INSTANCE_ID` (`1`), `PRM_MODULE_INSTANCE_ID` (`2`), `struct apm_cmd_header`, `struct apm_module_param_data` |
| §3 Proxy Resource Manager (PRM) Protocol | `sound/soc/qcom/qdsp6/q6prm.c` | `27-188` | `PRM_CMD_REQUEST_HW_RSC` (`0x0100100F`), `PRM_CMD_RELEASE_HW_RSC` (`0x01001010`), `PARAM_ID_RSC_HW_CORE` (`0x08001032`), `PARAM_ID_RSC_AUDIO_HW_CLK` (`0x0800102C`) |
| §4.1 Apps SMMU Stream `0x0c01` High-Nibble Address Tagging | `sound/soc/qcom/qdsp6/q6apm-dai.c` | `42`, `670-685`, `848-852`, `1203-1208` | `pdata->sid = args.args[0] & 0xF` (`1` for `0x0c01`) and `phys = dma_addr \| (pdata->sid << 32)` |
| §4.2–4.4 AudioReach Graph Management & Shared-Memory DMA | `sound/soc/qcom/qdsp6/audioreach.h`, `sound/soc/qcom/qdsp6/q6apm.c` | `12-76`, `119-410` (`audioreach.h`); `21-272` (`q6apm.c`) | Module IDs (`0x07001000`–`0x0700117D`), `APM_CMD_GRAPH_OPEN`, `APM_CMD_SHARED_MEM_MAP_REGIONS` (`0x0100100C`), `DATA_CMD_WR_SH_MEM_EP_DATA_BUFFER_V2` (`0x0400100A`), `DATA_CMD_RD_SH_MEM_EP_DATA_BUFFER_V2` (`0x0400100B`) |
| §5.1 RX Macro (`0x03200000`) | `sound/soc/codecs/lpass-rx-macro.c` | `22-350` | `CDC_RX_CLK_RST_CTRL_MCLK_CONTROL` (`0x0100`), `SWR_CONTROL` (`0x0108`), LPASS Codec v2.5 RX path offsets (`0x400`, `0x4C0`, `0x580`), Companders (`0x800`, `0x840`) |
| §5.2 VA Macro (`0x03370000`) & TX Macro (`0x03220000`) | `sound/soc/codecs/lpass-va-macro.c` | `23-173` | `CDC_VA_CLK_RST_CTRL_MCLK_CONTROL` (`0x0000`), `FS_CNT_CONTROL` (`0x0004`), `SWR_CONTROL` (`0x0008`), DMIC & TX decimator controls (`0x0400`+`0x80*n`) |
| §6 Qualcomm SoundWire Master Controller (`v1.6.0`) | `drivers/soundwire/qcom.c` | `25-327` | `swrm_v1_6_data` (`swrm_v1_3_reg_layout`, `default_rows = 50`, `default_cols = 16`, `sw_clk_gate_required = true`), FIFO command packing (`SWRM_REG_VAL_PACK`), port bank registers (`0x1124`–`0x113C`) |
| §7.1–7.2 WCD9385 Codec & AHB Bridge | `drivers/soundwire/qcom.c`, `sound/soc/codecs/wcd938x.c` | `125-129`, `330-350` (`qcom.c`); `30-287` (`wcd938x.c`) | `SWRM_AHB_BRIDGE_*` (`0x0C85`, `0x0C89`, `0x0C8D`, `0x0C91`), `CHIPID_WCD9385` (`0x5`), supplies, IRQ chip (`0x3409`–`0x3414`) |
| §7.3 MBHC v2 Jack, Button, and Impedance Detection | `sound/soc/codecs/wcd938x.c`, `sound/soc/codecs/wcd-mbhc-v2.c` | `61-75`, `201-250` (`wcd938x.c`); `16-250` (`wcd-mbhc-v2.c`) | `wcd_mbhc_fields` register map (`0x3014`–`0x3019`, `0x33DE`–`0x33E2`), voltage thresholds (`75 mV`, `1700 mV`, `1800 mV` MICBIAS2), impedance calculation |
