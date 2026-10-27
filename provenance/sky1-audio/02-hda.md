# Provenance: sky1-audio/02-hda

- Baseline: mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e plus Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f (patches) and Sky1-Linux/linux@665339c3109ba8fbce67aca29e5327e134d6b2b6 (vendor DT)
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Only facts (offsets, bits, IDs, addresses, sequences, table formats) taken; no code or comments copied.
- Sources read:
  - mainline sound/hda/controllers/cix-ipbloq.c (GPL-2.0): 1-434 (resources, 32-bit DMA mask, -0x90000000 offset, polling/no RIRB IRQ, aligned MMIO, 100 us stream-stop delay, LPIB position, 8 codec slots, 1 s jack poll, 100 ms power save, runtime PM order, card names)
  - mainline sound/hda/core/controller.c (GPL-2.0): 40-95 (where the address offset and the RIRB interrupt suppression apply), 645-660 (position-buffer address offset); sound/hda/core/stream.c: 240-325, 480-495 (stream-reset delay; BDL and BDL-entry offsets); include/sound/hdaudio.h: 340-465 (aligned access semantics); sound/hda/common/controller.c: 605-620 (128-byte buffer constraint)
  - mainline Documentation/devicetree/bindings/sound/cix,sky1-ipbloq-hda.yaml (GPL-2.0-only OR BSD-2-Clause): all
  - Sky1-Linux/linux@665339c3109ba8fbce67aca29e5327e134d6b2b6 arch/arm64/boot/dts/cix/sky1.dtsi (BSD-3-Clause): 628-631 (HDA reserved memory), 1439-1465 (hda_bus dma-ranges, HDA node)
  - Sky1-Linux/linux@665339c3109ba8fbce67aca29e5327e134d6b2b6 arch/arm64/boot/dts/cix/sky1-orion-o6.dts (BSD-3-Clause): 839-848 (HDA pins), 1075-1082 (pdb GPIO, model, memory-region)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches 0007: 3545-4440 (vendor HDA driver: board verb tables and pin defaults, pdb/depop GPIO timing, reset pulse 10-20 us, ACPI ID CIXH6020)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches 0056: 1-76 (ALC256 fixup for SSID 10ec:129e)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches 0080: 190-208 (reserved-mem guarded under ACPI)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches 0083: all (ACPI DMA window, RSVL table format on CIXA1019, write-back mapping)
