# 02: Pin Controller and GPIO

**Status: not started.**

## Scope

Pin multiplexing, pad configuration (drive strength, pull, I/O voltage via `power-source`) and the GPIO controller with its interrupt routing.

| Block | DT compatible | Base |
|---|---|---|
| Pin controller | `spacemit,k3-pinctrl` | `0xD401E000` |
| GPIO | `spacemit,k3-gpio` | `0xD4019000` |

Clock and reset IDs for these blocks are in [01](01-clocks-resets.md).
