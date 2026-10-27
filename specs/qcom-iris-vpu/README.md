# Qualcomm Iris VPU 2.0 (SC8280XP / Radxa Dragon Q8B) Clean-Room Specification

## 1. Subsystem Overview

- **[HW]** The Qualcomm SC8280XP SoC on the Radxa Dragon Q8B integrates a Video Processing Unit (VPU) 2.0 hardware block (codenamed Iris / Venus, wrapper hardware version `0x60100608`, major 6, minor 16) that accelerates stateful H.264, HEVC (H.265), and VP9 video decoding and H.264 and HEVC video encoding at resolutions up to 8192×8192 (decode) and 4096×2160 (encode).
- **[HW]** The VPU contains an embedded microcontroller (Tensilica-based firmware processor, architecture ID `VIDEO_ARCH_LX = 1`) that executes signed firmware (`qcom/vpu/vpu20_p4_gen2_s6.mbn`) from a 7 MiB physical DRAM carve-out (`0x8c600000`–`0x8cd00000`) authenticated and booted via TrustZone Peripheral Authentication Service (PAS ID `9`).
- **[HW]** Host software communicates with the VPU microcontroller across shared coherent DMA ring queues using the Host Firmware Interface Generation 2 (HFI Gen2) packet protocol and receives asynchronous responses and state notifications via a level-high GICv3 SPI interrupt (GSIV `206` / SPI `174`).
- **[ABI]** Host operating systems expose the hardware to userspace multimedia frameworks (such as FFmpeg, GStreamer, and Chromium) as two Video4Linux2 Memory-to-Memory (`V4L2_CAP_VIDEO_M2M_MPLANE`) character device nodes: one stateful decoder (`qcom-iris-decoder`) and one stateful encoder (`qcom-iris-encoder`).

---

## 2. Hardware Topology Summary

| Resource | Value on SC8280XP (Radxa Dragon Q8B) | Tag |
|---|---|---|
| VPU Register Window | `0x0aa00000`, size `0x00100000` (1 MiB mapped by driver; `0x00200000` in ACPI `\_SB.GPU0` `VIDEO_REGS`) | **[HW]** |
| Video Clock Controller (`videocc`) | `0x0abf0000`, size `0x00010000` (64 KiB) | **[HW]** |
| Global Clock Controller (`gcc`) Video AXI Branch | `GCC_BASE + 0x28010` (`GCC_VIDEO_AXI0_CLK`) | **[HW]** |
| Interrupt | GSIV `206` (GIC SPI `174`), active-high level-sensitive (`VIDC_INTERRUPT`) | **[HW]** |
| RPMh Power Domains | `MX` (`mx.lvl`, `0x3001c`) and `MMCX` (`mmcx.lvl`, `0x30020`) | **[HW]** |
| Local GDSCs (`videocc`) | `MVS0C_GDSC` (`videocc + 0xbf8`, interface/wrapper domain) and `MVS0_GDSC` (`videocc + 0xd18`, core codec domain) | **[HW]** |
| Apps SMMU (`0x15000000`) Stream IDs | Non-secure stream `0x2a00` (mask `0x0400`) and firmware/secure stream `0x2a07` (mask `0x0400`), cache-coherent (`CCA = 1`) | **[HW]** |
| Firmware Carve-out (`pil_video_mem`) | Physical base `0x8c600000`, size `0x00700000` (7 MiB) | **[HW]** |
| TrustZone PAS ID | `9` | **[HW]** |
| Firmware Image Path | `qcom/vpu/vpu20_p4_gen2_s6.mbn` | **[ABI]** |
| Protected IOVA Reservation (`iris-iova`) | IOVA `0x00000000`..`0x257fffff` (size `0x25800000` = 600 MiB) reserved via TrustZone `MP_VIDEO_VAR`; host DMA allocations must use IOVAs in `0x25800000`..`0xdfffffff` | **[HW]** |

---

## 3. Specification Index

- **[`01-videocc-power-smmu-and-registers.md`](./01-videocc-power-smmu-and-registers.md)** — **[HW]** Complete specification of the `MMCX`/`MX` RPMh power rail hazard and voting requirements, interconnect bandwidth voting (`cpu-cfg` and `video-mem`), `videocc` (`0x0abf0000`) Lucid 5LPE PLL (`video_pll0`), RCGs, dividers, branch clocks, resets, and `MVS0C`/`MVS0` GDSCs, Apps SMMU stream IDs and IOVA window constraints, TrustZone PAS ID 9 boot and `MP_VIDEO_VAR` memory protection SMC calls, and the complete VPU 2.0 MMIO register map (`0x0aa00000`) with exact power-on, hardware-mode handoff, watchdog/NOC reset, and power-off sequences.
- **[`02-hfi-gen2-protocol.md`](./02-hfi-gen2-protocol.md)** — **[HW]** Complete specification of the HFI Gen2 shared memory layout (456-byte queue table header, three 1 MiB circular queues for Command, Message, and Debug, and the 4 KiB Subsystem Failure Reason buffer), the 32-byte HFI message header, 32-byte HFI packet header, and 64-byte HFI buffer descriptor, all HFI Gen2 command, property, port, payload, buffer type, and error constants, internal scratch/persist buffer sizing formulas for H.264/HEVC/VP9 decode and H.264/HEVC encode, and session lifecycle state machines.
- **[`03-v4l2-m2m-abi.md`](./03-v4l2-m2m-abi.md)** — **[ABI]** Complete specification of the V4L2 Memory-to-Memory multi-planar (`V4L2_BUF_TYPE_VIDEO_OUTPUT_MPLANE` and `V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE`) userspace ABI, supported pixel formats (`NV12`, `QC08C` UBWC 8-bit, `QC10C` UBWC 10-bit, `H264`, `HEVC`, `VP9`), geometry alignment and stride/plane size calculation formulas (linear NV12 and UBWC macrotile metadata+payload layouts), V4L2 controls and mappings to HFI Gen2 properties, dynamic resolution change (`V4L2_EVENT_SOURCE_CHANGE`) handling, drain (`V4L2_DEC_CMD_STOP` / `V4L2_ENC_CMD_STOP`), seek, and EOS behavior.
