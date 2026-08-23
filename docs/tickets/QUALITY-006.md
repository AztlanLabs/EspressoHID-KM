# QUALITY-006: Add WebSocket Protocol Unit Tests

| Field | Value |
| --- | --- |
| **ID** | QUALITY-006 |
| **Priority** | 🟡 P1 (High) |
| **Epic** | Quality & Testing |
| **Estimated Effort** | 2 hours |
| **Dependencies** | QUALITY-001, FEATURE-002 |
| **Branch** | `feature/quality-006-websocket-tests` |

---

## Description

WebSocket protocol tests verify that binary messages are parsed correctly and edge cases
are handled gracefully.

## References

> **Related docs for this ticket:** `docs/ARCHITECTURE.md`, `docs/FINDINGS.md`, `docs/STEALTH_HARDENING.md`, `docs/FEATURE_EXPANSION.md`, `docs/TICKETS.md` — read the relevant sections listed above before coding.

- `docs/FEATURE_EXPANSION.md` §3.2 (Binary Message Format)
- `docs/ARCHITECTURE.md` §11 (REST API Reference)

## Files to Modify

| File | Action | Description |
| --- | --- | --- |
| `test/test_websocket_protocol.cpp` | Create | WebSocket protocol unit tests |

## Objectives

1. Test mouse move message parsing
2. Test mouse scroll message parsing
3. Test key press message parsing
4. Test key release message parsing
5. Test text message parsing
6. Test emergency stop message parsing
7. Test invalid message handling
8. Test truncated message handling

## Acceptance Criteria

- [ ] Mouse move test verifies correct parsing
- [ ] Mouse scroll test verifies correct parsing
- [ ] Key press test verifies correct parsing
- [ ] Key release test verifies correct parsing
- [ ] Text test verifies correct parsing
- [ ] Emergency stop test verifies correct parsing
- [ ] Invalid message test verifies graceful rejection
- [ ] Truncated message test verifies graceful rejection
- [ ] All tests pass
- [ ] No compilation errors or warnings

## Unit Tests

Create file `test/test_websocket_protocol.cpp`:

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

void test_parse_mouse_move_negative_dx() {
  uint8_t data[] = {0x01, 246, 10, 0};  // dx=-10, dy=10
  HidCommand cmd;
  TEST_ASSERT_TRUE(parseBinaryCommand(data, 4, cmd));
  TEST_ASSERT_EQUAL(HID_CMD_MOUSE_MOVE, cmd.type);
  TEST_ASSERT_EQUAL(-10, cmd.mouse.dx);
  TEST_ASSERT_EQUAL(10, cmd.mouse.dy);
}

void test_parse_mouse_move_buttons() {
  uint8_t data[] = {0x01, 0, 0, 0x07};  // All buttons
  HidCommand cmd;
  TEST_ASSERT_TRUE(parseBinaryCommand(data, 4, cmd));
  TEST_ASSERT_EQUAL(0x07, cmd.mouse.buttons);
}

void test_parse_mouse_scroll() {
  uint8_t data[] = {0x02, 253, 0};  // vScroll=-3
  HidCommand cmd;
  TEST_ASSERT_TRUE(parseBinaryCommand(data, 3, cmd));
  TEST_ASSERT_EQUAL(HID_CMD_MOUSE_SCROLL, cmd.type);
  TEST_ASSERT_EQUAL(-3, cmd.scroll.v);
  TEST_ASSERT_EQUAL(0, cmd.scroll.h);
}

void test_parse_mouse_scroll_horizontal() {
  uint8_t data[] = {0x02, 0, 5};  // hScroll=5
  HidCommand cmd;
  TEST_ASSERT_TRUE(parseBinaryCommand(data, 3, cmd));
  TEST_ASSERT_EQUAL(0, cmd.scroll.v);
  TEST_ASSERT_EQUAL(5, cmd.scroll.h);
}

void test_parse_key_press() {
  uint8_t data[] = {0x10, 0x04, 0x02};  // A + Shift
  HidCommand cmd;
  TEST_ASSERT_TRUE(parseBinaryCommand(data, 3, cmd));
  TEST_ASSERT_EQUAL(HID_CMD_KEY_PRESS, cmd.type);
  TEST_ASSERT_EQUAL(0x04, cmd.key.keycode);
  TEST_ASSERT_EQUAL(0x02, cmd.key.modifiers);
}

void test_parse_key_press_ctrl_c() {
  uint8_t data[] = {0x10, 0x06, 0x01};  // C + Ctrl
  HidCommand cmd;
  TEST_ASSERT_TRUE(parseBinaryCommand(data, 3, cmd));
  TEST_ASSERT_EQUAL(HID_CMD_KEY_PRESS, cmd.type);
  TEST_ASSERT_EQUAL(0x06, cmd.key.keycode);
  TEST_ASSERT_EQUAL(0x01, cmd.key.modifiers);
}

void test_parse_key_release() {
  uint8_t data[] = {0x11, 0x04, 0x02};  // A + Shift
  HidCommand cmd;
  TEST_ASSERT_TRUE(parseBinaryCommand(data, 3, cmd));
  TEST_ASSERT_EQUAL(HID_CMD_KEY_RELEASE, cmd.type);
  TEST_ASSERT_EQUAL(0x04, cmd.key.keycode);
  TEST_ASSERT_EQUAL(0x02, cmd.key.modifiers);
}

