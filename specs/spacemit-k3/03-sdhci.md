# 03: SD/MMC Host (SDHCI)

**Status: not started.**

## Scope

Vendor-specific registers and quirks on top of the standard SD Host Controller interface: clock and tuning, UHS signalling-voltage switch, PHY or delay-line setup. The deliverable is a list of differences from K1 so that FreeBSD's existing K1 SDHCI driver can be extended.

| Instance | DT compatible | Base |
|---|---|---|
| sdhci0 (SD card on the CoM260 IFX carrier) | `spacemit,k3-sdhci` | `0xD4280000` |

Clock and reset IDs for these blocks are in [01](01-clocks-resets.md).
