# FEATURE-002: Implement Binary WebSocket Protocol

| Field | Value |
| --- | --- |
| **ID** | FEATURE-002 |
| **Priority** | 🔴 P0 (Critical) |
| **Epic** | Feature Expansion |
| **Estimated Effort** | 3 hours |
| **Dependencies** | FEATURE-001 |
| **Branch** | `feature/feature-002-binary-protocol` |

---

## Description

Binary WebSocket messages are 10× smaller than JSON and faster to parse. This ticket
implements the binary protocol for mouse/keyboard commands.

## References

> **Related docs for this ticket:** `docs/ARCHITECTURE.md`, `docs/FINDINGS.md`, `docs/STEALTH_HARDENING.md`, `docs/FEATURE_EXPANSION.md`, `docs/TICKETS.md` — read the relevant sections listed above before coding.

- `docs/FEATURE_EXPANSION.md` §3.2
- `docs/ARCHITECTURE.md` §11 (REST API Reference)

## Files to Modify

| File | Action | Description |
| --- | --- | --- |
| `include/hid_command.h` | Create | Define `HidCommand` struct and message types |
| `src/web_portal.cpp` | Modify | Implement `parseBinaryCommand()` |

## Objectives

1. Define binary message types (0x01=mouse move, 0x02=mouse scroll, 0x10=key press, 0x11=key release, 0x20=text, 0xF0=emergency stop)
2. Implement `parseBinaryCommand()` function
3. Implement `HidCommand` struct
4. Handle all message types in WebSocket event handler

## Acceptance Criteria

- [ ] Binary message types are defined as constants
- [ ] `HidCommand` struct is defined with type-specific unions
- [ ] `parseBinaryCommand()` function parses all message types
- [ ] Mouse move messages (0x01) are parsed correctly
- [ ] Mouse scroll messages (0x02) are parsed correctly
- [ ] Key press messages (0x10) are parsed correctly
- [ ] Key release messages (0x11) are parsed correctly
- [ ] Text messages (0x20) are parsed correctly
- [ ] Emergency stop messages (0xF0) are parsed correctly
- [ ] Invalid messages are rejected gracefully
- [ ] Truncated messages are rejected gracefully
- [ ] No compilation errors or warnings

## Unit Tests

Create file `test/test_binary_protocol.cpp`:

