# 11: Peripheral DMA (PDMA)

**Status: not started.**

## Scope

MMP-lineage peripheral DMA: descriptor format, channel and request-line mapping (`k3-pdma.h`), and the addressing extensions K3 needs for memory above 4 GiB. RAM on the CoM260 starts at `0x1_0000_0000`.

DT compatible `spacemit,k3-pdma`, base `0xD4000000`.

Clock and reset IDs for these blocks are in [01](01-clocks-resets.md).
