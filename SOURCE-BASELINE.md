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

Linux dependencies that are GPL-only and therefore cannot be ported directly:
`drivers/iommu/io-pgtable-arm.c` (GPL-2.0-only), which provides the page-table format, and
`drivers/gpu/drm/drm_gem_shmem_helper.c` (GPL-2.0), which provides BO backing.
`drm_gpuvm.c` and `drm_exec.c` are GPL-2.0 OR MIT. The DRM scheduler is MIT.
