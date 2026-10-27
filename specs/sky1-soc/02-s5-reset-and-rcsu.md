# 02: Reset Controllers, ACPI Resource Tables and Subsystem RCSUs

## 2.1 Reset controllers

| Controller | ACPI `_HID` | DT compatible (mainline / vendor) | Base | Size | ACPI name in RSTL |
|---|---|---|---|---|---|
| S5 (always-on) system controller | `CIXHA020` | `cix,sky1-s5-system-control` / `cix,sky1-src` | `0x16000000` | 4 KiB | `RST0` |
| FCH (peripheral) system controller | `CIXHA021` | `cix,sky1-system-control` / `cix,sky1-src-fch` | `0x04160000` | 256 bytes (mainline DT) | `RST1` |

- **[HW]** Both are plain MMIO register banks. Every reset line is one bit.
  **Active-low**: bit = 0 holds the block in reset, bit = 1 releases it.
  Reading the bit gives the state (0 = in reset).
- **[HW]** Use read-modify-write; other bits in the same word belong to
  other blocks.
- **[POLICY]** Linux waits 50–100 µs after an assert and again after a
  deassert when it does a full reset pulse. A plain assert or deassert has
  no wait in the vendor driver; the mainline driver waits in both.
- **[HW]** In mainline DT the S5 bank is also a syscon used by other
  drivers. Under ACPI the reset driver maps the `_CRS` range itself.
- The reset **index** numbers are the same in DT and in ACPI RSTL tables.
  They are the values in the public dual-licensed headers
  `cix,sky1-s5-system-control.h` (S5) and `cix,sky1-system-control.h`
  (FCH).

### 2.1.1 S5 controller register words

| Offset | Word | Lines |
|---|---|---|
| `0x304` | CSU PM reset | bit 0 |
| `0x308` | Sensor-fusion (sensor hub) reset | bit 0 |
| `0x30C` | Sensor-fusion NoC reset | bit 0 |
| `0x400` | Group 0, S0 domain word 0 | 27 lines (DDR controller, GIC, CI-700, NI-700s, …, DPU, DP) |
| `0x404` | Group 0, S0 domain word 1 | 21 lines |
| `0x408` | Group 1, USB PHYs | 20 lines |
| `0x40C` | Group 1, USB controllers | 20 lines |
| `0x800` | Group 0, RCSU resets | 32 lines (one per subsystem RCSU) |
| `0x804` | Group 1, RCSU resets | 20 lines |

The full index → (word, bit) map is in Annex A.

### 2.1.2 FCH controller register words

| Index | Line | Offset | Bit |
|---|---|---|---|
| 0 | I3C0 functional (G) | `0x008` | 0 |
| 1 | I3C0 functional (I) | `0x008` | 1 |
| 2 | I3C1 functional (G) | `0x008` | 2 |
| 3 | I3C1 functional (I) | `0x008` | 3 |
| 4–7 | UART0–3 functional | `0x008` | 4–7 |
| 8 | GPT timer functional | `0x008` | 20 |
| 9 | I3C0 APB | `0x00C` | 0 |
| 10 | I3C1 APB | `0x00C` | 1 |
| 11 | FCH DMA AXI | `0x00C` | 2 |
| 12–15 | UART0–3 APB | `0x00C` | 4–7 |
| 16, 17 | SPI0, SPI1 APB | `0x00C` | 8, 9 |
| 18–25 | I2C0–7 APB | `0x00C` | 12–19 |
| 26 | GPIO APB | `0x00C` | 21 |
| 27 | XSPI register | `0x010` | 0 |
| 28 | XSPI system | `0x010` | 1 |

## 2.2 RSTL: ACPI reset lookup table

**[HW]** A device that needs reset lines has a named object `RSTL` that
evaluates to a package of entries, each a package of 3 or 4 elements:

| Element | Type | Meaning |
|---|---|---|
| 0 | Reference or path string | Reset controller device (`RST0` = `CIXHA020`, `RST1` = `CIXHA021`) |
| 1 | Integer | Reset index (§2.1) |
| 2 | Reference or path string | Consumer device |
| 3 | String (optional) | Reset name (for example `"i2c_reset"`, `"apb_reset"`, `"gmac_rstn"`) |

Examples (public ACPI-to-DT mapping notes): I2C uses `RST1` index 18
(`i2c_reset`); GPIO uses `RST1` index 26 (`apb_reset`); a GMAC uses `RST0`
index 44.

