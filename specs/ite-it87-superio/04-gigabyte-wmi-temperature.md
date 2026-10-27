# 04: Gigabyte WMI Temperature Interface

**[ABI]** Many Gigabyte desktop boards' ACPI firmware exposes a WMI method object that returns temperatures the firmware reads itself, through its own access to the Super I/O EC or other sensors. Using it avoids touching the EC ports directly, which is useful when the EC ports are claimed by ACPI (section 01, §7).

## 1. Identification

| Item | Value |
|---|---|
| WMI GUID | `DEADBEEF-2001-0000-00A0-C90629100000` |
| Object type | Method object, evaluated with instance `0` |

**[ABI]** A driver locates this GUID in the board's ACPI WMI mapping table (the `_WDG` data of the WMI pseudo-device, PNP ID `PNP0C14`), and evaluates the matching `WMxx` method with: instance 0, method ID = command code, and an input buffer.

> OPEN QUESTION: whether the X870E reference board's firmware contains this GUID has not been checked. Confirm it from the board's ACPI table dump before implementing.

## 2. Commands

| Command code | Purpose | Argument | Result |
|---|---|---|---|
| `0x0001` | Firmware build date | — | Not characterized |
| `0x0002` | Mainboard type | — | Not characterized |
| `0x0004` | Firmware version | — | Not characterized |
| `0x0005` | Mainboard name | — | Not characterized |
| `0x0125` | Temperature read | Sensor index | ACPI Integer |

**[ABI]** The input buffer is 4 bytes: one little-endian 32-bit unsigned argument. For the temperature command it holds the sensor index `0`–`5`.

## 3. Temperature result

**[ABI]** The temperature command returns an ACPI Integer object. Only its least-significant byte matters: interpret it as a **signed 8-bit** value in degrees Celsius.

**[ABI]** A result of `0` means that sensor index is not present or not populated on this board.

**[HW]** Any result that is not an Integer object is an error.

## 4. Discovery

**[POLICY]** The reference driver queries all six indices once when it attaches, and offers only those that return a non-zero value. If none do, it declines to attach.

> OPEN QUESTION: which physical sensor each index corresponds to (for example CPU, system, chipset, VRM, PCIe slot) depends on the board and is not reported by the interface. Match the values against the firmware setup screen.

> OPEN QUESTION: the result formats of commands `0x0001`–`0x0005` are not known (they are probably strings or buffers). No driver needs them for temperature monitoring.
