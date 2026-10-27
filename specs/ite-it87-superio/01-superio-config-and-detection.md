# 01: Super I/O Configuration Access and Chip Detection

## 1. Configuration port pairs

**[HW]** Each ITE Super I/O answers on one of two LPC I/O port pairs. The first port of each pair is the **index** port and the next one is the **data** port:

| Pair | Index port | Data port | Entry key (four bytes, all written to the index port) |
|---|---|---|---|
| Primary | `0x2E` | `0x2F` | `0x87`, `0x01`, `0x55`, `0x55` |
| Secondary | `0x4E` | `0x4F` | `0x87`, `0x01`, `0x55`, `0xAA` |

**[HW]** After the entry key, a configuration register is read by writing its number to the index port and then reading the data port. A write works the same way, but the value goes to the data port.

**[HW]** To **exit** configuration mode, write `0x02` to configuration register `0x02` (index port `0x02`, then data port `0x02`).

**[HW]** A 16-bit configuration value is held in two consecutive registers, **most significant byte first** (register *n* holds bits 15:8 and register *n+1* holds bits 7:0).

**[POLICY]** Other OS components (watchdog, GPIO, other sensor drivers) may also use the same port pairs. The reference driver takes a lock shared with those users for the whole entry, access, exit sequence, because the index register keeps its value between cycles.

## 2. Global configuration registers

| Register | Name | Meaning |
|---|---|---|
| `0x02` | Configuration control | **[HW]** Writing `0x02` leaves configuration mode. |
| `0x07` | Logical device number (LDN) | **[HW]** Selects which logical device's registers appear at `0x30` and above. |
| `0x20` | Chip ID high byte | **[HW]** Upper byte of the device ID (for example `0x86` for IT8689E). |
| `0x21` | Chip ID low byte | **[HW]** Lower byte (for example `0x89`). |
| `0x22` | Chip revision | **[HW]** Bits 3:0 hold the silicon revision. Bits 7:4 are ignored. |

**[HW]** A read of `0xFFFF` from registers `0x20`/`0x21` means either no chip is present or the chip is not in configuration mode.

### 2.1 Device IDs in scope

| Device ID | Chip |
|---|---|
| `0x8689` | IT8689E |
| `0x8695` | IT87952E (the device ID does **not** match the part number) |

> OPEN QUESTION: `0x8696` (IT8696E) is not recognized by the reference driver. See the README.

## 3. Detection sequence

**[HW]** The IT87952E is **already in configuration mode** when it powers up, and must **never** be sent the exit command. Doing so can hang the system or make the chip at the other address unreachable. An IT8689E behaves conventionally: it needs the entry key and may be exited.

**[HW]** The following probe order is safe for both kinds of chip at each port pair:

1. Without sending any key, read the device ID (registers `0x20`, `0x21`).
2. If it reads `0xFFFF`, send that pair's entry key and read it again. If it still reads `0xFFFF`, no chip is present.
3. If the first read returned a valid ID for a chip that is *not* of the always-configured kind (an IT8689E), send the entry key now anyway.
4. Note whether you sent the entry key during this probe.
5. Select LDN `0x04` (the Environment Controller) and check the activation register (§4). If the EC is inactive, abandon this chip.
6. Read the EC base address (§4). If it is zero, abandon this chip.
7. Read the revision and the pin-mux registers (§5, §6).
8. Send the exit command **only** if step 2 or 3 sent the entry key.

**[HW]** Probe the primary pair (`0x2E`) first and the secondary pair (`0x4E`) second. If the secondary chip reports the same EC base address as the primary one, it is the same EC seen twice. Register it only once.

**[POLICY]** For later configuration-space accesses (for example the SMBus interlock in §6) the reference driver omits the entry key and the exit command on always-configured chips, and uses both on all others.

## 4. Environment Controller logical device (LDN `0x04`)

| Register | Meaning |
|---|---|
| `0x30` | **[HW]** Activation. Bit 0 = 1 means the EC is enabled. |
| `0x60`/`0x61` | **[HW]** EC I/O base address, MSB first. Clear the low 3 bits to get an 8-byte-aligned base. |
| `0xF3` | **[HW]** Special configuration. On IT8689E, bits 1 and 2 enable SMBus-side access to the EC (§6). |

