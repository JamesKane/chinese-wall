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