void test_parse_text() {
  uint8_t data[] = {0x20, 5, 0, 0x01, 'H', 'e', 'l', 'l', 'o'};
  HidCommand cmd;
  TEST_ASSERT_TRUE(parseBinaryCommand(data, 9, cmd));
  TEST_ASSERT_EQUAL(HID_CMD_TEXT, cmd.type);
  TEST_ASSERT_EQUAL(5, cmd.text.length);
  TEST_ASSERT_EQUAL(0, cmd.text.mode);
  TEST_ASSERT_EQUAL(0x01, cmd.text.flags);
  TEST_ASSERT_EQUAL_STRING("Hello", cmd.text.data);
}

void test_parse_text_net_zero() {
  uint8_t data[] = {0x20, 3, 1, 0x03, 'A', 'B', 'C'};
  HidCommand cmd;
  TEST_ASSERT_TRUE(parseBinaryCommand(data, 7, cmd));
  TEST_ASSERT_EQUAL(HID_CMD_TEXT, cmd.type);
  TEST_ASSERT_EQUAL(3, cmd.text.length);
  TEST_ASSERT_EQUAL(1, cmd.text.mode);
  TEST_ASSERT_EQUAL(0x03, cmd.text.flags);
  TEST_ASSERT_EQUAL_STRING("ABC", cmd.text.data);
}

void test_parse_emergency_stop() {
  uint8_t data[] = {0xF0};
  HidCommand cmd;
  TEST_ASSERT_TRUE(parseBinaryCommand(data, 1, cmd));
  TEST_ASSERT_EQUAL(HID_CMD_EMERGENCY_STOP, cmd.type);
}

void test_parse_invalid_message_type() {
  uint8_t data[] = {0xFF};
  HidCommand cmd;
  TEST_ASSERT_FALSE(parseBinaryCommand(data, 1, cmd));
}

void test_parse_invalid_message_0x00() {
  uint8_t data[] = {0x00};
  HidCommand cmd;
  TEST_ASSERT_FALSE(parseBinaryCommand(data, 1, cmd));
}

void test_parse_truncated_mouse_move() {
  uint8_t data[] = {0x01, 10};  // Too short
  HidCommand cmd;
  TEST_ASSERT_FALSE(parseBinaryCommand(data, 2, cmd));
}

void test_parse_truncated_mouse_scroll() {
  uint8_t data[] = {0x02, 10};  // Too short
  HidCommand cmd;
  TEST_ASSERT_FALSE(parseBinaryCommand(data, 2, cmd));
}

void test_parse_truncated_key_press() {
  uint8_t data[] = {0x10, 0x04};  // Too short
  HidCommand cmd;
  TEST_ASSERT_FALSE(parseBinaryCommand(data, 2, cmd));
}

void test_parse_truncated_text() {
  uint8_t data[] = {0x20, 5, 0, 0x01, 'H'};  // Length=5 but only 1 byte
  HidCommand cmd;
  TEST_ASSERT_FALSE(parseBinaryCommand(data, 5, cmd));
}

void test_parse_empty_message() {
  uint8_t data[] = {};
  HidCommand cmd;
  TEST_ASSERT_FALSE(parseBinaryCommand(data, 0, cmd));
}

void setup() {
  UNITY_BEGIN();
  RUN_TEST(test_parse_mouse_move);
  RUN_TEST(test_parse_mouse_move_negative_dx);
  RUN_TEST(test_parse_mouse_move_buttons);
  RUN_TEST(test_parse_mouse_scroll);
  RUN_TEST(test_parse_mouse_scroll_horizontal);
  RUN_TEST(test_parse_key_press);
  RUN_TEST(test_parse_key_press_ctrl_c);
  RUN_TEST(test_parse_key_release);
  RUN_TEST(test_parse_text);
  RUN_TEST(test_parse_text_net_zero);
  RUN_TEST(test_parse_emergency_stop);
  RUN_TEST(test_parse_invalid_message_type);
  RUN_TEST(test_parse_invalid_message_0x00);
  RUN_TEST(test_parse_truncated_mouse_move);
  RUN_TEST(test_parse_truncated_mouse_scroll);
  RUN_TEST(test_parse_truncated_key_press);
  RUN_TEST(test_parse_truncated_text);
  RUN_TEST(test_parse_empty_message);
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
git checkout -b feature/quality-006-websocket-tests
```

### Step 2: Create Test File

Create `test/test_websocket_protocol.cpp` with the tests above.

### Step 3: Verify and Commit

```bash
git status
git diff
git add -A
git commit -m "feat(quality): add WebSocket protocol unit tests

- Test mouse move parsing (positive, negative, buttons)
- Test mouse scroll parsing (vertical, horizontal)
- Test key press/release parsing (with modifiers)
- Test text parsing (humanized, net-zero)
- Test emergency stop parsing
- Test invalid and truncated messages
- Test empty message
- Fixes QUALITY-006"
```

### Step 4: Push and Merge to Dev

```bash
git push origin feature/quality-006-websocket-tests
git checkout dev
git pull origin dev
git merge feature/quality-006-websocket-tests
git push origin dev
```

### Step 5: Reset to Dev

```bash
git checkout dev
git pull origin dev
```
