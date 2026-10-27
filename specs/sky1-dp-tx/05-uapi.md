# 05: Userspace-Visible Interface

Generic KMS clients and compositors drive the DP outputs through standard
KMS objects. Nothing DP-specific is needed by Mesa. This section lists what
the vendor driver exposes, so a clean-room driver can match it where
userspace might notice.

## 5.1 KMS objects

- **[ABI]** One connector per DP unit, attached to the DPU's KMS device
  (one KMS device per DPU). Connector type: **DisplayPort**; **eDP** when
  an `edp-panel` is attached (unit 2); **HDMI-A** when `port@2` leads to a
  bridge chain ending in an HDMI connector (Orange Pi 6 Plus).
- **[ABI]** Connector polling: HPD-driven (no periodic polling).
- **[POLICY]** *(vendor)* The SST encoder can only be driven by the DPU's
  first CRTC (pipeline 0). The vendor driver registers it as a "TMDS"
  encoder type; a DisplayPort-type encoder is more correct and harmless
  to userspace.
- **[ABI]** MST: when an MST branch is attached, the SST connector reports
  "disconnected" and one connector per branch port appears, with the
  standard `PATH` and `TILE` properties. Two MST encoders (`DP-MST A`,
  `DP-MST B`) can each use either CRTC of the DPU.

## 5.2 Connector properties

| Property | Values | Notes |
|---|---|---|
| `Colorspace` | Standard DP colorspace list | A change forces a full mode-set (02 §2.7.1) |
| `HDR_OUTPUT_METADATA` | Blob | Sent as an HDR static-metadata InfoFrame SDP (02 §2.7) |
| `max bpc` | 8 … 10 | Also on MST connectors |
| `subconnector` | Standard DP subconnector enum | From the downstream port type |
| `content type` | Standard enum | |
| `vrr_capable` | 0/1 | Set at detect (02 §2.13.2) |
| `EDID`, `PATH`, `TILE`, `link-status`, `DPMS`, `CRTC_ID` | Standard | |

- **[ABI]** The CRTC's `VRR_ENABLED` and the self-refresh helpers are used
  in the standard way; PSR entry/exit is invisible to userspace (02 §2.10).
- **[ABI]** Mode list: from the EDID; for eDP, standard lower resolutions
  (640×480 … 3840×2160 DMT/CTA timings smaller than the panel's preferred
  mode) are added. Without an EDID, standard modes up to 1920×1080 with
  1080p preferred.

## 5.3 debugfs (per connector) *(vendor, not ABI)*

| File | Content |
|---|---|
| `register_info` | Dump of the core registers (link, enables, AUX, PHY status, MST, source blocks) |
| `link_rate` | Current link rate (kHz) and lane count |
| `mst_topology` | MST topology dump |
| `psr_default_on` | Read/write boolean: PSR enable |
| `mst_default_on` | Read/write boolean: allow MST on the next connect |

Module parameters `aux_timeout_ms` (default 50) and `power_on_delay_ms`
(default 4), read-only. Neither is needed for compatibility.

## 5.4 Audio (ALSA)

- **[ABI]** Each DP unit registers a generic HDMI audio codec instance
  named `hdmi-audio-codec.N`. Under ACPI, N is the DP device's `_UID`
  (0 … 4, matching the DP unit); otherwise N is assigned automatically.
  The machine driver binds I2S(5 + N) to codec N with DAI name `i2s-hifi`.
- **[ABI]** Codec capabilities: I2S input only, **playback only** (no
  capture), up to 8 channels, 32–192 kHz, 16/18/20/24-bit.
- **[ABI]** The codec reports the sink's ELD (from the EDID) once the EDID
  has been read, and a plug/unplug jack event per output. With no sink or
  no ELD yet, the ELD query reports "not available" rather than an
  all-zero ELD.
- **[ABI]** *(community)* Sound card model `cix_sky1` (matches the ALSA
  UCM2 profile directory); DAI links named `HDMI/DP 0` … `HDMI/DP 4`;
  jack pins named `HDMI/DP,pcm=N`.

## 5.5 HDCP private interface *(vendor, summary only)*

- **[ABI]** *(vendor)* One misc character device per DP unit, named
  `hdcp-<device name>` (the vendor driver appends a newline to the name by
  mistake). It supports `open`, `read`, `poll` and `ioctl`; `read`/`poll`
  deliver events (HPD changes, timer expiries, `CP_IRQ` status).
- **[ABI]** *(vendor)* Ioctl magic `'H'`, numbers `0x00` … `0x08`: read
  receiver state, start/stop the HDCP timer, raw DPCD access over AUX,
  hardware initialisation, cipher enable/disable, and two key-derivation
  helpers. The argument structures are private to the CIX HDCP daemon.
- **[ABI]** No open-source userspace uses this interface. HDCP keys and
  the protocol state machine live in the closed daemon. A clean-room
  driver may omit it; nothing in Mesa or common compositors depends on it.

## 5.6 CEC

- **[ABI]** *(community)* When a DP-to-HDMI converter supports CEC
  tunnelling, a standard CEC adapter device appears for the connector
  (02 §2.13.3).
