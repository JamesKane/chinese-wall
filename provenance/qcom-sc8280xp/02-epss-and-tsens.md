# Provenance: `02-epss-and-tsens.md`

- **Baseline commits**:
  - `mainline d135ab453a7400a34930bdda1ef4bdc8acfac80e`
  - `radxa/kernel@c00e1e17e65104c8143a8f3d559ef723ae5824d3` (`linux-7.0.11`)
  - `TC956X/TC9564_Host_Driver@3eca71ce0c5708c92e20f14b8e858cd4483d58f3`
- **Extracted date**: `2026-10-27`

## Source Files Read

1. **`/home/jkane/Projects/OS/linux/drivers/cpufreq/qcom-cpufreq-hw.c` (lines 1–781)**
   - Extracted `qcom_cpufreq_soc_data_epss` register offset layout (`reg_enable = 0x0`, `reg_domain_state = 0x328`, `reg_dcvs_ctrl = 0xb0`, `reg_freq_lut = 0x100`, `reg_volt_lut = 0x200`, `reg_intr_clr = 0xb4`, `reg_current_vote = 0x320`, `reg_perf_state = 0x020`, `lut_row_size = 4`), LUT bit masks (`LUT_SRC` bit 30 or `[15:12]`, `LUT_L_VAL` `GENMASK(7, 0)` / `[11:0]`, `LUT_CORE_COUNT` `GENMASK(18, 16)`, `LUT_VOLT` `GENMASK(11, 0)`), `LUT_MAX_ENTRIES = 40`, duplicate-entry termination check, and `REG_CURRENT_VOTE` multiplier field (`GENMASK(19, 12)`).
2. **`/home/jkane/Projects/OS/linux/drivers/thermal/qcom/tsens-v2.c` (lines 1–317)**
   - Extracted TSENS v2 SROT and TM register offsets (`SROT_HW_VER_OFF = 0x0`, `SROT_CTRL_OFF = 0x4`, `TM_INT_EN_OFF = 0x0000`, `TM_UPPER_LOWER_INT_STATUS_OFF = 0x0004`, `TM_UPPER_LOWER_INT_CLEAR_OFF = 0x0008`, `TM_UPPER_LOWER_INT_MASK_OFF = 0x000c`, `TM_CRITICAL_INT_STATUS_OFF = 0x0010`, `TM_CRITICAL_INT_CLEAR_OFF = 0x0014`, `TM_CRITICAL_INT_MASK_OFF = 0x0018`, `TM_Sn_UPPER_LOWER_THRESHOLD_OFF = 0x0020`, `TM_Sn_CRITICAL_THRESHOLD_OFF = 0x0060`, `TM_Sn_STATUS_OFF = 0x00a0`, `TM_TRDY_OFF = 0x00e4`) and `feat_tsens_v2` (`VER_2_X`, `crit_irqs = 1`, `max_sensors = 16`, `trip_min_temp = -40000`, `trip_max_temp = 120000`).
3. **`/home/jkane/Projects/OS/linux/drivers/thermal/qcom/tsens.c` (lines 1–1300) and `/home/jkane/Projects/OS/linux/drivers/thermal/qcom/tsens.h` (lines 1–685)**
   - Extracted `TM_Sn_STATUS` bitfields (`valid_st` bit 21, `crit_st` bit 20, `upper_st` bit 19, `lower_st` bit 18, `last_temp` `[11:0]`), 12-bit two's-complement sign extension (`sign_extend32(last_temp, 11)`), deci-Celsius (`0.1 °C`) to millidegree Celsius (`* 100`) conversion when `CODE_OR_TEMP == 1`, and retry-on-invalid (`VALID == 0`) polling logic.
4. **Empirical Board Documentation and Dumps (`/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/`)**:
   - `power-thermal-idle.md` (lines 1–254): Extracted EPSS cluster topology (`epss0` `0x18591000` CPUs 0–3, `epss1` `0x18592000` CPUs 4–7) and TSENS v2 controller summary.
   - `dumps/freebsd/qcpeek-out.txt` (lines 1–139): Extracted exact empirical 40-entry `FREQ_LUT` (`0x100`–`0x19C`) and `VOLT_LUT` (`0x200`–`0x29C`) register values for both `epss0` and `epss1`, plus live SROT and TM register dumps for `tsens0` (`0x0C263000`/`0x0C222000`), `tsens1` (`0x0C265000`/`0x0C223000`), `tsens2` (`0x0C251000`/`0x0C224000`), and `tsens3` (`0x0C252000`/`0x0C225000`).
   - `dumps/dt/live.dts` (lines 148–158, 758–768, 3970–3980, 4359–4369, 7457–8305): Extracted exact sensor counts (`14`, `16`, `11`, `5`), PDC interrupts, and thermal-zone sensor mappings and trip points.
