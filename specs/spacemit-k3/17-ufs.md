# 17: UFS Host

**Status: not started.** *(vendor)*: mainline has no driver or devicetree
node for this block at the baseline.

## Scope

Vendor glue around the standard UFS host controller interface (`spacemit,k3-ufshcd`): clocks (01 §1.8.2), reset, M-PHY and UniPro attribute setup, and link start-up quirks. UFSHCI itself is a published JEDEC standard.

Clock and reset IDs for these blocks are in [01](01-clocks-resets.md).
