# 00: Overview and Architecture

## 0.1 Target hardware

Arm Mali GPUs that use the **Command Stream Frontend (CSF)**, architecture
majors 10–14 (Valhall CSF and the later "5th generation" parts). The
first-priority platform is the **Rockchip RK3588/RK3588S**:

| Property | RK3588 value |
|---|---|
| GPU | Mali-G610 MP4 (arch 10, product major 7) |
| MMIO | 2 MiB at `0xFB000000` |
| IRQs | three level-triggered SPIs: job, mmu, gpu |
| Clocks | core, coregroup, stacks |
| Supplies | `mali` (GPU rail), `sram` (GPU memory rail) |
| Coherency | not I/O-coherent, so the driver must do CPU cache maintenance |
| Firmware | `arm/mali/arch10.8/mali_csffw.bin` (from linux-firmware) |

Other SoCs with the same IP family include MediaTek MT8196 and NXP i.MX95
(Mali-G310). The register interface is discoverable through `GPU_ID`
(02-register-map).

## 0.2 How a CSF GPU works

Unlike older Mali "job manager" GPUs, the host does not submit jobs to
hardware slots directly. A **microcontroller (MCU)** inside
the GPU runs Arm-supplied firmware. The firmware schedules work onto the
shader cores, tiler and other units.

```
  host CPU                          GPU
 ┌──────────┐   MMIO regs   ┌──────────────────────────────┐
 │  driver  │──────────────▶│ GPU ctrl / PWR / MMU / MCU   │
 │          │   doorbells   │                              │
 │          │──────────────▶│  MCU (firmware) ── schedules ─▶ shader cores,
 │          │◀──────────────│                                 tiler, L2
 │          │   JOB IRQ     │                              │
 │          │               │                              │
 │          │◀═════ shared memory interface (GLB/CSG/CS) ═▶│
 └──────────┘               └──────────────────────────────┘
       ▲
       │ ioctls (MIT UAPI)
   userspace (Mesa panvk / panfrost)
       writes CS instruction streams into GPU-visible ring buffers
```

The driver is responsible for:

1. **Platform bring-up** (01): clocks, regulators, power domain, DVFS.
2. **GPU control** (03): identification, reset, power-up of L2 and cores,
   cache maintenance, interrupt handling.
3. **MMU** (04): per-process GPU address spaces using the Arm VMSAv8-64
   stage-1 page-table format. Hardware address-space slots are multiplexed
   across VMs.
4. **Firmware** (05): parse the firmware image, map its sections into the
   MCU's own address space (slot 0), boot the MCU, and recover it on hang.
5. **CSF interface** (06): talk to the firmware through a shared-memory
   protocol of request/acknowledge toggle bits at three levels:
   - **Global (GLB)**: firmware-wide configuration, power-off and idle timers, ping
   - **Command Stream Group (CSG)**: a group of queues sharing an address
     space and priority. The firmware exposes a small number of CSG *slots*.
   - **Command Stream (CS)**: one ring buffer of CS instructions.
6. **Scheduling** (07): time-multiplex userspace groups onto the limited CSG
   slots (suspend and resume through suspend buffers), track job completion
   through sync objects in GPU memory, and handle faults and timeouts.
7. **Tiler heap** (08): grow tiler heap memory on firmware out-of-memory
   requests.
8. **Memory** (09): buffer objects, CPU mappings, cache maintenance, dma-buf.
9. **Userspace ABI** (10): the ioctl interface Mesa uses.

## 0.3 Lifecycle at a glance

1. Enable power domain, regulators and clocks. Read `GPU_ID`, then pick
   behavior by arch major (10–13 legacy power registers, 14 PWR block).
2. Probe features and present masks. Set the coherency protocol.
3. Set up the MMU. Create the MCU VM and bind it to AS slot 0.
4. Load firmware sections into MCU memory. Set `MCU_CONTROL` = AUTO and wait
   for the global-interface interrupt.
5. Parse the global interface, enumerate CSG and CS slots, and program the
   global configuration (timers, ping, idle).
6. Accept userspace: VMs, BOs, groups, submissions.
7. Runtime suspend: suspend all resident groups, halt the MCU, power down L2,
   gate clocks. Resume reverses this. RK3588 may lose the GPU rail in system
   suspend, so all state must be restorable from memory.
8. On fatal errors: perform a full GPU reset and reload the firmware. Groups
   that were suspended cleanly survive; others are terminated.

## 0.4 Licensing note for the clean room

**[ABI]** The userspace header (`panthor_drm.h`) is MIT-licensed and may be
used directly. The firmware binary is distributed by Arm under its own
redistributable license in linux-firmware. The clean room loads it and does
not need to understand its contents beyond 05.
