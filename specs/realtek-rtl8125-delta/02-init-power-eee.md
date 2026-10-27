# 02: Initialization, PCIe Power Management, WoL and EEE

Registers named "OCP `0xNNNN`" are MAC OCP registers (section 01, §5). "Modify OCP `R`: clear *M*, set *S*" means read `R`, clear the bits in *M*, OR in *S*, and write the result back.

## 1. One-time hardware initialization (after reset, before PHY bring-up)

**[HW]**

1. Gate the RX data path: set bit 19 of the 32-bit register at MMIO `0xF0`. Wait about 2 ms. Then wait for the TX and RX FIFOs to drain.
2. Clear the TX-enable and RX-enable bits (bits 2 and 3) of the command register at MMIO `0x37`. Wait 1 ms.
3. Clear bit 7 of MMIO `0xD3` (hands ownership from the out-of-band firmware to the host).
4. Modify OCP `0xE8DE`: clear bit 14.
5. Wait for MMIO `0xD3` bit 1 (link-list ready) to become 1.
6. Write OCP `0xC0AA` = `0x07D0`, OCP `0xC0A6` = `0x0150` and OCP `0xC01E` = `0x5555`.
7. Wait for MMIO `0xD3` bit 1 again.

**[POLICY]** The reference driver polls the link-list-ready bit every 100 µs, up to 42 times.

**[HW]** A software reset is a write of bit 4 to MMIO `0x37`, followed by waiting for that bit to clear. The reference driver allows 100 × 100 µs.

## 2. Per-start initialization (each time the interface comes up)

**[HW]** The order matters. Config registers must be unlocked (section 01, §4).

| Step | Action | Applies to |
|---|---|---|
| 1 | Turn off ASPM and CLKREQ in the chip (§3.3) before any PCIe-PHY access | all |
| 2 | Write the 16-bit C+ command register (MMIO `0xE0`) with the RX checksum and RX VLAN enables wanted | all |
| 3 | Program the EEE TX idle timer (§5) | all |
| 4 | Write MMIO `0x34` = `0x00` | all |
| 5 | Zero the interrupt-moderation table: 32-bit writes of 0 across `0xA00`–`0xAFF` | 8125A, gen 3, RTL8127A |
| 5′ | Zero `0xA00`–`0xA7F` instead, and write MMIO `0x7A` = `0x0000` | RTL8125B, RTL8126A |
| 5″ | Also write MMIO `0x7A` = `0x0000` | RTL8127A |
| 6 | Modify OCP `0xEA84`: set bits 1:0 (extended statistics counters) | all |
| 7 | Set the ASPM entry latency (§3.1) | 8125B, gen 3, 8126A, 8127A |
| 8 | Run the PCIe Gen3 quirk (§3.2) | 8126A |
| 9 | Run the common MAC table (§2.1) | all |
| 10 | Turn on the L1-exit triggers: modify OCP `0xC0AC`, set `0x1F80` | all |
| 11 | Turn ASPM/CLKREQ back on if allowed (§3.3) | all |
| 12 | Write the 16-bit RX max size register (MMIO `0xDA`) = `0x4000` (16 KiB, which effectively turns off size filtering) | all |
| 13 | Program the TX and RX descriptor ring bases (high dword before low): TX at `0x24`/`0x20`, RX queue 0 at `0xE8`/`0xE4`. Additional RX queues are in section 04 | all |
| 14 | Write RSS control (MMIO `0x4500`) = 0 when using a single queue. Otherwise program it as in section 04 | all |
| 15 | Lock the config registers. Set the PCIe max read request to 4096 bytes | all |
| 16 | Set the TX-enable and RX-enable bits in MMIO `0x37` | all |
| 17 | Write the RX configuration register (MMIO `0x44`): bits 31:27 = 8 (RX fetch count), bits 10:8 = 7 (unlimited DMA burst), and on all parts **except** 8125A also bit 11 (pause slot) | all |
| 18 | Write the TX configuration register (MMIO `0x40`): bits 10:8 = 7 (DMA burst), bits 25:24 = 3 (shortest inter-frame gap) | all |
| 19 | Program the RX filter and multicast hash. Unmask interrupts | all |

### 2.1 Common 8125-family MAC table (step 9)

**[HW]** Apply in this order:

