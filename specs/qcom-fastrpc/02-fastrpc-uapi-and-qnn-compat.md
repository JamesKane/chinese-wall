# FastRPC Userspace ABI and QNN/HTP Compatibility

## 1. Character Device Nodes and Protection Domain Security Policy

### 1.1 Character Device Registration
- **[ABI]** For the `CDSP` (`NSP0`) domain (`CDSP_DOMAIN_ID = 3`), the FastRPC subsystem registers **two** miscellaneous character devices so that both signed and unsigned protection domains can be used concurrently:
  1. **`/dev/fastrpc-cdsp` (`secure = false`):** Non-secure device node intended for unprivileged applications and unsigned DSP Protection Domains (such as Qualcomm QNN HTP / SNPE with `FASTRPC_MODE_UNSIGNED_MODULE`).
  2. **`/dev/fastrpc-cdsp-secure` (`secure = true`):** Secure device node intended for signed DSP Protection Domains.
- **[ABI]** On SC8280XP, the `fastrpc` device tree node under `remoteproc_nsp0` specifies `qcom,non-secure-domain`, which marks the channel as non-secure (`secure = false`).
- **[POLICY]** **Session Rejection Rule:**
  - If a channel is configured as secure (`qcom,non-secure-domain` is absent) AND the user opened the non-secure device node (`/dev/fastrpc-cdsp`):
    - Calling `FASTRPC_IOCTL_INIT_CREATE` **without** `FASTRPC_MODE_UNSIGNED_MODULE` (`1 << 3` = `0x8`) in `attrs` (or on a domain that does not support unsigned PDs, such as `ADSP`/`SDSP`/`MDSP`) is rejected with `-ECONNREFUSED`.
  - When `qcom,non-secure-domain` is present (as on SC8280XP `CDSP` and `SDSP`), the channel is marked non-secure, so both signed and unsigned `INIT_CREATE` requests are permitted on `/dev/fastrpc-cdsp`.

---

## 2. Complete `ioctl` Command Table (`'R'` = `0x52`)

- **[ABI]** All FastRPC ioctls use magic byte `'R'` (`0x52`). The exact 32-bit ioctl command numbers on 64-bit ARM (`aarch64`) are:

| Ioctl Name | Number (`nr`) | Direction & Size | Exact Hex Command (`aarch64`) | Argument Type | Description | Tag |
|---|---|---|---|---|---|---|
| `FASTRPC_IOCTL_ALLOC_DMA_BUFF` | `1` | `_IOWR('R', 1, 16)` | `0xc0105201` | `fastrpc_alloc_dma_buf` (`16` B) | Allocates a coherent SMMU DMA-BUF and returns a file descriptor | **[ABI]** |
| `FASTRPC_IOCTL_FREE_DMA_BUFF` | `2` | `_IOWR('R', 2, 4)` | `0xc0045202` | `u32` (`4` B) | Legacy/unimplemented (`-ENOTTY`); DMA-BUFs are freed by `close(fd)` | **[ABI]** |
| `FASTRPC_IOCTL_INVOKE` | `3` | `_IOWR('R', 3, 16)` | `0xc0105203` | `fastrpc_invoke` (`16` B) | Invokes an RPC method on a remote handle (`handle != 1`) | **[ABI]** |
| `FASTRPC_IOCTL_INIT_ATTACH` | `4` | `_IO('R', 4)` | `0x00005204` | None (`0` B) | Attaches the session to the DSP `ROOT_PD` (`pd = 0`) | **[ABI]** |
| `FASTRPC_IOCTL_INIT_CREATE` | `5` | `_IOWR('R', 5, 24)` | `0xc0185205` | `fastrpc_init_create` (`24` B) | Spawns a dynamic `USER_PD` (`pd = 1`) process using a shell ELF (`fastrpc_shell_3` or `fastrpc_shell_unsigned_3`) | **[ABI]** |
| `FASTRPC_IOCTL_MMAP` | `6` | `_IOWR('R', 6, 32)` | `0xc0205206` | `fastrpc_req_mmap` (`32` B) | Allocates and maps pages (`ADSP_MMAP_ADD_PAGES = 0x1000`) into the DSP user heap | **[ABI]** |
| `FASTRPC_IOCTL_MUNMAP` | `7` | `_IOWR('R', 7, 16)` | `0xc0105207` | `fastrpc_req_munmap` (`16` B) | Unmaps and frees pages previously mapped via `FASTRPC_IOCTL_MMAP` | **[ABI]** |
| `FASTRPC_IOCTL_INIT_ATTACH_SNS` | `8` | `_IO('R', 8)` | `0x00005208` | None (`0` B) | Attaches the session to the DSP `SENSORS_PD` (`pd = 2`) | **[ABI]** |
| `FASTRPC_IOCTL_INIT_CREATE_STATIC` | `9` | `_IOWR('R', 9, 16)` | `0xc0105209` | `fastrpc_init_create_static` (`16` B) | Attaches to a static protection domain (e.g., `"audiopd"` on ADSP) | **[ABI]** |
| `FASTRPC_IOCTL_MEM_MAP` | `10` | `_IOWR('R', 10, 64)` | `0xc040520a` | `fastrpc_mem_map` (`64` B) | Maps an existing DMA-BUF file descriptor into the DSP SMMU and DSP virtual address space | **[ABI]** |
| `FASTRPC_IOCTL_MEM_UNMAP` | `11` | `_IOWR('R', 11, 48)` | `0xc030520b` | `fastrpc_mem_unmap` (`48` B) | Unmaps a DMA-BUF previously mapped via `FASTRPC_IOCTL_MEM_MAP` | **[ABI]** |
| `FASTRPC_IOCTL_SET_OPTION` | `12` | `_IOWR('R', 12, 32)` | `0xc020520c` | `fastrpc_ioctl_set_option` (`32` B) | Enables or disables poll mode (`FASTRPC_POLL_MODE = 1`) on the session | **[ABI]** |
| `FASTRPC_IOCTL_GET_DSP_INFO` | `13` | `_IOWR('R', 13, 28)` | `0xc01c520d` | `fastrpc_ioctl_capability` (`28` B) | Queries DSP hardware/firmware capability attribute `attribute_id` (`0..255`) | **[ABI]** |

