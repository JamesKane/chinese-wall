# 03: Firmware Interface

This section covers the `.fwb` firmware binary, how it is placed in a
session's address space, the shared-memory queues, every message the host
and firmware exchange, the firmware RPC channel, and how sessions are
scheduled onto LSIDs and cores.

All multi-byte fields are little-endian. Structures use natural alignment
(each field aligned to its own size, and each structure padded to its
largest member) unless marked *packed*. Offsets are in bytes. Every layout
here is a **[HW]** firmware-ABI fact unless tagged otherwise.

> OPEN QUESTION: The vendor driver carries definitions for host-interface
> protocol **2.5** and **3.3**. Which major version the Sky1 `.fwb` files
> report has not been checked. Read it from the header (§3.1.1). The
> layouts below are the same for both, except where a difference is noted.

## 3.1 Firmware binary

### 3.1.1 Header (at file offset 0)

| Offset | Size | Field | Meaning |
|---|---|---|---|
| `0x00` | 4 | `rasc_jmp` | Firmware start address (not used by the host) |
| `0x04` | 1 | protocol minor | Host-interface protocol minor version |
| `0x05` | 1 | protocol major | Host-interface protocol major version (2 or 3) |
| `0x06` | 2 | reserved | 0 |
| `0x08` | 56 | info string | Human-readable codec information (NUL-padded) |
| `0x40` | 8 | part number | |
| `0x48` | 8 | SVN revision | |
| `0x50` | 16 | version string | Firmware version (NUL-padded) |
| `0x60` | 4 | `text_length` | Bytes of the image, counted from file offset 0, that form the read-only text |
| `0x64` | 4 | `bss_start_address` | Instance-relative VA of BSS page 0. Page-aligned |
| `0x68` | 4 | `bss_bitmap_size` | Number of valid bits in the BSS bitmap (≤ 512) |
| `0x6C` | 64 | BSS bitmap | 16 × u32. Bit *i* set = BSS page *i* is needed |
| `0xAC` | 4 | `master_rw_start_address` | Instance-relative VA of the shared-BSS window |
| `0xB0` | 4 | `master_rw_size` | Size of the shared-BSS window, in bytes |

Header size: `0xB4` bytes.

- **[HW]** Validation: the file must be at least `0xB4` bytes, and
  `text_length` must not exceed the file size.
- **[HW]** Page counts: text = ceil(`text_length` / 4096). Walk the bitmap
  bits 0 … `bss_bitmap_size`−1, with page *i* at VA
  `bss_start_address + 4096·i`. A page inside the master-RW window is a
  **shared** page, whether or not its bit is set. Otherwise, a page whose
  bit is set is a **private** page. Each core needs its own set of private
  pages. One set of shared pages serves all cores.
- **[HW]** **Checksum variant.** If the version string contains the
  substring `-sum`, the firmware expects checksummed host-to-firmware
  messages (§3.3.4).

### 3.1.2 Protocol version effects

| Version | Effect |
|---|---|
| major 2 | Memory map 02 §2.3.2 |
| major 3 | Memory map 02 §2.3.1. `RELEASE_REF_FRAME` is decode-only. Profile `H264_HIGH_10` = 4 is available |
| ≥ 2.4 and ≥ 3.1 | The host must answer each `IDLE` with `IDLE_ACK` (§3.7.3) |
| other major | Not supported |

## 3.2 Per-session firmware setup

Do these steps for each new session, after the binary is loaded:

1. Allocate the session's zeroed level-1 page table (02 §2.2).
2. Allocate the text pages and copy `text_length` bytes into them,
   starting at file offset 0. Allocate zeroed private BSS pages (count ×
   number of mapped instances) and zeroed shared BSS pages.
3. For each instance *n* in 0…*h*, where *h* is the highest usable core
   (instance 0 always included), map text, private BSS and shared BSS as in
   02 §2.3.3.
4. Map the seven communication pages and the print buffer (02 §2.4).
5. Clean caches for everything written (02 §2.6).
6. Queue the initial messages (§3.8.1). Then request scheduling (§3.7).
   The firmware starts when its LSID is first scheduled on a core.

- **[HW]** Firmware memory is per session. Two sessions never share text
  pages in the vendor driver, though they could, since text is read-only.

## 3.3 Shared-memory queues

### 3.3.1 Channels

Three channels exist. Each has a pair of pages:

| Channel | Host-written page | Firmware-written page |
|---|---|---|
| Messages | `0x1007_9000` (host → fw messages) | `0x1007_A000` (fw → host messages) |
| Input buffers | `0x1007_B000` (host queues input buffers) | `0x1007_C000` (fw returns input buffers) |
| Output buffers | `0x1007_D000` (host queues output buffers) | `0x1007_E000` (fw returns output buffers) |

### 3.3.2 Page layout

Host-written page:

| Offset | Size | Field |
|---|---|---|
| `0x00` | 2 | `out_rpos`: host's read index into the **firmware** page's ring |
| `0x02` | 2 | `in_wpos`: host's write index into **this** page's ring |
| `0x04` | 12 | reserved[3] (checksum words, §3.3.4) |
| `0x10` | 4080 | ring: 1020 × u32 |

Firmware-written page:

| Offset | Size | Field |
|---|---|---|
| `0x00` | 2 | `out_wpos`: firmware's write index into **this** page's ring |
| `0x02` | 2 | `in_rpos`: firmware's read index into the **host** page's ring |
| `0x04` | 12 | reserved[3] (checksum words) |
| `0x10` | 4080 | ring: 1020 × u32 |

