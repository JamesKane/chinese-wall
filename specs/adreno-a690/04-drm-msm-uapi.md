# DRM/MSM Userspace ABI Specification for Adreno 690 (Mesa Turnip & Freedreno)

## 1. DRM Driver Identification & Capabilities

**[ABI]** Userspace Vulkan (Mesa Turnip, `libvulkan_freedreno.so`) and OpenGL/OpenGL ES (Mesa Freedreno Gallium) drivers open the primary or render DRM node (`/dev/dri/renderD128` or `/dev/dri/card1`) and validate the kernel driver identity via `DRM_IOCTL_VERSION` (`_IOWR('d', 0x00, 64)`) and `DRM_IOCTL_GET_CAP` (`_IOWR('d', 0x0c, 16)`):

| Property | Expected Value | Tag | Description |
| :--- | :--- | :--- | :--- |
| Driver `name` | `"msm"` | **[ABI]** | Exact ASCII string required by Mesa `drmGetVersion()` probe to select the Freedreno/Turnip backend. |
| Driver `desc` | `"MSM Snapdragon"` | **[ABI]** | Human-readable driver description string. |
| Version (`major.minor.patchlevel`) | `1.13.0` | **[ABI]** | Major version must be `1`. Minor version encodes supported UAPI extensions: `>= 2` (`SUBMITQUEUE`), `>= 3` (`SYNCOBJ` / `NO_IMPLICIT`), `>= 5` (`SUBMIT_REL_IOVA` / `SET_IOVA`), `>= 6` (`SUBMITQUEUE_QUERY` / per-context faults), `>= 7` (`CACHED_COHERENT`), `>= 8` (`PRIO` / `NR_RINGS`), `>= 9` (`SUSPENDS`), `>= 10` (`NO_SHARE`), `>= 11` (`SYNCOBJ_TIMELINE`), `>= 12` (`VA_START` / `VA_SIZE`), `>= 13` (`VM_BIND`). |
| DRM Driver Feature Flags | `DRIVER_GEM \| DRIVER_RENDER \| DRIVER_SYNCOBJ \| DRIVER_SYNCOBJ_TIMELINE \| DRIVER_GPUVA` | **[ABI]** | Standard DRM core capabilities (`DRM_CAP_PRIME = 0x3`, `DRM_CAP_SYNCOBJ = 1`, `DRM_CAP_SYNCOBJ_TIMELINE = 1`). |
| Pipe Identifier (`MSM_PIPE_3D0`) | `0x00000010` | **[ABI]** | Value passed in the `pipe` field of `GET_PARAM`, `SET_PARAM`, `GEM_SUBMIT` (lower 16 bits of `flags`), and `WAIT_FENCE` (`MSM_PIPE_2D0 = 0x01`, `MSM_PIPE_2D1 = 0x02`, `MSM_PIPE_3D0 = 0x10`). |

---

## 2. Device-Specific Ioctl Table (`DRM_COMMAND_BASE = 0x40`)

**[ABI]** All device-specific ioctls use ioctl type `'d'` (`0x64`) and command numbers starting at `DRM_COMMAND_BASE` (`0x40`). On 64-bit little-endian architectures (ARM64), the complete 32-bit ioctl request codes are:

