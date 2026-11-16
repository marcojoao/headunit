# Investigation: gartnera/headunit — Bluetooth auto-connect, backup-camera resume, audio control

Date: 2026-11-15
Scope: read-only investigation of `~/Repos/headunit` (fork of `gartnera/headunit`, origin of
`spadival/headunit`). No code changes, no commits. Target hardware: Mazda CX-3 CMU (i.MX6-class
ARM Cortex-A9, Linux, M3 toolchain, GStreamer 0.10).

## 1. Current architecture summary

Three build targets share `common/` and `hu/`:

- `hu/` — the Android Auto / AOAP protocol engine (AASDK-style): `hu_aap.cpp/.h` (framing,
  SSL/crypto handshake, channel open/media/control message dispatch — `hu_handle_*`),
  `hu_usb.cpp` (libusb AOAP accessory-mode negotiation + bulk transfer I/O thread),
  `hu_tcp.cpp` (Wi-Fi/TCP transport alternative), `hu_ssl.cpp` (OpenSSL handshake wrapper),
  `hu_aad.cpp` (auth), `hu.proto` (protobuf message schema, hand-copied from AA's real proto,
  not fetched from Google).
- `common/` — `audio.cpp/.h` (ALSA playback/mic, generic, used verbatim by both `mazda` and
  `ubuntu` targets), `command_server.cpp` (local control/status HTTP-ish server), `config.cpp`,
  `glib_utils.cpp` (glib main-loop dispatch helpers used for cross-thread callback marshalling).
- `mazda/` — JCI (Mazda's internal "Connected Car" middleware) integration: `main.cpp` (process
  entry point, GStreamer + DBus init, outer connect/disconnect loop), `callbacks.cpp`
  (`MazdaEventCallbacks` implements `IHUConnectionThreadEventCallbacks`; `AudioManagerClient` /
  `VideoManagerClient` talk to JCI's `com.jci.bucpsa` / `com.xsembedded.service.AudioManagement`
  D-Bus services for video surface and audio-focus arbitration), `outputs.cpp` (GStreamer video
  sink + uinput/evdev touch and hardware-button handling), `bt/mzd_bluetooth.cpp` (reads CMU's
  fused MAC address out of `/sys/fsl_otp/HW_OCOTP_MAC{0,1}` to report as "car Bluetooth address"
  to the phone), `hud/`, `gps/`, `nm/` (night mode) sensor feeders.
- `ubuntu/` — desktop test harness with its own `bt/ub_bluetooth.cpp`, used for dev/testing only
  (not what ships on the car).

Threading model: one dedicated `hu_thread` inside `HUServer` owns the AOAP transport and protocol
state machine; a glib main loop (`run_on_thread_main_context`) on the "main" thread is where
`MazdaEventCallbacks` work is marshalled via `run_on_main_thread(...)`/`hu_queue_command(...)`
(two separate cross-thread queues, one glib-based, one pipe+command-queue based in `HUServer`).
Separate OS threads: USB libusb event thread (`hu_usb.cpp`), mic capture thread
(`common/audio.cpp` `MicInput`), video/touch input thread (`mazda/outputs.cpp`), GPS thread,
night-mode thread, HUD thread — all in `mazda/main.cpp`.

**Transport is USB-only for actual AA session data.** `hu_aap_start()` takes a
`HU_TRANSPORT_TYPE` of `USB` or `WIFI` (`hu_usb.cpp` / `hu_tcp.cpp`), selected from
`config::transport_type`. Bluetooth is **never** used as a data transport. The only Bluetooth
touchpoint in the whole codebase is `GetCarBluetoothAddress()` (`mazda/bt/mzd_bluetooth.cpp`,
`hu/hu_aap.cpp:572-575`), which hands the phone the CMU's own paired BT MAC address inside the
`ServiceDiscoveryResponse`/`BluetoothPairingRequest` exchange (`HU::BLUETOOTH_PARING_METHOD_A2DP`
/ `_HFP`) so the phone can auto-pair A2DP/HFP for calls and music *fallback*, and
`HU_BLUETOOTH_CHANNEL_MESSAGE::BluetoothAuthData`/`BluetoothPairingRequest` handlers in
`hu_aap.cpp` (~line 906+) which just acknowledge/forward pairing status — there's no code that
initiates BT pairing or scanning, listens for BT connection events, or bridges a BT event to
starting USB/accessory negotiation. This confirms the user's assumption: BT today is a
one-shot "tell the phone which MAC to also use for HFP/A2DP" step embedded in the AA
handshake, not a connection trigger.

