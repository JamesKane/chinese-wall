# 01: Identification and Register Access

## 1. PCI identity

**[HW]** Vendor ID `0x10EC`. The device IDs below belong to the 8125/8126/8127 family:

| Device ID | Family |
|---|---|
| `0x8125` | RTL8125 (all generations) |
| `0x3000` | 8125 family, alternate ID |
| `0x5000` | 8125 family, alternate ID |
| `0x8126` | RTL8126 |
| `0x8127` | RTL8127 |
| `0x0E10` | Matched by the reference driver; family membership not stated |

> OPEN QUESTION: which variants use device IDs `0x3000`, `0x5000` and `0x0E10` is not stated. Use the chip-ID register (§2), not the device ID, to tell variants apart.

**[HW]** Registers are memory-mapped through a BAR. All multi-byte registers are little-endian.

## 2. Chip identification

**[HW]** Read the 32-bit **TX configuration register** at MMIO `0x40`. If it reads `0xFFFFFFFF`, the device is not responding.

**[HW]** Form a 12-bit chip ID from bits 31:20 of that register. Mask it with `0x7CF`, which ignores chip-ID bits 11 and 5:4 (register bits 31 and 25:24), and compare:

| Masked chip ID | Chip | Firmware file for PHY patches (section 05) |
|---|---|---|
| `0x609` | RTL8125A | `rtl_nic/rtl8125a-3.fw` |
| `0x641` | RTL8125B | `rtl_nic/rtl8125b-2.fw` |
| `0x688` | RTL8125D (rev 1) | `rtl_nic/rtl8125d-1.fw` |
| `0x689` | RTL8125D (rev 2) | `rtl_nic/rtl8125d-2.fw` |
| `0x68A` | RTL8125K | `rtl_nic/rtl8125k-1.fw` |
| `0x68B` | RTL9151A | `rtl_nic/rtl9151a-1.fw` |
| `0x708` | RTL8125CP | `rtl_nic/rtl8125cp-1.fw` |
| `0x681` | RTL8125BP | `rtl_nic/rtl8125bp-2.fw` |
| `0x649` | RTL8126A (rev 2) | `rtl_nic/rtl8126a-2.fw` |
| `0x64A` | RTL8126A (rev 3) | `rtl_nic/rtl8126a-3.fw` |
| `0x6C9` | RTL8127A | `rtl_nic/rtl8127a-1.fw` |
| `0x7C8` | *Extended ID*: read §2.1 | |

**[HW]** The permissive driver recognizes only the masked chip IDs `0x609`, `0x641`, `0x649`, `0x64A`, `0x688`, `0x689` and `0x6C9`. It does **not** recognize `0x68A`, `0x68B`, `0x708`, `0x681` or the extended ID.

### 2.1 Extended chip ID

**[HW]** If the masked chip ID is `0x7C8`, read the 32-bit register at MMIO `0x60B0`. Mask it with `0x7FFFFFFF`. A value of `0x00000000` identifies **RTL9151AS**, which is handled as an 8125 generation-3 part and uses `rtl_nic/rtl9151a-1.fw`.

**[HW]** In the rest of this suite, "gen 3" means the chip IDs `0x688`/`0x689`/`0x68A`/`0x68B`, plus RTL9151AS and RTL8125CP/RTL8125BP, unless a row says otherwise.

## 3. Factory MAC address

**[HW]** On 8125-family parts, a backup copy of the factory station address is at MMIO `0x19E0`–`0x19E5`, in byte order. Use it if platform firmware supplies no address. If it is invalid, read the live station-address registers at `0x00`–`0x05`.

## 4. Configuration-register lock

**[HW]** Configuration registers `Config0`–`Config5` (`0x51`–`0x56`) can be written only while register `0x50` holds `0xC0`. Write `0x00` there to lock them again.

## 5. MAC OCP window

**[HW]** Most 8125-family tuning registers live in a 16-bit "MAC OCP" address space, reached through the 32-bit window register at MMIO `0xB0`:

| Operation | Value written to `0xB0` | Then |
|---|---|---|
| Write | bit 31 set, OCP address shifted left 15 bits, 16-bit data in bits 15:0 | none |
| Read | OCP address shifted left 15 bits, bit 31 clear | Read `0xB0`. Bits 15:0 are the data |

**[HW]** OCP addresses are even and lie in `0x0000`–`0xFFFE`. An odd address, or one above 16 bits, is invalid. A MAC OCP read-modify-write must not be interrupted by another MAC OCP access.

## 6. PHY OCP window and paged PHY access

**[HW]** The integrated PHY's registers sit in a separate 16-bit "PHY OCP" space, reached through MMIO `0xB8`, laid out like `0xB0`. Writes and reads complete asynchronously:

- Write: write `0xB8` with bit 31 set, then poll until bit 31 reads 0.
- Read: write `0xB8` with bit 31 clear, then poll until bit 31 reads 1. The data is in bits 15:0.

**[POLICY]** The reference driver polls at 25 µs intervals, 10 times.

**[HW]** Standard Clause 22 PHY registers 0–15 map to PHY OCP `0xA400 + 2 × reg`.

**[HW]** Realtek's paged access (page number written to Clause 22 register `0x1F`) maps page *p*, register *r* (with *r* ≥ `0x10`) to PHY OCP address `(p << 4) + 2 × (r − 0x10)`. Page 0 selects the standard base `0xA400`.

**[HW]** For Clause 45 access to vendor MMD 31 (Vendor 2), the register number **is** the PHY OCP address. Registers at or below the standard status-2 register are not mapped.

**[HW]** Only PHY address 0 exists on the internal MDIO bus.

**[HW]** "Indexed PHY parameters" are reached in two ways:

| Variant | Index register | Data register |
|---|---|---|
| A | PHY OCP `0xB87C` | PHY OCP `0xB87E` |
| B | Page `0xA43`, register `0x13` | Page `0xA43`, register `0x14` |

## 7. Doorbell and interrupt registers (8125 layout)

| MMIO | Width | Use |
|---|---|---|
| `0x34` | 8 | Interrupt configuration 0. Bit 0 enables per-vector interrupt mapping. Bit 3 enables CLKREQ on RTL8126/RTL8127 |
| `0x38` | 32 | Interrupt mask (single-vector mode) |
| `0x3C` | 32 | Interrupt status (single-vector mode). Write 1 to clear |
| `0x7A` | 16 | Interrupt configuration 1 |
| `0x90` | 16 | TX doorbell. Write bit 0 to poll the normal-priority queue |

**[HW]** Single-vector status and mask bits: RX OK = bit 0, RX error = bit 1, TX OK = bit 2, TX error = bit 3, link change = bit 5. A status read whose low 16 bits are all ones means the device is gone (surprise removal).

**[HW]** Section 04 covers the multi-vector registers.
