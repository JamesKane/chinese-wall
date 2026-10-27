# Provenance: spacemit-k3/13-pcie

- Primary baseline: mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e (no coverage)
- Secondary: spacemit-com/linux-6.18@7d17e358c660f1046eb1f27f26eea33955813afd (k3-br-v1.0.y)
- Extracted: not yet
- Sources to read:
  - drivers/pci/controller/dwc/pcie-spacemit.c (vendor; EP variant pcie-spacemit-ep.c out of scope)
  - drivers/phy/spacemit/phy-k3-pcie.c (vendor)
  - arch/riscv/boot/dts/spacemit/k3.dtsi 2272-2580 (vendor, GPL-2.0 OR MIT)
  - Permissive cross-check: edk2-platforms Silicon/Spacemit/K3/Library/K3PcieHostBridgeLib, Silicon/Spacemit/Library/DesignWarePcieControllerLib; Platform/Spacemit/K3/AcpiTables/Mcfg.aslc, Dsdt/Pci.asl
