# Adreno 690 Platform Memory Map, GPUCC, and GPU SMMU

## 1. Platform Memory Map and ACPI Resource Layout

**[ABI]** On the Radxa Dragon Q8B, the ACPI DSDT node `\_SB.GPU0` (`_HID = "QCOM0636"`) declares 7 memory resources and 4 interrupt resources. Several memory resources overlap or alias sub-regions of larger blocks.

**[ABI]** The complete ACPI `\_SB.GPU0` memory resource table is:

| Resource Index | Physical Base | Size | Hardware Function | Overlap / Aliasing Notes |
|---|---|---|---|---|
| **0** | `0x03d00000` | `0x00060000` (384 KiB) | Adreno 690 3D Core (`GFX3D`) MMIO | Main GPU register space (dwords `0x0000`–`0x17fff`). |
| **1** | `0x00780134` | `0x0000000c` (12 B) | QFPROM Speed-Bin Fuse | Fuse row containing GPU speed-bin bits. |
| **2** | `0x00784000` | `0x00000918` (~2.3 KiB) | QFPROM Corrected Fuse Block | Secondary QFPROM fuse region. |
| **3** | `0x03d60000` | `0x00040000` (256 KiB) | Combined GMU + CX + GPUCC Window | Contains GMU registers at `+0x0a000` (`0x03d6a000`, size `0x34000`) and **overlaps Resource 6 (`GPUCC` at `0x03d90000`, offset `+0x30000`)**. |
| **4** | `0x0b290000` | `0x00010000` (64 KiB) | APSS Shared / GMU PDC Auxiliary | System-level power control window. |
| **5** | `0x0b490000` | `0x00010000` (64 KiB) | PDC (Power Domain Controller) | RPMh PDC sequence/enable registers for GPU GX/CX rails. |
| **6** | `0x03d90000` | `0x00009000` (36 KiB) | GPU Clock Controller (`GPUCC`) | Overlaps the upper 64 KiB of Resource 3 (`0x03d60000 + 0x30000`). |

**[HW]** In addition to the ACPI `\_SB.GPU0` resources, the GPU subsystem depends on the following fixed physical blocks on SC8280XP:

| Block | Physical Base | Size | Description |
|---|---|---|---|
| **GMU MMIO Sub-Window** | `0x03d6a000` | `0x00034000` (208 KiB) | Offset `+0x0a000` inside Resource 3 (`0x03d60000`). Note: Hardware register definitions for the GMU are often specified as 32-bit dword offsets from `0x03d00000` (starting at dword `0x1a800`). To convert a dword offset relative to `0x03d00000` into a byte offset from `0x03d6a000`, compute `(dword_offset * 4) - 0x6a000`. |
| **GMU RSCC** | `0x03d8c000` | `0x00004000` (16 KiB) | Offset `+0x2c000` inside Resource 3 (`0x03d60000`), or `+0x22000` from GMU base `0x03d6a000` (dword offset `0x23000` from `0x03d00000`). |
| **GPU SMMUv2** | `0x03da0000` | `0x00020000` (128 KiB) | Dedicated ARM SMMUv2 (`qcom,sc8280xp-smmu-500` / `qcom,adreno-smmu`) for GPU 3D and GMU DMA. |

**[POLICY]** Because Resource 3 (`0x03d60000`–`0x03d9ffff`) encompasses both the GMU (`0x03d6a000`), GMU RSCC (`0x03d8c000`), and GPUCC (`0x03d90000`–`0x03d98fff`), an operating system that enforces exclusive physical resource reservation will fail to allocate Resource 6 if Resource 3 is already claimed. A driver must either share the mapping of Resource 3 (accessing GMU at `+0x0a000` and GPUCC at `+0x30000`) or split the resource reservation into non-overlapping sub-ranges.

---

## 2. Interrupt Routing (GICv3 GSIVs)

**[HW]** The Adreno 690 and its GMU use three primary GICv3 SPIs (Shared Peripheral Interrupts), all active-high level-triggered:

