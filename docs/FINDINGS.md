# EspressoHID: KM — Codebase Findings & Analysis

> Comprehensive review of the current firmware (v0.6.0-release) identifying strengths,
> architectural patterns, gaps, and optimization opportunities to inform future expansion.

---

## Executive Summary

The EspressoHID: KM firmware is a well-structured, compact (~2,400 lines) ESP32-S3 HID
activity simulator with solid foundations for expansion. The codebase demonstrates good
separation of concerns, persistent configuration, and human-like timing simulation. However,
several architectural patterns present both opportunities and challenges for adding real-time
remote control capabilities.

**Key findings:**

- ✅ **Strengths:** Clean modular architecture, NVS persistence, humanized timing, stealth USB identity
- ⚠️ **Gaps:** No mouse HID, blocking delays in actions, HTTP-only (no WebSocket), single-threaded loop
- 🚀 **Opportunities:** Dual-core ESP32-S3, non-blocking HID reports, WebSocket for low-latency control

---

## 1. Codebase Metrics

> Metrics measured on `dev` branch (commit `c71557a`) via `wc -l` and `grep -rn`.

| Metric | Value | How measured |
| --- | --- | --- |
| Total firmware lines (`src/`+`include/`+`lib/`+`platformio.ini`) | **2,867** | `wc -l src/*.cpp include/*.h lib/... platformio.ini` |
| Header files (`include/` + `lib/.../include/`) | **15** (14 in `include/` + 1 umbrella `fake_keyboard_core.h`) | `ls include/*.h` |
| Implementation files (`src/` + `lib/.../src/`) | **14** (2 in `src/` + 12 in `lib/fake_keyboard_core/src/`) | `ls src/*.cpp lib/.../src/*.cpp` |
| Built-in actions | **12** | `ACTIONS[]` in `lib/fake_keyboard_core/src/actions.cpp:215` |
| Runtime profiles | **2** (`PROFILE_ACTIVE`, `PROFILE_MEETING`) | `enum Profile` in `include/config.h:54` |
| NVS keys | **13** (`prov`, `wdis`, `ssid`, `pass`, `p0Min`/`p0Max`/`p1Min`/`p1Max`, `aMask`, `wcfg`, `wAct`, `wMet`, `txt`) | `lib/fake_keyboard_core/src/runtime_config.cpp` NVS namespace `"fk"` |
| Blocking `delay()` calls (total) | **33–35** | `grep -rn "delay(" src/ lib/` |
| ↳ in action path (`actions.cpp` + `human_input.cpp`) | **22** (15 in `actions.cpp`, 7 in `human_input.cpp`) | — |
| ↳ in web / setup / control path | **11** (7 in `web_portal.cpp`, 2 in `main.cpp`, 1 in `button_handler.cpp`, 1 in `control.cpp`) | — |
| HID devices exposed | **2** (`USBHIDKeyboard`, `USBHIDConsumerControl`) | `src/main.cpp:25-26` — no `USBHIDMouse` yet |
| Network stack | HTTP only — `WebServer` on port 80 + `DNSServer` on 53; **no** `WebSocketsServer` | `platformio.ini:31` has only `lib_deps = USB`; `src/web_portal.cpp:22-24` |

> **Correction vs prior draft:** earlier revision quoted 2,369 lines and 27 delays in action code.
> Accurate counts are 2,867 total / 2,836 without `platformio.ini`, and 22 delays in the action path
> (33–35 total including networking/setup).

---

## 2. Architectural Strengths

### 2.1 Clean Modular Design

The firmware follows a clear layered architecture:

```
┌─────────────────────────────────────────┐
│  Entry Point (main.cpp)                 │  ← Boot + loop scheduling
├─────────────────────────────────────────┤
│  Web Tier (web_portal.cpp)              │  ← HTTP server, captive portal, OTA
├─────────────────────────────────────────┤
│  Core Library (fake_keyboard_core)      │  ← Actions, control, sleep, LED, config
├─────────────────────────────────────────┤
│  Platform (Arduino/ESP-IDF)             │  ← USB HID, WiFi, NVS, mDNS
└─────────────────────────────────────────┘
```

**Benefits:**

- Each module has a single responsibility
- Headers in `include/` define clean APIs
- Implementations in `lib/` are self-contained
- Easy to extend without modifying core logic

