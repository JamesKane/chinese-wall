# Qualcomm FastRPC and Hexagon CDSP/NSP0 (SC8280XP / Radxa Dragon Q8B) Clean-Room Specification

## 1. Subsystem Overview

- **[HW]** The Qualcomm SC8280XP SoC on the Radxa Dragon Q8B includes a Hexagon Compute DSP / Neural Signal Processor (`NSP0` / `CDSP`, architecture `v68`, 4 HVX 128-byte vector threads + HMX matrix engine) booted via TrustZone Peripheral Authentication Service (PAS ID `18`, `0x12`) from a 30 MiB reserved physical DRAM region (`0x8a600000`–`0x8c400000`). Note that the second NSP instance (`NSP1` at `0x21300000`) is fused off by Qualcomm PTE fuses on the Dragon Q8B SKU (`QCS8550` / `SC8280XP` industrial variant) and is non-functional.
- **[HW]** Host software communicates with the Hexagon `CDSP` (and optionally `SLPI`/`SDSP` or `ADSP`) using **FastRPC**, a zero-copy Remote Procedure Call and SMMU shared-memory management protocol layered over the GLINK (`rpmsg`) transport channel `"fastrpcglink-apps-dsp"`.
- **[HW]** FastRPC maps host DMA buffers into per-session Context Banks (`compute-cb@1`..`compute-cb@14`) on the Apps ARM SMMU (`0x15000000`) and passes consolidated 64-bit addresses (`(session_id << 32) | (iova & 0xffffffff)`) in a coherent DMA invocation page to the DSP.
- **[ABI]** Host operating systems expose FastRPC to userspace runtimes (such as Qualcomm QNN HTP, SNPE, and `libcdsprpc.so`) via two character devices for CDSP: `/dev/fastrpc-cdsp` (non-secure / unsigned protection domain) and `/dev/fastrpc-cdsp-secure` (signed protection domain).

---

## 2. Hardware Topology Summary (`NSP0` / `CDSP` on Radxa Dragon Q8B)

| Resource | Value on SC8280XP (Radxa Dragon Q8B) | Tag |
|---|---|---|
| Remoteproc Node (`remoteproc_nsp0`) | Physical MMIO `0x1b300000`, size `0x00010000` (64 KiB) | **[HW]** |
| TrustZone PAS ID | `18` (`0x12`) | **[HW]** |
| Firmware Carve-out (`pil_nsp0_mem`) | Physical base `0x8a600000`, size `0x01e00000` (30 MiB) on Dragon Q8B | **[HW]** |
| Firmware Image Path | `qcom/sc8280xp/qccdsp8280.mbn` | **[ABI]** |
| RPMh Power Domains | `NSP`, `CX` (`cx.lvl`, `0x30018`), `MXC` (`mxc.lvl`, `0x30040`) | **[HW]** |
| SMP2P Interrupts / SMEM | SMEM items `<94>` (in) and `<432>` (out); IPCC client `IPCC_CLIENT_CDSP` (`6`), signal `IPCC_MPROC_SIGNAL_SMP2P` (`2`), remote PID `5`; Watchdog GIC SPI `578` | **[HW]** |
| GLINK Edge | Label `"nsp0"`, remote PID `5`, IPCC client `IPCC_CLIENT_CDSP` (`6`), signal `IPCC_MPROC_SIGNAL_GLINK_QMP` (`0`) | **[HW]** |
| FastRPC GLINK Channel Name | `"fastrpcglink-apps-dsp"` (domain label `"cdsp"`, `CDSP_DOMAIN_ID = 3`, `qcom,non-secure-domain`) | **[HW]** |
| Apps SMMU (`0x15000000`) Compute Context Banks | `14` sessions (`compute-cb@1`..`compute-cb@14`), Stream IDs `0x3181`..`0x3188` and `0x318b`..`0x318f` with mask `0x0420`, all hardware cache-coherent (`dma-coherent`) | **[HW]** |
| Consolidated Address Format (SC8280XP Default SoC Data) | `sid_pos = 32`, `dma_addr_bits = 32`: `consolidated_addr = ((u64)sid << 32) \| (iova & 0xffffffff)` | **[HW]** |
| Userspace Device Nodes | `/dev/fastrpc-cdsp` (non-secure) and `/dev/fastrpc-cdsp-secure` (secure) | **[ABI]** |

---

## 3. Specification Index

- **[`01-glink-rpc-and-smmu-sessions.md`](./01-glink-rpc-and-smmu-sessions.md)** — **[HW]** Complete specification of the CDSP/NSP0 PAS 18 boot and GLINK channel binding (`"fastrpcglink-apps-dsp"`), the 14 Apps SMMU compute context bank sessions (`compute-cb@1..14`) and consolidated 64-bit `(sid << 32) | iova` addressing, the 40-byte GLINK request packet and 16-byte GLINK response packet, the 32-bit scalar descriptor (`sc`) encoding, context ID (`ctxid`) sequence numbering and PD exit sentinel (`0xABCDABCD`), the DMA invocation buffer layout (Remote Arguments, Invoke Buffer List, Physical Page List, FD List, CRC List, Poll Word, and 128-byte-aligned inline overlap buffers), and internal root-handle (`handle = 1` and `handle = 2`) RPC messages.
- **[`02-fastrpc-uapi-and-qnn-compat.md`](./02-fastrpc-uapi-and-qnn-compat.md)** — **[ABI]** Complete specification of the `/dev/fastrpc-*` character device nodes, protection domain security rules (Signed vs Unsigned PD `FASTRPC_MODE_UNSIGNED_MODULE` for QNN HTP), all 13 `FASTRPC_IOCTL_*` commands and their exact binary structure layouts at fixed byte offsets, DMA-BUF export and SMMU mapping lifecycles, and user-space runtime (`libcdsprpc.so` / QNN / SNPE) compatibility requirements.
