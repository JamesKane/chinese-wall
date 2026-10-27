# 07: USB2 PHY and USB3/PCIe Combo PHY

**Status: not started.**

## Scope

PHY power-up, calibration and mode selection for the USB2 PHYs and the shared USB3/PCIe combo PHY. The combo PHY also needs a control register in APMU, named in the devicetree by `spacemit,apmu = <&syscon_apmu 0x14>`.

| Block | DT compatible | Base |
|---|---|---|
| USB2 PHYs | `spacemit,k3-usb2-phy` | `0x81500000`, `0x81800000`, `0x81B00000`, `0xC0A20000` |
| Combo PHY | `spacemit,k3-combo-phy` | `0x81D00000` |

Clock and reset IDs for these blocks are in [01](01-clocks-resets.md).
