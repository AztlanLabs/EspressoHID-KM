# FEATURE-003: Implement HID Command Processor

| Field | Value |
| --- | --- |
| **ID** | FEATURE-003 |
| **Priority** | 🔴 P0 (Critical) |
| **Epic** | Feature Expansion |
| **Estimated Effort** | 3 hours |
| **Dependencies** | FEATURE-002, STEALTH-001 |
| **Branch** | `feature/feature-003-hid-processor` |

---

## Description

The HID command processor reads commands from a queue and executes them on the HID devices.
It runs on core 1 (Arduino loop) and processes at most 1 command per 8 ms (125 Hz).

## References

> **Related docs for this ticket:** `docs/ARCHITECTURE.md`, `docs/FINDINGS.md`, `docs/STEALTH_HARDENING.md`, `docs/FEATURE_EXPANSION.md`, `docs/TICKETS.md` — read the relevant sections listed above before coding.

- `docs/FEATURE_EXPANSION.md` §5.2
- `docs/ARCHITECTURE.md` §4 (Architecture Overview)

## Files to Modify

| File | Action | Description |
| --- | --- | --- |
| `include/hid_command.h` | Modify | Add queue functions |
| `lib/fake_keyboard_core/src/hid_command.cpp` | Create | Implement queue and processor |
| `src/main.cpp` | Modify | Integrate processor into main loop |

## Objectives

1. Implement `HidCommand` queue (ring buffer, 64 entries)
2. Implement `hidCommandPush()` function
3. Implement `hidCommandPop()` function
4. Implement `hidCommandProcessor_loop()` function
5. Implement `executeHidCommand()` function
6. Integrate processor into main loop

## Acceptance Criteria

- [ ] `HidCommand` queue is implemented as a ring buffer (64 entries)
- [ ] `hidCommandPush()` adds commands to the queue
- [ ] `hidCommandPop()` removes commands from the queue
- [ ] `hidCommandProcessor_loop()` processes at most 1 command per 8 ms
- [ ] `executeHidCommand()` executes all command types
- [ ] Mouse move commands are executed on `Mouse`
- [ ] Mouse scroll commands are executed on `Mouse`
- [ ] Key press commands are executed on `Keyboard`
- [ ] Key release commands are executed on `Keyboard`
- [ ] Text commands are executed on `Keyboard`
- [ ] Emergency stop releases all keys and mouse buttons
- [ ] Queue overflow is handled gracefully (drop oldest)
- [ ] Processor is integrated into main loop
- [ ] No compilation errors or warnings

## Unit Tests

Create file `test/test_hid_command_processor.cpp`:

```cpp
#include <unity.h>
#include "hid_command.h"

void test_command_queue_push_pop() {
  HidCommand cmd1;
  cmd1.type = HID_CMD_MOUSE_MOVE;
  cmd1.mouse.dx = 10;
  cmd1.mouse.dy = 20;
  cmd1.mouse.buttons = 0;

  TEST_ASSERT_TRUE(hidCommandPush(cmd1));

  HidCommand cmd2;
  TEST_ASSERT_TRUE(hidCommandPop(cmd2));
  TEST_ASSERT_EQUAL(cmd1.type, cmd2.type);
  TEST_ASSERT_EQUAL(cmd1.mouse.dx, cmd2.mouse.dx);
}

void test_command_queue_overflow() {
  HidCommand cmd;
  cmd.type = HID_CMD_MOUSE_MOVE;
  cmd.mouse.dx = 0;
  cmd.mouse.dy = 0;
  cmd.mouse.buttons = 0;

  for (int i = 0; i < 64; i++) {
    TEST_ASSERT_TRUE(hidCommandPush(cmd));
  }
  TEST_ASSERT_FALSE(hidCommandPush(cmd));
}

void test_command_queue_empty() {
  HidCommand cmd;
  TEST_ASSERT_FALSE(hidCommandPop(cmd));
}

void setup() {
  UNITY_BEGIN();
  RUN_TEST(test_command_queue_push_pop);
  RUN_TEST(test_command_queue_overflow);
  RUN_TEST(test_command_queue_empty);
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
git checkout -b feature/feature-003-hid-processor
```

### Step 2: Add Queue Functions to hid_command.h

```cpp
bool hidCommandPush(const HidCommand& cmd);
bool hidCommandPop(HidCommand& cmd);
uint8_t hidCommandQueueSize();
void hidCommandProcessor_loop();
void executeHidCommand(const HidCommand& cmd);
```

### Step 3: Implement Queue and Processor

Create `lib/fake_keyboard_core/src/hid_command.cpp` with ring buffer and processor.

### Step 4: Integrate into Main Loop

In `src/main.cpp`, call `hidCommandProcessor_loop()` in `loop()`.

### Step 5: Verify and Commit

```bash
git status
git diff
git add -A
git commit -m "feat(hid): implement HID command processor

- Ring buffer queue (64 entries)
- hidCommandPush/Pop with overflow handling
- hidCommandProcessor_loop() at 125 Hz
- executeHidCommand() for all command types
- Integrated into main loop
- Fixes FEATURE-003"
```

### Step 6: Push and Merge to Dev

```bash
git push origin feature/feature-003-hid-processor
git checkout dev
git pull origin dev
git merge feature/feature-003-hid-processor
git push origin dev
```

### Step 7: Reset to Dev

```bash
git checkout dev
git pull origin dev
```