- **[HW]** RSTL lines are only a description. The firmware may leave a
  block in reset; the OS deasserts what it needs.
- **[HW]** RSTL, RSNL and DLKL objects can be attached to any device, not
  only their consumers. Process them for every present device.

## 2.3 RSNL and DLKL

### 2.3.1 RSNL: resource names

**[HW]** Package of 4-element entries: (device, resource type, index,
name). It gives a name to resource `index` of the given type in that
device's `_CRS`, matching the names DT bindings use with `reg-names` /
`interrupt-names` (for example `"rcsu"`, `"dp_phy"`).

> OPEN QUESTION: The resource-type integer uses Linux's resource-type flag
> values in the Sky1-Linux driver (`0x200` memory, `0x400` IRQ). Whether
> the DSDT really encodes Linux flag values or ACPI resource types should
> be checked against the DSDT.

If a named resource is an interrupt marked disabled, Sky1-Linux re-reads it
from `_CRS`.

### 2.3.2 DLKL: device links

**[HW]** Package of 3-element entries: (supplier device, consumer device,
flags). The consumer must not be started before the supplier.

> OPEN QUESTION: The flags integer is passed through as Linux device-link
> flags in the Sky1-Linux driver. Treat it as "ordering only" in another
> OS.

### 2.3.3 RSVL: reserved memory

**[HW]** *(vendor)* A further CIX table on `CIXA1019`, used for the HDA
controller's reserved DMA memory: entries (base, size, flags, consumer
device). It is the ACPI stand-in for a DT `memory-region`. See the audio
spec (sky1-audio 02) for the known entry and open questions on the flags.

### 2.3.4 `CIXA1019`

- **[HW]** The ACPI device whose presence means "process the CIX lookup
  tables". (Note the HID is `CIXA1019`, not `CIXHA019`, which is an
  unrelated device.) It has no registers.

## 2.4 RCSU: per-subsystem control block

Most Sky1 subsystems have an **RCSU**: a small register bank, separate from
the IP block, that holds the subsystem's clock-gating control, power-switch
straps, harvesting fuses, address remaps and bus-timeout settings. Each RCSU
also has a reset line in the S5 controller words `0x800`/`0x804`.

> OPEN QUESTION: "RCSU" is not expanded in any source. Probably "reset,
> clock and system unit" or similar.

A common layout appears across subsystems:

| Offset | Name | Seen in | Meaning |
|---|---|---|---|
| `0x218` | `PGCTRL` | GPU | `[0]` Q-channel clock gating enable |
| `0x21C` | `PGCTRL` | VPU | `[0]` Q-channel clock gating enable |
| `0x300` | `STRAP_PIN0` | VPU, PCIe | Straps / power-switch controls (VPU: §2.4.3) |
| `0x304` | strap / harvesting | GPU | Shader-core harvesting fuses (§2.4.1) |
| `0x308` | `STRAP_PIN2` | VPU | Core harvesting fuses (§2.4.3) |
| `0x300 + 4n` | strap words | PCIe | Lane / mode straps |
| `0x400 + 4n` | status words | PCIe | Controller status |

### 2.4.1 GPU RCSU

- **[HW]** The GPU's ACPI/DT resource 0 is its RCSU and resource 1 the GPU
  register block.
- **[HW]** Power-up order (vendor-derived): power domain on → clocks on →
  GPU reset asserted → GPU reset released → set `PGCTRL` (`0x218`) bit 0.
  Only then touch GPU registers.
- **[HW]** Harvesting: let v = RCSU word `0x304`. The usable shader cores
  are `~((v & 0x0FFFFFF0) >> 4) & 0x550555`, ANDed into the GPU's own
  shader-present mask. (The constant `0x550555` is the shader-present
  pattern of the full part.)

Details belong to the Panthor spec, section 11.

### 2.4.2 Audio RCSU (`0x07000000`, 64 KiB)

| Offset | Value written | Meaning |
|---|---|---|
| `0x34` | `0x20000000` | Address remap for the DSP's view of system memory. Must be restored after every audio power-up |
| `0x1000` | `0x80000078` | Bus timeout: `[31]` enable, `[15:0]` timeout count 0x78 |

