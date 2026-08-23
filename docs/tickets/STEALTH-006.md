# STEALTH-006: Implement Humanized Mouse Movement

| Field | Value |
| --- | --- |
| **ID** | STEALTH-006 |
| **Priority** | 🔴 P0 (Critical) |
| **Epic** | Stealth Hardening |
| **Estimated Effort** | 4 hours |
| **Dependencies** | STEALTH-001 |
| **Branch** | `feature/stealth-006-humanized-mouse` |

---

## Description

Current mouse movement (when added by STEALTH-001) uses simple `Mouse.move(dx, dy)` calls
which produce instant, straight-line movement. Real human mouse movement is curved, variable
speed, and includes micro-tremors. The WindMouse algorithm generates curved, non-linear paths
with variable speed that mimic natural human behavior.

## Detection Vector

**Movement pattern analysis** — "most USB jigglers produce mechanically regular movement:
the same short displacement at the same interval, forever."

## References

> **Related docs for this ticket:** `docs/ARCHITECTURE.md`, `docs/FINDINGS.md`, `docs/STEALTH_HARDENING.md`, `docs/FEATURE_EXPANSION.md`, `docs/TICKETS.md` — read the relevant sections listed above before coding.

- `docs/STEALTH_HARDENING.md` §V2, §6.2
- `docs/FEATURE_EXPANSION.md` §6.2 (Humanized Movement)

## Files to Modify

| File | Action | Description |
| --- | --- | --- |
| `include/human_input.h` | Modify | Add WindMouse function declarations |
| `lib/fake_keyboard_core/src/human_input.cpp` | Modify | Implement WindMouse algorithm |
| `lib/fake_keyboard_core/src/actions.cpp` | Modify | Integrate WindMouse into mouse actions |

## Objectives

1. Implement `WindMouseState` struct
2. Implement `windMouseInit()` function
3. Implement `windMouseStep()` function
4. Implement `windMouseMove()` function that moves mouse to target using WindMouse algorithm
5. Integrate WindMouse into mouse movement actions

## Acceptance Criteria