---

## 3. Binary Structure Layouts and Enumerations

### 3.1 `FASTRPC_IOCTL_INVOKE` (`0xc0105203`) Structures
- **[ABI]** **`fastrpc_invoke` (`16` Bytes):**

| Byte Offset | Size | Type | Field Name | Description | Tag |
|---|---|---|---|---|---|
| `+0x00` | `4` | `u32` | `handle` | Remote RPC handle (`> 1`; `2` = DSP utilities, `3` = `adsp_current_process1`, `4` = `adsp_listener1`, `> 20` = dynamically loaded `.so` skel handles) | **[ABI]** |
| `+0x04` | `4` | `u32` | `sc` | 32-bit Scalars descriptor (`method`, `inbufs`, `outbufs`, `inhandles`, `outhandles`) | **[ABI]** |
| `+0x08` | `8` | `u64` | `args` | Userspace pointer to an array of `nscalars` `fastrpc_invoke_args` structures (`24` bytes each) | **[ABI]** |

- **[ABI]** **`fastrpc_invoke_args` (`24` Bytes per entry):**

| Byte Offset | Size | Type | Field Name | Description | Tag |
|---|---|---|---|---|---|
| `+0x00` | `8` | `u64` | `ptr` | Userspace virtual address of buffer (for `i < nbufs`), or 32-bit byte offset (for DMA handles `i >= nbufs`) | **[ABI]** |
| `+0x08` | `8` | `u64` | `length` | Byte length of buffer or DMA handle mapping | **[ABI]** |
| `+0x10` | `4` | `s32` | `fd` | DMA-BUF file descriptor (`0` or `-1` if buffer is plain userspace memory to be copied inline) | **[ABI]** |
| `+0x14` | `4` | `u32` | `attr` | Buffer attributes (`0` = normal, `1` = `FASTRPC_ATTR_SECUREMAP` for hyp-assigned VMID memory) | **[ABI]** |

### 3.2 `FASTRPC_IOCTL_INIT_CREATE` (`0xc0185205`) and Process Attributes
- **[ABI]** **`fastrpc_init_create` (`24` Bytes):**

