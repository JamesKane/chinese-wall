# Zhouyi NPU (CIX Sky1) Functional Specification

Target: the neural processing unit in the **CIX Sky1** SoC (CD8180), on
boards such as the Radxa Orion O6 and Orange Pi 6 Plus. The NPU is an
**Arm China Zhouyi X2**, which identifies itself as Zhouyi ISA version
**v3**. Linux drives it with an out-of-tree vendor driver that registers
a character device `/dev/aipu`. The vendor's closed userspace runtime (the
Zhouyi "Compass" user-mode driver) talks to that device. Booting with
**ACPI** (the UEFI default) and with devicetree are both covered.

Sources: the Arm China / CIX driver carried in the Sky1-Linux kernel patch
set (GPL-2.0; the userspace header is GPL-2.0 WITH Linux-syscall-note),
the vendor Sky1 devicetree (BSD-3-Clause), the public dual-licensed Sky1
power, clock and reset headers, and the MIT-licensed Sky1-Linux
ACPI-to-DT notes. Mainline Linux has **no** Zhouyi driver and no Sky1 NPU
node. No DSDT was available, so ACPI facts come from what the driver
parses. Facts known only from vendor code are marked *(vendor)*. In this
spec set that covers almost everything, because the vendor driver is the
only source.

Each statement is tagged:

- **[HW]**: required by the hardware or firmware
- **[POLICY]**: a choice the vendor driver makes, informative only
- **[ABI]**: needed for binary compatibility with the vendor userspace
  runtime, or with firmware tables

| # | Section | Status |
|---|---------|--------|
| 00 | [Overview: architecture, resources, ACPI/DT attachment, power, DVFS, memory](00-overview.md) | draft: TEC/core counts and ACPI child HIDs unknown |
| 01 | [Register map (TSM, command pools, clusters, debug)](01-registers.md) | draft: no DMRP or NPU RCSU registers found |
| 02 | [Operation: reset, identification, job submission, TCBs, interrupts, PM](02-operation.md) | draft |
| 03 | [`/dev/aipu` userspace ABI](03-uapi.md) | draft: layouts verified by compiling the header on LP64 |

## Note on ACPI hardware IDs

- `CIXH4000` is the NPU. It is the only ACPI ID in the driver's match
  table, and it sits at the NPU's address (`0x14260000`, SPI 327). The
  public ACPI-to-DT notes call `CIXH4000` "GPU", and their generated
  example DTS prints this device as a Mali node at `0x14260000`. That is
  wrong; the GPU is `CIXH5000` at `0x15000000` (see the sky1-soc README).
- `CIXH4010` is not the NPU. No Sky1-Linux driver matches it. The
  ACPI-to-DT script maps it to an ISP compatible, and the vendor DT puts
  the ISP (`armcb,sky1-isp`) at `0x14340000`. It is probably the camera
  ISP. Nothing here depends on it.
- The notes also list `CIXHA002` as "AIPU". No driver matches that ID
  either. Treat it as unverified.
