# QUALITY-003: Add Timing Distribution Unit Tests

| Field | Value |
| --- | --- |
| **ID** | QUALITY-003 |
| **Priority** | 🔴 P0 (Critical) |
| **Epic** | Quality & Testing |
| **Estimated Effort** | 2 hours |
| **Dependencies** | QUALITY-001, STEALTH-002 |
| **Branch** | `feature/quality-003-timing-tests` |

---

## Description

Timing distributions are critical for stealth. This ticket adds unit tests to verify that
timing distributions are human-like (log-normal, gamma) and not uniform.

## References

> **Related docs for this ticket:** `docs/ARCHITECTURE.md`, `docs/FINDINGS.md`, `docs/STEALTH_HARDENING.md`, `docs/FEATURE_EXPANSION.md`, `docs/TICKETS.md` — read the relevant sections listed above before coding.

- `docs/STEALTH_HARDENING.md` §8.2 (Timing Entropy Testing)
- `docs/ARCHITECTURE.md` §15 (Human Input Timing)

## Files to Modify

| File | Action | Description |
| --- | --- | --- |
| `test/test_timing_distribution.cpp` | Create | Timing distribution unit tests |

## Objectives

1. Test gaussian random mean and stddev
2. Test log-normal delay range and median
3. Test gamma delay range
4. Test timing entropy
5. Test that distributions are not uniform

## Acceptance Criteria

- [ ] Gaussian random test verifies mean is close to expected
- [ ] Gaussian random test verifies stddev is close to expected
- [ ] Log-normal delay test verifies range is reasonable
- [ ] Log-normal delay test verifies median is close to expected
- [ ] Gamma delay test verifies range is reasonable
- [ ] Timing entropy test verifies entropy > 3.5 bits
- [ ] Non-uniform test verifies distributions are not uniform
- [ ] All tests pass
- [ ] No compilation errors or warnings

## Unit Tests

Create file `test/test_timing_distribution.cpp`:

```cpp
#include <unity.h>
#include "human_input.h"
#include <math.h>

void test_gaussian_random_mean() {
  float sum = 0;
  for (int i = 0; i < 1000; i++) {
    sum += gaussianRandom(100.0f, 20.0f);
  }
  float mean = sum / 1000.0f;
  TEST_ASSERT_FLOAT_WITHIN(10.0f, 100.0f, mean);
}

void test_gaussian_random_stddev() {
  float samples[1000];
  for (int i = 0; i < 1000; i++) {
    samples[i] = gaussianRandom(100.0f, 20.0f);
  }
  float mean = 0;
  for (int i = 0; i < 1000; i++) mean += samples[i];
  mean /= 1000.0f;
  float variance = 0;
  for (int i = 0; i < 1000; i++) {
    variance += (samples[i] - mean) * (samples[i] - mean);
  }
  variance /= 1000.0f;
  float stddev = sqrt(variance);
  TEST_ASSERT_FLOAT_WITHIN(10.0f, 20.0f, stddev);
}

void test_log_normal_delay_range() {
  for (int i = 0; i < 100; i++) {
    unsigned long delay = logNormalDelay(800, 300);
    TEST_ASSERT_GREATER_OR_EQUAL(10, delay);
    TEST_ASSERT_LESS_OR_EQUAL(4000, delay);
  }
}

void test_log_normal_delay_not_uniform() {
  // Verify delays are not uniformly distributed
  unsigned long samples[1000];
  for (int i = 0; i < 1000; i++) {
    samples[i] = logNormalDelay(800, 300);
  }
  // Count samples in each quartile
  int q1 = 0, q2 = 0, q3 = 0, q4 = 0;
  for (int i = 0; i < 1000; i++) {
    if (samples[i] < 400) q1++;
    else if (samples[i] < 800) q2++;
    else if (samples[i] < 1200) q3++;
    else q4++;
  }
  // Log-normal should have more samples in lower quartiles
  TEST_ASSERT_GREATER_THAN(q4, q1);
}

void test_gamma_delay_range() {
  for (int i = 0; i < 100; i++) {
    unsigned long delay = gammaDelay(2.0f, 400.0f);
    TEST_ASSERT_GREATER_OR_EQUAL(10, delay);
    TEST_ASSERT_LESS_OR_EQUAL(4000, delay);
  }
}

void setup() {
  UNITY_BEGIN();
  RUN_TEST(test_gaussian_random_mean);
  RUN_TEST(test_gaussian_random_stddev);
  RUN_TEST(test_log_normal_delay_range);
  RUN_TEST(test_log_normal_delay_not_uniform);
  RUN_TEST(test_gamma_delay_range);
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
git checkout -b feature/quality-003-timing-tests
```

### Step 2: Create Test File

Create `test/test_timing_distribution.cpp` with the tests above.

### Step 3: Verify and Commit

```bash
git status
git diff
git add -A
git commit -m "feat(quality): add timing distribution unit tests

- Test gaussian random mean and stddev
- Test log-normal delay range and distribution
- Test gamma delay range
- Verify distributions are not uniform
- Fixes QUALITY-003"
```

### Step 4: Push and Merge to Dev

```bash
git push origin feature/quality-003-timing-tests
git checkout dev
git pull origin dev
git merge feature/quality-003-timing-tests
git push origin dev
```

### Step 5: Reset to Dev

```bash
git checkout dev
git pull origin dev
```
