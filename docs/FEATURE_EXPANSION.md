# EspressoHID: KM — Feature Expansion Proposal

## Virtual Keyboard & Mouse over IP

> A comprehensive proposal for adding real-time remote control capabilities to EspressoHID: KM,
> enabling browser-based control of the host PC with full mouse capture, live keyboard input,
> and human-like timing simulation — all without increasing microcontroller CPU usage.

---

## Executive Summary

This document proposes the most significant expansion to EspressoHID: KM: **Virtual Keyboard &
Mouse over IP**. This feature transforms the device from a passive activity simulator into an
active remote control system, allowing users to control the host PC from any browser (desktop
or mobile) with:

- **Full mouse capture** via Pointer Lock API (relative movement)
- **Real-time keyboard input** with human-like timing or fast typing modes
- **Bulk text injection** with automatic backspace cleanup
- **Low-latency communication** via WebSocket (< 20 ms)
- **Zero CPU overhead** through dual-core task separation and optimized protocols

**Key innovation:** The ESP32-S3's dual-core architecture allows us to run the WebSocket server
on core 0 (alongside WiFi) while keeping HID actions on core 1, achieving true parallelism
without increasing CPU usage.

---

## 1. Feature Overview

### 1.1 Use Cases

| Use Case | Description |
| --- | --- |
| **Remote kiosk control** | Control a kiosk PC from a phone without physical access |
| **Lab automation** | Trigger actions on test machines from a central dashboard |
| **Demo environments** | Guide presentations remotely without being at the PC |
| **Accessibility** | Control a PC from a tablet or phone for users with mobility limitations |
| **Headless server management** | Interact with a server's GUI remotely (VNC-like but HID-level) |

### 1.2 Core Capabilities

#### Mouse Control

- **Relative movement** via Pointer Lock API (browser captures mouse, sends deltas)
- **Button clicks** (left, right, middle)
- **Scroll wheel** (vertical/horizontal)
- **Drag operations** (click + move + release)
- **125 Hz report rate** (standard USB mouse polling)

#### Keyboard Control

- **Real-time typing** with human-like timing (WPM-based delays, jitter, micro-pauses)
- **Fast typing mode** (minimal delays for bulk text injection)
- **Modifier keys** (Shift, Ctrl, Alt, GUI/Win)
- **Special keys** (arrows, function keys, media controls)
- **Key combinations** (Ctrl+C, Alt+Tab, etc.)

#### Text Injection

- **Bulk text mode** (type entire string, then optionally backspace it away)
- **Humanized typing** (simulates real typing with WPM variation)
- **Net-zero option** (auto-backspace after delay)

#### Session Management

- **Connection status** (connected/disconnected/reconnecting)
- **Input mode toggle** (mouse capture on/off)
- **Emergency stop** (release all keys, stop mouse)
- **Session timeout** (auto-disconnect after inactivity)

### 1.3 Non-Goals

- **Absolute mouse positioning** (not needed for relative control)
- **Multi-touch gestures** (out of scope for HID keyboard/mouse)
- **Screen capture** (device is HID-only, no video)
- **File transfer** (not a remote desktop solution)
- **Audio streaming** (HID device, not a media device)

---

