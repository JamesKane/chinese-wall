# Linlon VPU (Mali-V family "v5276") Video Codec Functional Specification

Target: the **video codec block of the CIX Sky1** SoC (ACPI `_HID`
`CIXH3010`, devicetree compatible `armChina,linlon-v8`). The block is an
Arm China **Linlon** multi-core video engine of the Arm Mali-V500/V550/V61
lineage. The vendor software calls the engine "MVE" and its newest
generation "v52_v76" (product IDs `0x5662`–`0x5664`). The engine has up to
four cores and four hardware sessions ("LSIDs"). It runs **vendor
firmware** that is loaded per codec. All codec work happens in the
firmware. The host driver sets up power and the per-session MMU, loads
firmware, passes messages through shared-memory queues, and exposes a V4L2
memory-to-memory decoder and encoder.

Every fact here comes from the CIX/Arm China vendor driver as carried by
the Sky1-Linux kernel patch set, and from the Sky1-Linux devicetree. No
mainline Linux driver or binding exists for this block. All of it is
therefore *(vendor)* unless a public origin is named.

These documents describe hardware and firmware behaviour and the
userspace-visible interface, so that a driver can be written without
looking at the vendor source. Each statement is tagged:

- **[HW]**: required by the hardware or by the firmware ABI. A different
  driver must do this.
- **[POLICY]**: a choice made by the vendor driver. Informative only.
- **[ABI]**: userspace-visible interface that existing userspace (ffmpeg
  and GStreamer V4L2 plugins with Sky1 patches) relies on.

| # | Section | Status |
|---|---------|--------|
| 00 | [Overview, platform integration, ACPI/DT, power, firmware files](00-overview.md) | draft |
| 01 | [Host-visible register map](01-registers.md) | draft: several registers' meanings inferred from names |
| 02 | [Internal MMU, memory map and coherency](02-mmu-and-memory.md) | draft |
| 03 | [Firmware binary, boot, message queues, messages, RPC, session scheduling, power management](03-firmware-interface.md) | draft: protocol v3 assumed for Sky1 firmware |
| 04 | [V4L2 userspace interface](04-v4l2-uapi.md) | draft |

## Bring-up order for a new driver

1. Platform: 00 §0.4–§0.7 (resources, clocks, reset, power domains).
2. Register init, core-count discovery and harvesting: 01, 00 §0.7.
3. Per-session MMU: 02.
4. Firmware load and mapping, queues, the boot handshake: 03 §3.1–§3.5.
5. A stateful H.264/HEVC decode: 03 §3.8 (decoder flow), then 04.
6. Encode: 03 §3.9, 04 §4.4 and §4.7.

## Related specifications

- `sky1-soc/01-mailbox-and-scmi.md`: SCMI power domains (VPU top = 11,
  VPU cores 0–3 = 12–15), SCMI performance domain 9 (VPU), the ACPI `CLKT`
  and `RSTL` tables, GSIV = SPI + 32.
- `sky1-soc/02-s5-reset-and-rcsu.md` §2.4.3: VPU RCSU registers `PGCTRL`
  (`0x21C`), `STRAP_PIN0` (`0x300`) and `STRAP_PIN2` (`0x308`). This spec
  references those registers and does not repeat the field tables.
