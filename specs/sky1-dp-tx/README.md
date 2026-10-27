# CIX Sky1 DisplayPort Transmitter Functional Specification

Target: the **DisplayPort transmitter controller** of the **CIX Sky1** SoC
(Radxa Orion O6, Orange Pi 6 Plus and similar boards). The controller is a
licensed **Trilinear Technologies DPTX** core with CIX glue (an "RCSU"
register window and a combined USB-C/DP or eDP PHY). There are five
instances, one behind each Linlon-D6 DPU.

All facts come from the CIX vendor driver and Sky1 devicetree published by
the Sky1-Linux project, plus later community fixes. Nothing here comes from
mainline Linux, which has no driver for this block. Facts that come only
from the vendor driver are marked *(vendor)*. They describe what the
shipping driver does on this silicon; no Trilinear datasheet was available.

These documents describe hardware behaviour and the userspace-visible
interface so that a driver can be written without looking at the Linux
source. Each statement is tagged:

- **[HW]**: required by the hardware or by the DisplayPort standard
- **[POLICY]**: a vendor-driver design choice, informative only
- **[ABI]**: needed for compatibility with existing userspace

DisplayPort protocol facts (DPCD addresses, link-training rules, SDP
formats) are those of the public VESA DisplayPort 1.4 standard and are cited
only where the controller depends on them.

| # | Section | Status |
|---|---------|--------|
| 00 | [Overview, instances, resources, ACPI attachment](00-overview.md) | draft |
| 01 | [DPTX core register map and RCSU adapter registers](01-registers.md) | draft |
| 02 | [Operation: power-up, HPD, AUX, link training, video, audio, PSR, MST, PM, firmware handover](02-operation.md) | draft |
| 03 | [USB-C/DP combo PHY](03-usbdp-combo-phy.md) | draft (separate extraction) |
| 04 | [eDP panel and eDP PHY (DP 2)](04-edp-panel-and-phy.md) | draft (separate extraction) |
| 05 | [Userspace-visible interface](05-uapi.md) | draft |

## Scope limits

- First priority is single-stream (SST) DisplayPort output. Audio, PSR and
  MST follow. HDCP is summarised only (register locations and the existence
  of a private userspace interface).
- The `dsc` register window is declared by the devicetree but never mapped
  or programmed by the vendor driver. Display Stream Compression is out of
  scope.
- PHY programming (lane power states, swing/pre-emphasis tables, PLL) is in
  sections 03 and 04. This section only states what the controller needs
  from the PHY.

## Related specifications

- `specs/linlon-dpu/08-cix-sky1.md`: the DPU side (instances, pixel clocks,
  pipeline k → DP input k, two-pixels-per-clock mode, firmware splash).
- `specs/sky1-soc/`: SCMI clocks, S5 reset controller, `CLKT`/`RSTL` ACPI
  tables.
