# 01: Mailbox, SCMI Transports and ACPI Clock Tables

This section covers how the application processor (AP) talks to the Sky1
system firmware:

- the **CIX mailbox** hardware (ACPI `CIXHA001`, DT `cix,sky1-mbox`)
- the **SCMI over mailbox** agent to the power-management processor (ACPI
  `CIXHA006`), used for clocks and DVFS
- the **SCMI over SMC** agent to Trusted Firmware-A, used for power domains
- the **CLKT** ACPI clock table and the ACPI clock-lookup device `CIXHA010`

SCMI is the public Arm *System Control and Management Interface*
specification (DEN0056). Its message format, shared-memory layout and
protocol commands are not repeated here except where Sky1 deviates or picks
values.

## 1.1 Mailbox controller

### 1.1.1 Instances

**[HW]** Each mailbox is **one-way**. Pairs give a two-way link. Sky1 has 8
instances (4 pairs), each 64 KiB:

| Link | AP→peer (tx) | Peer→AP (rx) | Transfer mode used |
|---|---|---|---|
| AP ↔ PM (power-management processor) | `0x06590000`, SPI 363 | `0x065A0000`, SPI 359 | Doorbell (channel 8) + shared memory |
| AP ↔ SE (security engine) | `0x05060000`, SPI 378 | `0x05070000`, SPI 379 | Register (channel 10) |
| AP ↔ DSP (audio DSP) | `0x070F0000`, SPI 232 | `0x07100000`, SPI 233 | FIFO (channel 9) |
| AP ↔ SFH (sensor hub) | `0x080A0000`, SPI 390 | `0x08090000`, SPI 391 | FIFO (channel 9) |

- Interrupts are level-high. "SPI n" is the GIC shared peripheral interrupt
  number. **[HW]** In ACPI `_CRS` the same interrupt appears as the absolute
  GSIV n + 32 (for example SPI 363 = GSIV 395).
- **[HW]** Direction is a property of the instance, not of the register
  map: a tx instance is written by the AP and read by the peer; an rx
  instance the reverse. DT uses `cix,mbox-dir = "tx" | "rx"`. The vendor
  devicetree and the ACPI `_DSD` use `cix,mbox_dir = 0` (tx) or `1` (rx).

> OPEN QUESTION: The ACPI `_DSD` property name for direction is inferred
> from the vendor devicetree and the vendor driver (which reads an integer
> `cix,mbox_dir`). Confirm against the actual DSDT.

### 1.1.2 Channels

**[HW]** Each instance exposes 11 logical channels:

| Channel | Type | Payload |
|---|---|---|
| 0–7 | Fast | One 32-bit word per channel, with its own interrupt |
| 8 | Doorbell | None; interrupt only (plus acknowledge) |
| 9 | FIFO | Up to 32 words through a 32-deep FIFO |
| 10 | Register | Up to 32 words in message registers, then doorbell + acknowledge |

Consumers name a channel by mailbox instance plus channel index
(`mboxes = <&mbox 8>` in DT; a reference plus one integer in an ACPI `_DSD`
package).

### 1.1.3 Register map

Offsets from the instance base. All registers 32-bit.

| Offset | Name | Description |
|---|---|---|
| `0x00 + 4n`, n = 0–31 | `MSG<n>` | Message words for the register channel. **Also used as SCMI shared memory** on the PM link (§1.2.2) |
| `0x80` | `DB_ACK` | Write bit 0 = raise the doorbell at the peer. Write bit 1 = raise the acknowledge back at the sender |
| `0x84` | `ERR_COMP` | Error/completion status (not used by Linux) |
| `0x88` | `ERR_COMP_CLR` | Clears `ERR_COMP` (not used by Linux) |
| `0x8C + 4k`, k = 0–7 | `FAST<k>` | Fast-channel data word k |
| `0xAC` | `FIFO_WR` | FIFO write port (tx side) |
| `0xB0` | `FIFO_RD` | FIFO read port (rx side) |
| `0xB4` | `FIFO_STAT` | `[0]` at/above watermark; `[1]` full; `[2]` empty; `[3]` underflow; `[4]` overflow |
| `0xB8` | `FIFO_WM` | FIFO watermark in words |
| `0xBC` | `INT_EN` | Interrupt enables for the **sender** (side A) |
| `0xC0` | `INT_EN_B` | Interrupt enables for the **receiver** (side B) |
| `0xC4` | `INT_CLR` | Write 1 to clear the matching `INT_STAT` bit |
| `0xC8` | `INT_STAT` | Interrupt status |
| `0xCC` | `FIFO_RST` | `[0]` = reset the FIFO |

