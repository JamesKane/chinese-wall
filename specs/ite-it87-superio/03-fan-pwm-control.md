# 03: Fan PWM Control

Applies to IT8689E (PWM channels 1–6) and IT87952E (PWM channels 1–3). Both chips use the model in which each channel has a mode/source register plus a separate duty register. All registers are EC registers (section 02, §1).

## 1. Per-channel registers

| Channel | PWM control | Automatic block base (*B*) | Duty / start PWM (*B* + 3) |
|---|---|---|---|
| 1 | `0x15` | `0x60` | `0x63` |
| 2 | `0x16` | `0x68` | `0x6B` |
| 3 | `0x17` | `0x70` | `0x73` |
| 4 | `0x7F` | `0x78` | `0x7B` |
| 5 | `0xA7` | `0xA0` | `0xA3` |
| 6 | `0xAF` | `0xA8` | `0xAB` |

### 1.1 PWM control register

| Bits | Meaning |
|---|---|
| 7 | **[HW]** 1 = automatic (temperature-driven) mode. 0 = manual mode, where the output follows the duty register |
| 1:0 | **[HW]** Temperature source for automatic mode (§3.1) |
| 6:2 | **[HW]** Preserve on every write |

### 1.2 Duty register (*B* + 3)

**[HW]** Full 8-bit duty cycle: `0x00` = 0 % and `0xFF` = 100 %. In manual mode it sets the output directly. In automatic mode the same register is the **start PWM** value, the duty the controller applies at the start temperature (§3.2).

**[POLICY]** While a channel is in automatic mode, the reference driver refuses manual duty writes, to avoid confusion, but still lets the user write the start-PWM value. Both writes go to the same address.

> OPEN QUESTION: the reference driver describes the duty register as read-only in automatic mode, yet it writes the start PWM there in that mode. Whether the chip ignores host writes to *B* + 3 while bit 7 is set has not been confirmed.

## 2. Fans 1–3: on/off mode vs PWM mode

**[HW]** Channels 1–3 have an extra gate. Register `0x13` (fan main control) bits 0, 1, 2 select the mode for channels 1, 2, 3:

| `0x13` bit *n* | Channel *n*+1 behavior |
|---|---|
| 0 | **On/off mode**: the output is fully on or fully off according to register `0x14` bit *n* (1 = on). The PWM control and duty registers have no effect |
| 1 | **PWM mode**: the output follows the PWM control register (manual or automatic) |

**[HW]** Bits 6:4 of `0x13` are the fan 1–3 tachometer enables (section 02, §5.1). Preserve them when changing bits 2:0.

**[HW]** Channels 4–6 have no on/off mode. To force full speed there, use manual mode with duty `0xFF`.

**[POLICY]** How the reference driver maps its three user-visible modes:

| User mode | Channels 1–3 | Channels 4–6 |
|---|---|---|
| Full speed | Set `0x14` bit *n*, then clear `0x13` bit *n* | Write duty `0xFF`, then clear control bit 7 |
| Manual | Clear control bit 7 (keeping the source), then set `0x13` bit *n* | Clear control bit 7 |
| Automatic | Set control bit 7 (keeping the source), then set `0x13` bit *n* | Set control bit 7 |

## 3. Automatic mode

### 3.1 Temperature source

**[HW]** Control register bits 1:0 select one of three temperature inputs:

| Channels | Field value 0, 1, 2 selects |
|---|---|
| 1–3 | temp1, temp2, temp3 |
| 4–6 (IT8689E) | temp4, temp5, temp6 |

**[HW]** A field value of 3 is not a valid source.

> OPEN QUESTION: the mapping for channels 4–6 (field value 0 means temp4) is the reference driver's interpretation for chips with a three-entry map and six channels. It has not been confirmed against hardware.

### 3.2 Automatic block layout

**[HW]** For each channel with block base *B*:

