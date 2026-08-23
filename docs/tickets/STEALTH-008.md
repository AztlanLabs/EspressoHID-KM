# STEALTH-008: Improve Serial Number Format

| Field | Value |
| --- | --- |
| **ID** | STEALTH-008 |
| **Priority** | 🟡 P1 (High) |
| **Epic** | Stealth Hardening |
| **Estimated Effort** | 2 hours |
| **Dependencies** | None |
| **Branch** | `feature/stealth-008-serial-number` |

---

## Description

The current serial number is a 16-hex-char string which is unusual for real keyboards. Real
keyboards have serial numbers in specific formats (e.g., Dell uses `CN0XXXXX`, HP uses
`CZCXXXXX`). This ticket generates serial numbers that match the format of the spoofed
manufacturer.

## Detection Vector

**USB enumeration forensics** — serial number format analysis.

## References

> **Related docs for this ticket:** `docs/ARCHITECTURE.md`, `docs/FINDINGS.md`, `docs/STEALTH_HARDENING.md`, `docs/FEATURE_EXPANSION.md`, `docs/TICKETS.md` — read the relevant sections listed above before coding.

- `docs/STEALTH_HARDENING.md` §V8, §6.8
- `docs/ARCHITECTURE.md` §12 (Configuration Reference)

## Files to Modify

| File | Action | Description |
| --- | --- | --- |
| `lib/fake_keyboard_core/src/usb_identity.cpp` | Modify | Implement `generateSerialNumber()`, update `applyRandomIdentity()` |

## Objectives

1. Define serial number formats for each manufacturer
2. Implement `generateSerialNumber()` function
3. Update `applyRandomIdentity()` to use format-specific serial numbers
4. Verify serial numbers match manufacturer formats

## Acceptance Criteria

- [ ] Serial number formats are defined for all 5 manufacturers
- [ ] `generateSerialNumber()` function generates format-specific serial numbers
- [ ] Dell serial numbers start with `CN0` followed by alphanumeric characters
- [ ] HP serial numbers start with `CZC` followed by alphanumeric characters
- [ ] Logitech serial numbers are alphanumeric (no prefix)
- [ ] Microsoft serial numbers are alphanumeric (no prefix)
- [ ] Lenovo serial numbers start with `R90` followed by alphanumeric characters
- [ ] Serial numbers are 10–16 characters long
- [ ] `applyRandomIdentity()` uses `generateSerialNumber()`
- [ ] No compilation errors or warnings

## Unit Tests

Create file `test/test_serial_number.cpp`:

```cpp
#include <unity.h>
#include "usb_identity.h"

void test_dell_serial_format() {
  char serial[20];
  generateSerialNumber(serial, sizeof(serial), 0);
  TEST_ASSERT_EQUAL('C', serial[0]);
  TEST_ASSERT_EQUAL('N', serial[1]);
  TEST_ASSERT_EQUAL('0', serial[2]);
  TEST_ASSERT_EQUAL(10, strlen(serial));
}

void test_hp_serial_format() {
  char serial[20];
  generateSerialNumber(serial, sizeof(serial), 1);
  TEST_ASSERT_EQUAL('C', serial[0]);
  TEST_ASSERT_EQUAL('Z', serial[1]);
  TEST_ASSERT_EQUAL('C', serial[2]);
  TEST_ASSERT_EQUAL(10, strlen(serial));
}

void test_serial_uniqueness() {
  char serial1[20], serial2[20];
  generateSerialNumber(serial1, sizeof(serial1), 0);
  generateSerialNumber(serial2, sizeof(serial2), 0);
  TEST_ASSERT_NOT_EQUAL(0, strcmp(serial1, serial2));
}

void setup() {
  UNITY_BEGIN();
  RUN_TEST(test_dell_serial_format);
  RUN_TEST(test_hp_serial_format);
  RUN_TEST(test_serial_uniqueness);
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
git checkout -b feature/stealth-008-serial-number
```

### Step 2: Implement generateSerialNumber()

In `lib/fake_keyboard_core/src/usb_identity.cpp`:

```cpp
void generateSerialNumber(char* buf, size_t len, int identityIndex) {
  const char* prefixes[] = {"CN0", "CZC", "", "", "R90"};
  const char* chars = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ";
  size_t pos = 0;
  const char* prefix = prefixes[identityIndex % USB_IDENTITY_COUNT];
  size_t prefixLen = strlen(prefix);
  if (prefixLen > 0 && prefixLen < len) {
    memcpy(buf, prefix, prefixLen);
    pos = prefixLen;
  }
  for (; pos < len - 1; pos++) {
    buf[pos] = chars[random(0, 36)];
  }
  buf[len - 1] = '\0';
}
```

### Step 3: Update applyRandomIdentity()

Replace the current serial number generation with `generateSerialNumber()`.

### Step 4: Verify and Commit

```bash
git status
git diff
git add -A
git commit -m "feat(stealth): improve serial number format

- Generate manufacturer-specific serial numbers
- Dell: CN0XXXXX, HP: CZCXXXXX, Lenovo: R90XXXXX
- Logitech/Microsoft: alphanumeric
- Fixes STEALTH-008"
```

### Step 5: Push and Merge to Dev

```bash
git push origin feature/stealth-008-serial-number
git checkout dev
git pull origin dev
git merge feature/stealth-008-serial-number
git push origin dev
```

### Step 6: Reset to Dev

```bash
git checkout dev
git pull origin dev
```
