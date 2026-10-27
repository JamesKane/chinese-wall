# Provenance: `specs/qcom-sc8280xp/04-smem-ipcc-glink-usbc.md`

## Baseline Commits & Date
- **Mainline Linux commit**: `d135ab453a7400a34930bdda1ef4bdc8acfac80e` (`/home/jkane/Projects/OS/linux`)
- **Radxa vendor kernel commit**: `radxa/kernel@c00e1e17e65104c8143a8f3d559ef723ae5824d3` (`linux-7.0.11`)
- **Empirical board documentation**: `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/`
- **Extracted date**: `2026-10-27`

## Source Files Read & Facts Extracted

1. `/home/jkane/Projects/OS/linux/drivers/hwspinlock/qcom_hwspinlock.c` (lines 1–261)
   - Extracted TCSR hardware mutex register stride (`0x1000`), number of locks (`32`), APSS lock token value (`1`), and compare-and-set acquire (`write(1)` + `read() == 1`) and release (`write(0)`) sequences.

2. `/home/jkane/Projects/OS/linux/drivers/soc/qcom/smem.c` (lines 1–1308) and `/home/jkane/Projects/OS/linux/include/linux/soc/qcom/smem.h` (lines 1–23)
   - Extracted SMEM v12 architecture, global header (`smem_header` at `0x0000`, `proc_comm[4]`, `version[32]`, `initialized`, `free_offset`, `available`, `reserved`, `toc[512]`), partition table (`smem_ptable` at `base + size - 0x1000`, magic `"$TOC"`, `smem_ptable_entry` 48-byte layout), `smem_info` (magic `"SIII"`), partition header (`smem_partition_header`, magic `"$PRT"`), 16-byte private item entry (`smem_private_entry`, canary `0xa5a5`), uncached upward heap vs. cached downward heap traversal and allocation algorithms, TCSR hwspinlock 3 usage, and host IDs (`0..12`, `0xfffe`).

3. `/home/jkane/Projects/OS/linux/drivers/soc/qcom/socinfo.c` (lines 1–970) and `/home/jkane/Projects/OS/linux/include/linux/soc/qcom/socinfo.h` (lines 1–116)
   - Extracted SMEM item ID `137` (`SMEM_HW_SW_BUILD_ID`), `socinfo` structure layout (`fmt`, `id`, `ver`, `build_id`, `raw_id`, `hw_plat`, `pmic_model`, `serial_num`), and SC8280XP SoC ID (`449` / `0x01c1`).

4. `/home/jkane/Projects/OS/linux/drivers/mailbox/qcom-ipcc.c` (lines 1–382) and `/home/jkane/Projects/OS/linux/include/dt-bindings/mailbox/qcom-ipcc.h` (lines 1–40)
   - Extracted IPCC MPROC protocol register offsets (`IPC_VERSION = 0x00`, `IPC_CONFIG = 0x08` bit 0 `CLEAR_ON_RECV_RD`, `IPC_SEND_ID = 0x0c`, `IPC_RECV_ID = 0x10`, `IPC_RECV_SIGNAL_ENABLE = 0x14`, `IPC_RECV_SIGNAL_DISABLE = 0x18`, `IPC_RECV_SIGNAL_CLEAR = 0x1c`, `IPC_CLIENT_CLEAR = 0x38`), `(client_id << 16) | signal_id` bitfield layout, client IDs (`0..18`), and signal IDs (`0..3`).

5. `/home/jkane/Projects/OS/linux/drivers/rpmsg/qcom_glink_native.c` (lines 1–1987), `/home/jkane/Projects/OS/linux/drivers/rpmsg/qcom_glink_smem.c` (lines 1–386), and `/home/jkane/Projects/OS/linux/drivers/rpmsg/qcom_glink_native.h` (lines 1–42)
   - Extracted GLINK SMEM item IDs (`478` descriptors, `479` TX FIFO, `480` RX FIFO, `16384` bytes), 8-byte ring alignment rules, `FIFO_FULL_RESERVE = 8`, all 15 GLINK wire commands (`0..15`), `CMD_VERSION` (`1`) and `CMD_OPEN` handshakes, reusable intent pre-allocation (`5 × 1 KB` default, `20 × 512 B` for `"adsp_apps"`), and `CMD_RX_DONE_W_REUSE` (`14`) flow control.