| Register | Meaning | Encoding |
|---|---|---|
| *B* + 0 | Fan-off temperature: below this the fan stops (or runs at minimum) | Signed 8-bit °C |
| *B* + 1 | Start temperature: the fan starts at the start PWM | Signed 8-bit °C |
| *B* + 2 | Full-speed temperature: at or above this the output is 100 % | Signed 8-bit °C |
| *B* + 3 | Start PWM (shared with the manual duty register) | 8-bit duty |
| *B* + 4 | Slope: bits 6:0 give the duty increase per °C above the start temperature, in units of 1/8 of a PWM step. Preserve bit 7 | Unsigned |
| *B* + 5 | Hysteresis: bits 4:0 give how far, in °C, the temperature must fall below the fan-off temperature before the fan actually turns off. Preserve bits 7:5 | Unsigned |

**[HW]** The temperatures must satisfy off ≤ start ≤ full-speed for the controller to behave sensibly.

**[POLICY]** The reference driver refuses to enter automatic mode unless that ordering holds. It presents a fourth curve point fixed at 100 % duty.

> OPEN QUESTION: the meaning of *B* + 4 bit 7 and *B* + 5 bits 7:5 is undocumented. Read-modify-write them unchanged.

## 4. PWM output frequency

**[HW]** A 3-bit frequency-select field chooses a base clock. The PWM output frequency is that clock divided by **256**:

| Field value | Base clock | Output frequency |
|---|---|---|
| 0 | 48 MHz | 187.5 kHz |
| 1 | 24 MHz | 93.75 kHz |
| 2 | 12 MHz | 46.875 kHz |
| 3 | 8 MHz | 31.25 kHz |
| 4 | 6 MHz | 23.44 kHz |
| 5 | 3 MHz | 11.72 kHz |
| 6 | 1.5 MHz | 5.86 kHz |
| 7 | 750 kHz | 2.93 kHz |

**[HW]** Field locations:

| Select | Register | Bits | Applies to |
|---|---|---|---|
| Frequency select 1 | `0x14` | 6:4 | Channel 1 (and every channel without its own select) |
| Frequency select 2 | `0x55` | 6:4 | Channel 2. **IT8689E only** |

**[HW]** On IT87952E there is a single frequency select, `0x14` bits 6:4. Bit 7 of `0x55` has its own meaning (section 02, §4.2), so preserve it when writing bits 6:4.

> OPEN QUESTION: some datasheets reportedly give 51 kHz instead of 750 kHz for field value 7 (about 200 Hz output). It is also unclear which select governs channels 3–6 on IT8689E. The reference driver reports select 1 for them.

## 5. Fan control register `0x14` summary

| Bits | Meaning |
|---|---|
| 7 | **[HW]** PWM output polarity. 1 = active high (the normal setting) |
| 6:4 | **[HW]** Frequency select 1 (§4) |
| 3 | **[HW]** Preserve |
| 2:0 | **[HW]** Output state for channels 1–3 in on/off mode (§2) |

## 6. PWM polarity sanity check

**[HW]** If `0x14` reads with bits 7 and 2:0 all clear (`value & 0x87 == 0`), the outputs are active-low and every on/off fan is off. The reference driver regards this as a sign that board firmware never configured the fan controller.

**[POLICY]** In that case the reference driver turns off its PWM interface (monitoring only). As an explicit, documented-as-dangerous user option, it can repair the setting instead. The repair happens only if none of channels 1–3 is in automatic mode:

1. Set `0x14` bits 7 and 2:0.
2. For each channel 1–3, invert its old duty value bitwise and keep the result's low 7 bits (`0x7F` minus the old value's low 7 bits), then write it to the channel's control register.

> OPEN QUESTION: the repair step writes the inverted duty to the control register (the older register model), not to *B* + 3. This repair path may not be correct for IT8689E or IT87952E. Prefer the monitoring-only fallback.

## 7. Initialization defaults

**[POLICY]** The reference driver does not change the fan configuration that board firmware set up. It only reads the current mode, source and curve. A clean-room driver should do likewise unless the user asks for a change, so that fan behavior does not jump when the OS loads the driver.
