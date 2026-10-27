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
| 01 | [Clock and reset controllers (APBS PLLs, MPMU, APBC, APMU, DCIU)](01-clocks-resets.md) | draft: PLL band-byte rule and constant fields unknown; C0/C1-to-hart order needs a hardware check |
| 02 | [Pin controller and GPIO](02-pinctrl-gpio.md) | draft: slew-rate bits disputed between sources; eMMC pads 153–163 vendor-only |
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
| 18 | [Video codec (delta against Sky1)](18-vpu.md) | not started *(vendor)*: identity and firmware known |

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
| SpacemiT OpenSBI (`github.com/spacemit-com/opensbi`, branch `k3-br-v1.0.y`) | Hart start (SBI HSM) for all 16 harts, RPMI services, K3 platform setup | None: BSD-2-Clause |
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
| Video codec (`arm china,linlon-v5` at `0xC0500000`, 2 MiB window, interrupt 74) | The same v52_v76 hardware family as Sky1's codec in [`../linlon-vpu`](../linlon-vpu/), despite the "v5" compatible string. Delta in [18](18-vpu.md). The vendor driver is GPL-only and needs SpacemiT's firmware. |
| Firmware services over RPMI (`riscv,sbi-mpxy-mbox`, `riscv,rpmi-regulator`, `-rtc`, `-pwrkey`) | Public RISC-V specifications (SBI MPXY extension, RPMI). On the vendor build, firmware owns some regulators, the RTC and the power key and exposes them through SBI. Mainline already has the MPXY mailbox client. |
| Mailboxes (`spacemit,k3-mailbox`, from `0xCAC90000`) and remote cores (`spacemit,k3-rproc`) | Inter-processor messaging to the real-time CPU subsystem. Only needed to run firmware on those cores. |
| 2D engine (`spacemit,v2d` at `0xC0100000`), camera (`spacemit,ccic`, CSI D-PHY), CAN (`spacemit,k1-flexcan` ×3 from `0xD4028000`), eFuse, hardware spinlock, EtherCAT master, eSPI | Later. Vendor userspace for the codec and camera (`k3x-vpu-*`, `k3x-cam`) is published separately. |
| Watchdog (`spacemit-k1,wdt` at `0xD4014000`), RTC (`mrvl,mmp-rtc` at `0xD4010000`) | Same IP as on K1 and older Marvell MMP parts. Check FreeBSD for existing MMP drivers before specifying. |
| A100 AI cores (harts 8–15, vendor devicetree only) | *(vendor)* Started like any other hart through SBI HSM. See "A100 harts" below. |

### GPU

[HW] The kernel driver and firmware must match the core ID: B=36, V=56,
N=104, C=183. The GPU sits in its own power domain (14) and takes one
clock and one reset from the APMU (01). The vendor OPP table runs it from
409 MHz upward at 0.8 V.

There are two routes, and the clean room may read the kernel source for
both:

| | Upstream route | Vendor route |
|---|---|---|
| Kernel driver | Mainline PowerVR DRM driver (dual GPL-2.0/MIT). Mainline binds `spacemit,k3-gpu` + `img,img-bxm-4-64` + `img,img-rogue` (binding added Oct 2026). | Imagination DDK 24.2 kernel module (dual MIT/GPLv2, two excluded files listed above). Binds `img,rgx`. |
| Core ID support | Not on the driver's supported list yet; mainline probes it only with its "experimental hardware" option turned on. No K3 GPU devicetree node yet. | Built for this core. |
| Firmware | `powervr/rogue_36.56.104.183_v1.fw`, version 1.1, redistributable, in Imagination's staging linux-firmware repository (`gitlab.freedesktop.org/imagination/linux-firmware`, branch `powervr`). Not yet in kernel.org linux-firmware. | `rgx.fw.36.56.104.183` and `rgx.sh.36.56.104.183` from SpacemiT's `img-gpu-powervr` packages; no stated license. |
| Userspace | Upstream Mesa has a device-info entry for this core. It is not on Mesa's conformant list, so the Vulkan driver warns. | Closed, built for Linux and DDK 24.2. |

