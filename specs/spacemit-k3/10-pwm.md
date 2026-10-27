# 10: PWM

**Status: not started.**

## Scope

PXA-lineage PWM: register semantics, period and duty encoding, and enable sequencing. There are 20 instances, `pwm0`–`pwm19`, at `0xD401A000`–`0xD401BC00` and `0xD4020000`–`0xD4022C00` (1 KiB stride).

DT compatible: `spacemit,k3-pwm` (fallback `marvell,pxa910-pwm`).

Clock and reset IDs for these blocks are in [01](01-clocks-resets.md).
