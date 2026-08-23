# STEALTH-003: Implement Burst/Pause Work Patterns

| Field | Value |
| --- | --- |
| **ID** | STEALTH-003 |
| **Priority** | 🔴 P0 (Critical) |
| **Epic** | Stealth Hardening |
| **Estimated Effort** | 3 hours |
| **Dependencies** | STEALTH-002 |
| **Branch** | `feature/stealth-003-burst-pause-patterns` |

---

## Description

Actions happen at predictable intervals (10–60 s for ACTIVE, 45–180 s for MEETING) with
uniform distribution. Real human work patterns have bursts of activity followed by pauses
(reading, thinking, bathroom breaks). Monitoring tools flag consistent activity without
natural breaks.

## Detection Vector

**Behavioral analytics** — "95%+ activity for 30 min" and "under 4% fluctuation for 90 min"
are Hubstaff thresholds for flagging simulated activity.

> "A perfectly metronomic wiggle for eight hours is not how a human uses a mouse."

## References

> **Related docs for this ticket:** `docs/ARCHITECTURE.md`, `docs/FINDINGS.md`, `docs/STEALTH_HARDENING.md`, `docs/FEATURE_EXPANSION.md`, `docs/TICKETS.md` — read the relevant sections listed above before coding.

- `docs/STEALTH_HARDENING.md` §V4, §6.4
- `docs/ARCHITECTURE.md` §6 (Runtime Behavior)
- `docs/FINDINGS.md` §3.1 (Single-Threaded Cooperative Loop)

## Files to Modify

| File | Action | Description |
| --- | --- | --- |
| `include/config.h` | Modify | Add burst/pause timing constants |
| `include/state.h` | Modify | Add `WorkPattern` struct and global instance |
| `lib/fake_keyboard_core/src/state.cpp` | Modify | Define work pattern global |
| `src/main.cpp` | Modify | Integrate work pattern into main loop |

## Objectives

1. Implement `WorkPattern` struct with burst/pause state
2. Implement `updateWorkPattern()` function
3. Implement `getNextDelay()` function that returns humanized delays based on work pattern
4. Integrate work pattern into the main loop
5. Ensure activity patterns have natural variation

## Acceptance Criteria

- [ ] `WorkPattern` struct is defined with `sessionStart`, `lastAction`, `burstEnd`, `pauseEnd`, `burstCount`, `inBurst`, `inPause` fields
- [ ] `updateWorkPattern()` function is implemented
- [ ] `getNextDelay()` function returns different delays based on work pattern state
- [ ] Burst mode produces faster actions (5–15 s intervals)
- [ ] Pause mode produces slower actions (1–5 min intervals)
- [ ] Normal mode produces moderate actions (15–90 s intervals)
- [ ] Burst mode lasts 2–5 minutes
- [ ] Pause mode lasts 1–5 minutes
- [ ] 5% chance of entering burst mode from normal mode
- [ ] Work pattern is integrated into the main loop
- [ ] Activity patterns have natural variation (not metronomic)
- [ ] No compilation errors or warnings

## Unit Tests

Create file `test/test_work_pattern.cpp`:

```cpp
#include <unity.h>
#include "state.h"
#include "config.h"

void test_work_pattern_initialization() {
  WorkPattern pattern;
  memset(&pattern, 0, sizeof(pattern));
  TEST_ASSERT_FALSE(pattern.inBurst);
  TEST_ASSERT_FALSE(pattern.inPause);
  TEST_ASSERT_EQUAL(0, pattern.burstCount);
}

void test_burst_mode_fast_delays() {
  WorkPattern pattern;
  memset(&pattern, 0, sizeof(pattern));
  pattern.inBurst = true;
  unsigned long delay = getNextDelay(pattern);
  TEST_ASSERT_GREATER_OR_EQUAL(5000, delay);
  TEST_ASSERT_LESS_OR_EQUAL(15000, delay);
}

void test_pause_mode_slow_delays() {
  WorkPattern pattern;
  memset(&pattern, 0, sizeof(pattern));
  pattern.inPause = true;
  unsigned long delay = getNextDelay(pattern);
  TEST_ASSERT_GREATER_OR_EQUAL(60000, delay);
  TEST_ASSERT_LESS_OR_EQUAL(300000, delay);
}

void test_normal_mode_moderate_delays() {
  WorkPattern pattern;
  memset(&pattern, 0, sizeof(pattern));
  unsigned long delay = getNextDelay(pattern);
  TEST_ASSERT_GREATER_OR_EQUAL(15000, delay);
  TEST_ASSERT_LESS_OR_EQUAL(90000, delay);
}

void setup() {
  UNITY_BEGIN();
  RUN_TEST(test_work_pattern_initialization);
  RUN_TEST(test_burst_mode_fast_delays);
  RUN_TEST(test_pause_mode_slow_delays);
  RUN_TEST(test_normal_mode_moderate_delays);
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
git checkout -b feature/stealth-003-burst-pause-patterns
```