### 2.2 Two-Tier Configuration System

**Compile-time defaults** (`config.h`) + **runtime NVS overrides** (`runtime_config.cpp`)

- All tunables centralized in `config.h`
- NVS persistence survives reboots
- Transparent fallback: NVS → compile-time defaults
- Factory reset clears entire namespace

**Example flow:**

```cpp
// In profiles.cpp
unsigned long profileIntervalMin(Profile profile) {
  return runtimeConfigProfileIntervalMinMs(profile);  // Reads NVS first
}

// In runtime_config.cpp
uint32_t runtimeConfigProfileIntervalMinMs(Profile profile) {
  if (profile >= PROFILE_COUNT) return 10000;
  return cfg.intervalMinMs[profile];  // Loaded from NVS or defaults
}
```

### 2.3 Humanized Timing Simulation

The `human_input.cpp` module provides sophisticated timing:

- **WPM-based delays:** `humanDelayMs()` computes character timing from words-per-minute
- **Jitter:** ±25% random variation on base delays
- **Micro-pauses:** 4% chance of 120–320 ms hesitation during typing
- **Backspace variation:** 5% chance of extra delays between deletions

**Code quality:** All timing helpers are reusable and parameterized.

### 2.4 Stealth USB Identity

`usb_identity.cpp` randomizes:

- VID/PID from 5 real-world keyboard identities
- Manufacturer/product strings
- **16-hex-char serial number** (generated from `analogRead(1) ^ micros() ^ esp_random()`)

**Effect:** Each boot presents as a different keyboard to the host OS.

### 2.5 Net-Zero Action Design

Most actions are deliberately reversible:

- ArrowScroll: N presses one way, N presses back
- Volume/Brightness: increment then decrement
- CapsLock/NumLock: toggle twice
- AltTab: switch window, dwell, switch back
- TypeText: type string, then backspace it away

**Benefit:** Minimal side effects on the host system.

### 2.6 Comprehensive LED Feedback

The LED state machine provides rich visual feedback:

- 8 distinct modes (boot, idle, charging, ready, blinking, stopped, sleeping, profile flash)
- Idle overlays for device status (5 patterns)
- Manual blink overlay for web-triggered actions
- Gamma-corrected charge ramp (amber → green)
- Sine-wave breathing during sleep

**Quality:** Non-blocking LED updates via `updateLed()` in every loop iteration.

---

## 3. Architectural Patterns & Concerns

### 3.1 Single-Threaded Cooperative Loop

**Current design:**

```cpp
void loop() {
  webPortalLoop();      // HTTP/DNS servicing
  handleButton();       // User input
  updateLed();          // LED state machine
  if (isActive && isSleeping) {
    handleSleep();
    return;
  }
  if (isActive && !isSleeping && /* charged && due time */) {
    performJiggle();    // ← BLOCKING: 22 delay() calls in actions+human_input, ~600-1700ms per AltTab
    maybeScheduleBreak();
  }
}
```

**Observations:**

- ✅ Simple, predictable execution model
- ✅ No race conditions (single-threaded)
- ⚠️ **Blocking delays** in `performJiggle()` halt the entire loop
- ⚠️ Web server polling latency during action execution
- ⚠️ Button input delayed during long actions (e.g., TypeText)

**Impact on expansion:**

- Real-time mouse control requires **non-blocking** HID reports
- Current architecture would cause 100–2000 ms latency during actions
- Solution needed: Either make actions non-blocking or use FreeRTOS tasks

### 3.2 Blocking Delay Usage

**22 `delay()` calls in the action path** (33–35 total in firmware):

| Module | Count | Typical duration | File:line |
| --- | --- | --- | --- |
| `lib/fake_keyboard_core/src/actions.cpp` | 15 | 20–1500 ms | `59:64,66,70,74,76,78,82,112,123,131,139,147,163,178,192,201` |
| `lib/fake_keyboard_core/src/human_input.cpp` | 7 | 20–320 ms | `12:14,16,24,26,34,37,44,51` |
| `src/web_portal.cpp` | 7 | 200–800 ms | `326,334,359,502,839,874,941` |
| `src/main.cpp` | 2 | 100–1000 ms | `35,45` |
| `lib/fake_keyboard_core/src/button_handler.cpp` | 1 | 300 ms (factory reset) | `59` |
| `lib/fake_keyboard_core/src/control.cpp` | 1 | 100 ms (reboot) | `104` |

