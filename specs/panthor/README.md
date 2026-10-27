# Panthor (Mali CSF GPU) Functional Specification

Target: Arm Mali GPUs with the Command Stream Frontend (CSF). These are
architecture majors 10–14 (Valhall CSF and 5th-gen), for example the
**Mali-G610 MP4 in the Rockchip RK3588/RK3588S**.

These documents describe hardware, firmware and userspace-ABI behavior so that
a driver can be written without looking at the Linux source. Each statement is
tagged:

- **[HW]**: required by hardware or firmware
- **[POLICY]**: a Linux design choice, informative only
- **[ABI]**: needed for binary compatibility with existing userspace (Mesa `panvk`/`panfrost` Gallium)

| # | Section | Status |
|---|---------|--------|
| 00 | [Overview and architecture](00-overview.md) | draft |
| 01 | [Platform integration (DT, clocks, regulators, power domains, devfreq)](01-platform.md) | draft |
| 02 | [Register map](02-register-map.md) | draft |
| 03 | [GPU control: reset, power, caches, interrupts](03-gpu-control.md) | draft |
| 04 | [MMU and GPU virtual memory](04-mmu.md) | draft |
| 05 | [Firmware image format, loading and MCU boot](05-firmware.md) | draft |
| 06 | [CSF host↔firmware interface (global, CSG and CS interfaces)](06-csf-interface.md) | draft |
| 07 | [Scheduling of command-stream groups](07-scheduler.md) | draft |
| 08 | [Tiler heap management](08-heap.md) | draft |
| 09 | [Buffer objects and memory management](09-memory.md) | draft |
| 10 | [Userspace ABI (ioctls)](10-uapi.md) | draft |
