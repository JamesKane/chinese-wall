# 02: Environment Controller Sensor Registers

## 1. Register access

**[HW]** All EC registers are 8 bits wide and are reached indirectly: write the register number to port **base + 5**, then read or write port **base + 6** (base comes from section 01, §4). There is no auto-increment.

**[POLICY]** The reference driver ignores the EC's busy indication, and inserts a short I/O delay (one legacy "pause" port cycle) after each port access.

**[HW]** On IT8689E, every group of EC accesses must sit inside the SMBus interlock described in section 01, §6.

## 2. Identification and start-up

| Register | Name | Use |
|---|---|---|
| `0x00` | Configuration | **[HW]** Bit 0: start monitoring. Bit 5: write 1 to clear the chassis-intrusion latch. Bit 6: write 1 to request a fresh battery-voltage sample (the bit clears itself once the sample is taken). Bit 7: must read 0 on a working, initialized EC. |
| `0x58` | Vendor ID | **[HW]** Reads `0x90` on ITE environment controllers. |

**[HW]** Treat the chip as absent if configuration bit 7 is set or the vendor ID is not `0x90`.

**[HW]** To start monitoring: read configuration, keep bits 5:1 (mask `0x3E`), set bit 0, optionally set bit 6, and write the result back.

**[POLICY]** The reference driver sets bit 6 only if the user asks for live battery readings. Otherwise the battery value it reports is the one sampled at power-up.

### 2.1 Fix-ups for unconfigured firmware state

**[POLICY]** Some firmware leaves limits and enables at their reset values. The reference driver corrects this at initialization and again after every resume:

| Condition | Action |
|---|---|
| A voltage low-limit register (inputs 0–7) reads `0xFF` | Write `0x00`, so the alarm does not fire spuriously |
| A temperature high-limit register (inputs 1–3) reads `0xFF` (−1 °C) | Write `0x7F` (127 °C) |
| Voltage enable register `0x50` reads `0x00` | Write `0xFF` (enable all of inputs 0–7) |
| Fan tachometer enables (register `0x13` bits 6:4, after removing pins that are muxed away) are all clear | Set them |

## 3. Voltage inputs

### 3.1 Register map

| Input | Reading | High limit | Low limit | Notes |
|---|---|---|---|---|
| in0 | `0x20` | `0x30` | `0x31` | |
| in1 | `0x21` | `0x32` | `0x33` | |
| in2 | `0x22` | `0x34` | `0x35` | |
| in3 | `0x23` | `0x36` | `0x37` | May be internal AVCC (IT87952E, 01 §5.2) |
| in4 | `0x24` | `0x38` | `0x39` | |
| in5 | `0x25` | `0x3A` | `0x3B` | |
| in6 | `0x26` | `0x3C` | `0x3D` | |
| in7 | `0x27` | `0x3E` | `0x3F` | Internal standby rail on both chips |
| in8 | `0x28` | none | none | Internal battery (VBAT) on both chips |
| in9 | `0x2F` | none | none | Internal AVCC3. IT8689E only |
| in10 | `0x2C` | none | none | IT8689E only, if selected (§4.3) |
| in11 | `0x2D` | none | none | IT8689E only, if selected (§4.3) |
| in12 | `0x2E` | none | none | IT8689E only, if selected (§4.3) |

**[HW]** Register `0x50` holds an enable bit for each of in0–in7 (bit *n* = in*n*).

### 3.2 Conversion

**[HW]** The voltage at the pin is the register value (0–255) times the ADC step:

| Chip | ADC step |
|---|---|
| IT8689E | 12.0 mV |
| IT87952E | 10.9 mV |

**[HW]** Internally connected rails pass through an on-chip divide-by-two, so their step is **twice** the table value:

| Chip | Inputs with the doubled step |
|---|---|
| IT8689E | in7, in8, in9 |
| IT87952E | in7, in8, plus in3 when LDN 7 `0x2C` bit 0 is set |