| Cmd Offset | Full Ioctl Nr | Symbolic Name | Direction & Size | Request Code (Hex) | Tag | Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `0x00` | `0x40` | `DRM_IOCTL_MSM_GET_PARAM` | `_IOWR('d', 0x40, 24)` | `0xc0186440` | **[ABI]** | Query hardware or context parameter (`MSM_PARAM_*`). |
| `0x01` | `0x41` | `DRM_IOCTL_MSM_SET_PARAM` | `_IOW('d', 0x41, 24)` | `0x40186441` | **[ABI]** | Set context parameter (`SYSPROF`, `COMM`, `CMDLINE`, `EN_VM_BIND`). |
| `0x02` | `0x42` | `DRM_IOCTL_MSM_GEM_NEW` | `_IOWR('d', 0x42, 16)` | `0xc0106442` | **[ABI]** | Allocate a new GEM buffer object (`BO`). |
| `0x03` | `0x43` | `DRM_IOCTL_MSM_GEM_INFO` | `_IOWR('d', 0x43, 24)` | `0xc0186443` | **[ABI]** | Query or configure BO mmap offset, GPU IOVA, or debug name. |
| `0x04` | `0x44` | `DRM_IOCTL_MSM_GEM_CPU_PREP` | `_IOW('d', 0x44, 24)` | `0x40186444` | **[ABI]** | Wait for GPU rendering on a BO to finish before CPU access. |
| `0x05` | `0x45` | `DRM_IOCTL_MSM_GEM_CPU_FINI` | `_IOW('d', 0x45, 4)` | `0x40046445` | **[ABI]** | Complete CPU access to a BO (no-op on coherent/WC mappings). |
| `0x06` | `0x46` | `DRM_IOCTL_MSM_GEM_SUBMIT` | `_IOWR('d', 0x46, 64)` | `0xc0406446` | **[ABI]** | Submit `IB1` command streams and BO table to a submitqueue. |
| `0x07` | `0x47` | `DRM_IOCTL_MSM_WAIT_FENCE` | `_IOW('d', 0x47, 32)` | `0x40206447` | **[ABI]** | Wait until a submitqueue fence sequence number completes. |
| `0x08` | `0x48` | `DRM_IOCTL_MSM_GEM_MADVISE` | `_IOWR('d', 0x48, 12)` | `0xc00c6448` | **[ABI]** | Mark a BO as purgeable (`DONTNEED`) or active (`WILLNEED`). |
| `0x0A` | `0x4A` | `DRM_IOCTL_MSM_SUBMITQUEUE_NEW` | `_IOWR('d', 0x4a, 12)` | `0xc00c644a` | **[ABI]** | Create a prioritized command submission queue. |
| `0x0B` | `0x4B` | `DRM_IOCTL_MSM_SUBMITQUEUE_CLOSE` | `_IOW('d', 0x4b, 4)` | `0x4004644b` | **[ABI]** | Destroy a submitqueue by ID. |
| `0x0C` | `0x4C` | `DRM_IOCTL_MSM_SUBMITQUEUE_QUERY` | `_IOW('d', 0x4c, 24)` | `0x4018644c` | **[ABI]** | Query fault count for a specific submitqueue. |
| `0x0D` | `0x4D` | `DRM_IOCTL_MSM_VM_BIND` | `_IOWR('d', 0x4d, 88)` | `0xc058644d` | **[ABI]** | Execute synchronous or asynchronous GPU virtual address bind/unbind ops. |
| `0x0E` | `0x4E` | `DRM_IOCTL_MSM_PERFCNTR_CONFIG` | `_IOW('d', 0x4e, 32)` | `0x4020644e` | **[ABI]** | Configure hardware performance counter sampling. |

---

## 3. Parameter Queries (`MSM_PARAM_*`) & Adreno 690 Values

### 3.1 `GET_PARAM` / `SET_PARAM` Payload Layout (`24` bytes)

**[ABI]** Both `DRM_IOCTL_MSM_GET_PARAM` (`0x40`) and `DRM_IOCTL_MSM_SET_PARAM` (`0x41`) pass a 24-byte structure:

| Byte Offset | Size | Type | Field Name | Tag | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `+0x00` | 4 | `uint32` | `pipe` | **[ABI]** | Must be `0x00000010` (`MSM_PIPE_3D0`). |
| `+0x04` | 4 | `uint32` | `param` | **[ABI]** | Parameter selector (`MSM_PARAM_*` from table below). |
| `+0x08` | 8 | `uint64` | `value` | **[ABI]** | Returned 64-bit value (for `GET_PARAM`), input 64-bit scalar (for `SET_PARAM`), or userspace pointer (when `len > 0`). |
| `+0x10` | 4 | `uint32` | `len` | **[ABI]** | Must be `0` for all scalar parameters; set to byte length of user buffer at `value` for `MSM_PARAM_COMM` and `MSM_PARAM_CMDLINE`. |
| `+0x14` | 4 | `uint32` | `pad` | **[ABI]** | Reserved padding; must be `0` (returns `-EINVAL` if non-zero). |

### 3.2 Complete Parameter ID Table & Expected Adreno 690 Responses

**[ABI]** On SC8280XP with Adreno 690, `GET_PARAM` and `SET_PARAM` must implement the following parameter IDs:

| Param ID | Symbolic Name | Access | Expected Value on Adreno 690 (SC8280XP) | Tag | Detailed Semantics |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `0x01` | `MSM_PARAM_GPU_ID` | RO | `0x00000000` | **[ABI]** | Legacy decimal GPU revision (`0` on Adreno 690 because `revn` is `0`; Mesa Turnip/Freedreno matches Adreno 690 via `MSM_PARAM_CHIP_ID`). |
| `0x02` | `MSM_PARAM_GMEM_SIZE` | RO | `0x00400000` (`4,194,304`) | **[ABI]** | On-chip GMEM tile buffer size in bytes (`4 MiB` catalog value; see Section 1 of File 03). |
| `0x03` | `MSM_PARAM_CHIP_ID` | RO | `0x0000ffff06090000` (or `0x0000ffff06090001`) | **[ABI]** | Bits 31:0 = `chip_id` (`0x06090000` or `0x06090001`: `core=6, major=9, minor=0, patchid=0/1`); Bits 47:32 = `speedbin` (`0xffff` when no speedbin fuse table exists). |
| `0x04` | `MSM_PARAM_MAX_FREQ` | RO | `690000000` (or `900000000`) | **[ABI]** | Maximum GPU core clock frequency in Hz from the OPP table. |
| `0x05` | `MSM_PARAM_TIMESTAMP` | RO | 64-bit live counter | **[ABI]** | Reads the 64-bit 19.2 MHz `GMU_ALWAYS_ON_COUNTER_L`/`_H` (`0x1f888`/`0x1f889`). |
| `0x06` | `MSM_PARAM_GMEM_BASE` | RO | `0x0000000000000000` | **[ABI]** | Virtual base address of GMEM (`0x0` on Gen 3/Gen 4 Adreno 6xx; used by userspace for `RB_BLIT_BASE_GMEM`). |
| `0x07` | `MSM_PARAM_PRIORITIES` (`NR_RINGS`) | RO | `12` (`4 * 3`) or `3` (`1 * 3`) | **[ABI]** | Total number of exposable submitqueue priority levels (`nr_rings * 3`). Lower numeric priority values are higher priority (`0` = highest). |
| `0x08` | `MSM_PARAM_PP_PGTABLE` | RO | `0` | **[ABI]** | Legacy flag (always returns `0`; per-process page tables are discovered via `MSM_PARAM_VA_START != 0`). |
| `0x09` | `MSM_PARAM_FAULTS` | RO | `uint32` count | **[ABI]** | Total GPU page faults + context faults (when `len == 0`). |
| `0x0a` | `MSM_PARAM_SUSPENDS` | RO | `uint32` count | **[ABI]** | Number of times the GPU has entered runtime suspend (used by Mesa to detect lost perf counter state). |
| `0x0b` | `MSM_PARAM_SYSPROF` | WO | `0`, `1`, or `2` | **[ABI]** | `1` = retain perf counter registers across IFPC/preemption; `2` = also hold a runtime PM active reference. |
| `0x0c` | `MSM_PARAM_COMM` | WO | User string ptr | **[ABI]** | Overrides process name (`comm`) recorded in GPU crash dumps (`len` = string byte length). |
| `0x0d` | `MSM_PARAM_CMDLINE` | WO | User string ptr | **[ABI]** | Overrides process command line recorded in GPU crash dumps (`len` = string byte length). |
| `0x0e` | `MSM_PARAM_VA_START` | RO | `0x0000000100000000` (`4 GiB`) | **[ABI]** | Base address of per-process GPU virtual address space (`4 GiB` when per-process SMMU Context Bank 0 page tables are supported; `0` if global VM only). |
| `0x0f` | `MSM_PARAM_VA_SIZE` | RO | `0x0000ffff00000000` (`256 TiB - 4 GiB`) | **[ABI]** | Size in bytes of the per-process GPU virtual address space (for 48-bit SMMU IAS `(1ULL << 48) - 0x100000000`). |
| `0x10` | `MSM_PARAM_HIGHEST_BANK_BIT` | RO | `16` | **[ABI]** | Highest DDR bank bit (`16` on SC8280XP LPDDR5). Used by Mesa Turnip to compute image pitch/tile alignment. |
| `0x11` | `MSM_PARAM_RAYTRACING` | RO | `0` | **[ABI]** | Hardware raytracing support (`0` on Adreno 690). |
| `0x12` | `MSM_PARAM_UBWC_SWIZZLE` | RO | `0x00000006` | **[ABI]** | Bitmask of enabled UBWC bank swizzle levels (`0x6` = Levels 2 and 3 enabled on UBWC 4.0). |
| `0x13` | `MSM_PARAM_MACROTILE_MODE` | RO | `1` | **[ABI]** | UBWC macrotile channel mode (`1` = 8-channel macrotiling on SC8280XP). |
| `0x14` | `MSM_PARAM_UCHE_TRAP_BASE` | RO | `0x00001fffffffff000` | **[ABI]** | 64-bit GPU VA of the UCHE trap page (`0x1fffffffff000`), used by shaders for timestamp reads and trap handling. |
| `0x15` | `MSM_PARAM_HAS_PRR` | RO | `0` | **[ABI]** | Partially Resident Region / sparse residency support (`0` unless SMMU PRR hook is present). |
| `0x16` | `MSM_PARAM_EN_VM_BIND` | RW | `0` or `1` | **[ABI]** | Write `1` via `SET_PARAM` (before first VM creation) to opt the DRM file context into explicit `DRM_IOCTL_MSM_VM_BIND` management. |
| `0x17` | `MSM_PARAM_AQE` | RO | `0` | **[ABI]** | Auxiliary Queue Engine support (`0` on Adreno 690). |