| Byte Offset | Size | Type | Field Name | Description | Tag |
|---|---|---|---|---|---|
| `+0x00` | `4` | `u32` | `filelen` | Byte length of the DSP shell ELF file (`<= 2,097,152` bytes) | **[ABI]** |
| `+0x04` | `4` | `s32` | `filefd` | DMA-BUF file descriptor holding the shell ELF file contents | **[ABI]** |
| `+0x08` | `4` | `u32` | `attrs` | Bitmask of `fastrpc_proc_attr` flags (see table below) | **[ABI]** |
| `+0x0c` | `4` | `u32` | `siglen` | Signature length in bytes (`0` for unsigned shell) | **[ABI]** |
| `+0x10` | `8` | `u64` | `file` | Userspace virtual address of the mapped shell ELF buffer | **[ABI]** |

- **[ABI]** **Process Attribute Flags (`fastrpc_proc_attr`):**

| Flag Name | Hex Value | Bit | Description | Tag |
|---|---|---|---|---|
| `FASTRPC_MODE_DEBUG` | `0x00000001` | `0` | Enable DSP process debugging | **[ABI]** |
| `FASTRPC_MODE_PTRACE` | `0x00000002` | `1` | Enable DSP ptrace | **[ABI]** |
| `FASTRPC_MODE_CRC` | `0x00000004` | `2` | Enable buffer CRC verification | **[ABI]** |
| `FASTRPC_MODE_UNSIGNED_MODULE` | `0x00000008` | `3` | **Unsigned Protection Domain:** Launches `fastrpc_shell_unsigned_3` without OEM code-signing verification; used by QNN HTP and open-source DSP compute workloads | **[ABI]** |
| `FASTRPC_MODE_ADAPTIVE_QOS` | `0x00000010` | `4` | Enable Adaptive QoS on DSP | **[ABI]** |
| `FASTRPC_MODE_SYSTEM_PROCESS` | `0x00000020` | `5` | Mark as system process | **[ABI]** |
| `FASTRPC_MODE_PRIVILEGED` | `0x00000040` | `6` | Mark as privileged process | **[ABI]** |

### 3.3 `FASTRPC_IOCTL_INIT_CREATE_STATIC` (`0xc0105209`)
- **[ABI]** **`fastrpc_init_create_static` (`16` Bytes):**

| Byte Offset | Size | Type | Field Name | Description | Tag |
|---|---|---|---|---|---|
| `+0x00` | `4` | `u32` | `namelen` | Length of static PD process name including null terminator (`<= 128` bytes) | **[ABI]** |
| `+0x04` | `4` | `u32` | `memlen` | Memory length | **[ABI]** |
| `+0x08` | `8` | `u64` | `name` | Userspace pointer to static PD process name ASCII string | **[ABI]** |

### 3.4 `FASTRPC_IOCTL_ALLOC_DMA_BUFF` (`0xc0105201`)
- **[ABI]** **`fastrpc_alloc_dma_buf` (`16` Bytes):**

| Byte Offset | Size | Type | Field Name | Description | Tag |
|---|---|---|---|---|---|
| `+0x00` | `4` | `s32` | `fd` | **[Out]** Returned DMA-BUF file descriptor (`O_RDWR`) | **[ABI]** |
| `+0x04` | `4` | `u32` | `flags` | Allocation flags | **[ABI]** |
| `+0x08` | `8` | `u64` | `size` | **[In]** Requested allocation size in bytes | **[ABI]** |

### 3.5 `FASTRPC_IOCTL_MMAP` (`0xc0205206`) and `FASTRPC_IOCTL_MUNMAP` (`0xc0105207`)
- **[ABI]** **`fastrpc_req_mmap` (`32` Bytes):**
  - Note: Only `flags == ADSP_MMAP_ADD_PAGES` (`0x1000`) and `vaddrin == 0` are supported by this ioctl; any other `flags` or non-zero `vaddrin` returns `-EINVAL`.

| Byte Offset | Size | Type | Field Name | Description | Tag |
|---|---|---|---|---|---|
| `+0x00` | `4` | `s32` | `fd` | File descriptor (unused for `ADSP_MMAP_ADD_PAGES`) | **[ABI]** |
| `+0x04` | `4` | `u32` | `flags` | Must be `0x1000` (`ADSP_MMAP_ADD_PAGES`) | **[ABI]** |
| `+0x08` | `8` | `u64` | `vaddrin` | Must be `0` | **[ABI]** |
| `+0x10` | `8` | `u64` | `size` | Byte size of heap pages to allocate and add to DSP User PD | **[ABI]** |
| `+0x18` | `8` | `u64` | `vaddrout` | **[Out]** DSP virtual address assigned to the mapped region | **[ABI]** |

