# Qualcomm SC8280XP CPU Frequency Scaling (EPSS) and Thermal Sensors (TSENS v2) Specification

## 1. Overview

**[HW]** On the Qualcomm SC8280XP (Radxa Dragon Q8B), CPU dynamic voltage and frequency scaling (DVFS) is managed autonomously by the Embedded Power Subsystem (EPSS / `cpufreq-hw`) hardware blocks, while on-die temperature monitoring is provided by four Thermal Sensor v2 (TSENS v2) controllers.

| Hardware Block | Physical Base Address | Region Size | Role / Coverage | Interrupts (GICv3 / PDC) |
|---|---|---|---|---|
| `epss0` (`cpufreq@18591000` Domain 0) | `0x18591000` | `0x1000` | Silver / Efficiency Cluster (CPUs 0–3, MPIDR `0x000`–`0x300`) | GIC SPI 36 (`dcvsh-irq-0`) |
| `epss1` (`cpufreq@18592000` Domain 1) | `0x18592000` | `0x1000` | Gold / Performance Cluster (CPUs 4–7, MPIDR `0x400`–`0x700`) | GIC SPI 37 (`dcvsh-irq-1`) |
| `epss_l3` (`interconnect@18590000`) | `0x18590000` | `0x1000` | Shared L3 Cache / EPSS Top-Level | — |
| `tsens0` TM (`thermal-sensor@c263000`) | `0x0C263000` | `0x200` | Thermal Monitor 0 (14 sensors: `s0`–`s13`) | PDC `0x1A` (`uplow`), `0x1C` (`critical`) |
| `tsens0` SROT | `0x0C222000` | `0x008` | Secure Root of Trust 0 (`VER_2_X`, `0x10020200`) | — |
| `tsens1` TM (`thermal-sensor@c265000`) | `0x0C265000` | `0x200` | Thermal Monitor 1 (16 sensors: `s0`–`s15`) | PDC `0x1B` (`uplow`), `0x1D` (`critical`) |
| `tsens1` SROT | `0x0C223000` | `0x008` | Secure Root of Trust 1 (`VER_2_X`, `0x10020200`) | — |
| `tsens2` TM (`thermal-sensor@c251000`) | `0x0C251000` | `0x200` | Thermal Monitor 2 (11 sensors: `s0`–`s10`) | PDC `0x7A` (`uplow`), `0x7C` (`critical`) |
| `tsens2` SROT | `0x0C224000` | `0x008` | Secure Root of Trust 2 (`VER_2_X`, `0x10020200`) | — |
| `tsens3` TM (`thermal-sensor@c252000`) | `0x0C252000` | `0x200` | Thermal Monitor 3 (5 sensors: `s0`–`s4`) | PDC `0x7B` (`uplow`), `0x7D` (`critical`) |
| `tsens3` SROT | `0x0C225000` | `0x008` | Secure Root of Trust 3 (`VER_2_X`, `0x10020200`) | — |

---

## 2. EPSS Hardware Frequency Scaling

### 2.1 EPSS Register Layout

**[HW]** Each EPSS frequency domain (`0x18591000` for CPUs 0–3, `0x18592000` for CPUs 4–7) exposes a 4 KiB MMIO page using the EPSS register layout (which differs from legacy OSM/LUT offsets):

| Offset | Register Name | Access | Description |
|---|---|---|---|
| `0x0000` | `REG_ENABLE` | RW | Bit `0`: `1` = EPSS hardware frequency scaling enabled (`0x00000001` at boot) |
| `0x0020` | `REG_PERF_STATE` | RW | Requested performance state index (`0`..`MAX_CORNERS - 1`, written by OS to change cluster frequency) |
| `0x00B0` | `REG_INTR_EN` | RW | Throttle / DCVSH interrupt enable |
| `0x00B4` | `REG_INTR_CLR` | WO | Throttle / DCVSH interrupt clear |
| `0x00B8` | `REG_INTR_STATUS` | RO | Throttle / DCVSH interrupt status |
| `0x0100` | `REG_FREQ_LUT_TABLE` | RO | Frequency Look-Up Table: 40 entries of 32-bit words (`0x0100` through `0x019C`, stride `4`) |
| `0x0200` | `REG_VOLT_LUT_TABLE` | RO | Voltage Look-Up Table: 40 entries of 32-bit words (`0x0200` through `0x029C`, stride `4`) |
| `0x0320` | `REG_CURRENT_VOTE` | RO | Current hardware-aggregated performance vote and frequency factor |
| `0x0328` | `REG_DOMAIN_STATE` | RO | Hardware thermal/power throttle limit status (`[7:0]` = throttled performance index) |