- **[HW]** Order: audio power domain on, audio NoC reset pulse (assert,
  ≥ 1 µs, deassert), then the two RCSU writes, then audio clock-controller
  registers.

### 2.4.3 VPU RCSU

| Offset | Field | Bits | Meaning |
|---|---|---|---|
| `0x21C` (`PGCTRL`) | Q-channel clock gating | `[0]` | 1 = enable |
| `0x300` (`STRAP_PIN0`) | disable AFBC / RealVideo / VPx / HEVC | `[0]` / `[1]` / `[2]` / `[3]` | Feature-disable straps |
| | power-switch "on" timing | `[11:4]` | |
| | power-switch acknowledge/isolation timing | `[19:12]` | |
| | power-switch "off" timing | `[27:20]` | |
| | per-core automatic power-off enable | `[31:28]` | One bit per VPU core |
| `0x308` (`STRAP_PIN2`) | core harvesting | `[7:4]` | 1 = core fused off. Usable cores = inverted |

Details belong to the VPU spec (Priority 4).

### 2.4.4 DisplayPort and DPU RCSUs

The DP controller's RCSU is its fourth register window (`dp_rcsu`, node
base − `0x10000`, 0x304 or 0x320 bytes). It is described with the DP
transmitter (planned spec `sky1-dp-tx`). The DPU RCSU has a reset line
(`SKY1_DPUn_RCSU_RESET_N`) but no register use is known.

### 2.4.5 USB RCSU

- **[HW]** The USB RCSU resets are released by default at boot.

## 2.5 DMRP and memory repair

**[HW]** Some subsystems need their SRAMs **repaired** (fuse-driven
redundancy loaded into the memories) each time their power domain turns
on. On Sky1 under ACPI this is done by firmware: the audio subsystem's ACPI
**power resource** `_ON` method enables the audio power gates (RCSU
`PD_CTRL`) and triggers repair through a block the firmware calls
**DMRP**.

- **[HW]** An OS must put the device's ACPI power resource in D0 before
  touching any register of that subsystem (for audio: before the audio
  clock controller, reset controller, DMA or I2S).

> OPEN QUESTION: DMRP's registers, the `PD_CTRL` offset and the repair
> handshake are only in the DSDT AML (not available to this extraction).
> Under DT boot the power-domain firmware (SCMI, section 01 §1.3) is
> presumed to do the same internally. A DSDT disassembly would be needed
> to specify DMRP.

## Annex A: S5 reset index map

Index = value in `cix,sky1-s5-system-control.h` and in RSTL entries that
reference `RST0`. All lines are active-low.

