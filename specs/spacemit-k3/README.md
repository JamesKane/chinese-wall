# SpacemiT K3 Functional Specification

Target: the **SpacemiT K3** RISC-V SoC on the **K3-CoM260** compute module.
The first board is the Banana Pi **BPI-SM10** developer kit. The SoC has
eight X100 harts in two four-core clusters. Per the devicetree ISA list,
they are RV64 with the H, V and AIA (`smaia`/`ssaia`) extensions. The CoM260
devicetree declares 16 GiB of RAM at `0x1_0000_0000`.

Each statement is tagged:

- **[HW]**: required by the hardware or firmware
- **[POLICY]**: a Linux design choice, informative only
- **[ABI]**: needed for compatibility with devicetree bindings or userspace

The devicetree source (`k3.dtsi`, `k3-com260.dtsi`, `k3-com260-ifx.dts`) is
dual-licensed GPL-2.0 OR MIT. The clock and reset binding headers are
GPL-2.0-only OR BSD-2-Clause. All of these can be used directly in the
clean room and are not restated here.

## Sections

| # | Section | Status |
|---|---------|--------|
| 01 | [Clock and reset controllers (APBS PLLs, MPMU, APBC, APMU, DCIU)](01-clocks-resets.md) | draft: PLL word decoding inferred (consistent with vendor firmware); C0/C1-to-cluster order unconfirmed |
| 02 | [Pin controller and GPIO](02-pinctrl-gpio.md) | not started |
| 03 | [SD/MMC host (SDHCI deltas from K1)](03-sdhci.md) | not started |
| 04 | [I2C (TWSI)](04-i2c.md) | not started |
| 05 | [P1 PMIC](05-pmic-p1.md) | not started |
| 06 | [Ethernet MAC integration](06-dwmac.md) | not started |
| 07 | [USB2 PHY and USB3/PCIe combo PHY](07-usb-phys.md) | not started |
| 08 | [USB controllers (DWC3 integration)](08-usb-dwc3.md) | not started |
| 09 | [SPI and QSPI](09-spi-qspi.md) | not started |
| 10 | [PWM](10-pwm.md) | not started |
| 11 | [Peripheral DMA](11-pdma.md) | not started |
| 12 | [I2S](12-i2s.md) | not started |
| 13 | [PCIe host controllers and PCIe PHY](13-pcie.md) | not started *(vendor)* |
| 14 | [Power domains](14-power-domains.md) | not started *(vendor)* |
| 15 | [Thermal sensor](15-thermal.md) | not started *(vendor)* |
| 16 | [Display (DPU, DSI, DisplayPort/eDP)](16-display.md) | not started *(vendor)* |
| 17 | [UFS host](17-ufs.md) | not started *(vendor)* |
| 18 | [Video codec (Linlon v5 delta)](18-vpu.md) | not started *(vendor)* |

Suggested order for bring-up: 01, 02, 03, 06, 04, 05, 07, 08, 13, then the rest.
Sections marked *(vendor)* have no mainline source at the baseline and are
extracted from the SpacemiT vendor kernel. Facts that come only from vendor
sources are tagged *(vendor)* inline.

## Sources the clean room may read directly

These are permissively licensed or public hardware documentation. A
clean-room implementer may consult them alongside these specs. Revisions
are pinned in `SOURCE-BASELINE.md`.

