# 05: Pin Controllers (Pad Multiplexing and Configuration)

Sky1 has two pin controllers. Each one is a flat array of per-pad
registers that select the pad's function and set its electrical
configuration. There are no group registers: a "group" is only a list of
pads that firmware or the OS programs together.

## 5.1 Instances

| Controller | Base | Size | Pads | ACPI `_HID` | DT compatible (mainline / vendor) | Power domain |
|---|---|---|---|---|---|---|
| FCH (S0) | `0x04170000` | 4 KiB | 138 (0–137) | `CIXHA016` | `cix,sky1-pinctrl` / `cix,sky1-iomuxc` | FCH, lost in system suspend |
| S5 | `0x16007000` | 4 KiB | 66 (0–65) | `CIXHA017` | `cix,sky1-pinctrl-s5` / `cix,sky1-iomuxc-s5` | Always-on |

- **[HW]** No interrupt, clock or reset is described for either
  controller.
- **[ABI]** The ACPI device name of the FCH controller is `MUX0` (consumers
  name `"\_SB.MUX0"` as the resource source).
- **[ABI]** The public ACPI notes and their conversion tool list `CIXH1006`
  and `CIXH1007` as the pin controllers. That is wrong: driver match
  tables give `CIXH1006` to a DMA controller and `CIXH1007` to the GPT
  timer. The pin controllers are `CIXHA016` and `CIXHA017`.

> OPEN QUESTION: The ACPI name of the S5 pin controller is not in the
> public material (probably `MUX1`).

## 5.2 Pad register

**[HW]** One 32-bit register per pad, at offset pad number × 4.

| Bits | Field | Meaning |
|---|---|---|
| 31:9 | – | Unknown. Preserve on write |
| 8:7 | `FUNC` | Function (mux mode) 0–3. The meaning of each mode per pad is in Annex A |
| 6 | `PU` | 1 = pull-up enabled |
| 5 | `PD` | 1 = pull-down enabled |
| 4 | `ST` | 1 = Schmitt-trigger input enabled *(vendor)* |
| 3:0 | `DS` | Drive strength code (§5.2.1) |

- **[HW]** `PU` = `PD` = 0 means no bias. Do not set both.
- **[HW]** Mux and configuration can be written separately with
  read-modify-write. *(vendor)* The vendor driver writes `FUNC` when a
  group is selected and bits 6:0 when the configuration is applied.
- **[HW]** A pad with no mux options (Annex A, "no mux") has a fixed
  function. Its register may still hold configuration bits, but nothing in
  the sources writes them.

> OPEN QUESTION: Bit 4 is used as Schmitt-trigger enable by the vendor
> devicetrees (`ST` flag) but the mainline driver and binding do not
> mention it. Mainline never sets it. Assume "input Schmitt trigger"; keep
> firmware's value if unsure.

### 5.2.1 Drive strength codes

**[HW]** The value is the typical current with the output high (public
devicetree binding):

| Code | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| mA | 2 | 3 | 5 | 6 | 8 | 9 | 11 | 12 | 13 | 14 | 17 | 18 | 20 | 21 | 23 | 24 |

- **[POLICY]** Mainline Linux uses code 4 (8 mA) when asked for a current
  not in the table.
- **[POLICY]** *(vendor)* The vendor devicetrees name the codes
  `DS_LEVEL1`…`DS_LEVEL15`, the level number being the code. Typical
  choices: 4 (8 mA) for plain GPIOs, 7 (12 mA) for I²C0/1/4–7 and
  UARTs, 12 (20 mA) for I²C2/3, SPI, XSPI, HDA, I²S and Ethernet.

## 5.3 Pad numbering and naming

- **[HW]** Pads are numbered from 0 separately in each controller. The
  number is the register index. Annex A gives every pad, its offset, its
  name and its four modes.
- **[HW]** Pad names follow the mode-0 function for most pads (for example
  `I2C0_CLK`, `UART2_TXD`). Pads named `GPIOnnn` have the GPIO as mode 0.
  Many named pads offer a `GPIOnnn` function in mode 1 or 2.
- **[HW]** `GPIO001`–`GPIO042` are S5 pads; `GPIO043`–`GPIO153` are FCH
  pads. For how `GPIOnnn` numbers relate to GPIO controller lines, see
  01-gpio §1.6.1.
- **[HW]** Some pads are on the "wrong" controller compared to the block
  they serve. The SoC SPI0 controller (in the FCH) uses S5 pads
  `SPI1_*`; the XSPI uses S5 pads `SE_QSPI_*`; the SPI1 controller uses
  FCH pads `GPIO138`–`GPIO142` in mode 2 (`SPI2_*`). See 03-spi-qspi §3.2.

