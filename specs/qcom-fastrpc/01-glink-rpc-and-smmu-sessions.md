# GLINK RPC Transport, SMMU Compute Sessions, and Invocation Buffer Protocol

## 1. Domain Identifiers, GLINK Transport Binding, and SMMU Sessions

### 1.1 FastRPC Domain Identifiers
- **[HW]** FastRPC identifies target DSP subsystems by a domain string and numeric domain ID:

| Domain Label | Numeric Domain ID | Unsigned PD Offload Supported | Device Nodes Registered | Tag |
|---|---|---|---|---|
| `"adsp"` | `0` (`ADSP_DOMAIN_ID`) | No (`false`) | `/dev/fastrpc-adsp` (or `-secure`) | **[ABI]** |
| `"mdsp"` | `1` (`MDSP_DOMAIN_ID`) | No (`false`) | `/dev/fastrpc-mdsp` (or `-secure`) | **[ABI]** |
| `"sdsp"` | `2` (`SDSP_DOMAIN_ID`) | No (`false`) | `/dev/fastrpc-sdsp` (or `-secure`) | **[ABI]** |
| `"cdsp"` | `3` (`CDSP_DOMAIN_ID`) | Yes (`true`) | `/dev/fastrpc-cdsp` AND `/dev/fastrpc-cdsp-secure` | **[ABI]** |
| `"gdsp"` | `4` (`GDSP_DOMAIN_ID`) | Yes (`true`) | `/dev/fastrpc-gdsp` AND `/dev/fastrpc-gdsp-secure` | **[ABI]** |

- **[HW]** When the `CDSP` (`NSP0`) firmware (`qcom/sc8280xp/qccdsp8280.mbn`) finishes booting via PAS ID `18` (`0x12`) and completes the SMP2P `"ready"`/`"handover"` handshake, it opens the GLINK (`rpmsg`) channel named **`"fastrpcglink-apps-dsp"`** on the `nsp0` GLINK edge (remote PID `5`, IPCC client `IPCC_CLIENT_CDSP = 6`, signal `IPCC_MPROC_SIGNAL_GLINK_QMP = 0`).

### 1.2 Apps SMMU Compute Context Banks (`compute-cb@1`..`compute-cb@14`)
- **[HW]** Each open file descriptor (`/dev/fastrpc-cdsp` or `/dev/fastrpc-cdsp-secure`) is assigned an exclusive SMMU session slot (`session_index` `0`..`13`, `client_id = session_index + 1`, up to `FASTRPC_MAX_SESSIONS = 14`) backed by a dedicated Context Bank (`CB`) on the Apps SMMU (`0x15000000`). If all 14 sessions are in use, opening the character device returns `EBUSY`.
- **[HW]** On SC8280XP, the 14 `CDSP` compute sessions have the following Session IDs (`sid`, `reg` property) and Apps SMMU Stream IDs (all with Stream Match Mask `0x0420` and hardware cache-coherent `dma-coherent`):

| Session Node | Session ID (`sid`) | `client_id` | Apps SMMU Stream ID | Apps SMMU Mask | DMA Mask Bits | Tag |
|---|---|---|---|---|---|---|
| `compute-cb@1` | `1` | `1` | `0x3181` | `0x0420` | `32` (`0xffffffff`) | **[HW]** |
| `compute-cb@2` | `2` | `2` | `0x3182` | `0x0420` | `32` (`0xffffffff`) | **[HW]** |
| `compute-cb@3` | `3` | `3` | `0x3183` | `0x0420` | `32` (`0xffffffff`) | **[HW]** |
| `compute-cb@4` | `4` | `4` | `0x3184` | `0x0420` | `32` (`0xffffffff`) | **[HW]** |
| `compute-cb@5` | `5` | `5` | `0x3185` | `0x0420` | `32` (`0xffffffff`) | **[HW]** |
| `compute-cb@6` | `6` | `6` | `0x3186` | `0x0420` | `32` (`0xffffffff`) | **[HW]** |
| `compute-cb@7` | `7` | `7` | `0x3187` | `0x0420` | `32` (`0xffffffff`) | **[HW]** |
| `compute-cb@8` | `8` | `8` | `0x3188` | `0x0420` | `32` (`0xffffffff`) | **[HW]** |
| `compute-cb@9` | `9` | `9` | `0x318b` | `0x0420` | `32` (`0xffffffff`) | **[HW]** |
| `compute-cb@10` | `10` | `10` | `0x318b` | `0x0420` | `32` (`0xffffffff`) | **[HW]** |
| `compute-cb@11` | `11` | `11` | `0x318c` | `0x0420` | `32` (`0xffffffff`) | **[HW]** |
| `compute-cb@12` | `12` | `12` | `0x318d` | `0x0420` | `32` (`0xffffffff`) | **[HW]** |
| `compute-cb@13` | `13` | `13` | `0x318e` | `0x0420` | `32` (`0xffffffff`) | **[HW]** |
| `compute-cb@14` | `14` | `14` | `0x318f` | `0x0420` | `32` (`0xffffffff`) | **[HW]** |