**Example blocking sequence (AltTab):**

```cpp
Keyboard.press(KEY_LEFT_ALT);
delay(humanDelayMs(60, 120));      // 5–10 ms
Keyboard.press(KEY_TAB);
delay(50);                          // 50 ms
Keyboard.releaseAll();
delay(random(500, 1500));          // 500–1500 ms ← BLOCKING
Keyboard.press(KEY_LEFT_ALT);
delay(humanDelayMs(60, 120));
Keyboard.press(KEY_LEFT_SHIFT);
delay(30);
Keyboard.press(KEY_TAB);
delay(50);
Keyboard.releaseAll();
delay(20);
Keyboard.releaseAll();
```

**Total blocking time:** ~600–1700 ms per AltTab action.

**Impact:** During this time, `loop()` cannot service:

- Web requests (HTTP polling delayed)
- Button input (unresponsive)
- LED updates (frozen)
- Sleep timer (delayed wake-up)

**Mitigation strategies:**

1. **Non-blocking state machines** for each action (complex refactor)
2. **FreeRTOS tasks** on separate core (ESP32-S3 is dual-core)
3. **Accept latency** for non-real-time features (current approach)

### 3.3 HTTP-Only Networking

**Current stack:**

- `WebServer` on port 80
- Polling-based dashboard (500 ms – 2 s refresh)
- No WebSocket support
- No push notifications

**Latency analysis:**

| Operation | Best case | Worst case |
| --- | --- | --- |
| Button press → action | 500 ms (polling interval) | 2000 ms |
| Web trigger → action | ~50 ms (HTTP round-trip) | ~200 ms |
| Status update | 500 ms | 2000 ms |

**Impact on expansion:**

- Real-time mouse control requires **< 20 ms latency**
- HTTP polling is 25–100× too slow
- Solution: WebSocket for bidirectional low-latency communication

### 3.4 No Mouse HID Support

**Current HID devices** (`src/main.cpp:25-26`):

```cpp
USBHIDKeyboard       Keyboard;       // `USBHIDKeyboard.h` — keyboard reports
USBHIDConsumerControl Consumer;      // `USBHIDConsumerControl.h` — media keys
// no USBHIDMouse — mouse not exposed
```

**Verified:** `grep -rn USBHID src/ include/ lib/` returns only the two above; no `Mouse`
instance exists. `platformio.ini:30-32` only pulls `USB` — no mouse dependency yet.

**Missing:**

- `USBHIDMouse` — relative/absolute mouse movement
- `USBHIDAbsoluteMouse` — for absolute positioning
- Multi-touch or digitizer support

**ESP32-S3 Arduino core (`espressif32` + `framework = arduino`) ships with:**

- `USBHIDMouse` — relative movement `move(x, y)` — available via `#include "USBHIDMouse.h"` (ships with core, no extra `lib_deps` needed)
- Button press/release `click()`, `press()`, `release()`
- Scroll wheel `move(0, 0, wheel)` — same instance

**Impact:** Adding mouse control requires:

1. Instantiate `USBHIDMouse` in `main.cpp`
2. Add mouse movement functions to `human_input.cpp`
3. Create mouse-specific actions or real-time control interface

### 3.5 Memory Usage Patterns

**Static allocations:**

| Component | Size | Notes |
| --- | --- | --- |
| Action history | 20 entries × ~20 bytes | ~400 bytes |
| Event log | 40 entries × ~50 bytes | ~2 KB (String objects) |
| NVS config | ~300 bytes | Fixed-size struct |
| Web server buffers | ~4 KB | HTML templates, JSON responses |
| WiFi stack | ~50 KB | ESP-IDF internal |
| USB HID | ~2 KB | Descriptor buffers |

**Total estimated RAM:** ~60–80 KB (ESP32-S3 has 512 KB SRAM)

**Observation:** Plenty of headroom for WebSocket buffers, mouse state, etc.

### 3.6 NVS Validation & Safety

**Strengths:**

- Interval clamping: [250 ms, 3 600 000 ms]
- Auto-swap if min > max
- Weight table size validation
- Factory reset clears entire namespace

**Gap:**

- No checksum or versioning
- Corrupt NVS data could cause undefined behavior
- No migration path for schema changes

