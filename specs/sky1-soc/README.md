# CIX Sky1 SoC Infrastructure Functional Specification

Target: the **CIX Sky1** SoC (CD8180), on boards such as the Radxa Orion O6
and Orange Pi 6 Plus, booted with **ACPI** (the UEFI default) or devicetree.
These sections describe the shared SoC plumbing that device drivers depend
on: firmware communication, clocks, power domains, resets, and the timer.

Sources: the mainline Sky1 devicetree, bindings and drivers; the
Sky1-Linux kernel patch set and devicetree (CIX vendor drivers adapted for
ACPI); and the MIT-licensed Sky1-Linux ACPI-to-DT mapping notes. No DSDT
was available, so ACPI facts come from what the drivers parse and from
the mapping notes. Each statement is tagged:

- **[HW]**: required by the hardware or firmware
- **[POLICY]**: a Linux design choice, informative only
- **[ABI]**: needed for compatibility with existing firmware tables or
  userspace

| # | Section | Status |
|---|---------|--------|
| 01 | [Mailbox, SCMI transports, CLKT and the ACPI clock provider](01-mailbox-and-scmi.md) | draft |
| 02 | [Reset controllers, RSTL/RSNL/DLKL, subsystem RCSUs, memory repair](02-s5-reset-and-rcsu.md) | draft: DMRP registers unknown |
| 03 | [General-purpose timer (GPT)](03-gpt-timer.md) | draft: deep-idle behaviour unverified |

## Note on ACPI hardware IDs

The public Sky1-Linux ACPI-to-DT notes contain a HID table that disagrees
with the drivers' ACPI match tables in several places (for example it lists
`CIXH5010` as USB and `CIXH4000` as the GPU). The drivers are authoritative:
`CIXH5010` is the DPU, `CIXH5000` the GPU, `CIXH4000` the NPU. HIDs in these
specs come from driver match tables.