- **[ABI]** **`fastrpc_req_munmap` (`16` Bytes):**

| Byte Offset | Size | Type | Field Name | Description | Tag |
|---|---|---|---|---|---|
| `+0x00` | `8` | `u64` | `vaddrout` | **[In]** DSP virtual address returned by `FASTRPC_IOCTL_MMAP` | **[ABI]** |
| `+0x08` | `8` | `u64` | `size` | **[In]** Byte size of mapping | **[ABI]** |

### 3.6 `FASTRPC_IOCTL_MEM_MAP` (`0xc040520a`) and `FASTRPC_IOCTL_MEM_UNMAP` (`0xc030520b`)
- **[ABI]** **`fastrpc_mem_map` (`64` Bytes):** Used by QNN HTP and modern `libcdsprpc` to map shared neural network weight/tensor DMA-BUFs persistently into the DSP User PD.

| Byte Offset | Size | Type | Field Name | Description | Tag |
|---|---|---|---|---|---|
| `+0x00` | `4` | `s32` | `version` | Structure version (`0`) | **[ABI]** |
| `+0x04` | `4` | `s32` | `fd` | **[In]** DMA-BUF file descriptor to map | **[ABI]** |
| `+0x08` | `4` | `s32` | `offset` | **[In]** Byte offset within the buffer | **[ABI]** |
| `+0x0c` | `4` | `u32` | `flags` | **[In]** Mapping control flag from `fastrpc_map_flags`: `0` = `FASTRPC_MAP_STATIC`, `1` = `FASTRPC_MAP_RESERVED`, `2` = `FASTRPC_MAP_FD`, `3` = `FASTRPC_MAP_FD_DELAYED`, `16` = `FASTRPC_MAP_FD_NOMAP` | **[ABI]** |
| `+0x10` | `8` | `u64` | `vaddrin` | **[In]** Host userspace virtual address of buffer | **[ABI]** |
| `+0x18` | `8` | `u64` | `length` | **[In]** Byte length to map | **[ABI]** |
| `+0x20` | `8` | `u64` | `vaddrout` | **[Out]** Remote DSP virtual address returned by firmware | **[ABI]** |
| `+0x28` | `4` | `s32` | `attrs` | Buffer attributes | **[ABI]** |
| `+0x2c` | `16` | `s32[4]` | `reserved` | Reserved (`0`) | **[ABI]** |

- **[ABI]** **`fastrpc_mem_unmap` (`48` Bytes):**

| Byte Offset | Size | Type | Field Name | Description | Tag |
|---|---|---|---|---|---|
| `+0x00` | `4` | `s32` | `version` | Structure version (`0`; misspelled `vesion` in Linux header, at offset `0`) | **[ABI]** |
| `+0x04` | `4` | `s32` | `fd` | **[In]** DMA-BUF file descriptor (or `< 0` to match by `vaddr` only) | **[ABI]** |
| `+0x08` | `8` | `u64` | `vaddr` | **[In]** Remote DSP virtual address (`vaddrout` from `MEM_MAP`) | **[ABI]** |
| `+0x10` | `8` | `u64` | `length` | **[In]** Byte length of mapping | **[ABI]** |
| `+0x18` | `20` | `s32[5]` | `reserved` | Reserved (`0`) | **[ABI]** |

### 3.7 `FASTRPC_IOCTL_SET_OPTION` (`0xc020520c`) and `FASTRPC_IOCTL_GET_DSP_INFO` (`0xc01c520d`)
- **[ABI]** **`fastrpc_ioctl_set_option` (`32` Bytes):**
  - All 6 `reserved` words must be `0` (otherwise returns `-EINVAL`).
  - `request_id` must be `1` (`FASTRPC_POLL_MODE`); `value` must be `0` (`DISABLE`) or `1` (`ENABLE`).
  - Note: On SC8280XP, `poll_mode_supported` is `false` in the default SoC configuration and SC8280XP is not in the poll-mode exception machine list, so `FASTRPC_IOCTL_SET_OPTION` returns `-EOPNOTSUPP` unless explicitly enabled by the OS driver.