| Interrupt Role | GICv3 GSIV | GIC SPI Number | ACPI `\_SB.GPU0` Resource Index | Description |
|---|---|---|---|---|
| **GPU 3D IRQ (`RBBM_INT_0`)** | **332** (`0x14c`) | SPI 300 | Index 3 | Fires on CP ringbuffer completion (`CP_RB_DONE_TS`), cache flush, AHB error, hang detect, or UCHE out-of-bounds access. |
| **GMU HFI IRQ** | **336** (`0x150`) | SPI 304 | *Omitted from ACPI* | Fires when the GMU firmware posts messages to HFI queues or raises `GMU_AO_HOST_INTERRUPT_STATUS`. |
| **GMU AO / Fault IRQ** | **337** (`0x151`) | SPI 305 | *Omitted from ACPI* | Fires on GMU watchdog timeout (`WDOG_BITE`), AHB bus error, or FENCE error. |

**[ABI]** Note that ACPI `\_SB.GPU0` lists four GSIVs at indices 0–3: `605` (Index 0), `606` (Index 1), `240` (Index 2), and `332` (Index 3). Only Index 3 (`GSIV 332`) is the 3D GPU interrupt; the GMU HFI interrupt (`GSIV 336`) and GMU fault interrupt (`GSIV 337`) are not listed in the ACPI `\_SB.GPU0` CRS table and must be registered directly by their hardware GSIV numbers `336` and `337`.

---

## 3. GPU Clock Controller (`GPUCC` at `0x03d90000`)

### 3.1 Power Domains (GDSCs)

**[HW]** The GPU subsystem contains two Global Distributed Switch Controllers (GDSCs) inside `GPUCC` (`0x03d90000`). Both GDSCs **must** be powered on before accessing GMU (`0x03d6a000`) or GPU 3D (`0x03d00000`) registers; reading `0x03d00000` while `CX_GDSC` or `GX_GDSC` is off causes an immediate synchronous External Abort (bus hang/panic).

| GDSC Name | `GDSCR` Offset | `CFG_GDSCR` Offset | Wait Field Values (`EN_REST`, `EN_FEW`, `CLK_DIS`) | Hardware Flags & Behavior |
|---|---|---|---|---|
| **`gpu_cc_cx_gdsc`** | `+0x106c` | `+0x1070` | `EN_REST=2`, `EN_FEW=2`, `CLK_DIS=8` | Supports `ON` and `OFF` states; retains flip-flops (`RETAIN_FF_ENABLE`, Bit 11). Controls the CX (Always-On / GMU / Cnoc) logic domain. |
| **`gpu_cc_gx_gdsc`** | `+0x100c` | `+0x1010` | `EN_REST=2`, `EN_FEW=2`, `CLK_DIS=0` (`0x220` in bits 23:12 if configured) | Clamps I/O (`CLAMP_IO`), AON reset (`AON_RESET`), retains flip-flops (`RETAIN_FF_ENABLE`). Controls the 3D shader/GX domain. |

**[HW]** Each `GDSCR` register has the following bit layout:

| Bits | Field Name | Description |
|---|---|---|
| 31 | `PWR_ON` | Read-only status bit: `1` = GDSC power switch is ON; `0` = OFF. |
| 29 | `EN_ALL` | Power-on all memory/logic arrays. |
| 23:20 | `CLK_DIS_WAIT` | Cycles to wait after disabling clocks before asserting power-down. |
| 19:16 | `EN_FEW_WAIT` | Cycles to wait during initial power-switch turn-on. |
| 15:12 | `EN_REST_WAIT` | Cycles to wait during remaining power-switch turn-on. |
| 11 | `RETAIN_FF_ENABLE` | Set to `1` after power-on to retain register flip-flop state across power transitions. |
| 1 | `HW_CONTROL` | `0` = Software control; `1` = Hardware (GMU) control. |
| 0 | `SW_COLLAPSE` | `0` = Request power ON; `1` = Request power collapse (OFF). |

**[HW]** In the companion `CFG_GDSCR` register (`GDSCR + 0x04`), Bit 31 (`GDSC_POWER_UP_COMPLETE`) and Bit 30 (`GDSC_POWER_DOWN_COMPLETE`) report state-machine completion when `PWR_ON` polling is supplemented by `CFG_GDSCR`.

