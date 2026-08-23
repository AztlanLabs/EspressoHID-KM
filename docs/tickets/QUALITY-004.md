# QUALITY-004: Add NVS Config Unit Tests

| Field | Value |
| --- | --- |
| **ID** | QUALITY-004 |
| **Priority** | 🟡 P1 (High) |
| **Epic** | Quality & Testing |
| **Estimated Effort** | 2 hours |
| **Dependencies** | QUALITY-001 |
| **Branch** | `feature/quality-004-nvs-tests` |

---

## Description

NVS configuration persistence is critical for reliability. This ticket adds unit tests to
verify that configuration is persisted correctly and survives reboots.

## References

> **Related docs for this ticket:** `docs/ARCHITECTURE.md`, `docs/FINDINGS.md`, `docs/STEALTH_HARDENING.md`, `docs/FEATURE_EXPANSION.md`, `docs/TICKETS.md` — read the relevant sections listed above before coding.

- `docs/ARCHITECTURE.md` §13 (NVS Storage Schema)
- `docs/FINDINGS.md` §3.1 (Configuration Flow)

## Files to Modify

| File | Action | Description |
| --- | --- | --- |
| `test/test_nvs_config.cpp` | Create | NVS config unit tests |

## Objectives

1. Test NVS initialization
2. Test Wi-Fi credentials persistence
3. Test profile interval persistence
4. Test action mask persistence
5. Test weight persistence
6. Test factory reset

## Acceptance Criteria

- [ ] NVS initialization test verifies `runtimeConfigBegin()` succeeds
- [ ] Wi-Fi credentials test verifies SSID and password are persisted
- [ ] Profile interval test verifies min/max are persisted
- [ ] Action mask test verifies mask is persisted
- [ ] Weight test verifies weights are persisted
- [ ] Factory reset test verifies all keys are cleared
- [ ] All tests pass
- [ ] No compilation errors or warnings

## Unit Tests

Create file `test/test_nvs_config.cpp`:

```cpp
#include <unity.h>
#include "runtime_config.h"

void test_nvs_initialization() {
  TEST_ASSERT_TRUE(runtimeConfigBegin());
}

void test_wifi_credentials_persistence() {
  runtimeConfigSetWifiCredentials("TestSSID", "TestPass");
  const RuntimeConfig& cfg = runtimeConfigGet();
  TEST_ASSERT_EQUAL_STRING("TestSSID", cfg.wifiSsid);
  TEST_ASSERT_EQUAL_STRING("TestPass", cfg.wifiPass);
}

void test_wifi_credentials_empty() {
  runtimeConfigSetWifiCredentials("", "");
  TEST_ASSERT_FALSE(runtimeConfigHasWifiCredentials());
}

void test_profile_interval_persistence() {
  runtimeConfigSetProfileIntervalsMs(PROFILE_ACTIVE, 5000, 30000);
  TEST_ASSERT_EQUAL(5000, runtimeConfigProfileIntervalMinMs(PROFILE_ACTIVE));
  TEST_ASSERT_EQUAL(30000, runtimeConfigProfileIntervalMaxMs(PROFILE_ACTIVE));
}

void test_profile_interval_clamping() {
  // Verify intervals are clamped to [250, 3600000]
  runtimeConfigSetProfileIntervalsMs(PROFILE_ACTIVE, 100, 5000000);
  TEST_ASSERT_GREATER_OR_EQUAL(250, runtimeConfigProfileIntervalMinMs(PROFILE_ACTIVE));
  TEST_ASSERT_LESS_OR_EQUAL(3600000, runtimeConfigProfileIntervalMaxMs(PROFILE_ACTIVE));
}

void test_action_mask_persistence() {
  runtimeConfigSetActionEnabledMask(0x12345678);
  TEST_ASSERT_EQUAL(0x12345678, runtimeConfigActionEnabledMask());
}

void test_action_mask_default() {
  runtimeConfigFactoryReset();
  TEST_ASSERT_EQUAL(0xFFFFFFFF, runtimeConfigActionEnabledMask());
}

void test_custom_text_persistence() {
  runtimeConfigSetCustomText("Hello World");
  TEST_ASSERT_EQUAL_STRING("Hello World", runtimeConfigCustomText());
}

void test_factory_reset() {
  runtimeConfigSetWifiCredentials("TestSSID", "TestPass");
  runtimeConfigSetCustomText("test");
  runtimeConfigFactoryReset();
  const RuntimeConfig& cfg = runtimeConfigGet();
  TEST_ASSERT_EQUAL_STRING("", cfg.wifiSsid);
  TEST_ASSERT_EQUAL_STRING("", cfg.wifiPass);
  TEST_ASSERT_EQUAL_STRING("", cfg.customText);
}

void setup() {
  UNITY_BEGIN();
  RUN_TEST(test_nvs_initialization);
  RUN_TEST(test_wifi_credentials_persistence);
  RUN_TEST(test_wifi_credentials_empty);
  RUN_TEST(test_profile_interval_persistence);
  RUN_TEST(test_profile_interval_clamping);
  RUN_TEST(test_action_mask_persistence);
  RUN_TEST(test_action_mask_default);
  RUN_TEST(test_custom_text_persistence);
  RUN_TEST(test_factory_reset);
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
git checkout -b feature/quality-004-nvs-tests
```

### Step 2: Create Test File

Create `test/test_nvs_config.cpp` with the tests above.

### Step 3: Verify and Commit

```bash
git status
git diff
git add -A
git commit -m "feat(quality): add NVS config unit tests

- Test NVS initialization
- Test Wi-Fi credentials persistence
- Test profile interval persistence and clamping
- Test action mask persistence
- Test custom text persistence
- Test factory reset
- Fixes QUALITY-004"
```

### Step 4: Push and Merge to Dev

```bash
git push origin feature/quality-004-nvs-tests
git checkout dev
git pull origin dev
git merge feature/quality-004-nvs-tests
git push origin dev
```

### Step 5: Reset to Dev

```bash
git checkout dev
git pull origin dev
```