**[HW]** Limits use the same encoding as the reading for that input.

**[POLICY]** Labels the reference driver gives the internal rails on these chips: in3 (when internal) "+3.3V", in7 "3VSB", in8 "Vbat", in9 "+3.3V". The external inputs in0–in6 depend on board resistor dividers. Their scaling and meaning are board-specific and are not stored in the chip.

> OPEN QUESTION: the X870E board's external divider ratios for in0–in6 (for example Vcore, +12V, +5V, DRAM) are not known. They would have to come from board firmware's setup-screen readouts or from measurement.

## 4. Temperature inputs

### 4.1 Register map

| Input | Reading | High limit | Low limit | Offset |
|---|---|---|---|---|
| temp1 | `0x29` | `0x40` | `0x41` | `0x56` |
| temp2 | `0x2A` | `0x42` | `0x43` | `0x57` |
| temp3 | `0x2B` | `0x44` | `0x45` | `0x59` |
| temp4 | `0x2C` | none | none | none |
| temp5 | `0x2D` | none | none | none |
| temp6 | `0x2E` | none | none | none |

**[HW]** All temperature values are signed 8-bit two's complement, in whole degrees Celsius.

**[HW]** temp4–temp6 exist only on IT8689E. Their addresses are shared with voltage inputs in10–in12 (§4.3).

**[HW]** Before writing an offset register, set bit 7 of the beep-enable register `0x5C` (keep the other bits). The offset registers ignore writes otherwise.

> OPEN QUESTION: the meaning of `0x5C` bit 7 as a gate for offset-register writes comes from driver behavior, not from documentation.

### 4.2 Sensor type selection (temp1–temp3)

**[HW]** Register `0x51` (temperature enable) selects the sensing method for temp1–3:

| Bits | Meaning |
|---|---|
| bit *n* (*n* = 0..2) | temp*(n+1)* uses a thermal diode |
| bit *n*+3 (*n* = 0..2) | temp*(n+1)* uses a thermistor |
| bits 7:6 = *k* (1..3) | temp*k* takes its value from the digital CPU interface (PECI or AMD-TSI) instead of the analog pin. 0 = no digital source |

**[HW]** On IT87952E, a second, older way to select a digital source also exists: register `0x55` bit 7 set means **temp2** takes its value from the digital interface.

**[HW]** Register `0x0A` bits 6:4 select the digital CPU interface protocol. The value `0b100` means **AMD SB-TSI**. Other values mean Intel PECI. On an AM5 board, a temperature input with a digital source therefore reports the CPU temperature over SB-TSI when the field is `0b100`.

**[HW]** If none of the type bits apply to an input, it is disabled.

> OPEN QUESTION: the reference driver supports the digital-source fields (bits 7:6 of `0x51`) only on IT87952E, not on IT8689E. Whether IT8689E also implements them is unknown. A read-only check of `0x51` bits 7:6 and `0x0A` bits 6:4 on the X870E would settle this.

**[POLICY]** The reference driver never enables a temperature input on its own initiative, because it cannot tell whether a diode or a thermistor is fitted. The user must choose.

### 4.3 Shared temperature/voltage inputs (IT8689E)

**[HW]** Register `0x77` holds three 2-bit fields that decide the role of registers `0x2C`–`0x2E`:

| Field | Register address governed |
|---|---|
| bits 1:0 | `0x2C` (temp4 / in10) |
| bits 3:2 | `0x2D` (temp5 / in11) |
| bits 5:4 | `0x2E` (temp6 / in12) |

| Field value | Role |
|---|---|
| `0` | Unused |
| `1` | Voltage input (in10/in11/in12) |
| `2` or `3` | Temperature input (temp4/temp5/temp6) |

> OPEN QUESTION: what distinguishes field values 2 and 3 (probably diode vs thermistor) has not been confirmed.

