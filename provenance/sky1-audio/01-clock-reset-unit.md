# Provenance: sky1-audio/01-clock-reset-unit

- Baseline: mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e plus Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f (patches) and Sky1-Linux/linux@665339c3109ba8fbce67aca29e5327e134d6b2b6 (vendor DT)
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Only facts (offsets, bits, IDs, addresses, sequences, table formats) taken; no code or comments copied.
- Sources read:
  - mainline drivers/clk/cix/clk-sky1-audss.c (GPL-2.0-only): 1-500 (register offsets, save list at 26-32, default rates, parent lists, mux tables, gate/div/mux fields for all 41 outputs from 192), 500-962 (divider is power-of-two; gate semantics), 963-1206 (probe order, fixed dividers, runtime suspend/resume order at 1122-1165)
  - mainline drivers/reset/reset-sky1-audss.c (GPL-2.0-only): 1-127 (SW_RST at 0x78, bit map, active-low, 50 us wait)
  - mainline include/dt-bindings/clock/cix,sky1-audss-cru.h and include/dt-bindings/reset/cix,sky1-audss-cru.h (GPL-2.0-only OR BSD-2-Clause): all
  - mainline include/dt-bindings/clock/cix,sky1.h (dual): 75-92 (AUDIO_CLK0-5 SCMI IDs)
  - mainline Documentation/devicetree/bindings/soc/cix/cix,sky1-audss-cru.yaml (dual): all (parent clock roles)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches 0003: 131-928 (vendor clk-sky1-audss.c: six parents and rates, four-input I2S3/4 mux, RCSU writes, NoC pulse timing, resume order, gate tied to runtime power), 3682-3829 (vendor reset driver: 10 ms pulse waits, status read), 5213-5289 (vendor clock header, rates), 6289-6317 (vendor reset header)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches 0007: 4535-4545 (INFO_MCLK divider field layout), 4655-4760 (MCLK divider codes per rate), 6520-6530 and 6600-6660 (I2S_LOOP at 0x5C and its encodings), 338-343 and 901-912 (AP_IRQ at 0x54), 1735-1760 (INFO_HIFI0-2 bits)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches 0056: 77-100 (192 kHz mclk_fs correction, confirming code 0 = /6)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches 0079: all (CLKA table format, audss_cru reference, con_id length)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches 0107: 95-130 (D0 before register access, dependency release)
