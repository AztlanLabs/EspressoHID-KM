# STEALTH-002: Replace Uniform Random with Log-Normal Distribution

| Field | Value |
| --- | --- |
| **ID** | STEALTH-002 |
| **Priority** | 🔴 P0 (Critical) |
| **Epic** | Stealth Hardening |
| **Estimated Effort** | 3 hours |
| **Dependencies** | None |
| **Branch** | `feature/stealth-002-log-normal-distribution` |

---

## Description

All timing uses Arduino's `random(min, max)` which produces a uniform distribution. Real
human timing follows a log-normal distribution — mostly fast with occasional long pauses.
This is detectable by statistical analysis of timing distributions. The Linux `hid-omg-detect`
kernel driver uses "keystroke timing entropy" as one of its three scoring factors.

## Detection Vector

**Timing entropy analysis** — uniform distributions have low entropy (~2.0 bits) and are
easily distinguishable from human timing (~3.5–4.5 bits).

> "Keystroke timing entropy" is one of the three scoring factors in the Linux
> `hid-omg-detect` kernel driver. Low-entropy timing is a strong signal of automation.

## References

> **Related docs for this ticket:** `docs/ARCHITECTURE.md`, `docs/FINDINGS.md`, `docs/STEALTH_HARDENING.md`, `docs/FEATURE_EXPANSION.md`, `docs/TICKETS.md` — read the relevant sections listed above before coding.

- `docs/STEALTH_HARDENING.md` §V3, §6.3
- `docs/ARCHITECTURE.md` §15 (Human Input Timing)
- `docs/FINDINGS.md` §3.1 (Single-Threaded Cooperative Loop)

## Files to Modify

| File | Action | Description |
| --- | --- | --- |
| `include/human_input.h` | Modify | Add `gaussianRandom()`, `logNormalDelay()`, `gammaDelay()` declarations |
| `lib/fake_keyboard_core/src/human_input.cpp` | Modify | Implement new functions, replace `random()` calls |
| `lib/fake_keyboard_core/src/actions.cpp` | Modify | Replace `random()` calls with humanized distributions |

## Objectives

1. Implement `gaussianRandom(mean, stddev)` using Box-Muller transform
2. Implement `logNormalDelay(medianMs, sigmaMs)` function
3. Implement `gammaDelay(shape, scale)` function
4. Replace all `random(min, max)` calls in `human_input.cpp` with humanized distributions
5. Replace all `random(min, max)` calls in `actions.cpp` with humanized distributions
6. Ensure timing entropy is > 3.5 bits (human-like)

## Acceptance Criteria

- [ ] `gaussianRandom(float mean, float stddev)` is implemented using Box-Muller transform
- [ ] `logNormalDelay(unsigned long medianMs, unsigned long sigmaMs)` is implemented
- [ ] `gammaDelay(float shape, float scale)` is implemented
- [ ] `humanDelayMs()` uses `logNormalDelay()` instead of `random()`
- [ ] `tapKey()` uses `logNormalDelay()` for hold and gap delays
- [ ] `chordKey()` uses `logNormalDelay()` for down-gap and hold delays
- [ ] `typeTextHuman()` uses `logNormalDelay()` for character delays
- [ ] `backspaceHuman()` uses `logNormalDelay()` for backspace delays
- [ ] `tapConsumer()` uses `logNormalDelay()` for hold delays
- [ ] All action delays in `actions.cpp` use humanized distributions
- [ ] Timing entropy is > 3.5 bits when measured over 1000 samples
- [ ] No compilation errors or warnings
- [ ] Existing functionality is not broken

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
git checkout -b feature/stealth-002-log-normal-distribution
```

### Step 2: Add Functions to human_input.h

In `include/human_input.h`, add:

```cpp
float gaussianRandom(float mean, float stddev);
unsigned long logNormalDelay(unsigned long medianMs, unsigned long sigmaMs);
unsigned long gammaDelay(float shape, float scale);
```

### Step 3: Implement Functions in human_input.cpp

In `lib/fake_keyboard_core/src/human_input.cpp`, add at the top:

```cpp
float gaussianRandom(float mean, float stddev) {
  float u1 = (float)random(1, 10000) / 10000.0f;
  float u2 = (float)random(1, 10000) / 10000.0f;
  float z = sqrt(-2.0f * log(u1)) * cos(2.0f * M_PI * u2);
  return mean + stddev * z;
}

unsigned long logNormalDelay(unsigned long medianMs, unsigned long sigmaMs) {
  float mu = log((float)medianMs);
  float sigma = (float)sigmaMs / (float)medianMs;
  float delay = exp(mu + sigma * gaussianRandom(0, 1));
  if (delay < 10) delay = 10;
  if (delay > medianMs * 5) delay = medianMs * 5;
  return (unsigned long)delay;
}

unsigned long gammaDelay(float shape, float scale) {
  float d, c, x, v, u;
  if (shape >= 1) {
    d = shape - 1.0f / 3.0f;
    c = 1.0f / sqrt(9.0f * d);
    do {
      do {
        x = gaussianRandom(0, 1);
        v = 1.0f + c * x;
      } while (v <= 0);
      v = v * v * v;
      u = (float)random(1, 10000) / 10000.0f;
    } while (u >= 1 - 0.0331f * (x * x) * (x * x) &&
             log(u) >= 0.5f * x * x + d * (1 - v + log(v)));
    return (unsigned long)(d * v * scale);
  }
  return (unsigned long)(scale * 100);
}
```

### Step 4: Replace random() Calls

Replace all `random(min, max)` calls in `human_input.cpp` and `actions.cpp` with
`logNormalDelay()` or `gammaDelay()` as appropriate.

### Step 5: Verify and Commit

```bash
git status
git diff
git add -A
git commit -m "feat(stealth): replace uniform random with log-normal distribution

- Add gaussianRandom(), logNormalDelay(), gammaDelay() functions
- Replace all random() calls in human_input.cpp and actions.cpp
- Timing entropy now > 3.5 bits (human-like)
- Fixes STEALTH-002"
```

### Step 6: Push and Merge to Dev

```bash
git push origin feature/stealth-002-log-normal-distribution
git checkout dev
git pull origin dev
git merge feature/stealth-002-log-normal-distribution
git push origin dev
```

### Step 7: Reset to Dev

```bash
git checkout dev
git pull origin dev
```