- **[HW]** Note: For `SLPI` / `SDSP` (`fastrpc` under `slpi` GLINK edge, remote PID `3`), 3 sessions exist (`compute-cb@1`..`3` with `sid = 1..3` and SMMU Stream IDs `0x0521`, `0x0522`, `0x0523`, mask `0x0000`).

### 1.3 Consolidated 64-Bit SMMU Address Encoding (`sid_pos`)
- **[HW]** When the DSP accesses host memory through the Apps SMMU, the DSP hardware uses the upper bits of the 64-bit address to select the SMMU Stream ID (`sid`) and the lower bits as the 32-bit IOVA within that Context Bank:
  - On SC8280XP, `sid_pos = 32` and `dma_addr_bits = 32` (whereas on newer Kaanapali SoCs, `sid_pos = 56` and `dma_addr_bits_cdsp = 34`).
  - Therefore, every DMA address (`addr`) sent to the CDSP in GLINK messages or page lists **must** be formatted as:
    `consolidated_dma_addr = ((u64)sid << 32) | (iova & 0xffffffffULL)`
  - Conversely, when freeing or unmapping a buffer on the host SMMU, the host strips the upper bits to recover the raw 32-bit SMMU IOVA:
    `raw_iova = consolidated_dma_addr & 0xffffffffULL` (i.e., masking bits `[sid_pos - 1 : 0]`).

---

## 2. GLINK Message Wire Format

### 2.1 Host-to-DSP Invoke Request Message (`40` Bytes)
- **[HW]** Every RPC invocation sends a **40-byte (`0x28`-byte)** packet over the `"fastrpcglink-apps-dsp"` GLINK endpoint:

| Byte Offset | Size | Type | Field Name | Description | Tag |
|---|---|---|---|---|---|
| `+0x00` | `4` | `s32` | `client_id` | `0` for kernel-initiated internal RPC calls (`kernel == true`); `session_idx + 1` (`1`..`14`) for userspace-initiated RPC calls (`kernel == false`) | **[HW]** |
| `+0x04` | `4` | `s32` | `tid` | Caller thread/process ID (`pid`) | **[HW]** |
| `+0x08` | `8` | `u64` | `ctx` | Invocation context identifier: `ctxid \| pd` (see Section 2.3) | **[HW]** |
| `+0x10` | `4` | `u32` | `handle` | Target RPC handle (`1` = `FASTRPC_INIT_HANDLE`, `2` = `FASTRPC_DSP_UTILITIES_HANDLE`, `3..20` = static handles, `> 20` = dynamic loaded module handles) | **[HW]** |
| `+0x14` | `4` | `u32` | `sc` | 32-bit Scalars descriptor encoding method ID and argument counts (see Section 2.4) | **[HW]** |
| `+0x18` | `8` | `u64` | `addr` | Consolidated 64-bit DMA address (`(sid << 32) \| iova`) of the Invocation Metadata/Payload Buffer (`0` if `nscalars == 0`) | **[HW]** |
| `+0x20` | `8` | `u64` | `size` | Byte size of the Invocation Metadata/Payload Buffer, **rounded up to a 4 KiB (`0x1000`) page boundary** | **[HW]** |

### 2.2 DSP-to-Host Invoke Response Message (`16` Bytes)
- **[HW]** When an RPC invocation completes, the DSP sends a **16-byte (`0x10`-byte)** response packet over the `"fastrpcglink-apps-dsp"` GLINK endpoint:

