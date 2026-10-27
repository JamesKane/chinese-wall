# VideoCC, Power Domains, SMMU, PAS Boot, and VPU 2.0 Registers

## 1. RPMh Power Domains and Critical Hardware Hazards

### 1.1 `MMCX` Power Rail Hazard
- **[HW]** Both the Video Clock Controller (`videocc` at physical address `0x0abf0000`) and the Iris VPU 2.0 register space (`0x0aa00000`) reside inside the Multimedia CX (`MMCX`) power rail.
- **[HW]** **CRITICAL HAZARD:** Reading or writing any `videocc` (`0x0abf0000`) or VPU (`0x0aa00000`) MMIO register while the `MMCX` power rail is unpowered causes an immediate, unrecoverable bus abort that resets the SC8280XP SoC without generating a crash dump.
- **[HW]** On the Radxa Dragon Q8B, UEFI firmware votes `MMCX` on prior to OS handoff to power the display subsystem (`MDSS0`). An OS power domain driver must aggregate RPMh resource votes across all consumers (`MDSS0` and `VPU`) so that powering down or probing the VPU never drops the `MMCX` vote below the level required by active display controllers.

### 1.2 RPMh ARC Power Domain Resources and Operating Performance Points (OPPs)
- **[HW]** The VPU requires two RPMh ARC voltage rail resources found in Command DB:
  - `mx.lvl` (Command DB address `0x3001c`): Memory logic rail (`MX`).
  - `mmcx.lvl` (Command DB address `0x30020`): Multimedia core logic rail (`MMCX`).
- **[HW]** The SC8280XP VPU defines six Operating Performance Points (OPPs) mapping core codec clock rates (`VIDEO_CC_MVS0_CLK`) and their parent PLL divider source (`video_cc_mvs0_clk_src`, which runs at 3× the core clock rate) to required `MX` and `MMCX` RPMh ARC performance levels:

| Core Codec Clock (`VIDEO_CC_MVS0_CLK`) | Parent RCG Rate (`video_cc_mvs0_clk_src`) | `MVS0C` Bus/Wrapper Clock (`÷2` of RCG) | Required `mx.lvl` Corner | Required `mmcx.lvl` Corner | Tag |
|---|---|---|---|---|---|
| `240,000,000` Hz (240 MHz) | `720,000,000` Hz | `360,000,000` Hz | `SVS` (`128`) | `LOW_SVS` (`64`) | **[HW]** |
| `338,000,000` Hz (338 MHz) | `1,014,000,000` Hz | `507,000,000` Hz | `SVS` (`128`) | `SVS` (`128`) | **[HW]** |
| `366,000,000` Hz (366 MHz) | `1,098,000,000` Hz | `549,000,000` Hz | `SVS_L1` (`192`) | `SVS_L1` (`192`) | **[HW]** |
| `444,000,000` Hz (444 MHz) | `1,332,000,000` Hz | `666,000,000` Hz | `SVS_L1` (`192`) | `NOM` (`256`) | **[HW]** |
| `533,000,000` Hz (533 MHz) | `1,599,000,000` Hz | `799,500,000` Hz | `NOM` (`256`) | `TURBO` (`384`) | **[HW]** |
| `560,000,000` Hz (560 MHz) | `1,680,000,000` Hz | `840,000,000` Hz | `NOM` (`256`) | `TURBO_L1` (`416`) | **[HW]** |

- **[POLICY]** For initial bring-up and single-OPP operation without dynamic frequency scaling, voting `mx.lvl >= 256` (`NOM`) and `mmcx.lvl >= 384` (`TURBO`) before touching `videocc` or VPU registers allows continuous operation at the 533 MHz core clock rate.

---

## 2. Interconnect Bandwidth Voting

- **[HW]** The VPU requires two NoC interconnect paths to be voted active via the RPMh Bus Clock Manager (BCM):
  1. **`cpu-cfg` path:** Master `MASTER_APPSS_PROC` (`gem_noc`) to Slave `SLAVE_VENUS_CFG` (`config_noc`), tag `ACTIVE_ONLY`. Enables CPU MMIO access to `0x0aa00000`.
  2. **`video-mem` path:** Master `MASTER_VIDEO_P0` (`mmss_noc`) to Slave `SLAVE_EBI1` (`mc_virt`), tag `ALWAYS` (BCM `MM1`). Enables VPU DMA access to DDR memory.
- **[POLICY]** Bandwidth bounds and per-resolution DDR bandwidth tables used by host software:

| Path / Operating Mode | Average Bandwidth (kB/s) | Peak Bandwidth (kB/s) | Tag |
|---|---|---|---|
| `cpu-cfg` (when VPU active) | `1,000` | `1,000` | **[POLICY]** |
| `video-mem` (initial power-on / minimum) | `1,000` | `1,000` | **[POLICY]** |
| `video-mem` (maximum bound) | `15,000,000` | `15,000,000` | **[POLICY]** |
| `video-mem` (1920×1080 @ 30 fps) | `416,000` | `416,000` | **[POLICY]** |
| `video-mem` (1920×1080 @ 60 fps) | `812,000` | `812,000` | **[POLICY]** |
| `video-mem` (4096×2160 @ 30 fps) | `1,224,000` | `1,224,000` | **[POLICY]** |
| `video-mem` (4096×2160 @ 60 fps) | `2,403,000` | `2,403,000` | **[POLICY]** |

---

## 3. Global Clock Controller (`GCC`) and Video Clock Controller (`videocc`)