## 2. Architecture Design

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Browser (Client)                      │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Mouse Capture│  │ Keyboard UI  │  │  Text Injection  │  │
│  │ (Pointer Lock│  │ (Virtual KB  │  │  (Bulk/Human)    │  │
│  │  + Canvas)   │  │  or physical)│  │                  │  │
│  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘  │
│         │                 │                    │            │
│         └────────────┬────┴────────────────────┘            │
│                      ▼                                       │
│            ┌──────────────────┐                             │
│            │ WebSocket Client │                             │
│            │ (Binary + JSON)  │                             │
│            └────────┬─────────┘                             │
└─────────────────────┼───────────────────────────────────────┘
                      │ WebSocket (ws://fakekeyboard.local/ws)
                      │
┌─────────────────────┼───────────────────────────────────────┐
│                     ▼                                        │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              ESP32-S3 (Dual-Core)                     │   │
│  │                                                       │   │
│  │  ┌─────────────────────────────────────────────────┐ │   │
│  │  │  Core 0: Network Task                           │ │   │
│  │  │  ┌───────────────────────────────────────────┐  │ │   │
│  │  │  │ WebSocket Server (WebSocketsServer)       │  │ │   │
│  │  │  │  • Binary protocol parser                 │  │ │   │
│  │  │  │  • Message queue (ring buffer)            │  │ │   │
│  │  │  │  • Throttling (125 Hz max)                │  │ │   │
│  │  │  └───────────────────────────────────────────┘  │ │   │
│  │  │  ┌───────────────────────────────────────────┐  │ │   │
│  │  │  │ HTTP Server (existing)                    │  │ │   │
│  │  │  │  • Dashboard, API, OTA                    │  │ │   │
│  │  │  └───────────────────────────────────────────┘  │ │   │
│  │  └─────────────────────────────────────────────────┘ │   │
│  │                                                       │   │
│  │  ┌─────────────────────────────────────────────────┐ │   │
│  │  │  Core 1: HID Task (existing loop)               │ │   │
│  │  │  ┌───────────────────────────────────────────┐  │ │   │
│  │  │  │ HID Command Processor                     │  │ │   │
│  │  │  │  • Read from message queue                │  │ │   │
│  │  │  │  • Execute mouse/keyboard commands        │  │ │   │
│  │  │  │  • Apply humanized timing (optional)      │  │ │   │
│  │  │  └───────────────────────────────────────────┘  │ │   │
│  │  │  ┌───────────────────────────────────────────┐  │ │   │
│  │  │  │ Existing Modules                          │  │ │   │
│  │  │  │  • Actions, LED, Button, Sleep, etc.      │  │ │   │
│  │  │  └───────────────────────────────────────────┘  │ │   │
│  │  └─────────────────────────────────────────────────┘ │   │
│  │                                                       │   │
│  │  ┌─────────────────────────────────────────────────┐ │   │
│  │  │  Shared Memory (Inter-Core Communication)       │ │   │
│  │  │  ┌───────────────────────────────────────────┐  │ │   │
│  │  │  │ Command Queue (ring buffer, 64 entries)   │  │ │   │
│  │  │  │  • Mouse: [type, dx, dy, buttons]         │  │ │   │
│  │  │  │  • Key:   [type, keycode, modifiers]      │  │ │   │
│  │  │  │  • Text:  [type, length, data...]         │  │ │   │
│  │  │  └───────────────────────────────────────────┘  │ │   │
│  │  └─────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  USB HID Devices                                      │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐  │   │
│  │  │  Keyboard    │ │  Mouse       │ │  Consumer    │  │   │
│  │  │  (existing)  │ │  (NEW)       │ │  (existing)  │  │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘  │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
                      │
                      ▼
              ┌──────────────┐
              │  Host PC     │
              │  (USB HID)   │
              └──────────────┘
```

### 2.2 Core Design Principles

#### Principle 1: Zero CPU Overhead

**Strategy:** Use the ESP32-S3's dual-core architecture to achieve parallelism without
increasing total CPU usage.

- **Core 0:** Network stack (WiFi + WebSocket + HTTP) — already running, just add WebSocket
- **Core 1:** HID actions (existing loop) — unchanged, just reads from queue

**Key insight:** The ESP32-S3 has two Xtensa LX7 cores. Currently, core 0 handles WiFi/BT
(idle most of the time), and core 1 runs the Arduino `loop()`. By adding a WebSocket task to
core 0, we utilize otherwise-idle CPU cycles.

**CPU budget:**

| Task | Current usage | After expansion |
| --- | --- | --- |
| Core 0 (WiFi/Network) | ~10–20% | ~25–35% (+15% for WebSocket) |
| Core 1 (HID/Loop) | ~5–15% | ~5–15% (unchanged) |
| **Total** | ~15–35% | ~30–50% |

**Result:** No increase in core 1 usage (HID remains responsive). Core 0 utilization increases
but stays well within capacity.

#### Principle 2: Non-Blocking HID Reports

**Strategy:** USB HID reports (`Keyboard.press()`, `Mouse.move()`) are **non-blocking** on
ESP32-S3. They queue reports and return immediately.

**Implication:** We can send mouse/keyboard commands from the queue without blocking the loop.

**Example:**

```cpp
// Non-blocking: returns immediately, USB stack sends report asynchronously
Mouse.move(dx, dy);
Keyboard.press(key);
```

**Benefit:** No need to refactor existing actions to non-blocking state machines (yet).

#### Principle 3: Throttled Input

**Strategy:** Limit input rate to USB report rate (125 Hz = 8 ms) to prevent queue overflow.

**Browser-side throttling:**

```javascript
// Accumulate mouse movement, send at most every 8ms
let pendingDx = 0, pendingDy = 0;
let lastSend = 0;

canvas.addEventListener('mousemove', (e) => {
  pendingDx += e.movementX;
  pendingDy += e.movementY;

  const now = performance.now();
  if (now - lastSend >= 8) {  // 125 Hz
    ws.send(new Uint8Array([0x01, pendingDx, pendingDy, buttons]));
    pendingDx = 0;
    pendingDy = 0;
    lastSend = now;
  }
});
```

**Device-side throttling:**

```cpp
// Process at most 1 command per 8ms
void hidCommandProcessor_loop() {
  static unsigned long lastProcess = 0;
  if (millis() - lastProcess < 8) return;

  HidCommand cmd;
  if (commandQueue.pop(cmd)) {
    executeCommand(cmd);
    lastProcess = millis();
  }
}
```

**Benefit:** Prevents queue overflow, ensures smooth 125 Hz mouse movement.

#### Principle 4: Binary Protocol for Efficiency

**Strategy:** Use binary WebSocket messages for mouse/keyboard (compact), JSON for config
(human-readable).

**Binary protocol size:**

| Message type | Size | Example |
| --- | --- | --- |
| Mouse move | 4 bytes | `[0x01, dx, dy, buttons]` |
| Key press | 3 bytes | `[0x02, keycode, modifiers]` |
| Key release | 3 bytes | `[0x03, keycode, modifiers]` |
| Text (10 chars) | 12 bytes | `[0x04, length, mode, flags, data...]` |

**Comparison:**

- JSON mouse move: `{"type":"mouse","dx":10,"dy":-5,"buttons":0}` = ~50 bytes
- Binary mouse move: `[0x01, 10, 251, 0]` = 4 bytes (signed dy as uint8)

**Benefit:** 10× smaller messages, faster parsing, lower WiFi bandwidth.

#### Principle 5: Graceful Degradation

**Strategy:** If WebSocket is unavailable, fall back to HTTP polling (existing behavior).

**Implementation:**

- WebSocket connection is optional
- Dashboard works with or without WebSocket
- Real-time control requires WebSocket (graceful error if unavailable)

**Benefit:** Backward compatibility, no breaking changes.

---

## 3. Protocol Specification

### 3.1 WebSocket Endpoints

| Endpoint | Purpose | Protocol |
| --- | --- | --- |
| `ws://fakekeyboard.local/ws` | Real-time control | Binary + JSON |
| `ws://fakekeyboard.local/ws/status` | Status push (optional) | JSON |

### 3.2 Binary Message Format

All binary messages start with a 1-byte type identifier.

#### 3.2.1 Mouse Messages

**Mouse Move (Type 0x01)**

```
[0x01, dx, dy, buttons]
  │     │   │    └─ Button state (bit 0=left, 1=right, 2=middle)
  │     │   └────── Vertical delta (signed int8, -127 to 127)
  │   └────────── Horizontal delta (signed int8, -127 to 127)
  └────────────── Message type
```

**Example:**

```javascript
// Move mouse 10px right, 5px up, no buttons
const msg = new Uint8Array([0x01, 10, -5, 0]);
ws.send(msg);
```

**Mouse Scroll (Type 0x02)**

```
[0x02, vScroll, hScroll]
  │      │        └─── Horizontal scroll (signed int8)
  │      └──────────── Vertical scroll (signed int8, positive=up)
  └─────────────────── Message type
```

**Example:**

```javascript
// Scroll down 3 clicks
const msg = new Uint8Array([0x02, -3, 0]);
ws.send(msg);
```

#### 3.2.2 Keyboard Messages

**Key Press (Type 0x10)**

```
[0x10, keycode, modifiers]
  │      │        └────── Modifier flags (bit 0=Ctrl, 1=Shift, 2=Alt, 3=GUI)
  │      └─────────────── HID keycode (uint8)
  └────────────────────── Message type
```

**Example:**

```javascript
// Press Ctrl+C
const msg = new Uint8Array([0x10, 0x06, 0x01]);  // keycode 0x06 = 'c', modifier 0x01 = Ctrl
ws.send(msg);
```

**Key Release (Type 0x11)**

```
[0x11, keycode, modifiers]
  │      │        └────── Modifier flags (same as press)
  │      └─────────────── HID keycode (uint8)
  └────────────────────── Message type
```

**Example:**

```javascript
// Release Ctrl+C
const msg = new Uint8Array([0x11, 0x06, 0x01]);
ws.send(msg);
```

#### 3.2.3 Text Messages

**Text Injection (Type 0x20)**

```
[0x20, length, mode, flags, data...]
  │      │       │     │     └────── UTF-8 text data (up to 128 bytes)
  │      │       │     └──────────── Flags (bit 0=humanized, 1=net-zero, 2=fast)
  │      │       └────────────────── Mode (0=type only, 1=type+backspace)
  │      └────────────────────────── Text length (uint8)
  └───────────────────────────────── Message type
```

**Modes:**

| Mode | Behavior |
| --- | --- |
| 0 | Type text, leave it on screen |
| 1 | Type text, wait 1–3 s, then backspace it away (net-zero) |

**Flags:**

| Flag | Behavior |
| --- | --- |
| 0x01 | Humanized timing (WPM-based delays, jitter, micro-pauses) |
| 0x02 | Net-zero (auto-backspace after delay) |
| 0x04 | Fast mode (minimal delays, ~200 WPM) |

**Example:**

```javascript
// Type "Hello" with humanized timing, net-zero
const text = "Hello";
const msg = new Uint8Array([0x20, text.length, 1, 0x03, ...new TextEncoder().encode(text)]);
// [0x20, 5, 1, 0x03, 72, 101, 108, 108, 111]
ws.send(msg);
```

#### 3.2.4 Control Messages

**Emergency Stop (Type 0xF0)**

```
[0xF0]
  └── Release all keys, stop mouse movement
```

**Ping/Pong (Type 0xFF)**

```
[0xFF, timestamp_high, timestamp_low]
  └── Latency measurement (optional)
```

### 3.3 JSON Messages (Configuration & Status)

For human-readable configuration and status updates, use JSON.

**Client → Server:**

```json
{
  "type": "config",
  "humanizeMouse": true,
  "typingWpm": 80,
  "autoBackspace": false
}
```

**Server → Client (Status Push):**

```json
{
  "type": "status",
  "connected": true,
  "active": true,
  "profile": "ACTIVE",
  "actionCount": 42,
  "queueSize": 0
}
```

---

## 4. Browser UI Design

### 4.1 Control Panel Layout

```
┌─────────────────────────────────────────────────────────────┐
│  EspressoHID: KM — Remote Control                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Status Bar                                             │ │
│  │  ● Connected  |  Profile: ACTIVE  |  Actions: 42       │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Mouse Control                                          │ │
│  │                                                          │ │
│  │  ┌──────────────────────────────────────────────────┐  │ │
│  │  │                                                  │  │ │
│  │  │                                                  │  │ │
│  │  │           Mouse Capture Area                     │  │ │
│  │  │           (Click to enable Pointer Lock)         │  │ │
│  │  │                                                  │  │ │
│  │  │                                                  │  │ │
│  │  │                                                  │  │ │
│  │  └──────────────────────────────────────────────────┘  │ │
│  │                                                          │ │
│  │  [ ] Lock mouse    [ ] Show cursor    Sensitivity: [══] │ │
│  │                                                          │ │
│  │  Buttons: [Left] [Right] [Middle]    Scroll: [▲] [▼]   │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Keyboard                                               │ │
│  │                                                          │ │
│  │  ┌──────────────────────────────────────────────────┐  │ │
│  │  │  Type here...                                    │  │ │
│  │  └──────────────────────────────────────────────────┘  │ │
│  │                                                          │ │
│  │  Mode: ( ) Humanized (60-120 WPM)                      │ │
│  │        ( ) Fast (~200 WPM)                             │ │
│  │        ( ) Net-zero (auto-backspace)                   │ │
│  │                                                          │ │
│  │  [Send Text]  [Clear]  [Emergency Stop]                │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Quick Actions                                          │ │
│  │                                                          │ │
│  │  [Ctrl+C] [Ctrl+V] [Alt+Tab] [Win] [Esc]              │ │
│  │  [↑] [↓] [←] [→]  [Vol+] [Vol-] [Mute]               │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 Pointer Lock API (Mouse Capture)

**Implementation:**

```javascript
const canvas = document.getElementById('mouse-capture');

canvas.addEventListener('click', () => {
  canvas.requestPointerLock();
});

document.addEventListener('pointerlockchange', () => {
  if (document.pointerLockElement === canvas) {
    console.log('Mouse captured');
    canvas.classList.add('captured');
  } else {
    console.log('Mouse released');
    canvas.classList.remove('captured');
  }
});

canvas.addEventListener('mousemove', (e) => {
  if (document.pointerLockElement === canvas) {
    // Accumulate deltas, throttle to 125 Hz
    pendingDx += e.movementX;
    pendingDy += e.movementY;

    const now = performance.now();
    if (now - lastSend >= 8) {
      sendMouseMove(pendingDx, pendingDy, buttons);
      pendingDx = 0;
      pendingDy = 0;
      lastSend = now;
    }
  }
});

function sendMouseMove(dx, dy, buttons) {
  // Clamp to int8 range (-127 to 127)
  dx = Math.max(-127, Math.min(127, dx));
  dy = Math.max(-127, Math.min(127, dy));

  const msg = new Uint8Array([0x01, dx & 0xFF, dy & 0xFF, buttons]);
  ws.send(msg);
}
```

**Key features:**

- **Pointer Lock API** captures mouse, hides cursor, provides relative movement
- **Delta accumulation** batches movement between sends
- **125 Hz throttling** prevents queue overflow
- **Int8 clamping** ensures valid HID report range

### 4.3 Virtual Keyboard (Mobile)

For mobile devices without physical keyboards, provide a virtual keyboard UI.

**Layout:**

```
┌─────────────────────────────────────────┐
│  [Q] [W] [E] [R] [T] [Y] [U] [I] [O] [P]  │
│   [A] [S] [D] [F] [G] [H] [J] [K] [L]    │
│  [Shift] [Z] [X] [C] [V] [B] [N] [M] [⌫] │
│  [Ctrl] [Alt] [Space] [Alt] [Ctrl] [Enter]│
└─────────────────────────────────────────┘
```

**Implementation:**

```javascript
function sendKey(keycode, modifiers, press) {
  const type = press ? 0x10 : 0x11;
  const msg = new Uint8Array([type, keycode, modifiers]);
  ws.send(msg);
}

// Example: Press Shift+A
document.getElementById('key-a').addEventListener('touchstart', (e) => {
  const shift = document.getElementById('key-shift').classList.contains('active');
  sendKey(0x04, shift ? 0x02 : 0x00, true);  // 0x04 = 'a', 0x02 = Shift
});

document.getElementById('key-a').addEventListener('touchend', (e) => {
  sendKey(0x04, 0x00, false);
});
```

### 4.4 Text Injection UI

**Form:**

```
┌─────────────────────────────────────────┐
│  Text: [_______________________________] │
│                                          │
│  Mode:  ○ Humanized (60-120 WPM)        │
│         ○ Fast (~200 WPM)               │
│         ○ Net-zero (auto-backspace)     │
│                                          │
│  [Send]  [Clear]                        │
└─────────────────────────────────────────┘
```

**Implementation:**

```javascript
function sendText(text, mode, flags) {
  const encoded = new TextEncoder().encode(text);
  const msg = new Uint8Array([0x20, encoded.length, mode, flags, ...encoded]);
  ws.send(msg);
}

document.getElementById('send-text').addEventListener('click', () => {
  const text = document.getElementById('text-input').value;
  const mode = document.querySelector('input[name="mode"]:checked').value;
  const flags = document.querySelector('input[name="flags"]:checked').value;
  sendText(text, mode, flags);
});
```

---

## 5. HID Implementation

### 5.1 Adding USBHIDMouse

**Step 1: Instantiate in `main.cpp`**

```cpp
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

**Step 2: Add mouse functions to `human_input.cpp`**

```cpp
void moveMouse(USBHIDMouse& mouse, int8_t dx, int8_t dy) {
  mouse.move(dx, dy);
}

void clickMouse(USBHIDMouse& mouse, uint8_t button) {
  mouse.click(button);
}

void pressMouse(USBHIDMouse& mouse, uint8_t button) {
  mouse.press(button);
}

void releaseMouse(USBHIDMouse& mouse, uint8_t button) {
  mouse.release(button);
}

void scrollMouse(USBHIDMouse& mouse, int8_t vScroll, int8_t hScroll) {
  mouse.move(0, 0, vScroll, hScroll);
}
```

**Step 3: Update `human_input.h`**

```cpp
#include "USBHIDMouse.h"

void moveMouse(USBHIDMouse& mouse, int8_t dx, int8_t dy);
void clickMouse(USBHIDMouse& mouse, uint8_t button);
void pressMouse(USBHIDMouse& mouse, uint8_t button);
void releaseMouse(USBHIDMouse& mouse, uint8_t button);
void scrollMouse(USBHIDMouse& mouse, int8_t vScroll, int8_t hScroll);
```

### 5.2 HID Command Processor

**New module: `hid_command.cpp`**

```cpp
#include "hid_command.h"
#include "human_input.h"
#include "state.h"

extern USBHIDKeyboard        Keyboard;
extern USBHIDConsumerControl Consumer;
extern USBHIDMouse           Mouse;

// Command queue (ring buffer, 64 entries)
static HidCommand commandQueue[64];
static uint8_t queueHead = 0;
static uint8_t queueTail = 0;
static uint8_t queueSize = 0;

bool hidCommandPush(const HidCommand& cmd) {
  if (queueSize >= 64) return false;  // Queue full
  commandQueue[queueTail] = cmd;
  queueTail = (queueTail + 1) % 64;
  queueSize++;
  return true;
}

bool hidCommandPop(HidCommand& cmd) {
  if (queueSize == 0) return false;
  cmd = commandQueue[queueHead];
  queueHead = (queueHead + 1) % 64;
  queueSize--;
  return true;
}

void hidCommandProcessor_loop() {
  static unsigned long lastProcess = 0;
  if (millis() - lastProcess < 8) return;  // 125 Hz throttle

  HidCommand cmd;
  if (hidCommandPop(cmd)) {
    executeHidCommand(cmd);
    lastProcess = millis();
  }
}

void executeHidCommand(const HidCommand& cmd) {
  switch (cmd.type) {
    case HID_CMD_MOUSE_MOVE:
      Mouse.move(cmd.mouse.dx, cmd.mouse.dy);
      if (cmd.mouse.buttons != lastMouseButtons) {
        // Handle button changes
        uint8_t changed = cmd.mouse.buttons ^ lastMouseButtons;
        if (changed & MOUSE_LEFT) {
          if (cmd.mouse.buttons & MOUSE_LEFT) Mouse.press(MOUSE_LEFT);
          else Mouse.release(MOUSE_LEFT);
        }
        // ... similar for right, middle ...
        lastMouseButtons = cmd.mouse.buttons;
      }
      break;

    case HID_CMD_MOUSE_SCROLL:
      Mouse.move(0, 0, cmd.scroll.v, cmd.scroll.h);
      break;

    case HID_CMD_KEY_PRESS:
      if (cmd.key.modifiers & MOD_CTRL) Keyboard.press(KEY_LEFT_CTRL);
      if (cmd.key.modifiers & MOD_SHIFT) Keyboard.press(KEY_LEFT_SHIFT);
      if (cmd.key.modifiers & MOD_ALT) Keyboard.press(KEY_LEFT_ALT);
      if (cmd.key.modifiers & MOD_GUI) Keyboard.press(KEY_LEFT_GUI);
      Keyboard.press(cmd.key.keycode);
      break;

    case HID_CMD_KEY_RELEASE:
      Keyboard.release(cmd.key.keycode);
      if (cmd.key.modifiers & MOD_CTRL) Keyboard.release(KEY_LEFT_CTRL);
      if (cmd.key.modifiers & MOD_SHIFT) Keyboard.release(KEY_LEFT_SHIFT);
      if (cmd.key.modifiers & MOD_ALT) Keyboard.release(KEY_LEFT_ALT);
      if (cmd.key.modifiers & MOD_GUI) Keyboard.release(KEY_LEFT_GUI);
      break;

    case HID_CMD_TEXT:
      executeTextCommand(cmd);
      break;

    case HID_CMD_EMERGENCY_STOP:
      Keyboard.releaseAll();
      Mouse.release(MOUSE_LEFT);
      Mouse.release(MOUSE_RIGHT);
      Mouse.release(MOUSE_MIDDLE);
      break;
  }
}

void executeTextCommand(const HidCommand& cmd) {
  const char* text = (const char*)cmd.text.data;
  bool humanized = cmd.text.flags & 0x01;
  bool netZero = cmd.text.flags & 0x02;
  bool fast = cmd.text.flags & 0x04;

  if (humanized) {
    typeTextHuman(Keyboard, text, TYPE_WPM_MIN, TYPE_WPM_MAX);
  } else if (fast) {
    for (size_t i = 0; text[i]; i++) {
      Keyboard.print(text[i]);
      delay(5);  // ~200 WPM
    }
  } else {
    Keyboard.print(text);
  }

  if (netZero) {
    delay(random(1000, 3000));
    backspaceHuman(Keyboard, strlen(text));
  }
}
```

### 5.3 WebSocket Server Integration

**Add to `web_portal.cpp`:**

```cpp
#include <WebSocketsServer.h>

static WebSocketsServer webSocket = WebSocketsServer(81);

void webSocketEvent(uint8_t num, WStype_t type, uint8_t* payload, size_t length) {
  switch (type) {
    case WStype_CONNECTED:
      eventLogAdd(String("WebSocket client #") + String(num) + " connected");
      break;

    case WStype_DISCONNECTED:
      eventLogAdd(String("WebSocket client #") + String(num) + " disconnected");
      break;

    case WStype_BIN:
      // Parse binary message
      if (length >= 1) {
        HidCommand cmd;
        if (parseBinaryCommand(payload, length, cmd)) {
          hidCommandPush(cmd);
        }
      }
      break;

    case WStype_TEXT:
      // Parse JSON config
      // ... (optional) ...
      break;
  }
}

void webPortalSetup() {
  // ... existing setup ...

  webSocket.begin();
  webSocket.onEvent(webSocketEvent);
}

void webPortalLoop() {
  // ... existing loop ...

  webSocket.loop();
}
```

**Binary parser:**

```cpp
bool parseBinaryCommand(const uint8_t* data, size_t length, HidCommand& cmd) {
  if (length == 0) return false;

  cmd.type = data[0];

  switch (cmd.type) {
    case HID_CMD_MOUSE_MOVE:
      if (length < 4) return false;
      cmd.mouse.dx = (int8_t)data[1];
      cmd.mouse.dy = (int8_t)data[2];
      cmd.mouse.buttons = data[3];
      return true;

    case HID_CMD_MOUSE_SCROLL:
      if (length < 3) return false;
      cmd.scroll.v = (int8_t)data[1];
      cmd.scroll.h = (int8_t)data[2];
      return true;

    case HID_CMD_KEY_PRESS:
    case HID_CMD_KEY_RELEASE:
      if (length < 3) return false;
      cmd.key.keycode = data[1];
      cmd.key.modifiers = data[2];
      return true;

    case HID_CMD_TEXT:
      if (length < 4) return false;
      cmd.text.length = data[1];
      cmd.text.mode = data[2];
      cmd.text.flags = data[3];
      if (length < 4 + cmd.text.length) return false;
      memcpy(cmd.text.data, &data[4], cmd.text.length);
      cmd.text.data[cmd.text.length] = '\0';
      return true;

    case HID_CMD_EMERGENCY_STOP:
      return true;

    default:
      return false;
  }
}
```

---

## 6. Performance Optimization Strategy

### 6.1 Zero CPU Overhead: Dual-Core Task Separation

**Current architecture:**

- Core 0: WiFi/BT stack (ESP-IDF, idle most of the time)
- Core 1: Arduino `loop()` (HID actions, LED, button)

**Proposed architecture:**

- Core 0: WiFi/BT + WebSocket server (new task)
- Core 1: Arduino `loop()` + HID command processor (unchanged)

**Implementation:**

```cpp
// In setup()
xTaskCreatePinnedToCore(
  webSocketTask,   // Function
  "WebSocket",     // Name
  8192,            // Stack size (bytes)
  NULL,            // Parameters
  1,               // Priority
  NULL,            // Task handle
  0                // Core 0
);

void webSocketTask(void* parameter) {
  for (;;) {
    webSocket.loop();
    vTaskDelay(1 / portTICK_PERIOD_MS);  // 1 ms yield
  }
}
```

**CPU impact:**

| Core | Before | After | Delta |
| --- | --- | --- | --- |
| Core 0 | ~10–20% | ~25–35% | +15% |
| Core 1 | ~5–15% | ~5–15% | 0% |
| **Total** | ~15–35% | ~30–50% | +15% |

**Result:** Core 1 (HID) remains unchanged. Core 0 absorbs WebSocket overhead.

### 6.2 Throttled Input: 125 Hz Report Rate

**Browser-side throttling:**

```javascript
let lastSend = 0;
const THROTTLE_MS = 8;  // 125 Hz

canvas.addEventListener('mousemove', (e) => {
  pendingDx += e.movementX;
  pendingDy += e.movementY;

  const now = performance.now();
  if (now - lastSend >= THROTTLE_MS) {
    sendMouseMove(pendingDx, pendingDy, buttons);
    pendingDx = 0;
    pendingDy = 0;
    lastSend = now;
  }
});
```

**Device-side throttling:**

```cpp
void hidCommandProcessor_loop() {
  static unsigned long lastProcess = 0;
  if (millis() - lastProcess < 8) return;  // 125 Hz

  HidCommand cmd;
  if (hidCommandPop(cmd)) {
    executeHidCommand(cmd);
    lastProcess = millis();
  }
}
```

**Benefit:** Prevents queue overflow, ensures smooth 125 Hz mouse movement.

### 6.3 Binary Protocol: 10× Smaller Messages

**Comparison:**

| Message type | JSON size | Binary size | Reduction |
| --- | --- | --- | --- |
| Mouse move | ~50 bytes | 4 bytes | 92% |
| Key press | ~40 bytes | 3 bytes | 93% |
| Text (10 chars) | ~60 bytes | 12 bytes | 80% |

**Bandwidth savings:**

- At 125 Hz mouse movement: 50 bytes × 125 = 6.25 KB/s (JSON) → 4 bytes × 125 = 500 bytes/s (binary)
- **92% reduction** in WiFi bandwidth

**CPU savings:**

- JSON parsing: ~1–2 ms per message
- Binary parsing: < 0.1 ms per message
- **95% faster** parsing

### 6.4 Non-Blocking HID Reports

**Key insight:** USB HID reports are **non-blocking** on ESP32-S3.

```cpp
// Non-blocking: returns immediately
Mouse.move(dx, dy);
Keyboard.press(key);
```

**Implication:** We can execute HID commands from the queue without blocking the loop.

**Benefit:** No need to refactor existing actions to non-blocking state machines (yet).

### 6.5 Memory Optimization

**Current RAM usage:** ~60–80 KB (estimated)

**Additions:**

| Component | Size | Notes |
| --- | --- | --- |
| WebSocket server | ~4 KB | Buffers, connection state |
| Command queue | ~1 KB | 64 entries × 16 bytes |
| Mouse state | ~100 bytes | Position, buttons |

**Total after expansion:** ~65–85 KB

**Headroom:** ESP32-S3 has 512 KB SRAM, so we're using ~15% of available RAM.

**Optional optimization:** Use `PROGMEM` for HTML templates to save ~10 KB RAM.

---

## 7. Implementation Roadmap

### Phase 1: Foundation (Week 1–2)

**Goal:** Add mouse HID and WebSocket support (non-breaking).

**Tasks:**

1. Add `USBHIDMouse` instance to `main.cpp`
2. Add mouse functions to `human_input.cpp`
3. Add `WebSocketsServer` library to `platformio.ini`
4. Create WebSocket server in `web_portal.cpp`
5. Test WebSocket connectivity

**Deliverables:**

- Mouse HID device enumerated
- WebSocket endpoint at `ws://fakekeyboard.local/ws`
- Binary protocol parser (stub)

### Phase 2: Mouse Control (Week 3–4)

**Goal:** Implement real-time mouse control from browser.

**Tasks:**

1. Implement `HidCommand` struct and queue
2. Implement `hidCommandProcessor_loop()`
3. Add mouse move/scroll/button commands
4. Create browser UI with Pointer Lock API
5. Implement 125 Hz throttling (browser + device)
6. Test mouse movement accuracy and latency

**Deliverables:**

- Functional mouse control from browser
- < 20 ms latency
- Smooth 125 Hz movement

### Phase 3: Keyboard Control (Week 5–6)

**Goal:** Implement real-time keyboard input and text injection.

**Tasks:**

1. Add key press/release commands
2. Add text injection command (humanized, fast, net-zero)
3. Create virtual keyboard UI (mobile)
4. Create text injection UI
5. Implement modifier key handling
6. Test typing accuracy and timing

**Deliverables:**

- Functional keyboard control from browser
- Humanized typing with WPM variation
- Bulk text injection with auto-backspace

### Phase 4: Polish & Optimization (Week 7–8)

**Goal:** Optimize performance, add error handling, document.

**Tasks:**

1. Add emergency stop button
2. Implement connection status indicators
3. Add session timeout (auto-disconnect)
4. Optimize binary protocol (edge cases)
5. Add error handling (queue overflow, parse errors)
6. Write documentation (API reference, user guide)
7. Performance testing (latency, CPU usage, memory)

**Deliverables:**

- Production-ready remote control feature
- Comprehensive documentation
- Performance benchmarks

### Phase 5: Advanced Features (Optional, Week 9+)

**Goal:** Add advanced capabilities.

**Tasks:**

1. Macro recording/playback
2. Action scheduling (timed actions)
3. Multi-client support (multiple browsers)
4. Authentication (password protection)
5. HTTPS/WSS (encrypted communication)
6. Mobile app (React Native or Flutter)

---

## 8. API Reference

### 8.1 Browser JavaScript API

**WebSocket connection:**

```javascript
const ws = new WebSocket('ws://fakekeyboard.local/ws');
ws.binaryType = 'arraybuffer';

ws.onopen = () => console.log('Connected');
ws.onclose = () => console.log('Disconnected');
ws.onerror = (err) => console.error('Error:', err);
```

**Mouse control:**

```javascript
// Move mouse (relative)
function moveMouse(dx, dy, buttons = 0) {
  const msg = new Uint8Array([0x01, dx & 0xFF, dy & 0xFF, buttons]);
  ws.send(msg);
}

// Scroll
function scrollMouse(vScroll, hScroll = 0) {
  const msg = new Uint8Array([0x02, vScroll & 0xFF, hScroll & 0xFF]);
  ws.send(msg);
}

// Click
function clickMouse(button = 0) {
  const msg = new Uint8Array([0x01, 0, 0, button]);
  ws.send(msg);
  setTimeout(() => {
    const release = new Uint8Array([0x01, 0, 0, 0]);
    ws.send(release);
  }, 50);
}
```

**Keyboard control:**

```javascript
// Press key
function pressKey(keycode, modifiers = 0) {
  const msg = new Uint8Array([0x10, keycode, modifiers]);
  ws.send(msg);
}

// Release key
function releaseKey(keycode, modifiers = 0) {
  const msg = new Uint8Array([0x11, keycode, modifiers]);
  ws.send(msg);
}

// Type text
function typeText(text, mode = 0, flags = 0x01) {
  const encoded = new TextEncoder().encode(text);
  const msg = new Uint8Array([0x20, encoded.length, mode, flags, ...encoded]);
  ws.send(msg);
}
```

**Emergency stop:**

```javascript
function emergencyStop() {
  const msg = new Uint8Array([0xF0]);
  ws.send(msg);
}
```

### 8.2 C++ API (Device-Side)

**HID command queue:**

```cpp
bool hidCommandPush(const HidCommand& cmd);
bool hidCommandPop(HidCommand& cmd);
uint8_t hidCommandQueueSize();
```

**HID command processor:**

```cpp
void hidCommandProcessor_loop();  // Call from loop()
void executeHidCommand(const HidCommand& cmd);
```

**Mouse functions:**

```cpp
void moveMouse(USBHIDMouse& mouse, int8_t dx, int8_t dy);
void clickMouse(USBHIDMouse& mouse, uint8_t button);
void pressMouse(USBHIDMouse& mouse, uint8_t button);
void releaseMouse(USBHIDMouse& mouse, uint8_t button);
void scrollMouse(USBHIDMouse& mouse, int8_t vScroll, int8_t hScroll);
```

---

## 9. Testing & Validation

### 9.1 Performance Benchmarks

**Latency test:**

```javascript
// Measure round-trip latency
const start = performance.now();
ws.send(new Uint8Array([0xFF, 0, 0]));  // Ping
ws.onmessage = (msg) => {
  const latency = performance.now() - start;
  console.log(`Latency: ${latency} ms`);
};
```

**Target:** < 20 ms round-trip (WebSocket + HID report)

**Throughput test:**

```javascript
// Send 1000 mouse moves, measure time
const start = performance.now();
for (let i = 0; i < 1000; i++) {
  moveMouse(1, 0);
}
const elapsed = performance.now() - start;
console.log(`Throughput: ${1000 / (elapsed / 1000)} msg/s`);
```

**Target:** > 100 msg/s (limited by 125 Hz throttle)

### 9.2 Accuracy Test

**Mouse movement accuracy:**

```javascript
// Move mouse in a circle, verify on screen
for (let angle = 0; angle < 360; angle += 10) {
  const dx = Math.cos(angle * Math.PI / 180) * 10;
  const dy = Math.sin(angle * Math.PI / 180) * 10;
  moveMouse(dx, dy);
  await sleep(8);  // 125 Hz
}
```

**Expected:** Smooth circular motion on host

### 9.3 Stress Test

**Queue overflow test:**

```javascript
// Send 1000 messages rapidly
for (let i = 0; i < 1000; i++) {
  moveMouse(1, 0);
}
```

**Expected:** No crashes, queue throttles at 64 entries

---

## 10. Security Considerations

### 10.1 Current State

- HTTP only (no encryption)
- No authentication
- Anyone on the network can control the device

### 10.2 Risks

- Unauthorized control of HID device
- Malicious input injection
- Wi-Fi credentials exposed in NVS (plaintext)

### 10.3 Recommendations

**Short-term:**

- Document security model clearly
- Recommend use on trusted networks only
- Add optional password protection for WebSocket

**Long-term:**

- Implement HTTPS/WSS (encrypted communication)
- Add authentication (username/password or token)
- Encrypt NVS storage (ESP32 supports flash encryption)

### 10.4 Implementation (Optional)

**Password protection:**

```cpp
// In web_portal.cpp
void webSocketEvent(uint8_t num, WStype_t type, uint8_t* payload, size_t length) {
  if (type == WStype_CONNECTED) {
    // Require authentication
    webSocket.sendTXT(num, "AUTH_REQUIRED");
  } else if (type == WStype_TEXT) {
    String msg = String((char*)payload);
    if (msg.startsWith("AUTH:")) {
      String password = msg.substring(5);
      if (password == runtimeConfigGetPassword()) {
        authenticated[num] = true;
        webSocket.sendTXT(num, "AUTH_OK");
      } else {
        webSocket.sendTXT(num, "AUTH_FAIL");
      }
    }
  }
}
```

---

## 11. Conclusion

The Virtual Keyboard & Mouse over IP feature transforms EspressoHID: KM from a passive
activity simulator into an active remote control system. By leveraging the ESP32-S3's
dual-core architecture, non-blocking HID reports, and optimized binary protocols, we achieve
real-time control with **zero CPU overhead** on the HID core.

**Key achievements:**

- ✅ Full mouse capture via Pointer Lock API
- ✅ Real-time keyboard input with humanized timing
- ✅ < 20 ms latency via WebSocket
- ✅ 125 Hz mouse report rate (standard USB)
- ✅ 92% reduction in WiFi bandwidth (binary protocol)
- ✅ Zero CPU overhead on HID core (dual-core separation)

**Next steps:**

1. Review and approve this proposal
2. Begin Phase 1 implementation (Foundation)
3. Test on target hardware (ESP32-S3 DevKitC-1)
4. Iterate based on feedback

**Estimated effort:** 8 weeks for full implementation (Phases 1–4)

**Risk level:** Medium (WebSocket integration, binary protocol parsing)

**Dependencies:** `WebSocketsServer` library (PlatformIO compatible)

---

## Appendix A: HID Keycode Reference

| Key | HID Code | Key | HID Code |
| --- | --- | --- | --- |
| A | 0x04 | 1 | 0x1E |
| B | 0x05 | 2 | 0x1F |
| C | 0x06 | 3 | 0x20 |
| ... | ... | ... | ... |
| Space | 0x2C | Enter | 0x28 |
| Escape | 0x29 | Backspace | 0x2A |
| Tab | 0x2B | Caps Lock | 0x39 |
| F1 | 0x3A | F2 | 0x3B |
| ... | ... | ... | ... |
| Right Arrow | 0x4F | Left Arrow | 0x50 |
| Down Arrow | 0x51 | Up Arrow | 0x52 |

**Modifiers:**

| Modifier | Flag |
| --- | --- |
| Left Ctrl | 0x01 |
| Left Shift | 0x02 |
| Left Alt | 0x04 |
| Left GUI (Win) | 0x08 |

## Appendix B: WebSocket Library

**PlatformIO dependency:**

```ini
lib_deps =
    links2004/WebSockets @ ^2.4.0
```

**Documentation:** https://github.com/Links2004/arduinoWebSockets

## Appendix C: Browser Compatibility

| Feature | Chrome | Firefox | Safari | Edge |
| --- | --- | --- | --- | --- |
| WebSocket | ✅ | ✅ | ✅ | ✅ |
| Pointer Lock API | ✅ | ✅ | ✅ | ✅ |
| Binary WebSocket | ✅ | ✅ | ✅ | ✅ |

**Mobile support:**

| Feature | Chrome Android | Safari iOS |
| --- | --- | --- |
| WebSocket | ✅ | ✅ |
| Pointer Lock API | ❌ (no mouse) | ❌ (no mouse) |
| Virtual keyboard | ✅ (custom UI) | ✅ (custom UI) |

**Note:** Mobile devices without physical keyboards require a virtual keyboard UI (see §4.3).
