# Qualcomm SC8280XP SoC, GLINK Fabric, USB-C & Audio Functional Specification

Target: the **Qualcomm SC8280XP** (Snapdragon 8cx Gen 3, `soc_id = 449`) SoC on boards such as the **Radxa Dragon Q8B**, booted under **ACPI** (`BOOT.MXF.1.1` UEFI firmware) or devicetree.

Because Qualcomm's Windows-on-Arm (WoA) ACPI tables route power, clocks, thermal sensors, and audio through the undocumented Windows Power Engine Plug-in (`\_SB.PEP0`, `_HID QCOM0617`, `_CID PNP0D80`), an operating system cannot rely on standard ACPI `_CPC`, `_PSS`, `_TMP`, or power resource methods. Instead, the OS must drive the SC8280XP hardware blocks directly using SoC address tables keyed on `QCOM0617` or `\_SB.SOID` (`449`), while respecting the strict stage-2 SMMU and TrustZone invariants enforced by Qualcomm's EL2 hypervisor.

Each statement in this specification suite is tagged:

- **[HW]**: required by the hardware, EL2 hypervisor, or DSP/TrustZone firmware
- **[POLICY]**: a driver design choice (timeouts, polling intervals, heuristics), informative only
- **[ABI]**: needed for compatibility with existing firmware tables (ACPI/DT/SMEM) or userspace

| # | Section | Phase | Status |
|---|---|---|---|
| 01 | [QUPv3 GENI UART & I²C, TLMM ACPI GPIO, SDHCI-MSM v5 & DWC3 RX Threshold](01-geni-uart-i2c-tlmm-sdhci.md) | Phase 1 | Draft |
| 02 | [EPSS CPU Frequency Scaling & TSENS v2 Thermal Sensors](02-epss-and-tsens.md) | Phase 1 | Draft |
| 03 | [TrustZone SCM, Command DB, RPMh RSC DRV2 & ADSP/CDSP PAS Loader](03-scm-cmddb-rpmh-adsp.md) | Phase 1 | Draft |
| 04 | [TCSR Mutex, SMEM v12, IPCC, GLINK Native Protocol, QRTR & PMIC GLINK USB-C](04-smem-ipcc-glink-usbc.md) | Phase 3 | Draft |
| 05 | [GPR, AudioReach APM/PRM, Apps SMMU Stream `0xc01`, LPASS Macros, SoundWire & WCD9385](05-audioreach-soundwire-wcd9385.md) | Phase 3 | Draft |

## Related SC8280XP & Radxa Dragon Q8B Specifications

- [Toshiba TC956x 2.5 GbE PCIe NIC (`specs/tc956x/`)](../tc956x/README.md) (Phase 2)
- [Qualcomm Adreno 690 GPU & GMU (`specs/adreno-a690/`)](../adreno-a690/README.md) (Phase 4)
- [Qualcomm MDSS DPU1 & DisplayPort (`specs/qcom-mdss-dpu/`)](../qcom-mdss-dpu/README.md) (Phase 4)
- [Qualcomm Iris VPU 2.0 Video Codec (`specs/qcom-iris-vpu/`)](../qcom-iris-vpu/README.md) (Phase 4)
- [Qualcomm CDSP FastRPC & Hexagon v68 NPU (`specs/qcom-fastrpc/`)](../qcom-fastrpc/README.md) (Phase 4)
