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

## Gap research pass, 2026-10-27 (four research subagents; key claims spot-checked by the main session)

- GPU: kernel.org linux-firmware @a6bd225 powervr/ listing; gitlab.freedesktop.org/imagination/linux-firmware branch powervr @8a58f81 (WHENCE entry for rogue_36.56.104.183_v1.fw, file added 0fb5cfd); mesa main @7a3d72c src/imagination/common/device_info/bxm-4-64.h 216-231 (MIT), pvr_device_info.c 62, vulkan/pvr_physical_device.c ~1144; mainline drivers/gpu/drm/imagination pvr_device.c 37, 348, 379, 532-535, pvr_fw.c 27, 90-135, pvr_fw_info.h 70 (GPL-2.0 OR MIT); vendor img-rogue config_kernel.h 2, rgxinit.c ~1893-1926, rgx_fw_info.h 108, rgxfwimageutils.c ~803-835 (dual MIT/GPLv2); github.com/spacemit-com/img-gpu-powervr branch k3-br-v1.0.y firmware header parsed, not redistributed; lkml.iu.edu 2607.3/06720 and 09852.
- A100: github.com/spacemit-com/opensbi branch k3-br-v1.0.y @ad6091f (BSD-2-Clause) k3_defconfig, lib/utils/hsm/fdt_hsm_rpmi.c ~49-52, platform spacemit_k3.c ~51-60, 129-137, 233-241; vendor k3-cpus.dtsi 12-15, 444-447, 498-501; drivers/soc/spacemit/hmp.c; arch/riscv/kernel/vector.c 30-42; cpufeature.c ~895-918.
- Board: docs.banana-pi.org BPI-SM10 pages (en/zh); carrier schematic cdn-resource.spacemit.com/file/product/K3/k3_com260_hw/k3-com260_kit_v02_20260306.pdf; forum.banana-pi.org thread 27431 (boot log); bpi-shop.com listing; archive.spacemit.com Bianbu v4.0 images (env.bin DTB selection by product_name, u-boot.itb fdt list); linux-riscv list 2026-May/091797; mainline k3-com260-ifx.dts 23-120 and k3-com260.dtsi 15-17, 218-234; vendor k3_com260_ifx.dts, k3_com260_kit_v02.dts, k3_com260.dtsi 406, 416, 932.
