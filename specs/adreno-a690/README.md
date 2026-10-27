# Adreno 690 GPU Clean-Room Hardware Specification

## 1. Overview

**[HW]** The Qualcomm Adreno 690 (`0x06090000`, Architecture Generation 6, Family 9, Major 0, Minor 0, Patchid 0) is the integrated 3D and compute graphics processor on the SC8280XP SoC used by the Radxa Dragon Q8B board. It belongs to the fourth-generation Adreno 6xx architecture family and shares its command processor (SQE) microcode and Graphics Management Unit (GMU) firmware binaries with the Adreno 660, while featuring a larger 4 MiB on-chip Graphics Memory (GMEM), 4 Shader Processor (SP) clusters, 16 Render Backend (RB) / Color-Depth Cache Unit (CCU) slices, and SC8280XP-specific clock, power-domain, and memory-interleaving configurations.

**[ABI]** On the Radxa Dragon Q8B running UEFI/ACPI firmware, the GPU subsystem is exposed under the ACPI namespace node `\_SB.GPU0` with Hardware ID `QCOM0636` and CID `PNP0D80`.

## 2. Specification Index

**[POLICY]** This specification is organized into four functional documents:

| Document | Scope |
|---|---|
| `01-platform-gpucc-and-smmu.md` | Platform memory map, ACPI resource aliasing, GPU Clock Controller (`GPUCC`) PLLs/RCGs/branches/GDSCs, interrupt lines, and GPU ARM SMMUv2 (`0x03da0000`) stream IDs and context banks. |
| `02-gmu-hfi-and-devfreq.md` | Graphics Management Unit (GMU) Cortex-M3 processor, ITCM/DTCM firmware loading, GMU shared-memory IOVA allocator, OOB (out-of-band) handshake protocol, Host Firmware Interface (HFI) queues and messages, and OPP frequency/voltage tables. |
| `03-a690-cp-ringbuffer-and-zap.md` | Adreno 690 3D core initialization, UBWC 4.0 and GMEM configuration, hardware clock-gating (`HWCG`), register protection (`CP_PROTECT`), SQE microcode bootstrap, PM4 ringbuffer and preemption registers, and TrustZone PIL Zap shader (`PAS ID 13`) bring-up. |
| `04-drm-msm-uapi.md` | Direct Rendering Manager (DRM) userspace binary interface (`msm` driver v1.13.0) required by Mesa Turnip (Vulkan) and Freedreno (`fd_pipe` / Gallium), including all 16 device-specific ioctls, parameter IDs, GEM flags, and command submission structures. |

## 3. Top-Level Hardware Block Diagram

**[HW]** The Adreno 690 subsystem consists of five interacting hardware blocks:
1. **GPU Clock Controller (`GPUCC` at `0x03d90000`)**: Generates the CX and GX clocks and controls the `CX_GDSC` and `GX_GDSC` power switches.
2. **Graphics Management Unit (`GMU` at `0x03d6a000`)**: An embedded Cortex-M3 microcontroller that manages GX power-rail sequencing, clock switching via RPMh, dynamic voltage/frequency scaling (DVFS), and inter-frame power collapse (`IFPC`).
3. **Adreno 690 3D Core (`GFX3D` at `0x03d00000`)**: Contains the RBBM (Register Bus Block Manager), CP (Command Processor / SQE), UCHE (Unified L2 Cache), SP/TP/RB/VFD/VPC/PC/GRAS/LRZ/VSC 3D pipeline blocks, GBIF (Graphics Bus Interface), and 4 MiB GMEM.
4. **GPU System MMU (`GPU SMMU` at `0x03da0000`)**: Dedicated ARM SMMUv2 translation unit with 8 Stream Match Registers and separate translation context banks for the 3D engine (Context Bank 0) and the GMU (Context Bank 1).
5. **TrustZone Peripheral Image Loader (`PAS ID 13`)**: Authenticates and boots the signed "Zap" shader firmware image (`qcdxkmsuc8280.mbn`) required to transition the GPU out of secure-world protection into non-secure operation.

## 4. Required Firmware Files

**[HW]** Bringing up the Adreno 690 to execute 3D/compute command streams requires three firmware images:

| Component | Standard Firmware Path | Size (Empirical) | Purpose |
|---|---|---|---|
| **CP SQE Microcode** | `qcom/a660_sqe.fw` | ~37 KiB | Command Processor instruction stream loaded into GEM/SMMU Context Bank 0 and pointed to by `CP_SQE_INSTR_BASE`. Includes a 4-byte legacy header skipped before loading. |
| **GMU Firmware** | `qcom/a660_gmu.bin` | ~50 KiB | Block-formatted Cortex-M3 firmware loaded into GMU ITCM, DTCM, and IOVA-mapped shared buffers before releasing GMU reset. |
| **Zap Shader MDT/MBN** | `qcom/sc8280xp/LENOVO/21BX/qcdxkmsuc8280.mbn` | ~8 KiB | Signed ELF image loaded into a reserved physical carveout and authenticated by TrustZone SMC (`PAS ID 13`) so the GPU exits secure mode. |
