# 08: USB Controllers (DWC3 Integration)

**Status: not started.**

## Scope

What a host needs to bring up the Synopsys DWC3 controllers on K3: clocks, the three resets per port, PHY ordering, and which DWC3 quirk properties apply. The DWC3 core is a published Synopsys design and is not respecified. Unlike most SoCs, the K3 node has no separate child node for the core.

| Instance | Base | Notes |
|---|---|---|
| usb3b | `0x81400000` | CoM260 IFX: USB3 hub |
| usb3c | `0x81700000` | |
| usb3d | `0x81A00000` | |
| usb2_host | `0xC0A00000` | high-speed host only |

All four use DT compatible `spacemit,k3-dwc3`.

Clock and reset IDs for these blocks are in [01](01-clocks-resets.md).