---

## 4. Buffer Object Management (`GEM_NEW`, `GEM_INFO`, `CPU_PREP`, `MADVISE`)

### 4.1 `DRM_IOCTL_MSM_GEM_NEW` (`0x42`, `16` bytes)

**[ABI]** Allocates a shmem-backed or DMA-backed GEM buffer object:

| Byte Offset | Size | Type | Field Name | Tag | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `+0x00` | 8 | `uint64` | `size` | **[ABI]** | Requested allocation size in bytes (rounded up to `4096`-byte page size by the kernel). |
| `+0x08` | 4 | `uint32` | `flags` | **[ABI]** | Bitmask of `MSM_BO_*` allocation flags (table below). |
| `+0x0c` | 4 | `uint32` | `handle` | **[ABI]** | Output: returned 32-bit GEM object handle (`> 0`). |

| Flag Constant | Hex Value | Tag | Description |
| :--- | :--- | :--- | :--- |
| `MSM_BO_SCANOUT` | `0x00000001` | **[ABI]** | Scanout-capable buffer. |
| `MSM_BO_GPU_READONLY` | `0x00000002` | **[ABI]** | Map as read-only (`IOMMU_READ` without `IOMMU_WRITE`) in the GPU SMMU page table. |
| `MSM_BO_NO_SHARE` | `0x00000004` | **[ABI]** | Private per-process BO that cannot be exported via PRIME DMA-BUF; may be mapped inside a shared low-VA region by the kernel. |
| `MSM_BO_CACHED` | `0x00010000` | **[ABI]** | CPU-cached mapping requiring explicit cache maintenance. |
| `MSM_BO_WC` | `0x00020000` | **[ABI]** | Write-combining (`WC`) CPU mapping (default for most command/vertex buffers). |
| `MSM_BO_UNCACHED` | `0x00040000` | **[ABI]** | Strongly-ordered uncached mapping. |
| `MSM_BO_CACHED_COHERENT` | `0x00080000` | **[ABI]** | Hardware I/O-coherent cached mapping (`IOMMU_CACHE` enabled in SMMU PTEs; supported on Adreno 690 because `HAS_CACHED_COHERENT` is set). |

### 4.2 `DRM_IOCTL_MSM_GEM_INFO` (`0x43`, `24` bytes)

**[ABI]** Queries or sets per-BO properties:

| Byte Offset | Size | Type | Field Name | Tag | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `+0x00` | 4 | `uint32` | `handle` | **[ABI]** | Target GEM handle. |
| `+0x04` | 4 | `uint32` | `info` | **[ABI]** | Operation code (`MSM_INFO_*` below). |
| `+0x08` | 8 | `uint64` | `value` | **[ABI]** | Output scalar (`GET_OFFSET`, `GET_IOVA`, `GET_FLAGS`), input 64-bit IOVA (`SET_IOVA`), or userspace string pointer (`GET_NAME`, `SET_NAME`). |
| `+0x10` | 4 | `uint32` | `len` | **[ABI]** | Must be `0` for `GET_OFFSET`, `GET_IOVA`, `SET_IOVA`, and `GET_FLAGS`; string buffer length for `GET_NAME` / `SET_NAME`. |
| `+0x14` | 4 | `uint32` | `pad` | **[ABI]** | Must be `0`. |