### 3.1 `GCC` Video AXI0 Interface Clock and Bus Reset
- **[HW]** Before accessing VPU registers, the `iface` clock (`GCC_VIDEO_AXI0_CLK`) in the Global Clock Controller (`GCC` base `0x00100000`) must be enabled.

| Offset from `GCC` Base | Register Name | Bits | Field Description | Tag |
|---|---|---|---|---|
| `0x28010` | `GCC_VIDEO_AXI0_CBCR` | `[31]` | `CLK_OFF`: `0` = clock is running, `1` = clock is off (poll until `0` after enabling) | **[HW]** |
| `0x28010` | `GCC_VIDEO_AXI0_CBCR` | `[2]` | `ARES` (`GCC_VIDEO_AXI0_CLK_ARES`): `1` = assert `bus` reset, `0` = deassert `bus` reset | **[HW]** |
| `0x28010` | `GCC_VIDEO_AXI0_CBCR` | `[0]` | `CLK_ENABLE`: `1` = enable branch clock, `0` = disable branch clock | **[HW]** |

### 3.2 `videocc` (`0x0abf0000`) Register Map
- **[HW]** The Video Clock Controller on SC8280XP (`compatible = "qcom,sc8280xp-videocc"`) is mapped at physical address `0x0abf0000` (size `0x10000`). Note that SC8280XP relocates the XO and Sleep RCG/CBCR offsets relative to SM8350.

| Offset | Register / Block Name | Description | Tag |
|---|---|---|---|
| `0x042c` | `VIDEO_CC_PLL0` | Lucid 5LPE Alpha PLL (`video_pll0`) base register (VCO range `249.6 MHz` – `1800 MHz`) | **[HW]** |
| `0x07d0` | `VIDEO_CC_PLL1` | Lucid 5LPE Alpha PLL (`video_pll1`) base register (unused by VPU 0; used by CVP/MVS1 on dual-instance variants) | **[HW]** |
| `0x0b94` | `VIDEO_CC_MVS0_CMD_RCGR` | Root Clock Generator for `video_cc_mvs0_clk_src` (`+0x00` `CMD_RCGR`, `+0x04` `CFG_RCGR`) | **[HW]** |
| `0x0bd4` | `VIDEO_CC_AHB_CMD_RCGR` | Shared AHB Root Clock Generator (`+0x00` `CMD_RCGR`, `+0x04` `CFG_RCGR`) | **[HW]** |
| `0x0bf4` | `VIDEO_CC_CVP_MVS0C_BCR` | Block reset for `MVS0C` wrapper domain (bit 0 = `BLK_ARES`) | **[HW]** |
| `0x0bf8` | `MVS0C_GDSC` | `venus` power domain Global Distributed Switch Controller (`CFG_GDSCR` at `0xbfc`) | **[HW]** |
| `0x0c34` | `VIDEO_CC_MVS0C_CBCR` | `core` clock (`VIDEO_CC_MVS0C_CLK`) branch control & `VIDEO_CC_MVS0C_CLK_ARES` reset (bit 2, requires 400 µs delay) | **[HW]** |
| `0x0c54` | `VIDEO_CC_MVS0C_DIV_CDIVR` | `video_cc_mvs0c_div2_div_clk_src` divider register (bits `[3:0]` = `div - 1`, fixed divide-by-2 = `1`) | **[HW]** |
| `0x0d14` | `VIDEO_CC_CVP_MVS0_BCR` | Block reset (`core` reset in driver) for `MVS0` codec core (bit 0 = `BLK_ARES`) | **[HW]** |
| `0x0d18` | `MVS0_GDSC` | `vcodec0` power domain Global Distributed Switch Controller (`CFG_GDSCR` at `0xd1c`) | **[HW]** |
| `0x0d34` | `VIDEO_CC_MVS0_CBCR` | `vcodec0_core` clock (`VIDEO_CC_MVS0_CLK`) branch control (bit 0 = enable, bit 1 = HWCG, bit 31 = `CLK_OFF`) | **[HW]** |
| `0x0d54` | `VIDEO_CC_MVS0_DIV_CDIVR` | `video_cc_mvs0_div_clk_src` divider register (bits `[3:0]` = `div - 1`, fixed divide-by-3 = `2`) | **[HW]** |
| `0x0e54` | `VIDEO_CC_CVP_INTERFACE_BCR` | Interface block reset (`VIDEO_CC_INTERFACE_BCR`, bit 0 = `BLK_ARES`) | **[HW]** |
| `0x0e58` | `VIDEO_CC_AHB_CBCR` | Always-on AHB branch clock (`VIDEO_CC_AHB_CLK`) | **[HW]** |
| `0x0f14` | `VIDEO_CC_XO_CMD_RCGR` | SC8280XP XO RCG (`video_cc_xo_clk_src`) | **[HW]** |
| `0x0f34` | `VIDEO_CC_XO_CBCR` | SC8280XP Always-on XO branch clock (`VIDEO_CC_XO_CLK`) | **[HW]** |
| `0x0f38` | `VIDEO_CC_SLEEP_CMD_RCGR` | SC8280XP Sleep RCG (`video_cc_sleep_clk_src`) | **[HW]** |
| `0x0f58` | `VIDEO_CC_SLEEP_CBCR` | SC8280XP Sleep branch clock (`VIDEO_CC_SLEEP_CLK`, bit 0 = enable) | **[HW]** |

### 3.3 `video_pll0` (Lucid 5LPE PLL at `videocc + 0x042c`) Configuration and Locking
- **[HW]** `video_pll0` is a Lucid 5LPE Alpha PLL clocked from the 19.2 MHz `bi_tcxo` reference. Its register offsets relative to `0x042c` are:

