# EspressoHID: KM — Complete Project Reference

> ESP32-S3 USB HID activity simulator ("keep-awake jiggler") that masquerades as a regular
> keyboard, performs subtle randomized HID actions to keep the host awake, and exposes a
> phone-friendly web dashboard for provisioning, control, and OTA firmware updates.

- **Firmware version (current):** `0.6.0-release`
- **Target hardware:** ESP32-S3 DevKitC (`esp32-s3-devkitc-1`)
- **Framework:** Arduino on PlatformIO (`espressif32` platform)
- **License:** see [LICENSE](../LICENSE)

Intended for authorized lab, kiosk, demo, and test environments where you want device-side
activity simulation **without installing any software on the host machine**. Use only on
systems you own or are explicitly authorized to manage.

---

## Table of Contents

1. [What It Does](#1-what-it-does)
2. [Repository Structure](#2-repository-structure)
3. [Hardware & Wiring](#3-hardware--wiring)
4. [Architecture Overview](#4-architecture-overview)
5. [Module Reference](#5-module-reference)
6. [How It Works — Runtime Behavior](#6-how-it-works--runtime-behavior)
7. [Action Catalog](#7-action-catalog)
8. [Usage Guide (Build, Flash, Provision)](#8-usage-guide-build-flash-provision)
9. [Button Controls & LED States](#9-button-controls--led-states)
10. [Web Dashboard](#10-web-dashboard)
11. [REST API Reference](#11-rest-api-reference)
12. [Configuration Reference](#12-configuration-reference)
13. [NVS Storage Schema](#13-nvs-storage-schema)
14. [Networking & Provisioning Flow](#14-networking--provisioning-flow)
15. [Human Input Timing](#15-human-input-timing)
16. [Debug vs Release Builds](#16-debug-vs-release-builds)
17. [Current Limitations](#17-current-limitations)
18. [Developer Quick Reference](#18-developer-quick-reference)

---

## 1. What It Does

The device enumerates over native USB as a standard keyboard (+ consumer-control device) and,
while "active", periodically fires low-impact HID events such as arrow-key movement, modifier
taps, media nudges, and optional typed text — all with human-like timing jitter and mostly
*net-zero* effects (e.g., volume up then down). This keeps the host machine from sleeping or
locking without installing anything host-side.

Key features:

- Randomized USB vendor/product identity **and serial number** per boot (Dell / HP / Logitech / Microsoft / Lenovo)
- Two runtime profiles: `ACTIVE` (frequent activity) and `MEETING` (gentle, infrequent)
- Weighted random action selection; per-action enable/disable and weight overrides at runtime
- Simulated breaks: random short naps plus longer breaks after sustained work sessions
- First-boot captive portal for Wi-Fi provisioning (`KBSetup-XXXX` SoftAP)
- Live browser dashboard: status, controls, settings, history, logs, manual triggers
- OTA firmware updates from the web UI (works even before Wi-Fi is configured)
- Persistent runtime configuration in NVS (survives reboots)
- Single physical button for start/stop, profile switching, and factory reset
- WS2812 RGB LED status feedback with multiple idle-state blink patterns

---

## 2. Repository Structure

```text
EspressoHID-KM/
├── platformio.ini              # PlatformIO build config (env: esp32-s3-devkitc-1)
├── src/
│   ├── main.cpp                # Firmware entry point: setup() + loop()
│   └── web_portal.cpp          # Captive portal, dashboard UI, REST API, OTA handlers
├── include/
│   ├── config.h                # ALL compile-time tunables (pins, timing, weights, identities)
│   ├── state.h                 # Global state variables + LED state-machine enums
│   ├── runtime_config.h        # NVS-backed runtime configuration API
│   ├── actions.h               # Action catalog API + ActionSource enum
│   ├── profiles.h              # Profile enum + interval accessors
│   ├── control.h               # High-level state transition API (active/sleep/profile)
│   ├── sleep_manager.h         # Sleep/break scheduling
│   ├── led_controller.h        # RGB LED rendering
│   ├── usb_identity.h          # Random USB VID/PID identity selection
│   ├── button_handler.h        # BOOT button debounce + gestures
│   ├── event_log.h             # In-RAM event log
│   ├── device_status.h         # Device status flags (for the web UI)
│   ├── human_input.h           # Human-like key timing helpers
│   └── web_portal.h            # Web server entry points
├── lib/
│   └── fake_keyboard_core/     # PlatformIO library wrapper so LDF links the core modules
│       ├── include/
│       │   └── fake_keyboard_core.h  # Umbrella header (LDF hook)
│       └── src/
│           ├── actions.cpp         # 12 built-in actions + weighted selection + history
│           ├── button_handler.cpp  # Debounce, short/long press, factory-reset hold
│           ├── control.cpp         # State transitions (start/stop/sleep/wake/profile)
│           ├── device_status.cpp   # Status flag helpers
│           ├── event_log.cpp       # Ring-buffer event log (RAM only, 40 entries)
│           ├── human_input.cpp     # tapKey/chordKey/typeTextHuman/backspaceHuman helpers
│           ├── led_controller.cpp  # LED mode → color/brightness/pulse rendering
│           ├── profiles.cpp        # Profile names + interval ranges (runtime-aware)
│           ├── runtime_config.cpp  # NVS persistence: WiFi creds, intervals, mask, weights
│           ├── sleep_manager.cpp   # Naps, long breaks, wake handling
│           ├── state.cpp           # Global variable definitions
│           └── usb_identity.cpp    # Picks a random USB identity + serial at boot
├── test/                       # Default PlatformIO test placeholder (no tests yet)
├── docs/                       # This documentation
├── EspressoHID-StayAwake.code-workspace
├── LICENSE
└── README.md
```

**Key architectural note:** Header files live in the top-level `include/`; implementations live
in `lib/fake_keyboard_core/src/`. The umbrella header `fake_keyboard_core.h` exists so
PlatformIO's Library Dependency Finder (LDF) pulls in and links those sources automatically.

---

## 3. Hardware & Wiring

### Required

- ESP32-S3 development board **with native USB** (e.g., ESP32-S3 DevKitC-1 N8)
- USB cable (power + flashing + acting as an HID device on the target machine)

### Pins used by the firmware

| Component | Pin | Purpose |
| --- | --- | --- |
| Button | `GPIO 0` (BOOT) | Start/stop, profile switch, factory reset |
| WS2812 RGB LED | `RGB_BUILTIN`, defaults to `GPIO 38` | Status feedback |
| Native USB | ESP32-S3 USB OTG | Keyboard + Consumer Control HID |
| Wi-Fi | Built-in radio | Captive portal, dashboard, REST API, OTA |

Notes:

- If your board has its RGB LED on GPIO 48 (some variants), override `RGB_BUILTIN` in
  `include/config.h`.
- In release builds, `ARDUINO_USB_MODE=1` and `ARDUINO_USB_CDC_ON_BOOT=0`: the device
  enumerates as **HID only**, with no CDC serial port exposed ("stealth" mode).

---

## 4. Architecture Overview

The firmware is a **single-threaded cooperative `loop()`** with no RTOS tasks of its own. Every
subsystem is polled each iteration:

```text
                 ┌───────────────────────────────────────────────────┐
                 │                    setup()                        │
                 │  1. pinMode(PIN_BUTTON, INPUT_PULLUP)            │
                 │  2. Set LED blue                                 │
                 │  3. BOOT_COUNTDOWN_S (2s) delay                  │
                 │  4. applyRandomIdentity() — VID/PID/serial       │
                 │  5. Keyboard.begin() / Consumer.begin() / USB.begin()│
                 │  6. webPortalSetup() — calls runtimeConfigBegin()│
                 └───────────────────────┬───────────────────────────┘
                                         ▼
                 ┌───────────────────────────────────────────────────┐
   each pass ─►  │                     loop()                        │
                 │                                                   │
                 │  1. webPortalLoop()                               │
                 │     • dns.processNextRequest() (AP mode only)    │
                 │     • server.handleClient()                      │
                 │     • staRetryLoop()                             │
                 │  2. handleButton()    user input gestures        │
                 │  3. updateLed()       LED state machine          │
                 │  4. handleSleep()     wake-up when timer ends    │
                 │  5. fire action?      if charged && due time     │
                 │       • performJiggle()  weighted pick + run     │
                 │       • maybeScheduleBreak()  naps / breaks      │
                 │  6. periodic DEBUG status line (debug builds)    │
                 └───────────────────────────────────────────────────┘
```

### Layering

| Layer | Files | Responsibility |
| --- | --- | --- |
| Entry point | `src/main.cpp` | Boot sequence, main scheduling loop |
| Web tier | `src/web_portal.cpp` | AP/STA Wi-Fi, captive portal, dashboard HTML, JSON API, OTA |
| Core library | `lib/fake_keyboard_core/src/*` | Actions, control transitions, sleep, LED, config, logging, button |
| Shared headers | `include/*.h` | Public APIs and `config.h` tunables |
| Arduino/ESP-IDF | `USBHIDKeyboard`, `USBHIDConsumerControl`, `WebServer`, `DNSServer`, `Update`, `ESPmDNS`, `Preferences` (NVS) | Platform services |

### State model

Global mutable state lives in `state.cpp` (declared in `include/state.h`). Key variables:

| Variable | Type | Purpose |
| --- | --- | --- |
| `isActive` | `bool` | Idle (blue LED, no actions) vs active (charging/acting) |
| `isSleeping` | `bool` | Dormant break window; suppresses actions until `sleepUntil` |
| `ledMode` | `LedMode` | Current LED state machine mode |
| `currentProfile` | `Profile` | `PROFILE_ACTIVE` (0) or `PROFILE_MEETING` (1) |
| `nextActionTime` | `unsigned long` | `millis()` timestamp when the next action should fire |
| `stagedInterval` | `unsigned long` | Next interval, applied after blink ends |
| `chargeStartTime` | `unsigned long` | When the current charge ramp began |
| `nextInterval` | `unsigned long` | Current interval duration |
| `nextLongBreakAt` | `unsigned long` | When the next long break is scheduled |
| `actionCount` | `unsigned long` | Actions performed in current session |
| `sleepUntil` | `unsigned long` | Wake-up timestamp |
| `sessionStartTime` | `unsigned long` | When current active session started |

**LED modes** (`LedMode` enum in `state.h`):

| Mode | Value | Description |
| --- | --- | --- |
| `LED_BOOT_BLUE` | 0 | Initial boot-up blue |
| `LED_IDLE_BLUE` | 1 | Waiting for button press (with status overlays) |
| `LED_ACTIVE_CHARGING` | 2 | Amber→green ramp while waiting for next action |
| `LED_ACTIVE_READY` | 3 | Brief bright pulse when charge is full |
| `LED_ACTIVE_BLINKING` | 4 | Fast green blink during/after an action |
| `LED_STOP_RED` | 5 | Red flash when deactivated |
| `LED_SLEEPING` | 6 | Dim yellow breathing while dormant |
| `LED_PROFILE_FLASH` | 7 | Brief color flash on profile change |

### Configuration flow

Two-tier configuration:

1. **Compile-time defaults** — every constant lives in `include/config.h` (pin numbers,
   intervals, weights, USB identity pool, sleep schedule, LED colors).
2. **Runtime overrides persisted in NVS** (`lib/fake_keyboard_core/src/runtime_config.cpp`) —
   Wi-Fi credentials, provisioned flag, per-profile min/max intervals, 32-bit action enable
   mask, per-action weights, custom text, Wi-Fi-disabled flag. Accessors like
   `profileIntervalMin()` / `actionWeightActive()` transparently prefer the NVS value and fall
   back to the compiled default.

Factory reset (button hold ≥10 s, or web request) clears the entire NVS namespace `"fk"` and
reboots into first-boot mode.

---

## 5. Module Reference

| Module | Key functions | Notes |
| --- | --- | --- |
| `actions.cpp` | `performJiggle()`, `performActionByIndex()`, `actionsCount()`, `actionName()`, `actionWeightActive()`, `actionWeightMeeting()`, `actionEnabled()`, `actionHistoryToJson()`, `actionHistoryClear()` | 12 built-in actions; weighted roulette respecting enable mask and profile; TypeText gets weight 0 unless custom text is set; keeps a **20-entry** RAM history ring buffer |
| `control.cpp` | `controlSetActive()`, `controlToggleActive()`, `controlSetProfile()`, `controlSleepNow()`, `controlWakeNow()`, `controlReboot()` | All state transitions funnel through here. On activate: resets `actionCount`, schedules long break, starts charging. On deactivate: releases all keys/consumer, sets red LED. `controlSetProfile()` wakes from sleep if sleeping. `controlSleepNow()` only works if `isActive`. `controlWakeNow()` only works if `isSleeping`. |
| `sleep_manager.cpp` | `enterSleep()`, `handleSleep()`, `maybeScheduleBreak()` | Long break if work session exceeded (`WORK_SESSION_*`), else `SLEEP_CHANCE_PER_ACTION`% chance of a 1–3 min nap |
| `button_handler.cpp` | `handleButton()` | 50 ms debounce; short press = toggle; hold ≥1.5 s = next profile; hold ≥10 s = factory reset + reboot |
| `led_controller.cpp` | `updateLed()`, `setLed()`, `setChargeLed()`, `ledManualBlinkGreen()` | Renders the current `LedMode`. **Manual blink overlay** takes priority over all modes (used for web-triggered actions). Charging uses gamma curve. Idle mode overlays status blink patterns. |
| `profiles.cpp` | `profileName()`, `profileFirstMin()`, `profileFirstMax()`, `profileIntervalMin()`, `profileIntervalMax()`, `scheduleNextLongBreak()` | Reads NVS overrides before compile-time defaults |
| `runtime_config.cpp` | `runtimeConfigBegin()`, `runtimeConfigGet()`, `runtimeConfigHasWifiCredentials()`, getters/setters, `runtimeConfigFactoryReset()` | NVS namespace `"fk"`. Interval clamping: [250ms, 3600000ms], auto-swap if min>max. Default `actionEnabledMask` = `0xFFFFFFFF` (all enabled). |
| `usb_identity.cpp` | `applyRandomIdentity()` | Seeds RNG from `analogRead(1) ^ (micros() * 2654435761UL) ^ esp_random()`. Sets VID/PID/manufacturer/product from pool + generates random 16-hex-char serial number. Called **before** `USB.begin()`. |
| `human_input.cpp` | `humanDelayMs()`, `tapKey()`, `chordKey()`, `typeTextHuman()`, `backspaceHuman()`, `tapConsumer()` | WPM-derived random delays with jitter. See §15 for formulas. |
| `event_log.cpp` | `eventLogAdd()`, `eventLogToJson()`, `eventLogClear()` | **40-entry** ring buffer. Uses NTP timestamps when available (`time(nullptr) > 1700000000`), otherwise `millis()` prefix. |
| `device_status.cpp` | `deviceStatusSet()`, `deviceStatusGet()` | Volatile `DeviceSetupStatus` flag read by LED controller for idle overlays. |
| `web_portal.cpp` | `webPortalSetup()`, `webPortalLoop()` | See §10–14. `webPortalSetup()` calls `runtimeConfigBegin()` first. |

---

## 6. How It Works — Runtime Behavior

### Boot sequence

1. `setup()` configures button pin (`INPUT_PULLUP`), sets LED blue.
2. Waits `BOOT_COUNTDOWN_S` seconds (default **2**) so the OS doesn't catch mid-reboot enumeration.
3. `applyRandomIdentity()` — one USB identity from the pool is chosen randomly; a random 16-hex-char serial is generated.
4. `Keyboard.begin()`, `Consumer.begin()`, `USB.begin()` — enumerate as HID.
5. `webPortalSetup()` is called, which calls `runtimeConfigBegin()` to load NVS config, then decides Wi-Fi mode.
6. Enters idle (blue LED with status overlays).

### The activity cycle (when ACTIVE)

1. On activation (`controlSetActive(true)`):
   - Resets `actionCount = 0`, `sessionStartTime = now`, `isSleeping = false`
   - Sets `ledMode = LED_ACTIVE_CHARGING`, `chargeStartTime = now`
   - Draws first interval from `profileFirstMin()`–`profileFirstMax()` (shorter range)
   - Sets `nextActionTime = chargeStartTime + nextInterval`
   - Calls `scheduleNextLongBreak()` to set `nextLongBreakAt`
2. While waiting, the LED "charges": amber→green ramp over the interval using gamma curve.
3. When `millis() >= nextActionTime`:
   - LED switches to `LED_ACTIVE_BLINKING` (fast green blink for `ACTION_BLINK_MS` = 1000 ms)
   - `performJiggle()` picks an enabled action via weighted roulette for the current profile and executes it
   - The *next* interval is drawn from the profile's steady-state range and stored in `stagedInterval`
   - `nextActionTime = 0` (prevents re-trigger during blink)
   - `maybeScheduleBreak()` may enter a short nap (8% chance, 1–3 min) or, if `now >= nextLongBreakAt`, a long break (3–7 min) and reschedules the next long break
4. After blink ends (`LED_ACTIVE_BLINKING` → `LED_ACTIVE_CHARGING`):
   - `nextInterval = stagedInterval`, `stagedInterval = 0`
   - `nextActionTime = now + nextInterval`
   - Charge ramp restarts
5. Sleep suppresses actions until `millis() >= sleepUntil` (yellow breathing LED); waking restarts the charge cycle.

### Net-zero design

Most actions are deliberately reversible so they don't disturb the host state:

- **ArrowScroll**: presses N times one way, then N times the other way (net-zero scroll)
- **Volume/Brightness**: increment then decrement
- **CapsLock/NumLock**: toggle twice (on then off)
- **AltTab**: switches windows, dwells, then Alt+Shift+Tab back
- **TypeText**: types the configured string then backspaces it away
- **WinArrow**: opens/closes Windows Start menu (net-zero)

---

## 7. Action Catalog

Selection is weighted per profile (weights are runtime-editable; defaults below):

| Index | Name | Active weight | Meeting weight | Effect (key sequence) |
| --- | --- | --- | --- | --- |
| 0 | `ArrowScroll` | 35 | 35 | 80% vertical / 20% horizontal. Presses `N` times one direction (1–8 presses), pauses 220–900 ms, presses `N` times the other direction. Each press: hold 40–120 ms, gap 60–500 ms. |
| 1 | `AltTab` | 10 | 0 | Alt+Tab (switch window), dwell 500–1500 ms, Alt+Shift+Tab (return). Human delays between key presses. |
| 2 | `Volume` | 3 | 3 | Consumer control: volume increment (hold 35–85 ms), delay 80–200 ms, volume decrement. |
| 3 | `Brightness` | 3 | 3 | Consumer control: brightness increment (hold 35–85 ms), delay 80–200 ms, brightness decrement. |
| 4 | `CapsToggle` | 5 | 5 | CapsLock on (hold 45–95 ms), delay 150–400 ms, CapsLock off (hold 45–95 ms). |
| 5 | `NumLockToggle` | 5 | 5 | NumLock on (hold 45–95 ms), delay 200–500 ms, NumLock off (hold 45–95 ms). |
| 6 | `ShiftTap` | 15 | 30 | Left Shift press (hold 25–70 ms), gap 60–160 ms. Completely invisible. |
| 7 | `WinArrow` | 10 | 0 | Left GUI (Windows key) press (hold 35–80 ms), delay 180–300 ms, press again (hold 35–80 ms). Opens/closes Start menu. |
| 8 | `TypeText` | 2 | 0 | Types `customText` (if set) at human WPM, delay 120–420 ms, then backspaces each character. **Skipped if `customText` is empty.** |
| 9 | `CtrlTap` | 10 | 10 | Left Ctrl press (hold 25–70 ms), gap 60–160 ms. |
| 10 | `WinSearch` | 3 | 0 | Win+S chord (each key 20–60 ms gap, hold 40–120 ms), delay 220–700 ms, ESC (hold 25–70 ms). |
| 11 | `EmojiPeek` | 2 | 0 | Win+. chord, delay 220–700 ms, ESC. |

**Selection rules:**

- Higher weight = more likely to be picked.
- Actions can be individually disabled at runtime via the 32-bit `actionMask` (bit `i` = action `i`).
- Default `actionMask` = `0xFFFFFFFF` (all enabled).
- If all effective weights are zero, a `ShiftTap` is performed as a safety fallback.
- `TypeText` automatically gets weight 0 if `customText` is empty.
- Weights are read from NVS if `weightsConfigured == true`; otherwise compile-time defaults are used.

---

## 8. Usage Guide (Build, Flash, Provision)

### Build & flash

```bash
# Install PlatformIO (VS Code extension, or CLI):
pip install platformio

# Build
pio run -e esp32-s3-devkitc-1

# Flash (upload speed is intentionally lowered to 115200 for stability)
pio run -e esp32-s3-devkitc-1 -t upload
```

Serial monitor (debug builds only):

```bash
pio device monitor -e esp32-s3-devkitc-1
```

### First boot / provisioning

1. Power the board. With no stored credentials it starts a SoftAP captive portal named
   `KBSetup-XXXX` (suffix = last 4 hex digits of ESP32 MAC, uppercase).
2. Join that AP from a phone/laptop. The captive portal should pop up automatically; otherwise
   browse to `http://192.168.4.1/`.
3. On the setup page you can:
   - enter Wi-Fi SSID (max 32 chars) / password (max 64 chars),
   - check **Disable Wi-Fi features** to permanently disable networking until factory reset,
   - edit per-profile min/max intervals (in seconds),
   - toggle which actions are enabled,
   - upload new firmware (OTA works here too).
4. Save → the device stores config in NVS, sets `provisioned = true`, and reboots.

### Day-to-day use

- Press BOOT once to start/stop activity.
- Open `http://fakekeyboard.local/` (mDNS) or `http://<device-ip>/` for the full dashboard:
  start/stop, profile switching, sleep/wake, intervals, weights, custom text, manual action
  triggering, history, logs, OTA.

---

## 9. Button Controls & LED States

### Button gestures

| Gesture | Result |
| --- | --- |
| Short press (< 1.5 s) | Toggle active/idle (`controlToggleActive()`) |
| Hold ≥ 1.5 s (`LONG_PRESS_MS`) | Cycle to next profile (`controlSetProfile()`) |
| Hold ≥ 10 s (`FACTORY_RESET_HOLD_MS`) | Factory reset (clears NVS) and reboot |

### LED language

#### Active modes (when `isActive == true`)

| Color / pattern | Mode | Meaning |
| --- | --- | --- |
| Amber → green ramp | `LED_ACTIVE_CHARGING` | Charging toward the next scheduled action. Uses `powf(progress, 2.2)` gamma curve. Color transitions from amber (red=30, green=low) to green (red=0, green=high). |
| Bright green pulse | `LED_ACTIVE_READY` | Charge full — action imminent. Sine-based brightness modulation from `LED_GREEN_CHARGE_MAX` (60) to `CHARGE_READY_BRIGHTNESS` (80) over `CHARGE_READY_PULSE_MS` (400 ms). |
| Fast green blink (~1 s) | `LED_ACTIVE_BLINKING` | An action just ran. Toggles every `BLINK_TOGGLE_MS` (100 ms). After blink ends, transitions to `LED_ACTIVE_CHARGING` with `stagedInterval`. |
| Purple flash | `LED_PROFILE_FLASH` | Profile changed to `ACTIVE`. Duration: `PROFILE_INDICATOR_MS` (800 ms). |
| Cyan flash | `LED_PROFILE_FLASH` | Profile changed to `MEETING`. Duration: `PROFILE_INDICATOR_MS` (800 ms). |
| Yellow breathing | `LED_SLEEPING` | Sleeping / taking a break. Sine wave breathing over `LED_SLEEP_PULSE_MS` (1000 ms) cycle. Colors: `LED_SLEEP_R` (30), `LED_SLEEP_G` (20). |

#### Idle modes (when `isActive == false`)

| Color / pattern | Meaning |
| --- | --- |
| Blue (dim, `LED_BLUE_IDLE` = 10) | Idle / boot complete |
| Red hold (~2 s, `LED_RED_STOP` = 20) | Just stopped (`LED_STOP_RED` mode) |

#### Idle status overlays (when `ledMode == LED_IDLE_BLUE`)

The idle blue LED is overlaid with status blink patterns based on `DeviceSetupStatus`:

| Status | Pattern | Colors |
| --- | --- | --- |
| `DEV_OK` | Occasional green blip every 8 s (on for 120 ms) | R=0, G=30, B=0 |
| `DEV_NEEDS_SETUP` | Double purple blink every 3 s | R=30, G=0, B=40 |
| `DEV_WIFI_CONNECTING` | Amber slow blink (on for 650 ms of 3 s cycle) | R=30, G=18, B=0 |
| `DEV_WIFI_OFF` | Single red blink (on for 250 ms of 3 s cycle) | R=35, G=0, B=0 |
| `DEV_WIFI_DISABLED` | Triple red blink (on for 120+120+120 ms of 3 s cycle) | R=35, G=0, B=0 |

#### Manual blink overlay

`ledManualBlinkGreen(durationMs)` provides immediate feedback for web-triggered actions without
disturbing the ACTIVE scheduling state machine. This overlay takes **priority over all other
LED modes** while active.

---

## 10. Web Dashboard

Served by an embedded `WebServer` on port 80. The same code serves two different front ends:

### Setup portal (AP mode)

- Served at `/` and all unknown URLs (for captive-portal probe compatibility)
- Wi-Fi provisioning form (SSID/password)
- Interval editing (per-profile min/max in seconds)
- Action enable checkboxes
- OTA firmware upload
- Link to `/ui` for companion UI without saving Wi-Fi

**Captive portal probe URLs handled** (all return the portal HTML):

| URL | OS |
| --- | --- |
| `/generate_204`, `/gen_204`, `/redirect` | Android |
| `/hotspot-detect.html`, `/library/test/success.html` | Apple iOS/macOS |
| `/canonical.html` | Apple |
| `/ncsi.txt`, `/connecttest.txt`, `/fwlink` | Windows |
| `/success.txt`, `/wpad.dat` | Various |

### Companion UI (STA mode)

Served at `/`. `/ui` redirects to `/`.

A single-page app with:

- **Status card**: Wi-Fi mode/IP, FW version, mDNS hostname, state (ACTIVE/IDLE), profile, sleep countdown, action count, next action countdown
- **Controls**: start/stop, wake, profile select, timed sleep (seconds), reboot, factory reset
- **Settings**: custom text (max 128 chars), per-profile intervals (seconds), per-action enable + Active/Meeting weights (0–255)
- **Manual action runner**: trigger any enabled action by index
- **History table**: recent 20 actions with timestamp, name, source (auto/manual)
- **Log viewer**: in-RAM event log (40 entries)
- **OTA firmware upload**

**Frontend technology:** The companion UI tries to load Vue 3 from `unpkg.com` with a 1400 ms
timeout. If the CDN is unreachable (offline networks), it falls back to a bundled vanilla-JS
template with identical functionality.

**Refresh cadence:** The dashboard auto-adapts its polling interval:

| Condition | Refresh interval |
| --- | --- |
| Wi-Fi mode = STA (connected) | 500 ms |
| Otherwise (AP, connecting, etc.) | 2000 ms |
| After repeated fetch errors | 5000 ms |

---

## 11. REST API Reference

All endpoints are plain HTTP (no auth, no HTTPS). Form endpoints accept
`application/x-www-form-urlencoded`; OTA is `multipart/form-data`.

### GET endpoints

| Endpoint | Returns |
| --- | --- |
| `/api/status` | `{active: bool, sleeping: bool, sleepRemainingMs: number, profileId: number, profile: string, actionCount: number, nextInMs: number, fwVersion: string, mdns: string, wifi: {mode: string, ssid: string, ip: string}}`. `wifi.mode` ∈ `AP \| STA \| STA_CONNECTING \| OFF \| DISCONNECTED` |
| `/api/config` | `{provisioned: bool, wifiConfigured: bool, profiles: [{id, name, minMs, maxMs}], actionMask: number, customText: string, actions: [{id, name, enabled, wA, wM}]}` |
| `/api/history` | `[{ms: number, name: string, src: number}]` — `src`: 1 = manual, 0 = auto. RAM-only, resets on reboot. |
| `/api/logs` | `string[]` — Event-log ring buffer (40 entries). Uses NTP timestamps when available. |

### POST endpoints

| Endpoint | Fields | Purpose |
| --- | --- | --- |
| `/api/config` | `p{N}MinMs`, `p{N}MaxMs` (or second-based `p{N}MinS`/`p{N}MaxS`), `actionMask`, `wA{i}`, `wM{i}`, `customText` | Persist intervals, enabled-actions bitmask, per-action weight overrides, custom typing text. **Weight seeding:** When the first weight is set via web, the entire 32-entry weight table is initialized from compile-time defaults to prevent unset entries from becoming 0. |
| `/api/control` | `active` (bool), `profile` (int), `sleepS`/`sleepMs` (duration), `wake` (bool), `clearHistory` (bool), `factoryReset` (bool), `reboot` (bool) | Runtime control. `factoryReset` and `reboot` respond with `{"ok":true,"restarting":true}` then restart the ESP32 after 300 ms delay. |
| `/api/trigger` | `id` (int) | Manually execute one action by index. Returns 400 if invalid or disabled. |
| `/api/ota` | multipart field `firmware` | Stream a .bin to the Update partition using ESP-IDF `Update` library. Reboots on success. |

**Examples:**

```bash
# Start activity
curl -X POST http://fakekeyboard.local/api/control -d 'active=1'

# Switch to MEETING profile
curl -X POST http://fakekeyboard.local/api/control -d 'profile=1'

# Trigger ShiftTap (index 6)
curl -X POST http://fakekeyboard.local/api/trigger -d 'id=6'

# Sleep for 60 seconds
curl -X POST http://fakekeyboard.local/api/control -d 'sleepS=60'

# Wake immediately
curl -X POST http://fakekeyboard.local/api/control -d 'wake=1'

# Get status
curl http://fakekeyboard.local/api/status

# Update custom text
curl -X POST http://fakekeyboard.local/api/config -d 'customText=Hello World'

# Factory reset
curl -X POST http://fakekeyboard.local/api/control -d 'factoryReset=1'
```

---

## 12. Configuration Reference

Everything is centralized in [`include/config.h`](../include/config.h). Highlights:

| Group | Constants | Defaults |
| --- | --- | --- |
| Version | `FIRMWARE_VERSION` | `"0.6.0-release"` (override via `-D FIRMWARE_VERSION=\"x.y.z\"`) |
| Pins | `PIN_BUTTON`, `RGB_BUILTIN` | GPIO 0, GPIO 38 |
| Profiles | `ACTIVE_INTERVAL_MIN/MAX_MS` | 10 000–60 000 ms |
| | `MEETING_INTERVAL_MIN/MAX_MS` | 45 000–180 000 ms |
| First action | `ACTIVE_FIRST_MIN/MAX_MS` | 1000–3000 ms |
| | `MEETING_FIRST_MIN/MAX_MS` | 3000–8000 ms |
| Short naps | `SLEEP_CHANCE_PER_ACTION` | 8 (%) |
| | `SLEEP_MIN/MAX_MS` | 60 000–180 000 ms (1–3 min) |
| Long breaks | `WORK_SESSION_MIN/MAX_MS` | 1 200 000–2 400 000 ms (20–40 min) |
| | `LONG_BREAK_MIN/MAX_MS` | 180 000–420 000 ms (3–7 min) |
| Arrows | `ARROW_PRESS_MIN/MAX` | 1–8 presses |
| | `ARROW_HOLD_MIN/MAX_MS` | 40–120 ms |
| | `ARROW_GAP_MIN/MAX_MS` | 60–500 ms |
| | `ARROW_REVERSE_CHANCE` | 30 (%) |
| | `ARROW_REVERSE_RATIO` | 3 |
| Typing | `TYPE_WPM_MIN/MAX` | 60–120 WPM |
| Weights | `WEIGHT_ARROW_SCROLL` … `WEIGHT_WIN_ARROW` | See §7 |
| LED brightness | `LED_BLUE_IDLE`, `LED_RED_STOP`, `LED_GREEN_CHARGE_MIN/MAX` | 10, 20, 0–60 |
| LED charge | `CHARGE_CURVE_GAMMA` | 2.2 |
| | `CHARGE_RED_START/END` | 30 → 0 (amber to green) |
| | `CHARGE_READY_PULSE_MS`, `CHARGE_READY_BRIGHTNESS` | 400 ms, 80 |
| LED sleep | `LED_SLEEP_R/G`, `LED_SLEEP_PULSE_MS` | 30/20, 1000 ms |
| LED timing | `ACTION_BLINK_MS`, `BLINK_TOGGLE_MS`, `STOP_RED_HOLD_MS` | 1000 / 100 / 2000 ms |
| Boot | `BOOT_COUNTDOWN_S` | 2 s |
| Button | `BUTTON_DEBOUNCE_MS`, `LONG_PRESS_MS`, `FACTORY_RESET_HOLD_MS` | 50 / 1500 / 10 000 ms |
| Profile | `PROFILE_INDICATOR_MS` | 800 ms |
| Debug | `DEBUG_MODE`, `STATUS_LOG_INTERVAL_MS` | Disabled; 10 000 ms |
| USB pool | `USB_IDENTITIES[]` | 5 identities (see below) |

**USB identity pool:**

| Index | VID | PID | Manufacturer | Product |
| --- | --- | --- | --- | --- |
| 0 | 0x413C | 0x2107 | Dell | Dell USB Entry Keyboard |
| 1 | 0x03F0 | 0x034A | HP | HP Elite USB Keyboard |
| 2 | 0x046D | 0xC31C | Logitech | USB Keyboard |
| 3 | 0x045E | 0x07F8 | Microsoft | Wired Keyboard 600 |
| 4 | 0x17EF | 0x608D | Lenovo | ThinkPad USB Keyboard |

Compile-time defaults can be overridden per build through `-D` build flags; runtime changes go
through the dashboard/NVS and win after the first save.

---

## 13. NVS Storage Schema

Runtime configuration is persisted in ESP32 NVS using the `Preferences` library under
namespace `"fk"`.

| Key | Type | Description | Default |
| --- | --- | --- | --- |
| `prov` | `bool` | Provisioned flag | `false` |
| `wdis` | `bool` | Wi-Fi disabled flag | `false` |
| `ssid` | `string` | Wi-Fi SSID (max 32 chars) | `""` |
| `pass` | `string` | Wi-Fi password (max 64 chars) | `""` |
| `p0Min` | `uint32` | Profile 0 (ACTIVE) min interval (ms) | `ACTIVE_INTERVAL_MIN_MS` |
| `p0Max` | `uint32` | Profile 0 (ACTIVE) max interval (ms) | `ACTIVE_INTERVAL_MAX_MS` |
| `p1Min` | `uint32` | Profile 1 (MEETING) min interval (ms) | `MEETING_INTERVAL_MIN_MS` |
| `p1Max` | `uint32` | Profile 1 (MEETING) max interval (ms) | `MEETING_INTERVAL_MAX_MS` |
| `aMask` | `uint32` | Action enabled bitmask (bit `i` = action `i`) | `0xFFFFFFFF` |
| `wcfg` | `bool` | Weights configured flag | `false` |
| `wAct` | `bytes[32]` | Per-action Active weights | Compile-time defaults |
| `wMet` | `bytes[32]` | Per-action Meeting weights | Compile-time defaults |
| `txt` | `string` | Custom text for TypeText action (max 128 chars) | `""` |

**Validation rules:**

- Intervals are clamped to [250 ms, 3 600 000 ms] (0.25 s to 1 hour).
- If `min > max`, they are swapped.
- If stored intervals are invalid (min > max after load), defaults are restored.
- Weight tables are validated by size; if corrupt, `weightsConfigured` is set to `false`.

**Factory reset:** `runtimeConfigFactoryReset()` calls `prefs.clear()` which erases all keys
in the `"fk"` namespace, then reloads defaults.

---

## 14. Networking & Provisioning Flow

```text
webPortalSetup()
    │
    ├─► runtimeConfigBegin() — load NVS config
    │
    ├─► runtimeConfigWifiDisabled() == true?
    │       └─► YES: set DEV_WIFI_DISABLED, return (no web UI until factory reset)
    │
    ├─► runtimeConfigHasWifiCredentials() == false?
    │       └─► YES: startApPortal()
    │               • WiFi.mode(WIFI_AP)
    │               • SSID = "KBSetup-" + last 4 hex of MAC (uppercase)
    │               • IP = 192.168.4.1, gateway = 192.168.4.1
    │               • DNS wildcard → portal page
    │               • set DEV_NEEDS_SETUP
    │               • Serve setup portal at /
    │
    └─► YES (has credentials): startStaIfPossible()
            │
            ├─► WiFi.mode(WIFI_STA), setHostname("fakekeyboard")
            ├─► WiFi.begin(ssid, pass)
            ├─► Wait up to 10 s for connection
            │
            ├─► Connected?
            │       └─► YES: configTime() for NTP, mdnsStart(), set DEV_OK
            │               Serve companion UI at /
            │
            └─► NO: return (retry loop in webPortalLoop)
                    │
                    └─► staRetryLoop() (called every loop iteration)
                            • If staAttempts >= 10: WiFi.disconnect(), WIFI_OFF,
                              set DEV_WIFI_OFF, stop server (no more retries this boot)
                            • Else: retry every 60 s, wait up to 8 s per attempt
                            • On success: bring up UI, mDNS, NTP
```

**Key details:**

- mDNS service `http.tcp` on port 80 under host `fakekeyboard` → `fakekeyboard.local`
- NTP servers: `pool.ntp.org`, `time.nist.gov` (best-effort sync once STA-connected)
- AP mode uses fixed IP `192.168.4.1` with a wildcard DNS responder (`dns.start(53, "*", ...)`)
  to trigger captive-portal detection across Android/iOS/Windows probe URLs
- **No automatic AP fallback** after provisioning: if STA fails 10 times, Wi-Fi is disabled
  for the rest of that boot. Recovery requires a reboot (with AP in range) or factory reset.

---

## 15. Human Input Timing

All actions use humanized timing to avoid robotic patterns. The helpers in `human_input.cpp`:

### `humanDelayMs(wpmMin, wpmMax)`

Computes a delay based on words-per-minute typing speed:

```cpp
const int wpm = random(wpmMin, wpmMax + 1);
const long base = 60000L / (wpm * 5L);          // ms per character
const long jitter = random(-base / 4, base / 4); // ±25% jitter
return max(20L, base + jitter);                  // minimum 20 ms
```

### `tapKey(keyboard, key, holdMin, holdMax, gapMin, gapMax)`

```cpp
keyboard.press(key);
delay(random(holdMin, holdMax + 1));
keyboard.releaseAll();
delay(random(gapMin, gapMax + 1));
```

### `chordKey(keyboard, keys[], keyCount, downGapMin, downGapMax, holdMin, holdMax)`

Presses multiple keys in sequence with gaps between each press, holds, then releases all:

```cpp
for (size_t i = 0; i < keyCount; i++) {
  keyboard.press(keys[i]);
  delay(random(downGapMin, downGapMax + 1));
}
delay(random(holdMin, holdMax + 1));
keyboard.releaseAll();
```

### `typeTextHuman(keyboard, text, wpmMin, wpmMax)`

Types each character with `humanDelayMs()` between them. **4% chance** of an extra micro-pause
(120–320 ms) after each character to simulate human hesitation.

### `backspaceHuman(keyboard, count)`

Backspaces `count` times using `tapKey()` with hold 25–65 ms, gap 30–110 ms. **5% chance** of
an extra delay (80–200 ms) between backspaces.

### `tapConsumer(consumer, code, holdMin, holdMax)`

```cpp
consumer.press(code);
delay(random(holdMin, holdMax + 1));
consumer.release();  // Note: release(), not releaseAll()
```

---

## 16. Debug vs Release Builds

Release/stealth defaults (`platformio.ini`):

- `ARDUINO_USB_MODE=1`, `ARDUINO_USB_CDC_ON_BOOT=0` — HID only, **no serial port exposed**.
- All `DEBUG_PRINT*` macros compile to nothing unless `-D DEBUG_MODE` is defined.

For development:

1. Uncomment `-D DEBUG_MODE` in `platformio.ini`.
2. Change `ARDUINO_USB_CDC_ON_BOOT` to `1`.
3. Rebuild and flash; monitor with `pio device monitor -e esp32-s3-devkitc-1`.

Debug builds emit:

- Boot steps (`[BOOT]`)
- USB identity selection (`[USB]`)
- State transitions (`[STATE]`, `[PROFILE]`, `[SLEEP]`)
- Action selections (`[JIGGLE]`, `[ACT]`)
- LED transitions (`[LED]`)
- Wi-Fi events (`[WIFI]`)
- Periodic status line every `STATUS_LOG_INTERVAL_MS` (10 s)

---

## 17. Current Limitations

- **No automated tests** yet (`test/README` is still the PlatformIO placeholder).
- **Web interface is HTTP-only**: no authentication, HTTPS, or rate limiting — treat the
  dashboard as LAN-trusted only.
- **Several actions are Windows-oriented**: WinSearch, EmojiPeek, WinArrow.
- **History and event logs are in-RAM only** and lost on reboot (20 actions, 40 log entries).
- **No automatic AP fallback** once STA credentials exist; recovery requires waiting out
  retries (up to 10 min), a reboot with the AP in range, or a factory reset (button hold ≥ 10 s).
- **Disabling Wi-Fi** during provisioning makes the web UI unavailable until factory reset.
- **Interval validation**: min/max are clamped to [250 ms, 1 hour]; invalid stored values
  trigger a reset to defaults.

---

## 18. Developer Quick Reference

### Adding a new action

1. Add the action function in `lib/fake_keyboard_core/src/actions.cpp`:
   ```cpp
   static void actionMyAction() {
     DEBUG_PRINTLN("[ACT] MyAction");
     // ... your HID code using Keyboard/Consumer ...
   }
   ```

2. Add it to the `ACTIONS[]` array with Active/Meeting weights:
   ```cpp
   {actionMyAction, 10, 5, "MyAction"},
   ```

3. The action index is now stable. Update `ACTION_COUNT` automatically via `sizeof`.

4. Optionally add compile-time weight constants in `config.h`:
   ```cpp
   #define WEIGHT_MY_ACTION 10
   ```

5. Rebuild and flash. The action will appear in the web UI automatically.

### Adding a new profile

1. Add to the `Profile` enum in `config.h`:
   ```cpp
   enum Profile : uint8_t {
     PROFILE_ACTIVE = 0,
     PROFILE_MEETING,
     PROFILE_NEW,      // <-- add here
     PROFILE_COUNT
   };
   ```

2. Add interval constants:
   ```cpp
   #define NEW_FIRST_MIN_MS 2000
   #define NEW_FIRST_MAX_MS 5000
   #define NEW_INTERVAL_MIN_MS 20000
   #define NEW_INTERVAL_MAX_MS 90000
   ```

3. Update `profiles.cpp` to handle the new profile in `profileFirstMin/Max()` and
   `profileName()`.

4. Update `runtime_config.cpp` `loadOrDefaults()` to set defaults for the new profile.

5. Rebuild and flash.

### Changing USB identities

Edit the `USB_IDENTITIES[]` array in `config.h`. Update `USB_IDENTITY_COUNT` automatically
via `sizeof`.

### Tuning LED behavior

All LED colors, brightness levels, and timing constants are in `config.h` under
**LED Brightness** and **Timing — LED Effects**.

### Accessing runtime config from code

```cpp
const RuntimeConfig& cfg = runtimeConfigGet();

// Read current profile interval
uint32_t minMs = runtimeConfigProfileIntervalMinMs(currentProfile);

// Check if an action is enabled
bool enabled = actionEnabled(actionIndex);

// Get custom text
const char* text = runtimeConfigCustomText();
```

### Triggering actions manually from code

```cpp
// By index (returns false if invalid or disabled)
performActionByIndex(6, ACTION_SRC_MANUAL);  // ShiftTap

// Weighted random (respects profile and enable mask)
performJiggle();
```

### State transitions from code

```cpp
controlSetActive(true);           // Start activity
controlSetActive(false);          // Stop activity
controlSetProfile(PROFILE_MEETING); // Switch profile
controlSleepNow(60000, "Manual"); // Sleep for 60s
controlWakeNow();                 // Wake immediately
```