**[HW]** **Empirical Radxa Dragon Q8B GDSC Bring-Up Sequence**:
1. Enable the `GPU_OXILI_CX` RPMh power rail (or ensure UEFI left CX rail active).
2. Read `GPUCC + 0x106c` (`gpu_cc_cx_gdsc`). Clear Bit 0 (`SW_COLLAPSE = 0`) and write back. Poll until Bit 31 (`PWR_ON`) reads `1` (empirically `0x80228000` -> `0x80228800` once `RETAIN_FF_ENABLE` Bit 11 is set).
3. Read `GPUCC + 0x100c` (`gpu_cc_gx_gdsc`). If `0x80220001` (`SW_COLLAPSE=1`, `PWR_ON=1` stuck from boot state), clear Bit 0 (`SW_COLLAPSE = 0`), set Bit 11 (`RETAIN_FF_ENABLE = 1`), and poll until Bit 31 (`PWR_ON`) reads `1` (empirically `0x80220800`).

### 3.2 Lucid 5LPE Alpha PLLs (`gpu_cc_pll0` and `gpu_cc_pll1`)

**[HW]** `GPUCC` contains two Lucid 5LPE (`CLK_ALPHA_PLL_TYPE_LUCID_5LPE`) PLLs:
- **`gpu_cc_pll0`** at base offset `+0x0000`:
  - `L_VAL = 0x001c` (28), `ALPHA_VAL = 0x5555` (`1/3`), `CONFIG_CTL_VAL = 0x20485699`, `CONFIG_CTL_U_VAL = 0x00002261`, `CONFIG_CTL_U1_VAL = 0x029a299c`, `TEST_CTL_VAL = 0x00000000`, `TEST_CTL_U_VAL = 0x00000000`, `TEST_CTL_U1_VAL = 0x00000000`, `USER_CTL_VAL = 0x00000001`, `USER_CTL_U_VAL = 0x00000805`, `USER_CTL_U1_VAL = 0x00000000`.
- **`gpu_cc_pll1`** at base offset `+0x0100`:
  - `L_VAL = 0x0034` (52), `ALPHA_VAL = 0x1555` (`1/12`), `CONFIG_CTL_VAL = 0x20485699`, `CONFIG_CTL_U_VAL = 0x00002261`, `CONFIG_CTL_U1_VAL = 0x029a299c`, `USER_CTL_VAL = 0x00000001`, `USER_CTL_U_VAL = 0x00000805`, `USER_CTL_U1_VAL = 0x00000000`. Default output rate: `19.2 MHz * (52 + 1/12) = 1000 MHz` (`500 MHz` after even divider).

**[HW]** Each Lucid 5LPE PLL uses the following register offsets relative to its PLL base (`+0x0000` or `+0x0100`):

| Offset | Register Name | Description |
|---|---|---|
| `+0x00` | `PLL_MODE` | Bit 31 = `PLL_LOCK_DET`, Bit 29 = `PLL_ACTIVE_FLAG`, Bit 22 = `PCAL_NOT_DONE`, Bit 20 = `PLL_LATCH_INPUT`, Bit 19 = `PLL_ACK_LATCH`, Bits 18:14 = `PLL_OPMODE` (`1` = RUN), Bit 2 = `PLL_RESET_N`, Bit 0 = `PLL_OUTCTRL`. |
| `+0x04` | `PLL_L_VAL` | Integer multiplier `L`. |
| `+0x08` | `PLL_CAL_L_VAL` | Calibration `L` value (`0x44`). |
| `+0x0c` | `PLL_USER_CTL` | Output enables and post-dividers (Bit 0 = `PLL_OUT_MAIN`, Bit 1 = `PLL_OUT_EVEN`). |
| `+0x10` | `PLL_USER_CTL_U` | Upper user control (`0x00000805`). |
| `+0x14` | `PLL_USER_CTL_U1` | Extended user control (`0x00000000`). |
| `+0x18` | `PLL_CONFIG_CTL` | Loop filter config (`0x20485699`). |
| `+0x1c` | `PLL_CONFIG_CTL_U` | Upper config (`0x00002261`). |
| `+0x20` | `PLL_CONFIG_CTL_U1` | Extended config (`0x029a299c`). |
| `+0x24` | `PLL_TEST_CTL` | Test control. |
| `+0x28` | `PLL_TEST_CTL_U` | Upper test control. |
| `+0x2c` | `PLL_TEST_CTL_U1` | Extended test control. |
| `+0x30` | `PLL_STATUS` | Status register. |
| `+0x40` | `PLL_ALPHA_VAL` | 16-bit fractional multiplier `Alpha` (`rate = ref * (L + Alpha / 2^16)`). |