| Offset from `0x042c` | Absolute Offset | Register Name | Description | Tag |
|---|---|---|---|---|
| `+0x00` | `0x042c` | `PLL_MODE` | Bit 0: `PLL_OUTCTRL`; Bit 2: `PLL_RESET_N`; Bit 11: `PCAL_DONE`; Bit 13: `ALPHA_PLL_ACK_LATCH`; Bit 14: `PLL_LATCH_INPUT`; Bit 23: `PLL_UPDATE_BYPASS`; Bit 29: `PLL_VOTE_FSM_ENA`; Bit 31: `PLL_LOCK_DET` | **[HW]** |
| `+0x04` | `0x0430` | `PLL_L_VAL` | Integer multiplier `L` (bits `[15:0]`) | **[HW]** |
| `+0x08` | `0x0434` | `PLL_CAL_L_VAL` | Calibration `L` value (bits `[15:0]`, initialized to `0x44`) | **[HW]** |
| `+0x0c` | `0x0438` | `PLL_USER_CTL` | Bit 0: `PLL_POST_DIV_MAIN` (`0x7` mask); Bit 21: `ENABLE_VOTE_RUN` | **[HW]** |
| `+0x10` | `0x043c` | `PLL_USER_CTL_U` | Initialized to `0x00000805` | **[HW]** |
| `+0x14` | `0x0440` | `PLL_USER_CTL_U1` | Initialized to `0x00000000` | **[HW]** |
| `+0x18` | `0x0444` | `PLL_CONFIG_CTL` | Initialized to `0x20485699` | **[HW]** |
| `+0x1c` | `0x0448` | `PLL_CONFIG_CTL_U` | Initialized to `0x00002261` | **[HW]** |
| `+0x20` | `0x044c` | `PLL_CONFIG_CTL_U1` | Initialized to `0x2a9a699c` | **[HW]** |
| `+0x24` | `0x0450` | `PLL_TEST_CTL` | Initialized to `0x00000000` | **[HW]** |
| `+0x28` | `0x0454` | `PLL_TEST_CTL_U` | Initialized to `0x00000000` | **[HW]** |
| `+0x2c` | `0x0458` | `PLL_TEST_CTL_U1` | Initialized to `0x01800000` | **[HW]** |
| `+0x30` | `0x045c` | `PLL_STATUS` | PLL status register | **[HW]** |
| `+0x38` | `0x0464` | `PLL_OPMODE` | Operational mode (`0` = Standby, `1` = Run) | **[HW]** |
| `+0x40` | `0x046c` | `PLL_ALPHA_VAL` | 16-bit fractional multiplier `ALPHA` (`f_pll = 19.2 MHz * (L + ALPHA / 2^16)`) | **[HW]** |

- **[HW]** Multiplier values (`L_VAL` and `ALPHA_VAL`) for each SC8280XP OPP frequency:

| Target PLL Rate (`video_cc_mvs0_clk_src`) | `PLL_L_VAL` (`0x0430`) | `PLL_ALPHA_VAL` (`0x046c`) | Exact Formula (`19.2 MHz × (L + ALPHA/65536)`) | Tag |
|---|---|---|---|---|
| `720,000,000` Hz (init default) | `0x0025` (37) | `0x8000` (32768) | `19.2 MHz × 37.5 = 720 MHz` | **[HW]** |
| `1,014,000,000` Hz | `0x0034` (52) | `0xd000` (53248) | `19.2 MHz × 52.8125 = 1014 MHz` | **[HW]** |
| `1,098,000,000` Hz | `0x0039` (57) | `0x3000` (12288) | `19.2 MHz × 57.1875 = 1098 MHz` | **[HW]** |
| `1,332,000,000` Hz | `0x0045` (69) | `0x6000` (24576) | `19.2 MHz × 69.375 = 1332 MHz` | **[HW]** |
| `1,599,000,000` Hz (533 MHz core) | `0x0053` (83) | `0x4800` (18432) | `19.2 MHz × 83.28125 = 1599 MHz` | **[HW]** |
| `1,680,000,000` Hz (560 MHz core) | `0x0057` (87) | `0x8000` (32768) | `19.2 MHz × 87.5 = 1680 MHz` | **[HW]** |

- **[HW]** **Lucid 5LPE PLL Initialization and Enable Sequence:**
  1. Write initial tuning registers (`PLL_CAL_L_VAL = 0x44`, `PLL_CONFIG_CTL = 0x20485699`, `PLL_CONFIG_CTL_U = 0x00002261`, `PLL_CONFIG_CTL_U1 = 0x2a9a699c`, `PLL_TEST_CTL = 0`, `PLL_TEST_CTL_U = 0`, `PLL_TEST_CTL_U1 = 0x01800000`, `PLL_USER_CTL = 0`, `PLL_USER_CTL_U = 0x00000805`, `PLL_USER_CTL_U1 = 0`).
  2. Write desired `PLL_L_VAL` (e.g., `0x53` for 1599 MHz) and `PLL_ALPHA_VAL` (e.g., `0x4800` for 1599 MHz).
  3. Set `PLL_UPDATE_BYPASS` (bit 23) in `PLL_MODE`, clear `PLL_OUTCTRL` (bit 0) in `PLL_MODE`, write `0` (Standby) to `PLL_OPMODE` (`0x0464`), and set `PLL_RESET_N` (bit 2) in `PLL_MODE`.
  4. If changing `L`/`ALPHA` while `PLL_MODE` already has `PLL_RESET_N` asserted: set `PLL_LATCH_INPUT` (bit 14 of `PLL_MODE`), poll `PLL_MODE` until `ALPHA_PLL_ACK_LATCH` (bit 13) reads `1` (timeout 500 µs), then clear `PLL_LATCH_INPUT` (bit 14).
  5. To enable the PLL:
     - Set `PLL_OUT_MAIN` (bit 0) in `PLL_USER_CTL` (`0x0438`).
     - Clear `PLL_OUTCTRL` (bit 0) in `PLL_MODE`, set `PLL_RESET_N` (bit 2) in `PLL_MODE`, and write `1` (`OPMODE_RUN`) to `PLL_OPMODE` (`0x0464`).
     - Poll `PLL_MODE` (`0x042c`) until `PLL_LOCK_DET` (bit 31) reads `1` (timeout 1500 µs).
     - Set `PLL_OUTCTRL` (bit 0) in `PLL_MODE` and execute a memory barrier.