### 5.3.1 Devicetree encodings

| Source | Encoding |
|---|---|
| Mainline binding | `pinmux` cells of (pad number << 8) \| mode, plus generic `bias-*` and `drive-strength` (mA) properties |
| Vendor devicetree | `sky1,pins` triples: (register offset, mode, configuration bits 6:0) |

**[HW]** In both, the register written is base + 4 × pad, with `FUNC` =
mode and, for the vendor form, bits 6:0 = the configuration word.

## 5.4 Groups used by ACPI firmware

### 5.4.1 How ACPI refers to pins

- **[ABI]** A consumer's `_CRS` contains a `PinGroupFunction` resource:
  shared/exclusive flag, function number (0 in the published example),
  resource source `"\_SB.MUX0"`, resource source index 0, and the
  **group name** (for example `"pinctrl_fch_i2c0"`).
- **[ABI]** The group names are the vendor devicetree labels. The
  definitions (`PinGroup` resources in the pin controller's `_CRS`, with
  pad lists and possibly vendor data) were not published.
- **[POLICY]** *(vendor)* Sky1-Linux under ACPI does **not** read
  `PinGroup`, `PinGroupFunction`, `PinFunction` or `PinConfig` at all.
  Firmware has already set every pad. The OS driver registers the
  controller with no groups so that consumers can probe. A clean-room OS
  can do the same: treat `PinGroupFunction` as documentation of which pads
  a device owns.

> OPEN QUESTION: The format of any vendor data inside the DSDT `PinGroup`
> resources is unknown. If a clean-room OS wants to re-apply pin settings
> (for example after FCH power loss, §5.5) it needs either that data or
> the group table below.

### 5.4.2 Group contents (vendor devicetree)

**[HW]** *(vendor)* Pads, modes and configuration of the groups named by
ACPI consumers. "Ctl" is the controller; mode numbers refer to Annex A;
DS is the drive code of §5.2.1.

| Group | Ctl | Pads (mode) | Pull | ST | DS |
|---|---|---|---|---|---|
| `pinctrl_fch_i2c0` | FCH | 30 `I2C0_CLK`, 31 `I2C0_SDA` (0) | up | – | 7 |
| `pinctrl_fch_i2c1` | FCH | 32 `I2C1_CLK`, 33 `I2C1_SDA` (0) | up | – | 7 |
| `pinctrl_fch_i2c2` | FCH | 34 `I2C2_SCL`, 35 `I2C2_SDA` (0) | up | yes | 12 |
| `pinctrl_fch_i2c3` | FCH | 37 `I2C3_CLK`, 38 `I2C3_SDA` (0) | up | yes | 12 |
| `pinctrl_fch_i2c4` | FCH | 40 `I2C4_CLK`, 41 `I2C4_SDA` (0) | up | – | 7 |
| `pinctrl_fch_i2c5` | FCH | 26 `I2C5_SCL`, 27 `I2C5_SDA` (0) | up | – | 7 |
| `pinctrl_fch_i2c6` | FCH | 28 `I2C6_SCL`, 29 `I2C6_SDA` (0) | up | – | 7 |
| `pinctrl_fch_i2c7` | FCH | 21 `I2C7_SCL`, 22 `I2C7_SDA` (0) | up | – | 7 |
| `pinctrl_fch_i3c0` | FCH | 34, 35 (1: `I3C0_SCL/SDA`); 36 `GPIO057` (1: `I3C0_PUR_EN_L`) | up | yes | 12 |
| `pinctrl_fch_i3c1` | FCH | 37, 38 (1: `I3C1_CLK/SDA`); 39 `GPIO060` (1: `I3C1_PUR_EN_L`) | up | yes | 12 |
| `pinctrl_fch_spi0` | S5 | 42 `SPI1_MISO`, 43 `SPI1_CS0`, 44 `SPI1_CS1`, 45 `SPI1_MOSI`, 46 `SPI1_CLK` (0) | up (clock: none) | yes | 12 |
| `pinctrl_fch_spi1` | FCH | 122–126 `GPIO138`–`GPIO142` (2: `SPI2_MISO/CS0/CS1/MOSI/CLK`) | up (clock: none) | yes | 12 |
| `pinctrl_fch_xspi` | S5 | 60–65 `SE_QSPI_CLK/CS_L/DATA0–3` (1: `QSPI_*`) | up | yes | 12 |
| `pinctrl_fch_uart0` | FCH | 79–82 `UART0_TXD/RXD/CTS/RTS` (0) | up | yes | 7 |
| `pinctrl_fch_uart1` | FCH | 83–86 `UART1_TXD/RXD/CTS/RTS` (0) | up | yes | 7 |
| `pinctrl_fch_uart2` | FCH | 87, 88 `UART2_TXD/RXD` (0) | up | – | 7 |
| `pinctrl_fch_uart3` | FCH | 89, 90 `UART3_TXD/RXD` (0) | up | – | 7 |
| `pinctrl_fch_pwm0` | FCH | 20 `DP2_VARY_BL` (0) | none | – | 7 |
| `pinctrl_fch_pwm1` | FCH | 79 `UART0_TXD` (1: `PWM0`) | down | yes | 7 |
| `pinctrl_edp0` | FCH | 18 `DP2_DIGON`, 19 `DP2_BLON` (0) | down | – | 4 |
| `pinctrl_hda` | FCH | 42–48 `HDA_*` (0) | mixed (SDOUT0, SYNC up; others down) | yes | 12 |
| `pinctrl_usb0`…`usb3` | S5 | 53–56 `GPIO036`–`GPIO039` (0) | up | – | 4 |
| `pinctrl_usb4` / `usb5` | S5 | `GPIO034` / `GPIO035` (0) and `GPIO041` / `GPIO042` (1: `USB_DRIVE_VBUS4/5`, pull down) | up | – | 4 |
| `pinctrl_pcie_x8_rc`, `x4`, `x2`, `x1_1`, `x1_0` | S5 | `GPIO002`, `GPIO004`, `GPIO005`, `GPIO003`, `GPIO006` (0) | down | – | 4 |