| `info` Code | Symbolic Name | Tag | Semantics |
| :--- | :--- | :--- | :--- |
| `0x00` | `MSM_INFO_GET_OFFSET` | **[ABI]** | Returns fake 64-bit byte offset in `value` to pass to `mmap(2)` on the DRM fd. |
| `0x01` | `MSM_INFO_GET_IOVA` | **[ABI]** | Allocates (if not yet pinned) and returns the 64-bit GPU virtual address (`IOVA`) of `handle` in the caller's GPU VM (`-EINVAL` if `EN_VM_BIND` is enabled). |
| `0x02` | `MSM_INFO_SET_NAME` | **[ABI]** | Copies up to `len` bytes from user pointer `value` as the BO debug label. |
| `0x03` | `MSM_INFO_GET_NAME` | **[ABI]** | Copies the BO debug label to user pointer `value` and returns actual string length in `len`. |
| `0x04` | `MSM_INFO_SET_IOVA` | **[ABI]** | Binds the BO at the caller-specified 64-bit GPU virtual address `value` (within `[VA_START, VA_START + VA_SIZE)`) when userspace manages its own VA allocator (Mesa Turnip default when `VA_START != 0` and `EN_VM_BIND == 0`). Pass `value = 0` to clear an unassigned IOVA. |
| `0x05` | `MSM_INFO_GET_FLAGS` | **[ABI]** | Returns the BO's immutable `MSM_BO_*` creation flags in `value`. |

### 4.3 `DRM_IOCTL_MSM_GEM_CPU_PREP` (`0x44`, `24` bytes) & `CPU_FINI` (`0x45`, `4` bytes)

**[ABI]** `CPU_PREP` waits until all pending GPU reads/writes (or writes only, if `op` excludes `MSM_PREP_WRITE`) on `handle` complete before `timeout`:
- `+0x00`: `handle` (`uint32`), `+0x04`: `op` (`uint32`: `MSM_PREP_READ = 0x01`, `MSM_PREP_WRITE = 0x02`, `MSM_PREP_NOSYNC = 0x04`), `+0x08`: `timeout.tv_sec` (`int64`, absolute `CLOCK_MONOTONIC` seconds), `+0x10`: `timeout.tv_nsec` (`int64`, nanoseconds). Returns `-EBUSY` if `MSM_PREP_NOSYNC` is set and the BO is busy, or `-ETIMEDOUT` on timeout.
- `CPU_FINI` takes a single 4-byte `handle` (`+0x00`, `uint32`).

### 4.4 `DRM_IOCTL_MSM_GEM_MADVISE` (`0x48`, `12` bytes)

**[ABI]** Controls BO cacheability in userspace BO caches:
- `+0x00`: `handle` (`uint32`), `+0x04`: `madv` (`uint32`: `MSM_MADV_WILLNEED = 0`, `MSM_MADV_DONTNEED = 1`), `+0x08`: `retained` (`uint32`, output: `1` if backing pages have not been purged by the shrinker, `0` if purged).

---

## 5. Submitqueues, Command Submission (`GEM_SUBMIT`), & Fences

### 5.1 Submitqueue Ioctls (`0x4A`, `0x4B`, `0x4C`)

**[ABI]** Before submitting commands, userspace creates at least one submitqueue via `DRM_IOCTL_MSM_SUBMITQUEUE_NEW` (`0x4a`, `12` bytes):

| Byte Offset | Size | Type | Field Name | Tag | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `+0x00` | 4 | `uint32` | `flags` | **[ABI]** | `0` (or `MSM_SUBMITQUEUE_ALLOW_PREEMPT = 0x01`). |
| `+0x04` | 4 | `uint32` | `prio` | **[ABI]** | Priority index in `[0, MSM_PARAM_PRIORITIES - 1]`. Mapped to ringbuffer `prio / 3` and scheduler priority `prio % 3`. |
| `+0x08` | 4 | `uint32` | `id` | **[ABI]** | Output: 32-bit `queueid` passed to `GEM_SUBMIT` and `WAIT_FENCE`. |

