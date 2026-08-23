# FEATURE-004: Implement Browser Mouse Capture

| Field | Value |
| --- | --- |
| **ID** | FEATURE-004 |
| **Priority** | 🔴 P0 (Critical) |
| **Epic** | Feature Expansion |
| **Estimated Effort** | 4 hours |
| **Dependencies** | FEATURE-001, FEATURE-002 |
| **Branch** | `feature/feature-004-browser-mouse` |

---

## Description

The browser UI uses the Pointer Lock API to capture mouse movement and send relative deltas
to the device via WebSocket. This enables real-time mouse control from the browser.

## References

> **Related docs for this ticket:** `docs/ARCHITECTURE.md`, `docs/FINDINGS.md`, `docs/STEALTH_HARDENING.md`, `docs/FEATURE_EXPANSION.md`, `docs/TICKETS.md` — read the relevant sections listed above before coding.

- `docs/FEATURE_EXPANSION.md` §4.2
- `docs/ARCHITECTURE.md` §10 (Web Dashboard)

## Files to Modify

| File | Action | Description |
| --- | --- | --- |
| `src/web_portal.cpp` | Modify | Add companion app HTML with mouse capture JavaScript |

## Objectives

1. Implement Pointer Lock API integration
2. Implement mouse delta accumulation
3. Implement 125 Hz throttling
4. Implement binary message construction
5. Implement mouse button handling
6. Implement scroll wheel handling

## Acceptance Criteria

- [ ] Pointer Lock API is integrated into the canvas element
- [ ] Mouse capture is activated on canvas click
- [ ] Mouse deltas are accumulated between sends
- [ ] Mouse movement is throttled to 125 Hz (8 ms)
- [ ] Binary messages are constructed correctly (0x01, dx, dy, buttons)
- [ ] Mouse buttons (left, right, middle) are handled
- [ ] Scroll wheel is handled
- [ ] Mouse capture status is displayed in UI
- [ ] Emergency stop button releases mouse capture
- [ ] No JavaScript errors

## Unit Tests

Create file `test/test_mouse_capture.js`:

```javascript
function test_pointer_lock_request() {
  const canvas = document.getElementById('mouse-capture');
  canvas.click();
  assert(document.pointerLockElement === canvas);
}

function test_mouse_delta_accumulation() {
  const event1 = new MouseEvent('mousemove', { movementX: 10, movementY: 5 });
  const event2 = new MouseEvent('mousemove', { movementX: -3, movementY: 2 });
  canvas.dispatchEvent(event1);
  canvas.dispatchEvent(event2);
  assert(pendingDx === 7);
  assert(pendingDy === 7);
}

function test_throttling() {
  for (let i = 0; i < 100; i++) {
    const event = new MouseEvent('mousemove', { movementX: 1, movementY: 0 });
    canvas.dispatchEvent(event);
  }
  assert(sentCount >= 10 && sentCount <= 15);
}

function test_binary_message_format() {
  sendMouseMove(10, -5, 0x01);
  assert(lastMessage[0] === 0x01);
  assert(lastMessage[1] === 10);
  assert(lastMessage[2] === 251);
  assert(lastMessage[3] === 0x01);
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
git checkout -b feature/feature-004-browser-mouse
```

### Step 2: Add Mouse Capture HTML/JS

In `src/web_portal.cpp`, add to the companion app HTML:

```html
<canvas id="mouse-capture" width="400" height="300"
  style="background:#0b1326;border-radius:12px;cursor:crosshair;width:100%"></canvas>
<div class="muted" id="capture-status">Click to capture mouse</div>

<script>
const canvas = document.getElementById('mouse-capture');
const status = document.getElementById('capture-status');
let pendingDx = 0, pendingDy = 0, lastSend = 0, buttons = 0;

canvas.addEventListener('click', () => canvas.requestPointerLock());
document.addEventListener('pointerlockchange', () => {
  const captured = document.pointerLockElement === canvas;
  status.textContent = captured ? 'Mouse captured — ESC to release' : 'Click to capture mouse';
  canvas.style.borderColor = captured ? '#4fd1c5' : 'transparent';
});

canvas.addEventListener('mousemove', (e) => {
  if (document.pointerLockElement !== canvas) return;
  pendingDx += e.movementX;
  pendingDy += e.movementY;
  const now = performance.now();
  if (now - lastSend >= 8) {
    pendingDx = Math.max(-127, Math.min(127, pendingDx));
    pendingDy = Math.max(-127, Math.min(127, pendingDy));
    ws.send(new Uint8Array([0x01, pendingDx & 0xFF, pendingDy & 0xFF, buttons]));
    pendingDx = 0; pendingDy = 0; lastSend = now;
  }
});

canvas.addEventListener('mousedown', (e) => {
  buttons |= (1 << e.button);
  ws.send(new Uint8Array([0x01, 0, 0, buttons]));
});
canvas.addEventListener('mouseup', (e) => {
  buttons &= ~(1 << e.button);
  ws.send(new Uint8Array([0x01, 0, 0, buttons]));
});
canvas.addEventListener('wheel', (e) => {
  e.preventDefault();
  ws.send(new Uint8Array([0x02, Math.sign(-e.deltaY), 0]));
});
</script>
```

### Step 3: Verify and Commit

```bash
git status
git diff
git add -A
git commit -m "feat(ui): implement browser mouse capture

- Pointer Lock API integration
- Mouse delta accumulation with 125 Hz throttling
- Binary protocol messages (0x01, 0x02)
- Mouse button and scroll wheel handling
- Fixes FEATURE-004"
```

### Step 4: Push and Merge to Dev

```bash
git push origin feature/feature-004-browser-mouse
git checkout dev
git pull origin dev
git merge feature/feature-004-browser-mouse
git push origin dev
```

### Step 5: Reset to Dev

```bash
git checkout dev
git pull origin dev
```
