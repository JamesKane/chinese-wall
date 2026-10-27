# Provenance: sky1-soc/01-mailbox-and-scmi

- Baseline: mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e plus the Sky1-Linux revisions below
- Extracted: 2026-10-27 by Claude (Opus 5.5) in a Claude Code session for jkane
- Only facts (offsets, bits, IDs, sequences, table formats) taken; no code or comments copied.
- Sources read:
  - mainline drivers/mailbox/cix-mailbox.c (GPL-2.0): 1-639 (register offsets, channel types, interrupt bits, send/receive and startup sequences)
  - mainline Documentation/devicetree/bindings/mailbox/cix,sky1-mbox.yaml (GPL-2.0 OR BSD-2-Clause): all
  - mainline arch/arm64/boot/dts/cix/sky1.dtsi (BSD-3-Clause): 227-258 (SCMI agents), 545-580 (mailboxes, shmem), 824-828 (TF-A shmem)
  - mainline arch/arm64/boot/dts/cix/sky1-power.h: all (PD and perf IDs); include/dt-bindings/clock/cix,sky1.h (dual): count and tail
  - mainline drivers/firmware/arm_scmi/transports/smc.c: 54-73, 142-186, 232-256 (param page/offset only for arm,scmi-smc-param); include/linux/scmi_protocol.h 208-214 (power state encoding)
  - Sky1-Linux/linux@665339c3109ba8fbce67aca29e5327e134d6b2b6 arch/arm64/boot/dts/cix/sky1.dtsi (BSD-3-Clause): 1467-1481, 3588-3639 (all 8 mailbox instances, cix,mbox_dir integer)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0003 (cix-mailbox.c ACPI variant: CIXHA001, cix,mbox_dir; shmem.c/smc.c debug; scmi_pm_domain.c state verify, early init)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0061 (CIXHA006), 0070 (removal of 0x80 shmem offset logic), 0071, 0072 (CIXHA004/5 shmem via _DSD + _CRS), 0073 (_DSD mboxes/shmem counting), 0074 (ACPI protocol enumeration, reg-io-width), 0075 (ACPI mbox references, overlapping shmem/mailbox resources)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0076 (CLKT format, CIXHA010, scmi clock registration), 0077 (CLKT dependency on CIXHA010), 0106 (CLKT element 2 consumer), 0080 (SCMI PD disabled under ACPI; GPT clock names), 0091/0092 commit messages (CIXHA010 dependency chain)
  - Sky1-Linux/linux-sky1@57e018a398248d7e5e4d798610df79a557c0629f patches/0096 lines 290-385 (raw SMC SCMI POWER_STATE_SET sequence, 0xc2000001, 0x84380000), 0015/0023 titles (GPU DVFS rate limiting)
  - Sky1-Linux/acpi-to-dts-tools@9ddd22c9926d329a2037a800e99545d09f493b51 docs/ACPI-to-DTS-Mapping.md (MIT): CLKT/_DSD/interrupt-numbering sections, HID table (noted as unreliable)
