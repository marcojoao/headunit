# Mazda Connect platform research (no vehicle access)

Date: 2026-11-16. Desk research only. Nothing here was run on a car.

## Summary

1. A Mazda Connect emulator exists: **mzd-emu** (ScorpAL/mzd-emu, MIT, active Nov 2026). It runs the **GUI layer only** (JCI Opera HTML/JS) in Chrome with a mock MMUI backend. It does **not** run native ARM binaries such as `headunit`. It needs a `jci/` firmware dump supplied by the user (based on 74.00.324 EU N).
2. The CMU's internal D-Bus API is documented (silverchris/mazda-cmu-documentation, 97 interfaces). This answers most "does the hardware support X" questions without touching the car.
3. Kernel config confirms Bluetooth (`CONFIG_BT=m`, L2CAP, SCO, RFCOMM, BNEP, HIDP) and ALSA (`CONFIG_SND=y`, `SND_PCM`, `SND_ALOOP=m`, `SND_USB_AUDIO=m`, i.MX6 SoC audio). Bluetooth is owned by JCI's own stack (`com.jci.bds`), not BlueZ D-Bus.
4. **Strategic finding:** firmware v70+ ships Mazda's own Android Auto (`aap_service` + `blmjciaapa.so`, D-Bus `com.jci.aapa`). That OEM stack already exposes Bluetooth pairing, audio focus and reverse-camera handling. The community project **oem-aa-mod** (VitaliyKurokhtin/oem-aa-mod, AGPL-3.0) improves it with LD_PRELOAD shims and already ships a mute to pause/resume media bridge.

## Relevant D-Bus interfaces

| Interface | Useful members | Maps to goal |
| --- | --- | --- |
| `com.jci.bds` (Bluetooth device service) | `SignalConnected_cb`, `SignalDisconnected_cb`, `SignalDeviceGetConnectedDevices_cb`, `SignalAudioSink*`, `SignalCallClientSpeakerVolumeChange_cb` | BT auto-connect trigger: watch for the paired phone connecting |
| `com.jci.aapa` (OEM Android Auto adapter) | `StartBtConnection`, `BTPairingRequest`, `BTPairingResult`, `BTReadyToPair`, `NotifyBTConnectionComplete`, `NotifyAudioFocusStatus`, `NotifyDeviceConnection`, `GetAOASessionStatus`, `DisplayReconnectingDialog` | BT pairing + session prompt, audio focus |
| `com.jci.bucpsa` (back-up camera) | `GetReverseStatus`, `ReverseStatusChanged`, `DisplayMode` | Camera enter/exit, resume state |
| `com.jci.audiosettings` | `BLM_SetVolume`, `BLM_VolumeLevel`, `BLM_GetSettingInfo` | Volume control |
| `com.jci.cpp.vbi.Amplifier` | `getVolume`, `setVolume` | Direct amplifier volume |
| `com.jci.btmusic` | `StartStreaming`, `StopStreaming`, `StreamingStatus` | BT A2DP fallback |

Full specs: `~/Repos/mazda-reference/mazda-cmu-documentation/dbus/`.

## Two paths for the three goals

### Path A: keep improving the legacy `headunit` (this repo)
- Works on old firmware (< v70) without the USB hub retrofit.
- BT auto-connect: subscribe to `com.jci.bds` `SignalConnected_cb`, match the paired phone, then show a prompt and wait for USB AOAP. Wireless AA would need new Wi-Fi/BT RFCOMM handshake work (see aa-proxy-rs / AAWirelessDongle for reference).
- Camera resume: already handled via `bucpsa`; remove the fixed 1000 ms refocus delay, add audio ducking.
- Sound: call `com.jci.audiosettings` / `Amplifier` instead of raw ALSA only; wire the dropped `AA_CH_AU2` channel.

### Path B: build on OEM Android Auto + oem-aa-mod (recommended to evaluate first)
- Requires firmware v70+ (74.00.324 recommended) and a CarPlay/AA USB hub. The CX-3 is supported by Mazda's retrofit.
- OEM stack already handles BT pairing (`com.jci.aapa`), audio focus and the camera. Wireless is not supported by OEM firmware.
- Goals become small LD_PRELOAD shims next to oem-aa-mod's `blmjciaapa` patch: auto-prompt on `bds` connect, faster camera resume, better ducking/volume via `audiosettings`.
- Maintained in 2026, modern Makefile, release `-O3 -flto`, toolchain `lmagder/m3-toolchain` (GCC 4.9.1, Cortex-A9 NEON).

## Testing without the car

| Layer | Tool | Coverage |
| --- | --- | --- |
| GUI (JS apps, AIO tweaks, `_androidauto` app) | mzd-emu + firmware `jci/` dump | Good, mocked backend |
| Native ARM binaries (`headunit`, shims) | qemu-arm user mode + extracted rootfs as sysroot | Partial: starts, logic and D-Bus can be mocked, no real hardware |
| D-Bus behaviour | Local session bus + mock services generated from the documented XML | Good for unit tests of signal handling |
| Real BT/audio/camera | Car only | Not possible offline |

## Open questions for the owner
- Which firmware is on the CX-3 now (Settings > System > About)? Decides Path A vs B.
- Is the CarPlay/AA USB hub retrofit already installed?
- A firmware dump is needed for mzd-emu and qemu. Mazda distributes update files publicly; extracting is offline and does not touch the car.

## Cloned references (local, read-only)
`~/Repos/mazda-reference/`: `mzd-emu`, `oem-aa-mod`, `mazda-cmu-documentation`, `mazda-toolchain`.
Also worth reading: `Trevelopment/headunit` (fork oem-aa-mod derives from), patrikguth's D-Bus logger gist for 74.00.324.