`INT_EN`, `INT_EN_B`, `INT_CLR` and `INT_STAT` share this bit layout:

| Bit | Meaning |
|---|---|
| 0 | Doorbell raised (seen by the receiver) |
| 1 | Acknowledge raised (seen by the sender) |
| 2 | FIFO full |
| 3 | FIFO empty |
| 4 | FIFO crossed the watermark, rising (below → at/above) |
| 5 | FIFO crossed the watermark, falling |
| 6 | FIFO overflow |
| 7 | FIFO underflow |
| 8 | FIFO not empty |
| 9 + k | Fast channel k written (k = 0–7) |

- **[HW]** Both ends see the same instance. The AP enables the bits it
  wants delivered to itself: on a **tx** instance in `INT_EN` (it is
  side A), on an **rx** instance in `INT_EN_B` (it is side B). Each
  instance has one interrupt line, which goes to the AP.

> OPEN QUESTION: Whether the peer has its own interrupt line for the same
> instance (presumably yes, routed to the peer processor) is not visible.

### 1.1.4 Transfer procedures

Message length rule for register and FIFO channels: **[HW]** word 0 of the
message carries the message length in **bytes** in bits `[6:0]` (max 128).
The number of words sent is that length rounded up to whole words (word 0
included).

| Channel | Send (tx instance) | Receive (rx instance) |
|---|---|---|
| Doorbell | Write `0x1` to `DB_ACK`. Completion: wait for `INT_STAT[1]` (enable `INT_EN[1]`), clear it | On `INT_STAT[0]`: clear it, process, then write `0x2` to `DB_ACK` to acknowledge |
| Register | Write words to `MSG0…`, then `0x1` to `DB_ACK`. Completion as for doorbell | On `INT_STAT[0]`: clear, read `MSG0` to learn the length, read the rest, write `0x2` to `DB_ACK` |
| FIFO | Write the word count to `FIFO_WM`, push words into `FIFO_WR`, enable `INT_EN[3]` (empty). Completion: `INT_STAT[3]`, then disable bit 3 again | On `INT_STAT[2]` or `[4]`: clear them, pop `FIFO_RD` until `FIFO_STAT[2]` (empty) or 32 words |
| Fast k | Write the word to `FAST<k>` (no completion interrupt) | On `INT_STAT[9+k]`: clear, read `FAST<k>` |

FIFO channel setup: write `0x1` to `FIFO_RST`, then 32 to `FIFO_WM`. Enable
overflow (bit 6) on tx, or underflow + rising watermark (bits 7, 4) on rx.

- **[POLICY]** Linux logs FIFO overflow and underflow and clears them; it
  does not recover the message.
- **[POLICY]** Linux keeps the mailbox interrupt enabled across system
  suspend (the PM link must work during suspend and resume).

### 1.1.5 ACPI attachment (`CIXHA001`)

- **[HW]** One ACPI device per instance with `_HID` `CIXHA001`, a
  `Memory32Fixed` covering the 64 KiB instance, and one `Interrupt`.
- **[HW]** On the PM link the SCMI shared-memory devices (`CIXHA005`,
  §1.2.2) describe the **first 128 bytes of the same 64 KiB window**. The
  two `_CRS` ranges overlap. A driver must not treat this as a resource
  conflict.
- **[HW]** The mailbox register offsets are always relative to the 64 KiB
  base. (An earlier mainline devicetree split the window into a 128-byte
  shmem node plus a mailbox node starting at base + `0x80`, and the driver
  subtracted `0x80` when it detected that. The ACPI layout does not do
  that.)

## 1.2 SCMI agent to the power-management processor

### 1.2.1 Identity and protocols

| Item | Value |
|---|---|
| ACPI | `_HID` `CIXHA006`, with a `_DSD` holding `mboxes` and `shmem` references |
| DT | `compatible = "arm,scmi"`, `mbox-names = "tx", "rx"` |
| Transport | SCMI shared memory + CIX mailbox doorbell channel 8 |
| tx (AP → PM) | Mailbox `0x06590000` channel 8, shmem `0x06590000`, 128 bytes |
| rx (PM → AP, notifications / delayed responses) | Mailbox `0x065A0000` channel 8, shmem `0x065A0000`, 128 bytes |
| Protocols used | `0x13` Performance (DVFS), `0x14` Clock |
| Shared-memory access width | 32-bit only (`reg-io-width = 4`) |

- **[HW]** The 128-byte shared memory is exactly the mailbox's 32 message
  registers (§1.1.3). It holds the standard SCMI shared-memory channel
  layout. Access it with aligned 32-bit reads and writes only.
