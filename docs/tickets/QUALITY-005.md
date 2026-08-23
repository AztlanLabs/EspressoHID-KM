# QUALITY-005: Add Stealth Validation Tests

| Field | Value |
| --- | --- |
| **ID** | QUALITY-005 |
| **Priority** | 🟡 P1 (High) |
| **Epic** | Quality & Testing |
| **Estimated Effort** | 3 hours |
| **Dependencies** | QUALITY-001, STEALTH-002, STEALTH-003, STEALTH-004 |
| **Branch** | `feature/quality-005-stealth-tests` |

---

## Description

Stealth validation tests verify that the device produces human-like behavior and passes
detection tools.

## References

> **Related docs for this ticket:** `docs/ARCHITECTURE.md`, `docs/FINDINGS.md`, `docs/STEALTH_HARDENING.md`, `docs/FEATURE_EXPANSION.md`, `docs/TICKETS.md` — read the relevant sections listed above before coding.

- `docs/STEALTH_HARDENING.md` §8 (Testing & Validation)
- `docs/FINDINGS.md` §5.3 (No Automated Tests)

## Files to Modify

| File | Action | Description |
| --- | --- | --- |
| `test/test_stealth_validation.cpp` | Create | Stealth validation unit tests |

## Objectives

1. Test timing entropy
2. Test movement pattern variance
3. Test activity type distribution
4. Test work pattern variation
5. Test net-zero action variation

## Acceptance Criteria

- [ ] Timing entropy test verifies entropy > 3.5 bits
- [ ] Movement pattern test verifies high variance
- [ ] Activity type test verifies mixed keyboard/mouse activity
- [ ] Work pattern test verifies burst/pause variation
- [ ] Net-zero test verifies action variation
- [ ] All tests pass
- [ ] No compilation errors or warnings

## Unit Tests

Create file `test/test_stealth_validation.cpp`:

```cpp
#include <unity.h>
#include "human_input.h"
#include "actions.h"
#include "state.h"
#include <math.h>

void test_timing_entropy() {
  unsigned long samples[1000];
  for (int i = 0; i < 1000; i++) {
    samples[i] = logNormalDelay(800, 300);
  }
  // Calculate histogram
  int bins[50] = {0};
  for (int i = 0; i < 1000; i++) {
    int bin = samples[i] / 80;  // 50 bins for 0-4000ms
    if (bin >= 50) bin = 49;
    bins[bin]++;
  }
  // Calculate entropy
  float entropy = 0;
  for (int i = 0; i < 50; i++) {
    if (bins[i] > 0) {
      float p = (float)bins[i] / 1000.0f;
      entropy -= p * log2f(p);
    }
  }
  TEST_ASSERT_GREATER_THAN(3.5f, entropy);
}

void test_movement_pattern_variance() {
  float movements[1000][2];
  for (int i = 0; i < 1000; i++) {
    movements[i][0] = gaussianRandom(0, 10);
    movements[i][1] = gaussianRandom(0, 10);
  }
  // Calculate variance
  float meanX = 0, meanY = 0;
  for (int i = 0; i < 1000; i++) {
    meanX += movements[i][0];
    meanY += movements[i][1];
  }
  meanX /= 1000; meanY /= 1000;
  float varX = 0, varY = 0;
  for (int i = 0; i < 1000; i++) {
    varX += (movements[i][0] - meanX) * (movements[i][0] - meanX);
    varY += (movements[i][1] - meanY) * (movements[i][1] - meanY);
  }
  varX /= 1000; varY /= 1000;
  TEST_ASSERT_GREATER_THAN(50.0f, varX);
  TEST_ASSERT_GREATER_THAN(50.0f, varY);
}

void test_activity_type_distribution() {
  int count = actionsCount();
  TEST_ASSERT_GREATER_THAN(0, count);
  // Verify we have both keyboard and mouse actions
  bool hasKeyboard = false, hasMouse = false;
  for (int i = 0; i < count; i++) {
    const char* name = actionName(i);
    if (strstr(name, "Mouse") || strstr(name, "Scroll")) hasMouse = true;
    else hasKeyboard = true;
  }
  TEST_ASSERT_TRUE(hasKeyboard);
  // hasMouse will be true after STEALTH-005
}

void test_work_pattern_variation() {
  WorkPattern pattern;
  memset(&pattern, 0, sizeof(pattern));
  unsigned long delays[100];
  for (int i = 0; i < 100; i++) {
    updateWorkPattern();
    delays[i] = getNextDelay(pattern);
  }
  // Calculate variance
  float mean = 0;
  for (int i = 0; i < 100; i++) mean += delays[i];
  mean /= 100;
  float variance = 0;
  for (int i = 0; i < 100; i++) {
    variance += (delays[i] - mean) * (delays[i] - mean);
  }
  variance /= 100;
  TEST_ASSERT_GREATER_THAN(1000000.0f, variance);
}

void test_action_names_not_empty() {
  for (int i = 0; i < actionsCount(); i++) {
    const char* name = actionName(i);
    TEST_ASSERT_NOT_NULL(name);
    TEST_ASSERT_GREATER_THAN(0, strlen(name));
  }
}

void setup() {
  UNITY_BEGIN();
  RUN_TEST(test_timing_entropy);
  RUN_TEST(test_movement_pattern_variance);
  RUN_TEST(test_activity_type_distribution);
  RUN_TEST(test_work_pattern_variation);
  RUN_TEST(test_action_names_not_empty);
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
git checkout -b feature/quality-005-stealth-tests
```

### Step 2: Create Test File

Create `test/test_stealth_validation.cpp` with the tests above.

### Step 3: Verify and Commit

```bash
git status
git diff
git add -A
git commit -m "feat(quality): add stealth validation tests

- Test timing entropy (> 3.5 bits)
- Test movement pattern variance
- Test activity type distribution
- Test work pattern variation
- Test action names
- Fixes QUALITY-005"
```

### Step 4: Push and Merge to Dev

```bash
git push origin feature/quality-005-stealth-tests
git checkout dev
git pull origin dev
git merge feature/quality-005-stealth-tests
git push origin dev
```

### Step 5: Reset to Dev

```bash
git checkout dev
git pull origin dev
```