| Byte Offset | Size | Type | Field Name | Description | Tag |
|---|---|---|---|---|---|
| `+0x00` | `8` | `u64` | `ctx` | Echoed `ctx` value from the request (`ctxid \| pd`), or the fixed sentinel `0x00000000ABCDABCD` for unsolicited Protection Domain exit notifications | **[HW]** |
| `+0x08` | `4` | `s32` | `retval` | Signed 32-bit DSP return status (`0` = success; non-zero = DSP error code, e.g., `0x80000414` = `DSP_UNSUPPORTED_API`) | **[HW]** |
| `+0x0c` | `4` | `u32` | `padding` | Alignment padding ( total message length >= 12 or 16 bytes) | **[HW]** |

### 2.3 Context ID (`ctx`) Bitfield Encoding and Sentinel Handling
- **[HW]** The 64-bit `ctx` field in the GLINK request and response messages is structured as follows:
  - **Bits `[3:0]`:** Protection Domain ID (`pd`):
    - `0` = `ROOT_PD` (Root Protection Domain, used by `FASTRPC_IOCTL_INIT_ATTACH`).
    - `1` = `USER_PD` (User Protection Domain, used by `FASTRPC_IOCTL_INIT_CREATE` and `FASTRPC_IOCTL_INIT_CREATE_STATIC`).
    - `2` = `SENSORS_PD` (Sensors Protection Domain, used by `FASTRPC_IOCTL_INIT_ATTACH_SNS`).
  - **Bits `[7:4]`:** Reserved (`0`).
  - **Bits `[15:8]` (`FASTRPC_CTXID_MASK`):** Active context slot index (`1`..`255`, maximum `FASTRPC_CTX_MAX = 256` concurrent contexts per channel).
  - **Bits `[63:16]` (`FASTRPC_CTXID_SEQ_MASK`):** 48-bit monotonically incrementing per-channel sequence number.
- **[HW]** **Response Validation Rules:**
  1. If `rsp.ctx == 0xABCDABCD` (`FASTRPC_DSP_PD_NOTIFY_CTX`), the message is an unsolicited PD exit notification from newer DSP firmware. Silently drop it and return `0` without logging an error.
  2. Extract `slot_id = (rsp.ctx >> 8) & 0xff` and look up the active context in that slot.
  3. Compare `(rsp.ctx >> 16)` against the active context's sequence number `(ctx.ctxid >> 16)`. If they do not match, the response is a stale or duplicate GLINK message for a previously recycled slot; silently drop it so the new occupant of `slot_id` is not prematurely woken.

### 2.4 Scalars (`sc`) 32-Bit Descriptor Encoding
- **[HW]** The 32-bit `sc` field encodes the method index on the target `handle` and the counts of the four categories of arguments:

| Bits | Width | Field Name | Extraction Formula | Description | Tag |
|---|---|---|---|---|---|
| `[31:29]` | `3` | `attr` | `(sc >> 29) & 0x07` | Call attributes (`0` for standard calls) | **[HW]** |
| `[28:24]` | `5` | `method` | `(sc >> 24) & 0x1f` | Remote method ID (`0`..`31`) on the target `handle` | **[HW]** |
| `[23:16]` | `8` | `inbufs` | `(sc >> 16) & 0xff` | Number of input data buffers (`0`..`255`) | **[HW]** |
| `[15:8]` | `8` | `outbufs` | `(sc >> 8) & 0xff` | Number of output data buffers (`0`..`255`) | **[HW]** |
| `[7:4]` | `4` | `inhandles` | `(sc >> 4) & 0x0f` | Number of input DMA-BUF file descriptor handles (`0`..`15`) | **[HW]** |
| `[3:0]` | `4` | `outhandles` | `sc & 0x0f` | Number of output DMA-BUF file descriptor handles (`0`..`15`) | **[HW]** |

- **[HW]** Derived counts:
  - `nbufs = inbufs + outbufs` (total data buffer arguments).
  - `nhandles = inhandles + outhandles` (total DMA handle arguments).
  - `nscalars = inbufs + outbufs + inhandles + outhandles` (total argument entries).

