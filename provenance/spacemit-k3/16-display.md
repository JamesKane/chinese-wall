# Provenance: spacemit-k3/16-display

- Primary baseline: mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e (no coverage)
- Secondary: spacemit-com/linux-6.18@7d17e358c660f1046eb1f27f26eea33955813afd (k3-br-v1.0.y)
- Extracted: not yet
- Sources to read:
  - drivers/gpu/drm/spacemit/* (vendor)
  - arch/riscv/boot/dts/spacemit/k3-dp0.dtsi, k3-dp1.dtsi, k3-edp0.dtsi (vendor)
  - Permissive cross-check: edk2-platforms Silicon/Spacemit/K3/Drivers/LcdDpDxe (InnoDp.c, InnoDpPhy.c), DpuDxe, LcdMipiDxe; Silicon/Spacemit/Library/LcdMipiDsiCommonLib
