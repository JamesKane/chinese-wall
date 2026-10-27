# Provenance: tc956x/README

- **Baselines**:
  - `mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e`
  - `radxa/kernel@c00e1e17e65104c8143a8f3d559ef723ae5824d3` (`linux-7.0.11`)
  - `TC956X/TC9564_Host_Driver@3eca71ce0c5708c92e20f14b8e858cd4483d58f3`
- **Extracted**: `2026-10-27`
- **Methodology**: Clean-room functional specification extraction. Only hardware facts (PCI IDs, BAR sizes, register offsets, bit positions, DMA translation formulas, initialization sequences) were extracted; no source code or comments were copied.
- **Sources Read**:
  - `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/ethernet-tc956x.md` (lines 1–130): Board-level PCIe topology, EEPROM MAC storage, BAR overview, and empirical bring-up observations.
  - `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/dumps/freebsd/tc956x-p0.txt` (lines 1–36): Live hardware register dump of Port 0 on Radxa Dragon Q8B.
  - `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/dumps/linux/lspci.txt` (lines 487–799): Full `lspci -vvv` output for PCIe segment `0004` (`0004:00:00.0` through `0004:03:00.1`).
  - `radxa-kernel/drivers/misc/tc956x_pci.c` (lines 1–668): PCIe bridge, SFR, clock/reset, GPIO, and virtual irqchip architecture.
  - `radxa-kernel/drivers/net/ethernet/stmicro/stmmac/dwmac-tc956x.c` (lines 1–810): Ethernet port glue, TAMAP, PMA, XPCS, and speed reconfiguration.
  - `TC9564_Host_Driver/tc956x_pci.c` (lines 1–2400) and `TC9564_Host_Driver/tc956xmac_main.c` (lines 1–4000): Out-of-tree Toshiba reference driver cross-check.