### 3.4 `video_cc_mvs0_clk_src` RCG (`videocc + 0x0b94`) and Dividers
- **[HW]** `video_cc_mvs0_clk_src` selects `video_pll0` via source select `1` (`CFG_RCGR` at `0x0b98` bits `[10:8] = 1`) and pre-divider `1` (`hid_div = 1`, meaning `(2 * 1) - 1 = 1` in bits `[4:0]`):
  1. Write `(1 << 8) | 1 = 0x00000101` to `VIDEO_CC_MVS0_CFG_RCGR` (`0x0b98`).
  2. Set `CMD_UPDATE` (bit 0 = `1`) in `VIDEO_CC_MVS0_CMD_RCGR` (`0x0b94`).
  3. Poll `VIDEO_CC_MVS0_CMD_RCGR` (`0x0b94`) until bit 0 clears to `0` (timeout 500 µs).
- **[HW]** Set fixed dividers for the two child domains:
  - Write `1` (divide by 2) to `VIDEO_CC_MVS0C_DIV_CDIVR` (`0x0c54`) so `MVS0C` runs at `PLL / 2` (`799.5 MHz` when PLL is `1599 MHz`).
  - Write `2` (divide by 3) to `VIDEO_CC_MVS0_DIV_CDIVR` (`0x0d54`) so `MVS0` runs at `PLL / 3` (`533 MHz` when PLL is `1599 MHz`).

### 3.5 `MVS0C_GDSC` (`0x0bf8`) and `MVS0_GDSC` (`0x0d18`) Power Domain Control
- **[HW]** Both GDSCs share the same register bit layout:
  - Bit 0: `SW_COLLAPSE` (`0` = power on, `1` = power collapse).
  - Bit 1: `HW_CONTROL` (`0` = software control, `1` = hardware solver control; used only on `MVS0_GDSC`).
  - Bit 11: `RETAIN_FF_ENABLE` (`1` = enable retention flip-flops; must be set after power-on).
  - Bits `[15:12]` of `GDSCR`: `PWR_ON` / `EN_REST_WAIT`; in `CFG_GDSCR` (`GDSCR + 0x04`), bit 31 (`GDSC_POWER_UP_COMPLETE`) or bit 31 of `GDSCR` (`PWR_ON`) indicates power-on completion, and bit 16 of `CFG_GDSCR` (`GDSC_POWER_DOWN_COMPLETE`) indicates power-off completion.
- **[HW]** **GDSC Power-On Sequence (`MVS0C_GDSC` at `0x0bf8`, then `MVS0_GDSC` at `0x0d18`):**
  1. Clear `SW_COLLAPSE` (bit 0) and `HW_CONTROL` (bit 1) in `GDSCR`.
  2. Poll `GDSCR` bit 31 (`PWR_ON`) (or `CFG_GDSCR` at `GDSCR + 4` bit 31) until it reads `1` (timeout 1500 µs).
  3. Set `RETAIN_FF_ENABLE` (bit 11) in `GDSCR`.
- **[HW]** **GDSC Hardware Mode Handoff (`MVS0_GDSC` only):**
  - During power-on and register initialization, `MVS0_GDSC` (`0x0d18`) must remain in software control (`HW_CONTROL` bit 1 = `0`).
  - Once VPU registers are initialized (or immediately after powering on before firmware execution), set `HW_CONTROL` (bit 1 = `1`) in `MVS0_GDSC` (`0x0d18`) so the VPU firmware/hardware controller can autonomously power-collapse and wake the `vcodec0` core between frames.
  - Before powering off `MVS0_GDSC` or accessing `VCODEC_BASE` (`0x0aa00000`–`0x0aa9ffff`) registers during power-down, clear `HW_CONTROL` (bit 1 = `0`) in `MVS0_GDSC` (`0x0d18`), wait 1 µs, and poll bit 31 (`PWR_ON`) until `1`.

---

## 4. Apps SMMU, TrustZone PAS 9, and Memory Protection (`MP_VIDEO_VAR`)

### 4.1 Apps SMMU Stream IDs
- **[HW]** The VPU issues DMA transactions through the Apps ARM SMMU (`0x15000000`) using two masked stream ID ranges, both hardware cache-coherent (`dma-coherent`):
  1. **Host / Non-Secure Stream (`0x2a00`, mask `0x0400`):** Used by the VPU microcontroller (`UC_REGION` queues, SFR buffer) and non-secure pixel/bitstream/scratch DMA transfers. Matches Stream IDs `0x2a00` and `0x2e00`.
  2. **Firmware / Secure Stream (`0x2a07`, mask `0x0400`):** Matches Stream IDs `0x2a07` and `0x2e07`.
- **[HW]** Both stream entries in the SMMU Stream Match Registers (`SMR`) must be bound to the **same** Stage-1 Context Bank (CB) and page table so the VPU sees a unified IOVA space across `0x2a00` and `0x2a07`.