**[HW]** The two firmware images are not interchangeable. The mainline
driver reads an information block at the end of the image and requires
format version 3, the open-source flag set, and firmware major version 1.
The vendor image uses format version 2, has the open-source flag clear,
and is tied to DDK 24.2. Each driver rejects the other's image.

[POLICY] The upstream route is the one to pursue: open userspace, a stable
interface and a redistributable firmware. On FreeBSD it would come through
the ported Linux DRM drivers (drm-kmod), not the base system. The missing
pieces are a supported-list entry for 36.56.104.183, a K3 GPU devicetree
node, and the firmware reaching kernel.org.

### A100 harts

*(vendor)* SpacemiT's OpenSBI (BSD-2-Clause) powers up harts 8–15 through
the standard SBI HSM hart-start call. It programs the reset vectors for
clusters 2 and 3 and sets some A100-only cache and prefetch controls. The
vendor kernel then runs them as ordinary SMP CPUs, but a vendor scheduler
patch keeps threads off them unless a thread is explicitly marked.

**[HW]** An OS that starts these harts must handle two differences from
the X100 harts:

| | X100 (harts 0–7) | A100 (harts 8–15) |
|---|---|---|
| Hypervisor extension (H and its Sh* extensions) | yes | no |
| Vector register length | 256 bits (1 KiB of vector state) | 1024 bits (4 KiB of vector state) |

Vector state must be sized for the larger hart, and a thread using vectors
cannot simply migrate between the two hart types. [POLICY] The simplest
course is to leave harts 8–15 stopped.

## Board: Banana Pi BPI-SM10

The kit is a CoM260 module on SpacemiT's "K3-COM260 KIT" carrier
(schematic revision V2.0, March 2026). The matching devicetree is
**`k3-com260-ifx.dts`** (mainline) / `k3_com260_ifx.dts` (vendor), not the
vendor `k3_com260_kit_v02.dts`, which enables CAN and UART ports the carrier
does not wire. The module's EEPROM holds a `product_name` of
`k3_com260_ifx`, and the shipped U-Boot loads `spacemit/<product_name>.dtb`.

| Item | Detail |
|---|---|
| Boot chain | Boot ROM → SPL → OpenSBI (SBI v2.0) plus a firmware image that provides RPMI services → U-Boot. All in an 8 MiB SPI NOR. No UEFI firmware on shipped boards; "UEFI" OS images boot through U-Boot's EFI support. |
| Console | UART0 (`0xD4017000`), 115200 8N1, 3.3 V on the carrier's 12-pin button header (pin 3 RX, pin 4 TX). Carrier V2.0 swapped TX and RX relative to V1.0. Another pin on that header forces recovery (flashing) mode. |
| Memory | 8, 16 or 32 GB LPDDR5 depending on the module. Mainline's devicetree hard-codes 16 GiB, so take the size from firmware. |
| Storage | 128 GB UFS on the module (17); SD card slot on the module, card detect on GPIO 4 |
| Ethernet | One GbE port: the module's RTL8211F PHY (RGMII, MDIO address 1) on the MAC at `0xCAC82000` |
| USB | VL817 USB 3 hub on USB3 port B (shares a lane with PCIe), reset on GPIO 53, port power on GPIO 67; USB-C dual-role port with an FUSB301 at I2C0 `0x25` |
| PCIe | M.2 M-key 2280 (Gen3 x4), M.2 M-key 2230 (x1), M.2 E-key (x1, plus USB2, SDIO, UART6, I2S3) |
| Wi-Fi/BT | RTL8852BE card in the E-key slot; Wi-Fi disable on GPIO 73, Bluetooth enable on GPIO 57 |
| Display, camera | DisplayPort 1.2 (DP1), 2-lane MIPI DSI header, two 22-pin CSI connectors |
| Other | Carrier EEPROM at I2C1 `0x50`; one CAN-FD transceiver on CAN2; 4-pin fan header; a power-sequencing MCU not visible to the OS |

Mainline's devicetree lacks PCIe, UFS, display, USB-C, CAN and camera
nodes. It describes the P1 PMIC directly, where the vendor build drives
the regulators through firmware (RPMI).

> OPEN QUESTION: Which SoC pins carry the fan PWM and tachometer, and
> whether carrier V1.0 units carry a different `product_name`.