## 5. Fan tachometers

**[HW]** Every fan counter on IT8689E and IT87952E is always 16 bits wide. There is no divisor to program.

| Fan | Count low | Count high | Limit low | Limit high | Exists on |
|---|---|---|---|---|---|
| 1 | `0x0D` | `0x18` | `0x10` | `0x1B` | both |
| 2 | `0x0E` | `0x19` | `0x11` | `0x1C` | both |
| 3 | `0x0F` | `0x1A` | `0x12` | `0x1D` | both |
| 4 | `0x80` | `0x81` | `0x84` | `0x85` | IT8689E |
| 5 | `0x82` | `0x83` | `0x86` | `0x87` | IT8689E |
| 6 | `0x4C` | `0x4D` | `0x4E` | `0x4F` | IT8689E |

**[HW]** Speed in RPM = 1,350,000 ÷ (count × 2), where *count* is the 16-bit value. A count of `0xFFFF` means the fan is stopped or below the measurable range (report 0). A count of `0x0000` is invalid. The low-speed limit uses the same encoding (a limit of 0 RPM is written as `0xFFFF`), and the valid range for a written limit is `0x0001`–`0xFFFE`.

**[POLICY]** The reference driver reads the low byte before the high byte. Nothing indicates the chip latches the high byte when the low byte is read.

> OPEN QUESTION: whether reading the low byte latches the high byte is unknown. A driver can avoid a torn read by reading low, high, low and retrying if the two low readings differ.

### 5.1 Which fans exist

**[HW]** A fan is present only if its tachometer is enabled **and** its pin is not muxed away (section 01, §5):

| Fan | Enable bit |
|---|---|
| 1, 2, 3 | Register `0x13` bits 4, 5, 6 |
| 4 | Register `0x0C` bit 4 (IT8689E) |
| 5 | Register `0x0C` bit 5 (IT8689E) |
| 6 | Register `0x0C` bit 2 (IT8689E) |

## 6. Additional PWM5/PWM6 availability test (IT8689E)

**[HW]** On top of the pin-mux checks in section 01, §5.1, EC register `0x0B` reports:

- bits 7:6 both set: PWM5 is not available
- bit 3 clear: PWM6 is not available

> OPEN QUESTION: the reference driver applies this test to all six-PWM family members, and says it may only be valid for one older chip. Confirm it on the X870E before relying on it.

## 7. Alarms

**[HW]** Three alarm status registers latch when an input crosses its limit:

| Register | Bit | Source |
|---|---|---|
| `0x01` | 0, 1, 2, 3 | Fan 1, 2, 3, 4 below limit |
| `0x01` | 4 | Chassis intrusion |
| `0x01` | 6, 7 | Fan 5, 6 below limit |
| `0x02` | 0–7 | in0–in7 outside limits |
| `0x03` | 0, 1, 2 | temp1, temp2, temp3 outside limits |

**[HW]** To clear the chassis-intrusion latch, write 1 to configuration register `0x00` bit 5 (keeping the other bits).

> OPEN QUESTION: whether reading the alarm registers clears them is not established. The reference driver re-reads them every polling cycle without ever writing them.

## 8. Beep enables

**[HW]** Register `0x5C` holds global beep enables. They work only if a beeper pin is assigned (LDN 7 `0xF6` bits 5:0 non-zero):

| Bit | Beep on |
|---|---|
| 0 | Any fan alarm |
| 1 | Any voltage alarm |
| 2 | Any temperature alarm |
| 7 | See §4.1 (offset-register write gate) |

**[HW]** There is one enable per class, not one per input.

## 9. Polling and resume

**[POLICY]** The reference driver caches all readings and refreshes them at most once every 1.5 s.

**[HW]** On resume from suspend, the EC's settings can't be assumed to survive. Repeat the fix-ups in §2.1, the PWM polarity check (section 03, §6) and the monitoring start (§2), then re-read everything.