### 4.2 Firmware Loading (`PAS ID 9`) and `MP_VIDEO_VAR` SMC
- **[HW]** The VPU firmware image `qcom/vpu/vpu20_p4_gen2_s6.mbn` is loaded into the reserved physical memory region `pil_video_mem` at physical base `0x8c600000` (size `0x00700000` = 7 MiB) using the standard Qualcomm MDT/MBN relocation loader and authenticated via TrustZone Peripheral Authentication Service (PAS) with **PAS ID `9`**:
  1. Invoke PAS Init Image (`pas_id = 9`, metadata) (`SVC = 0x02`, `CMD = 0x01`).
  2. Copy ELF `PT_LOAD` segments into `0x8c600000`..`0x8ccfffff` and flush CPU caches to PoC.
  3. Invoke PAS Mem Setup (`pas_id = 9`, `addr = 0x8c600000`, `size = 0x700000`) (`SVC = 0x02`, `CMD = 0x02`).
  4. Invoke PAS Auth and Reset (`pas_id = 9`) (`SVC = 0x02`, `CMD = 0x05`).
- **[HW]** Immediately after PAS Auth and Reset (`pas_id = 9`) succeeds, the host **must** invoke the TrustZone Memory Protection Video Variable SMC (`SVC_MP = 0x0c`, `MP_VIDEO_VAR = 0x08`) to partition the VPU's Stage-2/XPU address space:

| SMC Parameter | Value | Description | Tag |
|---|---|---|---|
| `owner_id` | `2` (`ARM_SMCCC_OWNER_SIP`) | Qualcomm SiP SMC owner | **[HW]** |
| `svc_id` | `0x0c` (`SVC_MP`) | Memory Protection service | **[HW]** |
| `cmd_id` | `0x08` (`MP_VIDEO_VAR`) | Video variable region setup (`fn_id = 0x02000c08` or `0x42000c08` for 64-bit) | **[HW]** |
| `arginfo` | `4` (four 32-bit values: `0x00000004`) | Four 32-bit value arguments | **[HW]** |
| `arg0` (`cp_start`) | `0x00000000` | Start of content-protected IOVA region | **[HW]** |
| `arg1` (`cp_size`) | `0x25800000` (600 MiB) | Total size of content-protected IOVA region | **[HW]** |
| `arg2` (`cp_nonpixel_start`) | `0x01000000` (16 MiB) | Start of non-pixel protected sub-region | **[HW]** |
| `arg3` (`cp_nonpixel_size`) | `0x24800000` (584 MiB) | Size of non-pixel protected sub-region | **[HW]** |

- **[HW]** **IOVA Allocation Constraint:** Because TrustZone reserves IOVA `0x00000000`–`0x257fffff` (`0x25800000` bytes) for content protection and the VPU's DMA mask is `0xdfffffff` (`0xe0000000 - 1`), **all** host-allocated DMA buffers (the HFI queue table, SFR buffer, bitstream buffers, raw YUV frame buffers, and internal scratch/persist buffers) **must** be mapped in the IOVA window **`0x25800000` to `0xdfffffff`**. Any DMA buffer mapped below `0x25800000` will trigger a permission fault on the VPU.

---

## 5. VPU 2.0 MMIO Register Map (`0x0aa00000`)

### 5.1 Sub-Block Base Offsets
- **[HW]** The VPU 2.0 register space at `0x0aa00000` (size `0x00100000` = 1 MiB) is organized into the following sub-blocks:

| Sub-Block Name | Offset from `0x0aa00000` | Description | Tag |
|---|---|---|---|
| `VCODEC_BASE` | `0x00000000` | Video codec hardware engine registers (powered by `MVS0_GDSC`) | **[HW]** |
| `AON_MVP_NOC_RESET` | `0x0001f000` | MVP NoC reset control block | **[HW]** |
| `CPU_BASE` | `0x000a0000` | VPU firmware CPU Command/Status (`CPU_CS`) and Interrupt Controller (`CPU_IC`) | **[HW]** |
| `WRAPPER_BASE` | `0x000b0000` | VPU wrapper configuration, clock gating, and interrupt status | **[HW]** |
| `WRAPPER_TZ_BASE` | `0x000c0000` | TrustZone wrapper status and watchdog interrupt status | **[HW]** |
| `AON_BASE` | `0x000e0000` | Always-On (`AON`) wrapper power collapse control | **[HW]** |

### 5.2 Complete VPU 2.0 Register Table