- **[HW]** Indices count **32-bit words**, modulo **1020**.
- **[HW]** Each side writes only its own page. The host never writes the
  firmware page, and the reverse also holds.

### 3.3.3 Message framing and procedure

- **[HW]** A message is a 4-byte header, `code` (u16, offset 0) and `size`
  (u16, offset 2, the payload length in bytes, 0 if none), followed by the
  payload, padded to a multiple of 4 bytes. It occupies
  ceil((4 + size) / 4) ring words and may wrap around the ring end.
- **[HW]** Send (host → firmware):
  1. Invalidate and read the peer's `in_rpos`.
  2. free = `in_rpos − in_wpos`. If free ≤ 0, add 1020.
  3. If free < words needed, the queue is full. Retry later.
  4. Write header and payload at `in_wpos` onwards, wrapping at 1020.
  5. Barrier, clean the page, then store the new `in_wpos`, barrier, clean.
  6. Ring the doorbell: write the LSID's `IRQHOST` = 1. If the session has
     no LSID, remember the request and set `IRQHOST` on the next attach.
- **[HW]** Receive (firmware → host):
  1. Invalidate the firmware page.
  2. available = `out_wpos − out_rpos` (mod 1020). If 0, the queue is empty.
  3. Read the header. Codes outside 2001…3004 are invalid. Skip them.
  4. If the message is longer than the available words, it is inconsistent.
     Drop it.
  5. Read the payload. Store `out_rpos`, then barrier, then clean.
- **[HW]** Readers must use the header `size`, not a fixed structure size.
  Messages may grow in later firmware.
- **[POLICY]** When the ring is completely empty the sender's computation
  gives 1020 free words. A clean-room driver should keep at least one word
  free so that "full" cannot look like "empty".

### 3.3.4 Checksum mode (firmware version string contains `-sum`)

- **[HW]** For each host-written channel, the host keeps a running u32 sum.
  For each message, add every u32 word of the header and of the payload
  (rounded up to whole words) to the running sum. Write the header, then
  **one extra word holding the running sum**, then the payload. The header
  `size` does **not** include the extra word.
