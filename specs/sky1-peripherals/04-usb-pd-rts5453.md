# 04: USB-C Power-Delivery Controller (Realtek RTS5453H)

The Orion O6 and the Orange Pi 6 Plus each carry two Realtek RTS5453H
USB Type-C power-delivery controllers. Each one runs its own firmware: it
handles CC detection, PD negotiation and DisplayPort alternate-mode entry
by itself. The host only reads the result over I²C and passes it on to
the USB controller, the USB/DP combo PHY and the DP transmitter.

This section extends sky1-dp-tx 03 §3.11.2, which records the status bits
the PHY side needs. The facts here agree with that section.

## 4.1 Board wiring

| Board | I²C bus | Address | Port index (`id`) | USB controller / combo PHY it drives | Interrupt |
|---|---|---|---|---|---|
| Orion O6 | FCH I2C1, 100 kHz | `0x30` | 0 | USB-C controller 0 / combo PHY 0 | S5 GPIO 0 line 8, level low |
| Orion O6 | FCH I2C1, 100 kHz | `0x31` | 2 | USB-C controller 2 / combo PHY 2 | same line (shared) |
| Orange Pi 6 Plus | FCH I2C7, 100 kHz | `0x30` | 0 | USB-C controller 0 / combo PHY 0 | S5 GPIO 0 line 7, level low |
| Orange Pi 6 Plus | FCH I2C7, 100 kHz | `0x31` | 1 | USB-C controller 1 / combo PHY 1 | same line (shared) |

- **[HW]** The interrupt line is active low and shared by both chips on a
  board (so it is presumably open-drain). It stays low while either chip
  has an unacknowledged event. A handler must service **both** chips (or each chip's handler
  must tolerate a call when its own chip has nothing pending).
- **[HW]** Use a level-low trigger. An edge trigger can lose the second
  chip's event if the line never goes high between them.
- **[HW]** The interrupt GPIO is in the always-on S5 domain and can wake
  the system (01-gpio §1.5).
- The Orion O6N uses a different PD controller (a TCPCI-compatible part)
  and is not covered here.

## 4.2 Firmware description

### 4.2.1 Devicetree (vendor and Sky1-Linux)

| Property | Meaning |
|---|---|
| `compatible = "realtek,rts5453h"` | This device |
| `reg` | 7-bit I²C address |
| `interrupts` (with `interrupt-parent` an S5 GPIO controller) | Line and trigger |
| `id` (u32, **required**) | Port index, used to pair the chip with its USB controller and PHY |
| One child node (the connector) | Holds the port capabilities and graph links |

Connector child properties:

| Property | Values on both boards |
|---|---|
| `data-role` | `"host"` |
| `power-role` | `"source"` |
| `try-power-role` | `"source"` |
| `port@0` endpoint | Links to the USB controller's role switch |
| `port@1` endpoint | Links to the combo PHY's orientation/lane switch |

### 4.2.2 ACPI

- **[ABI]** `_HID` `CIXH200D`, a child of the I²C controller device
  (`CIXH200B`). `_CRS` has `I2cSerialBusV2` (address `0x30` or `0x31`)
  and a `GpioInt` (level, active low, shared) on the S5 GPIO controller.
- **[ABI]** *(vendor)* The Sky1-Linux driver reads the same properties as
  in devicetree through the firmware-neutral property interface: `id`
  from `_DSD`, and a child node for the connector with `data-role`,
  `power-role` and `try-power-role` and graph links.

> OPEN QUESTION: The public ACPI notes say the PD controller's properties
> were not extracted from the SSDT. Whether the SSDT carries `id`, a
> connector child (as a `_DSD` hierarchical data extension or a child
> device) and graph links is unknown. Without `id` the Sky1-Linux driver
> refuses the device. A clean-room driver should derive the port index
> from the `_UID` or from the I²C address and bus, using §4.1.

## 4.3 Register interface

**[HW]** *(vendor)* The host uses plain I²C register access: a write of
one register-address byte, followed by data bytes (write) or a repeated
START and a read (read). Register addresses are 8 bits, `0x00`–`0x7F`.
The host never uses the chip's PD message or UCSI interfaces.

| Register | Direction | Length | Use |
|---|---|---|---|
| `0x5F` | Read | 6 bytes | Connection and alt-mode status (§4.3.1) |
| `0x50` | Write | 3 bytes | Interrupt and HPD acknowledge (§4.3.2) |

> OPEN QUESTION: The RTS5453 normally speaks a Realtek command protocol
> in which a write to an offset carries a length byte and the reply to a
> read starts with a length byte. Byte 0 of the `0x5F` status and byte 0
> (`0x02`) of the `0x50` write fit that model. The full command set,
> firmware revision dependencies and whether a command needs a status
> poll before the next one are unknown.

### 4.3.1 Status (`0x5F`, 6 bytes)

| Byte | Bit | Meaning |
|---|---|---|
| 0 | – | Not interpreted (probably the reply length) |
| 1 | 1 | Plug orientation: 1 = reversed (flipped), 0 = normal |
| 1 | 4 | USB 2.0 data connection present |
| 1 | 5 | USB 3 (SuperSpeed) data connection present |
| 1 | 7 | Data role: 1 = this port is the USB device, 0 = host |
| 2 | 0 | DisplayPort alternate mode entered |
| 2 | 1 | DP role (source/sink); defined, not used by the vendor driver |
| 2 | 4 | Debug-accessory mode |
| 2 | 6 | HPD IRQ (short HPD pulse) pending |
| 2 | 7 | HPD level: 1 = sink connected (HPD high) |
| 3 | 1:0 | Power role: `01` = source, `10` = sink, other = unchanged |
| 4, 5 | – | Not interpreted |