| Offset | Register Name | Bits & Field Descriptions | Tag |
|---|---|---|---|
| `0x0000115c` | `VCODEC_NOC_ERL_MAIN_SWID_LOW` | Read during VPU power-off sequence before NOC reset to flush NoC bus | **[HW]** |
| `0x0001f000` | `AON_WRAPPER_MVP_NOC_RESET_REQ` | Bit 0 (`0x1`): Request MVP NoC reset; Bit 1 (`0x2`): Reset logic | **[HW]** |
| `0x0001f004` | `AON_WRAPPER_MVP_NOC_RESET_ACK` | Bits `[1:0]` (`0x3`): Poll until reads `0x3` after writing `0x3` to `RESET_REQ` | **[HW]** |
| `0x000a001c` | `CPU_CS_A2HSOFTINTCLR` | Bit 0 (`0x1`): Write `1` to clear ARM-to-Host software interrupt from VPU firmware | **[HW]** |
| `0x000a0048` | `CTRL_INIT` | Bit 0 (`0x1`): Write `1` to trigger VPU firmware boot/initialization | **[HW]** |
| `0x000a004c` | `CTRL_STATUS` | Bit 0 (`0x1`): `CTRL_INIT_STATUS__M` (`1` = firmware booted and ready); Bit 1 (`0x2`): `CTRL_ERROR_STATUS__M`; Bits `[7:1]` (`0xfe`): Watchdog/fault error status | **[HW]** |
| `0x000a0050` | `QTBL_INFO` | Bit 0 (`0x1`): `1` = HFI queue table is configured and valid | **[HW]** |
| `0x000a0054` | `QTBL_ADDR` | 32-bit IOVA of the HFI Interface Queue Table header (must be 4 KiB-aligned, in `0x25800000`–`0xdfffffff`) | **[HW]** |
| `0x000a0058` | `CPU_CS_SCIACMDARG3` | System Control Interface Argument 3; Bits `[5:3]` (`0x38`): `INIT_IDLE_MSG_MASK` (polled for `!= 0` before power-off) | **[HW]** |
| `0x000a005c` | `SFR_ADDR` | 32-bit IOVA of the 4 KiB Subsystem Failure Reason (SFR) buffer ORed with `core_arch` (`sfr_iova + 1` where `VIDEO_ARCH_LX = 1`) | **[HW]** |
| `0x000a0064` | `UC_REGION_ADDR` | 32-bit base IOVA of the Uncached Queue Region (the DMA buffer containing the queue table and queues, 4 KiB-aligned) | **[HW]** |
| `0x000a0068` | `UC_REGION_SIZE` | Byte size of the Uncached Queue Region (4 KiB-aligned, `0x00301000` = 3,149,824 bytes) | **[HW]** |
| `0x000a0148` | `CPU_CS_H2XSOFTINTEN` | Bit 0 (`0x1`): Host-to-Xtensa software interrupt enable (`1` = enabled) | **[HW]** |
| `0x000a0150` | `CPU_IC_SOFTINT` | Bit 16 (`1 << 16` = `0x00010000`): `CPU_IC_SOFTINT_H2A_SHFT`; write `0x00010000` to ring the doorbell interrupt to VPU firmware after enqueuing a command | **[HW]** |
| `0x000a0160` | `CPU_CS_AHB_BRIDGE_SYNC_RESET` | Write `0x3`, then `0x2`, then `0x0` during power-off before disabling `MVS0_GDSC` | **[HW]** |
| `0x000a0168` | `CPU_CS_X2RPMH` | Write `0x0` during boot to mask RPMh handshake; write `0x3` during power-off | **[HW]** |
| `0x000b0000` | `WRAPPER_HW_VERSION` | Hardware version register: Bits `[31:28]` Major, Bits `[27:16]` Minor, Bits `[15:0]` Step. Reads `0x60100608` on SC8280XP | **[HW]** |
| `0x000b0004` | `WRAPPER_CLOCK_CONFIG` | Write `0x0` to enable wrapper clocks | **[HW]** |
| `0x000b000c` | `WRAPPER_INTR_STATUS` | Bit 2 (`0x4`): `WRAPPER_INTR_STATUS_A2H_BMSK` (HFI message interrupt); Bit 3 (`0x8`): `WRAPPER_INTR_STATUS_A2HWD_BMSK` (firmware watchdog bite) | **[HW]** |
| `0x000b0010` | `WRAPPER_INTR_MASK` | Interrupt mask (`0` = enabled, `1` = masked). Clear Bit 2 (`A2H`) and Bit 3 (`A2HWD`) to unmask HFI and watchdog interrupts | **[HW]** |
| `0x000b0024` | `WRAPPER_CPU_CLOCK_CONFIG` | Write `0x0` to enable VPU CPU clock | **[HW]** |
| `0x000b0050` | `WRAPPER_CORE_CLOCK_CONFIG` | Write `0x1` before NOC reset during power-off to park core clock | **[HW]** |
| `0x000b0078` | `WRAPPER_DEBUG_BRIDGE_LPI_CONTROL` | Low-Power Interface (LPI) bridge control | **[HW]** |
| `0x000b007c` | `WRAPPER_DEBUG_BRIDGE_LPI_STATUS` | LPI bridge status (polled until `0` during power-off) | **[HW]** |
| `0x000b0080` | `WRAPPER_IRIS_CPU_NOC_LPI_CONTROL` | Bit 0 (`0x1`): Request CPU NoC LPI handshake during power-off | **[HW]** |
| `0x000b0084` | `WRAPPER_IRIS_CPU_NOC_LPI_STATUS` | Bit 0 (`0x1`): Poll until `1` after requesting CPU NoC LPI | **[HW]** |
| `0x000b0188` | `WRAPPER_CORE_POWER_STATUS` | Bit 1 (`0x2`): `0` = VCODEC core domain (`MVS0`) is powered on, `1` = VCODEC core is power-collapsed | **[HW]** |
| `0x000b01a8` | `WRAPPER_IRIS_VCODEC_VPU_WRAPPER_SPARE_0` | Write `0x0` after switching `MVS0_GDSC` to hardware control mode | **[HW]** |
| `0x000c0010` | `WRAPPER_TZ_INTR_STATUS` | Bit 4 (`0x10`): TrustZone watchdog fault status | **[HW]** |
| `0x000c0034` | `WRAPPER_TZ_CTL_AXI_CLOCK_CONFIG` | Write `0x3` before resetting `MVS0C` during power-off | **[HW]** |
| `0x000c0038` | `WRAPPER_TZ_QNS4PDXFIFO_RESET` | Write `0x1` then `0x0` before resetting `MVS0C` during power-off | **[HW]** |
| `0x000e0004` | `AON_WRAPPER_MVP_NOC_LPI_CONTROL` | Bit 0 (`0x1`): Request MVP NoC LPI power-down handshake | **[HW]** |
| `0x000e0008` | `AON_WRAPPER_MVP_NOC_LPI_STATUS` | Bit 0 (`0x1`): Poll until `1` after requesting MVP NoC LPI | **[HW]** |