- **[HW]** Function number 0 in an ACPI `PinGroupFunction` does not mean
  every pad uses mode 0. The group as a whole selects a set of
  (pad, mode) pairs, as shown above.
- **[POLICY]** *(vendor)* Pull, Schmitt and drive values are board
  choices. The pad lists and modes are SoC facts.

> OPEN QUESTION: The `pinctrl_substrate_i2s*`, `pinctrl_gmac*` and other
> groups named by the ACPI notes follow the same pattern; their pads can
> be read from Annex A by function name. Whether the DSDT groups are
> identical to the vendor devicetree groups is assumed, not proven.

## 5.5 Power management

- **[HW]** *(vendor)* The FCH pin controller loses its register contents in
  system suspend; the S5 controller keeps them.
- **[POLICY]** *(vendor)* In devicetree mode the vendor driver saves every
  FCH pad register before suspend, restores it on resume, and then
  re-applies the default pin states. It runs late in suspend and early in
  resume, so that pins are back before any consumer resumes.
- **[HW]** Under ACPI the sources show no OS action. Either firmware
  restores the FCH pads on resume or the OS must. A clean-room OS should
  save all 138 FCH pad registers on suspend and write them back first
  thing on resume. That is safe whichever is true.

> OPEN QUESTION: Whether Sky1 firmware restores FCH pad state on resume
> from suspend-to-RAM under ACPI is not known.

## Annex A: Pad tables

**[HW]** Generated mechanically from the public dual-licensed Sky1 pad
function header. "Mode *n*" is the `FUNC` value. "(no mux)" marks a pad
with no selectable functions. "–" means the mode is not defined.

### S5 controller (`CIXHA017`, `0x16007000`)