**[HW]** Bytes 1 and 2 both zero means nothing is attached.

### 4.3.2 Acknowledge (`0x50`)

**[HW]** *(vendor)* Write the register address `0x50` followed by three
bytes:

| Byte | Value |
|---|---|
| 0 | `0x02` (number of following bytes) |
| 1 | `0x04` (bit 2: acknowledge the interrupt) |
| 2 | `0x20` (bit 5: acknowledge the HPD IRQ) or `0x00` |

This matches sky1-dp-tx 03 §3.11.2 ("bit 2 interrupt ack, bit 5 HPD-IRQ
ack"), where the bit numbers are within bytes 1 and 2 of this payload.

## 4.4 Event handling

**[HW]** *(vendor)* On the interrupt (in a thread, because I²C sleeps):

1. Write the acknowledge with byte 2 = `0x00`. This releases the chip's
   interrupt output.
2. Read the 6-byte status.
3. Work out and apply the port state (§4.5).
4. If status byte 2 bit 6 (HPD IRQ) was set, write the acknowledge again
   with byte 2 = `0x20`.

- **[HW]** At start-up and after system resume, do steps 2–3 without an
  interrupt (the chip may have changed state while the host was not
  listening).
- **[POLICY]** *(vendor)* If the I²C bus reports the device has gone
  (shutdown), the handler disables the interrupt instead of retrying.

## 4.5 Decoding the status

### 4.5.1 Orientation → combo PHY

| Condition | Orientation sent to the PHY |
|---|---|
| Neither USB 3 nor DP connected | none (the PHY keeps its last value, sky1-dp-tx 03 §3.5.3) |
| Byte 1 bit 1 = 1 | reversed |
| otherwise | normal |

**[HW]** USB 2.0-only attachment does not use the SuperSpeed lanes, so no
orientation is needed for it.

### 4.5.2 Lane mode → combo PHY and DP controller

| USB 3 (byte 1 bit 5) | DP (byte 2 bit 0) | Mode | DP lanes |
|---|---|---|---|
| 1 | 1 | DP pin assignment **D**: USB 3 + DP | 2 |
| 0 | 1 | DP pin assignment **E** (vendor label): DP only | 4 |
| 1 | 0 | USB only | 0 |
| 0 | 0 | Safe (no alternate mode) | 0 |

- **[HW]** The PHY's response to each mode is in sky1-dp-tx 03 §3.5.1 and
  §3.11.1 (it treats C and E alike as 4-lane DP).
- **[POLICY]** *(vendor)* The status does not report the actual pin
  assignment, only "USB 3 present". The driver infers D or 4-lane DP from
  it. It labels the 4-lane case "E"; for a DFP_D receptacle the usual
  4-lane assignment is C. Only the lane count matters to the PHY.
- **[HW]** In debug-accessory mode (byte 2 bit 4) the vendor driver stops
  after setting orientation and changes nothing else.

### 4.5.3 HPD → DP controller

| Status | DP controller sees |
|---|---|
| DP connected and byte 2 bit 7 = 1 | HPD high |
| DP connected and byte 2 bit 7 = 0 | HPD low |
| DP connected and byte 2 bit 6 = 1 | HPD IRQ (short pulse): the sink wants attention (link status, MST messages) |
| DP not connected | HPD low |

**[HW]** *(vendor)* The sources show no separate HPD pin for DP over
USB-C. The vendor stack takes HPD only from this status and forwards it
to the DP transmitter (sky1-dp-tx 03 §3.11.3). A clean-room driver must do
the same.

### 4.5.4 USB data role → USB controller

| Status | USB controller role |
|---|---|
| Neither USB 2 nor USB 3 connected | none (controller may idle) |
| Byte 1 bit 7 = 1 | device (peripheral) |
| otherwise | host |

**[POLICY]** *(vendor)* The role is only pushed to the USB controller when
it changes.

### 4.5.5 Power role

**[HW]** Byte 3 bits 1:0 report whether the port is sourcing (`01`) or
sinking (`10`) VBUS. It is informational. The chip negotiates power by
itself.

## 4.6 What the host may not do

- **[HW]** *(vendor)* Data-role swap and power-role swap requested by the
  host are not supported through this interface. Requests must be
  refused.
- **[HW]** *(vendor)* The host cannot select an alternate mode or pin
  assignment. It only reports what the chip negotiated.

## 4.7 Type-C framework reporting

**[POLICY]** *(vendor)* What Linux tells its Type-C layer (informative):

| Item | Value |
|---|---|
| Port type, data, preferred role | From the connector `power-role`, `data-role`, `try-power-role` |
| Type-C revision | 1.2 |
| PD revision | 3.0 |
| Alternate mode offered | DisplayPort, SVID `0xFF01`, mode 1, DFP_D and UFP_D capable, pin assignments C, D, E |
| Partner | Registered while byte 1 or byte 2 is non-zero; no PD identity, no accessory |

## 4.8 Power management

- **[POLICY]** *(vendor)* On system suspend: disable the interrupt, and arm
  it as a wake source if wake is allowed. On resume: disarm, re-enable,
  and re-read the status (§4.4).
- **[POLICY]** The host does not power the chip down. It keeps running in
  suspend and may be supplying VBUS.