**[HW]** The EC's own registers are reached through a second index/data pair at **base + 5** (index) and **base + 6** (data). Only these two ports are used. See section 02.

## 5. GPIO logical device (LDN `0x07`): pin-mux registers

**[HW]** Several fan tachometer and PWM pins are shared with GPIO and other functions. Board firmware sets the pin function in LDN `0x07`. A pin assigned to another function must be treated as missing, whatever the EC registers report.

### 5.1 IT8689E

| LDN 7 register | Bit | Condition | Effect |
|---|---|---|---|
| `0x25` | 6 | set | PWM5 is not available |
| `0x26` | 5 | **clear** | Fan 4 tachometer is not available |
| `0x26` | 4 | **clear** | Fan 5 tachometer is not available |
| `0x27` | 6 | set | PWM3 is not available |
| `0x27` | 7 | set | Fan 3 tachometer is not available |
| `0x28` | 2 | set | PWM4 is not available |
| `0x29` | 1 | set | PWM2 is not available |
| `0x29` | 2 | set | Fan 2 tachometer is not available |
| `0x29` | 7 | **clear** | Both PWM6 and fan 6 are not available |
| `0xF6` | 5:0 | non-zero | A beeper output pin is assigned, so the EC beep-enable register is meaningful |

**[HW]** On IT8689E, voltage input 9 is always the internal AVCC3 rail, so there's nothing to check.

**[HW]** Section 02 §6 gives a second availability test for PWM5 and PWM6 on IT8689E, using an EC register.

### 5.2 IT87952E

| LDN 7 register | Bit | Condition | Effect |
|---|---|---|---|
| `0x27` | 6 | set | PWM3 is not available |
| `0x27` | 7 | set | Fan 3 tachometer is not available |
| `0x29` | 1 | set | PWM2 is not available |
| `0x29` | 2 | set | Fan 2 tachometer is not available |
| `0x2C` | 0 | set | Voltage input 3 is internally connected to AVCC (scaled, see 02 §3) |
| `0x2C` | 1 | set | Voltage input 7 is internally connected to the standby rail (on this chip it is always internal) |
| `0xF6` | 5:0 | non-zero | A beeper pin is assigned |

**[HW]** The IT87952E has no fan 4–6, no PWM 4–6, and no voltage input 9.

> OPEN QUESTION: the IT87952E's VID input pins (LDN 7 `0x27` bits 3:0) are unused because the chip has no VID function. The reference driver reads but ignores them.

## 6. SMBus interlock (IT8689E)

**[HW]** On IT8689E, the bits of LDN 4 register `0xF3` covered by the mask `0x06` (bits 1 and 2) enable the chip's SMBus-side paths into the Environment Controller. While they are set, host LPC accesses to the EC index/data ports can collide with SMBus accesses. A driver must:

1. During detection, read LDN 4 `0xF3` and save the whole value. Also note which of bits 1 and 2 are set.
2. If either bit was set, then before **every** group of EC register accesses: select LDN 4 and write the saved value with bits 1 and 2 cleared.
3. After the group of accesses, write the saved value back unchanged.

**[HW]** Bracket the whole of initialization, every periodic sensor read and every user-initiated register write this way. The IT87952E has no such interlock.

> OPEN QUESTION: what drives the SMBus side is not documented: board firmware, an embedded controller, or an external BMC. The reference driver assumes it is safe to suspend those paths briefly.

## 7. ACPI resource ownership

**[HW]** Board firmware may declare an ACPI operation region over the EC's base+5/base+6 ports, so that AML methods (for example the WMI methods in section 04, or fan curves in the setup UI) read the EC directly. If such a region exists, firmware and the OS may both use the index/data pair at once and corrupt each other's cycles.

**[POLICY]** By default, the reference driver refuses to attach to a chip whose EC ports overlap an ACPI-declared resource, unless the user overrides this. A clean-room driver should check the ACPI namespace in the same way and default to read-only operation, or decline to attach, when it finds a conflict.

## 8. Silicon revision adjustments

**[HW]** There are no revision-specific behaviors for IT8689E or IT87952E. (Older family members change their fan counter width by revision, but that does not apply here.)