| Pad | Offset | Pad name | Mode 0 | Mode 1 | Mode 2 | Mode 3 |
|---|---|---|---|---|---|---|
| 0 | `0x000` | GPIO001 | GPIO001 | – | – | – |
| 1 | `0x004` | GPIO002 | GPIO002 | – | – | – |
| 2 | `0x008` | GPIO003 | GPIO003 | – | – | – |
| 3 | `0x00C` | GPIO004 | GPIO004 | – | – | – |
| 4 | `0x010` | GPIO005 | GPIO005 | – | – | – |
| 5 | `0x014` | GPIO006 | GPIO006 | – | – | – |
| 6 | `0x018` | GPIO007 | GPIO007 | – | – | – |
| 7 | `0x01C` | GPIO008 | GPIO008 | – | – | – |
| 8 | `0x020` | GPIO009 | GPIO009 | – | – | – |
| 9 | `0x024` | GPIO010 | GPIO010 | – | – | – |
| 10 | `0x028` | GPIO011 | GPIO011 | – | – | – |
| 11 | `0x02C` | GPIO012 | GPIO012 | – | – | – |
| 12 | `0x030` | GPIO013 | GPIO013 | – | – | – |
| 13 | `0x034` | GPIO014 | GPIO014 | – | – | – |
| 14 | `0x038` | RSMRST_L | (no mux) | – | – | – |
| 15 | `0x03C` | SRST_L | (no mux) | – | – | – |
| 16 | `0x040` | SLP_S3_L | (no mux) | – | – | – |
| 17 | `0x044` | SLP_S5_L | (no mux) | – | – | – |
| 18 | `0x048` | PWRGD | (no mux) | – | – | – |
| 19 | `0x04C` | PWROK | (no mux) | – | – | – |
| 20 | `0x050` | PWRBTN_L | (no mux) | – | – | – |
| 21 | `0x054` | VDD_DDRIO_GATE | (no mux) | – | – | – |
| 22 | `0x058` | JTAG_GPIO_L | (no mux) | – | – | – |
| 23 | `0x05C` | JTAG_TCK | (no mux) | – | – | – |
| 24 | `0x060` | JTAG_TDI | (no mux) | – | – | – |
| 25 | `0x064` | JTAG_TDO | (no mux) | – | – | – |
| 26 | `0x068` | TMS | (no mux) | – | – | – |
| 27 | `0x06C` | TRSL_L | (no mux) | – | – | – |
| 28 | `0x070` | SFI_I2C0_SCL | SFI_I2C0_SCL | SFI_I3C0_SCL | – | – |
| 29 | `0x074` | SFI_I2C0_SDA | SFI_I2C0_SDA | SFI_I3C0_SDA | – | – |
| 30 | `0x078` | SFI_I2C1_SCL | SFI_I2C1_SCL | SFI_I3C1_SCL | SFI_SPI_CS0 | – |
| 31 | `0x07C` | SFI_I2C1_SDA | SFI_I2C1_SDA | SFI_I3C1_SDA | SFI_SPI_CS1 | – |
| 32 | `0x080` | SFI_GPIO0 | GPIO015 | SFI_SPI_SCK | SFI_GPIO0 | – |
| 33 | `0x084` | SFI_GPIO1 | GPIO016 | SFI_SPI_MOSI | SFI_GPIO1 | – |
| 34 | `0x088` | SFI_GPIO2 | GPIO017 | SFI_SPI_MISO | SFI_GPIO2 | – |
| 35 | `0x08C` | GPIO018 | SFI_GPIO3 | GPIO018 | – | – |
| 36 | `0x090` | GPIO019 | SFI_GPIO4 | GPIO019 | – | – |
| 37 | `0x094` | GPIO020 | SFI_GPIO5 | GPIO020 | – | – |
| 38 | `0x098` | GPIO021 | SFI_GPIO6 | GPIO021 | – | – |
| 39 | `0x09C` | GPIO022 | SFI_GPIO7 | GPIO022 | – | – |
| 40 | `0x0A0` | GPIO023 | SFI_GPIO8 | GPIO023 | SFI_I3C0_PUR_EN_L | – |
| 41 | `0x0A4` | GPIO024 | SFI_GPIO9 | GPIO024 | SFI_I3C1_PUR_EN_L | – |
| 42 | `0x0A8` | SPI1_MISO | SPI1_MISO | GPIO025 | – | – |
| 43 | `0x0AC` | SPI1_CS0 | SPI1_CS0 | GPIO026 | – | – |
| 44 | `0x0B0` | SPI1_CS1 | SPI1_CS1 | GPIO027 | – | – |
| 45 | `0x0B4` | SPI1_MOSI | SPI1_MOSI | GPIO028 | – | – |
| 46 | `0x0B8` | SPI1_CLK | SPI1_CLK | GPIO029 | – | – |
| 47 | `0x0BC` | GPIO030 | GPIO030 | USB_OC0_L | – | – |
| 48 | `0x0C0` | GPIO031 | GPIO031 | USB_OC1_L | – | – |
| 49 | `0x0C4` | GPIO032 | GPIO032 | USB_OC2_L | – | – |
| 50 | `0x0C8` | GPIO033 | GPIO033 | USB_OC3_L | – | – |
| 51 | `0x0CC` | GPIO034 | GPIO034 | USB_OC4_L | – | – |
| 52 | `0x0D0` | GPIO035 | GPIO035 | USB_OC5_L | – | – |
| 53 | `0x0D4` | GPIO036 | GPIO036 | USB_OC6_L | – | – |
| 54 | `0x0D8` | GPIO037 | GPIO037 | USB_OC7_L | – | – |
| 55 | `0x0DC` | GPIO038 | GPIO038 | USB_OC8_L | – | – |
| 56 | `0x0E0` | GPIO039 | GPIO039 | USB_OC9_L | – | – |
| 57 | `0x0E4` | GPIO040 | GPIO040 | USB_DRIVE_VBUS0 | – | – |
| 58 | `0x0E8` | GPIO041 | GPIO041 | USB_DRIVE_VBUS4 | – | – |
| 59 | `0x0EC` | GPIO042 | GPIO042 | USB_DRIVE_VBUS5 | – | – |
| 60 | `0x0F0` | SE_QSPI_CLK | SE_QSPI_CLK | QSPI_CLK | – | – |
| 61 | `0x0F4` | SE_QSPI_CS_L | SE_QSPI_CS_L | QSPI_CS_L | – | – |
| 62 | `0x0F8` | SE_QSPI_DATA0 | SE_QSPI_DATA0 | QSPI_DATA0 | – | – |
| 63 | `0x0FC` | SE_QSPI_DATA1 | SE_QSPI_DATA1 | QSPI_DATA1 | – | – |
| 64 | `0x100` | SE_QSPI_DATA2 | SE_QSPI_DATA2 | QSPI_DATA2 | – | – |
| 65 | `0x104` | SE_QSPI_DATA3 | SE_QSPI_DATA3 | QSPI_DATA3 | – | – |