- **[HW]** Sending a command = fill the SCMI shared memory, then ring the
  doorbell on the tx instance (§1.1.4). The PM acknowledges with the
  mailbox acknowledge interrupt. The response is in the same shared memory
  once the SCMI "channel free" bit is set.
- **[HW]** Maximum message size is bounded by 128 bytes minus the SCMI
  shared-memory header overhead.

### 1.2.2 ACPI wiring

**[HW]** The `CIXHA006` device's `_DSD` (device-properties UUID
`daffd814-6eba-4d8c-8a91-bc9bbf4aa301`) contains:

| Property | Value |
|---|---|
| `mboxes` | Two entries: (reference to the tx `CIXHA001` device, 8) and (reference to the rx `CIXHA001` device, 8) |
| `shmem` | Two references: tx shared memory, rx shared memory. Each is a separate ACPI device (`CIXHA004` / `CIXHA005`) whose `_CRS` gives the 128-byte region |
| `reg-io-width` | 4 |

Entry 0 is tx and entry 1 is rx in both lists.

- **[HW]** Under ACPI the per-protocol child devices carry **no channels
  of their own**. A client discovers protocols with the SCMI Base protocol
  (`PROTOCOL_ATTRIBUTES`, `DISCOVER_LIST_PROTOCOLS`) and uses the one
  tx/rx channel pair for all of them.
- **[HW]** *(vendor)* The `CIXHA006` device does have protocol child
  nodes, for example `\_SB.SCMI.DVFS` with an integer `reg` = `0x13`
  (performance protocol). They may report `_STA` = 0 (namespace-only
  stubs), so they must be looked up even when "not present". Consumers
  (GPU, NPU) name performance domains through a `_DSD` `power-domains`
  reference to such a child plus a domain index; the child is the
  provider handle, not a separate device to probe.

> OPEN QUESTION: Which of `CIXHA004` and `CIXHA005` is tx and which is rx
> is not stated; follow the `shmem` order. The source only names
> `CIXHA005` as overlapping a mailbox window.

### 1.2.3 Performance domains (protocol `0x13`)

**[HW]** Domain IDs (from the public `sky1-power.h`):

| ID | Domain | ID | Domain |
|---|---|---|---|
| 0 | GPU core | 6 | CPU middle cluster 1 |
| 1 | GPU top | 7 | DSU |
| 2 | CPU little cluster | 8 | NPU |
| 3 | CPU big cluster 0 | 9 | VPU |
| 4 | CPU big cluster 1 | 10 | CI-700 interconnect |
| 5 | CPU middle cluster 0 | 11 | NI-700 interconnect |

Operating points are discovered with `PERFORMANCE_DESCRIBE_LEVELS`.

- **[POLICY]** The Sky1-Linux GPU driver rate-limits GPU level requests
  because the firmware is reported to be unstable under rapid requests
  (see the Panthor spec, section 11).

### 1.2.4 Clocks (protocol `0x14`)

- **[HW]** Clock IDs are the `CLK_TREE_*` values in the public
  dual-licensed header `cix,sky1.h` (0 … 272; IDs 267–271 are not exposed to
  the AP). The same numbers appear in ACPI CLKT tables (§1.4).
- **[HW]** Clocks are gated and rated with standard SCMI clock commands.

## 1.3 SCMI agent to Trusted Firmware-A (power domains)

### 1.3.1 Identity

| Item | Value |
|---|---|
| DT | `compatible = "arm,scmi-smc"`, `arm,smc-id = <0xc2000001>` |
| Conduit | SMC, SMCCC function ID **`0xC2000001`** (SiP service, fast call, 32-bit convention) |
| Shared memory | `0x84380000`, 128 bytes, reserved (`no-map`) |
| Protocol | `0x11` Power domain management |
| ACPI | No SCMI device for this agent (see §1.3.4) |

### 1.3.2 Calling sequence

**[HW]** To send one SCMI command:

1. Wait until the shared memory's channel-status "free" bit (bit 0 of the
   word at offset `0x04`) is set.
2. Clear channel status (write 0 to offset `0x04`) and flags (offset
   `0x10`; 0 = polled, no completion interrupt).
3. Write the message header at offset `0x18` (message ID in `[7:0]`,
   message type 0 in `[9:8]`, protocol ID in `[17:10]`, token in `[27:18]`)
   and the payload from offset `0x1C`.
4. Write the length (header + payload, in bytes) at offset `0x14`.
5. Issue `SMC #0` with `x0 = 0xC2000001`. `x1` and `x2` may carry the
   shared-memory page number (address ÷ 4096) and the offset within the
   page; the firmware also accepts zeros (see the open question).
