# 05: PHY Patch Firmware Format and LED Control

## 1. Firmware file

**[ABI]** Realtek distributes PHY/MAC patch programs as files named `rtl_nic/<chip>.fw` (section 01, §2 lists the names). They are published with the standard Linux firmware collection under a redistribution licence. A driver may either load these files or embed equivalent patch tables, as the permissive driver does.

### 1.1 Container layouts

**[ABI]** A file has one of two layouts, chosen by its first 32-bit little-endian word:

**Headed layout** (first word = 0):

| Offset | Size | Field |
|---|---|---|
| 0 | 4 | Magic (0) |
| 4 | 32 | Version string, NUL-padded |
| 36 | 4 | Byte offset of the program (little-endian) |
| 40 | 4 | Program length, in **4-byte instructions** (little-endian) |
| 44 | 1 | Checksum byte |

**[ABI]** Validation for the headed layout:

- the 8-bit sum of **every byte in the file** must be 0
- the program offset must not exceed the file size
- the program must fit within the file

**Bare layout** (first word ≠ 0): the whole file is the program. The file size must be a multiple of 4.

### 1.2 Instruction encoding

**[ABI]** Each instruction is a 32-bit little-endian word:

| Bits | Field |
|---|---|
| 31:28 | Opcode |
| 27:16 | Register / count operand *R* (12 bits) |
| 15:0 | Data operand *D* |

**[ABI]** An interpreter keeps three pieces of state: an accumulator *A* (16-bit), a read counter *C*, and a target selector (PHY or MAC MCU). Opcodes:

| Opcode | Effect |
|---|---|
| `0x0` | *A* ← read target register *R*. Then *C* ← *C* + 1 |
| `0x1` | *A* ← *A* OR *D* |
| `0x2` | *A* ← *A* AND *D* |
| `0x3` | Jump backwards: the next instruction executed is the one *R* places before this one (position − *R*). Valid only if *R* ≤ the current position |
| `0x4` | Select target: *D* = 0 selects PHY, *D* = 1 selects MAC MCU. Other values are invalid |
| `0x7` | *C* ← 0 |
| `0x8` | Write *D* to target register *R* |
| `0x9` | If *C* = *D*, skip the next instruction. At least two instructions must follow |
| `0xA` | If *A* = *D*, skip the next *R* instructions |
| `0xB` | If *A* ≠ *D*, skip the next *R* instructions |
| `0xC` | Write *A* to target register *R* |
| `0xD` | Skip the next *R* instructions |
| `0xE` | Delay *D* milliseconds |
| others | Invalid. Reject the whole file |

**[ABI]** Before running anything, check every skip and jump against the program bounds, and reject the file if any of them falls outside.

### 1.3 Target register semantics

**[HW]** **PHY target:** the register number is a Clause 22 register on the paged PHY (section 01, §6). Writing register `0x1F` selects the page: page *p* ≠ 0 sets the OCP base to *p* × 16, and page 0 restores the standard base.

**[HW]** **MAC MCU target:** writing register `0x1F` with value *v* sets a base of *v* × 16. Any other register *r* is MAC OCP address (base + *r*).

**[HW]** When the program ends, reset the PHY page base to standard. Then wait until the PHY's basic-control reset bit (Clause 22 register 0, bit 15) clears.

**[POLICY]** The reference driver allows up to 600 ms, polling every 50 ms.

### 1.4 When to run it

**[HW]** Run the patch program first in PHY configuration, before any other PHY tuning. On 8125 gen 3 and 8126A, the tuning that follows is short:

1. Page `0xA44` register `0x11`: set bit 11 (enables 10 M in the giga PHY).
2. Page `0xA5B` register `0x12`: clear bit 15 (legacy forced-mode behavior).
3. Page `0xA43` register `0x10`: clear bit 2 (turns ALDPS power saving off).
4. Configure EEE on the PHY side (section 02, §5.2).

**[HW]** RTL8125CP and RTL8125BP need extra parameter writes. For RTL8125CP:

- Page `0xAD0` register `0x17`: clear `0x007F`, set `0x000B`.
- Page `0xAD7` register `0x14`: set bit 4.
- Variant-A parameter `0x807F`: clear `0xFF00`, set `0x5300`.
- Variant-B parameters: `0x81B8` = `0x00B4`, `0x81BA` = `0x00E4`, `0x81C5` = `0x0104`, `0x81D0` = `0x054D`.
- Page `0xA43` register `0x10`: set the 10M-PLL-off and ALDPS-PLL-off bits.
- Page `0xA44` register `0x11`: set bit 7.

For RTL8125BP:

- Variant-B parameter `0x8010`: clear bit 11.
- Variant-A parameters `0x8088` and `0x808F`: clear `0xFF00`, set `0x9000`.
- Variant-B parameter `0x8174`: clear `0x2000`, set `0x1800`.

> OPEN QUESTION: the bit positions of the 10M-PLL-off and ALDPS-PLL-off bits in page `0xA43` register `0x10` were not recorded in this pass. Take them from the permissive driver's PHY definitions.

## 2. Hardware LED control (8125/8126)

**[HW]** There are four LED outputs. Each has a 16-bit select register:

| LED | MMIO |
|---|---|
| 0 | `0x18` |
| 1 | `0x86` |
| 2 | `0x84` |
| 3 | `0x96` |

**[HW]** Field mask `0x023F`. Preserve the bits outside it.

| Bit | LED lights on |
|---|---|
| 0 | Link at 10 M |
| 1 | Link at 100 M |
| 3 | Link at 1 G |
| 5 | Link at 2.5 G |
| 9 | Activity (blinks on TX or RX, which cannot be selected separately) |

**[HW]** Setting several link bits lights the LED at any of those speeds. A field value of 0 turns the LED off.

**[POLICY]** The reference driver rejects modes that separate TX from RX activity, or that depend on duplex.

> OPEN QUESTION: bits 2 and 4 are inside the field mask but unassigned. They may be the 5 G (RTL8126A) and 10 G link indications. The polarity and blink-rate controls are not documented.