---

## 3. DMA Invocation Buffer Layout (Metadata and Inline Payload)

### 3.1 Overall Buffer Layout in Coherent DMA Memory
- **[HW]** For every invocation with `nscalars > 0`, the host allocates a contiguous coherent DMA buffer from the session's SMMU Context Bank (or from the `rpdev` device if `sid == 0`).
- **[HW]** Let `N = nscalars`. The metadata region at the start of the DMA buffer has exact byte size:
  `metalen = (16 + 8 + 16) * N + (8 * 16) + (4 * 64) + 4 = (40 * N) + 388` bytes.
- **[HW]** The sections inside the DMA buffer are laid out in the following exact order:

| Offset (Bytes) | Size (Bytes) | Element Type & Count | Section Name & Description | Tag |
|---|---|---|---|---|
| `0` | `16 * N` | `RemoteArg[N]` (`16` B each) | **1. Remote Arguments Array (`rpra`):** For indices `0 .. nbufs - 1`: `pv` (`u64` at `+0x00`) and `len` (`u64` at `+0x08`). For indices `nbufs .. N - 1` (DMA handles): `fd` (`s32` at `+0x00`), `offset` (`u32` at `+0x04`), `len` (`u32` at `+0x08`), and `pad` (`u32` at `+0x0c`) | **[HW]** |
| `16 * N` | `8 * N` | `InvokeBuf[N]` (`8` B each) | **2. Invoke Buffer List (`list`):** `num` (`u32` at `+0x00`) and `pgidx` (`u32` at `+0x04`). For each argument `i` (`0 .. N - 1`), `num = (length != 0) ? 1 : 0` and `pgidx = i` | **[HW]** |
| `24 * N` | `16 * N` | `PhyPage[N]` (`16` B each) | **3. Physical/SMMU Page Info List (`pages`):** `addr` (`u64` at `+0x00`) and `size` (`u64` at `+0x08`). Gives the consolidated SMMU address `(sid << 32) \| iova` and page-aligned byte size of each argument's backing memory | **[HW]** |
| `40 * N` | `128` | `u64[16]` (`FASTRPC_MAX_FDLIST = 16`) | **4. FD List (`fdlist`):** Written by the DSP during execution with up to 16 DMA-BUF file descriptors that the DSP unmapped and wants the host to release after the call returns | **[HW]** |
| `40 * N + 128` | `256` | `u32[64]` (`FASTRPC_MAX_CRCLIST = 64`) | **5. CRC List (`crclist`):** 64-entry 32-bit CRC check array | **[HW]** |
| `40 * N + 384` | `4` | `u32` | **6. Poll Completion Word (`poll_addr`):** In poll mode, the DSP writes `0x000decaf` (`FASTRPC_POLL_RESPONSE`) here when the RPC call finishes | **[HW]** |
| `round_up(40 * N + 388, 128)` | Variable | `u8[]` (`128`-byte aligned chunks) | **7. Inline Arguments Area:** Copies of non-FD (`fd == 0` or `fd == -1`) input and output buffers, each non-overlapping buffer starting at a **128-byte (`FASTRPC_ALIGN = 128`)** aligned offset | **[HW]** |

### 3.2 Overlap Deduplication and Inline vs. Mapped Buffer Rules
- **[HW]** Userspace callers frequently pass multiple buffer arguments (`i < nbufs`) whose virtual address ranges `[ptr, ptr + length)` overlap (for example, a struct header and a sub-array inside the same struct). FastRPC deduplicates overlapping virtual address ranges before copying them into the Inline Arguments Area:
  1. Construct an overlap array `olaps[0 .. nbufs - 1]` with `start = args[i].ptr`, `end = start + args[i].length`, `raix = i`.
  2. Sort `olaps` in ascending order of `start`, breaking ties by descending order of `end` (largest enclosing buffer first).
  3. Sweep through the sorted `olaps` maintaining `max_end`:
     - If `olaps[oix].start >= max_end`: this buffer starts a new disjoint region (`mstart = start`, `mend = end`, `offset = 0`, `max_end = end`). Its inline copy is aligned up to a **128-byte (`FASTRPC_ALIGN`)** boundary.
     - If `olaps[oix].start < max_end`: this buffer overlaps the previous region (`offset = max_end - start`). If `end > max_end`, only the tail `[max_end, end)` (`mend - mstart = end - max_end`) is appended without 128-byte alignment padding, and `max_end = end`; otherwise (`end <= max_end`), the buffer is completely contained inside the already-allocated region (`mstart = 0, mend = 0`).
