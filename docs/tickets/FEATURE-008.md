# FEATURE-008: Add OTA Update via WebSocket

| Field | Value |
| --- | --- |
| **ID** | FEATURE-008 |
| **Priority** | 🟢 P2 (Medium) |
| **Epic** | Feature Expansion |
| **Estimated Effort** | 3 hours |
| **Dependencies** | FEATURE-001 |
| **Branch** | `feature/feature-008-ota-websocket` |

---

## Description

Currently, OTA updates are only available via HTTP. This ticket adds OTA update support via
WebSocket for faster and more reliable firmware updates.

## References

> **Related docs for this ticket:** `docs/ARCHITECTURE.md`, `docs/FINDINGS.md`, `docs/STEALTH_HARDENING.md`, `docs/FEATURE_EXPANSION.md`, `docs/TICKETS.md` — read the relevant sections listed above before coding.

- `docs/FEATURE_EXPANSION.md` §11
- `docs/ARCHITECTURE.md` §11 (REST API Reference)

## Files to Modify

| File | Action | Description |
| --- | --- | --- |
| `include/hid_command.h` | Modify | Add OTA message types |
| `src/web_portal.cpp` | Modify | Implement OTA WebSocket handler |

## Objectives

1. Implement OTA message type (0x30)
2. Implement OTA start message
3. Implement OTA data message
4. Implement OTA end message
5. Implement OTA progress reporting
6. Integrate with ESP-IDF Update library

## Acceptance Criteria

- [ ] OTA message type (0x30) is defined
- [ ] OTA start message is handled
- [ ] OTA data message is handled
- [ ] OTA end message is handled
- [ ] OTA progress is reported to client
- [ ] Firmware is written to flash correctly
- [ ] Device reboots after successful update
- [ ] Failed update is handled gracefully
- [ ] No compilation errors or warnings

## Unit Tests

Create file `test/test_ota_websocket.cpp`:

```cpp
#include <unity.h>
#include "hid_command.h"

void test_ota_start_message() {
  uint8_t data[] = {0x30, 0x01, 0x00, 0x00, 0x10, 0x00};
  HidCommand cmd;
  TEST_ASSERT_TRUE(parseBinaryCommand(data, 6, cmd));
  TEST_ASSERT_EQUAL(HID_CMD_OTA_START, cmd.type);
}

void test_ota_end_message() {
  uint8_t data[] = {0x30, 0x03};
  HidCommand cmd;
  TEST_ASSERT_TRUE(parseBinaryCommand(data, 2, cmd));
  TEST_ASSERT_EQUAL(HID_CMD_OTA_END, cmd.type);
}

void setup() {
  UNITY_BEGIN();
  RUN_TEST(test_ota_start_message);
  RUN_TEST(test_ota_end_message);
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
git checkout -b feature/feature-008-ota-websocket
```

### Step 2: Add OTA Message Types

In `include/hid_command.h`:

```cpp
HID_CMD_OTA_START = 0x30,
HID_CMD_OTA_DATA  = 0x31,
HID_CMD_OTA_END   = 0x32,
```

### Step 3: Implement OTA Handler

In `src/web_portal.cpp`, implement OTA WebSocket handler using `Update` library.

### Step 4: Verify and Commit

```bash
git status
git diff
git add -A
git commit -m "feat(network): add OTA update via WebSocket

- OTA message types (0x30, 0x31, 0x32)
- OTA start/data/end handling
- Progress reporting to client
- Device reboot after successful update
- Fixes FEATURE-008"
```

### Step 5: Push and Merge to Dev

```bash
git push origin feature/feature-008-ota-websocket
git checkout dev
git pull origin dev
git merge feature/feature-008-ota-websocket
git push origin dev
```

### Step 6: Reset to Dev

```bash
git checkout dev
git pull origin dev
```
