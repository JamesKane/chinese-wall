# 18: Video codec delta against Sky1

**Status: not started.** *(vendor)*: mainline has no driver or devicetree
node for this block at the baseline. The facts below come from the vendor
kernel and SpacemiT's published firmware.

## Scope

The K3 video codec (`arm china,linlon-v5`, 2 MiB register window at
`0xC0500000`, interrupt 74, one APMU clock and reset, its own power domain)
is the same firmware-driven hardware family that
[`../linlon-vpu`](../linlon-vpu/) specifies for Sky1. The "v5" in the
compatible string does not mean an older generation. This section records
only the differences.

## Known so far

| Item | K3 | Sky1 spec |
|---|---|---|
| Hardware family | v52_v76 | v52_v76 |
| `HARDWARE_ID[31:16]` products the K3 vendor driver accepts for this family | `0x5662`, `0x5663` | `0x5662`, `0x5663`, `0x5664` |
| Firmware build | revision 80, patch 2; codec message protocol 3.5 | — |
| Firmware location | `linlon-v52_v76-80-2/<codec><dec\|enc>.fwb`: a per-build directory in front of the Sky1 file names | `<codec><dec\|enc>.fwb` at the firmware root |
| Decoders shipped | AVS, AVS2, H.264, HEVC, JPEG, MPEG-2, MPEG-4, RealVideo, VC-1, VP8, VP9 | also AV1 |
| Encoders shipped | H.264, HEVC, JPEG, VP8, VP9 | same |
| Cores, LSIDs | read from the hardware at run time; the vendor driver allows up to 4 LSIDs | 4 cores, 4 LSIDs |

- **[HW]** *(vendor)* No AV1 firmware ships for K3, and the vendor driver
  has no AV1 codec mapping. Do not advertise AV1 decode.
- *(vendor)* SpacemiT publishes the firmware in its `k3x-vpu-firmware`
  repository with no license statement. Check redistribution terms before
  shipping it.

> OPEN QUESTION: The firmware's build string begins with `5663`, which
> suggests the K3 core reports product `0x5663`. Confirm by reading
> `HARDWARE_ID` on hardware.

> OPEN QUESTION: Whether firmware protocol 3.5 differs from the protocol
> the Sky1 spec describes. Compare once the Sky1 firmware version is known.

Still to extract: MMU and bus-attribute differences, and K3 clock, reset
and power-domain sequencing (01, 14).
