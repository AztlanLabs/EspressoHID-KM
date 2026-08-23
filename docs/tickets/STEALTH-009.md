# STEALTH-009: Add Circadian Rhythm Variation

| Field | Value |
| --- | --- |
| **ID** | STEALTH-009 |
| **Priority** | 🟡 P1 (High) |
| **Epic** | Stealth Hardening |
| **Estimated Effort** | 3 hours |
| **Dependencies** | STEALTH-003 |
| **Branch** | `feature/stealth-009-circadian-rhythm` |

---

## Description

Activity patterns should vary throughout the day to simulate realistic work behavior. Real
humans are more active in the morning, less active after lunch, and have natural breaks
throughout the day. This ticket adds circadian rhythm variation to the work pattern.

## Detection Vector

**Behavioral analytics** — consistent activity throughout the day is suspicious.

## References

> **Related docs for this ticket:** `docs/ARCHITECTURE.md`, `docs/FINDINGS.md`, `docs/STEALTH_HARDENING.md`, `docs/FEATURE_EXPANSION.md`, `docs/TICKETS.md` — read the relevant sections listed above before coding.

- `docs/STEALTH_HARDENING.md` §6.4
- `docs/ARCHITECTURE.md` §6 (Runtime Behavior)

## Files to Modify

| File | Action | Description |
| --- | --- | --- |
| `include/config.h` | Modify | Add circadian rhythm constants |
| `lib/fake_keyboard_core/src/profiles.cpp` | Modify | Implement circadian rhythm functions |
| `src/main.cpp` | Modify | Integrate circadian rhythm into main loop |

## Objectives

1. Add time-of-day awareness (via NTP)
2. Implement circadian rhythm multiplier
3. Adjust activity frequency based on time of day
4. Add natural break simulation (lunch, coffee breaks)
5. Integrate with work pattern

## Acceptance Criteria

- [ ] Time-of-day is obtained from NTP
- [ ] Circadian rhythm multiplier is implemented (0.0–1.0)
- [ ] Activity is more frequent in the morning (9–12)
- [ ] Activity is less frequent after lunch (13–15)
- [ ] Activity is moderate in the afternoon (15–17)
- [ ] Natural breaks are simulated (lunch 12–13, coffee 10:30, 15:00)
- [ ] Break duration is variable (15–60 min)
- [ ] Circadian rhythm integrates with work pattern
- [ ] No compilation errors or warnings

## Unit Tests

Create file `test/test_circadian.cpp`:

```cpp
#include <unity.h>

void test_circadian_morning_multiplier() {
  float multiplier = getCircadianMultiplier(10);
  TEST_ASSERT_GREATER_THAN(0.8f, multiplier);
}

void test_circadian_afternoon_multiplier() {
  float multiplier = getCircadianMultiplier(14);
  TEST_ASSERT_LESS_THAN(0.6f, multiplier);
}

void test_circadian_evening_multiplier() {
  float multiplier = getCircadianMultiplier(18);
  TEST_ASSERT_LESS_THAN(0.3f, multiplier);
}

void test_circadian_lunch_break() {
  bool isBreak = isBreakTime(12, 30);
  TEST_ASSERT_TRUE(isBreak);
}

void setup() {
  UNITY_BEGIN();
  RUN_TEST(test_circadian_morning_multiplier);
  RUN_TEST(test_circadian_afternoon_multiplier);
  RUN_TEST(test_circadian_evening_multiplier);
  RUN_TEST(test_circadian_lunch_break);
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
git checkout -b feature/stealth-009-circadian-rhythm
```

### Step 2: Add Config Constants

In `include/config.h`:

```cpp
#define CIRCADIAN_MORNING_START     9
#define CIRCADIAN_MORNING_END       12
#define CIRCADIAN_AFTERNOON_START   13
#define CIRCADIAN_AFTERNOON_END     17
#define CIRCADIAN_LUNCH_START       12
#define CIRCADIAN_LUNCH_END         13
#define CIRCADIAN_COFFEE_HOUR       10
#define CIRCADIAN_COFFEE_MINUTE     30
```

### Step 3: Implement Circadian Functions

```cpp
float getCircadianMultiplier(int hour) {
  if (hour >= 9 && hour < 12) return 1.0f;
  if (hour >= 13 && hour < 15) return 0.5f;
  if (hour >= 15 && hour < 17) return 0.7f;
  if (hour >= 17 && hour < 20) return 0.3f;
  return 0.1f;
}

bool isBreakTime(int hour, int minute) {
  if (hour == 12 && minute >= 0 && minute < 60) return true;
  if (hour == 10 && minute >= 30 && minute < 45) return true;
  if (hour == 15 && minute >= 0 && minute < 15) return true;
  return false;
}
```

### Step 4: Integrate into Main Loop

Use `getCircadianMultiplier()` to adjust action probability.

### Step 5: Verify and Commit

```bash
git status
git diff
git add -A
git commit -m "feat(stealth): add circadian rhythm variation

- Activity varies throughout the day
- Morning: high, afternoon: moderate, evening: low
- Natural breaks: lunch, coffee
- Fixes STEALTH-009"
```

### Step 6: Push and Merge to Dev

```bash
git push origin feature/stealth-009-circadian-rhythm
git checkout dev
git pull origin dev
git merge feature/stealth-009-circadian-rhythm
git push origin dev
```

### Step 7: Reset to Dev

```bash
git checkout dev
git pull origin dev
```