| Idx | Line | Word | Bit |
|---|---|---|---|
| 0 | `CSU_PM` | `0x304` | 0 |
| 1 | `SENSORFUSION` | `0x308` | 0 |
| 2 | `SENSORFUSION_NOC` | `0x30C` | 0 |
| 3 | `DDRC` | `0x400` | 0 |
| 4 | `GIC` | `0x400` | 1 |
| 5 | `CI700` | `0x400` | 2 |
| 6 | `SYS_NI700` | `0x400` | 3 |
| 7 | `MM_NI700` | `0x400` | 4 |
| 8 | `PCIE_NI700` | `0x400` | 5 |
| 9 | `GPU` | `0x400` | 6 |
| 10 | `NPUTOP` | `0x400` | 7 |
| 11 | `NPUCORE0` | `0x400` | 8 |
| 12 | `NPUCORE1` | `0x400` | 9 |
| 13 | `NPUCORE2` | `0x400` | 10 |
| 14 | `VPU` | `0x400` | 11 |
| 15 | `ISP_SRESET` | `0x400` | 12 |
| 16 | `ISP_ARESET` | `0x400` | 13 |
| 17 | `ISP_HRESET` | `0x400` | 14 |
| 18 | `ISP_GDCRESET` | `0x400` | 15 |
| 19 | `DPU_RESET0` | `0x400` | 16 |
| 20 | `DPU_RESET1` | `0x400` | 17 |
| 21 | `DPU_RESET2` | `0x400` | 18 |
| 22 | `DPU_RESET3` | `0x400` | 19 |
| 23 | `DPU_RESET4` | `0x400` | 20 |
| 24 | `DP_RESET0` | `0x400` | 21 |
| 25 | `DP_RESET1` | `0x400` | 22 |
| 26 | `DP_RESET2` | `0x400` | 23 |
| 27 | `DP_RESET3` | `0x400` | 24 |
| 28 | `DP_RESET4` | `0x400` | 25 |
| 29 | `DP_PHY` | `0x400` | 26 |
| 30 | `AUDIO_HIFI5` | `0x404` | 0 |
| 31 | `AUDIO_HIFI5_NOC` | `0x404` | 1 |
| 32 | `CSIDPHY_PRST0` | `0x404` | 2 |
| 33 | `CSIDPHY_CMNRST0` | `0x404` | 3 |
| 34 | `CSI0` | `0x404` | 4 |
| 35 | `CSIDPHY_PRST1` | `0x404` | 5 |
| 36 | `CSIDPHY_CMNRST1` | `0x404` | 6 |
| 37 | `CSI1` | `0x404` | 7 |
| 38 | `CSI2` | `0x404` | 8 |
| 39 | `CSI3` | `0x404` | 9 |
| 40 | `CSIBRDGE0` | `0x404` | 10 |
| 41 | `CSIBRDGE1` | `0x404` | 11 |
| 42 | `CSIBRDGE2` | `0x404` | 12 |
| 43 | `CSIBRDGE3` | `0x404` | 13 |
| 44 | `GMAC0` | `0x404` | 14 |
| 45 | `GMAC1` | `0x404` | 15 |
| 46 | `PCIE0` | `0x404` | 16 |
| 47 | `PCIE1` | `0x404` | 17 |
| 48 | `PCIE2` | `0x404` | 18 |
| 49 | `PCIE3` | `0x404` | 19 |
| 50 | `PCIE4` | `0x404` | 20 |
| 51 | `USB_DP_PHY0_PRST` | `0x408` | 0 |
| 52 | `USB_DP_PHY1_PRST` | `0x408` | 1 |
| 53 | `USB_DP_PHY2_PRST` | `0x408` | 2 |
| 54 | `USB_DP_PHY3_PRST` | `0x408` | 3 |
| 55 | `USB_DP_PHY0` | `0x408` | 4 |
| 56 | `USB_DP_PHY1` | `0x408` | 5 |
| 57 | `USB_DP_PHY2` | `0x408` | 6 |
| 58 | `USB_DP_PHY3` | `0x408` | 7 |
| 59 | `USBPHY_SS_PST` | `0x408` | 8 |
| 60 | `USBPHY_SS` | `0x408` | 9 |
| 61 | `USBPHY_HS0_PRST` | `0x408` | 10 |
| 62 | `USBPHY_HS1_PRST` | `0x408` | 11 |
| 63 | `USBPHY_HS2_PRST` | `0x408` | 12 |
| 64 | `USBPHY_HS3_PRST` | `0x408` | 13 |
| 65 | `USBPHY_HS4_PRST` | `0x408` | 14 |
| 66 | `USBPHY_HS5_PRST` | `0x408` | 15 |
| 67 | `USBPHY_HS6_PRST` | `0x408` | 16 |
| 68 | `USBPHY_HS7_PRST` | `0x408` | 17 |
| 69 | `USBPHY_HS8_PRST` | `0x408` | 18 |
| 70 | `USBPHY_HS9_PRST` | `0x408` | 19 |
| 71 | `USBC_SS0_PRST` | `0x40C` | 0 |
| 72 | `USBC_SS1_PRST` | `0x40C` | 1 |
| 73 | `USBC_SS2_PRST` | `0x40C` | 2 |
| 74 | `USBC_SS3_PRST` | `0x40C` | 3 |
| 75 | `USBC_SS4_PRST` | `0x40C` | 4 |
| 76 | `USBC_SS5_PRST` | `0x40C` | 5 |
| 77 | `USBC_SS0` | `0x40C` | 6 |
| 78 | `USBC_SS1` | `0x40C` | 7 |
| 79 | `USBC_SS2` | `0x40C` | 8 |
| 80 | `USBC_SS3` | `0x40C` | 9 |
| 81 | `USBC_SS4` | `0x40C` | 10 |
| 82 | `USBC_SS5` | `0x40C` | 11 |
| 83 | `USBC_HS0_PRST` | `0x40C` | 12 |
| 84 | `USBC_HS1_PRST` | `0x40C` | 13 |
| 85 | `USBC_HS2_PRST` | `0x40C` | 14 |
| 86 | `USBC_HS3_PRST` | `0x40C` | 15 |
| 87 | `USBC_HS0` | `0x40C` | 16 |
| 88 | `USBC_HS1` | `0x40C` | 17 |
| 89 | `USBC_HS2` | `0x40C` | 18 |
| 90 | `USBC_HS3` | `0x40C` | 19 |
| 91 | `AUDIO_RCSU` | `0x800` | 0 |
| 92 | `CI700_RCSU` | `0x800` | 1 |
| 93 | `CSI_RCSU0` | `0x800` | 2 |
| 94 | `CSI_RCSU1` | `0x800` | 3 |
| 95 | `CSU_PM_RCSU` | `0x800` | 4 |
| 96 | `DDR_BROADCAST_RCSU` | `0x800` | 5 |
| 97 | `DDR_CTRL_RCSU_0` | `0x800` | 6 |
| 98 | `DDR_CTRL_RCSU_1` | `0x800` | 7 |
| 99 | `DDR_CTRL_RCSU_2` | `0x800` | 8 |
| 100 | `DDR_CTRL_RCSU_3` | `0x800` | 9 |
| 101 | `DDR_TZC400_RCSU_0` | `0x800` | 10 |
| 102 | `DDR_TZC400_RCSU_1` | `0x800` | 11 |
| 103 | `DDR_TZC400_RCSU_2` | `0x800` | 12 |
| 104 | `DDR_TZC400_RCSU_3` | `0x800` | 13 |
| 105 | `DP0_RCSU` | `0x800` | 14 |
| 106 | `DP1_RCSU` | `0x800` | 15 |
| 107 | `DP2_RCSU` | `0x800` | 16 |
| 108 | `DP3_RCSU` | `0x800` | 17 |
| 109 | `DP4_RCSU` | `0x800` | 18 |
| 110 | `DPU0_RCSU` | `0x800` | 19 |
| 111 | `DPU1_RCSU` | `0x800` | 20 |
| 112 | `DPU2_RCSU` | `0x800` | 21 |
| 113 | `DPU3_RCSU` | `0x800` | 22 |
| 114 | `DPU4_RCSU` | `0x800` | 23 |
| 115 | `DSU_RCSU` | `0x800` | 24 |
| 116 | `FCH_RCSU` | `0x800` | 25 |
| 117 | `GICD_RCSU` | `0x800` | 26 |
| 118 | `GMAC_RCSU` | `0x800` | 27 |
| 119 | `GPU_RCSU` | `0x800` | 28 |
| 120 | `ISP_RCSU0` | `0x800` | 29 |
| 121 | `ISP_RCSU1` | `0x800` | 30 |
| 122 | `NI700_MMHUB_RCSU` | `0x800` | 31 |
| 123 | `NPU_RCSU` | `0x804` | 0 |
| 124 | `NI700_PCIE_RCSU` | `0x804` | 1 |
| 125 | `PCIE_X421_RCSU` | `0x804` | 2 |
| 126 | `PCIE_X8_RCSU` | `0x804` | 3 |
| 127 | `SF_RCSU` | `0x804` | 4 |
| 128 | `RCSU_SMMU_MMHUB` | `0x804` | 5 |
| 129 | `RCSU_SMMU_PCIEHUB` | `0x804` | 6 |
| 130 | `RCSU_SYSHUB` | `0x804` | 7 |
| 131 | `NI700_SMN_RCSU` | `0x804` | 8 |
| 132 | `NI700_SYSHUB_RCSU` | `0x804` | 9 |
| 133 | `RCSU_USB2_HOST0` | `0x804` | 10 |
| 134 | `RCSU_USB2_HOST1` | `0x804` | 11 |
| 135 | `RCSU_USB2_HOST2` | `0x804` | 12 |
| 136 | `RCSU_USB2_HOST3` | `0x804` | 13 |
| 137 | `RCSU_USB3_TYPEA_DRD` | `0x804` | 14 |
| 138 | `RCSU_USB3_TYPEC_DRD` | `0x804` | 15 |
| 139 | `RCSU_USB3_TYPEC_HOST0` | `0x804` | 16 |
| 140 | `RCSU_USB3_TYPEC_HOST1` | `0x804` | 17 |
| 141 | `RCSU_USB3_TYPEC_HOST2` | `0x804` | 18 |
| 142 | `VPU_RCSU` | `0x804` | 19 |