## 2. Feature 1 — Bluetooth auto-connect when entering the car

### (a) Current state
- No BT stack ownership: the Mazda CMU's actual Bluetooth stack (pairing, auto-reconnect on
  ignition/door event) is owned entirely by JCI's own BT service, completely outside this
  codebase — headunit only reads the CMU's own MAC (`mzd_bluetooth.cpp:44-78`) to publish it.
- `headunit` itself has **no BlueZ/D-Bus BT client code at all** — no `org.bluez` interface use,
  no HCI socket code, nothing in `mazda/dbus/cmu_interfaces.xml` for BT.
- Connection trigger today is purely USB: `hu_usb.cpp:Start()` polls/waits for an AOAP device
  (`find_oap_device()`/`wait_for_device_connection()`), driven by physical USB cable insertion
  (or whatever already-paired USB hub/accessory is present), with no BT-driven kickoff.
- `config::phoneIpAddress` + `HU_TRANSPORT_TYPE::WIFI` (`hu_tcp.cpp`) exists for Wi-Fi AA but is
  manually configured (`d46c7eb` commit: user writes phone IP into config), again nothing
  BT-triggered.

### (b) Prior art
- **AAWireless / AAWireless Gateway**: dongle that does exactly this — listens for the phone's
  BT connection (classic BT SPP/legacy auto-connect using the car's existing paired profile as a
  "wake" signal), then creates a Wi-Fi hotspot and tells the phone (via a BT RFCOMM control
  channel, Google's own "Wireless AA" handshake) to connect AA over Wi-Fi Direct/TCP. It
  reverse-engineers Google's wireless-AA bootstrap protocol (same one used by factory wireless AA
  head units) layered over BT.
