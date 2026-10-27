# 06: Ethernet MAC Integration

**Status: not started.**

## Scope

SoC glue around the Synopsys DWC Ethernet QoS 5.40a MAC: syscon registers for interface mode and RGMII delays, clock and reset requirements, and anything that differs from a stock 5.40a. The MAC core itself is a published Synopsys design and is not respecified.

| Instance | Base |
|---|---|
| eth0 | `0xCAC80000` |
| eth1 (CoM260, RGMII-ID, PHY at MDIO address 1) | `0xCAC82000` |
| eth2 | `0xCAC8E000` |

All three use DT compatible `spacemit,k3-dwmac` with fallback `snps,dwmac-5.40a`.

Clock and reset IDs for these blocks are in [01](01-clocks-resets.md).
