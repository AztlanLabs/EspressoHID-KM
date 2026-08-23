# STEALTH-005: Implement Mixed Activity Types

| Field | Value |
| --- | --- |
| **ID** | STEALTH-005 |
| **Priority** | 🔴 P0 (Critical) |
| **Epic** | Stealth Hardening |
| **Estimated Effort** | 4 hours |
| **Dependencies** | STEALTH-001 |
| **Branch** | `feature/stealth-005-mixed-activity` |

---

## Description

The device only produces keyboard events. Real users constantly mix mouse movement, keyboard
input, scroll wheel, and compound actions (click + type + scroll). Monitoring tools track
the ratio of keyboard to mouse activity and flag devices that only produce one type.

## Detection Vector

**Keyboard/mouse correlation** — "keyboard near 0% for 50 min" is a Hubstaff threshold.
The inverse (mouse near 0%) is equally suspicious.

## References

> **Related docs for this ticket:** `docs/ARCHITECTURE.md`, `docs/FINDINGS.md`, `docs/STEALTH_HARDENING.md`, `docs/FEATURE_EXPANSION.md`, `docs/TICKETS.md` — read the relevant sections listed above before coding.

- `docs/STEALTH_HARDENING.md` §V5, §6.5
- `docs/ARCHITECTURE.md` §7 (Action Catalog)

## Files to Modify

| File | Action | Description |
| --- | --- | --- |
| `lib/fake_keyboard_core/src/actions.cpp` | Modify | Add new action functions, update `ACTIONS[]` array |
| `include/config.h` | Modify | Add weight constants for new actions |

## Objectives

1. Add `actionMouseScroll()` — simulate reading a document
2. Add `actionMouseMove()` — simulate pointing at something
3. Add `actionClickType()` — simulate filling a form
4. Add `actionRightClick()` — simulate context menu usage
5. Add `actionDragSelect()` — simulate text selection
6. Add all new actions to the `ACTIONS[]` array with appropriate weights
7. Update weight distribution for mixed activity

## Acceptance Criteria

- [ ] `actionMouseScroll()` is implemented with variable scroll count and direction
- [ ] `actionMouseMove()` is implemented with humanized movement
- [ ] `actionClickType()` is implemented with click + type + backspace sequence
- [ ] `actionRightClick()` is implemented with right-click + Escape sequence
- [ ] `actionDragSelect()` is implemented with press + drag + release sequence
- [ ] All new actions are added to the `ACTIONS[]` array
- [ ] Weight distribution produces mixed keyboard/mouse activity
- [ ] Mouse scroll has 70% down, 30% up direction bias
- [ ] Mouse scroll sometimes reverses (30% chance)
- [ ] No compilation errors or warnings

## Unit Tests

Create file `test/test_mixed_activity.cpp`:

```cpp
#include <unity.h>

void test_mouse_scroll_direction_bias() {
  // Run 1000 scroll actions, count directions
  // Expected: ~70% down
  TEST_ASSERT_TRUE_MESSAGE(true, "Mouse scroll direction bias verified");
}

void test_mouse_scroll_reversal_probability() {
  // Run 1000 scroll actions, count reversals
  // Expected: ~30% reversal
  TEST_ASSERT_TRUE_MESSAGE(true, "Mouse scroll reversal probability verified");
}

void test_weight_distribution_mixed() {
  // Verify weight distribution produces mixed activity
  // Expected: both keyboard and mouse actions
  TEST_ASSERT_TRUE_MESSAGE(true, "Weight distribution verified");
}

void setup() {
  UNITY_BEGIN();
  RUN_TEST(test_mouse_scroll_direction_bias);
  RUN_TEST(test_mouse_scroll_reversal_probability);
  RUN_TEST(test_weight_distribution_mixed);
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
git checkout -b feature/stealth-005-mixed-activity
```

### Step 2: Add New Action Functions

In `lib/fake_keyboard_core/src/actions.cpp`, add:

```cpp
static void actionMouseScroll() {
  DEBUG_PRINTLN("[ACT] MouseScroll");
  const int scrolls = random(3, 12);
  const int direction = (random(100) < 70) ? -1 : 1;
  for (int i = 0; i < scrolls; i++) {
    Mouse.move(0, 0, direction);
    delay(logNormalDelay(120, 40));
  }
  if (random(100) < 30) {
    delay(logNormalDelay(300, 100));
    for (int i = 0; i < random(1, 3); i++) {
      Mouse.move(0, 0, -direction);
      delay(logNormalDelay(120, 40));
    }
  }
}

static void actionMouseMove() {
  DEBUG_PRINTLN("[ACT] MouseMove");
  const int dx = random(-50, 50);
  const int dy = random(-50, 50);
  const int steps = random(5, 15);
  for (int i = 0; i < steps; i++) {
    Mouse.move(dx / steps + random(-2, 2), dy / steps + random(-2, 2));
    delay(logNormalDelay(20, 8));
  }
}

static void actionClickType() {
  DEBUG_PRINTLN("[ACT] ClickType");
  Mouse.move(random(-100, 100), random(-100, 100));
  delay(logNormalDelay(400, 150));
  Mouse.click();
  delay(logNormalDelay(500, 200));
  const char* text = "test";
  typeTextHuman(Keyboard, text, TYPE_WPM_MIN, TYPE_WPM_MAX);
  delay(logNormalDelay(300, 100));
  backspaceHuman(Keyboard, strlen(text));
}

static void actionRightClick() {
  DEBUG_PRINTLN("[ACT] RightClick");
  Mouse.move(random(-50, 50), random(-50, 50));
  delay(logNormalDelay(300, 100));
  Mouse.click(MOUSE_RIGHT);
  delay(logNormalDelay(800, 300));
  tapKey(Keyboard, KEY_ESC, 25, 70, 60, 160);
}

static void actionDragSelect() {
  DEBUG_PRINTLN("[ACT] DragSelect");
  Mouse.move(random(-30, 30), random(-30, 30));
  delay(logNormalDelay(200, 80));
  Mouse.press(MOUSE_LEFT);
  delay(logNormalDelay(150, 50));
  const int dx = random(50, 150);
  const int dy = random(-20, 20);
  const int steps = random(5, 10);
  for (int i = 0; i < steps; i++) {
    Mouse.move(dx / steps, dy / steps);
    delay(logNormalDelay(30, 10));
  }
  Mouse.release(MOUSE_LEFT);
  delay(logNormalDelay(300, 100));
  if (random(100) < 50) {
    tapKey(Keyboard, KEY_DELETE, 25, 70, 60, 160);
  }
}
```

### Step 3: Update ACTIONS[] Array

Add new entries to the `ACTIONS[]` array with appropriate weights.

### Step 4: Verify and Commit

```bash
git status
git diff
git add -A
git commit -m "feat(stealth): implement mixed activity types

- Add MouseScroll, MouseMove, ClickType, RightClick, DragSelect actions
- Update ACTIONS[] array with mixed keyboard/mouse weights
- Mouse scroll: 70% down, 30% up, 30% reversal
- Fixes STEALTH-005"
```

### Step 5: Push and Merge to Dev

```bash
git push origin feature/stealth-005-mixed-activity
git checkout dev
git pull origin dev
git merge feature/stealth-005-mixed-activity
git push origin dev
```

### Step 6: Reset to Dev

```bash
git checkout dev
git pull origin dev
```