**Recommendation:**

- Add `configVersion` key for future schema migrations
- Validate critical fields on load

---

## 4. Performance Analysis

### 4.1 Loop Iteration Timing

**Typical loop iteration (idle state):**

| Function | Time |
| --- | --- |
| `webPortalLoop()` | ~1–5 ms (HTTP polling) |
| `handleButton()` | < 1 ms |
| `updateLed()` | < 1 ms |
| **Total** | ~2–7 ms |

**During action execution:**

| Action | Blocking time |
| --- | --- |
| ShiftTap | ~100 ms |
| ArrowScroll (8 presses) | ~800 ms |
| AltTab | ~1000 ms |
| TypeText (20 chars) | ~2000 ms |

**Impact:** Loop is blocked for the duration of the action.

### 4.2 Web Server Throughput

**HTTP request handling:**

- JSON serialization: ~1–2 ms
- HTML template rendering: ~5–10 ms
- OTA upload: streaming (no memory bottleneck)

**Observation:** Web server is fast enough for current polling rates.

### 4.3 USB HID Report Rate

**USB HID spec:**

- Full-speed USB: 1 ms frame interval
- Typical keyboard/mouse: 125 Hz report rate (8 ms)

**ESP32-S3 USB stack:**

- Non-blocking `Keyboard.press()`, `Mouse.move()`
- Reports queued and sent at USB frame rate
- No CPU bottleneck for HID reports

**Key insight:** HID reports are **non-blocking and fast**. The bottleneck is WiFi/WebSocket, not USB.

---

## 5. Gaps & Missing Capabilities

### 5.1 No Real-Time Remote Control

**Current state:**

- Web UI is for configuration, not real-time control
- Manual action trigger exists but with 500 ms – 2 s latency
- No mouse control
- No live keyboard input

**Requirement for expansion:**

- WebSocket for low-latency bidirectional communication
- Mouse HID support
- Browser-based input capture (Pointer Lock API)
- Non-blocking HID report generation

### 5.2 No Authentication or Encryption

**Current state:**

- HTTP only (no HTTPS)
- No authentication
- No rate limiting
- Dashboard accessible to anyone on the network

**Security implications:**

- Unauthorized control of HID device
- Potential for malicious input injection
- Wi-Fi credentials stored in plaintext in NVS

**Recommendation:**

- Add optional password protection for web UI
- Consider HTTPS for sensitive operations (OTA, config changes)
- Document security model clearly

### 5.3 No Automated Tests

**Current state:**

- `test/README` is PlatformIO placeholder
- No unit tests
- No integration tests

**Risk:**

- Regressions undetected
- Refactoring risky
- Action selection logic untested

**Recommendation:**

- Add unit tests for:
  - Weighted action selection
  - NVS config persistence
  - Interval validation
  - Human timing calculations

### 5.4 No Action Scheduling or Macros

**Current state:**

- Actions are random or manually triggered
- No way to schedule specific actions at specific times
- No macro recording/playback

**Potential features:**

- Scheduled actions (e.g., "press F5 every 30 min")
- Macro recording (record sequence, replay with humanized timing)
- Conditional actions (e.g., "if idle for 5 min, press Shift")

### 5.5 Limited Error Handling

**Current state:**

- Most functions assume success
- No retry logic for HID reports
- No error reporting to web UI

**Examples:**

- `Keyboard.press()` returns void (no error check)
- NVS write failures logged but not surfaced
- WiFi connection failures retried but not reported to user

**Recommendation:**

- Add error counters and status reporting
- Surface critical errors in web UI

---

## 6. Optimization Opportunities

### 6.1 Non-Blocking Action Execution

**Current:** Actions use blocking `delay()` calls.

**Proposed:** Convert actions to state machines with non-blocking timing.

**Example (AltTab):**

```cpp
enum AltTabState {
  ALT_TAB_IDLE,
  ALT_TAB_PRESS_ALT,
  ALT_TAB_PRESS_TAB,
  ALT_TAB_DWELL,
  ALT_TAB_RETURN,
  ALT_TAB_DONE
};

void actionAltTab_update() {
  static AltTabState state = ALT_TAB_IDLE;
  static unsigned long stateStart = 0;

  switch (state) {
    case ALT_TAB_IDLE:
      Keyboard.press(KEY_LEFT_ALT);
      stateStart = millis();
      state = ALT_TAB_PRESS_TAB;
      break;

    case ALT_TAB_PRESS_TAB:
      if (millis() - stateStart >= humanDelayMs(60, 120)) {
        Keyboard.press(KEY_TAB);
        stateStart = millis();
        state = ALT_TAB_DWELL;
      }
      break;

    // ... more states ...
  }
}
```

