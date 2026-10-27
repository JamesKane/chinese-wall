# Source Baseline

| Item | Value |
|------|-------|
| Tree | `../linux` (mainline) |
| Commit | `d135ab453a7400a34930bdda1ef4bdc8acfac80e` |
| `git describe` | `v7.3-16755-gd135ab453a74` (7.4 merge window) |
| Commit date | 2026-10-26 |
| Extraction started | 2026-10-27 |

## Panthor scope

| Path | License (SPDX) |
|------|----------------|
| `drivers/gpu/drm/panthor/*` | GPL-2.0 or MIT |
| `include/uapi/drm/panthor_drm.h` | MIT |
| `Documentation/devicetree/bindings/gpu/arm,mali-valhall-csf.yaml` | GPL-2.0-only OR BSD-2-Clause |
| `Documentation/gpu/panthor.rst` | GPL-2.0+ |

## CIX Sky1 scope (section 11)

| Path | License (SPDX) |
|------|----------------|
| `arch/arm64/boot/dts/cix/sky1.dtsi` | BSD-3-Clause |
| `arch/arm64/boot/dts/cix/sky1-power.h` | GPL-2.0-only (numeric IDs only) |
| `include/dt-bindings/clock/cix,sky1.h` | GPL-2.0-only OR BSD-2-Clause |
| `include/dt-bindings/reset/cix,sky1-s5-system-control.h` | GPL-2.0-only OR BSD-2-Clause |
| `drivers/reset/reset-sky1.c` | GPL-2.0-only (register facts only) |
| `drivers/pmdomain/arm/scmi_perf_domain.c` | GPL-2.0 (behavior only) |

No Sky1 GPU node, binding entry or Panthor match-table entry exists at this
baseline.

Linux dependencies that are GPL-only and therefore cannot be ported directly:
`drivers/iommu/io-pgtable-arm.c` (GPL-2.0-only), which provides the page-table format, and
`drivers/gpu/drm/drm_gem_shmem_helper.c` (GPL-2.0), which provides BO backing.
`drm_gpuvm.c` and `drm_exec.c` are GPL-2.0 OR MIT. The DRM scheduler is MIT.

## Linlon DPU scope

| Path | License (SPDX) |
|------|----------------|
| `drivers/gpu/drm/arm/display/komeda/*`, `drivers/gpu/drm/arm/display/include/*` | GPL-2.0 |
| `Documentation/devicetree/bindings/display/arm,komeda.yaml` | GPL-2.0-only OR BSD-2-Clause |
| `drivers/gpu/drm/drm_color_mgmt.c`, `include/drm/drm_color_mgmt.h` (conversion behavior only) | Permissive MIT-style notice (no SPDX tag) |
| `include/uapi/drm/drm_fourcc.h` (FourCC and AFBC modifier codes) | MIT |
| Sky1 headers, `sky1.dtsi`, `reset-sky1.c` | as in the CIX Sky1 scope above |

Relevant commits: `8fd7576fd6d8` (Linlon-D6 support), `bf5348ed7257` (binding),
`ec902de02217` (`GLB_CORE_ID` field fix). No Sky1 display node, DP
controller binding or DP driver exists at this baseline.

The DRM atomic helpers, GEM DMA helpers and writeback framework the Linux
driver relies on are GPL-2.0 or MIT. The clean room should not depend on
them; the spec describes only the hardware and the KMS-visible behavior.

## Sky1-Linux sources (Linlon DPU second pass)