- **[ABI]** `DRM_IOCTL_MSM_SUBMITQUEUE_CLOSE` (`0x4b`, `4` bytes): Takes the 32-bit `id` at `+0x00` and destroys the queue.
- **[ABI]** `DRM_IOCTL_MSM_SUBMITQUEUE_QUERY` (`0x4c`, `24` bytes): `+0x00` `data` (`uint64` user pointer), `+0x08` `id` (`uint32` queue ID), `+0x0c` `param` (`uint32`: `MSM_SUBMITQUEUE_PARAM_FAULTS = 0`), `+0x10` `len` (`uint32` byte size of `*data`, `4` bytes), `+0x14` `pad` (`uint32`, `0`). Copies the 32-bit count of GPU faults attributed to queue `id` into `*data`.

### 5.2 `DRM_IOCTL_MSM_GEM_SUBMIT` (`0x46`, `64` bytes) & Sub-Structures

**[ABI]** Submits one or more `IB1` indirect command buffers and their referenced BOs to a submitqueue. **Important Hardware Rule**: On Adreno 6xx and newer (`Adreno 690`), in-kernel BO relocations are **forbidden** (`nr_relocs` in every command descriptor **must be `0`**, or the ioctl returns `-EINVAL`); userspace embeds 64-bit GPU IOVAs directly in command streams using `GET_IOVA`, `SET_IOVA`, or `VM_BIND`.

#### Main Submit Structure (`64` bytes)

| Byte Offset | Size | Type | Field Name | Tag | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `+0x00` | 4 | `uint32` | `flags` | **[ABI]** | Lower 16 bits = `MSM_PIPE_3D0` (`0x0010`); upper bits = `MSM_SUBMIT_*` flags (table below). |
| `+0x04` | 4 | `uint32` | `fence` | **[ABI]** | Output: 32-bit submitqueue-local fence sequence number (`seqno`). |
| `+0x08` | 4 | `uint32` | `nr_bos` | **[ABI]** | Number of entries in the `bos` array (`0` is allowed when `EN_VM_BIND` is active). |
| `+0x0c` | 4 | `uint32` | `nr_cmds` | **[ABI]** | Number of entries in the `cmds` array. |
| `+0x10` | 8 | `uint64` | `bos` | **[ABI]** | Userspace pointer to array of `nr_bos` BO descriptors (`16` bytes each). |
| `+0x18` | 8 | `uint64` | `cmds` | **[ABI]** | Userspace pointer to array of `nr_cmds` command descriptors (`32` bytes each). |
| `+0x20` | 4 | `int32` | `fence_fd` | **[ABI]** | Input `sync_file` fd (if `MSM_SUBMIT_FENCE_FD_IN`) and/or output `sync_file` fd (if `MSM_SUBMIT_FENCE_FD_OUT`). |
| `+0x24` | 4 | `uint32` | `queueid` | **[ABI]** | Target submitqueue `id` from `SUBMITQUEUE_NEW`. |
| `+0x28` | 8 | `uint64` | `in_syncobjs` | **[ABI]** | Userspace pointer to array of `nr_in_syncobjs` syncobj descriptors (`16` bytes each, if `MSM_SUBMIT_SYNCOBJ_IN`). |
| `+0x30` | 8 | `uint64` | `out_syncobjs` | **[ABI]** | Userspace pointer to array of `nr_out_syncobjs` syncobj descriptors (`16` bytes each, if `MSM_SUBMIT_SYNCOBJ_OUT`). |
| `+0x38` | 4 | `uint32` | `nr_in_syncobjs` | **[ABI]** | Count of input syncobjs. |
| `+0x3c` | 4 | `uint32` | `nr_out_syncobjs` | **[ABI]** | Count of output syncobjs. |
| `+0x40` | 4 | `uint32` | `syncobj_stride` | **[ABI]** | Byte stride of elements in `in_syncobjs` / `out_syncobjs` (`16` bytes). |
| `+0x44` | 4 | `uint32` | `pad` | **[ABI]** | Must be `0`. |

