# 09: SPI and QSPI Controllers

**Status: not started.**

## Scope

The SSP-family SPI controllers and the QSPI flash controller. The QSPI is a Freescale-lineage QSPI IP, so the work is the K3 differences from that family.

| Block | DT compatible | Bases |
|---|---|---|
| SPI | `spacemit,k3-spi` (fallback `spacemit,k1-spi`) | `0xD4040000`, `0xD4040800`, `0xD401C000` |
| QSPI | `spacemit,k3-qspi` (fallback `spacemit,k1-qspi`) | `0xD420C000` |

Clock and reset IDs for these blocks are in [01](01-clocks-resets.md).
