# CIX Sky1 Audio Subsystem Functional Specification

Target: the **audio subsystem** of the **CIX Sky1** SoC (CD8180) on the
Radxa Orion O6 and Orange Pi 6 Plus, booted with **ACPI** (the UEFI
default) or devicetree. It covers the audio clock-and-reset unit, the
IPbloq HD-Audio controller, the I2S controllers that feed the five
DisplayPort transmitters, the audio DMA-350, the sound-card topology, and
(briefly) the HiFi5 DSP.

Sources: the mainline Sky1 devicetree, bindings, audio clock/reset driver
and HDA driver; the Sky1-Linux kernel patch set (CIX vendor drivers and
community ACPI fixes) and vendor devicetree; and the MIT-licensed
Sky1-Linux ACPI-to-DT mapping notes. No DSDT was available, so ACPI facts
come from what the drivers parse. Facts found only in vendor or community
sources are marked *(vendor)* or *(community)*. Each statement is tagged:

- **[HW]**: required by the hardware or firmware
- **[POLICY]**: a Linux design choice, informative only
- **[ABI]**: needed for compatibility with existing firmware tables or
  userspace

Generic behaviour defined by public specifications (Intel HD Audio 1.0a,
Arm DMA-350 TRM, virtio/rpmsg) is not repeated; only Sky1 deviations and
integration facts are given. SoC plumbing (mailbox, SCMI, S5 resets, RCSU,
CLKT/RSTL/DLKL) is in `sky1-soc`; DP-side audio is in `sky1-dp-tx`.

| # | Section | Status |
|---|---------|--------|
| 00 | [Overview, address map, power, bring-up order, ACPI IDs (incl. what `CIXHA018` is)](00-overview.md) | draft |
| 01 | [Audio clock and reset unit: registers, clock tree, resets, CLKA](01-clock-reset-unit.md) | draft: shared divider and MCLK codes unverified |
| 02 | [IPbloq HDA controller: deviations from the Intel spec, DMA window, RSVL](02-hda.md) | draft |
| 03 | [I2S controllers, audio DMA-350, request lines, sound card, DSP](03-i2s-dma-dsp.md) | draft: DSP section brief; single-channel I2S not covered |

## Note on ACPI hardware IDs

The public mapping notes mislabel several audio HIDs (`CIXH6011` as a
temperature sensor, `CIXH6020` as a PL330 DMA, `CIXH6000` as a thermal
sensor, `CIXHA018` as a clock/reset unit). The HIDs here come from driver
match tables; see 00 §0.6.