### 2.2 LUT Entry Bitfields and Frequency/Voltage Formulas

**[HW]** Each of the 40 entries (`i = 0 .. 39`) in `REG_FREQ_LUT_TABLE` (`0x0100 + 4 * i`) and `REG_VOLT_LUT_TABLE` (`0x0200 + 4 * i`) has the following layout:

**[HW]** **`REG_FREQ_LUT_TABLE[i]` (`0x0100 + 4 * i`)**:

| Bits | Field Name | Description |
|---|---|---|
| `31:24` | `LUT_ROW_INDEX` | Row index (`0`..`39`) |
| `23:16` | `LUT_CORE_COUNT` | Boost core count limit (`[18:16]`). In EPSS, boost rows share the same `LUT_CORE_COUNT` as normal rows (`1`), or specify max active cores for turbo boost |
| `15:12` | `LUT_SRC` | PLL clock source selector (`0` = XO 19.2 MHz bypass, `1` = PLL source) |
| `11:0` | `LUT_L_VAL` | 12-bit PLL multiplier (`L_VAL`) |

**[HW]** **Frequency Calculation Formula**:
- The reference crystal oscillator (`xo-board-clk`) frequency is **19,200,000 Hz (19.2 MHz)**.
- For a given `L_VAL`:
  $$\text{Frequency (Hz)} = L\_VAL \times 19,200,000\text{ Hz}$$
  $$\text{Frequency (kHz)} = L\_VAL \times 19,200\text{ kHz}$$

**[HW]** **`REG_VOLT_LUT_TABLE[i]` (`0x0200 + 4 * i`)**:

| Bits | Field Name | Description |
|---|---|---|
| `27:16` | `LUT_ULV_VOLT` | Secondary/under-voltage rail in millivolts (`mV`) |
| `11:0` | `LUT_VOLT` | Primary cluster voltage in millivolts (`mV`). Multiply by `1000` for microvolts (`µV`) |

**[HW]** **LUT Parsing Termination Rule**:
- Firmware populates valid operating performance points (OPPs) in ascending order of frequency starting at index `0`.
- All remaining unused entries up to index `39` are filled with duplicates of the final maximum-frequency entry.
- When parsing `i = 0 .. 39`: if `i > 0` and `prev_freq == curr_freq` (and `prev_core_count == curr_core_count`), stop parsing—index `i - 1` is the highest valid performance state (`max_index = 20`, giving **21 distinct steps** `0`..`20` on both `epss0` and `epss1`).

### 2.3 Empirical LUT Tables on Radxa Dragon Q8B

**[HW]** **`epss0` (`0x18591000`, Silver Cluster CPUs 0–3) Empirical LUT Values**:
- At OS handoff from UEFI: `REG_ENABLE` (`0x000`) = `0x00000001`, `REG_PERF_STATE` (`0x020`) = `14` (`1900.8 MHz`).
- Entries `20` through `39` repeat `L_VAL = 127` (`2438.4 MHz`, `952 mV`).

