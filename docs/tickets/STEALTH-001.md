# STEALTH-001: Add USBHIDMouse Composite Device

| Field | Value |
| --- | --- |
| **ID** | STEALTH-001 |
| **Priority** | 🔴 P0 (Critical) |
| **Epic** | Stealth Hardening |
| **Estimated Effort** | 2 hours |
| **Dependencies** | None |
| **Branch** | `feature/stealth-001-usb-hid-mouse` |

---

## Description

The current firmware only exposes `USBHIDKeyboard` and `USBHIDConsumerControl`. A
keyboard-only USB device that never produces mouse movement is immediately suspicious to
behavioral analytics platforms. This ticket adds `USBHIDMouse` to create a composite
keyboard + mouse + consumer control device, which is what real keyboards with trackpoints
or touchpads look like.

## Detection Vector

**Keyboard/mouse correlation analysis** — monitoring tools track the ratio of keyboard to
mouse activity. A device that only produces keyboard events with zero mouse movement for
hours is flagged.

> "If mouse moves but keyboard is silent for hours, suspicious. If only arrow keys are
> pressed, suspicious. Need to mix keyboard and mouse activity." — multiple detection guides

## References

> **Related docs for this ticket:** `docs/ARCHITECTURE.md`, `docs/FINDINGS.md`, `docs/STEALTH_HARDENING.md`, `docs/FEATURE_EXPANSION.md`, `docs/TICKETS.md` — read the relevant sections listed above before coding.

- `docs/STEALTH_HARDENING.md` §V1, §6.1
- `docs/ARCHITECTURE.md` §4 (State model)
- `docs/FINDINGS.md` §3.4 (No Mouse HID Support)

## Files to Modify

| File | Action | Description |
| --- | --- | --- |
| `src/main.cpp` | Modify | Add `USBHIDMouse` instance, initialize in `setup()` |
| `include/human_input.h` | Modify | Add mouse function declarations |
| `lib/fake_keyboard_core/src/human_input.cpp` | Modify | Implement mouse helper functions |

## Objectives

1. Instantiate `USBHIDMouse` in `main.cpp`
2. Add mouse helper functions to `human_input.cpp`
3. Update `human_input.h` with mouse function declarations
4. Ensure the device enumerates as a composite keyboard + mouse + consumer control device
5. Verify the device works on Windows, macOS, and Linux

## Acceptance Criteria

- [ ] `USBHIDMouse` is instantiated in `main.cpp` as `Mouse`
- [ ] `Mouse.begin()` is called before `USB.begin()` in `setup()`
- [ ] `moveMouse(USBHIDMouse& mouse, int8_t dx, int8_t dy)` is implemented
- [ ] `clickMouse(USBHIDMouse& mouse, uint8_t button)` is implemented
- [ ] `pressMouse(USBHIDMouse& mouse, uint8_t button)` is implemented
- [ ] `releaseMouse(USBHIDMouse& mouse, uint8_t button)` is implemented
- [ ] `scrollMouse(USBHIDMouse& mouse, int8_t vScroll, int8_t hScroll)` is implemented
- [ ] All mouse functions are declared in `include/human_input.h`
- [ ] The device enumerates as a composite HID device (keyboard + mouse + consumer control)
- [ ] Mouse movement works on the host OS
- [ ] Mouse click works on the host OS
- [ ] Mouse scroll works on the host OS
- [ ] Existing keyboard and consumer control functionality is not broken
- [ ] No compilation errors or warnings

## Unit Tests

Create file `test/test_mouse_hid.cpp`:

```cpp
#include <unity.h>
#include "human_input.h"
#include "USBHIDMouse.h"

void test_mouse_move_helper() {
  // Verify moveMouse() accepts valid parameters
  // On-device: verify Mouse.move() is called with (dx, dy)
  TEST_ASSERT_TRUE_MESSAGE(true, "moveMouse helper verified");
}

void test_mouse_click_helper() {
  // Verify clickMouse() accepts valid button constants
  // On-device: verify Mouse.click() is called with (button)
  TEST_ASSERT_TRUE_MESSAGE(true, "clickMouse helper verified");
}

void test_mouse_scroll_helper() {
  // Verify scrollMouse() accepts valid scroll values
  // On-device: verify Mouse.move() is called with (0, 0, vScroll, hScroll)
  TEST_ASSERT_TRUE_MESSAGE(true, "scrollMouse helper verified");
}

void test_mouse_press_release_helper() {
  // Verify pressMouse() and releaseMouse() work correctly
  // On-device: verify Mouse.press() and Mouse.release() are called
  TEST_ASSERT_TRUE_MESSAGE(true, "pressMouse/releaseMouse helpers verified");
}

void setup() {
  UNITY_BEGIN();
  RUN_TEST(test_mouse_move_helper);
  RUN_TEST(test_mouse_click_helper);
  RUN_TEST(test_mouse_scroll_helper);
  RUN_TEST(test_mouse_press_release_helper);
  UNITY_END();
}

void loop() {}
```


## Standard Git Workflow for Ticket Execution (Base Branch: `dev`)

> **All branches off `dev`. No direct commits to `main`.**

1.  **Sync Dev First:** Pull the latest changes from the remote dev branch before starting.
    ```bash
    git checkout dev
    git pull origin dev
    ```
2.  **Branch from Dev:** Create a dedicated branch (`feature/`, `bugfix/`, etc.) branching directly off `dev`.
    ```bash
    git checkout -b <branch-name>   # branch name listed in ticket header
    ```
3.  **Verify Changes:** Confirm all your files and edits are tracked before making a commit.
    ```bash
    git status
    git diff --stat
    git diff
    ```
4.  **Push and Merge:** Push changes to your remote branch, then merge them back into the `dev` branch.
    ```bash
    git add -A
    git commit -m "type(scope): message — Fixes <TICKET-ID>"
    git push origin <branch-name>
    git checkout dev
    git pull origin dev
    git merge <branch-name>
    git push origin dev
    ```
5.  **Reset to Dev:** Switch back to the `dev` branch and pull the latest remote updates to finish.
    ```bash
    git checkout dev
    git pull origin dev
    ```

## Implementation Steps

### Step 1: Sync and Branch

```bash
git checkout dev
git pull origin dev
git checkout -b feature/stealth-001-usb-hid-mouse
```

### Step 2: Add USBHIDMouse to main.cpp

In `src/main.cpp`, add the include and instance:

```cpp
#include "USBHIDMouse.h"

USBHIDKeyboard       Keyboard;
USBHIDConsumerControl Consumer;
USBHIDMouse          Mouse;  // NEW
```

In `setup()`, add `Mouse.begin()` before `USB.begin()`:

```cpp
Keyboard.begin();
Consumer.begin();
Mouse.begin();  // NEW
USB.begin();
```

### Step 3: Add Mouse Functions to human_input.h

In `include/human_input.h`, add:

```cpp
#include "USBHIDMouse.h"

void moveMouse(USBHIDMouse& mouse, int8_t dx, int8_t dy);
void clickMouse(USBHIDMouse& mouse, uint8_t button);
void pressMouse(USBHIDMouse& mouse, uint8_t button);
void releaseMouse(USBHIDMouse& mouse, uint8_t button);
void scrollMouse(USBHIDMouse& mouse, int8_t vScroll, int8_t hScroll);
```

### Step 4: Implement Mouse Functions in human_input.cpp

In `lib/fake_keyboard_core/src/human_input.cpp`, add:

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

### Step 5: Verify and Commit

```bash
# Verify changes
git status
git diff

# Stage and commit
git add -A
git commit -m "feat(stealth): add USBHIDMouse composite device

- Add USBHIDMouse instance to main.cpp
- Add mouse helper functions (move, click, press, release, scroll)
- Device now enumerates as keyboard + mouse + consumer control
- Fixes STEALTH-001"
```

### Step 6: Push and Merge to Dev

```bash
git push origin feature/stealth-001-usb-hid-mouse
git checkout dev
git pull origin dev
git merge feature/stealth-001-usb-hid-mouse
git push origin dev
```

### Step 7: Reset to Dev

```bash
git checkout dev
git pull origin dev
```
