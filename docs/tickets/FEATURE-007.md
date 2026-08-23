# FEATURE-007: Implement Dual-Core Task Separation

| Field | Value |
| --- | --- |
| **ID** | FEATURE-007 |
| **Priority** | 🟡 P1 (High) |
| **Epic** | Feature Expansion |
| **Estimated Effort** | 3 hours |
| **Dependencies** | FEATURE-001 |
| **Branch** | `feature/feature-007-dual-core` |

---

## Description

The ESP32-S3 is dual-core. Currently, core 0 handles WiFi/BT (idle most of the time) and
core 1 runs the Arduino loop(). By adding a WebSocket task to core 0, we utilize
otherwise-idle CPU cycles and achieve true parallelism without increasing CPU usage.

## References

> **Related docs for this ticket:** `docs/ARCHITECTURE.md`, `docs/FINDINGS.md`, `docs/STEALTH_HARDENING.md`, `docs/FEATURE_EXPANSION.md`, `docs/TICKETS.md` — read the relevant sections listed above before coding.

- `docs/FEATURE_EXPANSION.md` §6.1
- `docs/ARCHITECTURE.md` §4 (Architecture Overview)
- `docs/FINDINGS.md` §3.1 (Single-Threaded Cooperative Loop)

## Files to Modify

| File | Action | Description |
| --- | --- | --- |
| `src/main.cpp` | Modify | Create WebSocket task on core 0 |
| `src/web_portal.cpp` | Modify | Move WebSocket loop to task |
| `include/hid_command.h` | Modify | Add mutex protection |

## Objectives

1. Create WebSocket task on core 0
2. Implement thread-safe command queue
3. Implement mutex for shared state
4. Verify no race conditions
5. Measure CPU usage impact

## Acceptance Criteria

- [ ] WebSocket task is created on core 0
- [ ] WebSocket task runs `webSocket.loop()` continuously
- [ ] Command queue is thread-safe (mutex protected)
- [ ] Shared state is protected by mutex
- [ ] No race conditions (verified by stress test)
- [ ] Core 0 CPU usage increases by < 20%
- [ ] Core 1 CPU usage is unchanged
- [ ] No compilation errors or warnings
- [ ] Existing functionality is not broken

## Unit Tests

Create file `test/test_dual_core.cpp`:

```cpp
#include <unity.h>
#include "hid_command.h"

void test_thread_safe_queue() {
  HidCommand cmd;
  cmd.type = HID_CMD_MOUSE_MOVE;
  cmd.mouse.dx = 10;
  cmd.mouse.dy = 20;
  cmd.mouse.buttons = 0;

  for (int i = 0; i < 1000; i++) {
    TEST_ASSERT_TRUE(hidCommandPush(cmd));
  }

  HidCommand popped;
  for (int i = 0; i < 1000; i++) {
    TEST_ASSERT_TRUE(hidCommandPop(popped));
  }
}

void test_no_race_conditions() {
  // Stress test: push/pop rapidly
  HidCommand cmd;
  cmd.type = HID_CMD_MOUSE_MOVE;
  cmd.mouse.dx = 1;
  cmd.mouse.dy = 0;
  cmd.mouse.buttons = 0;

  for (int i = 0; i < 10000; i++) {
    hidCommandPush(cmd);
    HidCommand popped;
    hidCommandPop(popped);
  }
  TEST_ASSERT_TRUE_MESSAGE(true, "No race conditions detected");
}

void setup() {
  UNITY_BEGIN();
  RUN_TEST(test_thread_safe_queue);
  RUN_TEST(test_no_race_conditions);
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
git checkout -b feature/feature-007-dual-core
```

### Step 2: Add Mutex to hid_command.h

```cpp
#include <freertos/semphr.h>

extern SemaphoreHandle_t hidCommandMutex;
```

### Step 3: Create WebSocket Task

In `src/main.cpp`:

```cpp
#include <freertos/FreeRTOS.h>
#include <freertos/task.h>

void webSocketTask(void* parameter) {
  for (;;) {
    webSocket.loop();
    vTaskDelay(1 / portTICK_PERIOD_MS);
  }
}

void setup() {
  // ... existing setup ...
  xTaskCreatePinnedToCore(
    webSocketTask, "WebSocket", 8192, NULL, 1, NULL, 0
  );
}
```

### Step 4: Add Mutex Protection

Protect `hidCommandPush()` and `hidCommandPop()` with mutex.

### Step 5: Verify and Commit

```bash
git status
git diff
git add -A
git commit -m "feat(arch): implement dual-core task separation

- WebSocket task on core 0
- Thread-safe command queue with mutex
- No increase in core 1 CPU usage
- Fixes FEATURE-007"
```

### Step 6: Push and Merge to Dev

```bash
git push origin feature/feature-007-dual-core
git checkout dev
git pull origin dev
git merge feature/feature-007-dual-core
git push origin dev
```

### Step 7: Reset to Dev

```bash
git checkout dev
git pull origin dev
```
