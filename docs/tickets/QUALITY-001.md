# QUALITY-001: Add Unit Test Framework

| Field | Value |
| --- | --- |
| **ID** | QUALITY-001 |
| **Priority** | 🔴 P0 (Critical) |
| **Epic** | Quality & Testing |
| **Estimated Effort** | 2 hours |
| **Dependencies** | None |
| **Branch** | `feature/quality-001-test-framework` |

---

## Description

The project currently has no automated tests. This ticket adds a unit test framework using
Unity (PlatformIO's default test framework) and sets up the test infrastructure.

## References

> **Related docs for this ticket:** `docs/ARCHITECTURE.md`, `docs/FINDINGS.md`, `docs/STEALTH_HARDENING.md`, `docs/FEATURE_EXPANSION.md`, `docs/TICKETS.md` — read the relevant sections listed above before coding.

- `docs/FINDINGS.md` §5.3 (No Automated Tests)
- `docs/ARCHITECTURE.md` §2 (Repository Structure)

## Files to Modify

| File | Action | Description |
| --- | --- | --- |
| `platformio.ini` | Modify | Add Unity dependency, configure test runner |
| `test/test_example.cpp` | Create | Example test file |

## Objectives

1. Add Unity test framework dependency
2. Create test directory structure
3. Create test runner configuration
4. Create example test file
5. Verify tests run successfully

## Acceptance Criteria

- [ ] Unity test framework is added to `platformio.ini`
- [ ] Test directory structure is created (`test/`)
- [ ] Test runner configuration is created
- [ ] Example test file is created
- [ ] Tests run successfully with `pio test`
- [ ] Test output is formatted correctly
- [ ] No compilation errors or warnings

## Unit Tests

Create file `test/test_example.cpp`:

```cpp
#include <unity.h>

void test_example_assertion() {
  TEST_ASSERT_EQUAL(2, 1 + 1);
}

void test_example_string() {
  TEST_ASSERT_EQUAL_STRING("hello", "hello");
}

void test_example_float() {
  TEST_ASSERT_FLOAT_WITHIN(0.1f, 3.14f, 3.14f);
}

void setup() {
  UNITY_BEGIN();
  RUN_TEST(test_example_assertion);
  RUN_TEST(test_example_string);
  RUN_TEST(test_example_float);
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
git checkout -b feature/quality-001-test-framework
```

### Step 2: Configure platformio.ini

Add test configuration:

```ini
[env:esp32-s3-devkitc-1]
; ... existing config ...

; Test configuration
test_framework = unity
test_filter = test_*
```

### Step 3: Create Example Test

Create `test/test_example.cpp` with the example tests above.

### Step 4: Verify and Commit

```bash
git status
git diff
git add -A
git commit -m "feat(quality): add unit test framework

- Add Unity test framework dependency
- Configure test runner in platformio.ini
- Create example test file
- Tests run with 'pio test'
- Fixes QUALITY-001"
```

### Step 5: Push and Merge to Dev

```bash
git push origin feature/quality-001-test-framework
git checkout dev
git pull origin dev
git merge feature/quality-001-test-framework
git push origin dev
```

### Step 6: Reset to Dev

```bash
git checkout dev
git pull origin dev
```