| Byte Offset | Size | Type | Field Name | Description | Tag |
|---|---|---|---|---|---|
| `+0x00` | `4` | `u32` | `request_id` | `1` (`FASTRPC_POLL_MODE`) | **[ABI]** |
| `+0x04` | `4` | `u32` | `value` | `0` (`FASTRPC_POLL_MODE_DISABLE`) or `1` (`FASTRPC_POLL_MODE_ENABLE`) | **[ABI]** |
| `+0x08` | `24` | `s32[6]` | `reserved` | Must be zero (`0`) | **[ABI]** |

- **[ABI]** **`fastrpc_ioctl_capability` (`28` Bytes):**

| Byte Offset | Size | Type | Field Name | Description | Tag |
|---|---|---|---|---|---|
| `+0x00` | `4` | `u32` | `unused` | Deprecated (`domain` in older headers), ignored by kernel | **[ABI]** |
| `+0x04` | `4` | `u32` | `attribute_id` | **[In]** Capability attribute index (`0`..`255`; `>= 256` returns `-EOVERFLOW`) | **[ABI]** |
| `+0x08` | `4` | `u32` | `capability` | **[Out]** Returned 32-bit capability value | **[ABI]** |
| `+0x0c` | `16` | `u32[4]` | `reserved` | Reserved (`0`) | **[ABI]** |

---

## 4. End-to-End Userspace Runtime Sequence (`libcdsprpc.so` / QNN HTP)

- **[ABI]** A typical Qualcomm QNN HTP or `libcdsprpc.so` session on the Radxa Dragon Q8B executes the following ioctl sequence on `/dev/fastrpc-cdsp`:
  1. **Open `/dev/fastrpc-cdsp`:** Kernel allocates one of the 14 SMMU sessions (`compute-cb@1`..`14`, `sid = 1..14`, `client_id = 1..14`).
  2. **Query DSP Capabilities (`FASTRPC_IOCTL_GET_DSP_INFO`):** Runtime queries attributes (such as Hexagon architecture version `v68`, VTCM size, HVX/HMX support, unsigned PD support, and async FastRPC support). Kernel invokes `handle = 2` (`FASTRPC_DSP_UTILITIES_HANDLE`) on the first query and caches all 256 attributes.
  3. **Allocate Shell Buffer (`FASTRPC_IOCTL_ALLOC_DMA_BUFF`):** Runtime allocates a coherent DMA-BUF, `mmap()`s it into userspace, and reads `/usr/lib/dsp/cdsp/fastrpc_shell_unsigned_3` (or `fastrpc_shell_3`) into the buffer.
  4. **Spawn DSP User PD (`FASTRPC_IOCTL_INIT_CREATE`):** Runtime passes `filefd`, `filelen`, `file` vaddr, and `attrs = FASTRPC_MODE_UNSIGNED_MODULE` (`0x8`). Kernel allocates the >= 2 MiB `init_mem` buffer in the SMMU session and invokes `handle = 1`, method `7` (`FASTRPC_RMID_INIT_CREATE_ATTR`).
  5. **Reverse RPC Listener Loop & Module Loading (`FASTRPC_IOCTL_INVOKE`):**
     - A background listener thread in `libcdsprpc.so` invokes `handle = 4` (`adsp_listener`) in a loop so the DSP User PD can make reverse RPC calls back to the host to `dlopen` / read DSP shared libraries (e.g., `libQnnHtpV68Skel.so`) from the host filesystem.
     - Main thread invokes `handle = 0` / dynamic module loader handle to open `libQnnHtpV68Skel.so`, receiving a dynamic `handle > 20`.
  6. **Map Tensor/Weight Buffers (`FASTRPC_IOCTL_MEM_MAP`) & Execute Graphs (`FASTRPC_IOCTL_INVOKE`):**
     - Runtime allocates DMA-BUFs via `FASTRPC_IOCTL_ALLOC_DMA_BUFF`, maps them into the DSP via `FASTRPC_IOCTL_MEM_MAP` (`FASTRPC_MAP_FD`), and executes QNN graphs via `FASTRPC_IOCTL_INVOKE` on the dynamic handle (`> 20`).
  7. **Close `/dev/fastrpc-cdsp`:** Kernel automatically invokes `handle = 1`, method `1` (`FASTRPC_RMID_INIT_RELEASE`), unmaps all remaining `maps` and `mmaps` from the SMMU Context Bank, frees `init_mem`, and marks the SMMU session slot free.