- [ ] `WindMouseState` struct is defined with position, velocity, gravity, wind, damping
- [ ] `windMouseInit()` function initializes state with target position
- [ ] `windMouseStep()` function updates position using WindMouse algorithm
- [ ] `windMouseMove()` function moves mouse to target using multiple steps
- [ ] Movement paths are curved (not straight lines)
- [ ] Movement speed varies (accelerates at start, decelerates at target)
- [ ] Movement duration increases with distance (Fitts' law)
- [ ] WindMouse is integrated into `actionMouseMove()`
- [ ] No compilation errors or warnings

## Unit Tests

Create file `test/test_wind_mouse.cpp`:

```cpp
#include <unity.h>
#include "human_input.h"

void test_wind_mouse_initialization() {
  WindMouseState state;
  windMouseInit(state, 100.0f, 200.0f, 0.5f, 2.0f, 0.8f);
  TEST_ASSERT_FLOAT_WITHIN(1.0f, 100.0f, state.targetX);
  TEST_ASSERT_FLOAT_WITHIN(1.0f, 200.0f, state.targetY);
}

void test_wind_mouse_convergence() {
  WindMouseState state;
  windMouseInit(state, 100.0f, 200.0f, 0.5f, 2.0f, 0.8f);
  for (int i = 0; i < 1000; i++) {
    windMouseStep(state);
    if (fabs(state.x - state.targetX) < 1.0f &&
        fabs(state.y - state.targetY) < 1.0f) break;
  }
  TEST_ASSERT_FLOAT_WITHIN(2.0f, 100.0f, state.x);
  TEST_ASSERT_FLOAT_WITHIN(2.0f, 200.0f, state.y);
}

void test_wind_mouse_curved_path() {
  WindMouseState state;
  windMouseInit(state, 100.0f, 0.0f, 0.5f, 2.0f, 0.8f);
  float maxDeviation = 0;
  for (int i = 0; i < 100; i++) {
    windMouseStep(state);
    float t = (float)i / 100.0f;
    float expectedX = t * 100.0f;
    float deviation = fabs(state.x - expectedX);
    if (deviation > maxDeviation) maxDeviation = deviation;
  }
  TEST_ASSERT_GREATER_THAN(5.0f, maxDeviation);
}

void setup() {
  UNITY_BEGIN();
  RUN_TEST(test_wind_mouse_initialization);
  RUN_TEST(test_wind_mouse_convergence);
  RUN_TEST(test_wind_mouse_curved_path);
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
git checkout -b feature/stealth-006-humanized-mouse
```

### Step 2: Add WindMouse to human_input.h

```cpp
struct WindMouseState {
  float x, y;
  float targetX, targetY;
  float velocityX, velocityY;
  float gravity, wind, damping;
};

void windMouseInit(WindMouseState& state, float targetX, float targetY,
                   float gravity, float wind, float damping);
void windMouseStep(WindMouseState& state);
void windMouseMove(USBHIDMouse& mouse, int targetDx, int targetDy,
                   float gravity, float wind, float damping);
```

### Step 3: Implement WindMouse in human_input.cpp

```cpp
void windMouseInit(WindMouseState& state, float targetX, float targetY,
                   float gravity, float wind, float damping) {
  state.x = 0; state.y = 0;
  state.targetX = targetX; state.targetY = targetY;
  state.velocityX = 0; state.velocityY = 0;
  state.gravity = gravity; state.wind = wind; state.damping = damping;
}

void windMouseStep(WindMouseState& state) {
  float dx = state.targetX - state.x;
  float dy = state.targetY - state.y;
  float dist = sqrt(dx * dx + dy * dy);
  if (dist < 1.0f) return;
  float gravX = dx / dist * state.gravity;
  float gravY = dy / dist * state.gravity;
  float windX = gaussianRandom(0, state.wind);
  float windY = gaussianRandom(0, state.wind);
  state.velocityX = (state.velocityX + gravX + windX) * state.damping;
  state.velocityY = (state.velocityY + gravY + windY) * state.damping;
  state.x += state.velocityX;
  state.y += state.velocityY;
}

void windMouseMove(USBHIDMouse& mouse, int targetDx, int targetDy,
                   float gravity, float wind, float damping) {
  WindMouseState state;
  windMouseInit(state, (float)targetDx, (float)targetDy, gravity, wind, damping);
  int steps = max(abs(targetDx), abs(targetDy)) * 2;
  for (int i = 0; i < steps; i++) {
    float prevX = state.x, prevY = state.y;
    windMouseStep(state);
    int8_t moveX = (int8_t)(state.x - prevX);
    int8_t moveY = (int8_t)(state.y - prevY);
    if (moveX != 0 || moveY != 0) mouse.move(moveX, moveY);
    delay(logNormalDelay(8, 3));
    if (fabs(state.x - state.targetX) < 1.0f &&
        fabs(state.y - state.targetY) < 1.0f) break;
  }
}
```

### Step 4: Integrate into actionMouseMove()

Update `actionMouseMove()` to use `windMouseMove()`.

### Step 5: Verify and Commit

```bash
git status
git diff
git add -A
git commit -m "feat(stealth): implement humanized mouse movement (WindMouse)

- Add WindMouseState struct and algorithm
- Curved paths, variable speed, micro-tremors
- Fitts' law: duration increases with distance
- Integrated into actionMouseMove()
- Fixes STEALTH-006"
```

### Step 6: Push and Merge to Dev

```bash
git push origin feature/stealth-006-humanized-mouse
git checkout dev
git pull origin dev
git merge feature/stealth-006-humanized-mouse
git push origin dev
```

### Step 7: Reset to Dev

```bash
git checkout dev
git pull origin dev
```