**Benefits:**

- Loop remains responsive during actions
- Button input processed immediately
- LED updates continue smoothly

**Cost:**

- Significant refactor of all 12 actions
- More complex code
- Increased state management overhead

**Alternative:** Use FreeRTOS tasks on core 0 for actions, keep core 1 for loop.

### 6.2 WebSocket for Real-Time Control

**Current:** HTTP polling (500 ms – 2 s latency).

**Proposed:** WebSocket for bidirectional low-latency communication.

**Benefits:**

- < 20 ms latency for mouse/keyboard input
- Push notifications for state changes
- Efficient binary protocol for mouse deltas

**Implementation:**

- Use `WebSocketsServer` library (PlatformIO compatible)
- Binary protocol for mouse movement (compact)
- JSON for configuration/status (human-readable)

**Impact on CPU:**

- WebSocket overhead: ~1–2 ms per message
- Negligible compared to HTTP polling
- No increase in CPU usage if throttled properly

### 6.3 Mouse HID with Throttling

**Proposed:** Add `USBHIDMouse` with 125 Hz report rate (8 ms).

**Optimization:**

- Batch mouse deltas in browser (accumulate movement)
- Send at most 1 report per 8 ms
- Use relative movement (smaller data than absolute)

**Binary protocol example:**

```
Mouse move: [0x01, dx, dy, buttons]  (4 bytes)
Key press:  [0x02, keycode]           (2 bytes)
Key release:[0x03, keycode]           (2 bytes)
```

**Impact:**

- Minimal USB overhead (non-blocking)
- WiFi bandwidth: ~500 bytes/sec at 125 Hz
- CPU: < 1% for mouse processing

### 6.4 Dual-Core Task Separation

**ESP32-S3 is dual-core:**

- Core 0: WiFi, Bluetooth, system tasks (ESP-IDF)
- Core 1: Arduino `loop()`, user code

**Proposed:**

- Core 0: WiFi stack + WebSocket server
- Core 1: HID actions + LED + button

**Benefits:**

- WebSocket processing doesn't block HID actions
- HID actions don't delay WebSocket responses
- True parallelism

**Implementation:**

```cpp
// In setup()
xTaskCreatePinnedToCore(
  webSocketTask,   // Function
  "WebSocket",     // Name
  8192,            // Stack size
  NULL,            // Parameters
  1,               // Priority
  NULL,            // Handle
  0                // Core 0
);

// loop() runs on core 1 by default
```

**Impact:**

- No increase in CPU usage (just better utilization)
- Lower latency for both HID and WebSocket
- More complex synchronization (mutexes for shared state)

### 6.5 Memory Optimization

**Current:** ~60–80 KB RAM usage (estimated).

**Opportunities:**

- Use `PROGMEM` for HTML templates (save ~10 KB RAM)
- Reduce event log size (40 → 20 entries)
- Use fixed-size buffers instead of `String` objects

**Example (PROGMEM):**

```cpp
const char PORTAL_HTML[] PROGMEM = R"rawliteral(
<!doctype html><html>...
)rawliteral";
```

**Impact:**

- Frees ~10–20 KB RAM
- Slightly slower HTML serving (flash read)
- Worth it for headroom

---

## 7. Expansion Readiness Assessment

### 7.1 What's Ready for Expansion

✅ **Clean APIs:** Easy to add new actions, profiles, HID devices

✅ **NVS persistence:** Proven pattern for runtime config

✅ **Humanized timing:** Reusable helpers for new input types

✅ **LED feedback:** Extensible state machine

✅ **Web UI framework:** Vue + fallback template works well

### 7.2 What Needs Refactoring

⚠️ **Blocking delays:** Must be addressed for real-time control

⚠️ **HTTP-only:** WebSocket required for low-latency input

⚠️ **No mouse HID:** Must add `USBHIDMouse` instance

⚠️ **Single-threaded:** Consider FreeRTOS tasks for parallelism