- **[HW]** Populating `rpra[i]`, `list[i]`, and `pages[i]` for each argument:
  1. **Data Buffers (`0 <= i < nbufs`) backed by a DMA-BUF (`fd != 0 && fd != -1`):**
     - Map the DMA-BUF into the session's SMMU Context Bank (`maps[i]`).
     - `rpra[i].buf.pv = args[i].ptr`, `rpra[i].buf.len = args[i].length`.
     - `pages[i].addr = maps[i].consolidated_dma_addr` (plus `(args[i].ptr & PAGE_MASK) - vma.vm_start` if `args[i].ptr` is offset within a CPU VMA).
     - `pages[i].size` = number of 4 KiB pages spanned by `[args[i].ptr, args[i].ptr + len - 1]` multiplied by `4096`.
  2. **Inline Data Buffers (`0 <= i < nbufs`) without a DMA-BUF (`fd == 0 || fd == -1`):**
     - Placed in the Inline Arguments Area at `inline_va = args_cursor - olaps[oix].offset`.
     - `rpra[i].buf.pv = inline_va`, `rpra[i].buf.len = args[i].length`.
     - `pages[i].addr = (invocation_buf_consolidated_dma_addr - olaps[oix].offset + (pkt_size - rlen)) & ~0xfffULL`.
     - `pages[i].size` = number of 4 KiB pages spanned by `[inline_va, inline_va + len - 1]` multiplied by `4096`.
     - If `i < inbufs`, copy `len` bytes from userspace `args[i].ptr` into `inline_va` before sending the GLINK message.
     - After the DSP responds, for all output buffers (`inbufs <= i < nbufs`) without a DMA-BUF map, copy `rpra[i].buf.len` bytes from `rpra[i].buf.pv` back to userspace `args[i].ptr`.
  3. **DMA Handle Arguments (`nbufs <= i < nscalars`):**
     - Map the DMA-BUF (`args[i].fd`) into the session's SMMU Context Bank (`maps[i]`).
     - `list[i].num = args[i].length ? 1 : 0`, `list[i].pgidx = i`.
     - `pages[i].addr = maps[i].consolidated_dma_addr`, `pages[i].size = maps[i].size`.
     - `rpra[i].dma.fd = args[i].fd`, `rpra[i].dma.offset = (u32)args[i].ptr`, `rpra[i].dma.len = (u32)args[i].length`.

### 3.3 Memory Barriers and Poll Mode Completion (`FASTRPC_POLL_MODE`)
- **[HW]** Before transmitting the 40-byte GLINK message, the host **must** execute a DMA write memory barrier (`dma_wmb` / `dmb ishst`) so all inline argument writes and metadata tables are visible to the SMMU and DSP.
- **[HW]** If `poll_mode` is enabled on the user session (`FASTRPC_IOCTL_SET_OPTION` with `FASTRPC_POLL_MODE = 1`), `pd == USER_PD` (`1`), and `handle > 20` (`FASTRPC_MAX_STATIC_HANDLE`), the host polls the 32-bit word at `*poll_addr` (executing a DMA read barrier `dma_rmb` before each read) every 1 µs up to `10,000` µs (`FASTRPC_POLL_MAX_TIMEOUT_US = 10 ms`):
  - If `*poll_addr == 0x000decaf` (`FASTRPC_POLL_RESPONSE`), the call succeeded (`retval = 0`) without waiting for a GLINK interrupt.
  - If polling times out after 10 ms without seeing `0x000decaf`, the host falls back to waiting for the normal GLINK response interrupt.
- **[HW]** After completion (via poll or GLINK response), the host executes a DMA read memory barrier (`dmb ishld`), copies inline output buffers (`inbufs .. nbufs - 1`) to userspace, and iterates `fdlist[0..15]`: for each non-zero `fdlist[k]`, the host looks up the corresponding DMA-BUF mapping in the session's map list and decrements its reference count (unmapping it from the SMMU if the reference count reaches zero).

