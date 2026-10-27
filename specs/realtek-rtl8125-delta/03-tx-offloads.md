# 03: Transmit Offloads: Checksum, TSO, Padding

## 1. TX descriptor (16 bytes, little-endian)

| Offset | Field |
|---|---|
| 0 | Command/status dword 1 |
| 4 | Command dword 2 |
| 8 | 64-bit buffer address |

### 1.1 Dword 1

| Bits | Meaning |
|---|---|
| 31 | **[HW]** Owned by the controller |
| 30 | **[HW]** End of ring (set on the last descriptor) |
| 29 | **[HW]** First segment of a frame |
| 28 | **[HW]** Last segment of a frame |
| 26 | **[HW]** TSO for IPv4/TCP |
| 25 | **[HW]** TSO for IPv6/TCP |
| 24:18 | **[HW]** In TSO mode: offset of the TCP header from the start of the frame, in bytes (0–127) |
| 15:0 | **[HW]** Length of this segment's buffer in bytes |

### 1.2 Dword 2

| Bits | Meaning |
|---|---|
| 31 | **[HW]** Compute UDP checksum (checksum mode) |
| 30 | **[HW]** Compute TCP checksum (checksum mode) |
| 29 | **[HW]** Frame is IPv4: compute IPv4 header checksum (checksum mode) |
| 28 | **[HW]** Frame is IPv6 (checksum mode) |
| 28:18 | **[HW]** In TSO mode: MSS (11 bits, at most `0x7FF`) |
| 27:18 | **[HW]** In checksum mode: offset of the L4 header from the start of the frame (0–1023) |
| 17 | **[HW]** Insert a VLAN tag |
| 15:0 | **[HW]** The VLAN tag, **byte-swapped** (high byte of the tag in bits 7:0) |

**[HW]** The TSO and checksum interpretations of bits 28:18 never apply together. A descriptor is either in TSO mode (dword 1 bit 26 or 25 set) or in checksum mode.

### 1.3 Multi-segment frames

**[HW]** Fill the descriptors for the second and later segments first, and set their own bit. Then set the last-segment bit on the final descriptor. Then make all descriptor writes visible. Only then set the own and first-segment bits on the first descriptor, and ring the doorbell (MMIO `0x90`, write bit 0). Repeat the TSO/checksum and VLAN fields in **every** descriptor of the frame.

## 2. TCP segmentation offload

**[HW]** Limits:

| Limit | Value |
|---|---|
| Maximum TSO payload | 64000 bytes |
| Maximum segments per TSO frame | 64 |
| Maximum TCP header offset | 127 bytes (otherwise segment in software) |
| Maximum MSS | 2047 |

**[HW]** Header preparation:

- **IPv6:** set the IPv6 payload length to 0, and seed the TCP checksum field with the pseudo-header checksum computed with a length of 0.
- **IPv4:** the TCP checksum field must also be seeded with a pseudo-header checksum.

> OPEN QUESTION: whether the IPv4 seed must use a zero length (as for IPv6) or the full length is not established. The reference driver relies on the network stack's usual seed for segmentation offload. Test on hardware.

**[POLICY]** The reference driver enables scatter-gather and TSO by default on every 8125-family part except 8125A, where the user must opt in because of reported TX timeouts.

## 3. Checksum offload

**[HW]** For TCP or UDP over IPv4 or IPv6, set the IP-version bit, the L4-protocol bit and the L4 header offset.

**[HW]** If the L4 header offset is above 1023 bytes, compute the checksum in software.

## 4. Short-frame rules

**[HW]** The 8125/8126 transmitter does **not** pad frames to the 60-byte Ethernet minimum (before FCS). The driver must pad with zeros.

**[HW]** Checksum offload gives wrong results for frames shorter than 60 bytes. For such frames, compute the checksum in software, then pad.

**[HW]** On **8125A and 8125B only**, a further UDP quirk applies. It covers frames shorter than 175 bytes that carry UDP whose bytes from the UDP header to the end of the frame number fewer than 47:

1. If the UDP destination port is 319 or 320 (PTP event or general), pad so that 47 bytes follow the start of the UDP header.
2. In any case, pad until a full 8-byte UDP header fits.
3. Do not use checksum offload for frames padded this way.

**[POLICY]** Gen 3 and 8126A are not subject to the UDP quirk in the reference driver.
