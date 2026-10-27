# Chinese Wall: Driver Specification Extraction

This repository holds functional specifications extracted from GPL-licensed
driver source, for clean-room reimplementation in a BSD-licensed operating
system.

## Process

```
 ┌──────────────── DIRTY ROOM ────────────────┐        ┌──── CLEAN ROOM ────┐
 │ reads ../linux source                      │        │ never sees source  │
 │ writes specs/  (facts, behavior, layouts)  │ ─────► │ reads specs/ only  │
 │ writes provenance/  (audit trail, private) │  wall  │ writes BSD driver  │
 └────────────────────────────────────────────┘        └────────────────────┘
```

Only `specs/` and this README cross the wall. `provenance/` stays in the
dirty room so that, if needed, we can show which source informed which
statement and that no expression was copied.

Writing rules are in [CLAUDE.md](CLAUDE.md). The upstream revision is in
[SOURCE-BASELINE.md](SOURCE-BASELINE.md).

## Specifications

| Driver | Hardware | Status |
|---|---|---|
| [Panthor](specs/panthor/README.md) | Arm Mali CSF GPUs (Mali-G610 in Rockchip RK3588; Immortalis-G720 in CIX Sky1) | In progress |
| [Linlon DPU](specs/linlon-dpu/README.md) | Arm China Linlon-D6 display processor (Mali-D71 register model) in CIX Sky1 | Draft |
| [Sky1 SoC](specs/sky1-soc/README.md) | CIX Sky1 mailbox/SCMI, clocks, power domains, resets/RCSU, GPT timer | Draft |
| [Sky1 DP TX](specs/sky1-dp-tx/README.md) | CIX Sky1 Trilinear DisplayPort transmitter, USB-C/DP combo PHY, eDP PHY and panel, PS185 bridge | Draft |
| [Sky1 audio](specs/sky1-audio/README.md) | CIX Sky1 audio subsystem: clock/reset unit, IPbloq HDA, I2S, DMA-350, DSP | Draft |
| [Sky1 peripherals](specs/sky1-peripherals/README.md) | CIX Sky1 Cadence GPIO, I2C, SPI/QSPI, RTS5453H USB-PD, pinctrl | Draft |
| [Linlon VPU](specs/linlon-vpu/README.md) | Arm China Linlon / Mali-V video codec (CIXH3010): MMU, firmware interface, V4L2 ABI | Draft |
| [Zhouyi NPU](specs/zhouyi-npu/README.md) | Arm China Zhouyi X2 v3 NPU (CIXH4000): registers, job model, /dev/aipu ABI | Draft |

## Reviewing a spec before release

Before a spec goes over the wall, a reviewer who did not write it checks that:

1. it contains no code fragments, macro syntax or Linux-internal identifiers
2. no sentence closely tracks a kernel comment (spot-check against provenance)
3. every statement carries a [HW], [POLICY] or [ABI] tag
4. open questions are marked rather than guessed