| Submit Flag | Hex Value | Tag | Description |
| :--- | :--- | :--- | :--- |
| `MSM_SUBMIT_NO_IMPLICIT` | `0x80000000` | **[ABI]** | Do not wait on implicit DMA-resv fences attached to the submitted BOs. |
| `MSM_SUBMIT_FENCE_FD_IN` | `0x40000000` | **[ABI]** | Wait on input `sync_file` file descriptor in `fence_fd` before executing on hardware. |
| `MSM_SUBMIT_FENCE_FD_OUT` | `0x20000000` | **[ABI]** | Create and return a `sync_file` file descriptor in `fence_fd` representing this submission's completion. |
| `MSM_SUBMIT_SUDO` | `0x10000000` | **[ABI]** | Execute `IB1` buffers with `CP_SET_PROTECTED_MODE` disabled (`CAP_SYS_RAWIO` required). |
| `MSM_SUBMIT_SYNCOBJ_IN` | `0x08000000` | **[ABI]** | Wait on DRM syncobjs in `in_syncobjs`. |
| `MSM_SUBMIT_SYNCOBJ_OUT` | `0x04000000` | **[ABI]** | Signal DRM syncobjs in `out_syncobjs` on completion. |
| `MSM_SUBMIT_FENCE_SN_IN` | `0x02000000` | **[ABI]** | `fence` field supplies an input sequence number to wait on. |

#### Submit Sub-Structures (`bo`, `cmd`, `syncobj`)

**[ABI]** The arrays pointed to by `bos`, `cmds`, and `in_syncobjs`/`out_syncobjs` have the following fixed binary layouts:

| Structure | Total Size | Field Offsets, Types, and Semantics | Tag |
| :--- | :--- | :--- | :--- |
| **BO Entry (`bos[i]`)** | `16` bytes | `+0x00`: `flags` (`uint32`: `MSM_SUBMIT_BO_READ = 0x01`, `MSM_SUBMIT_BO_WRITE = 0x02`, `MSM_SUBMIT_BO_DUMP = 0x04`, `MSM_SUBMIT_BO_NO_IMPLICIT = 0x08`); `+0x04`: `handle` (`uint32` GEM handle); `+0x08`: `presumed` (`uint64`, unused on A690). | **[ABI]** |
| **Command Entry (`cmds[i]`)** | `32` bytes | `+0x00`: `type` (`uint32`: `MSM_SUBMIT_CMD_BUF = 0x01` normal `IB1`, `MSM_SUBMIT_CMD_CTX_RESTORE_BUF = 0x02` only executed if context switched since last submit, `MSM_SUBMIT_CMD_PREEMPT_POSTAMBLE = 0x03`); `+0x04`: `submit_idx` (`uint32` index into `bos[]`, or unused when `EN_VM_BIND` is set); `+0x08`: `submit_offset` (`uint32` byte offset within BO, 4-byte aligned, or lower 32 bits of `iova` when `EN_VM_BIND` is set); `+0x0c`: `size` (`uint32` byte length of command stream, multiple of 4); `+0x10`: `pad` (`uint32`, or upper 32 bits of `iova` when `EN_VM_BIND` is set); `+0x14`: `nr_relocs` (`uint32`, **must be `0` on Adreno 690**); `+0x18`: `relocs` (`uint64`, `0`). | **[ABI]** |
| **Syncobj Entry (`syncobjs[i]`)** | `16` bytes | `+0x00`: `handle` (`uint32` DRM syncobj handle); `+0x04`: `flags` (`uint32`: `MSM_SUBMIT_SYNCOBJ_RESET = 0x01`); `+0x08`: `point` (`uint64` timeline point, `0` for binary syncobj). | **[ABI]** |

### 5.3 `DRM_IOCTL_MSM_WAIT_FENCE` (`0x47`, `32` bytes)

**[ABI]** Blocks until submitqueue `queueid`'s fence sequence number reaches `fence` or `timeout` expires:
- `+0x00`: `fence` (`uint32` seqno), `+0x04`: `flags` (`uint32`: `0` or `MSM_WAIT_FENCE_BOOST = 0x01`), `+0x08`: `timeout.tv_sec` (`int64` `CLOCK_MONOTONIC`), `+0x10`: `timeout.tv_nsec` (`int64`), `+0x18`: `queueid` (`uint32`), `+0x1c`: `pad` (`uint32`, `0`).

---

