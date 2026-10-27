# 04: Multi-queue Receive, RSS and MSI-X

> **Applicability.** **[HW]** The reference driver enables everything in this section on **RTL8127A only**. On RTL8125(A/B/D) and RTL8126A it uses one RX queue, one TX queue, one interrupt vector and the default RX descriptor format. The register addresses below are in the shared 8125-family register map. Realtek's own out-of-tree drivers are known to use RSS on 8125/8126 parts, but nothing here has been confirmed on them.
>
> OPEN QUESTION: does RTL8125D (or RTL8126A) implement the RSS block, the RSS descriptor format and the vector-mapping registers exactly as described here? How many RX queues and MSI-X vectors do they offer? Read the MSI-X capability table size first, then test.

## 1. Default RX descriptor (16 bytes)

| Offset | Field |
|---|---|
| 0 | Status/command dword 1 |
| 4 | Dword 2: VLAN |
| 8 | 64-bit buffer address |

**[HW]** Dword 1 when the controller returns the descriptor:

| Bits | Meaning |
|---|---|
| 31 | Owned by the controller |
| 30 | End of ring |
| 29 / 28 | First / last segment (a frame that does not have both set spans several buffers) |
| 22 | Receive watchdog (frame too long) |
| 21 | Error summary |
| 20 | Runt |
| 19 | CRC error |
| 18:17 | Protocol: 1 = TCP, 2 = UDP, 3 = IP only |
| 16 / 15 / 14 | IP / UDP / TCP checksum failed |
| 13:0 | Frame length including FCS |

**[HW]** Dword 2: bit 16 means a VLAN tag was stripped. Bits 15:0 hold that tag, byte-swapped.

**[HW]** To return a descriptor to the controller: clear dword 2, make the write visible, then write dword 1 as own bit + preserved end-of-ring bit + buffer size. The buffer size field holds the buffer length (the reference driver uses 16383).

## 2. RSS RX descriptor format

**[HW]** The RSS format is enabled by setting bit 1 of MMIO `0xD8`. It applies to all RX queues. The descriptor is still 16 bytes, but its fields are laid out differently:

| Offset | When given to the controller | When returned by the controller |
|---|---|---|
| 0 | Buffer address, low half | RSS information dword |
| 4 | Buffer address, high half | 32-bit RSS hash |
| 8 | 0 | VLAN dword (same encoding as default dword 2) |
| 12 | Own / end-of-ring / buffer size | Status dword |

**[HW]** Because the controller writes the hash information **over the buffer address**, the driver must rewrite the 64-bit buffer address at offset 0 every time it gives a descriptor back to the controller.

**[HW]** RSS information dword:

| Bit | Meaning |
|---|---|
| 27 | L4 protocol is UDP |
| 28 | L3 protocol is IPv4 |
| 29 | L3 protocol is IPv6 |
| 30 | L4 protocol is TCP |

**[HW]** If neither bit 28 nor bit 29 is set, the hash is not valid. If bit 27 or 30 is also set, the hash covers L4 ports. Otherwise it covers L3 addresses only.

**[HW]** Status dword in RSS format:

| Bits | Meaning |
|---|---|
| 31, 30, 29, 28 | Own, end-of-ring, first segment, last segment (as in the default format) |
| 22 | Error summary |
| 21 | Runt |
| 20 | CRC error |
| 19 | Frame is UDP |
| 18 | Frame is TCP |
| 16 | UDP checksum failed |
| 15 | TCP checksum failed |
| 13:0 | Frame length |

**[HW]** There is no over-length (watchdog) indication in this format.

## 3. Queue and RSS registers

| MMIO | Width | Meaning |
|---|---|---|
| `0x4000 + 8×(n−1)` | 32 | RX queue *n* (*n* ≥ 1) ring base, low half |
| `0x4004 + 8×(n−1)` | 32 | RX queue *n* ring base, high half |
| `0x4800` | 16 | Queue count. Bits 4:2 = log2(number of RX queues). Preserve the other bits |
| `0x4500` | 32 | RSS control (below) |
| `0x4600`–`0x4627` | 40 B | RSS hash key, written as ten 32-bit little-endian words (key byte 0 is the lowest byte at `0x4600`) |
| `0x4700`–`0x477F` | 128 B | Indirection table: one byte per entry, holding the RX queue index. Four entries per 32-bit little-endian word |

**[HW]** RX queue 0 keeps using the legacy ring base registers (`0xE4`/`0xE8`).

**[HW]** RSS control register:

| Bits | Meaning |
|---|---|
| 0 | Hash TCP over IPv4 |
| 1 | Hash IPv4 |
| 2 | Hash TCP over IPv6 |
| 3 | Hash IPv6 |
| 4 | Hash IPv6 with extension headers |
| 5 | Hash TCP over IPv6 with extension headers |
| 10:8 | log2(number of indirection-table entries). 7 means 128 |
| 11 | Hash UDP over IPv4 |
| 12 | Hash UDP over IPv6 |
| 18:16 | log2(number of RX queues) |

**[HW]** Up to **8** RX queues and **1** TX queue. The number of RX queues must be a power of two.

**[HW]** Programming order for RSS:

1. Set the queue count in `0x4800`.
2. Set the descriptor format bit in `0xD8`.
3. Program the ring bases.
4. Write RSS control.
5. Write the indirection table.
6. Write the key.

> OPEN QUESTION: the hash function is assumed to be the standard Toeplitz RSS hash, because the key is 40 bytes long. This has not been confirmed.

## 4. Multi-vector MSI-X

**[HW]** Vector assignment:

| MSI-X vector | Source |
|---|---|
| 0–7 | RX queue 0–7 |
| 8 | TX queue 0 |
| 29 | Link change |
| others | Unused |

**[HW]** At least 30 vectors are required, so that vector 29 exists. The reference driver requests 30 to 32. If it gets fewer, it falls back to a single vector (MSI or MSI-X) and to single-queue operation.

**[HW]** Turn on per-vector mapping by setting bit 0 of MMIO `0x34`. Once it is on, use these per-vector registers instead of the single-vector mask and status registers. Bit *k* in each corresponds to vector *k*:

| MMIO | Width | Write 1 to bit *k* to |
|---|---|---|
| `0x0D00` | 32 | Mask vector *k* |
| `0x0D0C` | 32 | Unmask vector *k* |
| `0x0D04` | 32 | Acknowledge (clear) vector *k*'s pending status |

**[HW]** The enable mask uses the same bit numbering: RX queue *n* = bit *n*, TX queue 0 = bit 8, link change = bit 29.

**[POLICY]** Per-vector handling in the reference driver: on an RX or TX vector, mask it, acknowledge it, poll the ring, then unmask it. On the link vector, acknowledge it and read the link state.