| # | Register | Operation |
|---|---|---|
| 1 | MMIO `0x54` | Clear bit 1 (ready-to-L2/L3 off) |
| 2 | MMIO `0x382` (16-bit) | Write `0x221B` |
| 3 | MMIO `0x4500` (32-bit) | Write 0 |
| 4 | MMIO `0x4800` (16-bit) | Write 0 |
| 5 | MMIO `0x34` | Set bit 0 **only** if multiple interrupt vectors are in use (section 04) |
| 6 | OCP `0xD40A` | Clear bit 4 (disables UPS) |
| 7 | MMIO `0x52` | Clear bit 4 |
| 8 | OCP `0xC140`, `0xC142` | Write `0xFFFF` to each |
| 9 | OCP `0xD3E2` | Clear `0x0FFF`, set `0x03A9` |
| 10 | OCP `0xD3E4` | Clear `0x00FF` |
| 11 | OCP `0xE860` | Set `0x0080` |
| 12 | OCP `0xEB58` | Clear bit 0 (keeps the legacy TX descriptor format) |
| 13 | MMIO `0xD8` | Clear bit 1. RTL8126A/RTL8127A only |
| 14 | MMIO `0x4800`, MMIO `0xD8` | Multi-queue only: see section 04 |
| 15 | OCP `0xE614` | Clear `0x0700`, then set: `0x0200` (8125B), `0x0300` (8125A and gen 3), `0x0400` (8126A). For RTL8127A, clear and set `0x0F00` |
| 16 | OCP `0xE63E` | Clear `0x0C30`, then set `0x0020`, except on 8125B, which sets nothing |
| 17 | OCP `0xC0B4` | Set `0x000C` |
| 18 | OCP `0xEB6A` | Clear `0x00FF`, set `0x0033` |
| 19 | OCP `0xEB50` | Clear `0x03E0`, set `0x0040` |
| 20 | OCP `0xE056` | Clear `0x00F0` |
| 21 | OCP `0xE040` | Clear bit 12 |
| 22 | OCP `0xEA1C` | Clear `0x0003`, set `0x0001` |
| 23 | OCP `0xEA1C` | Clear `0x0300` (8126A/8127A) or clear bit 2 (others) |
| 24 | OCP `0xE0C0` | Clear `0x4F0F`, set `0x4403` |
| 25 | OCP `0xE052` | Clear `0x0080`, set `0x0068` |
| 26 | OCP `0xD430` | Clear `0x0FFF`, set `0x047F` |
| 27 | OCP `0xEA1C` | Clear bit 2 |
| 28 | OCP `0xEB54` | Set bit 0, wait 1 µs, clear bit 0 |
| 29 | MMIO `0x1880` (16-bit) | Clear bits 5:4 |
| 30 | OCP `0xE098` | Write `0xC302` |
| 31 | OCP `0xE00E` | Wait for bit 13 to clear (reference: 10 polls, 1 ms apart) |
| 32 | MMIO `0xF0` | Clear bit 19 (ungates the RX data path) |

**[HW]** RTL8125B also needs this PCIe-PHY table before the common table. Each entry is a PCIe-PHY register, a mask to clear, and a value to set: `0x0B` ← `0xA908`, `0x1E` ← `0x20EB`, `0x4B` ← `0xA908`, `0x5E` ← `0x20EB`, `0x22` clear `0x0030` set `0x0020`, `0x62` clear `0x0030` set `0x0020`. Gen 3 and 8126A need no PCIe-PHY table.

> OPEN QUESTION: none of the OCP registers in §2.1 is documented. The values are cross-checked against the permissive driver's initialization. Where the two differ, prefer the permissive driver, because it also loads the matching MAC microcode.

## 3. PCIe power management

### 3.1 ASPM entry latency

**[HW]** PCI configuration byte `0x70F` sets the ASPM entry latencies. Bits 2:0 give the L0s latency, (value + 1) µs, capped at 7 µs. Bits 5:3 give the L1 latency, roughly 2^value µs, capped at 64 µs. Write `0x27` (L0s 7 µs, L1 16 µs).

**[HW]** If extended configuration space is not reachable, use the chip's indirect configuration-space window instead (CSI: data at MMIO `0x64`, address and command at MMIO `0x68`). Modify the byte as the top byte of the dword at `0x70C`.

### 3.2 RTL8126A PCIe Gen3 quirk

**[HW]** Clear bit 0 of PCI configuration byte `0x890` (the Gen3 "zero RX DC non-compliance" setting). If extended configuration space is not reachable, use the CSI window.

### 3.3 ASPM and CLKREQ

**[HW]** To turn ASPM on in the chip:

1. Set bit 0 of Config5 (MMIO `0x56`).
2. Set CLKREQ: bit 3 of MMIO `0x34` on RTL8126A/RTL8127A, or bit 7 of Config2 (MMIO `0x53`) on 8125 parts.
3. Program LTR (§3.4).
4. Modify OCP `0xE094`: clear `0xFF00`.
5. Modify OCP `0xE092`: clear `0x00FF`, set bit 2 (allows the chip to enter L1.2).

