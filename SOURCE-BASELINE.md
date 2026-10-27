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
