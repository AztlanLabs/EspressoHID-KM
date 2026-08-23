# EspressoHID: KM — Stealth Audit & Hardening Guide

> Comprehensive analysis of detection vectors, current vulnerabilities, and hardening
> strategies to make the device indistinguishable from a legitimate USB peripheral.

---

## Executive Summary

This document audits EspressoHID: KM against every known detection method used by enterprise
monitoring software, OS-level analysis, and USB forensic tools. It identifies **6 critical**,
**4 moderate**, and **3 minor** stealth weaknesses in the current firmware (v0.6.0-release)
and provides concrete hardening strategies for each.

**Current stealth rating: 5/10** — The device already bypasses OS injected-input flags (hardware HID
advantage) and has meaningful humanization (WPM-based delays with ±25% jitter, 4% micro-pauses on type,
5% backspace hesitation, weighted random action selection with 12 actions, sleep/break simulation, and
randomized USB identity per boot with `analogRead(1) ^ micros() ^ esp_random()` seed + 16-hex serial).
It remains vulnerable to behavioral analysis, USB descriptor fingerprinting, and uniform-distribution
timing.

**Target stealth rating: 9/10** — Achievable with the hardening strategies described herein
(mostly additive — few core changes, largely new mouse path + distribution hardening).

---

## Table of Contents

