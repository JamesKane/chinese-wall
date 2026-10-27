# Provenance: spacemit-k3/README

- Baseline: mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Sources read:
  - arch/riscv/boot/dts/spacemit/k3.dtsi, k3-com260.dtsi, k3-com260-ifx.dts (GPL-2.0 OR MIT): CPU/ISA list, cpu-map, memory node, AIA/ACLINT nodes, peripheral inventory
  - Compatible-to-driver survey: grep of drivers/ and sound/ for every K3 compatible (no driver bodies read for sections 02-12 yet)
  - ../freebsd-src (b9585475ee, 2026-10-24): sys/riscv/riscv/aplic.c 450-465 (rejects msi-parent), identcpu.c, sys/riscv/spacemit/, sys/dev/clk/spacemit/, sys/dev/sdhci/sdhci_fdt_spacemit.c, sys/dev/uart/uart_dev_ns8250.c compat list, sys/dev/eqos/if_eqos_fdt.c compat list, sys/dev/usb/controller/dwc3/dwc3.c compat list (BSD-licensed; read for the gap analysis only)
- Secondary pass, 2026-10-27: vendor repo survey (GitHub API listing of spacemit-com), license tallies of edk2-platforms Silicon/Spacemit and Platform/Spacemit (grep of SPDX tags, grep for GPL/Linux mentions), drivers/gpu/drm/img-rogue license notices, vendor k3.dtsi node inventory (PCIe 2272-2580, power controller 1358, thermal 1398, DPU/DP/UFS compatibles), vendor k3_com260*.dts model strings

## GPU and vendor-only block survey, 2026-10-27

- Compared every compatible in the vendor k3*.dtsi and k3_com260_kit_v02.dts against mainline arch/riscv/boot/dts/spacemit/k3*.dts* (script output only; nothing copied).
- Vendor k3.dtsi 1164-1190 (V2D, mailboxes), 1324-1354 (MPXY/RPMI), 1728-1760 (GPU node and OPPs), 2333-2340 (IOMMU), 2629-2639 (VPU).
- Vendor drivers/gpu/drm/img-rogue: config_kernel.h line 13 (selected core header), cores/rgxcore_km_36.56.104.183.h 1-100 (core ID, single cluster, MIPS firmware processor), pvrversion.h 47-56 (DDK 24.2@6603887), license headers of spacemit/*.
- Vendor drivers/iommu/riscv/iommu-spacemit-k3.c 1-5 (license header, 194 lines; GPL-2.0-only).
- Mainline drivers/gpu/drm/imagination/pvr_device.c 520-545 (accepted core IDs) and pvr_drv.c 1495-1515 (compatibles); Documentation/devicetree/bindings/gpu/img,powervr-rogue.yaml 38-185 and commit 16f30811a935.
- Web: Phoronix and the Imagination linux-firmware GitLab, for which core IDs have published firmware.