### 3.3 Root Clock Generators (RCGs)

**[HW]** `GPUCC` defines five Root Clock Generators (RCGs). Each RCG consists of a `CMD_RCGR` register at `+0x00` (Bit 0 = `CMD_UPDATE`, Bit 1 = `ROOT_EN`, Bit 31 = `ROOT_OFF`) and a `CFG_RCGR` register at `+0x04` (Bits 4:0 = `SRC_DIV` where `divider = (SRC_DIV + 1) / 2`, Bits 10:8 = `SRC_SEL`).

| RCG Name | `CMD_RCGR` Offset | `CFG_RCGR` Offset | Parent Map (`SRC_SEL` value -> Source) | Frequency Table `(Hz, Parent, PreDiv)` |
|---|---|---|---|---|
| **`gpu_cc_gmu_clk_src`** | `+0x1120` | `+0x1124` | `0` = `bi_tcxo` (`19.2 MHz`)<br>`1` = `gpu_cc_pll0_out_even`<br>`2` = `gpu_cc_pll1_out_even`<br>`5` = `gcc_gpu_gpll0_clk_src` (`600 MHz`)<br>`6` = `gcc_gpu_gpll0_div_clk_src` (`300 MHz`) | `19.2 MHz`: `bi_tcxo`, div `1` (`SRC_DIV=0`)<br>`200 MHz`: `gpu_cc_pll1_out_even`, div `2.5` (`SRC_DIV=4`)<br>`500 MHz`: `gpu_cc_pll1_out_even`, div `1` (`SRC_DIV=0`) |
| **`gpu_cc_hub_clk_src`** | `+0x11bc` | `+0x11c0` | Same as above | `150 MHz`: `gcc_gpu_gpll0_clk_src`, div `4` (`SRC_DIV=7`)<br>`240 MHz`: `gpu_cc_pll0_out_even`, div `2.5` (`SRC_DIV=4`)<br>`300 MHz`: `gcc_gpu_gpll0_clk_src`, div `2` (`SRC_DIV=3`) |
| **`gpu_cc_xo_clk_src`** | `+0x1010` | `+0x1014` | `0` = `bi_tcxo` (`19.2 MHz`) | `19.2 MHz`: `bi_tcxo`, div `1` (`SRC_DIV=0`) |
| **`gpu_cc_XO_div_clk_src`** | `+0x102c` (Regdiv) | — | Fixed regmap divider from `gpu_cc_xo_clk_src` | Bits 1:0 (`width=2`) at `+0x102c` |
| **`gpu_cc_hub_cx_int_div_clk_src`** | `+0x11ec` (Regdiv) | — | Fixed regmap divider from `gpu_cc_hub_clk_src` | Bits 1:0 (`width=2`) at `+0x11ec` |
| **`gpu_cc_hub_ahb_div_clk_src`** | `+0x11f0` (Regdiv) | — | Fixed regmap divider from `gpu_cc_hub_clk_src` | Bits 1:0 (`width=2`) at `+0x11f0` |

### 3.4 Branch Clocks (CBCRs) and Reset Maps

**[HW]** Each branch clock control register (`CBCR`) enables its clock via Bit 0 (`CLK_ENABLE = 1`) and reports halt status in Bits 31:28 (`CLK_OFF` is Bit 31: `0` = clock is running, `1` = clock is halted).

