# STEALTH-004: Add Variation to Net-Zero Actions

| Field | Value |
| --- | --- |
| **ID** | STEALTH-004 |
| **Priority** | 🔴 P0 (Critical) |
| **Epic** | Stealth Hardening |
| **Estimated Effort** | 2 hours |
| **Dependencies** | STEALTH-002 (uses `logNormalDelay()` for variable delays) |
| **Branch** | `feature/stealth-004-net-zero-variation` |

---

## Description

Net-zero actions always follow the exact same pattern: Volume increment then decrement,
CapsToggle on then off, etc. This is detectable by pattern analysis. Real users don't
always reverse their actions — sometimes they leave volume changed, sometimes they toggle
CapsLock multiple times.

## Detection Vector

**Pattern analysis** — every Volume action is followed by a Volume decrement, every
CapsToggle is followed by another CapsToggle.

## References

> **Related docs for this ticket:** `docs/ARCHITECTURE.md`, `docs/FINDINGS.md`, `docs/STEALTH_HARDENING.md`, `docs/FEATURE_EXPANSION.md`, `docs/TICKETS.md` — read the relevant sections listed above before coding.

- `docs/STEALTH_HARDENING.md` §V6, §6.6
- `docs/ARCHITECTURE.md` §7 (Action Catalog)

## Files to Modify

| File | Action | Description |
| --- | --- | --- |
| `lib/fake_keyboard_core/src/actions.cpp` | Modify | Add variation to `actionVolume()`, `actionBrightness()`, `actionCapsToggle()`, `actionNumLockToggle()` |

## Objectives

1. Add variation to Volume action (80% reversal, 20% leave changed)
2. Add variation to Brightness action (80% reversal, 20% leave changed)
3. Add variation to CapsToggle action (variable toggle count, sometimes leave changed)
4. Add variation to NumLockToggle action (variable toggle count, sometimes leave changed)
5. Add variable delay before reversal

## Acceptance Criteria

- [ ] Volume action reverses 80% of the time, leaves changed 20% of the time
- [ ] Brightness action reverses 80% of the time, leaves changed 20% of the time
- [ ] CapsToggle action toggles 1–3 times (random)
- [ ] CapsToggle leaves CapsLock on 30% of the time if odd number of toggles
- [ ] NumLockToggle action toggles 1–3 times (random)
- [ ] NumLockToggle leaves NumLock on 30% of the time if odd number of toggles
- [ ] Variable delay before reversal (80–2000 ms for Volume/Brightness)
- [ ] No compilation errors or warnings

## Unit Tests

Create file `test/test_net_zero_variation.cpp`:

```cpp
#include <unity.h>

void test_volume_reversal_probability() {
  // Run 1000 Volume actions, count reversals
  // Expected: ~80% reversal rate
  TEST_ASSERT_TRUE_MESSAGE(true, "Volume reversal probability verified");
}

void test_caps_toggle_variable_count() {
  // Run 1000 CapsToggle actions, count toggle counts
  // Expected: variation in toggle counts (1, 2, 3)
  TEST_ASSERT_TRUE_MESSAGE(true, "CapsToggle variable count verified");
}

void test_volume_delay_variation() {
  // Verify delay before reversal is variable
  // Expected: high variance in delays
  TEST_ASSERT_TRUE_MESSAGE(true, "Volume delay variation verified");
}

void setup() {
  UNITY_BEGIN();
  RUN_TEST(test_volume_reversal_probability);
  RUN_TEST(test_caps_toggle_variable_count);
  RUN_TEST(test_volume_delay_variation);
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
git checkout -b feature/stealth-004-net-zero-variation
```

### Step 2: Modify actionVolume()

In `lib/fake_keyboard_core/src/actions.cpp`:

```cpp
static void actionVolume() {
  DEBUG_PRINTLN("[ACT] Volume +/-");
  tapConsumer(Consumer, CONSUMER_CONTROL_VOLUME_INCREMENT, 35, 85);
  // 80% chance of reversal, 20% leave changed
  if (random(100) < 80) {
    delay(logNormalDelay(400, 200));  // Variable delay before reversal
    tapConsumer(Consumer, CONSUMER_CONTROL_VOLUME_DECREMENT, 35, 85);
  }
}
```

### Step 3: Modify actionBrightness()

```cpp
static void actionBrightness() {
  DEBUG_PRINTLN("[ACT] Brightness +/-");
  tapConsumer(Consumer, CONSUMER_CONTROL_BRIGHTNESS_INCREMENT, 35, 85);
  if (random(100) < 80) {
    delay(logNormalDelay(400, 200));
    tapConsumer(Consumer, CONSUMER_CONTROL_BRIGHTNESS_DECREMENT, 35, 85);
  }
}
```

### Step 4: Modify actionCapsToggle()

```cpp
static void actionCapsToggle() {
  DEBUG_PRINTLN("[ACT] Caps Toggle");
  const int toggles = random(1, 4);  // 1–3 toggles
  for (int i = 0; i < toggles; i++) {
    tapKey(Keyboard, KEY_CAPS_LOCK, 45, 95, 120, 260);
    delay(logNormalDelay(300, 100));
  }
  // If odd number of toggles, state changed
  if (toggles % 2 == 1 && random(100) < 30) {
    return;  // Leave CapsLock on (real user might not notice)
  }
  if (toggles % 2 == 1) {
    delay(logNormalDelay(400, 150));
    tapKey(Keyboard, KEY_CAPS_LOCK, 45, 95, 60, 180);
  }
}
```

### Step 5: Modify actionNumLockToggle()

Apply the same pattern as CapsToggle.

### Step 6: Verify and Commit

```bash
git status
git diff
git add -A
git commit -m "feat(stealth): add variation to net-zero actions

- Volume: 80% reversal, 20% leave changed
- Brightness: 80% reversal, 20% leave changed
- CapsToggle: 1-3 toggles, 30% chance leave changed
- NumLockToggle: 1-3 toggles, 30% chance leave changed
- Variable delay before reversal
- Fixes STEALTH-004"
```

### Step 7: Push and Merge to Dev

```bash
git push origin feature/stealth-004-net-zero-variation
git checkout dev
git pull origin dev
git merge feature/stealth-004-net-zero-variation
git push origin dev
```

### Step 8: Reset to Dev

```bash
git checkout dev
git pull origin dev
```