6. `/home/jkane/Projects/OS/linux/net/qrtr/af_qrtr.c` (lines 1–800), `/home/jkane/Projects/OS/linux/net/qrtr/ns.c` (lines 1–800), `/home/jkane/Projects/OS/linux/net/qrtr/qrtr.h` (lines 1–39), and `/home/jkane/Projects/OS/linux/include/uapi/linux/qrtr.h` (lines 1–50)
   - Extracted QRTR v1 (`version = 1`, 32-byte header) and v2 (`version = 3`, 16-byte header) wire formats, control port (`0xfffffffe`), broadcast node (`0xffffffff`), and control packet types (`1..11`, 20-byte `qrtr_ctrl_pkt`).

7. `/home/jkane/Projects/OS/linux/drivers/soc/qcom/pdr_interface.c` (lines 1–741), `/home/jkane/Projects/OS/linux/drivers/soc/qcom/pdr_internal.h` (lines 1–104), and `/home/jkane/Projects/OS/linux/include/linux/soc/qcom/pdr.h` (lines 1–31)
   - Extracted PDR QMI services (`SERVREG_LOCATOR = 0x40` instance `1`, `SERVREG_NOTIF = 0x42`), QMI message IDs (`0x0020`, `0x0022`, `0x0023`), TLV layouts, and `curr_state` values (`0x0fffffff` DOWN, `0x1fffffff` UP, `0x2fffffff` EARLY_DOWN).

8. `/home/jkane/Projects/OS/linux/drivers/soc/qcom/pmic_glink.c` (lines 1–451), `/home/jkane/Projects/OS/linux/drivers/soc/qcom/pmic_glink_altmode.c` (lines 1–719), and `/home/jkane/Projects/OS/linux/include/linux/soc/qcom/pmic_glink.h` (lines 1–34)
   - Extracted `"PMIC_RTR_ADSP_APPS"` GLINK channel name, `pmic_glink_hdr` (`owner`, `type`, `opcode`), SC8280XP owners (`0x800a` BATTMGR, `0x800b` USBC, `0x800c` USBC_PAN), opcodes (`0x15` `USBC_CMD_WRITE_REQ`, `0x16` `USBC_NOTIFY_IND`), commands (`0x10` `ALTMODE_PAN_EN`, `0x11` `ALTMODE_PAN_ACK`), 32-byte SC8280XP `usbc_sc8280xp_notify_ind` payload, and asynchronous worker ACK ordering.

9. `/home/jkane/Projects/OS/linux/drivers/phy/qualcomm/phy-qcom-qmp-combo.c` (lines 1–800, 2200–3599, 3845–5444), `/home/jkane/Projects/OS/linux/drivers/phy/qualcomm/phy-qcom-qmp-pcs-usb-v4.h` (lines 1–35), `/home/jkane/Projects/OS/linux/drivers/phy/qualcomm/phy-qcom-qmp-dp-com-v3.h` (lines 1–19), and `/home/jkane/Projects/OS/linux/drivers/phy/qualcomm/phy-qcom-qmp-pcs-v4.h` (lines 1–136)
   - Extracted QMP v5 Combo PHY sub-block offsets (`com = 0x0000`, `txa = 0x0400`, `rxa = 0x0600`, `txb = 0x0a00`, `rxb = 0x0c00`, `usb3_serdes = 0x1000`, `usb3_pcs_misc = 0x1200`, `usb3_pcs = 0x1400`, `usb3_pcs_usb = 0x1700`, `dp_serdes = 0x2000`, `dp_dp_phy = 0x2200`), `QPHY_V3_DP_COM_TYPEC_CTRL` (`+0x0010`: `0x02` normal, `0x03` reverse), and `QPHY_V4_PCS_SW_RESET` (`+0x1400` bit 0).

10. `/home/jkane/Projects/OS/linux/arch/arm64/boot/dts/qcom/sc8280xp.dtsi` (lines 2000–7000) and `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/dumps/dt/live.dts` (lines 1700–3300, 7500–9900)
    - Extracted physical base addresses (`0x01f40000` TCSR mutex, `0x80900000` SMEM, `0x00408000` IPCC with GIC SPI `229`, `0x088eb000` Port 0 Combo PHY, `0x08903000` Port 1 Combo PHY).

11. `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/README.md` (lines 1–336), `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/usb.md` (lines 1–115), and `/home/jkane/Projects/OS/AbyssBSD/docs/boards/radxa-dragon-q8b/lessons.md` (lines 1–134)
    - Extracted empirical Radxa Dragon Q8B findings: 1-in-3 boot hang hazard when GLINK channels are closed/reopened or accessed before ADSP readiness, `USBC_NOTIFY_IND` (`opcode = 0x16`) arriving with `owner = 0x800a` on SC8280XP, asynchronous `ALTMODE_PAN_ACK` requirement, and runtime QMP Combo PHY `DP_COM_TYPEC_CTRL` + `PCS_SW_RESET` sequence.