| Branch Clock Name | `CBCR` Offset | Halt Check Type | Parent Clock | Notes |
|---|---|---|---|---|
| `gpu_cc_ahb_clk` | `+0x1078` | `DELAY` (no poll) | `gpu_cc_hub_ahb_div_clk_src` | Required for AHB register access to GPU/GMU. |
| `gpu_cc_crc_ahb_clk` | `+0x107c` | `VOTED` | `gpu_cc_hub_ahb_div_clk_src` | CRC AHB branch. |
| `gpu_cc_cx_apb_clk` | `+0x1088` | `VOTED` | — | CX APB branch. |
| `gpu_cc_cx_gmu_clk` | `+0x1098` | `HALT` (poll Bit 31=0) | `gpu_cc_gmu_clk_src` | **Core GMU Cortex-M3 clock** (`200 MHz` or `500 MHz`). |
| `gpu_cc_cx_snoc_dvm_clk` | `+0x108c` | `VOTED` | — | SNOC DVM interface clock. |
| `gpu_cc_cxo_aon_clk` | `+0x1004` | `VOTED` | `gpu_cc_xo_clk_src` | Always-on 19.2 MHz XO clock. |
| `gpu_cc_cxo_clk` | `+0x109c` | `HALT` | `gpu_cc_xo_clk_src` | CX 19.2 MHz XO branch (kept always enabled). |
| `gpu_cc_demet_clk` | `+0x100c` (en) / `+0x10ec` (halt) | `HALT` | `gpu_cc_XO_div_clk_src` | Enabled via Bit 1 of `+0x100c` (`GPUCC_CX_GDSCR` / `DEMET`). |
| `gpu_cc_freq_measure_clk` | `+0x1008` | `HALT` | `gpu_cc_XO_div_clk_src` | Frequency measurement clock. |
| `gpu_cc_gx_gmu_clk` | `+0x1064` | `HALT` | `gpu_cc_gmu_clk_src` | GX-domain GMU clock. |
| `gpu_cc_gx_vsense_clk` | `+0x105c` | `VOTED` | — | Voltage sense clock. |
| `gpu_cc_hlos1_vote_gpu_smmu_clk` | `+0x5000` | `VOTED` | — | **Must be enabled for GPU SMMUv2 (`0x03da0000`) access**. |
| `gpu_cc_hub_aon_clk` | `+0x11e8` | `VOTED` | `gpu_cc_hub_clk_src` | Hub Always-On branch. |
| `gpu_cc_hub_cx_int_clk` | `+0x1204` | `HALT` | `gpu_cc_hub_cx_int_div_clk_src` | Hub CX internal clock (set `CLK_ENABLE` at probe). |
| `gpu_cc_mnd1x_0_gfx3d_clk` | `+0x802c` | `HALT` | — | MND1X 0 GFX3D branch. |
| `gpu_cc_mnd1x_1_gfx3d_clk` | `+0x8030` | `HALT` | — | MND1X 1 GFX3D branch. |
| `gpu_cc_sleep_clk` | `+0x1090` | `VOTED` | — | 32 kHz sleep clock branch. |

**[HW]** During `GPUCC` initialization, software must unconditionally set Bit 0 (`CLK_ENABLE`) in `+0x109c` (`gpu_cc_cxo_clk`) and `+0x1204` (`gpu_cc_hub_cx_int_clk`) to keep the CXO and Hub CX internal clocks always running.

**[HW]** `GPUCC` Block Reset Registers (`BCR`, Bit 0 = `BLK_ARES`):
- `GPUCC_GPU_CC_ACD_BCR`: `+0x1160`
- `GPUCC_GPU_CC_CX_BCR`: `+0x1068`
- `GPUCC_GPU_CC_FAST_HUB_BCR`: `+0x11e4`
- `GPUCC_GPU_CC_FF_BCR`: `+0x1200`
- `GPUCC_GPU_CC_GFX3D_AON_BCR`: `+0x10a0`
- `GPUCC_GPU_CC_GMU_BCR`: `+0x111c`
- `GPUCC_GPU_CC_GX_BCR`: `+0x1008`
- `GPUCC_GPU_CC_XO_BCR`: `+0x1000`

---