1. [Detection Landscape Overview](#1-detection-landscape-overview)
2. [Detection Vectors & Current Vulnerabilities](#2-detection-vectors--current-vulnerabilities)
3. [Critical Vulnerabilities](#3-critical-vulnerabilities)
4. [Moderate Vulnerabilities](#4-moderate-vulnerabilities)
5. [Minor Vulnerabilities](#5-minor-vulnerabilities)
6. [Hardening Strategies](#6-hardening-strategies)
7. [Implementation Roadmap](#7-implementation-roadmap)
8. [Testing & Validation](#8-testing--validation)

---

## 1. Detection Landscape Overview

### 1.1 Who Is Detecting Jigglers in 2026?

| Platform | Detection Method | Accuracy Claim |
| --- | --- | --- |
| **Insightful** | Activity Verification — distinguishes authentic from simulated input in real time | 99% (vendor claim) |
| **Hubstaff** | Insights add-on — flags on published behavioral thresholds | Threshold-based |
| **Time Doctor** | Unusual Activity Report — names jigglers directly, scores activity quality | Pattern-based |
| **Teramind** | Behavioral analytics + screenshot correlation | AI-assisted |
| **ActivTrak** | Activity monitoring + productivity scoring | Pattern-based |
| **Monitask** | Device audit + screenshot + app monitoring | Multi-signal |
| **Detect My Jiggler** | Open-source Windows tool — real-time HID behavioral analysis | Pattern-based |
| **Linux hid-omg-detect** | Kernel driver — keystroke timing entropy + descriptor fingerprinting | Scoring system |
| **Windows Defender** | Signature-based detection of software jigglers | Signature |

### 1.2 Detection Method Categories

| Category | What It Catches | EspressoHID Bypasses? |
| --- | --- | --- |
| **OS Injected-Input Flags** | Software jigglers using `SendInput()` API | ✅ Yes (hardware HID) |
| **USB Device Enumeration** | New USB devices plugged in | ⚠️ Partially |
| **HID Descriptor Fingerprinting** | Generic or mismatched descriptors | ❌ No |
| **Movement Pattern Analysis** | Repetitive, robotic, or non-human patterns | ❌ No |
| **Behavioral Analytics** | Activity that doesn't correlate with real work | ❌ No |
| **Keyboard/Mouse Correlation** | Keyboard-only or mouse-only activity | ❌ No |
| **Screenshot Correlation** | Activity with static screen content | ❌ N/A (no screen access) |
| **Timing Entropy Analysis** | Low-entropy, predictable timing patterns | ⚠️ Partially |

### 1.3 The Hardware HID Advantage

**Key insight from research:** Hardware USB jigglers have a fundamental advantage over software
jigglers. Windows sets an "injected-input flag" when input comes through the `SendInput()` API.
Software jigglers are detected this way. **Hardware HID devices bypass this entirely** because
they enumerate as real USB peripherals — the input travels the same kernel path as a physical
keyboard/mouse.

> "A USB jiggler dongle is not software pretending to be a mouse. It is a microcontroller
> that speaks USB HID, and the operating system enumerates it as an HID-compliant pointing
> device. That is the whole trick, and it is why these devices need no driver and no
> installation." — mouse-jiggler.org

**However:** This advantage only covers OS-level injection detection. Behavioral analysis,
descriptor fingerprinting, and movement pattern analysis can still detect the device.

---

## 2. Detection Vectors & Current Vulnerabilities

> **How to read this matrix:** `Existing mitigation` notes what `v0.6.0-release` already does.
> Several critical vectors are *partially* mitigated — hardening is additive, not from zero.

### 2.1 Vulnerability Matrix

| # | Vulnerability | Severity | Detection Vector | Existing Mitigation in v0.6.0 | Gap Still Open |
| --- | --- | --- | --- | --- | --- |
| V1 | No mouse HID device | 🔴 Critical | Behavioral analysis, USB enumeration | None — `src/main.cpp:25-26` only exposes `Keyboard` + `Consumer` | Keyboard-only device is suspicious; real office keyboards with trackpoint expose mouse |
| V2 | Repetitive action patterns | 🔴 Critical | Movement pattern analysis | Net-zero ArrowScroll (`N` one way, `N` back) already has `vertical 80% / horizontal 20%` + `startDown 50%` + `ARROW_GAP 60-500ms`; still deterministic structure | Structure is still `N`→pause→`N` — high pattern repetition |
| V3 | Uniform random distribution | 🔴 Critical | Timing entropy analysis | `lib/fake_keyboard_core/src/human_input.cpp:3-7` already has `humanDelayMs()` with WPM jitter ±25% + 4%/5% micro-pauses — *not* pure uniform — but `random()` elsewhere is uniform | `actions.cpp:98` `random(1,8)`, `random(500,1500)`, interval selection `random(min,max)` are uniform (low entropy ~2.0 bits) |
| V4 | Fixed action intervals | 🔴 Critical | Behavioral analytics | `SLEEP_CHANCE 8%` naps + `WORK_SESSION 20-40min` long breaks already add variation, but steady-state intervals are uniform `random(10s,60s)` | 10–60 s (ACTIVE) / 45–180 s (MEETING) with uniform dist → Hubstaff "95%+ activity 30 min" flag still possible |
| V5 | No mixed activity types | 🔴 Critical | Keyboard/mouse correlation | 12 actions already mix keyboard + consumer control, but zero mouse/scroll | Tracks to keyboard-only activity — zero mouse movement for hours |
| V6 | Net-zero actions too regular | 🔴 Critical | Pattern analysis | Actions are net-zero already (good for UX), but always paired 1:1 | Volume +/- always paired, CapsToggle always 2× — pattern-detectable |
| V7 | Generic USB descriptor | 🟡 Moderate | HID descriptor fingerprinting | Randomized VID/PID + manufacturer/product per boot (`usb_identity.cpp:8-21`) | Arduino ESP32-S3 TinyUSB default HID report descriptor is still identifiable vs real Dell/HP/Logitech descriptors |
| V8 | Serial number format | 🟡 Moderate | USB enumeration forensics | Randomized 16-hex serial per boot (`%08lX%08lX`) | Format `AAAAAAAA BBBBBBBB` is not vendor-realistic (`CN0…`, `CZC…`, etc.) |
| V9 | No USB version/power descriptors | 🟡 Moderate | USB enumeration | TinyUSB stack does provide basic USB descriptors | Real devices expose richer string/config/power descriptors — audit by extracting with `usbhid-dump` |
| V10 | Composite device class mismatch | 🟡 Moderate | USB enumeration | Keyboard + ConsumerControl | Real office keyboard+trackpoint is Keyboard + Mouse + ConsumerControl |
| V11 | No scroll wheel activity | 🟢 Minor | Behavioral analysis | None | Real users scroll constantly |
| V12 | No drag operations | 🟢 Minor | Behavioral analysis | None | Real users drag-select, drag-drop |
| V13 | No right-click/context menu | 🟢 Minor | Behavioral analysis | None | Real users use context menus |

---

## 3. Critical Vulnerabilities

### V1: No Mouse HID Device

**Problem:** The current firmware only exposes `USBHIDKeyboard` and `USBHIDConsumerControl`.
A keyboard-only USB device that never produces mouse movement is **immediately suspicious** to
behavioral analytics platforms.

**Detection method:** Monitoring tools track the ratio of keyboard to mouse activity. A device
that only produces keyboard events with zero mouse movement for hours is flagged.

**Evidence from research:**

> "If mouse moves but keyboard is silent for hours, suspicious. If only arrow keys are pressed,
> suspicious. Need to mix keyboard and mouse activity." — multiple detection guides

**Impact:** Any monitoring tool that tracks input device types will flag this.

**Fix:** Add `USBHIDMouse` to create a composite keyboard + mouse + consumer control device.
This is what real keyboards with trackpoints, touchpads, or integrated pointing devices look
like.

### V2: Repetitive Action Patterns

**Problem:** Current actions have mechanically regular patterns:

```cpp
// ArrowScroll: N presses one way, N presses back — TOO REGULAR
for (int i = 0; i < presses; i++) {
  tapKey(Keyboard, first, ...);
}
delay(random(220, 900));
for (int i = 0; i < presses; i++) {
  tapKey(Keyboard, second, ...);
}
```

**Detection method:** Monitoring tools analyze:

- **Movement distance variance** — real humans have high variance
- **Direction change frequency** — real humans change direction irregularly
- **Timing between events** — real humans have variable pauses
- **Pattern repetition** — real humans don't repeat exact sequences

**Evidence from research:**

> "Most USB jigglers produce mechanically regular movement: the same short displacement at
> the same interval, forever. That signature is unmistakable to anyone watching. Real cursor
> movement is irregular in distance, direction, speed, and timing, with natural pauses to
> read or think." — mouse-jiggler.org

**Impact:** Any tool that analyzes input timing patterns will detect this.

**Fix:** Implement humanized movement algorithms (see §6.2).

### V3: Uniform Random Distribution

**Problem:** All timing uses Arduino's `random(min, max)` which produces a **uniform
distribution**. Real human timing follows a **log-normal or gamma distribution** — mostly
fast with occasional long pauses.

```cpp
// Current: uniform distribution — every value equally likely
delay(random(500, 1500));  // 500, 750, 1000, 1250, 1500 all equally likely

// Real human: log-normal — mostly fast, occasional long pauses
delay(logNormalDelay(800, 300));  // 400, 600, 800 common; 1500 rare
```

**Detection method:** Statistical analysis of timing distributions. Uniform distributions
have low entropy and are easily distinguishable from human timing.

**Evidence from research:**

> "Keystroke timing entropy" is one of the three scoring factors in the Linux `hid-omg-detect`
> kernel driver. Low-entropy timing is a strong signal of automation.

**Impact:** Any tool that performs statistical analysis of input timing will detect this.

**Fix:** Replace `random()` with log-normal or gamma distribution generators (see §6.3).

### V4: Fixed Action Intervals

**Problem:** Actions happen at predictable intervals:

```cpp
// ACTIVE: 10–60 s, uniform distribution
#define ACTIVE_INTERVAL_MIN_MS  10000
#define ACTIVE_INTERVAL_MAX_MS  60000

// MEETING: 45–180 s, uniform distribution
#define MEETING_INTERVAL_MIN_MS 45000
#define MEETING_INTERVAL_MAX_MS 180000
```

**Detection method:** Monitoring tools flag:

- **95%+ activity for 30 min** (Hubstaff threshold)
- **Under 4% fluctuation for 90 min** (Hubstaff threshold)
- **Consistent activity without natural breaks** (all platforms)

**Evidence from research:**

> "A perfectly metronomic wiggle for eight hours is not how a human uses a mouse." —
> mouse-jiggler.org

**Impact:** Any tool that tracks activity consistency over time will detect this.

**Fix:** Implement realistic work patterns with bursts, pauses, and circadian variation
(see §6.4).

### V5: No Mixed Activity Types

**Problem:** The device only produces keyboard events. Real users constantly mix:

- Mouse movement (pointing, scrolling)
- Keyboard input (typing, shortcuts)
- Scroll wheel (reading documents)
- Click + type + scroll sequences (compound actions)

**Detection method:** Monitoring tools track the ratio of keyboard to mouse activity. A
device that only produces keyboard events with zero mouse movement for hours is flagged.

**Evidence from research:**

> "Keyboard near 0% for 50 min" is a Hubstaff threshold for flagging simulated activity.
> The inverse (mouse near 0%) is equally suspicious.

**Impact:** Any tool that correlates keyboard and mouse activity will detect this.

**Fix:** Add mouse HID and implement mixed activity patterns (see §6.5).

### V6: Net-Zero Actions Too Regular

**Problem:** Net-zero actions always follow the exact same pattern:

```cpp
// Volume: always increment then decrement — TOO REGULAR
tapConsumer(Consumer, CONSUMER_CONTROL_VOLUME_INCREMENT, 35, 85);
delay(random(80, 200));
tapConsumer(Consumer, CONSUMER_CONTROL_VOLUME_DECREMENT, 35, 85);

// CapsToggle: always on then off — TOO REGULAR
tapKey(Keyboard, KEY_CAPS_LOCK, 45, 95, 120, 260);
delay(random(150, 400));
tapKey(Keyboard, KEY_CAPS_LOCK, 45, 95, 60, 180);
```

**Detection method:** Pattern analysis detects that every Volume action is followed by a
Volume decrement, every CapsToggle is followed by another CapsToggle, etc.

**Impact:** Any tool that analyzes action sequences will detect this.

**Fix:** Add variation to net-zero actions — sometimes don't reverse, sometimes reverse
after a delay, sometimes reverse with a different action in between (see §6.6).

---

## 4. Moderate Vulnerabilities

### V7: Generic USB Descriptor

**Problem:** The Arduino ESP32-S3 USB stack generates a generic HID descriptor that is
identifiable as an Arduino/ESP32 device.

**Detection method:** USB descriptor fingerprinting tools can compare the descriptor against
a database of known devices. The Arduino descriptor has a distinctive structure.

**Evidence from research:**

> "By extracting USB report descriptor hex dump from a Logitech LX3 Mouse and using a USB
> Descriptor and Request Parser tool, the raspberry pi pico will have the identical report
> descriptor of the USB HID device." — arcanine300/raspberryPi-USB-HID

**Impact:** Enterprise endpoint management tools that inventory USB devices will flag this.

**Fix:** Override the default descriptor with one extracted from a real keyboard (see §6.7).

### V8: Serial Number Format

**Problem:** The current serial number is a 16-hex-char string:

```cpp
snprintf(serial, sizeof(serial), "%08lX%08lX",
         (unsigned long)random(0x7FFFFFFFL),
         (unsigned long)random(0x7FFFFFFFL));
```

**Detection method:** Real keyboards have serial numbers in specific formats (e.g., Dell uses
`CN0XXXXX`, HP uses `CZCXXXXX`, Logitech uses alphanumeric patterns). A 16-hex-char serial
is unusual.

**Impact:** Low — most monitoring tools don't analyze serial number formats.

**Fix:** Generate serial numbers that match the format of the spoofed manufacturer (see §6.8).

### V9: No USB Version/Power Descriptors

**Problem:** The Arduino USB stack may not include all standard USB descriptors that a real
keyboard would have (e.g., USB version, power consumption, string descriptors).

**Detection method:** USB enumeration forensics can detect missing or incomplete descriptors.

**Impact:** Low — most monitoring tools don't analyze USB descriptors in detail.

**Fix:** Ensure all standard USB descriptors are present (see §6.7).

### V10: Composite Device Class Mismatch

**Problem:** The device exposes Keyboard + ConsumerControl but no mouse. Real keyboards with
integrated pointing devices (trackpoint, touchpad) expose Keyboard + Mouse + ConsumerControl.

**Detection method:** USB enumeration tools can see the device class and flag mismatches.

**Impact:** Moderate — endpoint management tools that inventory USB devices will flag this.

**Fix:** Add `USBHIDMouse` to create a proper composite device (see §6.1).

---

## 5. Minor Vulnerabilities

### V11: No Scroll Wheel Activity

**Problem:** Real users scroll constantly — reading documents, browsing web pages, scrolling
through code. The device never produces scroll events.

**Detection method:** Behavioral analysis tracks scroll activity as a signal of real work.

**Impact:** Low — scroll activity is one of many signals.

**Fix:** Add scroll wheel actions and mouse scroll events (see §6.5).

### V12: No Drag Operations

**Problem:** Real users drag-select text, drag-drop files, and perform other drag operations.
The device never produces drag events.

**Detection method:** Behavioral analysis tracks drag activity as a signal of real work.

**Impact:** Low — drag activity is one of many signals.

**Fix:** Add drag-select actions (see §6.5).

### V13: No Right-Click/Context Menu

**Problem:** Real users right-click to open context menus. The device never produces
right-click events.

**Detection method:** Behavioral analysis tracks right-click activity as a signal of real work.

**Impact:** Low — right-click activity is one of many signals.

**Fix:** Add right-click actions (see §6.5).

---

## 6. Hardening Strategies

### 6.1 Add Mouse HID Device (Fixes V1, V10)

**Strategy:** Add `USBHIDMouse` to create a composite keyboard + mouse + consumer control
device. This is what real keyboards with trackpoints/touchpads look like.

**Implementation:**

```cpp
// main.cpp
#include "USBHIDMouse.h"

USBHIDKeyboard       Keyboard;
USBHIDConsumerControl Consumer;
USBHIDMouse          Mouse;  // NEW

void setup() {
  // ... existing setup ...
  Keyboard.begin();
  Consumer.begin();
  Mouse.begin();  // NEW
  USB.begin();
}
```

**Impact:** The device now appears as a keyboard with an integrated pointing device, which
is indistinguishable from a real keyboard with a trackpoint.

### 6.2 Humanize Movement Patterns (Fixes V2)

**Strategy:** Replace mechanical movement patterns with human-like algorithms.

**Algorithms to implement:**

#### 6.2.1 Brownian Motion (Simple)

```cpp
// Brownian motion: random walk with drift
struct BrownianState {
  float x, y;
  float driftX, driftY;
  float volatility;
};

void brownianStep(BrownianState& state) {
  // Random perturbation
  float dx = gaussianRandom(0, state.volatility);
  float dy = gaussianRandom(0, state.volatility);

  // Drift toward target (optional)
  dx += state.driftX * 0.1f;
  dy += state.driftY * 0.1f;

  state.x += dx;
  state.y += dy;
}
```

#### 6.2.2 Perlin Noise (Advanced)

```cpp
// Perlin noise: smooth, continuous random movement
float perlinNoise1D(float x) {
  // Simple 1D Perlin noise implementation
  int xi = (int)floor(x);
  float xf = x - xi;
  float u = fade(xf);

  int a = perm[xi & 255];
  int b = perm[(xi + 1) & 255];

  return lerp(grad1D(a, xf), grad1D(b, xf - 1), u);
}

// Use Perlin noise for smooth mouse movement
void perlinMouseMove() {
  static float t = 0;
  t += 0.01f;  // Time step

  float dx = perlinNoise1D(t) * 5.0f;
  float dy = perlinNoise1D(t + 100.0f) * 5.0f;

  Mouse.move((int8_t)dx, (int8_t)dy);
}
```

#### 6.2.3 WindMouse Algorithm (Best)

The WindMouse algorithm generates curved, non-linear paths with variable speed:

```cpp
// WindMouse: human-like mouse movement
struct WindMouseState {
  float x, y;
  float targetX, targetY;
  float velocityX, velocityY;
  float gravity, wind;
  float damping;
};

void windMouseStep(WindMouseState& state) {
  float dx = state.targetX - state.x;
  float dy = state.targetY - state.y;
  float dist = sqrt(dx * dx + dy * dy);

  // Gravity pulls toward target
  float gravX = dx / dist * state.gravity;
  float gravY = dy / dist * state.gravity;

  // Wind adds random perturbation
  float windX = gaussianRandom(0, state.wind);
  float windY = gaussianRandom(0, state.wind);

  // Update velocity with damping
  state.velocityX = (state.velocityX + gravX + windX) * state.damping;
  state.velocityY = (state.velocityY + gravY + windY) * state.damping;

  // Update position
  state.x += state.velocityX;
  state.y += state.velocityY;
}
```

**Key properties:**

- **Curved paths** — not straight lines
- **Variable speed** — accelerates at start, decelerates at target
- **Overshoot and correction** — sometimes exceeds target before correcting
- **Micro tremors** — small random perturbations
- **Fitts' law** — movement duration increases with distance

### 6.3 Replace Uniform Random with Human Distributions (Fixes V3)

**Strategy:** Replace `random(min, max)` with log-normal or gamma distributions.

**Implementation:**

```cpp
// Log-normal distribution: mostly fast, occasional long pauses
unsigned long logNormalDelay(unsigned long medianMs, unsigned long sigmaMs) {
  // Box-Muller transform for normal distribution
  float u1 = (float)random(1, 10000) / 10000.0f;
  float u2 = (float)random(1, 10000) / 10000.0f;
  float z = sqrt(-2.0f * log(u1)) * cos(2.0f * M_PI * u2);

  // Log-normal: exp(mu + sigma * z)
  float mu = log((float)medianMs);
  float sigma = (float)sigmaMs / (float)medianMs;
  float delay = exp(mu + sigma * z);

  // Clamp to reasonable range
  if (delay < 10) delay = 10;
  if (delay > medianMs * 5) delay = medianMs * 5;

  return (unsigned long)delay;
}

// Gamma distribution: flexible shape
unsigned long gammaDelay(float shape, float scale) {
  // Marsaglia and Tsang's method
  float d, c, x, v, u;
  if (shape >= 1) {
    d = shape - 1.0f / 3.0f;
    c = 1.0f / sqrt(9.0f * d);
    do {
      do {
        x = gaussianRandom(0, 1);
        v = 1.0f + c * x;
      } while (v <= 0);
      v = v * v * v;
      u = (float)random(1, 10000) / 10000.0f;
    } while (u >= 1 - 0.0331f * (x * x) * (x * x) &&
             log(u) >= 0.5f * x * x + d * (1 - v + log(v)));
    return (unsigned long)(d * v * scale);
  }
  return (unsigned long)(scale * 100);  // Fallback
}

// Usage: replace random() calls
// Before: delay(random(500, 1500));
// After:  delay(logNormalDelay(800, 300));
```

**Key properties:**

- **Log-normal:** Mostly fast, occasional long pauses (matches human typing)
- **Gamma:** Flexible shape, can model various human behaviors
- **High entropy:** Passes statistical analysis

### 6.4 Implement Realistic Work Patterns (Fixes V4)

**Strategy:** Replace fixed intervals with realistic work patterns.

**Implementation:**

```cpp
// Circadian rhythm: activity varies throughout the day
struct WorkPattern {
  unsigned long sessionStart;
  unsigned long lastAction;
  unsigned long burstEnd;
  unsigned long pauseEnd;
  uint8_t burstCount;
  bool inBurst;
  bool inPause;
};

void updateWorkPattern(WorkPattern& pattern) {
  const unsigned long now = millis();

  // Burst mode: rapid actions for 2–5 minutes
  if (pattern.inBurst) {
    if (now >= pattern.burstEnd) {
      pattern.inBurst = false;
      pattern.inPause = true;
      pattern.pauseEnd = now + random(60000, 300000);  // 1–5 min pause
    }
    return;
  }

  // Pause mode: no actions for 1–5 minutes
  if (pattern.inPause) {
    if (now >= pattern.pauseEnd) {
      pattern.inPause = false;
      pattern.burstCount++;
    }
    return;
  }

  // Normal mode: decide whether to start a burst
  if (random(100) < 5) {  // 5% chance of burst
    pattern.inBurst = true;
    pattern.burstEnd = now + random(120000, 300000);  // 2–5 min burst
  }
}

// Get next action delay based on work pattern
unsigned long getNextDelay(WorkPattern& pattern) {
  if (pattern.inBurst) {
    return random(5000, 15000);  // Fast: 5–15 s
  }
  if (pattern.inPause) {
    return random(60000, 300000);  // Slow: 1–5 min
  }
  return random(15000, 90000);  // Normal: 15–90 s
}
```

**Key properties:**

- **Burst mode:** Rapid actions for 2–5 minutes (simulates active work)
- **Pause mode:** No actions for 1–5 minutes (simulates reading, thinking)
- **Circadian variation:** Activity varies throughout the day
- **Natural breaks:** Simulates bathroom breaks, coffee breaks, meetings

### 6.5 Implement Mixed Activity Types (Fixes V5, V11, V12, V13)

**Strategy:** Combine keyboard, mouse, and scroll in realistic sequences.

**New actions to add:**

```cpp
// Mouse scroll: simulate reading a document
static void actionMouseScroll() {
  const int scrolls = random(3, 12);
  const int direction = (random(100) < 70) ? -1 : 1;  // 70% down, 30% up

  for (int i = 0; i < scrolls; i++) {
    Mouse.move(0, 0, direction);
    delay(random(80, 200));
  }

  // Sometimes scroll back a bit
  if (random(100) < 30) {
    delay(random(200, 500));
    for (int i = 0; i < random(1, 3); i++) {
      Mouse.move(0, 0, -direction);
      delay(random(80, 200));
    }
  }
}

// Mouse movement: simulate pointing at something
static void actionMouseMove() {
  const int dx = random(-50, 50);
  const int dy = random(-50, 50);

  // Move in steps (human-like)
  const int steps = random(5, 15);
  for (int i = 0; i < steps; i++) {
    int stepDx = dx / steps + random(-2, 2);
    int stepDy = dy / steps + random(-2, 2);
    Mouse.move(stepDx, stepDy);
    delay(random(10, 30));
  }
}

// Click + type: simulate filling a form
static void actionClickType() {
  // Move mouse to approximate location
  Mouse.move(random(-100, 100), random(-100, 100));
  delay(random(200, 500));

  // Click
  Mouse.click();
  delay(random(300, 800));

  // Type something
  const char* text = "test";
  typeTextHuman(Keyboard, text, TYPE_WPM_MIN, TYPE_WPM_MAX);
  delay(random(200, 500));

  // Backspace it
  backspaceHuman(Keyboard, strlen(text));
}

// Right-click: simulate context menu
static void actionRightClick() {
  Mouse.move(random(-50, 50), random(-50, 50));
  delay(random(200, 500));
  Mouse.click(MOUSE_RIGHT);
  delay(random(500, 1500));
  // Press Escape to close menu
  tapKey(Keyboard, KEY_ESC, 25, 70, 60, 160);
}

// Drag-select: simulate text selection
static void actionDragSelect() {
  // Move to start position
  Mouse.move(random(-30, 30), random(-30, 30));
  delay(random(100, 300));

  // Press and hold
  Mouse.press(MOUSE_LEFT);
  delay(random(100, 200));

  // Drag
  const int dx = random(50, 150);
  const int dy = random(-20, 20);
  const int steps = random(5, 10);
  for (int i = 0; i < steps; i++) {
    Mouse.move(dx / steps, dy / steps);
    delay(random(20, 50));
  }

  // Release
  Mouse.release(MOUSE_LEFT);
  delay(random(200, 500));

  // Sometimes delete selected text
  if (random(100) < 50) {
    tapKey(Keyboard, KEY_DELETE, 25, 70, 60, 160);
  }
}
```

**Weight distribution for mixed activity:**

| Action | Active Weight | Meeting Weight |
| --- | --- | --- |
| ArrowScroll | 20 | 20 |
| MouseScroll | 25 | 30 |
| MouseMove | 15 | 10 |
| AltTab | 8 | 0 |
| ClickType | 5 | 0 |
| RightClick | 3 | 0 |
| DragSelect | 3 | 0 |
| Volume | 2 | 2 |
| Brightness | 2 | 2 |
| CapsToggle | 3 | 3 |
| ShiftTap | 10 | 20 |
| WinArrow | 4 | 0 |

### 6.6 Add Variation to Net-Zero Actions (Fixes V6)

**Strategy:** Make net-zero actions less predictable.

**Implementation:**

```cpp
// Volume: sometimes don't reverse, sometimes reverse after delay
static void actionVolume() {
  tapConsumer(Consumer, CONSUMER_CONTROL_VOLUME_INCREMENT, 35, 85);

  // 80% chance of reversal, 20% leave it changed
  if (random(100) < 80) {
    delay(random(80, 2000));  // Variable delay before reversal
    tapConsumer(Consumer, CONSUMER_CONTROL_VOLUME_DECREMENT, 35, 85);
  }
}

// CapsToggle: sometimes toggle multiple times, sometimes leave it
static void actionCapsToggle() {
  const int toggles = random(1, 4);  // 1–3 toggles
  for (int i = 0; i < toggles; i++) {
    tapKey(Keyboard, KEY_CAPS_LOCK, 45, 95, 120, 260);
    delay(random(150, 400));
  }
  // If odd number of toggles, state changed — sometimes leave it
  if (toggles % 2 == 1 && random(100) < 30) {
    // Leave CapsLock on (real user might not notice)
    return;
  }
  // Otherwise toggle back
  if (toggles % 2 == 1) {
    delay(random(200, 600));
    tapKey(Keyboard, KEY_CAPS_LOCK, 45, 95, 60, 180);
  }
}
```

### 6.7 Improve USB Descriptor (Fixes V7, V9)

**Strategy:** Override the default Arduino descriptor with one extracted from a real keyboard.

**Implementation:**

1. Extract the HID report descriptor from a real keyboard using `usbhid-dump` or similar tools
2. Override the Arduino default descriptor

```cpp
// Custom HID report descriptor (extracted from a real keyboard)
static const uint8_t customKeyboardReportDescriptor[] PROGMEM = {
  // Paste the extracted descriptor here
  0x05, 0x01,  // Usage Page (Generic Desktop)
  0x09, 0x06,  // Usage (Keyboard)
  0xA1, 0x01,  // Collection (Application)
  // ... full descriptor ...
  0xC0         // End Collection
};

// Custom mouse report descriptor (extracted from a real mouse)
static const uint8_t customMouseReportDescriptor[] PROGMEM = {
  // Paste the extracted descriptor here
  0x05, 0x01,  // Usage Page (Generic Desktop)
  0x09, 0x02,  // Usage (Mouse)
  0xA1, 0x01,  // Collection (Application)
  // ... full descriptor ...
  0xC0         // End Collection
};
```

**Tools for extraction:**

- `usbhid-dump` — Linux tool for extracting HID descriptors
- `HIDrd` — converts hex format to HID specification format
- USB Descriptor and Request Parser — online tool for parsing descriptors

### 6.8 Improve Serial Number Format (Fixes V8)

**Strategy:** Generate serial numbers that match the format of the spoofed manufacturer.

**Implementation:**

```cpp
// Serial number formats by manufacturer
struct SerialFormat {
  const char* prefix;
  int length;
  bool alphanumeric;
};

static const SerialFormat SERIAL_FORMATS[] = {
  {"CN0", 10, true},      // Dell: CN0XXXXX1234567
  {"CZC", 10, true},      // HP: CZCXXXXX1234567
  {"", 16, true},         // Logitech: alphanumeric
  {"", 12, true},         // Microsoft: alphanumeric
  {"R90", 10, true},      // Lenovo: R90XXXXX1234567
};

void generateSerialNumber(char* buf, size_t len, int identityIndex) {
  const SerialFormat& fmt = SERIAL_FORMATS[identityIndex % USB_IDENTITY_COUNT];

  size_t pos = 0;

  // Copy prefix
  if (fmt.prefix[0] != '\0') {
    size_t prefixLen = strlen(fmt.prefix);
    memcpy(buf, fmt.prefix, prefixLen);
    pos = prefixLen;
  }

  // Generate remaining characters
  for (; pos < len - 1; pos++) {
    if (fmt.alphanumeric) {
      const char chars[] = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ";
      buf[pos] = chars[random(0, sizeof(chars) - 1)];
    } else {
      buf[pos] = '0' + random(0, 10);
    }
  }
  buf[len - 1] = '\0';
}
```

---

## 7. Implementation Roadmap

### Phase 1: Critical Fixes (Week 1–2)

| Task | Fixes | Priority |
| --- | --- | --- |
| Add `USBHIDMouse` instance | V1, V10 | 🔴 Critical |
| Implement log-normal delay distribution | V3 | 🔴 Critical |
| Add variation to net-zero actions | V6 | 🔴 Critical |
| Implement burst/pause work patterns | V4 | 🔴 Critical |

### Phase 2: Movement Humanization (Week 3–4)

| Task | Fixes | Priority |
| --- | --- | --- |
| Implement Brownian motion for mouse | V2 | 🔴 Critical |
| Add mouse scroll actions | V5, V11 | 🔴 Critical |
| Add mouse move actions | V5 | 🔴 Critical |
| Add click + type actions | V5 | 🔴 Critical |
| Add right-click and drag actions | V12, V13 | 🟢 Minor |

### Phase 3: USB Descriptor Hardening (Week 5–6)

| Task | Fixes | Priority |
| --- | --- | --- |
| Extract real keyboard descriptor | V7 | 🟡 Moderate |
| Extract real mouse descriptor | V7 | 🟡 Moderate |
| Override Arduino default descriptors | V7 | 🟡 Moderate |
| Improve serial number format | V8 | 🟡 Moderate |
| Add missing USB descriptors | V9 | 🟡 Moderate |

### Phase 4: Advanced Humanization (Week 7–8)

| Task | Fixes | Priority |
| --- | --- | --- |
| Implement WindMouse algorithm | V2 | 🔴 Critical |
| Add circadian rhythm variation | V4 | 🔴 Critical |
| Implement mixed activity sequences | V5 | 🔴 Critical |
| Add natural break simulation | V4 | 🔴 Critical |

---

## 8. Testing & Validation

### 8.1 Detection Testing

**Test against known detection tools:**

1. **Detect My Jiggler** (Windows) — open-source HID behavioral analysis
2. **Hubstaff Insights** — behavioral thresholds
3. **Insightful Activity Verification** — claimed 99% accuracy
4. **Linux hid-omg-detect** — kernel driver scoring

**Test procedure:**

1. Run EspressoHID for 8 hours on a test machine
2. Install detection tools
3. Check for alerts, flags, or warnings
4. Analyze timing distributions for entropy
5. Review USB device logs for anomalies

### 8.2 Timing Entropy Testing

**Measure timing entropy:**

```python
import numpy as np
from scipy import stats

# Collect timing samples
delays = [...]  # Collect 1000+ delay values

# Calculate entropy
hist, _ = np.histogram(delays, bins=50, density=True)
entropy = stats.entropy(hist)

# Compare to human baseline
# Human typing entropy: ~3.5–4.5 bits
# Uniform random entropy: ~5.6 bits
# Low entropy (<3.0) indicates automation
print(f"Timing entropy: {entropy:.2f} bits")
```

**Target:** Entropy > 3.5 bits (human-like)

### 8.3 Movement Pattern Testing

**Analyze movement patterns:**

```python
import numpy as np

# Collect movement samples
movements = [...]  # Collect 1000+ movement vectors

# Calculate statistics
distances = [np.sqrt(dx**2 + dy**2) for dx, dy in movements]
directions = [np.arctan2(dy, dx) for dx, dy in movements]

# Check for patterns
# Real humans: high variance, irregular direction changes
# Automation: low variance, regular patterns
print(f"Distance variance: {np.var(distances):.2f}")
print(f"Direction variance: {np.var(directions):.2f}")
```

**Target:** High variance in both distance and direction.

### 8.4 USB Descriptor Validation

**Validate USB descriptors:**

```bash
# Linux: extract and compare descriptors
sudo usbhid-dump -d <VID>:<PID> | head -20

# Compare against real device descriptor
# Use HIDrd to convert hex to specification format
hidrd_convert -i hex -o spec < descriptor.hex
```

**Target:** Descriptor matches a real keyboard/mouse exactly.

---

## Appendix A: Detection Tool Reference

| Tool | Type | What It Detects | EspressoHID Status |
| --- | --- | --- | --- |
| Detect My Jiggler | Windows app | HID behavioral patterns | ❌ Detects current version |
| Hubstaff Insights | SaaS | Behavioral thresholds | ❌ Detects current version |
| Insightful | SaaS | Authentic vs simulated input | ❌ Detects current version |
| Time Doctor | SaaS | Unusual activity patterns | ❌ Detects current version |
| Teramind | SaaS | Behavioral analytics | ❌ Detects current version |
| ActivTrak | SaaS | Activity monitoring | ❌ Detects current version |
| Monitask | SaaS | Device audit + screenshots | ⚠️ Partially detects |
| hid-omg-detect | Linux kernel | Timing entropy + descriptors | ❌ Detects current version |
| Windows Defender | OS | Software jigglers | ✅ Bypassed (hardware HID) |

## Appendix B: Human Timing Distributions

| Distribution | Use Case | Parameters | Entropy |
| --- | --- | --- | --- |
| Uniform | ❌ Not recommended | min, max | Low (~2.0 bits) |
| Normal | ⚠️ Acceptable | mu, sigma | Medium (~3.0 bits) |
| Log-normal | ✅ Recommended | mu, sigma | High (~3.5 bits) |
| Gamma | ✅ Recommended | shape, scale | High (~3.5 bits) |
| Weibull | ✅ Recommended | shape, scale | High (~3.5 bits) |

## Appendix C: Movement Humanization Algorithms

| Algorithm | Complexity | Quality | CPU Cost | Recommendation |
| --- | --- | --- | --- | --- |
| Straight line | O(1) | ❌ Poor | Minimal | ❌ Never use |
| Bezier curve | O(n) | ⚠️ Acceptable | Low | ⚠️ Use with noise |
| Brownian motion | O(n) | ✅ Good | Low | ✅ Good for simple cases |
| Perlin noise | O(n) | ✅ Good | Medium | ✅ Good for smooth movement |
| WindMouse | O(n) | ✅ Excellent | Medium | ✅ Best for human-like paths |
| SigmaDrift | O(n) | ✅ Excellent | High | ✅ Best overall quality |

## Appendix D: References

1. mouse-jiggler.org — "Can USB Mouse Jigglers Be Detected?" (2026)
2. mouse-jiggler.org — "Mouse Jiggler Detection Statistics" (2026)
3. overemployedtoolkit.com — "Best Mouse Jiggler in 2026"
4. CurrentWare — "Mouse Jiggler Detection Software"
5. Insightful — "Activity Verification" (2025)
6. Hubstaff — "Unusual Activity Tracking"
7. LWN.net — "HID: add malicious HID device detection driver" (2026)
8. arcanine300/raspberryPi-USB-HID — USB descriptor spoofing
9. ZeroTrace — USB Identity Spoofing documentation
10. WindMouse algorithm — GitHub implementation
11. SigmaDrift — Biomechanically-grounded mouse movement (2026)
12. NestBrowser — "Mouse Trajectory Simulation: Principles and Practices" (2026)
13. SCREENish — "How to Tell If Activity Is Real" (2026)
14. anysecura.com — "How to Detect A Mouse Jiggler"
15. detectmyjiggler.com — Open-source detection tool