| Index (`REG_PERF_STATE`) | `FREQ_LUT` Raw (`0x100 + 4*i`) | `SRC` | `L_VAL` | Frequency (MHz) | `VOLT_LUT` Raw (`0x200 + 4*i`) | Voltage (mV) |
|---|---|---|---|---|---|---|
| `0` | `0x00010000` | `0` | `0` (XO) | **300.0 MHz** | `0x023C023C` | 572 mV |
| `1` | `0x01011015` | `1` | `21` | **403.2 MHz** | `0x023C023C` | 572 mV |
| `2` | `0x0201101B` | `1` | `27` | **518.4 MHz** | `0x02440244` | 580 mV |
| `3` | `0x03011020` | `1` | `32` | **614.4 MHz** | `0x02580258` | 600 mV |
| `4` | `0x04011026` | `1` | `38` | **729.6 MHz** | `0x026C026C` | 620 mV |
| `5` | `0x0501102C` | `1` | `44` | **844.8 MHz** | `0x02800280` | 640 mV |
| `6` | `0x06011032` | `1` | `50` | **960.0 MHz** | `0x02980298` | 664 mV |
| `7` | `0x07011038` | `1` | `56` | **1075.2 MHz** | `0x02AC02AC` | 684 mV |
| `8` | `0x0801103E` | `1` | `62` | **1190.4 MHz** | `0x02B802B8` | 696 mV |
| `9` | `0x09011044` | `1` | `68` | **1305.6 MHz** | `0x02CC02CC` | 716 mV |
| `10` | `0x0A01104A` | `1` | `74` | **1420.8 MHz** | `0x02E002E0` | 736 mV |
| `11` | `0x0B011050` | `1` | `80` | **1536.0 MHz** | `0x02FC02FC` | 764 mV |
| `12` | `0x0C011056` | `1` | `86` | **1651.2 MHz** | `0x03100310` | 784 mV |
| `13` | `0x0D01105C` | `1` | `92` | **1766.4 MHz** | `0x03280328` | 808 mV |
| `14` | `0x0E011063` | `1` | `99` | **1900.8 MHz** | `0x03440344` | 836 mV |
| `15` | `0x0F01106A` | `1` | `106` | **2035.2 MHz** | `0x03680368` | 872 mV |
| `16` | `0x1001106E` | `1` | `110` | **2112.0 MHz** | `0x03780378` | 888 mV |
| `17` | `0x11011073` | `1` | `115` | **2208.0 MHz** | `0x03980398` | 920 mV |
| `18` | `0x12011079` | `1` | `121` | **2323.2 MHz** | `0x03B003B0` | 944 mV |
| `19` | `0x1301107C` | `1` | `124` | **2380.8 MHz** | `0x03B803B8` | 952 mV |
| `20` (`20`–`39`) | `0x1401107F` | `1` | `127` | **2438.4 MHz** | `0x03B803B8` | 952 mV |

*Note*: Entry `0` on `epss0` has `SRC = 0` and `L_VAL = 0`, which represents the 300 MHz GPLL0 bypass rate in hardware (`opp-300000000`).

**[HW]** **`epss1` (`0x18592000`, Gold Cluster CPUs 4–7) Empirical LUT Values**:
- At OS handoff from UEFI: `REG_ENABLE` (`0x000`) = `0x00000001`, `REG_PERF_STATE` (`0x020`) = `9` (`1862.4 MHz`).
- Entries `20` through `39` repeat `L_VAL = 156` (`2995.2 MHz`, `1084 mV`).

| Index (`REG_PERF_STATE`) | `FREQ_LUT` Raw (`0x100 + 4*i`) | `SRC` | `L_VAL` | Frequency (MHz) | `VOLT_LUT` Raw (`0x200 + 4*i`) | Voltage (mV) |
|---|---|---|---|---|---|---|
| `0` | `0x0001102B` | `1` | `43` | **825.6 MHz** | `0x02380238` | 568 mV |
| `1` | `0x01011031` | `1` | `49` | **940.8 MHz** | `0x02500250` | 592 mV |
| `2` | `0x02011037` | `1` | `55` | **1056.0 MHz** | `0x026C026C` | 620 mV |
| `3` | `0x0301103D` | `1` | `61` | **1171.2 MHz** | `0x028C028C` | 652 mV |
| `4` | `0x04011043` | `1` | `67` | **1286.4 MHz** | `0x02AC02AC` | 684 mV |
| `5` | `0x05011049` | `1` | `73` | **1401.6 MHz** | `0x02C802C8` | 712 mV |
| `6` | `0x0601104F` | `1` | `79` | **1516.8 MHz** | `0x02DC02DC` | 732 mV |
| `7` | `0x07011055` | `1` | `85` | **1632.0 MHz** | `0x02F002F0` | 752 mV |
| `8` | `0x0801105B` | `1` | `91` | **1747.2 MHz** | `0x03080308` | 776 mV |
| `9` | `0x09011061` | `1` | `97` | **1862.4 MHz** | `0x03200320` | 800 mV |
| `10` | `0x0A011067` | `1` | `103` | **1977.6 MHz** | `0x033C033C` | 828 mV |
| `11` | `0x0B01106C` | `1` | `108` | **2073.6 MHz** | `0x03580358` | 856 mV |
| `12` | `0x0C011071` | `1` | `113` | **2169.6 MHz** | `0x03680368` | 872 mV |
| `13` | `0x0D011077` | `1` | `119` | **2284.8 MHz** | `0x03880388` | 904 mV |
| `14` | `0x0E01107D` | `1` | `125` | **2400.0 MHz** | `0x03AC03AC` | 940 mV |
| `15` | `0x0F011082` | `1` | `130` | **2496.0 MHz** | `0x03CC03CC` | 972 mV |
| `16` | `0x10011087` | `1` | `135` | **2592.0 MHz** | `0x03DC03DC` | 988 mV |
| `17` | `0x1101108C` | `1` | `140` | **2688.0 MHz** | `0x03F003F0` | 1008 mV |
| `18` | `0x12011092` | `1` | `146` | **2803.2 MHz** | `0x04080408` | 1032 mV |
| `19` | `0x13011097` | `1` | `151` | **2899.2 MHz** | `0x041C041C` | 1052 mV |
| `20` (`20`–`39`) | `0x1401109C` | `1` | `156` | **2995.2 MHz** | `0x043C043C` | 1084 mV |