## 4. GPU ARM SMMUv2 (`0x03da0000`)

### 4.1 SMMU Architecture and Clock/Power Prerequisites

**[HW]** The Adreno 690 has a dedicated ARM SMMUv2 (MMU-500 derivative, `qcom,sc8280xp-smmu-500` / `qcom,adreno-smmu`) at physical base `0x03da0000` (size `0x20000`, `128 KiB`). Before reading or writing any register in `0x03da0000`:
1. `gpu_cc_cx_gdsc` (`0x03d9106c`) must be powered on.
2. `gpu_cc_hlos1_vote_gpu_smmu_clk` (`0x03d95000`) and `gpu_cc_ahb_clk` (`0x03d91078`) must be enabled.

**[HW]** Empirical `SMMU_IDR*` parameters read from `0x03da0000` on the Radxa Dragon Q8B:
- `SMMU_IDR0` (`+0x0020`) = `0x16013230`: Stage-1 and Stage-2 translation supported, 48 Stream Matching Registers (`NUMSMRG = 48`).
- `SMMU_IDR1` (`+0x0024`) = `0x30000108`:
  - `PAGESIZE` (Bit 31) = `0` (**4 KiB page granule**: Global Space 0 at `+0x0000`, Global Space 1 at `+0x1000`, Context Bank `n` at `+0x8000 + n * 0x1000`).
  - `NUMPAGENDXB` (Bits 30:28) = `3` (`2^(3+1) = 16` pages in Global/CB space; CB base offset is `16 * 0x1000 / 2 = 0x8000`).
  - `NUMCB` (Bits 7:0) = `8` (**8 Context Banks**, `CB0`–`CB7`, located at `+0x8000` through `+0xF000`).
- `SMMU_IDR2` (`+0x0028`) = `0x00003445`: 48-bit IAS/OAS/UBS (`aarch64` 48-bit page tables supported).

### 4.2 Stream IDs (`SMR`) and Context Bank Assignment (`S2CR`)

**[HW]** On SC8280XP, the Adreno 690 3D engine and GMU issue DMA transactions with 8 distinct 15-bit Stream IDs (`0x00, 0x01, 0x04, 0x08, 0x09, 0x0c, 0x0e, 0x0f`) and mask `0x0400` (or `0x0000` exact match).

**[HW]**Empirical boot state on the Radxa Dragon Q8B shows how firmware pre-allocates the Stream Match Registers (`SMMU_SMRn` at `+0x0800 + n * 4`, Bit 31 = `VALID`, Bits 30:16 = `MASK`, Bits 14:0 = `ID`) and Stream-to-Context Registers (`SMMU_S2CRn` at `+0x0c00 + n * 4`, Bits 17:16 = `TYPE` where `0` = Translation Context Bank, Bits 7:0 = `CBNDX`):

| Context Bank | Offset from `0x03da0000` | Assigned Stream IDs | Function |
|---|---|---|---|
| **Context Bank 0 (`CB0`)** | `+0x8000` (`0x03da8000`) | `0x00, 0x08, 0x09, 0x0c, 0x0e, 0x0f` (and `0x04`) | **Adreno 690 3D GPU (`GFX3D`)**: Translates ringbuffers, SQE microcode, shadow RPTR/WPTR, and all user GEM buffer objects. Supports split page tables (`TTBR0` for per-context user buffers and `TTBR1` for global kernel/ringbuffer mappings) or unified `TTBR0`. |
| **Context Bank 1 (`CB1`)** | `+0x9000` (`0x03da9000`) | `0x01` (`SMR0 = 0x84000001`, mask `0x0400`) | **Graphics Management Unit (`GMU`)**: Translates GMU Cortex-M3 accesses to host system RAM in the `0x20000000`–`0x20040000` IOVA window (HFI queues, boot image, DUMPTCM, and ICACHE). |

### 4.3 Context Bank Register Programming (`AArch64` Stage-1 4 KiB Page Tables)

**[HW]** Each Context Bank `n` (`0x03da8000 + n * 0x1000`) uses the following register layout:

| Offset in CB | Register Name | Required Configuration |
|---|---|---|
| `+0x000` | `CBn_SCTLR` | Bit 0 = `M` (MMU Enable = `1`), Bit 4 = `E` (`0` = Little Endian), Bit 5 = `CFRE` (`1` = Return abort on context fault), Bit 6 = `CFIE` (`1` = Context fault interrupt enable), Bit 7 = `CFCFG` (`0` = Terminate, or `1` = Stall on fault for `CB0` devcoredump). Empirical working value: `0x000000e1`. |
| `+0x020` | `CBn_TTBR0` (64-bit) | Physical base address of Level-0 (or Level-1) 4 KiB translation table (`Bits 47:12`), plus `ASID` in `Bits 63:48`. |
| `+0x028` | `CBn_TTBR1` (64-bit) | Physical base address of upper-half translation table (if `SEP` is enabled in `TCR`). |
| `+0x030` | `CBn_TCR` | `T0SZ` (Bits 5:0) = `16` (48-bit IOVA) or `32` (32-bit IOVA for GMU `CB1`); `IRGN0` (Bits 9:8) = `1` (WB-WA); `ORGN0` (Bits 11:10) = `1` (WB-WA); `SH0` (Bits 13:12) = `3` (Inner Shareable); `TG0` (Bits 15:14) = `0` (4 KiB granule). |
| `+0x034` | `CBn_TCR2` | `PASIZE` (Bits 2:0) = `5` (48-bit physical address); `AS` (Bit 4) = `1` (16-bit ASID); `SEP` (Bits 17:15) = `7` (sign-bit split if using `TTBR1`). |
| `+0x038` | `CBn_MAIR0` | Memory Attribute Indirection Register 0 (`Attr0 = 0x00` Device-nGnRnE, `Attr1 = 0x44` Normal Non-Cacheable, `Attr2 = 0xff` Normal Write-Back Read/Write-Allocate). |
| `+0x03c` | `CBn_MAIR1` | Memory Attribute Indirection Register 1. |
| `+0x058` | `CBn_FSR` | Fault Status Register (write `0xffffffff` to clear sticky faults). Bit 31 = `MULTI`, Bit 30 = `SS` (Stalled), Bit 3 = `PF` (Permission), Bit 2 = `AFF` (Access Flag), Bit 1 = `TF` (Translation Fault). |
| `+0x060` | `CBn_FAR` (64-bit) | Faulting IOVA address when `CBn_FSR != 0`. |
| `+0x068` | `CBn_FSYNR0` | Fault syndrome (Bit 4 = `WNR` Write-Not-Read, Bits 3:0 = Walk stage). |
| `+0x070` | `CBn_RESUME` | Write `0x00000001` (`TNR` = Terminate) to resume a stalled transaction when `CFCFG=1` and `RBBM_STATUS3.SMMU_STALLED_ON_FAULT` is set. |
| `+0x600` | `CBn_TLBIASID` | Write ASID to invalidate TLB entries for this context bank. |
| `+0x7f0` | `CBn_TLBSYNC` | Write `0` to synchronize TLB invalidation. |
| `+0x7f4` | `CBn_TLBSTATUS` | Poll until Bit 0 (`SACTIVE`) clears to `0`. |

**[HW]** In Global Space 0 (`0x03da0000`):
- **`SMMU_sCR0` (`+0x0000`)**: Bit 0 (`CLIENTPD`) must be `0` to enable translation (`0x00200912` empirical: `CLIENTPD=0`, `GFRE=1`, `GFIE=1`, `STALLD=0`, `USFCFG=1`).
- **`SMMU_CBA2Rn` (`+0x1800 + n * 4`)**: Bit 0 (`VA64`) **must be set to `1`** for both `CBA2R0` (`+0x1800`) and `CBA2R1` (`+0x1804`) to select AArch64 Long-Descriptor page table format (`1` = AArch64, `0` = AArch32).
- **`SMMU_sTLBGSTATUS` (`+0x0074`) / `SMMU_sTLBGSYNC` (`+0x0070`) / `SMMU_STLBIALL` (`+0x0060`)**: Global TLB invalidation and sync registers.