```cpp
#include <unity.h>
#include "hid_command.h"

void test_parse_mouse_move() {
  uint8_t data[] = {0x01, 10, 251, 0};
  HidCommand cmd;
  TEST_ASSERT_TRUE(parseBinaryCommand(data, 4, cmd));
  TEST_ASSERT_EQUAL(HID_CMD_MOUSE_MOVE, cmd.type);
  TEST_ASSERT_EQUAL(10, cmd.mouse.dx);
  TEST_ASSERT_EQUAL(-5, cmd.mouse.dy);
  TEST_ASSERT_EQUAL(0, cmd.mouse.buttons);
}

void test_parse_mouse_scroll() {
  uint8_t data[] = {0x02, 253, 0};
  HidCommand cmd;
  TEST_ASSERT_TRUE(parseBinaryCommand(data, 3, cmd));
  TEST_ASSERT_EQUAL(HID_CMD_MOUSE_SCROLL, cmd.type);
  TEST_ASSERT_EQUAL(-3, cmd.scroll.v);
  TEST_ASSERT_EQUAL(0, cmd.scroll.h);
}

void test_parse_key_press() {
  uint8_t data[] = {0x10, 0x04, 0x02};
  HidCommand cmd;
  TEST_ASSERT_TRUE(parseBinaryCommand(data, 3, cmd));
  TEST_ASSERT_EQUAL(HID_CMD_KEY_PRESS, cmd.type);
  TEST_ASSERT_EQUAL(0x04, cmd.key.keycode);
  TEST_ASSERT_EQUAL(0x02, cmd.key.modifiers);
}

void test_parse_key_release() {
  uint8_t data[] = {0x11, 0x04, 0x02};
  HidCommand cmd;
  TEST_ASSERT_TRUE(parseBinaryCommand(data, 3, cmd));
  TEST_ASSERT_EQUAL(HID_CMD_KEY_RELEASE, cmd.type);
}

void test_parse_text() {
  uint8_t data[] = {0x20, 5, 0, 0x01, 'H', 'e', 'l', 'l', 'o'};
  HidCommand cmd;
  TEST_ASSERT_TRUE(parseBinaryCommand(data, 9, cmd));
  TEST_ASSERT_EQUAL(HID_CMD_TEXT, cmd.type);
  TEST_ASSERT_EQUAL(5, cmd.text.length);
  TEST_ASSERT_EQUAL(0x01, cmd.text.flags);
}

void test_parse_emergency_stop() {
  uint8_t data[] = {0xF0};
  HidCommand cmd;
  TEST_ASSERT_TRUE(parseBinaryCommand(data, 1, cmd));
  TEST_ASSERT_EQUAL(HID_CMD_EMERGENCY_STOP, cmd.type);
}

void test_parse_invalid_message() {
  uint8_t data[] = {0xFF};
  HidCommand cmd;
  TEST_ASSERT_FALSE(parseBinaryCommand(data, 1, cmd));
}

void test_parse_truncated_message() {
  uint8_t data[] = {0x01, 10};
  HidCommand cmd;
  TEST_ASSERT_FALSE(parseBinaryCommand(data, 2, cmd));
}

void setup() {
  UNITY_BEGIN();
  RUN_TEST(test_parse_mouse_move);
  RUN_TEST(test_parse_mouse_scroll);
  RUN_TEST(test_parse_key_press);
  RUN_TEST(test_parse_key_release);
  RUN_TEST(test_parse_text);
  RUN_TEST(test_parse_emergency_stop);
  RUN_TEST(test_parse_invalid_message);
  RUN_TEST(test_parse_truncated_message);
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
git checkout -b feature/feature-002-binary-protocol
```

### Step 2: Create hid_command.h

```cpp
#pragma once
#include <Arduino.h>

enum HidCmdType : uint8_t {
  HID_CMD_MOUSE_MOVE      = 0x01,
  HID_CMD_MOUSE_SCROLL    = 0x02,
  HID_CMD_KEY_PRESS       = 0x10,
  HID_CMD_KEY_RELEASE     = 0x11,
  HID_CMD_TEXT            = 0x20,
  HID_CMD_EMERGENCY_STOP  = 0xF0,
};

enum HidModifier : uint8_t {
  MOD_CTRL  = 0x01,
  MOD_SHIFT = 0x02,
  MOD_ALT   = 0x04,
  MOD_GUI   = 0x08,
};

struct HidCommand {
  HidCmdType type;
  union {
    struct { int8_t dx; int8_t dy; uint8_t buttons; } mouse;
    struct { int8_t v; int8_t h; } scroll;
    struct { uint8_t keycode; uint8_t modifiers; } key;
    struct { uint8_t length; uint8_t mode; uint8_t flags; char data[129]; } text;
  };
};

bool parseBinaryCommand(const uint8_t* data, size_t length, HidCommand& cmd);
```

### Step 3: Implement parseBinaryCommand()

In `src/web_portal.cpp`, implement the parser.

### Step 4: Verify and Commit

```bash
git status
git diff
git add -A
git commit -m "feat(network): implement binary WebSocket protocol

- Define HidCommand struct with type-specific unions
- Implement parseBinaryCommand() for all message types
- Mouse move, scroll, key press/release, text, emergency stop
- Invalid and truncated messages rejected gracefully
- Fixes FEATURE-002"
```

### Step 5: Push and Merge to Dev

```bash
git push origin feature/feature-002-binary-protocol
git checkout dev
git pull origin dev
git merge feature/feature-002-binary-protocol
git push origin dev
```

### Step 6: Reset to Dev

```bash
git checkout dev
git pull origin dev
```