| Source | What it gives you | Exclusions |
|---|---|---|
| SpacemiT K3 user manual and datasheet (`github.com/spacemit-com/docs-chip`, `en/key_stone/k3`) | Address map, interrupt list, register descriptions for clock/reset, GPIO, I2C, UART, SPI, PWM, DMA, timers/watchdog, mailbox, Ethernet, USB, PCIe, display and more | Public documentation with no license file: use the facts, do not copy the text |
| SpacemiT UEFI firmware (`github.com/spacemit-com/edk2-platforms`, branch `k3`, `Silicon/Spacemit` and `Platform/Spacemit`) | BSD-2-Clause-Patent drivers for K3 clocks/PLLs, pinctrl, GPIO, SDHCI, I2C, QSPI, Ethernet (DW EQoS), USB host init, PCIe host bridge, UFS and DisplayPort; K3 ACPI tables (DSDT, MADT, RHCT, MCFG, PPTT, SPCR) | `Silicon/Spacemit/K3/Drivers/DwEqosDxe/GenphyDxeUtil.c` (says it contains Linux-derived code) and `Silicon/Spacemit/K3/Drivers/UfsHcDxe/` (pending review) |
| PowerVR Rogue kernel driver in the vendor kernel (`drivers/gpu/drm/img-rogue/`) | GPU kernel interface | Dual MIT/GPLv2, except `spacemit/spacemit_init.c` (no license notice) and `spacemit/devfreq_cooling.h` (GPL-2.0 only): do not read those two |
| Mainline Linux PowerVR DRM driver (`drivers/gpu/drm/imagination/`) | Upstream GPU kernel driver and firmware interface | None: every file is dual GPL-2.0/MIT |
| Vendor devicetrees (`arch/riscv/boot/dts/spacemit/k3*.dts*` in the vendor kernel) | Nodes for blocks mainline lacks; 16-hart CPU map; OPP tables; CoM260 carrier variants | GPL-2.0 OR MIT: usable directly. Bind drivers to mainline compatibles where they exist. |

The UEFI firmware also means the K3 can boot through **UEFI with ACPI** as
well as devicetree. The ACPI tables include an RHCT (RISC-V hart
capabilities) and an MCFG (PCIe ECAM).

## Work that needs no specification

These items come from public standards, permissively licensed sources, or
code already in FreeBSD:

| Item | Basis |
|---|---|
| Interrupt controller: IMSIC, and APLIC in MSI mode | Ratified RISC-V Advanced Interrupt Architecture specification. The K3 supervisor-level APLIC (`0xE0804000`, 512 sources) only delivers through the S-level IMSIC (`0xE0400000`). FreeBSD's APLIC driver supports only direct mode and refuses this configuration, so **no device interrupts work until this is implemented.** |
| ACLINT M/S software interrupts and machine timer | Public RISC-V ACLINT specification. A supervisor-mode OS uses SBI for its timer and IPIs. |
| CPU identification | The SpacemiT vendor ID is already in FreeBSD. An X100 `marchid` entry is needed; read it from hardware. |
| UART | ns16550-compatible with 32-bit register stride. FreeBSD matches the `intel,xscale-uart` fallback. |
| Ethernet MAC core, USB DWC3 core | Published Synopsys IP, already supported by FreeBSD drivers. Only SoC integration is specified (06, 08). |
| Devicetree | Import `k3-com260.dtsi` and `k3-com260-ifx.dts`. Refresh `spacemit,k3-clocks.h` (see 01). |

## Coverage beyond mainline

Mainline Linux at the baseline has no driver or devicetree node for K3 PCIe,
display, GPU, video codec, camera/ISP, UFS, thermal, power domains or
IOMMU. The SpacemiT vendor kernel covers all of them. Sections 13–17 take
the ones that matter for a usable board. The others:

| Block | Plan |
|---|---|
| GPU (Imagination PowerVR **BXM-4-64**, one cluster, core ID (BVNC) **36.56.104.183**; 512 KiB register window at `0xCAC00000`, interrupt 75) | See "GPU" below. No spec needed: both candidate kernel drivers are dual MIT/GPLv2. |
| IOMMU (`spacemit,k3-iommu` at `0xC0F00000`, interrupt 234) | A standard RISC-V IOMMU (public specification) with a small SoC glue layer. The vendor devicetree leaves it disabled and attaches no masters to it. Specify the glue only when DMA isolation is wanted. |
| Video codec (`arm china,linlon-v5` at `0xC0500000`, 2 MiB window, interrupt 74) | The same Linlon/MVE firmware-driven codec family as [`../linlon-vpu`](../linlon-vpu/), but an older generation (v5, where Sky1 has v8). Delta in [18](18-vpu.md). The vendor driver is GPL-only and needs a firmware blob. |
| Firmware services over RPMI (`riscv,sbi-mpxy-mbox`, `riscv,rpmi-regulator`, `-rtc`, `-pwrkey`) | Public RISC-V specifications (SBI MPXY extension, RPMI). On the vendor build, firmware owns some regulators, the RTC and the power key and exposes them through SBI. Mainline already has the MPXY mailbox client. |
| Mailboxes (`spacemit,k3-mailbox`, from `0xCAC90000`) and remote cores (`spacemit,k3-rproc`) | Inter-processor messaging to the real-time CPU subsystem. Only needed to run firmware on those cores. |
| 2D engine (`spacemit,v2d` at `0xC0100000`), camera (`spacemit,ccic`, CSI D-PHY), CAN (`spacemit,k1-flexcan` ×3 from `0xD4028000`), eFuse, hardware spinlock, EtherCAT master, eSPI | Later. Vendor userspace for the codec and camera (`k3x-vpu-*`, `k3x-cam`) is published separately. |
| Watchdog (`spacemit-k1,wdt` at `0xD4014000`), RTC (`mrvl,mmp-rtc` at `0xD4010000`) | Same IP as on K1 and older Marvell MMP parts. Check FreeBSD for existing MMP drivers before specifying. |

### GPU

[HW] The kernel driver and firmware must match the core ID: B=36, V=56,
N=104, C=183. The GPU sits in its own power domain (14) and takes one
clock and one reset from the APMU (01). The vendor OPP table runs it from
409 MHz upward at 0.8 V.

There are two routes, and the clean room may read the source for both:

| Route | Kernel side | Firmware and userspace | Gap |
|---|---|---|---|
| Upstream | Mainline PowerVR DRM driver (dual GPL-2.0/MIT). Mainline binds the K3 GPU as `spacemit,k3-gpu` + `img,img-bxm-4-64` + `img,img-rogue` (binding added Oct 2026), the same pairing as the T-Head TH1520. | Firmware is named after the core ID. Upstream linux-firmware carries 36.52.104.182 (TH1520) and 36.53.104.796, but I found no 36.56.104.183 blob. Mesa's PowerVR Vulkan driver needs a device-info entry for this core ID. | Mainline accepts only three core IDs and would reject 36.56.104.183. Mainline has no K3 GPU node yet. |
| Vendor | Imagination DDK 24.2 kernel module in the vendor kernel (dual MIT/GPLv2, two excluded files listed above). Binds `img,rgx`. | Closed userspace and firmware from SpacemiT's `img-gpu-powervr` packages. | The DDK ABI is private and changes with each release, and the closed userspace is built for Linux. |

[POLICY] The upstream route is the one to pursue: open userspace and a
stable ABI. On FreeBSD it would arrive through the ported Linux DRM
drivers (drm-kmod), not the base system. It is blocked until Imagination
publishes 36.56.104.183 firmware. Until then the vendor firmware blob is
the only image for this core, and whether it works with the upstream
kernel interface is not known.

> OPEN QUESTION: Whether the vendor DDK 24.2 firmware image is compatible
> with the firmware interface the mainline driver expects. Do not assume
> it is; the mainline driver checks the firmware version.
| A100 AI cores | RISC-V harts (RVA23 without H, plus the SpacemiT IME matrix extension) that the vendor devicetree lists as harts 8–15. Bringing them up is mainly a scheduler and ISA question rather than a device driver. Still unverified: whether firmware exposes them through SBI HSM. |

> OPEN QUESTION: Which devicetree matches the BPI-SM10 carrier. Candidates
> are mainline `k3-com260-ifx.dts` and the vendor `k3_com260_kit_v02.dts`
> ("CoM260 Kit V02"). Both use the same USB hub parts and RGMII PHY. Confirm
> against the kit's shipped devicetree or schematic before relying on
> carrier GPIO assignments (USB hub reset, VBUS enable, SD card detect,
> rfkill lines).