---

## 6. Exact Hardware Operational Sequences

### 6.1 VPU Power-On and Firmware Boot Sequence
- **[HW]** Execute the following exact sequence to power up the VPU and boot the HFI Gen2 firmware:
  1. **Vote Power Rails and Interconnects:**
     - Vote `mx.lvl` (`0x3001c`) and `mmcx.lvl` (`0x30020`) to the target OPP (e.g., `NOM` / `TURBO` for 533 MHz).
     - Vote `cpu-cfg` (`1000` kB/s) and `video-mem` (`>= 1000` kB/s) interconnects active.
  2. **Enable `MVS0C` Wrapper Domain and Clocks:**
     - Power on `MVS0C_GDSC` (`videocc + 0x0bf8`): clear bit 0, poll bit 31 until `1`, set bit 11 (`RETAIN_FF_ENABLE`).
     - Enable `GCC_VIDEO_AXI0_CLK` (`gcc + 0x28010`: set bit 0, poll bit 31 until `0`).
     - Configure and lock `video_pll0` (`videocc + 0x042c`) and `video_cc_mvs0_clk_src` (`videocc + 0x0b94`), set `VIDEO_CC_MVS0C_DIV_CDIVR` (`0x0c54`) = `1` and `VIDEO_CC_MVS0_DIV_CDIVR` (`0x0d54`) = `2`.
     - Enable `VIDEO_CC_MVS0C_CLK` (`videocc + 0x0c34`: set bit 0, poll bit 31 until `0`).
  3. **Enable `MVS0` Core Codec Domain and Clock:**
     - Power on `MVS0_GDSC` (`videocc + 0x0d18`) in software mode: clear bits 0 and 1, poll bit 31 until `1`, set bit 11 (`RETAIN_FF_ENABLE`).
     - Set parent clock rate on `VIDEO_CC_MVS0_CLK` (e.g., 240 MHz initial or 533 MHz target) and enable `VIDEO_CC_MVS0_CLK` (`videocc + 0x0d34`: set bit 0, poll bit 31 until `0`).
  4. **Authenticate and Reset Firmware via PAS 9:**
     - Allocate and initialize the 4 KiB-aligned HFI Queue Table + 3 queues (`0x301000` bytes) and 4 KiB SFR buffer in the IOVA range `0x25800000`–`0xdfffffff` (zero the SFR buffer and initialize the queue headers as specified in `02-hfi-gen2-protocol.md`).
     - Load `qcom/vpu/vpu20_p4_gen2_s6.mbn` into `0x8c600000` and execute PAS Init Image (`9`), PAS Mem Setup (`9, 0x8c600000, 0x700000`), and PAS Auth and Reset (`9`).
     - Invoke TrustZone `MP_VIDEO_VAR` (`0x0, 0x25800000, 0x01000000, 0x24800000`).
  5. **Program VPU Wrapper and Boot Registers:**
     - Write `0x0` to `WRAPPER_CLOCK_CONFIG` (`0x000b0004`).
     - Write `0x0` to `WRAPPER_CPU_CLOCK_CONFIG` (`0x000b0024`).
     - Unmask interrupts: read `WRAPPER_INTR_MASK` (`0x000b0010`), clear bit 2 (`A2H`) and bit 3 (`A2HWD`), and write back.
     - Write `0x1` to `CPU_CS_H2XSOFTINTEN` (`0x000a0148`).
     - Write `0x0` to `CPU_CS_X2RPMH` (`0x000a0168`).
     - Write the 32-bit IOVA of the queue buffer to `UC_REGION_ADDR` (`0x000a0064`) and its 4 KiB-aligned size (`0x00301000`) to `UC_REGION_SIZE` (`0x000a0068`).
     - Write the 32-bit IOVA of the queue table to `QTBL_ADDR` (`0x000a0054`) and write `0x1` to `QTBL_INFO` (`0x000a0050`).
     - Write `(sfr_iova + 1)` to `SFR_ADDR` (`0x000a005c`), where `+ 1` encodes `VIDEO_ARCH_LX = 1`.
     - Write `0x1` to `CTRL_INIT` (`0x000a0048`) to start the VPU processor.
  6. **Poll for Firmware Boot Completion:**
     - Poll `CTRL_STATUS` (`0x000a004c`) every 50–100 µs (up to 100 ms total):
       - If bit 0 (`CTRL_INIT_STATUS__M`) == `1`: firmware boot succeeded.
       - If bit 1 (`CTRL_ERROR_STATUS__M`) or bits `[7:1]` != `0`: firmware boot failed (read the null-terminated error string starting at offset `+4` of the SFR buffer).
  7. **Hand Off `MVS0_GDSC` to Hardware Control Mode:**
     - Set `HW_CONTROL` (bit 1 = `1`) in `MVS0_GDSC` (`videocc + 0x0d18`).
     - Write `0x0` to `WRAPPER_IRIS_VCODEC_VPU_WRAPPER_SPARE_0` (`0x000b01a8`).
  8. **Send `HFI_CMD_SYSTEM_INIT`:**
     - Enqueue the `HFI_CMD_SYSTEM_INIT` packet into the Command Queue and write `0x00010000` (`1 << 16`) to `CPU_IC_SOFTINT` (`0x000a0150`).

