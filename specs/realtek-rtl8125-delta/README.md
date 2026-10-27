# Realtek RTL8125 / RTL8126 Delta Functional Specification

**[HW]** Target hardware: the Realtek RTL8125-family 2.5 GbE and RTL8126 5 GbE PCIe controllers with integrated PHY, as used for onboard LAN on current AM5 desktop boards (the X870E reference machine).

**[POLICY]** This is a **delta** specification. A permissively licensed BSD driver already provides chip bring-up for RTL8125, RTL8125B, RTL8125D, RTL8126 and RTL8127: MAC/PHY microcode patching, descriptor rings, checksum offload, VLAN and Wake-on-LAN. The clean room may read that driver directly. This suite covers only what it lacks:

| Gap in the permissive driver | Covered in |
|---|---|
| Chip variants it does not recognize (RTL8125K, RTL9151A, RTL8125CP, RTL8125BP) and the extended chip-ID register | 01 |
| Register access windows (MAC OCP, PHY OCP, paged PHY mapping), shown for cross-checking | 01 |
| Reference initialization values, ASPM entry latency, LTR tables, the RTL8126 PCIe Gen3 quirk | 02 |
| Energy-Efficient Ethernet (the permissive driver turns it off) | 02 |
| TCP segmentation offload for IPv4 and IPv6, short-frame padding rules | 03 |
| Multi-queue RX with RSS, the RSS descriptor format and multi-vector MSI-X | 04 |
| Hardware LED control, and the PHY-patch firmware file format for loading `rtl_nic` firmware files | 05 |

## Chip scope

| Marketing name | Internal MAC generation | Line rate | Notes |
|---|---|---|---|
| RTL8125A | 8125 gen 1 | 2.5 G | No 2.5 G EEE. TSO off by default |
| RTL8125B | 8125 gen 2 | 2.5 G | |
| RTL8125D, RTL8125K, RTL9151A | 8125 gen 3 | 2.5 G | Possible X870E onboard part (unconfirmed) |
| RTL8125CP | 8125 gen 3 variant | 2.5 G | |
| RTL8125BP | 8125 gen 3 variant | 2.5 G | Has a remote-management (DASH) controller, not covered |
| RTL8126A | 8126 | 5 G | |
| RTL8127A | 8127 | 10 G | Out of scope, except that it is the only chip with confirmed RSS (section 04) |

> OPEN QUESTION: the exact chip on the X870E reference board has not been recorded. Read the chip-ID fields (section 01, §2) on the board and note the result in the board documentation.

## Tags

- **[HW]**: required by the hardware. Any driver must do it.
- **[POLICY]**: a reference-driver choice. Informative only.
- **[ABI]**: an externally defined format (here, the firmware file layout).

## Specification Index

| # | Section | Status |
|---|---|---|
| 01 | [Identification and Register Access](01-identification-and-access.md) | Draft |
| 02 | [Initialization, PCIe Power Management, WoL and EEE](02-init-power-eee.md) | Draft |
| 03 | [Transmit Offloads: Checksum, TSO, Padding](03-tx-offloads.md) | Draft |
| 04 | [Multi-queue Receive, RSS and MSI-X](04-rx-rss-msix.md) | Draft |
| 05 | [PHY Patch Firmware Format and LED Control](05-phy-firmware-leds.md) | Draft |

## Critical Summary

1. **[HW]** **RSS has been confirmed on RTL8127A only.** The RSS register block exists in the shared 8125-family register map. The reference driver enables it only on the 10 G part, and runs RTL8125D and RTL8126 with one RX queue and one interrupt vector. Treat RSS on RTL8125D/RTL8126 as an experiment (section 04).
2. **[HW]** **An RSS-format RX descriptor overwrites its buffer address.** On completion the controller writes hash information over the address field. The driver must write the buffer address back before it returns the descriptor to the controller.
3. **[HW]** **The chip does not pad short frames, and checksum offload is unreliable below 60 bytes.** Pad every frame to at least 60 bytes in software. Compute checksums in software for any frame that would be shorter.
4. **[HW]** **Flow control cannot be forced.** Pause behavior follows autonegotiation. With autonegotiation off, pause is disabled and software cannot override it.
5. **[POLICY]** **Wait before enabling ASPM L1.** The reference driver disables ASPM L1 unless the board vendor has marked the controller as ASPM-safe (section 02, §3).
