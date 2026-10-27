# Provenance: 03-usbdp-combo-phy

- Baseline (mainline, read-only): d135ab453a7400a34930bdda1ef4bdc8acfac80e
- Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f (patch set, GPL-2.0)
- Sky1-Linux/linux@665339c3109ba8fbce67aca29e5327e134d6b2b6 (devicetree, BSD-3-Clause, (C) Cix Technology)
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Only facts (addresses, register indices, bit positions, values, sequences, property names, IDs) were taken. No code or comments were copied. Lane-mapping and mode tables were rebuilt from the vendor tables and restated.

## Method

Patch 0004's `drivers/phy/cix/*` and `drivers/usb/typec/rts5453.c` were extracted with `git apply --include` into a fresh scratch tree. The `phy-cix-usbdp.c` hunks of 0087, 0100, 0124 and 0139 were applied on top. Patch 0112 did not apply cleanly (context drift). It was read and its effect (ACPI → full reset) was taken from the diff. Line numbers below refer to that applied tree ("phy tree"). DP-driver files refer to the tree after patch 0005 ("dp tree", `drivers/gpu/drm/cix/dptx/`).

## Sources read

- phy tree `drivers/phy/cix/phy-cix-usbdp.h`: 1-347 (all register indices, `PHY_PMA_CMN_CTRL2` bits, lane offsets, GOP status address/size, role enum)
- phy tree `drivers/phy/cix/phy-cix-usbdp.c`:
  - 1-3 (IP name "cdn_sd0804_t7g_typec"), 32-53 (mode bit encoding)
  - 59-218 (USB+DP normal table), 224-372 (DP-only table), 374-388 (8.1 / other-rate PLL1 fixups, incl. the duplicated SS_CTRL3 write), 390-547 (USB+DP flip table) → §3.7, §3.8.2
  - 549-554 (orientation syscon offset 0x420, bit = PHY index) → §3.5.3
  - 591-596 (per-rate HSCLK_DIV/CLK_SEL/INTDIV/FRACDIVL/HIGH_THR), 605-622 (voltage table) → §3.8.2, §3.9
  - 624-643 (DP lane count per mode), 651-669 (pclk retry), 671-757 (exit / init sequence) → §3.5.1, §3.6
  - 765-793 (index×4 addressing, 16-bit values) → §3.3
  - 795-830 (orientation handling, "none" ignored), 832-878 (mux state → mode mapping) → §3.5.3, §3.11.1
  - 912-1010 (re-init skip rules, shared power on/off, GOP skip count) → §3.6, §3.12
  - 1013-1046 (lane count / swing+pre ≤ 3 checks), 1048-1132 (rate change: 0xE003 bit 1, 10 µs waits, lane selection), 1134-1284 (voltage programming, TX_DIAG_ACYA bracket, lane-0 values for all lanes) → §3.8.3, §3.9
  - 1394-1411 (vendor PMA-lane mapping table for states C/D/E) → §3.5.2
  - 1419-1607 (probe: alias/`id`, reset names, `default_conf`, GOP byte handling, `pclk`, syscon lookup, `orientation-switch`/`mode-switch`, child names `dp-port`/`UDPP`/`usb-port`/`USB`, default-to-DP rule) → §3.2, §3.12
  - 1608-1633 (compatible `cix,sky1-usbdp-phy`, ACPI `CIXH2033`)
- patches 0087, 0100, 0112, 0124 (commit messages and diffs): ACPI reset history, syscon guard / "firmware manages direction" statement, final ACPI full-reset behaviour, default-DP rule → §3.2.2, §3.12
- patch 0139 (usbdp hunk only): removal of the FPGA no-op around register access
- phy tree `drivers/phy/cix/phy-cix-usb3.c`: 345-477 (CIXH2034 USB3-only PHY: clocks, resets, child naming) → §3.1 note
- phy tree `drivers/usb/typec/rts5453.c`: 25-50 (status/control register and bit positions), 141-265 (orientation and mode derivation), 340-350 (advertised pin assignments C/D/E) → §3.11.2
- patch 0065: commit message and rts5453 hunk (ACPI `CIXH200D`)
- patches 0110, 0111: commit messages and `cdnsp_acpi_phy_ref_get` hunk (ACPI child-reference resolution of PHY ports, CIXH2033/CIXH2034 ordering) → §3.2.2
- dp tree `trilin_phy.h`: 22-36 (ID register, interface type IDs 0x4D/0x54, power-state encodings), 56-75
- dp tree `trilin_usbdp_phy.h`: 26-62 (interface register offsets)
- dp tree `trilin_usbdp_phy.c`: 43 (power-state timeout), 90-134 (prepare), 146-177 (init), 189-225 (resets), 242-297 (power-state handshake, 200 ns tail), 316-349 (configure bracket), 392-432 (PLL clock disable/enable), 446-574 (poll loops and budgets), 585-595 (ID check) → §3.10
- dp tree `trilin_dptx.c`: 91-100 (interface window accessors), 112-134 (rate update passes the DPCD bw code, lane cap from PHY), 145-162 (controller PHY-status poll), 305-332 (lane-0 VS/PE only), 1316-1360 (PSR A3 / A2→A0), 1994-2080 (core on/off PHY order), 2292-2318 (DT/ACPI `dp_phy` lookup, ID-based variant selection), 2599-2626 (wait for PHY DP mode, 100 × 100 ms), 3199-3222 (`dp_phy` memory resource, ACPI index)
- dp tree `trilin_dptx.h`: 172-175 (ACPI memory resource indices); `trilin_dptx_reg.h`: 81 (controller PHY status register offset, referenced only)
- DT `sky1.dtsi`: 76-79 (aliases), 1652-1915 (four usbdp PHY nodes: reg, resets, clocks, syscon, children), 2023-2045 (USB3-only PHY), 2836-2980 (DP controller nodes: `dp_phy` windows, `phys`), 3198 (S5 syscon node)
- DT `sky1-orion-o6.dts`: 1452-1600 (PHY board config, `default_conf`, `svid`, switches), 1715-1790 (RTS5453 nodes, I²C1, IRQ, graph)
- patch 0020 (Orange Pi 6 Plus DTS): 1406-1545 (PHY `default_conf` per port), 1762-1810 (I²C7, RTS5453 IRQ pin 7)
- patches 0026, 0032: commit messages and diffs (O6/O6N PHY modes, "drives the HDMI bridge")
- mainline `include/dt-bindings/clock/cix,sky1.h`: 116-135 (PHY pclk IDs)
- mainline `git grep` over drivers/phy, Documentation/devicetree/bindings, arch/arm64/boot/dts/cix: no Sky1 USB/DP PHY driver or binding at the baseline
- Known context reused, not re-derived: specs/sky1-soc/02 (reset indices 51-58 in word 0x408), specs/sky1-soc/01 (CLKT/RSTL)

## Inferences (labelled in the spec)

- VCO = 24 MHz × (INTDIV + FRACDIVL/65536), and HIGH_THR ≈ ⅔ INTDIV: computed from table values. Not stated by the source.
- Meanings of MGNFS/CPOST and of the TX_DIAG_ACYA bit 0 bracket: from Cadence register naming.
- "Torrent family": from register names and layout, plus the vendor DT including the Cadence Torrent PHY binding header.
- Duplicated SS_CTRL3 write being a slip for SS_CTRL4.