### 7.3 Recommended Expansion Path

**Phase 1: Foundation (low risk)**

1. Add `USBHIDMouse` instance
2. Add WebSocket support (non-breaking)
3. Create mouse movement actions (non-real-time)

**Phase 2: Real-Time Control (medium risk)**

1. Implement WebSocket binary protocol
2. Add browser-based mouse capture (Pointer Lock API)
3. Add real-time keyboard input (non-blocking)

**Phase 3: Optimization (optional)**

1. Convert actions to non-blocking state machines
2. Use FreeRTOS tasks for core separation
3. Add PROGMEM for HTML templates

---

## 8. Conclusion

The EspressoHID: KM firmware is a solid foundation for expansion. The codebase is well-organized,
the configuration system is robust, and the humanized timing simulation is sophisticated.

**Key takeaways:**

1. **Blocking delays** are the biggest architectural challenge for real-time control
2. **WebSocket** is essential for low-latency remote input
3. **Mouse HID** is straightforward to add (ESP32-S3 Arduino core supports it)
4. **Dual-core ESP32-S3** offers a path to parallelism without CPU overhead
5. **Memory headroom** is sufficient for expansion

**Next steps:**

- Review `docs/FEATURE_EXPANSION.md` for the Virtual KM over IP proposal
- Prioritize WebSocket + Mouse HID as the first expansion phase
- Consider non-blocking action refactor as a prerequisite for real-time control

---

## Appendix A: File Dependency Graph

```
main.cpp
  ├─→ config.h
  ├─→ state.h
  ├─→ profiles.h
  ├─→ led_controller.h
  ├─→ usb_identity.h
  ├─→ actions.h
  ├─→ sleep_manager.h
  ├─→ button_handler.h
  └─→ web_portal.h
        ├─→ runtime_config.h
        ├─→ actions.h
        ├─→ control.h
        ├─→ event_log.h
        ├─→ state.h
        ├─→ profiles.h
        └─→ device_status.h

lib/fake_keyboard_core/src/
  ├─→ actions.cpp
  │     ├─→ led_controller.h
  │     ├─→ state.h
  │     ├─→ runtime_config.h
  │     ├─→ event_log.h
  │     └─→ human_input.h
  ├─→ control.cpp
  │     ├─→ state.h
  │     ├─→ profiles.h
  │     ├─→ led_controller.h
  │     ├─→ sleep_manager.h
  │     └─→ event_log.h
  ├─→ sleep_manager.cpp
  │     ├─→ state.h
  │     ├─→ profiles.h
  │     ├─→ led_controller.h
  │     └─→ event_log.h
  └─→ ... (other modules)
```

## Appendix B: Timing Constants Summary

| Constant | Value | Module |
| --- | --- | --- |
| `BOOT_COUNTDOWN_S` | 2 s | config.h |
| `BUTTON_DEBOUNCE_MS` | 50 ms | config.h |
| `LONG_PRESS_MS` | 1500 ms | config.h |
| `FACTORY_RESET_HOLD_MS` | 10 000 ms | config.h |
| `ACTION_BLINK_MS` | 1000 ms | config.h |
| `BLINK_TOGGLE_MS` | 100 ms | config.h |
| `STOP_RED_HOLD_MS` | 2000 ms | config.h |
| `PROFILE_INDICATOR_MS` | 800 ms | config.h |
| `LED_SLEEP_PULSE_MS` | 1000 ms | config.h |
| `STATUS_LOG_INTERVAL_MS` | 10 000 ms | config.h |
| `ACTIVE_FIRST_MIN/MAX_MS` | 1000–3000 ms | config.h |
| `ACTIVE_INTERVAL_MIN/MAX_MS` | 10 000–60 000 ms | config.h |
| `MEETING_FIRST_MIN/MAX_MS` | 3000–8000 ms | config.h |
| `MEETING_INTERVAL_MIN/MAX_MS` | 45 000–180 000 ms | config.h |
| `SLEEP_CHANCE_PER_ACTION` | 8% | config.h |
| `SLEEP_MIN/MAX_MS` | 60 000–180 000 ms | config.h |
| `WORK_SESSION_MIN/MAX_MS` | 1 200 000–2 400 000 ms | config.h |
| `LONG_BREAK_MIN/MAX_MS` | 180 000–420 000 ms | config.h |
