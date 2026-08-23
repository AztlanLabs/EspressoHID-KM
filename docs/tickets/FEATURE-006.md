# FEATURE-006: Implement Text Injection UI

| Field | Value |
| --- | --- |
| **ID** | FEATURE-006 |
| **Priority** | 🟡 P1 (High) |
| **Epic** | Feature Expansion |
| **Estimated Effort** | 2 hours |
| **Dependencies** | FEATURE-005 |
| **Branch** | `feature/feature-006-text-injection` |

---

## Description

The text injection UI provides a form for sending text with different typing modes
(humanized, fast, net-zero).

## References

> **Related docs for this ticket:** `docs/ARCHITECTURE.md`, `docs/FINDINGS.md`, `docs/STEALTH_HARDENING.md`, `docs/FEATURE_EXPANSION.md`, `docs/TICKETS.md` — read the relevant sections listed above before coding.

- `docs/FEATURE_EXPANSION.md` §4.4
- `docs/ARCHITECTURE.md` §10 (Web Dashboard)

## Files to Modify

| File | Action | Description |
| --- | --- | --- |
| `src/web_portal.cpp` | Modify | Add text injection UI to companion app HTML |

## Objectives

1. Implement text injection form
2. Implement typing mode radio buttons
3. Implement flags checkboxes (humanized, net-zero, fast)
4. Implement "Send" and "Clear" buttons
5. Implement character counter

## Acceptance Criteria

- [ ] Text injection form is implemented
- [ ] Typing mode radio buttons are implemented
- [ ] Flags checkboxes are implemented
- [ ] "Send" button sends text with selected mode and flags
- [ ] "Clear" button clears the text input
- [ ] Character counter shows current text length
- [ ] Maximum text length is 128 characters
- [ ] No JavaScript errors

## Unit Tests

Create file `test/test_text_injection.js`:

```javascript
function test_text_injection_humanized() {
  document.getElementById('text-input').value = 'Hello';
  document.getElementById('mode-humanized').checked = true;
  document.getElementById('send-text').click();
  assert(lastMessage[0] === 0x20);
  assert(lastMessage[3] & 0x01);
}

function test_text_injection_fast() {
  document.getElementById('text-input').value = 'Hello';
  document.getElementById('mode-fast').checked = true;
  document.getElementById('send-text').click();
  assert(lastMessage[0] === 0x20);
  assert(lastMessage[3] & 0x04);
}

function test_text_injection_net_zero() {
  document.getElementById('text-input').value = 'Hello';
  document.getElementById('mode-netzero').checked = true;
  document.getElementById('send-text').click();
  assert(lastMessage[0] === 0x20);
  assert(lastMessage[2] === 1);
  assert(lastMessage[3] & 0x02);
}
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
git checkout -b feature/feature-006-text-injection
```

### Step 2: Enhance Text Injection UI

Add character counter and improve the form layout in the companion app HTML.

### Step 3: Verify and Commit

```bash
git status
git diff
git add -A
git commit -m "feat(ui): implement text injection UI

- Text injection form with mode selection
- Character counter (max 128)
- Humanized, fast, net-zero modes
- Fixes FEATURE-006"
```

### Step 4: Push and Merge to Dev

```bash
git push origin feature/feature-006-text-injection
git checkout dev
git pull origin dev
git merge feature/feature-006-text-injection
git push origin dev
```

### Step 5: Reset to Dev

```bash
git checkout dev
git pull origin dev
```