**[HW]** To turn it off, do the reverse: clear `0x00FF` in OCP `0xE092`, clear CLKREQ, clear Config5 bit 0.

**[POLICY]** The reference driver enables ASPM in the chip only if the OS controls ASPM for the link. It also disables ASPM L1 on the link **unless** OCP `0xC0B2` bits 3:0 are non-zero. A non-zero value there means the board vendor validated ASPM L1.2.

### 3.4 LTR programming

**[HW]** Common to all 8125/8126 parts:

1. Write OCP `0xCDD0` = `0x9003`.
2. Modify OCP `0xE034`: set bits 15:14 (snoop LTR enable).
3. Modify OCP `0xE0A2`: set bit 0 (ALDPS LTR enable).

**[HW]** Then write the per-chip latency table:

| OCP | 8125 (A/B/gen 3) | 8126A |
|---|---|---|
| `0xCDD2` | `0x889C` | `0x8C09` |
| `0xCDD4` | `0x8C30` | `0x9003` |
| `0xCDD6` | `0x9003` | `0x9003` |
| `0xCDD8` | `0x9003` | `0x9003` |
| `0xCDDA` | `0x9003` | `0x9003` |
| `0xCDDC` | `0x9003` | `0x9003` |
| `0xCDE8` | `0x883E` | `0x887A` |
| `0xCDEA` | `0x9003` | `0x9003` |
| `0xCDEC` | `0x889C` | `0x8C09` |
| `0xCDEE` | `0x9003` | `0x9003` |
| `0xCDF0` | `0x8C09` | `0x8A62` |
| `0xCDF2` | `0x9003` | `0x9003` |
| `0xCDF4` | not written | `0x883E` |
| `0xCDF6` | not written | `0x9003` |

**[HW]** Finish with:

1. Modify OCP `0xE032`: set bit 14 (link-speed-change LTR).
2. Modify OCP `0xE032`: clear bits 1:0, set bit 0.
3. Modify OCP `0xE038`: set bit 0 (sends an LTR message).

> OPEN QUESTION: the LTR value encoding presumably follows the PCIe LTR format (bit 15 requirement, bits 12:10 scale, bits 9:0 value). The pairing of addresses to link speeds is unknown.

### 3.5 D3 PLL behavior

**[HW]** MMIO `0x6F`: bit 6 keeps the PLL running in D3hot, and bit 7 keeps it running in D3cold. On 8125/8126 parts, always set bit 6. Set bit 7 only when Wake-on-LAN or remote management is enabled.

## 4. Wake-on-LAN (8125-family encoding)

| Wake source | Control |
|---|---|
| Magic packet | OCP `0xC0B6` bit 0 |
| Link change | Config3 (MMIO `0x54`) bit 4, **and** OCP `0xE0C6` bits 5:0 = `0x13` (`0x00` when off) |
| Unicast / broadcast / multicast | Config5 bits 4 / 6 / 5 |
| Any wake enabled | Config5 bit 1 (LAN wake), Config2 bit 5 (PME signal) |

**[HW]** These writes require unlocked config registers.

## 5. Energy-Efficient Ethernet

### 5.1 MAC side

**[HW]** TX LPI idle timer: write 16-bit MMIO `0x6048` with the number of **byte times** the transmitter must stay idle before it enters LPI.

**[POLICY]** The reference driver uses MTU + 14 + 32.

**[HW]** Enabling TX LPI:

| Chip | Enable | Disable |
|---|---|---|
| 8125A | Modify OCP `0xE040`: set `0x0003`. Modify OCP `0xEB62`: set `0x0006` | Clear the same bits |
| 8125B, gen 3, 8126A, 8127A | Modify OCP `0xE040`: set `0x0003` | Clear them |

**[HW]** Speeds that support EEE:

| Chip | EEE-capable speeds |
|---|---|
| 8125A | 100 M, 1 G |
| 8125B, gen 3 | 100 M, 1 G, 2.5 G |
| 8126A | 100 M, 1 G, 2.5 G, 5 G |

### 5.2 PHY side (paged PHY registers, section 01, §6)

**[HW]** For 8125 parts, set bit 4 of page `0xA43` register `0x11`. Then, for 8125 and 8126 parts alike:

1. Clear bit 4 of page `0xA6D` register `0x14`.
2. Clear bit 7 of page `0xA42` register `0x14`.
3. Clear bit 9 of page `0xA4A` register `0x11`.

**[HW]** EEE advertisement itself uses the standard IEEE 802.3 Clause 45 EEE advertisement registers in MMD 7 (including the 2.5G/5G EEE advertisement register defined by 802.3bz).

> OPEN QUESTION: the meaning of the individual bits in step 1–3 is undocumented. Clearing them matches what Realtek's own drivers do.