---

## 4. Internal Kernel RPC Calls (`handle = 1` and `handle = 2`)

### 4.1 Reserved Static Handles and Remote Method IDs (`FASTRPC_INIT_HANDLE = 1`)
- **[HW]** Handle `1` (`FASTRPC_INIT_HANDLE`) is reserved exclusively for kernel-to-DSP process lifecycle and memory management commands (`kernel = true`, meaning `msg.client_id = 0` in the 40-byte GLINK header). Any attempt by userspace (`FASTRPC_IOCTL_INVOKE`) to invoke `handle == 1` directly **must** be rejected with `EPERM`.

| Remote Method Name | Method ID | `sc` Value (`(method << 24) \| (in << 16) \| (out << 8)`) | Description | Tag |
|---|---|---|---|---|
| `FASTRPC_RMID_INIT_ATTACH` | `0` | `0x00010000` (`in = 1, out = 0`) | Attaches session to `ROOT_PD` (`0`) or `SENSORS_PD` (`2`). Arg 0 (4 B): `client_id` (`s32` at `+0x00`) | **[HW]** |
| `FASTRPC_RMID_INIT_RELEASE` | `1` | `0x01010000` (`in = 1, out = 0`) | Releases DSP process when `/dev/fastrpc-*` FD is closed. Arg 0 (4 B): `client_id` (`s32` at `+0x00`) | **[HW]** |
| `FASTRPC_RMID_INIT_MMAP` | `4` | `0x04020100` (`in = 2, out = 1`) | Maps pages into DSP (`FASTRPC_IOCTL_MMAP`). Arg 0 (24 B): `client_id` (`s32` at `+0x00`), `flags` (`u32` at `+0x04`), `vaddr` (`u64` at `+0x08`), `num` (`s32` at `+0x10`), `pad` (`u32` at `+0x14`); Arg 1 (16 B): `addr` (`u64` at `+0x00`), `size` (`u64` at `+0x08`); Out Arg 2 (8 B): `vaddr_out` (`u64` at `+0x00`) | **[HW]** |
| `FASTRPC_RMID_INIT_MUNMAP` | `5` | `0x05010000` (`in = 1, out = 0`) | Unmaps pages from DSP (`FASTRPC_IOCTL_MUNMAP`). Arg 0 (24 B): `client_id` (`s32` at `+0x00`), `pad` (`u32` at `+0x04`), `vaddr` (`u64` at `+0x08`), `size` (`u64` at `+0x10`) | **[HW]** |
| `FASTRPC_RMID_INIT_CREATE` | `6` | `0x06040000` (`in = 4, out = 0`) | Spawns a dynamic `USER_PD` (`pd = 1`) process when `attrs == 0` | **[HW]** |
| `FASTRPC_RMID_INIT_CREATE_ATTR` | `7` | `0x07040000` (`in = 4, out = 0`) | Spawns a dynamic `USER_PD` (`pd = 1`) process when `attrs != 0` (e.g., `FASTRPC_MODE_UNSIGNED_MODULE = 0x8`). Note: `sc` encodes `in = 4` (the first 4 buffers are sent; Arg 0 already contains `attrs` and `siglen`) | **[HW]** |
| `FASTRPC_RMID_INIT_CREATE_STATIC` | `8` | `0x08030000` (`in = 3, out = 0`) | Attaches to a static PD process (e.g., Audio PD). Arg 0 (12 B): `client_id` (`s32` at `+0x00`), `namelen` (`u32` at `+0x04`), `pageslen` (`u32` at `+0x08`); Arg 1: process name string; Arg 2 (16 B): `addr` (`u64` at `+0x00`), `size` (`u64` at `+0x08`) | **[HW]** |
| `FASTRPC_RMID_INIT_MEM_MAP` | `10` | `0x0a030100` (`in = 3, out = 1`) | Maps a DMA-BUF on DSP (`FASTRPC_IOCTL_MEM_MAP`). Arg 0 (32 B): `client_id` (`s32` at `+0x00`), `fd` (`s32` at `+0x04`), `offset` (`s32` at `+0x08`), `flags` (`u32` at `+0x0c`), `vaddrin` (`u64` at `+0x10`), `num` (`s32` at `+0x18`), `data_len` (`s32` at `+0x1c`); Arg 1 (16 B): `addr` (`u64` at `+0x00`), `size` (`u64` at `+0x08`); Arg 2 (0 B); Out Arg 3 (8 B): `vaddr_out` (`u64` at `+0x00`) | **[HW]** |
| `FASTRPC_RMID_INIT_MEM_UNMAP` | `11` | `0x0b010000` (`in = 1, out = 0`) | Unmaps a DMA-BUF from DSP (`FASTRPC_IOCTL_MEM_UNMAP`). Arg 0 (24 B): `client_id` (`s32` at `+0x00`), `fd` (`s32` at `+0x04`), `vaddrin` (`u64` at `+0x08`), `len` (`u64` at `+0x10`) | **[HW]** |