Out-of-tree sources published by the Sky1-Linux project
(https://github.com/Sky1-Linux), read in the dirty room on 2026-10-27.

| Source | Revision | Paths read | License |
|---|---|---|---|
| `Sky1-Linux/linux-sky1` (patch set for v6.18 LTS) | `57e018a398248d7e5e4d798610df79a557c0629f` | `patches/0005` (CIX `linlon-dp` DPU driver, file list of `dptx`), `0021`, `0034`, `0038`, `0049`; `config/config.sky1` | Driver: GPL-2.0, (C) Arm Technology (China) |
| `Sky1-Linux/linux` branch `main` | `665339c3109ba8fbce67aca29e5327e134d6b2b6` | `arch/arm64/boot/dts/cix/sky1.dtsi`, `sky1-orion-o6.dts` | BSD-3-Clause, (C) Cix Technology |

Facts taken only from these sources are marked *(vendor)* in the specs.

## Sky1 SoC infrastructure scope (specs/sky1-soc)

Same mainline baseline and Sky1-Linux revisions as above, plus:

| Source | Revision | Paths read | License |
|---|---|---|---|
| mainline | baseline | `drivers/mailbox/cix-mailbox.c`, `drivers/reset/reset-sky1.c`, `Documentation/devicetree/bindings/mailbox/cix,sky1-mbox.yaml`, `include/dt-bindings/reset/cix,sky1-*.h`, `drivers/firmware/arm_scmi/transports/smc.c` | GPL-2.0 / dual for bindings |
| `Sky1-Linux/linux-sky1` | `57e018a` | patches 0002, 0003, 0004, 0006, 0010, 0061, 0063, 0070–0077, 0080, 0091, 0092, 0096, 0106, 0107 | GPL-2.0 |
| `Sky1-Linux/acpi-to-dts-tools` | `9ddd22c9926d329a2037a800e99545d09f493b51` | `docs/ACPI-to-DTS-Mapping.md` | MIT |

No DSDT/SSDT/GTDT binaries were available.


## Priority 2 sources (Panthor 11, Linlon DPU 08, sky1-dp-tx)

| Source | Revision | Paths read | License |
|---|---|---|---|
| `Sky1-Linux/linux-sky1` | `57e018a` | patches 0006, 0015, 0016, 0017, 0023, 0053, 0096, 0115, 0116, 0117; `linlon-dp` ACPI parse path; `dptx/` driver and DP/PHY patches (sky1-dp-tx) | GPL-2.0 |
| `Sky1-Linux/linux` branch `main` | `665339c` | `sky1.dtsi` GPU node, protected region; DP/PHY/panel nodes | BSD-3-Clause |
| `Sky1-Linux/sky1-firmware` | `dd81690747ddb092bcc2a221daa0f8f679ee4bc4` | `README.md`, `debian/rules` (GPU firmware install path only; binaries not inspected) | packaging; firmware proprietary |
| `Sky1-Linux/acpi-to-dts-tools` | `9ddd22c` | `examples/generated-o6-from-acpi.dts` | MIT |

## Priority 3 sources (sky1-audio, sky1-peripherals)

Mainline at the baseline: `drivers/clk/cix/clk-sky1-audss.c`, `drivers/reset/reset-sky1-audss.c`, the CIX IPbloq HDA controller under `sound/hda/controllers/`, `drivers/dma/arm-dma350.c`, `drivers/gpio/gpio-cadence.c`, `drivers/i2c/busses/i2c-cadence.c`, `drivers/spi/spi-cadence*.c`, `drivers/pinctrl/cix/*`, the related bindings and headers (GPL-2.0; bindings dual-licensed). `Sky1-Linux/linux-sky1@57e018a` patches 0003, 0004, 0007, 0012, 0020, 0056, 0058, 0062, 0065, 0067, 0079, 0080, 0083, 0091, 0094, 0102, 0107, 0134, 0138 (GPL-2.0). Vendor DT `Sky1-Linux/linux@665339c` (BSD-3-Clause). `acpi-to-dts-tools@9ddd22c` (MIT). Exact files and line ranges are in `provenance/sky1-audio/` and `provenance/sky1-peripherals/`.

## Priority 4 sources (linlon-vpu, zhouyi-npu)

`Sky1-Linux/linux-sky1@57e018a` patches 0010 (CIX/Arm China Linlon VPU driver, `drivers/media/platform/cix/`), 0127; 0009 (Arm China Zhouyi NPU driver, `drivers/misc/armchina-npu/`), 0101, 0103, 0132, 0139. Vendor DT `Sky1-Linux/linux@665339c` (BSD-3-Clause). `acpi-to-dts-tools@9ddd22c` (MIT). Mainline has neither driver at the baseline.

Licence notes: the VPU driver files carry SPDX `GPL-2.0-only` but their headers also contain an Arm China "confidential and proprietary" notice and an "or later" paragraph, which contradict each other. The NPU driver is GPL-2.0; its uapi header `armchina_aipu.h` is `GPL-2.0 WITH Linux-syscall-note` (not MIT).

## Radxa Dragon Q8B (Qualcomm SC8280XP & Toshiba TC956x) sources

Extracted in four phases on 2026-10-27 for the Radxa Dragon Q8B (Qualcomm SC8280XP / Snapdragon 8cx Gen 3, ACPI WoA PEP `QCOM0617`, `soc_id = 449`).

| Source | Revision | Paths read | License (SPDX) |
|---|---|---|---|
| `../linux` (mainline) | `d135ab453a7400a34930bdda1ef4bdc8acfac80e` (`v7.3-16755-gd135ab453a74`) | `drivers/soc/qcom/*`, `drivers/tty/serial/qcom_geni_serial.c`, `drivers/i2c/busses/i2c-qcom-geni.c`, `drivers/pinctrl/qcom/pinctrl-sc8280xp.c`, `drivers/pinctrl/qcom/pinctrl-msm.c`, `drivers/mmc/host/sdhci-msm.c`, `drivers/usb/dwc3/*`, `drivers/cpufreq/qcom-cpufreq-hw.c`, `drivers/thermal/qcom/tsens*`, `drivers/firmware/qcom/qcom_scm*`, `drivers/pmdomain/qcom/rpmhpd.c`, `drivers/interconnect/qcom/*`, `drivers/remoteproc/qcom_q6v5_pas.c`, `drivers/hwspinlock/qcom_hwspinlock.c`, `drivers/mailbox/qcom-ipcc.c`, `drivers/rpmsg/qcom_glink*`, `net/qrtr/*`, `drivers/phy/qualcomm/*`, `sound/soc/qcom/qdsp6/*`, `sound/soc/codecs/{lpass-*,wcd938x*}`, `drivers/soundwire/qcom.c`, `drivers/clk/qcom/{gpucc-sc8280xp.c,dispcc-sc8280xp.c,videocc-sm8350.c,clk-alpha-pll.c,clk-rcg2.c,gdsc.c}`, `drivers/gpu/drm/msm/*`, `drivers/misc/fastrpc.c`, `drivers/net/ethernet/stmicro/stmmac/dwxgmac2*`, `drivers/net/pcs/pcs-xpcs*`, `drivers/net/phy/qcom/qca808x.c`, and related headers/bindings | Drivers: GPL-2.0; bindings: GPL-2.0-only OR BSD-2-Clause; `drivers/gpu/drm/msm/registers/*.xml` and `include/uapi/drm/msm_drm.h`: MIT; `include/uapi/misc/fastrpc.h`: GPL-2.0 WITH Linux-syscall-note |
| `radxa/kernel` (`https://github.com/radxa/kernel`, branch `linux-7.0.11`) | `c00e1e17e65104c8143a8f3d559ef723ae5824d3` | `drivers/media/platform/qcom/iris/*`, `drivers/misc/tc956x_pci.c`, `drivers/net/ethernet/stmicro/stmmac/dwmac-tc956x.c`, `drivers/net/pcs/pcs-xpcs*`, `drivers/net/phy/qcom/qca808x.c`, `arch/arm64/boot/dts/qcom/sc8280xp*.dts*` | Drivers: GPL-2.0-only; SC8280XP devicetrees: BSD-3-Clause |
| `TC956X/TC9564_Host_Driver` (`https://github.com/TC956X/TC9564_Host_Driver`, branch `master`) | `3eca71ce0c5708c92e20f14b8e858cd4483d58f3` | `tc956x_pci.c`, `tc956x_pma.*`, `tc956x_xpcs.*`, `tc956xmac_*`, `dwxgmac2*`, `common.h`, `descs.h`, `hwif.*`, `mmc*` | GPL-2.0 |
| Radxa Dragon Q8B board dumps (`../AbyssBSD/docs/boards/radxa-dragon-q8b/`) | UEFI `BOOT.MXF.1.1` (2026-08-18) / Ubuntu `7.0.11-7-qcom` | `dumps/acpi/*.dsl`, `dumps/dt/live.dts`, `dumps/freebsd/*`, `dumps/linux/*`, empirical board notes | Hardware register/table dumps and BSD-2-Clause notes |

### Phase Breakdown and Permissive Assets

- **Phase 1 — Core SoC, Console, Storage, Thermal, Cpufreq & Fan Quiet-Down (`specs/qcom-sc8280xp/01..03`)**:
  - `01-geni-uart-i2c-tlmm-sdhci.md`: QUPv3 GENI UART (`0x884000`) & I²C (`0xa90000`), TLMM ACPI GPIO (`0x0F100000` + secure-pin mask), SDHCI-MSM v5 `PWRCTL` handshake (`0x8804000`), and DWC3 `GRXTHRCFG` quirk.
  - `02-epss-and-tsens.md`: EPSS cpufreq LUT/perf registers (`0x18591000`/`0x18592000`) and TSENS v2 SROT/TM registers (`0xc222000`–`0xc265000`).
  - `03-scm-cmddb-rpmh-adsp.md`: TrustZone SCM SMC64 protocol + PAS loader (`qcom_scm`, `qcom_adsp`), Command DB (`0x80860000`), and RPMh RSC DRV2 ARC/BCM voter (`qcom_rpmh`).
- **Phase 2 — 2.5 GbE Networking (`specs/tc956x/`)**:
  - Toshiba TC956x PCIe BAR0/BAR4 map, TAMAP `+0x10_0000_0000` DMA offset, XGMAC2 3.01a rings/offloads/MTL flow control, XPCS indirect window, PMA 5-step speed re-init, MSIGEN, and QCA8081 PHY C45 MMD1 `0x9072` SerDes reset.
- **Phase 3 — GLINK Fabric, USB-C Orientation & Audio (`specs/qcom-sc8280xp/04..05`)**:
  - `04-smem-ipcc-glink-usbc.md`: TCSR mutex (`0x1f40000`), SMEM v12 (`0x80900000`), IPCC (`0x408000`), GLINK native protocol + QRTR (`IPCRTR`), and `pmic_glink` USB-C orientation + QMP Combo PHY lane switching (`0x88eb000`/`0x8903000`).
  - `05-audioreach-soundwire-wcd9385.md`: GPR packet routing, APM/PRM AudioReach graph setup, Apps SMMU stream `0xc01` high-nibble IOVA tag (`iova | (1ULL << 32)`), LPASS RX/VA macros, SoundWire controller, and WCD9385 codec/jack-detect sequence.
- **Phase 4 — GPU, Display, VPU & NPU Accelerators (`specs/adreno-a690/`, `specs/qcom-mdss-dpu/`, `specs/qcom-iris-vpu/`, `specs/qcom-fastrpc/`)**:
  - `specs/adreno-a690/`: `qcom_gpucc` CX power-up, GMU HFI/devfreq, A690 CP/ringbuffer/zap-shader/split-SMMU, and DRM `msm` ioctl ABI. Note: `drivers/gpu/drm/msm/registers/*.xml` and `include/uapi/drm/msm_drm.h` are **MIT-licensed** and may be used directly in the clean room.
  - `specs/qcom-mdss-dpu/`: MDSS DPU1 SSPP/LM/CTL/INTF pipeline, DP0/DP1/DP2 AUX/link-training/MSA/TU calculations, `dispcc-sc8280xp`, and QMP Combo PHY DP Alt-Mode tables.
  - `specs/qcom-iris-vpu/`: `qcom_videocc` (Lucid 5LPE PLL + MVS0/MVS0C GDSCs), SCM `mem_protect_video_var`, Iris VPU 2.0 Gen2 HFI protocol, and V4L2 `m2m` decoder/encoder state machine.
  - `specs/qcom-fastrpc/`: `/dev/fastrpc-cdsp` ioctl ABI, GLINK RPC buffer marshaling, and coherent SMMU compute context bank management. Note: Qualcomm's userland `quic/fastrpc` (`libcdsprpc`) is **BSD-3-Clause** and may be read directly in the clean room.

Exact source files and line ranges for each specification are recorded in `provenance/qcom-sc8280xp/`, `provenance/tc956x/`, `provenance/adreno-a690/`, `provenance/qcom-mdss-dpu/`, `provenance/qcom-iris-vpu/`, and `provenance/qcom-fastrpc/`.