## 6. Explicit GPU Virtual Memory Management (`DRM_IOCTL_MSM_VM_BIND`)

**[ABI]** When userspace enables `MSM_PARAM_EN_VM_BIND` (`0x16`), it manages GPU page tables via `DRM_IOCTL_MSM_VM_BIND` (`0x4d`, `88` bytes):

| Byte Offset | Size | Type | Field Name | Tag | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `+0x00` | 4 | `uint32` | `flags` | **[ABI]** | Bitmask: `MSM_VM_BIND_FENCE_FD_IN = 0x01`, `MSM_VM_BIND_FENCE_FD_OUT = 0x02`. |
| `+0x04` | 4 | `uint32` | `nr_ops` | **[ABI]** | Number of bind operations (`1` returns inline at `+0x30`; `> 1` reads array at `ops_ptr` at `+0x30`). |
| `+0x08` | 4 | `int32` | `fence_fd` | **[ABI]** | Input/output `sync_file` fd. |
| `+0x0c` | 4 | `uint32` | `queue_id` | **[ABI]** | Submitqueue ID for asynchronous VM bind operations (`0` for synchronous execution). |
| `+0x10` | 8 | `uint64` | `in_syncobjs` | **[ABI]** | Userspace pointer to input syncobj array. |
| `+0x18` | 8 | `uint64` | `out_syncobjs` | **[ABI]** | Userspace pointer to output syncobj array. |
| `+0x20` | 4 | `uint32` | `nr_in_syncobjs` | **[ABI]** | Number of input syncobjs. |
| `+0x24` | 4 | `uint32` | `nr_out_syncobjs` | **[ABI]** | Number of output syncobjs. |
| `+0x28` | 4 | `uint32` | `syncobj_stride` | **[ABI]** | Byte stride of syncobj entries (`16`). |
| `+0x2c` | 4 | `uint32` | `op_stride` | **[ABI]** | Byte stride of bind op entries (`40`). |
| `+0x30` | 40 | `union` | `op` / `ops_ptr` | **[ABI]** | When `nr_ops == 1`, an inline 40-byte bind op descriptor (`+0x30` `op` `uint32`: `1` = `UNMAP`, `2` = `MAP`, `3` = `MAP_NULL`; `+0x34` `handle` `uint32`; `+0x38` `obj_offset` `uint64`; `+0x40` `iova` `uint64`; `+0x48` `range` `uint64`; `+0x50` `flags` `uint32`: `MSM_VM_BIND_OP_DUMP = 0x01`; `+0x54` `pad` `uint32`). When `nr_ops > 1`, `+0x30` holds the 64-bit user pointer `ops_ptr`. |

---

## 7. Mesa Turnip (`TU_DEBUG`) & Freedreno Startup Contract Summary

**[ABI]** To pass Mesa Turnip (`TU_DEBUG=startup,bos,submit`) and Freedreno Gallium initialization on Adreno 690 without errors:
1. **[ABI]** `drmGetVersion()` must return `name = "msm"` and `version_major = 1`, `version_minor >= 6` (recommend `1.13.0`).
2. **[ABI]** `MSM_PARAM_CHIP_ID` (`0x03`) must return `0xffff06090000` (or `0xffff06090001`), which Mesa's device table matches to **Adreno 690 (`FD_DEV_ID = 690`)**.
3. **[ABI]** `MSM_PARAM_GMEM_SIZE` (`0x02`) must return `0x00400000` (`4 MiB`), `MSM_PARAM_GMEM_BASE` (`0x06`) must return `0x0`, `MSM_PARAM_HIGHEST_BANK_BIT` (`0x10`) must return `16`, `MSM_PARAM_UBWC_SWIZZLE` (`0x12`) must return `0x6`, `MSM_PARAM_MACROTILE_MODE` (`0x13`) must return `1`, and `MSM_PARAM_UCHE_TRAP_BASE` (`0x14`) must return `0x1fffffffff000`.
4. **[ABI]** `MSM_PARAM_VA_START` (`0x0e`) and `MSM_PARAM_VA_SIZE` (`0x0f`) must return `0x100000000` and `0xffff00000000` so Turnip can manage its own 48-bit GPU virtual address vma heap via `MSM_INFO_SET_IOVA` (`0x04`) or `DRM_IOCTL_MSM_VM_BIND` (`0x4d`).