### FCH / S0 controller (`CIXHA016`, `0x04170000`)

| Pad | Offset | Pad name | Mode 0 | Mode 1 | Mode 2 | Mode 3 |
|---|---|---|---|---|---|---|
| 0 | `0x000` | GPIO043 | GPIO043 | – | – | – |
| 1 | `0x004` | GPIO044 | GPIO044 | – | – | – |
| 2 | `0x008` | GPIO045 | GPIO045 | – | – | – |
| 3 | `0x00C` | GPIO046 | GPIO046 | – | – | – |
| 4 | `0x010` | RESET_IN_L | (no mux) | – | – | – |
| 5 | `0x014` | PLT_RESET_L | (no mux) | – | – | – |
| 6 | `0x018` | THERMTRIP_L | (no mux) | – | – | – |
| 7 | `0x01C` | PROCHOT_L | (no mux) | – | – | – |
| 8 | `0x020` | PM_I2C0_CLK | (no mux) | – | – | – |
| 9 | `0x024` | PM_I2C0_DATA | (no mux) | – | – | – |
| 10 | `0x028` | PM_I2C1_CLK | (no mux) | – | – | – |
| 11 | `0x02C` | PM_I2C1_DATA | (no mux) | – | – | – |
| 12 | `0x030` | PM_I2C2_CLK | (no mux) | – | – | – |
| 13 | `0x034` | PM_I2C2_DATA | (no mux) | – | – | – |
| 14 | `0x038` | PM_I2C3_CLK | (no mux) | – | – | – |
| 15 | `0x03C` | PM_I2C3_DATA | (no mux) | – | – | – |
| 16 | `0x040` | STRAP0 | (no mux) | – | – | – |
| 17 | `0x044` | STRAP1 | (no mux) | – | – | – |
| 18 | `0x048` | DP2_DIGON | DP2_DIGON | – | – | – |
| 19 | `0x04C` | DP2_BLON | DP2_BLON | – | – | – |
| 20 | `0x050` | DP2_VARY_BL | DP2_VARY_BL | – | – | – |
| 21 | `0x054` | I2C7_SCL | I2C7_SCL | – | – | – |
| 22 | `0x058` | I2C7_SDA | I2C7_SDA | – | – | – |
| 23 | `0x05C` | UART6_CSU_SE_TXD | (no mux) | – | – | – |
| 24 | `0x060` | CLK_REQ1_L | (no mux) | – | – | – |
| 25 | `0x064` | CLK_REQ3_L | (no mux) | – | – | – |
| 26 | `0x068` | I2C5_SCL | I2C5_SCL | GPIO047 | – | – |
| 27 | `0x06C` | I2C5_SDA | I2C5_SDA | GPIO048 | – | – |
| 28 | `0x070` | I2C6_SCL | I2C6_SCL | GPIO049 | – | – |
| 29 | `0x074` | I2C6_SDA | I2C6_SDA | GPIO050 | – | – |
| 30 | `0x078` | I2C0_CLK | I2C0_CLK | GPIO051 | – | – |
| 31 | `0x07C` | I2C0_SDA | I2C0_SDA | GPIO052 | – | – |
| 32 | `0x080` | I2C1_CLK | I2C1_CLK | GPIO053 | – | – |
| 33 | `0x084` | I2C1_SDA | I2C1_SDA | GPIO054 | – | – |
| 34 | `0x088` | I2C2_SCL | I2C2_SCL | I3C0_SCL | GPIO055 | – |
| 35 | `0x08C` | I2C2_SDA | I2C2_SDA | I3C0_SDA | GPIO056 | – |
| 36 | `0x090` | GPIO057 | GPIO057 | I3C0_PUR_EN_L | – | – |
| 37 | `0x094` | I2C3_CLK | I2C3_CLK | I3C1_CLK | GPIO058 | – |
| 38 | `0x098` | I2C3_SDA | I2C3_SDA | I3C1_SDA | GPIO059 | – |
| 39 | `0x09C` | GPIO060 | GPIO060 | I3C1_PUR_EN_L | – | – |
| 40 | `0x0A0` | I2C4_CLK | I2C4_CLK | GPIO061 | – | – |
| 41 | `0x0A4` | I2C4_SDA | I2C4_SDA | GPIO062 | – | – |
| 42 | `0x0A8` | HDA_BITCLK | HDA_BITCLK | I2S0_SCK | I2S9_RSCK_DBG | – |
| 43 | `0x0AC` | HDA_RST_L | HDA_RST_L | I2S0_DATA_IN | I2S9_DATA_IN0_DBG | – |
| 44 | `0x0B0` | HDA_SDIN0 | HDA_SDIN0 | I2S0_MCLK | I2S9_TSCK_DBG | – |
| 45 | `0x0B4` | HDA_SDOUT0 | HDA_SDOUT0 | I2S0_DATA_OUT | I2S9_TWS_DBG | – |
| 46 | `0x0B8` | HDA_SYNC | HDA_SYNC | I2S0_WS | I2S9_RWS_DBG | – |
| 47 | `0x0BC` | HDA_SDIN1 | HDA_SDIN1 | GPIO063 | I2S9_DATA_IN1_DBG | – |
| 48 | `0x0C0` | HDA_SDOUT1 | HDA_SDOUT1 | GPIO064 | I2S9_DATA_OUT0_DBG | – |
| 49 | `0x0C4` | I2S1_MCLK | I2S1_MCLK | GPIO065 | – | – |
| 50 | `0x0C8` | I2S1_SCK | I2S1_SCK | GPIO066 | – | – |
| 51 | `0x0CC` | I2S1_WS | I2S1_WS | GPIO067 | – | – |
| 52 | `0x0D0` | I2S1_DATA_IN | I2S1_DATA_IN | GPIO068 | – | – |
| 53 | `0x0D4` | I2S1_DATA_OUT | I2S1_DATA_OUT | GPIO069 | – | – |
| 54 | `0x0D8` | I2S2_MCLK | I2S2_MCLK | GPIO070 | – | – |
| 55 | `0x0DC` | I2S2_RSCK | I2S2_RSCK | GPIO071 | I2S5_RSCK_DBG | I2S6_RSCK_DBG |
| 56 | `0x0E0` | I2S2_RWS | I2S2_RWS | GPIO072 | I2S5_RWS_DBG | I2S6_RWS_DBG |
| 57 | `0x0E4` | I2S2_TSCK | I2S2_TSCK | GPIO073 | I2S5_TSCK_DBG | I2S6_TSCK_DBG |
| 58 | `0x0E8` | I2S2_TWS | I2S2_TWS | GPIO074 | I2S5_TWS_DBG | I2S6_TWS_DBG |
| 59 | `0x0EC` | I2S2_DATA_IN0 | I2S2_DATA_IN0 | GPIO075 | I2S5_DATA_IN0_DBG | I2S6_DATA_IN0_DBG |
| 60 | `0x0F0` | I2S2_DATA_IN1 | I2S2_DATA_IN1 | GPIO076 | I2S5_DATA_IN1_DBG | I2S6_DATA_IN1_DBG |
| 61 | `0x0F4` | I2S2_DATA_OUT0 | I2S2_DATA_OUT0 | GPIO077 | I2S5_DATA_OUT0_DBG | I2S6_DATA_OUT0_DBG |
| 62 | `0x0F8` | I2S2_DATA_OUT1 | I2S2_DATA_OUT1 | GPIO078 | I2S5_DATA_OUT1_DBG | I2S6_DATA_OUT1_DBG |
| 63 | `0x0FC` | I2S2_DATA_OUT2 | I2S2_DATA_OUT2 | GPIO079 | – | – |
| 64 | `0x100` | I2S2_DATA_OUT3 | I2S2_DATA_OUT3 | GPIO080 | I2S9_DATA_OUT1_DBG | – |
| 65 | `0x104` | I2S3_MCLK | I2S3_MCLK | GPIO081 | – | – |
| 66 | `0x108` | I2S3_RSCK | I2S3_RSCK | GPIO082 | I2S7_RSCK_DBG | I2S8_RSCK_DBG |
| 67 | `0x10C` | I2S3_RWS | I2S3_RWS | GPIO083 | I2S7_RWS_DBG | I2S8_RWS_DBG |
| 68 | `0x110` | I2S3_TSCK | I2S3_TSCK | GPIO084 | I2S7_TSCK_DBG | I2S8_TSCK_DBG |
| 69 | `0x114` | I2S3_TWS | I2S3_TWS | GPIO085 | I2S7_TWS_DBG | I2S8_TWS_DBG |
| 70 | `0x118` | I2S3_DATA_IN0 | I2S3_DATA_IN0 | GPIO086 | I2S7_DATA_IN0_DBG | I2S8_DATA_IN0_DBG |
| 71 | `0x11C` | I2S3_DATA_IN1 | I2S3_DATA_IN1 | GPIO087 | I2S7_DATA_IN1_DBG | I2S8_DATA_IN1_DBG |
| 72 | `0x120` | I2S3_DATA_OUT0 | I2S3_DATA_OUT0 | GPIO088 | I2S7_DATA_OUT0_DBG | I2S8_DATA_OUT0_DBG |
| 73 | `0x124` | I2S3_DATA_OUT1 | I2S3_DATA_OUT1 | GPIO089 | I2S7_DATA_OUT1_DBG | I2S8_DATA_OUT1_DBG |
| 74 | `0x128` | GPIO090 | GPIO090 | I2S4_MCLK_LB | – | – |
| 75 | `0x12C` | GPIO091 | GPIO091 | I2S4_SCK_LB | – | – |
| 76 | `0x130` | GPIO092 | GPIO092 | I2S4_WS_LB | – | – |
| 77 | `0x134` | GPIO093 | GPIO093 | I2S4_DATA_IN_LB | – | – |
| 78 | `0x138` | GPIO094 | GPIO094 | I2S4_DATA_OUT_LB | – | – |
| 79 | `0x13C` | UART0_TXD | UART0_TXD | PWM0 | GPIO095 | – |
| 80 | `0x140` | UART0_RXD | UART0_RXD | PWM1 | GPIO096 | – |
| 81 | `0x144` | UART0_CTS | UART0_CTS | FAN_OUT2 | GPIO097 | – |
| 82 | `0x148` | UART0_RTS | UART0_RTS | FAN_TACH2 | GPIO098 | – |
| 83 | `0x14C` | UART1_TXD | UART1_TXD | FAN_OUT0 | GPIO099 | – |
| 84 | `0x150` | UART1_RXD | UART1_RXD | FAN_TACH0 | GPIO100 | – |
| 85 | `0x154` | UART1_CTS | UART1_CTS | FAN_OUT1 | GPIO101 | – |
| 86 | `0x158` | UART1_RTS | UART1_RTS | FAN_TACH1 | GPIO102 | – |
| 87 | `0x15C` | UART2_TXD | UART2_TXD | GPIO103 | – | – |
| 88 | `0x160` | UART2_RXD | UART2_RXD | GPIO104 | – | – |
| 89 | `0x164` | UART3_TXD | UART3_TXD | GPIO105 | – | – |
| 90 | `0x168` | UART3_RXD | UART3_RXD | GPIO106 | – | – |
| 91 | `0x16C` | UART3_CTS | UART3_CTS | GPIO107 | TRIGIN0 | – |
| 92 | `0x170` | UART3_RTS | UART3_RTS | GPIO108 | TRIGIN1 | – |
| 93 | `0x174` | UART4_CSU_PM_TXD | UART4_CSU_PM_TXD | GPIO109 | – | – |
| 94 | `0x178` | UART4_CSU_PM_RXD | UART4_CSU_PM_RXD | GPIO110 | – | – |
| 95 | `0x17C` | UART5_CSU_SE_TXD | UART5_CSU_SE_TXD | GPIO111 | – | – |
| 96 | `0x180` | UART5_CSU_SE_RXD | UART5_CSU_SE_RXD | GPIO112 | – | – |
| 97 | `0x184` | UART6_CSU_SE_RXD | UART6_CSU_SE_RXD | GPIO113 | – | – |
| 98 | `0x188` | CLK_REQ0_L | CLK_REQ0_L | GPIO114 | – | – |
| 99 | `0x18C` | CLK_REQ2_L | CLK_REQ2_L | GPIO115 | – | – |
| 100 | `0x190` | CLK_REQ4_L | CLK_REQ4_L | GPIO116 | – | – |
| 101 | `0x194` | CSI0_MCLK0 | CSI0_MCLK0 | GPIO117 | – | – |
| 102 | `0x198` | CSI0_MCLK1 | CSI0_MCLK1 | GPIO118 | – | – |
| 103 | `0x19C` | CSI1_MCLK0 | CSI1_MCLK0 | GPIO119 | – | – |
| 104 | `0x1A0` | CSI1_MCLK1 | CSI1_MCLK1 | GPIO120 | – | – |
| 105 | `0x1A4` | GPIO121 | GPIO121 | GMAC0_REFCLK_25M | – | – |
| 106 | `0x1A8` | GPIO122 | GPIO122 | GMAC0_TX_CTL | – | – |
| 107 | `0x1AC` | GPIO123 | GPIO123 | GMAC0_TXD0 | – | – |
| 108 | `0x1B0` | GPIO124 | GPIO124 | GMAC0_TXD1 | – | – |
| 109 | `0x1B4` | GPIO125 | GPIO125 | GMAC0_TXD2 | – | – |
| 110 | `0x1B8` | GPIO126 | GPIO126 | GMAC0_TXD3 | – | – |
| 111 | `0x1BC` | GPIO127 | GPIO127 | GMAC0_TX_CLK | – | – |
| 112 | `0x1C0` | GPIO128 | GPIO128 | GMAC0_RX_CTL | – | – |
| 113 | `0x1C4` | GPIO129 | GPIO129 | GMAC0_RXD0 | – | – |
| 114 | `0x1C8` | GPIO130 | GPIO130 | GMAC0_RXD1 | – | – |
| 115 | `0x1CC` | GPIO131 | GPIO131 | GMAC0_RXD2 | – | – |
| 116 | `0x1D0` | GPIO132 | GPIO132 | GMAC0_RXD3 | – | – |
| 117 | `0x1D4` | GPIO133 | GPIO133 | GMAC0_RX_CLK | – | – |
| 118 | `0x1D8` | GPIO134 | GPIO134 | GMAC0_MDC | – | – |
| 119 | `0x1DC` | GPIO135 | GPIO135 | GMAC0_MDIO | – | – |
| 120 | `0x1E0` | GPIO136 | GPIO136 | GMAC1_REFCLK_25M | – | – |
| 121 | `0x1E4` | GPIO137 | GPIO137 | GMAC1_TX_CTL | – | – |
| 122 | `0x1E8` | GPIO138 | GPIO138 | GMAC1_TXD0 | SPI2_MISO | – |
| 123 | `0x1EC` | GPIO139 | GPIO139 | GMAC1_TXD1 | SPI2_CS0 | – |
| 124 | `0x1F0` | GPIO140 | GPIO140 | GMAC1_TXD2 | SPI2_CS1 | – |
| 125 | `0x1F4` | GPIO141 | GPIO141 | GMAC1_TXD3 | SPI2_MOSI | – |
| 126 | `0x1F8` | GPIO142 | GPIO142 | GMAC1_TX_CLK | SPI2_CLK | – |
| 127 | `0x1FC` | GPIO143 | GPIO143 | GMAC1_RX_CTL | – | – |
| 128 | `0x200` | GPIO144 | GPIO144 | GMAC1_RXD0 | – | – |
| 129 | `0x204` | GPIO145 | GPIO145 | GMAC1_RXD1 | – | – |
| 130 | `0x208` | GPIO146 | GPIO146 | GMAC1_RXD2 | – | – |
| 131 | `0x20C` | GPIO147 | GPIO147 | GMAC1_RXD3 | – | – |
| 132 | `0x210` | GPIO148 | GPIO148 | GMAC1_RX_CLK | – | – |
| 133 | `0x214` | GPIO149 | GPIO149 | GMAC1_MDC | – | – |
| 134 | `0x218` | GPIO150 | GPIO150 | GMAC1_MDIO | – | – |
| 135 | `0x21C` | GPIO151 | GPIO151 | PM_GPIO0 | – | – |
| 136 | `0x220` | GPIO152 | GPIO152 | PM_GPIO1 | – | – |
| 137 | `0x224` | GPIO153 | GPIO153 | PM_GPIO2 | – | – |

