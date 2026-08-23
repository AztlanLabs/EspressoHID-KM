# STEALTH-007: Improve USB Descriptor Spoofing

| Field | Value |
| --- | --- |
| **ID** | STEALTH-007 |
| **Priority** | 🟡 P1 (High) |
| **Epic** | Stealth Hardening |
| **Estimated Effort** | 4 hours |
| **Dependencies** | None |
| **Branch** | `feature/stealth-007-descriptor-spoofing` |

---

## Description

The Arduino ESP32-S3 USB stack generates a generic HID descriptor that is identifiable as an
Arduino/ESP32 device. USB descriptor fingerprinting tools can compare the descriptor against
a database of known devices. This ticket overrides the default descriptor with one extracted
from a real keyboard.

## Detection Vector

**HID descriptor fingerprinting** — "the HID report descriptor tells the OS what the device
is. Tools can parse and compare descriptors to known devices."

## References

> **Related docs for this ticket:** `docs/ARCHITECTURE.md`, `docs/FINDINGS.md`, `docs/STEALTH_HARDENING.md`, `docs/FEATURE_EXPANSION.md`, `docs/TICKETS.md` — read the relevant sections listed above before coding.

- `docs/STEALTH_HARDENING.md` §V7, §6.7
- `docs/ARCHITECTURE.md` §3 (Hardware & Wiring)

## Files to Modify

| File | Action | Description |
| --- | --- | --- |
| `include/config.h` | Modify | Add custom descriptor arrays |
| `src/main.cpp` | Modify | Override default descriptors |

## Objectives

1. Extract HID report descriptor from a real keyboard (e.g., Dell KB216)
2. Extract HID report descriptor from a real mouse (e.g., Logitech M100)
3. Create custom descriptor arrays in PROGMEM
4. Override Arduino default descriptors
5. Verify descriptors match real devices exactly

## Acceptance Criteria

- [ ] Custom keyboard HID report descriptor is extracted from a real keyboard
- [ ] Custom mouse HID report descriptor is extracted from a real mouse
- [ ] Descriptors are stored in PROGMEM arrays
- [ ] Arduino default descriptors are overridden
- [ ] Device enumerates with the custom descriptors
- [ ] Keyboard functionality works with custom descriptor
- [ ] Mouse functionality works with custom descriptor
- [ ] No compilation errors or warnings

## Unit Tests

Create file `test/test_usb_descriptor.cpp`:

```cpp
#include <unity.h>
#include "config.h"

void test_keyboard_descriptor_size() {
  TEST_ASSERT_GREATER_THAN(0, sizeof(customKeyboardReportDescriptor));
}

void test_keyboard_descriptor_usage_page() {
  TEST_ASSERT_EQUAL(0x05, customKeyboardReportDescriptor[0]);
  TEST_ASSERT_EQUAL(0x01, customKeyboardReportDescriptor[1]);
}

void test_keyboard_descriptor_usage() {
  TEST_ASSERT_EQUAL(0x09, customKeyboardReportDescriptor[2]);
  TEST_ASSERT_EQUAL(0x06, customKeyboardReportDescriptor[3]);
}

void setup() {
  UNITY_BEGIN();
  RUN_TEST(test_keyboard_descriptor_size);
  RUN_TEST(test_keyboard_descriptor_usage_page);
  RUN_TEST(test_keyboard_descriptor_usage);
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
git checkout -b feature/stealth-007-descriptor-spoofing
```

### Step 2: Extract Real Descriptors

Use `usbhid-dump` on Linux to extract descriptors from a real keyboard and mouse.

### Step 3: Add Descriptors to config.h

```cpp
static const uint8_t customKeyboardReportDescriptor[] PROGMEM = {
  // Paste extracted descriptor here
};
```

### Step 4: Override Descriptors in main.cpp

Override the Arduino default descriptors before `USB.begin()`.

### Step 5: Verify and Commit

```bash
git status
git diff
git add -A
git commit -m "feat(stealth): improve USB descriptor spoofing

- Extract real keyboard/mouse HID descriptors
- Override Arduino default descriptors
- Device now appears as real hardware
- Fixes STEALTH-007"
```

### Step 6: Push and Merge to Dev

```bash
git push origin feature/stealth-007-descriptor-spoofing
git checkout dev
git pull origin dev
git merge feature/stealth-007-descriptor-spoofing
git push origin dev
```

### Step 7: Reset to Dev

```bash
git checkout dev
git pull origin dev
```