### Step 2: Add Config Constants

In `include/config.h`, add:

```cpp
#define BURST_DURATION_MIN_MS   120000UL   // 2 min
#define BURST_DURATION_MAX_MS   300000UL   // 5 min
#define PAUSE_DURATION_MIN_MS   60000UL    // 1 min
#define PAUSE_DURATION_MAX_MS   300000UL   // 5 min
#define BURST_CHANCE_PERCENT    5          // 5% chance per check
#define BURST_INTERVAL_MIN_MS   5000UL     // 5 s
#define BURST_INTERVAL_MAX_MS   15000UL    // 15 s
#define PAUSE_INTERVAL_MIN_MS   60000UL    // 1 min
#define PAUSE_INTERVAL_MAX_MS   300000UL   // 5 min
```

### Step 3: Add WorkPattern Struct

In `include/state.h`, add:

```cpp
struct WorkPattern {
  unsigned long sessionStart;
  unsigned long lastAction;
  unsigned long burstEnd;
  unsigned long pauseEnd;
  uint8_t burstCount;
  bool inBurst;
  bool inPause;
};

extern WorkPattern workPattern;

void updateWorkPattern();
unsigned long getNextDelay(const WorkPattern& pattern);
```

In `lib/fake_keyboard_core/src/state.cpp`, add:

```cpp
WorkPattern workPattern = {0, 0, 0, 0, 0, false, false};
```

### Step 4: Implement Functions

In `lib/fake_keyboard_core/src/profiles.cpp` or a new file, implement:

```cpp
void updateWorkPattern() {
  const unsigned long now = millis();
  if (workPattern.inBurst) {
    if (now >= workPattern.burstEnd) {
      workPattern.inBurst = false;
      workPattern.inPause = true;
      workPattern.pauseEnd = now + random(PAUSE_DURATION_MIN_MS, PAUSE_DURATION_MAX_MS);
    }
    return;
  }
  if (workPattern.inPause) {
    if (now >= workPattern.pauseEnd) {
      workPattern.inPause = false;
      workPattern.burstCount++;
    }
    return;
  }
  if (random(100) < BURST_CHANCE_PERCENT) {
    workPattern.inBurst = true;
    workPattern.burstEnd = now + random(BURST_DURATION_MIN_MS, BURST_DURATION_MAX_MS);
  }
}

unsigned long getNextDelay(const WorkPattern& pattern) {
  if (pattern.inBurst) return random(BURST_INTERVAL_MIN_MS, BURST_INTERVAL_MAX_MS);
  if (pattern.inPause) return random(PAUSE_INTERVAL_MIN_MS, PAUSE_INTERVAL_MAX_MS);
  return random(profileIntervalMin(), profileIntervalMax());
}
```

### Step 5: Integrate into Main Loop

In `src/main.cpp`, call `updateWorkPattern()` in the main loop and use `getNextDelay()`
for interval calculation.

### Step 6: Verify and Commit

```bash
git status
git diff
git add -A
git commit -m "feat(stealth): implement burst/pause work patterns

- Add WorkPattern struct with burst/pause state
- Implement updateWorkPattern() and getNextDelay()
- Burst mode: 5-15s intervals for 2-5 min
- Pause mode: 1-5 min intervals for 1-5 min
- 5% chance of entering burst from normal
- Fixes STEALTH-003"
```

### Step 7: Push and Merge to Dev

```bash
git push origin feature/stealth-003-burst-pause-patterns
git checkout dev
git pull origin dev
git merge feature/stealth-003-burst-pause-patterns
git push origin dev
```

### Step 8: Reset to Dev

```bash
git checkout dev
git pull origin dev
```