6. On return, `x0` = 0 means the call was delivered. The SCMI status is the
   first payload word (offset `0x1C`), followed by any return values.

(Offsets 0x04–0x1C are the standard SCMI shared-memory layout.)

> OPEN QUESTION: The mainline Sky1 devicetree uses the plain SMC transport,
> which passes no page/offset in `x1`/`x2`. The Sky1-Linux GPU driver's
> direct ACPI-path call passes them. Both are reported to work, so the
> firmware probably uses a fixed shared-memory address and ignores
> `x1`/`x2`. Passing them is the safer choice.

### 1.3.3 Power domains

**[HW]** Domain IDs (public `sky1-power.h`):

| ID | Domain | ID | Domain |
|---|---|---|---|
| 0 | Audio | 11 | VPU top |
| 1 | PCIe controller 0 | 12–15 | VPU cores 0–3 |
| 2 | PCIe (dummy) | 16–18 | NPU cores 0–2 |
| 3 | PCIe hub | 19 | NPU top |
| 4 | Multimedia hub | 20 | ISP 0 |
| 5 | Multimedia hub SMMU | 21 | GPU |
| 6–10 | DPU 0–4 | | (32 IDs reserved) |

- **[HW]** `POWER_STATE_SET` (message `0x04`): payload = flags (0 =
  synchronous), domain ID, power state (0 = on; `0x40000000` = off, the
  standard SCMI "context lost, off" encoding).
- **[POLICY]** Linux on Sky1 follows every `POWER_STATE_SET` with
  `POWER_STATE_GET` and treats a mismatch as an I/O error (copied from the
  vendor kernel).
- **[POLICY]** Linux registers the power-domain provider early (before
  ordinary drivers probe) on Sky1, because turning a domain on also sets up
  the access permissions some devices (the GPU) need before their first
  register access.

### 1.3.4 Under ACPI

- **[HW]** The DSDT provides no SCMI-over-SMC device. Device power is
  modelled with **ACPI power resources** (`_ON` / `_OFF`, `_PS0` / `_PS3`)
  instead. An OS that walks the ACPI device power state machine gets
  domains switched by firmware AML.
- **[POLICY]** Sky1-Linux disables the SCMI power-domain provider entirely
  under ACPI, and the GPU driver makes the raw SMC call of §1.3.2 for
  domain 21 because its ACPI node lacks a power resource that does it.

> OPEN QUESTION: What the AML power resources do internally (whether they
> call TF-A, write PM registers, or both) is not visible without the DSDT.
> The audio subsystem's power resource is reported to switch its power
> gates and run memory repair (section 02 §2.5.3).

## 1.4 ACPI clock table (CLKT) and clock lookup device (`CIXHA010`)

### 1.4.1 CLKT format

**[HW]** A device that consumes clocks has a named object `CLKT` (not a
standard ACPI method) that evaluates to a package of N entries:

| Element | Type | Meaning |
|---|---|---|
| 0 | Integer | SCMI clock ID (`CLK_TREE_*`, §1.2.4) |
| 1 | String | Connection (clock) name; empty string = unnamed |
| 2 | Reference (optional) | The **consumer device**. Usually the device that owns the CLKT, but not always |

- **[HW]** Element 2 matters: at least one DSDT `CLKT` lists a clock for a
  **different** device (the FCH DMA's clock appears in another DMA
  device's table). Assign each entry to the device in element 2, not to
  the table's owner.
- **[HW]** Clock names in CLKT are the vendor names (for example
  `fch_timer_func_clk`), which can differ from the short names used by
  mainline DT bindings.

### 1.4.2 `CIXHA010`

- **[HW]** A single ACPI device `CIXHA010` stands for "the ACPI clock
  provider". It has no registers. It exists so that devices can declare a
  dependency on it.
- **[POLICY]** Sky1-Linux binds a driver to it that walks the namespace,
  evaluates every `CLKT`, and registers each (consumer, name) → SCMI clock
  mapping. It defers until at least one SCMI clock exists. It also makes
  every CLKT consumer depend on `CIXHA010` so that consumers probe after
  the mappings exist.
- **[HW]** Some DSDT dependency chains run through `CIXHA010` (for example
  `CIXHA010` → USB/PCIe PHY → PCIe controller). An OS that honours `_DEP`
  must provide something that "satisfies" `CIXHA010`, or those devices
  never probe.

## 1.5 Other CIX ACPI tables (summary)

Covered in section 02: `RSTL` (reset lookups), `RSNL` (resource names),
`DLKL` (device links), all handled by the ACPI device `CIXA1019`.
