# Provenance: 04-edp-panel-and-phy

- Baseline (mainline, read-only): d135ab453a7400a34930bdda1ef4bdc8acfac80e
- Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f (patch set, GPL-2.0)
- Sky1-Linux/linux@665339c3109ba8fbce67aca29e5327e134d6b2b6 (devicetree, BSD-3-Clause, (C) Cix Technology)
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Only facts (offsets, register indices, values, sequences, delays, property names, GPIO/pad numbers, IDs) were taken. No code or comments were copied.

Line numbers for `drivers/gpu/drm/cix/dptx/*` refer to the tree after patch 0005 ("dp tree"). Later patches touching these files were checked: 0021 only marks the high-swing table unused. No other patch touches `trilin_edp_phy.*` or `cix_edp_panel.*`.

## Sources read

- dp tree `trilin_edp_phy.h`: 18-60 (interface register offsets incl. CDB, duplicate 0xF4), 61-700 (PMA register index dictionary, used to name indices), 701-740 (PMA_CMN_CTRL1/2 and PLL raw-control bits, lane offsets) → §4.2, §4.4
- dp tree `trilin_edp_phy.c`:
  - 60-187 (initial PMA table) → §4.5.1
  - 189-196 (six-rate table) → §4.5.2
  - 262-308 (prepare), 320-357 (init: table via CDB, PLL0 release, lane enables, ready polls, A2→A0), 369-404 (reset variants), 421-469 (power handshake, 50 µs budget) → §4.4
  - 554-619 (rate change incl. code→entry mapping 0x06/0x09/0x0A/0x0C/0x14/0x1E) → §4.4.3, §4.5.2
  - 640-695 (DP, eDP high-swing and eDP low-swing voltage tables), 697-820 (voltage programming, eDP-table selector, lane-count encoding mismatch → 1 lane = PMA 2 only, otherwise all lanes) → §4.6
  - 832-884 (PLL disable/enable via 0xE003 and 0xE001 bits 2/0), 890-1120 (poll loops and budgets), 1121-1163 (CDB write/read protocol), 1174-1185 (ID 0x4D check) → §4.3, §4.4
- dp tree `trilin_phy.h`: 22-36 (ID register and IDs, power-state encodings), 56-81 (lane-enable encodings 1/3/0xF, cause of the voltage-lane quirk), 116-124 (eDP flag never set; grep confirmed no writer)
- dp tree `cix_edp_panel.c`: 40-128 (panel description fields: delays, size, bpc), 251-285 (supply / power-invert), 287-333 (disable, unprepare order), 335-423 (HPD GPIO poll 1 ms / 2 s, prepare order, reset pulse), 425-444 (enable delay), 446-459 (fixed modes only), 535-618 (ACPI `rotation` and `backlight` reference), 621-715 (probe: property names, GPIO directions, `power` supply, backlight lookup), 734-750 (compatible `cix-edp-panel`, ACPI `CIXH5040`), 762-815 (timing-node and delay properties; size read only with a timing node) → §4.7
- dp tree `trilin_dptx.c`: 1840-1852 (panel disable at stream off), 2066-2072 (panel enable after training), 2292-2318 (PHY variant selection by ID), 2320-2345 (phy_reset pulse 10-20 µs, then 100-200 µs), 2362-2386 and 2890-2902 (panel prepare at host init + 50 ms), 2416-2430 (panel unprepare at host deinit), 3135-3160 (`edp-panel` lookup, DT-only), 3199-3222 (`dp_phy` resource) → §4.1, §4.7.3
- dp tree `trilin_drm.c`: 990-1010 (eDP connector uses preferred mode), 1150-1156 (eDP connector type when a panel is linked) → §4.7
- patch 0022 (whole: commit message, Orange Pi 6 Plus DTS hunk, DP driver bridge hunks): PS185 wiring, PDB as fixed regulator, port@2 graph, HDMI-A connector type → §4.9
- patch 0020 (Orange Pi 6 Plus DTS): 175-232 (backlight and panel nodes), 550-558 (PS185 pad GPIO074), 846-855 (eDP pads), 1113-1180 (dp2 and dp4 nodes incl. old `dp_to_hdmi`, `pdb-gpios` active-low) → §4.7–§4.9
- patch 0065: commit message and pwm_bl hunk (ACPI `CIXH5041`)
- DT `sky1.dtsi`: 2895-2918 (dp2 node: `dp_phy` 0x1414C000/0x4000, three resets incl. `phy_reset`, no `phys`), 3166-3176 (pwm0), 3320-3330 (fch_gpio3)
- DT `sky1-orion-o6.dts`: 130-178 (dp2_backlight: enable GPIO, PWM period, levels, default), 180-193 (panel_edp0: delays, size, enable GPIO, pinctrl), 620-630 (pinctrl_fch_pwm0: DP2_VARY_BL), 917-922 (pinctrl_edp0), 942-946 (pwm0 enabled), 1156-1160 (dp2: `edp-panel`, max rate)
- mainline `arch/arm64/boot/dts/cix/sky1-pinfunc.h`: 107, 112-114 (S0 pad indices 18/19/20, function 0)
- mainline `drivers/pinctrl/cix/pinctrl-sky1.c`: 186-188, 372-374 (only one function per eDP pad)
- mainline `Documentation/devicetree/bindings/display/bridge/simple-bridge.yaml`: 1-60 (`parade,ps185hdm`, transparent non-programmable, ports, optional enable GPIO; dual-licensed binding)
- mainline `drivers/gpio/gpiolib-of.c`: 301-312 (regulator-fixed polarity rule: only `enable-active-high` sets polarity), used to derive the physical PDB level
- mainline `include/dt-bindings/clock/cix,sky1.h`: used to confirm clock names only
- Known context reused: specs/sky1-soc/02 (DP_PHY reset index 29, word 0x400 bit 26)

## Inferences (labelled in the spec)

- Pad function 0 of DP2_DIGON/BLON = path to FCH GPIO3 lines 16/15, and DP2_VARY_BL = PWM0 output: from board pinctrl plus GPIO/PWM usage. Not stated.
- VCO frequencies and ratios: computed from table values with a 24 MHz reference.
- PS185 treated as a plain DP sink (HPD/DPCD/EDID over AUX): from the binding's "non-programmable" description and the DP driver's AUX-only sink access.
- Mapping of panel delays to eDP power-sequence intervals: general eDP knowledge. Not from the source.