### 4.2 Dynamic User PD Process Creation (`FASTRPC_RMID_INIT_CREATE` / `CREATE_ATTR`)
- **[HW]** When userspace calls `FASTRPC_IOCTL_INIT_CREATE` to spawn a User PD on the CDSP:
  1. Validate `init.filelen <= 2,097,152` (2 MiB, `INIT_FILELEN_MAX`).
  2. Set `pd = USER_PD` (`1`).
  3. If `init.filelen > 0 && init.filefd != 0`, map `init.filefd` (`init.filelen` bytes) into the session's SMMU Context Bank.
  4. Allocate a coherent initial shell memory buffer (`init_mem`) from the session's SMMU Context Bank of size:
     `memlen = round_up(max(2 * 1024 * 1024, init.filelen * 4), 1024 * 1024)` (at least 2 MiB, 1 MiB-aligned).
  5. Construct the 4 input arguments (`sc = 0x06040000` if `init.attrs == 0`, or `0x07040000` if `init.attrs != 0`):
     - **Arg 0 (`24` bytes, `fd = -1`):** `client_id` (`s32` at `+0x00`), `namelen` (`u32` at `+0x04`), `filelen` (`u32` at `+0x08`), `pageslen = 1` (`u32` at `+0x0c`), `attrs` (`u32` at `+0x10`), and `siglen` (`u32` at `+0x14`), where `namelen = strlen(calling_process_name) + 1`.
     - **Arg 1 (`namelen` bytes, `fd = -1`):** Null-terminated calling process name string.
     - **Arg 2 (`filelen` bytes, `fd = init.filefd`):** User pointer `init.file` and DMA-BUF `init.filefd` containing the DSP shell ELF (e.g., `fastrpc_shell_unsigned_3` or `fastrpc_shell_3`).
     - **Arg 3 (`16` bytes, `fd = -1`):** Single `PhyPage` entry with `addr = init_mem.consolidated_dma_addr` (`u64` at `+0x00`) and `size = init_mem.size` (`u64` at `+0x08`).
  6. Invoke `handle = 1` (`FASTRPC_INIT_HANDLE`, `kernel = true`). Retain `init_mem` for the lifetime of the file descriptor.

### 4.3 Querying DSP Capabilities (`FASTRPC_DSP_UTILITIES_HANDLE = 2`)
- **[HW]** When userspace calls `FASTRPC_IOCTL_GET_DSP_INFO` (`attribute_id < 256`):
  - Attribute `0` (`dsp_attributes[0]`) is reserved for userspace (`0`).
  - For attributes `1..255`, the kernel queries the DSP once per channel (and caches the 256 `u32` array) by invoking **`handle = 2` (`FASTRPC_DSP_UTILITIES_HANDLE`)**, method `0`, `inbufs = 1`, `outbufs = 1` (`sc = 0x00010100`, `kernel = true`):
    - **Arg 0 (`4` bytes input, `fd = -1`):** `u32 len = 255` (`FASTRPC_MAX_DSP_ATTRIBUTES - 1`).
    - **Arg 1 (`255 * 4 = 1020` bytes output, `fd = -1`):** `&dsp_attributes[1]`.
  - If the DSP returns `0x80000414` (`DSP_UNSUPPORTED_API`), return `-EOPNOTSUPP`.