### 2.4 Frequency Selection and Current Hardware Vote Readback

**[HW]** **Setting Cluster Frequency**:
1. Verify `REG_ENABLE` (`0x0000`) bit `0` is `1`.
2. Write the desired LUT index (`0`..`20`) as a 32-bit integer to `REG_PERF_STATE` (`0x0020`). The EPSS hardware autonomously slews the cluster voltage and PLL multiplier.

**[HW]** **Reading Actual Hardware-Aggregated Frequency**:
- Read `val = REG_CURRENT_VOTE` (`0x0320`).
- Bits `[19:12]` (mask `0x000FF000`, shift 12) contain the instantaneous frequency multiplier factor (`L_VAL`), or read `REG_DOMAIN_STATE` (`0x0328`) bits `[7:0]` for the hardware-throttled performance state index.

---

## 3. TSENS v2 Thermal Sensor Controllers

### 3.1 Architecture and Register Map (`VER_2_X`)

**[HW]** The SC8280XP contains **four** TSENS v2.2 controllers (`tsens0`–`tsens3`), providing a total of **46 on-die thermal sensors**. Each controller consists of two MMIO apertures:
1. **SROT (Secure Root of Trust) Block** (size `0x008`): Contains the hardware version and global enable status (configured by secure boot firmware).
2. **TM (Thermal Monitor) Block** (size `0x200`): Contains sensor temperature status registers, upper/lower threshold registers, critical threshold registers, and interrupt controls accessible to the OS.

**[HW]** **SROT Aperture Register Map (`0x0C222000`, `0x0C223000`, `0x0C224000`, `0x0C225000`)**:

| Offset | Register Name | Access | Description |
|---|---|---|---|
| `0x0000` | `SROT_HW_VER` | RO | Hardware version: `[31:28]` = Major (`1`), `[27:16]` = Minor (`2`), `[15:0]` = Step (`0x0200`). Raw value on SC8280XP is `0x10020200` |
| `0x0004` | `SROT_CTRL` | RO | Global control: `[0]` = `TSENS_EN` (`1`), `[1]` = `TSENS_SW_RST`, `[2]` = `SENSOR_EN`, `[3]` = `CODE_OR_TEMP` (`1` = status registers report signed 0.1 °C degrees rather than raw ADC codes), `[21:3]` = enabled sensor bitmask |

**[HW]** **TM Aperture Register Map (`0x0C263000`, `0x0C265000`, `0x0C251000`, `0x0C252000`)**:

| Offset | Register Name | Access | Description |
|---|---|---|---|
| `0x0000` | `TM_INT_EN` | RW | Interrupt enable mask: `[0]` = `TRDY_INT_EN`, `[1]` = `LOWER_INT_EN`, `[2]` = `UPPER_INT_EN`, `[3]` = `CRITICAL_INT_EN`. Initialized by firmware to `0x0000000E` (`LOWER`, `UPPER`, `CRITICAL` enabled) |
| `0x0004` | `TM_UPPER_LOWER_INT_STATUS` | RO | Per-sensor threshold interrupt status: `[15:0]` = `LOWER_STATUS` (bit `n` for sensor `n`), `[31:16]` = `UPPER_STATUS` (bit `16 + n` for sensor `n`) |
| `0x0008` | `TM_UPPER_LOWER_INT_CLEAR` | RW | Write `1` to bit `n` (lower) or `16 + n` (upper) to clear/acknowledge sensor `n` threshold interrupt |
| `0x000C` | `TM_UPPER_LOWER_INT_MASK` | RW | Mask register (`1` = masked/disabled, `0` = unmasked/enabled): `[15:0]` = `LOWER_MASK`, `[31:16]` = `UPPER_MASK` |
| `0x0010` | `TM_CRITICAL_INT_STATUS` | RO | Per-sensor critical interrupt status (`[15:0]`, bit `n` for sensor `n`) |
| `0x0014` | `TM_CRITICAL_INT_CLEAR` | RW | Per-sensor critical interrupt clear (`[15:0]`) |
| `0x0018` | `TM_CRITICAL_INT_MASK` | RW | Per-sensor critical interrupt mask (`[15:0]`, `1` = masked, `0` = unmasked) |
| `0x0020 + 4*n` | `TM_SN_UPPER_LOWER_THRESHOLD(n)` | RW | Thresholds for sensor `n` (`n = 0..15`): `[11:0]` = `LOWER_THRESHOLD`, `[23:12]` = `UPPER_THRESHOLD` (each is a 12-bit signed two's-complement integer in 0.1 °C units) |
| `0x0060 + 4*n` | `TM_SN_CRITICAL_THRESHOLD(n)` | RW | Critical threshold for sensor `n` (`n = 0..15`): `[11:0]` = `CRITICAL_THRESHOLD` (12-bit signed integer in 0.1 °C units) |
| `0x00A0 + 4*n` | `TM_SN_STATUS(n)` | RO | Live temperature reading and status flags for sensor `n` (`n = 0..15`, covering `0x00A0`–`0x00DC`) |
| `0x00E4` | `TM_TRDY` | RO | Bit `0`: `1` = measurement scan cycle complete (`TRDY`) |
| `0x00F0` | `TM_WDOG_LOG` | RO | Watchdog log register |

### 3.2 `TM_SN_STATUS(n)` (`0x00A0 + 4 * n`) Bitfields and Temperature Formula

**[HW]** Each sensor status register `TM_SN_STATUS(n)` at `TM_BASE + 0x00A0 + 4 * n` (`0 <= n <= 15`) has the following layout on TSENS v2 (`VER_2_X`):

| Bits | Field Name | Description |
|---|---|---|
| `21` | `VALID` | **`1` = `LAST_TEMP` contains a valid, completed measurement; `0` = measurement in progress / invalid** |
| `20` | `CRITICAL_STATUS` | `1` = temperature currently exceeds `CRITICAL_THRESHOLD` |
| `19` | `UPPER_STATUS` | `1` = temperature currently exceeds `UPPER_THRESHOLD` |
| `18` | `LOWER_STATUS` | `1` = temperature is currently below `LOWER_THRESHOLD` |
| `11:0` | `LAST_TEMP` | **12-bit signed two's-complement integer** representing temperature in **tenths of a degree Celsius (0.1 °C / deci-Celsius)** |

**[HW]** **Step-by-Step Temperature Reading Algorithm**:
1. Read `status = TM_SN_STATUS(n)` at `TM_BASE + 0x00A0 + 4 * n`.
2. Check bit `21` (`VALID`, mask `0x00200000`):
   - If bit `21` is `0`, wait `10` microseconds and re-read `TM_SN_STATUS(n)` (up to 3 attempts). Once bit `21` is `1`, proceed.
3. Extract the 12-bit raw value: `raw = status & 0xFFF`.
4. Sign-extend from 12 bits (bit `11` is the sign bit):
   - If `raw & 0x800` is non-zero, `deci_celsius = raw - 4096` (or `raw | ~0xFFF`).
   - Otherwise, `deci_celsius = raw`.
5. Convert to millidegrees Celsius (`m°C`):
   $$\text{Temperature (m°C)} = \text{deci\_celsius} \times 100$$
   $$\text{Temperature (°C)} = \frac{\text{deci\_celsius}}{10.0}$$
   - Example from empirical dump: `tsens0` `TM_SN_STATUS(1)` reads `0x002001B1`. Bit `21` is `1` (`VALID`), and `raw = 0x1B1 = 433`, which equals **43.3 °C** (`43,300 m°C`).

### 3.3 Complete SC8280XP Sensor-to-Thermal-Zone Mapping

**[HW]** **[ABI]** The table below gives the exact hardware sensor count, enabled sensor mask from `SROT_CTRL` (`0x0004`), and physical subsystem mapping for all four TSENS controllers on SC8280XP:

| Controller | TM Base | SROT Base | `SROT_CTRL` Raw | Sensor Count | Sensor Index -> Thermal Zone / Subsystem Mapping |
|---|---|---|---|---|---|
| **`tsens0`** | `0x0C263000` | `0x0C222000` | `0x000FFFBD` (`14` sensors) | 14 (`s0`–`s13`) | `s0`: `aoss-0-thermal`<br>`s1`: `cpu0-0-thermal` (Silver CPU 0)<br>`s2`: `cpu1-0-thermal` (Silver CPU 1)<br>`s3`: `cpu2-0-thermal` (Silver CPU 2)<br>`s4`: `cpu3-0-thermal` (Silver CPU 3)<br>`s5`: `cpu4-0-thermal` (Gold CPU 4)<br>`s6`: `cpu5-0-thermal` (Gold CPU 5)<br>`s7`: `cpu6-0-thermal` (Gold CPU 6)<br>`s8`: `cpu7-0-thermal` (Gold CPU 7)<br>`s9`: `cluster0-thermal` (CPU Cluster 0)<br>`s10`: `nsp0-0-thermal` (NSP0 Compute DSP)<br>`s11`: *(unused / unmapped)*<br>`s12`: `smss-0-thermal`<br>`s13`: `camss-0-thermal` (Camera Subsystem 0) |
| **`tsens1`** | `0x0C265000` | `0x0C223000` | `0x001FEFFD` (`16` sensors) | 16 (`s0`–`s15`) | `s0`: `aoss-1-thermal`<br>`s1`: `cpu0-1-thermal` (Silver CPU 0)<br>`s2`: `cpu1-1-thermal` (Silver CPU 1)<br>`s3`: `cpu2-1-thermal` (Silver CPU 2)<br>`s4`: `cpu3-1-thermal` (Silver CPU 3)<br>`s5`: `cpu4-1-thermal` (Gold CPU 4)<br>`s6`: `cpu5-1-thermal` (Gold CPU 5)<br>`s7`: `cpu6-1-thermal` (Gold CPU 6)<br>`s8`: `cpu7-1-thermal` (Gold CPU 7)<br>`s9`: `cluster1-thermal` (CPU Cluster 1)<br>`s10`: `nsp0-2-thermal` (NSP0 Compute DSP)<br>`s11`: *(unused / unmapped)*<br>`s12`: `smss-1-thermal`<br>`s13`: `camss-1-thermal` (Camera Subsystem 1)<br>`s14`: `pcie-0-thermal` (PCIe 0)<br>`s15`: `mem-0-thermal` (DDR Memory 0) |
| **`tsens2`** | `0x0C251000` | `0x0C224000` | `0x0000FFFD` (`11` sensors) | 11 (`s0`–`s10`) | `s0`: `aoss-2-thermal`<br>`s1`: `gpuss-0-thermal` (Adreno 690 GPU 0)<br>`s2`: `gpuss-1-thermal` (Adreno 690 GPU 1)<br>`s3`: `gpuss-2-thermal` (Adreno 690 GPU 2)<br>`s4`: `gpuss-3-thermal` (Adreno 690 GPU 3)<br>`s5`: `pcie-1-thermal` (PCIe 1)<br>`s6`: `mem-1-thermal` (DDR Memory 1)<br>`s7`: `audio-thermal` (LPASS / Audio)<br>`s8`: `video-thermal` (Venus / Iris Video)<br>`s9`: `nsp0-1-thermal` (NSP0 Compute DSP)<br>`s10`: *(enabled in SROT, unmapped)* |
| **`tsens3`** | `0x0C252000` | `0x0C225000` | `0x000001FD` (`5` sensors) | 5 (`s0`–`s4`) | `s0`: `aoss-3-thermal`<br>`s1`: `gpuss-4-thermal` (Adreno 690 GPU 4)<br>`s2`: `gpuss-5-thermal` (Adreno 690 GPU 5)<br>`s3`: `gpuss-6-thermal` (Adreno 690 GPU 6)<br>`s4`: `gpuss-7-thermal` (Adreno 690 GPU 7) |

**[POLICY]** **Thermal Trip Points**:
- CPU, NSP, Camera, PCIe, Audio, Video, and AOSS zones specify a critical trip point of `110.0 °C` (`110,000 m°C`, `0x1ADB0`) with `1.0 °C` (`1,000 m°C`, `0x3E8`) hysteresis.
- GPU zones (`gpuss-0` through `gpuss-7`) specify a passive cooling trip point at `85.0 °C` (`85,000 m°C`, `0x14C08`) with `1.0 °C` hysteresis and a critical trip point at `110.0 °C` (`110,000 m°C`, `0x1ADB0`).
- DDR memory zones (`mem-0`, `mem-1`) specify a hot trip point at `90.0 °C` (`90,000 m°C`, `0x15F90`) with `2.0 °C` (`2,000 m°C`, `0x7D0`) hysteresis.
