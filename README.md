# Chinese Wall: Driver Specification Extraction

This repository holds functional specifications extracted from GPL-licensed
driver source, for clean-room reimplementation in a BSD-licensed operating
system.

## Process

```
 ┌──────────────── DIRTY ROOM ────────────────┐        ┌──── CLEAN ROOM ────┐
 │ reads ../linux source                      │        │ never sees source  │
 │ writes specs/  (facts, behavior, layouts)  │ ─────► │ reads specs/ only  │
 │ writes provenance/  (audit trail, private) │  wall  │ writes BSD driver  │
 └────────────────────────────────────────────┘        └────────────────────┘
```

Only `specs/` and this README cross the wall. `provenance/` stays in the
dirty room so that, if needed, we can show which source informed which
statement and that no expression was copied.

Writing rules are in [CLAUDE.md](CLAUDE.md). The upstream revision is in
[SOURCE-BASELINE.md](SOURCE-BASELINE.md).

## Specifications

| Driver | Hardware | Status |
|---|---|---|
| [Panthor](specs/panthor/README.md) | Arm Mali CSF GPUs (Mali-G610 in Rockchip RK3588; Immortalis-G720 in CIX Sky1) | In progress |
| [Linlon DPU](specs/linlon-dpu/README.md) | Arm China Linlon-D6 display processor (Mali-D71 register model) in CIX Sky1 | Draft |
| [Sky1 SoC](specs/sky1-soc/README.md) | CIX Sky1 mailbox/SCMI, clocks, power domains, resets/RCSU, GPT timer | Draft |
| [Sky1 DP TX](specs/sky1-dp-tx/README.md) | CIX Sky1 Trilinear DisplayPort transmitter, USB-C/DP combo PHY, eDP PHY and panel, PS185 bridge | Draft |
| [Sky1 audio](specs/sky1-audio/README.md) | CIX Sky1 audio subsystem: clock/reset unit, IPbloq HDA, I2S, DMA-350, DSP | Draft |
| [Sky1 peripherals](specs/sky1-peripherals/README.md) | CIX Sky1 Cadence GPIO, I2C, SPI/QSPI, RTS5453H USB-PD, pinctrl | Draft |
| [Linlon VPU](specs/linlon-vpu/README.md) | Arm China Linlon / Mali-V video codec (CIXH3010): MMU, firmware interface, V4L2 ABI | Draft |
| [Zhouyi NPU](specs/zhouyi-npu/README.md) | Arm China Zhouyi X2 v3 NPU (CIXH4000): registers, job model, /dev/aipu ABI | Draft |
| [Qualcomm SC8280XP SoC](specs/qcom-sc8280xp/README.md) | Qualcomm SC8280XP (Radxa Dragon Q8B): GENI UART/I²C, TLMM GPIO, SDHCI-MSM v5, DWC3, EPSS cpufreq, TSENS v2, SCM, Command DB, RPMh, ADSP/CDSP PAS, SMEM v12, IPCC, GLINK/QRTR, PMIC GLINK USB-C, AudioReach/SoundWire/WCD9385 | Draft |
| [Toshiba TC956x 2.5 GbE](specs/tc956x/README.md) | Toshiba TC956x PCIe 2.5 GbE controller: BAR0/BAR4 SFR, TAMAP DMA offset, Synopsys XGMAC2 3.01a, XPCS, SerDes PMA, MSIGEN, QCA8081 PHY | Draft |
| [Qualcomm Adreno 690 GPU](specs/adreno-a690/README.md) | Qualcomm Adreno 690 GPU (QCOM0636): GPUCC CX power-up, GMU HFI/devfreq, A690 CP/ringbuffer/zap-shader/split-SMMU, `/dev/dri/renderD128` msm ioctl ABI | Draft |
| [Qualcomm MDSS DPU & DP](specs/qcom-mdss-dpu/README.md) | Qualcomm SC8280XP MDSS DPU1 (v8.0) pipeline (SSPP/LM/CTL/INTF), DisplayPort (DP0/DP1/DP2) AUX/link-training/MSA/TU math, dispcc, QMP Combo PHY DP Alt-Mode | Draft |
| [Qualcomm Iris VPU 2.0](specs/qcom-iris-vpu/README.md) | Qualcomm Iris VPU 2.0 video codec: videocc (Lucid 5LPE PLL, MVS0/MVS0C GDSCs), SCM `mem_protect_video_var`, Gen2 HFI protocol, V4L2 m2m ABI | Draft |
| [Qualcomm FastRPC (CDSP/NPU)](specs/qcom-fastrpc/README.md) | Qualcomm FastRPC kernel transport (`/dev/fastrpc-cdsp` ioctl ABI, GLINK RPC marshaling, coherent SMMU compute context banks for Hexagon v68 HTP NPU) | Draft |
| [SpacemiT K3](specs/spacemit-k3/README.md) | SpacemiT K3 RISC-V SoC on the K3-CoM260 module (BPI-SM10): PLLs and clock/reset controllers; stubs for pinctrl/GPIO, SDHCI, I2C, P1 PMIC, Ethernet glue, USB PHYs/DWC3, SPI/QSPI, PWM, PDMA, I2S, and (from the SpacemiT vendor kernel) PCIe, power domains, thermal, display, UFS | In progress |
| [ITE IT87 Super I/O hwmon](specs/ite-it87-superio/README.md) | ITE IT8689E / IT87952E Environment Controller on the X870E reference board: Super I/O access and detection, voltage/temperature/16-bit fan sensors, six-channel PWM and automatic fan control, Gigabyte WMI temperatures (IT8696E unconfirmed) | Draft |

## Reviewing a spec before release

Before a spec goes over the wall, a reviewer who did not write it checks that:

1. it contains no code fragments, macro syntax or Linux-internal identifiers
2. no sentence closely tracks a kernel comment (spot-check against provenance)
3. every statement carries a [HW], [POLICY] or [ABI] tag
4. open questions are marked rather than guessed