- **[HW]** The host also shifts its page's reserved words: `reserved[0] ←
  reserved[1]`, `reserved[1] ← reserved[2]`, `reserved[2] ← running sum`.
  Non-zero reserved words signal that checksums are in use.
- **[HW]** If any reserved word of the **firmware** message page is
  non-zero, the firmware is sending checksums. The host keeps its own
  running sum of all words it has consumed. Before reading, it adds all
  currently available words to that sum. The result must equal one of the
  firmware page's three reserved words, or the batch is corrupt.

> OPEN QUESTION: The firmware-side checksum rule is reconstructed from host
> code only. In particular, whether the extra sum word applies to the
> buffer channels in the firmware-to-host direction is unknown.

## 3.4 Interrupts and doorbells

- **[HW]** The host rings the firmware with `IRQHOST` = 1 on the session's
  LSID. The firmware interrupts the host by setting its LSID's `LIRQVE`,
  which shows in global `IRQVE`.
- **[HW]** On each interrupt for a session, the host must first serve a
  pending RPC (§3.6), then read **all** messages from the message channel.
  A message whose code is `INPUT` or `OUTPUT` means that one buffer
  descriptor is waiting in the matching return ring. The host reads exactly
  one descriptor from it. Then the host drains the print buffer (§3.6.2).
- **[POLICY]** The host may raise `LIRQVE` itself (write 1) to force a
  re-scan. The vendor driver does this after timeouts.

## 3.5 Messages

### 3.5.1 Codes

Requests (host → firmware, message channel):

| Code | Name | Payload | Expected reply |
|---|---|---|---|
| 1001 | `GO` | none | `STATE_CHANGE` (running) |
| 1002 | `STOP` | none | `STATE_CHANGE` (stopped) |
| 1003 | `INPUT_FLUSH` | none | all input buffers returned, then `INPUT_FLUSHED` |
| 1004 | `OUTPUT_FLUSH` | none | all output buffers returned, then `OUTPUT_FLUSHED` |
| 1005 | `SWITCH` | none | `SWITCHED_OUT` |
| 1006 | `PING` | none | `PONG` |
| 1008 | `DUMP` | none | `DUMP` |
| 1009 | `JOB` | §3.5.2 | (none, later `JOB_DEQUEUED`) |
| 1010 | `SET_OPTION` | §3.5.3 | `SET_OPTION_CONFIRM` or `SET_OPTION_FAIL` |
| 1011 | `RELEASE_REF_FRAME` | u32 frame-buffer VA | later `REF_FRAME_UNUSED` |
| 1012 | `IDLE_ACK` | none | none |
| 1013 | `DEBUG` | u32 log level (0 off, 1 error … 5 verbose) | `DEBUG` |

Responses (firmware → host, message channel):

| Code | Name | Payload |
|---|---|---|
| 2001 | `SWITCHED_IN` | u32 core number. The firmware has started on a core |
| 2002 | `SWITCHED_OUT` | u32 core, u32 reason, u32 sub-reason. The last core has left the session |
| 2003 | `SET_OPTION_CONFIRM` | (u32 option index) |
| 2004 | `JOB_DEQUEUED` | u32 valid-job flag. Every core has finished the current job |
| 2005 | `INPUT` | none. One descriptor is in the input-return ring |
| 2006 | `OUTPUT` | none. One descriptor is in the output-return ring |
| 2007 | `INPUT_FLUSHED` | none |
| 2008 | `OUTPUT_FLUSHED` | none |
| 2009 | `PONG` | none |
| 2010 | `ERROR` | u32 code, char[128] text. **Fatal**: the session cannot continue |
| 2011 | `STATE_CHANGE` | u32 new state: 0 = stopped, 2 = running |
| 2012 | `DUMP` | none |
| 2013 | `IDLE` | none. The firmware has nothing to do |
| 2014 | `FRAME_ALLOC_PARAM` | §3.5.5 (decode) |
| 2015 | `SEQUENCE_PARAMETERS` | §3.5.6 (decode) |
| 2016 | `EVENT` | §3.5.7 |
| 2017 | `SET_OPTION_FAIL` | u32 option index, char[128] text. Not fatal |
| 2018 | `REF_FRAME_UNUSED` | u32 frame-buffer VA |
| 2019 | `DEBUG` | none |
| 2050 | `TRACE` | u32 sum, u32 count (debug) |

Buffers (host → firmware on the buffer rings; returned on the return rings):

| Code | Name | Descriptor |
|---|---|---|
| 3001 | `FRAME` | §3.5.8 |
| 3002 | `BITSTREAM` | §3.5.9 |
| 3003 | `PARAM` | §3.5.10. Sent on the **input-buffer** ring. The firmware may also send it on the message channel (decoded metadata) |
| 3004 | `GENERAL` | §3.5.11 |

Error codes in `ERROR`: 1 abort, 2 out of memory, 3 assert, 4 unsupported,
6 invalid buffer, 8 invalid state, 9 watchdog.

- **[POLICY]** The vendor driver treats an `EVENT` with code 2 (stream not
  supported) as a fatal error. It also treats a `FRAME_ALLOC_PARAM` with a
  planar width or height below 144 as invalid stream parameters.
- **[POLICY]** The vendor driver counts outstanding requests that expect a
  reply (`GO`, `STOP`, the flushes, `SET_OPTION`, `DUMP`, `DEBUG`). It waits
  until the count is zero before changing state (timeout 3 s, then a
  software interrupt and a further 3 s, then error).

### 3.5.2 `JOB` payload (8 bytes)

| Offset | Size | Field |
|---|---|---|
| 0 | 2 | Number of cores, ≥ 1. Must equal the core count in the LSID's `JOBQUEUE` entry |
| 2 | 2 | Frames to process before the job ends. 0 = no limit |
| 4 | 4 | Flags. Bit 0 = disable the bound manager. Normally 0 |

- **[POLICY]** The vendor driver sends frames = 1 by default (the
  `JOB_FRAMES` control). With one open session it uses 0 (no limit); with
  two or more it uses 1, so that sessions take turns frame by frame.

### 3.5.3 `SET_OPTION` payload

| Offset | Size | Field |
|---|---|---|
| 0 | 4 | Option index |
| 4 | var | Argument. Most options take one u32. Send exactly 4 + argument size bytes |

Options most relevant to decode and to H.264/HEVC encode:

| Index | Option | Argument |
|---|---|---|
| 1 | NAL unit format | u32: 1 start codes, 2 one NALU per buffer, 4 / 8 / 16 one-, two-, four-byte length prefix |
| 2 | Emulation-prevention escaping | u32 0/1 (default 1) |
| 3 | Profile and level | u16 profile, u16 level (§3.5.4) |
| 6 | Disable features | u32 bitmask (below) |
| 7 | Ignore stream headers (decode) | u32 0/1 |
| 8 | Frame reordering (decode) | u32 0/1 (default 1) |
| 9 | Intermediate-buffer size limit (decode) | u32 bytes |
| 16 | P frames per GOP (encode) | u32 |
| 17 | B frames (encode) | u32 |
| 18 | GOP type | u32: 1 bidirectional, 2 low delay, 3 pyramid, 4 SVCT3, 5 GDR |
| 19 | Cyclic intra refresh MBs | u32 |
| 20 / 21 / 22 | Constrained intra pred / entropy sync / temporal MVP | u32 0/1 |
| 23 | Tiles | u16 rows, u16 cols (HEVC: cols must be 0; VP9: powers of two) |
| 24 | HEVC min luma CB size | u32 8 or 16 |
| 25 / 26 | Enable / disable MB types | u32 bitmask |
| 27 | H.264 CABAC | u32 0/1 (default 1) |
| 28 | Slice spacing | u32 MBs/CTUs per slice |
| 30 | VP9 probability update | u32 0 off, 1 implicit, 2 explicit |
| 31 / 32 / 33 | JPEG restart interval / Huffman table / quant table | see note |
| 36 | MV search range | s32 x, s32 y |
| 38 | Stream bit depth | u16 luma, u16 chroma |
| 39 | Stream chroma format | u32 |
| 40 | RGB→YUV mode | u32: 0 BT.601 studio, 1 BT.601 full, 2 BT.709 studio, 3 BT.709 full, 4 BT.2020 studio, 5 BT.2020 full |
| 41 | Bandwidth limit | u32 |
| 42 | Firmware watchdog timeout | u32. 0 = disabled |
| 43 | CABAC init IDC | u32 (H.264 0–2, HEVC 0–1) |
| 44 | Adaptive quantisation | u32 |
| 45 / 46 / 47 | QP delta I–P / I–B(ref) / I–B(non-ref) | u32 |
| 48 / 49 | Cb / Cr QP offset | u32 |
| 50 | Lambda scale | 8 × u16 Q8 (I, √I, P, √P, Bref, √Bref, Bnonref, √Bnonref) |
| 51 | Max encode cores | u32 |
| 56 | Fixed QP | u32 |
| 57 | Decode downscale | u16 width, u16 height |
| 60 / 61 | SAO luma / chroma enable | u32 |
| 62–65 | Encode crop left / right / top / bottom | u32 |
| 66 / 67 | LTR mode / period | u32 |
| 69 | Decode downscale position mode | u32 0–2 |
| 70 | Max mini-frame count | u32 |
| 71 | Encode statistics mode | Defined, but the vendor host sends statistics mode as `PARAM` 29 instead |
| 73 | Multiple SPS/PPS | u32 |
| 74 / 75 / 76 | Initial QP I / P / B | u32 |
| 77–79, 2011–2013 | Intra / inter penalties (angular, planar, DC) | u32 |
| 80–83 | RC / QP-map clip top / bottom | u32 |
| 84 | Reference ring buffer | u32 |
| 85 | JPEG rate control | u32 qscale, qscale luma, qscale chroma, fps |
| 86 / 87 | RC I-frame bit ratio / mode | u32 |
| 88 | SVCT3 level-1 period | u32 |
| 89 | Visual enable | u32 |
| 90 / 91 | GDR number / period | u32 |
| 93–95, 101 | Scene-change detection enable / percent / threshold / adaptive I | u32 |
| 96–100 | Adaptive-quant SSIM enable, neg/pos ratio, QP-delta limit, initial frame average variance | u32 |
| 103 | Decode YUV→RGB parameters | s16 coef[3][3], u16 offset[3] |
| 104 | Forced UV value | u16 |
| 105 | Downscale interpolation mode | u16 |
| 106 | Encode source crop | u8 enable, then u16 x, y, w, h (natural alignment: x at +2 of the argument) |
| 107 | Decode destination crop | same layout as 106 |
| 1004 | AV1 fast show frame | u32 |

Disable-features bitmask (option 6): `0x1` AFBC compression, `0x2`
reference cache, `0x4` deblocking, `0x8` SAO, `0x20` picture output, `0x40`
pipe (parser only), `0x80` clock gating (sleep), `0x100` legacy tiled-AFBC
reference, `0x400` reference pictures (static 16×16), `0x1000` reference
size bandwidth limit, `0x10000` AV1 non-IBC tile support.

- **[POLICY]** For AV1 decode the vendor driver always sets bit `0x100`.
- Note: the JPEG table options carry a u32 type (Huffman: 1 DC luma, 2 AC
  luma, 3 DC chroma, 4 AC chroma; quant: 1 luma, 2 chroma) followed by
  16 code-length bytes and 162 value bytes (Huffman), or 64 matrix bytes
  (quant).

> OPEN QUESTION: Option 4 (protocol prints), 5 (profiling), 34 (expose
> reference frames) and 35 (MB info output) are debug options. Their
> payloads beyond a u32 0/1 are not used by the host. In protocol 2,
> indices 52–55 and 58 have different meanings (extra references, raw QP
> deltas, flushless reference bank). Use the protocol-3 table only when
> major = 3.

### 3.5.4 Profile and level values

| Codec | Profile values |
|---|---|
| H.264 | 1 baseline, 2 main, 3 high, 4 high-10 (protocol 3) |
| HEVC | 1 main, 2 main still, 3 main intra, 4 main 10 |
| VC-1 | 1 simple, 2 main, 3 advanced |
| VP8 | 1 main |

- **[HW]** H.264 levels: 1 = 1, 2 = 1b, 3 = 1.1, 4 = 1.2, 5 = 1.3, 6 = 2,
  7 = 2.1, 8 = 2.2, 9 = 3, 10 = 3.1, 11 = 3.2, 12 = 4, 13 = 4.1, 14 = 4.2,
  15 = 5, 16 = 5.1, 17 = 5.2, 18 = 6, 19 = 6.1, 20 = 6.2. The value
  32 + *x* (*x* = 0…255) writes `level_idc` = *x* unchecked.
- **[HW]** HEVC levels: 2·*k*−1 = main tier and 2·*k* = high tier of the
  *k*-th level in the order 1, 2, 2.1, 3, 3.1, 4, 4.1, 5, 5.1, 5.2, 6, 6.1,
  6.2 (so 1 = main 1, 2 = high 1, … 25 = main 6.2, 26 = high 6.2).

### 3.5.5 `FRAME_ALLOC_PARAM` payload (decode)

| Offset | Size | Field |
|---|---|---|
| 0 | 2 | Planar allocation width |
| 2 | 2 | Planar allocation height |
| 4 | 4 | AFBC bytes per frame (per field if interlaced) |
| 8 | 4 | AFBC bytes for the downscaled frame |
| 12 | 2 | Recommended AFBC width in superblocks |
| 14 | 2 | Same, downscaled |
| 16 | 2 | Crop x (plane-address adjustment hint) |
| 18 | 2 | Crop y |
| 20 | 4 | MB-info bytes (debug) |
| 24 | 2 | Downscaled frame width |
| 26 | 2 | Downscaled frame height |
| 28 | 2 | Downscale position mode |
| 30 | 1 | CTU size |
| 32 | 2 × 4 | Destination crop x, y, width, height |

- **[HW]** These values are allocation **hints**, not the displayed
  resolution. Output buffers that are too small come back flagged
  `REJECTED`.

### 3.5.6 `SEQUENCE_PARAMETERS` payload (decode, 8 bytes)

| Offset | Field |
|---|---|
| 0 | Interlaced (0/1) |
| 1 | Chroma format: 0 mono, 1 4:2:0, 2 4:2:2, 3 4:4:0, 4 ARGB, 5 RGB, 6 4:4:4, 7 OSD-ARGB |
| 2 | Luma bit depth (8, 9, 10) |
| 3 | Chroma bit depth |
| 4 | Planar output buffers needed |
| 5 | AFBC output buffers needed |
| 6 | Range mapping enabled (VC-1) |
| 7 | reserved |

- **[HW]** After sending this message, the decoder **stops** until the host
  sends `OUTPUT_FLUSH`. The firmware then returns every queued output
  buffer and sends `OUTPUT_FLUSHED`. Only after that may the host queue
  output buffers again, sized for the new parameters.

### 3.5.7 `EVENT` payload (*packed*)

u32 event code, then the event data: 1 = stream corrupt (char[128]), 2 =
stream not supported (char[128]), 3 = frame processed (timing statistics,
debug), 4 = reference frame (debug), 5 = trace buffers (debug). Events are
informational.

### 3.5.8 `FRAME` descriptor (72 bytes)

| Offset | Size | Field |
|---|---|---|
| 0 | 8 | Host handle. Opaque, returned unchanged |
| 8 | 8 | User data tag. Copied between matching input and output buffers (used for timestamps) |
| 16 | 4 | Frame flags (below) |
| 20 | 2 | Visible height (luma samples) |
| 22 | 2 | Visible width |
| 24 | 2 | Format code (§3.5.8.1) |
| 26 | 2 | reserved |
| 28 | 40 | Planar or AFBC part (below) |
| 68 | 2 | Mini-frame first row |
| 70 | 2 | Mini-frame last row |

Planar part (at 28): u32 top-field plane VA [3] (Y, Cb, Cr) at 28, u32
bottom-field plane VA [3] at 40, s32 stride in bytes [3] at 52, u16 maximum
frame width at 64, u16 maximum frame height at 66.

AFBC part (at 28): u32 plane VA [2] at 28, u32 allocation bytes [2] at 36,
u16 crop x at 44, u16 crop y at 46, u16 width in superblocks [2] at 48, u32
AFBC parameters at 52.

AFBC parameter bits: `0x1` tiled body, `0x2` tiled headers, `0x4` 32×8
superblock (encode input only, default 16×16), `0x8` downscaled plane forced
to 8-bit, `0x10` downscaled plane forced to 4:2:0, `0x20` the firmware
chooses the superblock stride (decode), `0x40` block split, `0x80` AV1
decoder output, `0x100` AV1 tiled header, `0x200` AV1 monochrome.

Frame flags:

| Bit(s) | Name | Set by | Meaning |
|---|---|---|---|
| `0x1` | INTERLACE | host (dec) | Buffer holds both fields |
| `0x2` | BOT_FIRST | fw | Bottom field first |
| `0x4` | TOP_PRESENT | fw (dec), host (enc) | Top field or whole frame present |
| `0x8` | BOT_PRESENT | fw | Bottom field present |
| `0x30` | ROTATION | host | 1 = 90°, 2 = 180°, 3 = 270° (bits 4–5) |
| `0xC0` | SCALING | host (dec) | Downscale (1 = ½, 2 = ¼). With dual AFBC, `0x40` requests the half-size second plane |
| `0x100` / `0x200` | MIRROR_H / MIRROR_V | host (enc) | Mirror before encoding |
| `0x400` | FORCE_IDR | host (enc) | |
| `0x800` | RESET_GOP | host (enc) | |
| `0x1000` | REJECTED | fw | Too small. Reallocate |
| `0x2000` | CORRUPT | fw | Contains visible corruption |
| `0x4000` | DECODE_ONLY | fw | Do not display |
| `0x8000` | REF_FRAME | fw | Still used as a reference. Do not modify. Re-queue after display |
| `0x10000` | EOS | both | Last frame |
| `0x20000` | RESET_LTR_PERIOD | host (enc) | |
| `0x40000` | RESET_RC | host (enc) | |
| `[31:24]`, `[23:17]` | arbitrary downscale X / Y | host (dec) | Ratio − 1 in each direction |

- **[HW]** Planar decode output: the host gives each plane's VA and stride.
  For interlaced output, the host gives a stride of 2 × the line pitch
  (pitch rounded up to even), and puts the bottom field at plane VA + one
  line pitch. The maximum frame width/height tell
  the firmware the allocated capacity: max width = stride × 2 / (bytes per
  2 pixels of plane 0), and max height = plane-0 length / stride.
- **[HW]** AFBC interlaced: plane 1 = plane 0 + align32(total/2), with
  allocation bytes split the same way.
- **[HW]** A returned output frame with neither PRESENT flag, or with
  width or height 0, carries no picture.

#### 3.5.8.1 Format codes

Bit fields: `[2:0]` chroma format (as §3.5.6), `[7:4]` maximum bit depth − 8,
`[9:8]` number of planes, `[13:12]` variant, `[15]` AFBC.

| Code | Format | Code | Format |
|---|---|---|---|
| `0x8001` | YUV420 AFBC 8-bit | `0x0102` | YUY2 (4:2:2 packed) |
| `0x8021` | YUV420 AFBC 10-bit | `0x1102` | UYVY |
| `0x8002` | YUV422 AFBC 8-bit | `0x0182` | Y210 |
| `0x8022` | YUV422 AFBC 10-bit | `0x0122` | 4:2:2 1-plane 10-bit (16-bit LSB) |
| `0x0301` | I420 (3-plane 4:2:0) | `0x0104` / `0x1104` / `0x2104` / `0x3104` | RGBA / BGRA / ARGB / ABGR 8888 |
| `0x0321` | I420 10-bit (16-bit LSB) | `0x0105` / `0x1105` | RGB888 / BGR888 |
| `0x0201` | NV12 | `0x0305` | RGB 3-plane |
| `0x1201` | NV21 | `0x0100` / `0x0120` | Y8 / Y10 |
| `0x0281` | P010 | `0x0306` / `0x0326` | YUV444 3-plane 8 / 10-bit |
| `0x0221` | 4:2:0 2-plane 10-bit (16-bit LSB) | `0x0107` / `0x1107` / `0x2107` | ARGB1555 / ARGB4444 / RGB565 (OSD) |
| `0x0121` | Y0L2 (Arm packed 10-bit) | `0x0001` | MB info (debug) |
| `0x1121` | AQB1 (Arm packed 10-bit) | `0x0000` | unused |

- **[POLICY]** Monochrome AFBC (8/10-bit) is sent as the 4:2:0 AFBC codes.

### 3.5.9 `BITSTREAM` descriptor (40 bytes)

| Offset | Size | Field |
|---|---|---|
| 0 | 8 | Host handle |
| 8 | 8 | User data tag |
| 16 | 4 | Flags: `0x1` EOS, `0x10` end of frame, `0x20` sync frame (keyframe), `0x80` codec config, `0x400` end of sub-frame, `0x10000` encoder statistics, `0x100000` bitstream end-of-frame |
| 20 | 4 | Allocated bytes |
| 24 | 4 | Offset of first valid byte |
| 28 | 4 | Filled length |
| 32 | 4 | Buffer VA |
| 36 | 1 | Frame type (encode output): 0 I, 1 P, 2 B, 3 non-ref b, 4 P-key, 5 non-ref P (SVCT3), 6 GDR |
| 37 | 1 | Source transform: 0 none, 1/2/3 rotate 90/180/270, 4 vflip, 5–7 rotate + vflip |
| 38 | 2 | Bitstream KiB still held by the firmware |

- **[HW]** Decode input: fill the VA, allocated bytes and filled length.
  The offset is 0. A buffer that ends a frame should carry flags `0x10` and
  `0x100000`. Codec-config data carries `0x80` and `0x400`.
- **[HW]** Encode output: the firmware fills the offset, the length, the
  flags (sync frame on keyframes) and the frame type.
- **[POLICY]** If no NAL format is configured, the vendor driver marks every
  decode input buffer as end of frame.

### 3.5.10 `PARAM` descriptor

u32 type, then type-specific data. Send 4 + data size bytes. Main types:

| Type | Name | Data |
|---|---|---|
| 2 | QP | s32 qp, s32 EPR I-frame enable |
| 3 | Regions (ROI) | u8 count, 3 reserved, then 16 × {u16 mbx_left, mbx_right, mby_top, mby_bottom; s16 qp_delta; u8 priority; u8 force_intra} |
| 5 | Display size (dec → host) | u16 width, u16 height |
| 6 | Range map (VC-1) | 4 × u8 |
| 9 | Frame rate | u32, Q16.16 frames/s (30 fps = `0x001E0000`) |
| 10 | Rate control | u32 mode (0 off, 1 standard, 2 variable, 3 constant, 4 constrained variable), u32 target bit/s, u32 maximum bit/s |
| 12 / 13 / 14 | QP for I / P / B (no RC) | u32 |
| 15 | Colour description | §3.5.10.1 |
| 16 | Frame packing | u32 flags, 6 × u8, 2 reserved |
| 17 | Frame/field info | u8 pic_struct, source scan, duplicate, reserved |
| 18 | GOP reset | none |
| 19 | DPB held frames (dec → host) | u32 |
| 20 | Change rectangles | u8 count, 3 reserved, 2 × {u16 x_left, x_right, y_top, y_bottom} |
| 21 / 25 | RC QP range / I-frame QP range | s32 min, s32 max |
| 23 | HRD buffer size | u32 |
| 26 | SEI user data unregistered | u8 flags, u8 uuid[16], 221 bytes data, u8 length, 5 reserved |
| 27 / 28 | Dynamic GOP reset / LTR period reset | u32 |
| 29 | Encoder statistics | u32 sizes ×3, u32 flags, u16 MB width, u16 MB height |
| 30 | OSD rectangles | 2 × 16-byte OSD configuration |

#### 3.5.10.1 Colour description (offsets within the data)

| Offset | Field |
|---|---|
| 0 | u32 flags: bit 0 mastering display valid, bit 1 content light valid |
| 4 | u8 range: 0 unspecified, 1 limited, 2 full |
| 5 / 6 / 7 | u8 colour primaries / transfer characteristics / matrix coefficients (H.265 Annex E codes) |
| 8 | u16 display primaries x[3] (G, B, R order as in the SEI) |
| 14 | u16 display primaries y[3] |
| 20 / 22 | u16 white point x / y |
| 24 / 28 | u32 max / min mastering luminance |
| 32 / 36 | u32 max content light level / max frame-average light level |
| 40–44 | u8 video-format present, video format, aspect-ratio present, aspect IDC, timing-info present |
| 46 / 48 | u16 SAR width / height |
| 52 / 56 | u32 num_units_in_tick / time_scale |

> OPEN QUESTION: The vendor driver copies primaries index 0/1/2 to R/G/B.
> The colour order the firmware uses is not confirmed.

### 3.5.11 `GENERAL` descriptor

Header (32 bytes): u64 host handle at 0, u64 user data tag at 8, u32 data
buffer VA at 16, u32 data buffer size at 20, u16 type at 24 (1 block
configs, 2 assertive-display statistics, 3 encoder statistics), u16 config
size at 26, u32 reserved at 28. The type-specific config follows at 32. For
block configs this is u8 type (`0xFF` = uncompressed rows), 3 reserved, u8
columns − 1, u8 rows − 1, 2 reserved. The data buffer then holds one 8-byte
record per 32×32 quad: a u32 of four 6-bit QP deltas plus force bits, and a
u32 of four 6-bit minimum QPs.

## 3.6 Firmware RPC and print buffer

### 3.6.1 RPC area (`0x1007_F000`)

| Offset | Size | Field |
|---|---|---|
| 0 | 4 | State: 0 free, 1 request pending (firmware → host), 2 reply ready (host → firmware) |
| 4 | 4 | Call: 1 debug print, 2 memory alloc, 3 memory resize, 4 memory free |
| 8 | 4 | Size of the parameter data in bytes |
| 12 | 1012 | Parameters (253 × u32) |

| Call | Request parameters | Reply |
|---|---|---|
| 1 print | NUL-terminated text | size 0 |
| 2 alloc | u32 size at 12, u32 maximum size at 16, u8 region at 20 (0 protected/bitstream, 1 frame-buffer), u8 log2 alignment at 21 | size 4, u32 VA at 12 (0 = failure) |
| 3 resize | u32 VA at 12, u32 new size at 16 | size 4, u32 VA at 12 (0 = failure) |
| 4 free | u32 VA at 12 | size 0 |

- **[HW]** Procedure: when interrupted, invalidate the area. If state = 1,
  perform the call, write the reply fields, barrier, set state = 2, clean,
  then ring the doorbell.
- **[HW]** Alignment below 4 KiB is treated as 4 KiB. The allocation is
  mapped ATTR 3, AP 3 (02 §2.5). Resize keeps the VA. After a resize or
  free, flush the MMU.

### 3.6.2 Print buffer (`0x1010_0000`, 512 KiB)

Header (128 bytes): u32 read count at 0 (host-written), u32 magic at 64
(`0x11223356` when valid), u32 index at 68, u32 write count at 72. Entries
of 128 bytes follow from offset 128. The number of entries is 4095. Entry
*k* is at 128 + 128 × (*k* mod 4095). The host consumes entries from read
count to write count, then stores the new read count and cleans it.

- **[POLICY]** Reading the print buffer is optional (debug only).

## 3.7 Session scheduling across LSIDs

### 3.7.1 Model

- **[HW]** A session can make progress only while it is attached to an LSID
  that is in `JOBQUEUE` or running on cores. The hardware picks the head of
  `JOBQUEUE`, starts the session's firmware on the requested number of
  cores (`SWITCHED_IN`), and keeps it there until the firmware switches out.
- **[HW]** Firmware leaves the cores (`SWITCHED_OUT`) when the host sends
  `SWITCH`, or when the current `JOB` frame budget is used up
  (`JOB_DEQUEUED`). The session's queues and memory stay valid. Only the
  core assignment is released.

### 3.7.2 Switching in (vendor policy, satisfies the hardware rules)

- **[POLICY]** Each message the host sends is followed by a doorbell and a
  "switch in" request. The session is put on a pending list, ordered by
  priority (0–3).
- **[POLICY]** The scheduler worker takes each pending session in turn:
  - if it already has an LSID, re-add the LSID to `JOBQUEUE`;
  - otherwise use a free LSID or, failing that, an **idle** LSID (01 §1.3).
    Detach the idle LSID's session (and later re-check its queues), attach
    this session (01 §1.4.1) and add it to `JOBQUEUE`;
  - stop when no LSID is available.
- **[POLICY]** After every interrupt, the scheduler worker runs again.

### 3.7.3 Idle handshake and switching out

- **[HW]** When the firmware has nothing to do it sends `IDLE`. With
  protocol ≥ 2.4 / ≥ 3.1 the host must reply `IDLE_ACK` (and ring the
  doorbell).
- **[POLICY]** On the **second** consecutive `IDLE` (any other message
  resets the count) the host also sends `SWITCH`. On `SWITCHED_OUT` the
  session is marked not resident. If it still has work (state running and
  fewer than 2 idles, or replies outstanding), it is switched in again at
  once.
- **[HW]** On `JOB_DEQUEUED` the host must send a new `JOB` before the
  firmware will run again.

### 3.7.4 Timeouts and errors

- **[POLICY]** Session watchdog: started on switch-in with 100 ms, and
  re-armed on every interrupt. On expiry, if the session is still resident,
  the timeout is re-armed with 100 ms × count. After 15 expiries (about
  12 s in total) the host sends `DUMP`, terminates the LSID, marks the
  session switched out and reports an error to userspace.
- **[HW]** After `ERROR`, the session is dead. Terminate its LSID
  (01 §1.4.3) so that other sessions can run.
- **[POLICY]** The firmware's own watchdog (option 42) is sent with 0
  (disabled) by default.

## 3.8 Decoder flow

### 3.8.1 Start

Queue these messages before the first switch-in, in this order:

1. `DEBUG` (vendor uses level 5).
2. `SET_OPTION` 42 (watchdog timeout, 0).
3. `JOB` (usable-core count, frames per job).
4. Decoder options as configured: profile/level (VC-1 only), AV1 fast show
   frame, ignore stream headers, frame reordering, intermediate-buffer size,
   downscale size/position/interpolation, YUV→RGB coefficients (RGB output
   only), disable features (always for AV1), destination crop. Then the
   common options: NAL format (not for AV1) and stream escaping.
5. `GO`.
6. `PING`.

Then queue every pending input (bitstream) and output (frame) buffer on
its ring.

### 3.8.2 Steady state

1. The firmware consumes bitstream buffers and returns them (`INPUT` +
   descriptor) when done.
2. On the first sequence header (and on every change), the firmware sends
   `FRAME_ALLOC_PARAM`, then `SEQUENCE_PARAMETERS`, then waits.
3. The host compares the new values (size, AFBC bytes, buffer count, bit
   depth, chroma format) with the current ones.
   - Unchanged: send `OUTPUT_FLUSH`, wait for `OUTPUT_FLUSHED`, re-queue
     the output buffers.
   - Changed: report a source change to userspace (04 §4.3, §4.6). Keep returned
     frames back. After userspace has reallocated (or confirmed reuse), send
     `OUTPUT_FLUSH`, wait, and queue the new buffers.
4. Decoded frames come back as `OUTPUT` + `FRAME` descriptors. They carry
   visible width/height, flags, crop (AFBC) and the user data tag of the
   input buffer that produced them. Decoded metadata (`PARAM` display size,
   colour description) may arrive on the message channel.

### 3.8.3 End of stream, stop and close

- **[HW]** EOS (decode): queue a `BITSTREAM` descriptor on the input ring
  with flags EOS, length 0, VA = start of the bitstream region, and a host
  handle the host will recognise. The firmware drains, returns the last
  frame with `EOS`, and returns the EOS descriptor.
- **[HW]** Stop: send `STOP` and wait for `STATE_CHANGE` (stopped). Then
  send `INPUT_FLUSH` / `OUTPUT_FLUSH` for each port being stopped, and wait
  for the flushed replies. All buffers of a flushed port come back first.
- **[HW]** Restart after stop: send `GO`.
- **[HW]** Close: terminate and detach the LSID (01 §1.4.3), then free all
  session memory.

## 3.9 Encoder flow

- **[HW]** Same start sequence as §3.8.1, with encoder options in place of
  decoder options. The input ring carries `FRAME` descriptors (with
  TOP_PRESENT set when the buffer has data) and `PARAM` messages. The output
  ring carries empty `BITSTREAM` buffers. The firmware fills them and
  returns them with the frame type and the sync-frame flag.
- **[POLICY]** Vendor encoder option order: profile/level; rate control
  (`PARAM` 10, mode and bitrates); frame rate (`PARAM` 9, not for JPEG);
  RC I-bit options; scene-change and adaptive-quant options; QP range
  (`PARAM` 21) or fixed QPs (`PARAM` 12–14); I-frame QP range (`PARAM` 25);
  B frames (not VP8); P frames (always sent); GOP type; LTR and GOP reset;
  intra refresh; constrained intra; then the codec-specific options (HEVC
  entropy sync, temporal MVP, min CB size; tiles for HEVC/VP9; H.264
  CABAC, slice spacing, MV range, bit depth, chroma format, RGB→YUV,
  bandwidth limit, CABAC init IDC, crop, SEI, colour description, JPEG
  tables).
- **[HW]** Encoder EOS: queue a `FRAME` descriptor on the input ring with
  flags EOS, format NV12 and the host handle. The last bitstream buffer
  comes back with `EOS`.
- **[HW]** A bitrate of 0 is sent as rate-control mode 0 (off).

## 3.10 Core power management and suspend/resume

- **[HW]** Cores are powered by their power domains (00 §0.6). On the
  `0xE0C1AFE1` variant the engine can also gate power to idle cores by
  itself when `STRAP_PIN0[31:28]` bits are set and `CLKFORCE` is 0
  (00 §0.7.3).
- **[HW]** Before power-down, every LSID must be idle and detached. Firmware
  state lives in session memory, so a session can resume after power-up:
  re-attach its LSID (01 §1.4.1), re-add it to `JOBQUEUE`, and ring the
  doorbell.
- **[POLICY]** On suspend, the vendor driver sends `SWITCH` only to
  resident sessions with no frame budget (job frames = 0). Others are
  expected to switch out at the end of their job. It waits up to 1000 ms.
- **[POLICY]** On resume, the vendor driver re-runs the interrupt worker
  once, so that messages that arrived during the transition are seen.