### 6.2 Interrupt Handling and Watchdog Detection Sequence
- **[HW]** On receiving GIC interrupt GSIV `206` (SPI `174`):
  1. Read `WRAPPER_INTR_STATUS` (`0x000b000c`).
  2. Write `0x1` to `CPU_CS_A2HSOFTINTCLR` (`0x000a001c`) to clear the firmware-to-host interrupt.
  3. Check for watchdog or fatal firmware faults:
     - Bit 3 (`0x8`) of `WRAPPER_INTR_STATUS` (`0x000b000c`) is set, OR
     - Bit 4 (`0x10`) of `WRAPPER_TZ_INTR_STATUS` (`0x000c0010`) is set, OR
     - Bits `[7:1]` (`0xfe`) of `CTRL_STATUS` (`0x000a004c`) are non-zero.
     - If any of these conditions hold, the VPU firmware has crashed. Read the 4 KiB SFR buffer (`u32 buf_size` at offset `0`, null-terminated ASCII crash reason at offset `4`) and transition the controller to the error recovery state.
  4. If Bit 2 (`0x4`) of `WRAPPER_INTR_STATUS` is set, drain all pending packets from the Message Queue and Debug Queue (see `02-hfi-gen2-protocol.md`).

### 6.3 VPU Power-Off Sequence
- **[HW]** Execute the following sequence to cleanly power down the VPU without hanging the NoC:
  1. Poll `CPU_CS_SCIACMDARG3` (`0x000a0058`) until bits `[5:3]` (`0x38`, `INIT_IDLE_MSG_MASK`) are non-zero (timeout 15 ms; continue even on timeout).
  2. **Power Off VCODEC Hardware (`MVS0` Domain):**
     - Clear `HW_CONTROL` (bit 1 = `0`) in `MVS0_GDSC` (`videocc + 0x0d18`), wait 1 µs, and poll bit 31 (`PWR_ON`) until `1`.
     - Read `WRAPPER_CORE_POWER_STATUS` (`0x000b0188`). If bit 1 (`0x2`) is `0` (meaning the core is still powered on):
       - Write `0x1` to `WRAPPER_CORE_CLOCK_CONFIG` (`0x000b0050`).
       - Read `VCODEC_NOC_ERL_MAIN_SWID_LOW` (`0x0000115c`).
       - Write `0x3` to `AON_WRAPPER_MVP_NOC_RESET_REQ` (`0x0001f000`).
       - Poll `AON_WRAPPER_MVP_NOC_RESET_ACK` (`0x0001f004`) until bits `[1:0]` equal `0x3` (timeout 1000 µs).
       - Write `0x0` to `AON_WRAPPER_MVP_NOC_RESET_REQ` (`0x0001f000`).
       - Poll `AON_WRAPPER_MVP_NOC_RESET_ACK` (`0x0001f004`) until it equals `0x0` (timeout 1000 µs).
     - Write `0x3`, then `0x2`, then `0x0` to `CPU_CS_AHB_BRIDGE_SYNC_RESET` (`0x000a0160`).
     - Disable `VIDEO_CC_MVS0_CLK` (`videocc + 0x0d34` bit 0 = `0`).
     - Power off `MVS0_GDSC` (`videocc + 0x0d18`: set `SW_COLLAPSE` bit 0 = `1`, poll `CFG_GDSCR` `0x0d1c` bit 16 until `1`).
  3. **Power Off Controller (`MVS0C` Domain):**
     - Write `0x3` to `CPU_CS_X2RPMH` (`0x000a0168`).
     - Set bit 0 (`0x1`) in `AON_WRAPPER_MVP_NOC_LPI_CONTROL` (`0x000e0004`) and poll `AON_WRAPPER_MVP_NOC_LPI_STATUS` (`0x000e0008`) bit 0 until `1` (timeout 1000 µs).
     - Set bit 0 (`0x1`) in `WRAPPER_IRIS_CPU_NOC_LPI_CONTROL` (`0x000b0080`) and poll `WRAPPER_IRIS_CPU_NOC_LPI_STATUS` (`0x000b0084`) bit 0 until `1` (timeout 1000 µs).
     - Write `0x0` to `WRAPPER_DEBUG_BRIDGE_LPI_CONTROL` (`0x000b0078`) and poll `WRAPPER_DEBUG_BRIDGE_LPI_STATUS` (`0x000b007c`) until `0x0` (timeout 1000 µs).
     - Write `0x3` to `WRAPPER_TZ_CTL_AXI_CLOCK_CONFIG` (`0x000c0034`).
     - Write `0x1` then `0x0` to `WRAPPER_TZ_QNS4PDXFIFO_RESET` (`0x000c0038`).
     - Assert and deassert `bus` reset (`GCC_VIDEO_AXI0_CLK_ARES` bit 2 of `gcc + 0x28010`) and `core` reset (`VIDEO_CC_MVS0C_CLK_ARES` bit 2 of `videocc + 0x0c34`, holding asserted for at least 400 µs before deasserting).
     - Disable `VIDEO_CC_MVS0C_CLK` (`videocc + 0x0c34` bit 0 = `0`) and `GCC_VIDEO_AXI0_CLK` (`gcc + 0x28010` bit 0 = `0`).
     - Power off `MVS0C_GDSC` (`videocc + 0x0bf8`: set `SW_COLLAPSE` bit 0 = `1`).
     - Vote `video-mem` and `cpu-cfg` interconnects to `0` kB/s and release this driver's `mx.lvl` and `mmcx.lvl` votes (ensuring the system-wide aggregator preserves any active `MMCX` vote held by `MDSS0`).