- **OpenAuto / crankshaft (bluekitchen/f1xpl's openauto + AASDK)**: AASDK's Bluetooth submodule
  uses BlueZ D-Bus (`org.bluez.Adapter1`, `org.bluez.Device1`) to watch for `PropertiesChanged`
  on `Connected=true` for a known paired phone MAC, and on that signal, if USB/AOAP isn't already
  active, it re-triggers `libusb` AOAP negotiation or starts the wireless AA bootstrap sequence.
  This is architecturally the closest match since AASDK is this project's spiritual ancestor.
- **Generic "bluetooth-auto-connect" Linux projects / car-pc builds** (Raspberry Pi car-pc,
  `bluetoothctl`-script approaches): use `bluetoothctl`/BlueZ D-Bus signal
  `org.freedesktop.DBus.Properties.PropertiesChanged` on `/org/bluez/hciX/devYY` watching
  `Connected`, then shell out to reconnect/trigger other services. Simple polling-vs-signal
  tradeoff is well documented in these.

### (c) Options and tradeoffs
1. **D-Bus BlueZ signal listener inside headunit** (preferred): add a small BlueZ D-Bus client
   (reuse the existing `dbus-c++` library already linked in, same pattern as
   `AudioManagerClient`/`VideoManagerClient` in `callbacks.cpp`) that subscribes to
   `PropertiesChanged` on the JCI CMU's own `org.bluez` adapter/device objects for the known
   paired phone, and on `Connected: true`, if USB isn't already active, calls
   `HUServer::hu_aap_start` (retrigger loop already exists in `mazda/main.cpp`'s `while(true)`).
   - Pro: reuses existing architecture 1:1 (same cross-thread command-queue pattern already used
     for JCI audio/video), no new processes.
   - Con: need to confirm the Mazda CMU's BT stack is in fact BlueZ (JCI firmware is proprietary;
     likely a vendor fork of BlueZ or a completely closed stack with its own D-Bus service
     mirroring `org.bluez` semantics — must be probed on a real unit via `dbus-send`/`d-feet`
     against the CMU's system bus first). This is the single biggest unknown and should be the
     very first thing validated on hardware before committing to this approach.
2. **Polling via `hcitool`/`bluetoothctl` subprocess** — simpler, no D-Bus introspection needed,
   but adds latency (~1-2s poll interval) and process-spawn overhead on the target's modest CPU;
   fragile if BT tooling isn't present in the Mazda rootfs image.
3. **"Wireless AA over BT" bootstrap** (AAWireless-style) — only relevant if the goal is removing
   the physical USB cable entirely (not just "auto-start session when I get near the car" but
   phone connects BT→Wi-Fi hotspot→AA). This is a much bigger lift: requires implementing
   Google's wireless-AA RFCOMM handshake (reverse-engineered, not officially documented) plus a
   Wi-Fi AP/hostapd stack on the CMU, which is almost certainly not present/available on this
   locked-down embedded image. Feasibility: low on this hardware without root-level OS changes.
4. **Minimal version**: don't chase full wireless AA — just use BT "device connected" as the
   *signal* to immediately start polling for USB device presence more aggressively / surface a
   "phone connected via BT, plug in USB" prompt on the HUD. Much lower risk, no protocol
   reverse-engineering, delivers most of the UX win (no manual app launch) if the cable stays in.

### (d) Feasibility for this target
- Medium, if the Mazda BT stack exposes BlueZ-compatible D-Bus signals (need to probe on real
  hardware — 30 minutes of `dbus-monitor` against the CMU's buses would settle this). The
  D-Bus client plumbing pattern (`DBus::ObjectProxy`, `Notify` callbacks) already exists in this
  codebase (`AudioManagerClient::Notify`, `callbacks.cpp:567`) and can be copy-adapted.
- Full "USB-free wireless AA" is high-effort/high-risk and should be treated as a separate,
  much later milestone, not bundled with "auto-connect".

## 3. Feature 2 — Resume app/UI state when reverse camera triggers

### (a) Current state
This is **already substantially implemented**, just not labeled "resume". The flow:
- `VideoManagerClient` (`callbacks.cpp:180-252`) subscribes to JCI's `com.jci.bucpsa` HMI D-Bus
  service and tracks `DisplayMode` (backup camera surface) via `GetDisplayMode`/`DisplayMode`
  signal callback.
- On backup-camera activation (`DisplayMode(currentDisplayMode != 0)`): sets
  `allowedToGetFocus = false`, if AA currently has video focus it sets `waitsForFocus = true` and
  calls `releaseVideoFocus(VIDEO_FOCUS_REQUESTOR::BACKUP_CAMERA)` which tells the phone
  `VIDEO_FOCUS_MODE_UNFOCUSED` (`VideoFocusHappened(false, unrequested=true)`,
  `hu_aap.h` `HU::VideoFocus`) and requests the CMU switch screen surface back to
  `JCI_OPERA_PRIMARY` is skipped since backup cam owns the surface.
- On backup-camera deactivation: `DisplayMode(0)` → `allowedToGetFocus = true`; if
  `waitsForFocus` was set, calls `requestVideoFocus(VIDEO_FOCUS_REQUESTOR::BACKUP_CAMERA)`
  which, specifically for the `BACKUP_CAMERA` requestor, delays 1000ms
  (`run_on_main_thread_delay`, `VideoManagerClient::requestVideoFocus:215-220`) "to make sure the
  CMU has already changed the surface from backup camera to opera" before re-requesting AA video
  focus and re-switching the surface to `TV_TOUCH_SURFACE`.
- So: AA already gets told `VIDEO_FOCUS_MODE_UNFOCUSED`/`FOCUSED` around backup-camera events,
  and the phone-side AA app is itself responsible for "resuming" (the AA protocol's video-focus
  state machine is explicitly designed so the phone keeps rendering/logical-state and just
  stops/starts pushing frames — this is not a "kill and relaunch app" model, it's "pause/resume
  video stream", which is the correct AA-native behavior and already matches Google's own spec).
- **What's genuinely missing / fragile**: (1) the 1000ms fixed delay is a magic number, not an
  actual "surface switched" confirmation (there's no JCI signal awaited beyond the single
  `DisplayMode` property; a faster or slower CMU firmware could desync this), (2) no explicit
  "last foreground app" state is persisted/restored — relies entirely on the phone-side AA stack
  to remember, i.e. for touch-input/menu context it's reasonable but if the *headunit* process
  itself restarts (crash/disconnect) mid-reverse, `mazda/main.cpp`'s outer `while(true)` loop
  does a full re-handshake, losing any session continuity, (3) audio ducking is NOT tied to this
  at all currently — see Feature 3.

### (b) Prior art
- Google's official AA protocol: `VideoFocusRequest`/`VideoFocus` messages
  (`HU::VIDEO_FOCUS_MODE_FOCUSED`/`UNFOCUSED`, `unrequested` flag) are exactly the documented
  mechanism car OEMs use for "camera owns screen" interruption — this project already implements
  the correct, Google-sanctioned pattern, unlike some hobbyist units that instead disconnect AA
  entirely during reverse.
- OpenAuto/AASDK: same `VideoFocusRequest` message type exists; openauto's reference
  implementation does a similarly simple on/off toggle without extra app-state machinery, which
  validates that this project is not missing anything structural here.
- Crankshaft/RPi car-pc projects without JCI-style backup-camera HMI integration generally hard-
  switch an HDMI/composite input via relay, with no video-focus signaling to the phone at all —
  a cruder approach that this project's design is already better than.

### (c) Options for improvement (given it's largely already correct)
1. Replace the fixed 1000ms `run_on_main_thread_delay` with an explicit JCI signal/poll loop
   confirming the surface switch completed (if `com.jci.bucpsa` exposes a completion event) —
   reduces flicker/race risk, minor engineering effort, needs CMU D-Bus introspection to find
   the right signal (may not exist; fallback is tuning the constant empirically).
2. Persist "was focused before backup cam" and a small amount of UI-adjacent state (e.g. last
   known `channel_session_id` for AA_CH_VID) across a headunit process crash/reconnect, so a
   genuine process-level failure during reverse doesn't force the user to reopen the AA app
   manually — currently `hu_aap_start` cold-starts fully. Medium effort, mostly bookkeeping in
   `MazdaEventCallbacks`/`HUServer`.
3. Add a short audio duck/cue aligned with video focus loss (see Feature 3) so reverse-camera
   entry doesn't leave AA media audio playing uninterrupted while the screen shows a camera feed
   the driver didn't ask AA to pause music for — currently unrelated, no audio-focus message is
   sent when video focus alone changes (`VideoFocusHappened` never touches `audioMgrClient`).

### (d) Feasibility
Low-to-medium — this is the best-implemented of the three target areas already; the main real
deliverable is wiring audio ducking into it (shared with Feature 3) and hardening the fixed
delay/crash-recovery edges, not building something new.

## 4. Feature 3 — Better audio control (ducking, routing, codec quality)

### (a) Current state
- `common/audio.cpp`/`common/audio.h`: raw ALSA (`libasound`), opened once at startup
  (`AudioOutput::AudioOutput`, constructed in `MazdaEventCallbacks` ctor,
  `callbacks.cpp:23` `new AudioOutput("entertainmentMl")`) with **two fixed PCM handles**:
  `aud_handle` (stereo 48kHz S16LE — media/`AA_CH_AUD`) and `au1_handle` (mono 16kHz S16LE —
  guidance/`AA_CH_AU1`, e.g. nav voice prompts). No sample-rate negotiation, no bit-depth beyond
  S16LE, no mixing of AUD vs AU1 vs AU2 streams (AU2 channel is declared in `hu_aap.h`'s channel
  table but never actually opened/used in `AudioOutput`) — the "ducking" AA expects (its own nav
  voice-over-music is typically handled by AA sending separate transient AudioFocusRequest, see
  below) relies entirely on JCI's own focus arbitration rather than in-process mixing/ducking.
- Volume: **there is no volume control code in this repo at all** — no `snd_mixer_*` calls
  anywhere, no `amixer`/ALSA mixer element manipulation; actual loudness is whatever the CMU's
  own JCI audio pipeline/hardware mixer currently has configured for the `"entertainmentMl"`
  ALSA device, and application-level volume (phone side) is the only lever. Ducking for calls vs
  media is approximated only via JCI's **audio-focus session arbitration**
  (`AudioManagerClient` in `callbacks.cpp:330-566`), which opens two JCI audio sessions
  (`"permanent"`/`"transient"`, `aaStreamName`) via its proprietary JSON-over-D-Bus
  `com.xsembedded.service.AudioManagement` interface, and maps AA's own
  `HU::AudioFocusRequest`(`GAIN`/`GAIN_TRANSIENT`/`RELEASE`) to JCI session focus requests. This
  is reasonable and matches Google's official focus model, but it's JCI arbitrating loudness/
  mixing at a level below this codebase's control — this project cannot currently adjust
  relative ducking depth, fade speed, or do anything AA's own spec doesn't dictate.
- Routing: hardcoded to a single ALSA device string (`"entertainmentMl"` on Mazda,
  `audioOutput.reset(new AudioOutput("entertainmentMl"))`), no runtime selection of output
  device/route (e.g. no distinguishing speakers vs. a hypothetical aux/line-out), no handling of
  `AA_CH_AU2` (a second guidance-type channel AA can request) at all — any phone-side app that
  uses AU2 instead of AU1/AUD would silently get no audio out (dropped in `MediaPacket`
  dispatch in `callbacks.cpp:32-42`, which only switches on `AA_CH_VID/AUD/AU1`).
- Codec quality: AA's media/audio channel negotiation (`MediaSetupRequest`/`MediaSetupResponse`,
  `CustomizeOutputChannel` hook in `hu_aap.h:161`/used only for video margin fix in
  `callbacks.cpp:75-82`) is accepted largely as Google's phone-side default — there is no
  explicit advertisement of preferred/higher sample rates, no negotiation toward 16-bit vs any
  higher-resolution PCM option, and video is hardcoded to the CMU's native 800x480 H.264 decode
  path (`hu_usb`/`outputs.cpp` GStreamer pipeline `vpudec`), i.e. "quality" here is entirely
  capped by Google's AA protocol defaults plus this fixed ALSA/GStreamer pipeline, not something
  this code is leaving on the table beyond the AU2/ducking gaps above.

### (b) Prior art
- **OpenAuto/AASDK**: has a full `RtAudio`/ALSA abstraction with explicit per-channel
  (media/system/speech) stream objects, and ducking implemented by explicitly lowering a
  gain/mixer element on "transient" focus requests from AA rather than depending on an external
  focus arbiter — a cleaner separation of concerns than this project's "fully delegate to JCI"
  approach, useful as a reference if the user ever wants in-process ducking instead of relying
  on CMU audio-focus plumbing.
- **PulseAudio/ALSA car-audio routing projects (Raspberry Pi car-pc, OpenCarAudio)**: typically
  use PulseAudio's "role" based ducking (`module-role-ducking`, used heavily in GNOME
  `media-role=phone` ducking) to automatically attenuate music streams when an `event`/`phone`
  role stream appears — this gives free per-source volume ducking with configurable fade speed,
  trivially controllable via `pactl`/`pavucontrol`. Not directly reusable since the Mazda CMU
  almost certainly runs bare ALSA (no PulseAudio daemon observed in the Makefile's linked libs —
  only `alsa`), but conceptually this is the "add ducking ourselves" option if the project ever
  bypasses JCI's audio-focus service.
- **AAWireless/other ducking-focused forks**: generally still delegate ducking to the official
  AudioFocus protocol messages (same model this project already follows) rather than reinventing
  it — reinforces that the *protocol* side here is already correct; any real-world gap is in the
  Mazda-side JCI audio session plumbing (AU2 support, volume/mixer access) not the AA layer.

### (c) Options and tradeoffs
1. **Implement AU2 channel support** (low effort, clear gap): add a third `snd_pcm_t` handle in
   `AudioOutput` (or route AU2 through the existing au1_handle if JCI only exposes one spare
   stream) and dispatch `AA_CH_AU2` in `MediaPacket`/`MediaStart`/`MediaStop` — currently any AA
   app segment using AU2 is just silently dropped.
2. **Add direct ALSA mixer volume control** (`snd_mixer_*`) if the CMU's audio device exposes a
   software-controllable mixer element for the `entertainmentMl` path — would let this code
   implement its own fade/duck curve instead of trusting JCI's session arbitration timing; needs
   hardware probing (`amixer -D entertainmentMl scontrols` or equivalent) to know if such an
   element even exists outside JCI's closed audio HAL — real risk this is JCI-hardware-owned and
   not exposed to userspace at all.
2b. Alternatively, keep relying on JCI's AudioManagerClient session model (don't try to out-duck
   JCI) but tune the two-session setup (permanent/transient) more finely — e.g. a third
   "ducked-but-audible" session type if JCI's `AudioManagement` service actually supports more
   granular focus types than the three currently used — needs introspecting the JCI D-Bus service
   API surface (`populateStreamTable`/`dumpState` already dumps the live state, useful starting
   point, `callbacks.cpp:404-478`).
3. **Codec/quality bump**: request higher sample rate / stereo for the AU1/guidance channel in
   `CustomizeOutputChannel`/`MediaSetupRequest` negotiation if phone + AA protocol both support
   it (AA protocol's media config is largely phone-driven, so gains here are capped by what
   Google's protocol advertises — low expected payoff, but worth checking `hu.proto`'s
   `AudioConfig` message for unused fields this code could set).
4. **Fix magic-number buffer sizing**: `MicThreadMain`'s 1MB fixed temp buffer allocation
   per-read-loop-iteration (`new uint8_t[tempSize]` every loop, `common/audio.cpp:150`) is
   wasteful and worth a ring-buffer/reuse refactor — tangential to "better audio" but directly
   affects mic/voice latency and GC-like churn on a slow ARM target.

### (d) Feasibility
- AU2 support: low effort, high confidence win.
- Direct ALSA mixer/volume control: unknown feasibility until probed on real hardware — the CMU's
  audio path may be entirely JCI-HAL-owned with no exposed ALSA mixer controls for userspace,
  in which case ducking improvements are limited to tuning JCI's own focus-session API more
  cleverly (option 2b), not to adding new mixer code.
- Codec/sample-rate improvements: low expected payoff, protocol-capped.

## 5. General code-quality / optimization findings

- **Dependency/toolchain age**: build uses a bespoke M3 cross-toolchain
  (`mazda/Makefile:6-20`) targeting **GStreamer 0.10** (EOL since ~2012, superseded by 1.x for
  over a decade), **protobuf 2.6.1** (2014, long past EOL, missing modern protobuf API/arena
  allocation/perf improvements), hand-rolled `pkg-config-wrapper`, and raw Makefiles (no CMake)
  with manual `.arm.o`/`.arm.d` suffix tracking and no incremental-dependency correctness beyond
  `-include $(DEPS)`. A CMake-based cross-compilation setup (toolchain file +
  `find_package`/`pkg_config` modules) would materially improve maintainability and make it
  possible to add a native (host) debug build target more cleanly; the project half-does this
  already via separate `ubuntu/Makefile` duplicating most of the same source list.
- **Manual memory management mixed with RAII inconsistently**: `HUServer` (`hu_aap.h:204`) does
  `new std::vector<uint8_t>()` for `temp_assembly_buffer` with no corresponding visible `delete`
  in the header (leak or cleanup buried in .cpp, needs checking) instead of a plain
  `std::vector<uint8_t>` member; `hu_usb.cpp:98` `malloc`/`free` for a USB write buffer instead of
  `std::vector`/`std::unique_ptr`; `common/audio.cpp:150,169,178` raw `new uint8_t[]` / `delete[]`
  per read-loop iteration (no RAII, no reuse, and `delete [] tempBuffer` only happens in one of
  two error branches around line 169 — worth double-checking the non-error happy-path lambda at
  line 174-179 actually always frees it, since the lambda capturing `tempBuffer` by value and
  deleting it should be fine, but it's two different deletion sites for the same allocation
  pattern, a classic double-free/leak risk surface).
- **Error handling is log-and-continue almost everywhere**: ALSA, D-Bus, USB, JSON-parse failures
  (`callbacks.cpp` multiple `catch(std::domain_error)`/`catch(std::invalid_argument)` blocks)
  log and fall through rather than surfacing a recoverable state machine transition — acceptable
  for a hobbyist embedded project but means partial failures (e.g. JCI audio session registration
  failing) can silently leave audio broken for an entire session with only a log line as evidence.
- **Threading**: generally reasonable (dedicated glib main-loop based dispatch
  `run_on_main_thread`/`run_on_main_thread_delay` plus `HUServer`'s own pipe-based command queue
  for the AA protocol thread), but there are two separate cross-thread queuing mechanisms
  (`IHUAnyThreadInterface::hu_queue_command` vs. glib's `run_on_main_thread`) doing conceptually
  the same "marshal this lambda onto a specific loop" job — worth unifying, reduces cognitive
  load and the risk of calling the wrong one from a given thread.
- **Global mutable state**: `g_hu` (global raw pointer, `main.cpp:46`), `navi_data`/`hudmutex`
  (externs declared ad hoc inside `callbacks.cpp:652-653` rather than in a header) — functional
  but fragile; a small HUD/state singleton with a proper header would remove the `extern`
  declarations scattered in .cpp files.
- **Magic numbers / timing constants**: 1000ms backup-camera delay (`callbacks.cpp:219`), ALSA
  buffer durations (`1000000`/`250000` µs literals in `audio.cpp`), `8`-wraparound counters for
  `navi_data->previous_msg` repeated 6+ times near-identically (`callbacks.cpp:664-711`) — good
  candidates for named constants and a small helper function to de-duplicate the "increment
  wrap at 8" pattern.
- **Logging**: mixes `printf`/`fprintf(stderr, ...)` directly with the project's own
  `logd`/`logw`/`loge` macros (`hu_uti.h`) inconsistently across files (e.g.
  `outputs.cpp` uses raw `fprintf`, `callbacks.cpp` mixes both) — minor but makes log-level
  filtering/log capture on-device less reliable.
- **No automated tests found** anywhere in the repo (no `test/` dir with unit tests beyond the
  `ubuntu` manual test harness and an install "`test`" CLI arg in `main.cpp:199` that merely
  checks the binary starts) — any refactor here is inherently risky without hardware-in-the-loop
  verification; worth considering a minimal protocol-level unit test harness around `hu_aap.cpp`'s
  message encode/decode paths (pure logic, no hardware dependency) as a first step before bigger
  refactors.
- **Protobuf schema maintenance**: `hu/hu.proto` is a manually maintained reverse-engineered copy
  of Google's internal AA schema (not vendored/generated from an upstream source-of-truth) —
  drift risk whenever Google changes the real protocol; worth tracking upstream aasdk/openauto's
  `.proto` for any newer message types (e.g. newer audio config fields) this project hasn't
  picked up yet.

## 6. Prioritized next steps

1. **Hardware probe (do this before any code)**: `dbus-monitor`/`d-feet`/`busctl` against the
   CMU's buses to determine (a) whether a BlueZ-compatible `org.bluez` D-Bus service exists for
   BT connection-state signals (blocks Feature 1 entirely), (b) whether `entertainmentMl`'s ALSA
   device exposes any mixer/volume control elements (`amixer -D entertainmentMl scontrols`)
   (blocks the volume-control half of Feature 3), (c) confirm JCI's `AudioManagement` D-Bus
   service supports any additional focus-type granularity beyond permanent/transient (via
   `dumpState`, already called in `populateStreamTable`).
2. **Quick win, no probing needed**: implement `AA_CH_AU2` dispatch in `AudioOutput`/
   `MazdaEventCallbacks::MediaPacket` — closes a real protocol gap with near-zero risk.
3. **Quick win, no probing needed**: refactor `MicThreadMain`'s per-iteration `new`/`delete[]`
   into a reused buffer, and audit the `temp_assembly_buffer`/USB `malloc` sites for RAII
   conversion (`std::vector`, `std::unique_ptr`) — reduces crash/leak surface on a resource
   constrained ARM target.
4. **If BlueZ-equivalent D-Bus confirmed (step 1a)**: implement a `BluetoothConnectionWatcher`
   class mirroring `AudioManagerClient`'s existing `DBus::ObjectProxy`/`Notify` pattern, wired
   into `mazda/main.cpp`'s outer connect loop to retrigger `hu_aap_start` on BT-connect — this is
   Feature 1's actual deliverable.
5. **Tie video-focus loss to an explicit (even if JCI-owned) audio cue** during backup-camera
   events — small addition to `VideoManagerClient`/`MazdaEventCallbacks` to request a transient
   audio focus/duck alongside the existing video-focus release, closing the Feature 2/3 overlap
   gap.
6. **Replace the fixed 1000ms backup-camera re-focus delay** with a verified signal if one exists
   on the JCI HMI bus, otherwise document it as an empirically-tuned constant with a named define.
7. **Longer-term / optional**: CMake migration for the build system, modernize to GStreamer 1.x
   and a current protobuf release (both are larger, higher-risk efforts requiring full on-target
   re-validation — do only after the above are stable, and only if continued vendor/toolchain
   support for 0.10/2.6.1 becomes a real blocker).
8. **Explicitly out of scope / low payoff given findings**: full USB-free "wireless AA over BT"
   bootstrap (Feature 1 extended) — treat as a separate, later, much higher-risk project if ever
   pursued; AA codec/sample-rate upgrades beyond what Google's protocol already allows — capped
   by the phone-side protocol and not something this codebase can meaningfully improve alone.
