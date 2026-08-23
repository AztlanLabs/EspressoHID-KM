# FEATURE-005: Implement Browser Keyboard Input

| Field | Value |
| --- | --- |
| **ID** | FEATURE-005 |
| **Priority** | 🔴 P0 (Critical) |
| **Epic** | Feature Expansion |
| **Estimated Effort** | 3 hours |
| **Dependencies** | FEATURE-001, FEATURE-002 |
| **Branch** | `feature/feature-005-browser-keyboard` |

---

## Description

The browser UI provides a text input field and virtual keyboard for sending keyboard input
to the device via WebSocket.

## References

> **Related docs for this ticket:** `docs/ARCHITECTURE.md`, `docs/FINDINGS.md`, `docs/STEALTH_HARDENING.md`, `docs/FEATURE_EXPANSION.md`, `docs/TICKETS.md` — read the relevant sections listed above before coding.

- `docs/FEATURE_EXPANSION.md` §4.3, §4.4
- `docs/ARCHITECTURE.md` §10 (Web Dashboard)

## Files to Modify

| File | Action | Description |
| --- | --- | --- |
| `src/web_portal.cpp` | Modify | Add companion app HTML with keyboard input JavaScript |

## Objectives

1. Implement text input field
2. Implement "Send Text" button
3. Implement typing mode selection (humanized, fast, net-zero)
4. Implement binary message construction for text
5. Implement virtual keyboard for mobile
6. Implement quick action buttons (Ctrl+C, Alt+Tab, etc.)

## Acceptance Criteria

- [ ] Text input field is implemented
- [ ] "Send Text" button sends text via WebSocket
- [ ] Typing mode selection is implemented (humanized, fast, net-zero)
- [ ] Binary messages are constructed correctly (0x20, length, mode, flags, data)
- [ ] Virtual keyboard is implemented for mobile
- [ ] Quick action buttons are implemented
- [ ] Emergency stop button is implemented
- [ ] No JavaScript errors

## Unit Tests

Create file `test/test_keyboard_input.js`:

```javascript
function test_text_message_format() {
  sendText("Hello", 0, 0x01);
  assert(lastMessage[0] === 0x20);
  assert(lastMessage[1] === 5);
  assert(lastMessage[2] === 0);
  assert(lastMessage[3] === 0x01);
  const text = new TextDecoder().decode(lastMessage.slice(4));
  assert(text === "Hello");
}

function test_quick_action() {
  sendQuickAction('ctrl+c');
  assert(messages.length === 2);
  assert(messages[0][0] === 0x10);
  assert(messages[1][0] === 0x11);
}

function test_emergency_stop() {
  sendEmergencyStop();
  assert(lastMessage[0] === 0xF0);
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
git checkout -b feature/feature-005-browser-keyboard
```

### Step 2: Add Keyboard Input HTML/JS

In `src/web_portal.cpp`, add to the companion app HTML:

```html
<div class="card">
  <div class="title"><b>Keyboard</b><span class="muted">Remote input</span></div>
  <label>Text</label>
  <input id="text-input" maxlength="128" placeholder="Type text to inject...">
  <div class="row" style="margin-top:8px">
    <div><label style="margin-top:0"><input type="radio" name="type-mode" value="0" checked> Humanized</label></div>
    <div><label style="margin-top:0"><input type="radio" name="type-mode" value="1"> Fast</label></div>
    <div><label style="margin-top:0"><input type="radio" name="type-mode" value="2"> Net-zero</label></div>
  </div>
  <div class="row" style="margin-top:8px">
    <div><button id="send-text">Send Text</button></div>
    <div><button class="secondary" id="clear-text">Clear</button></div>
    <div><button class="danger" id="emergency-stop">Emergency Stop</button></div>
  </div>
</div>

<div class="card">
  <div class="title"><b>Quick Actions</b></div>
  <div style="display:flex;flex-wrap:wrap;gap:6px;margin-top:8px">
    <button class="secondary" onclick="sendQuickAction('ctrl+c')">Ctrl+C</button>
    <button class="secondary" onclick="sendQuickAction('ctrl+v')">Ctrl+V</button>
    <button class="secondary" onclick="sendQuickAction('alt+tab')">Alt+Tab</button>
    <button class="secondary" onclick="sendQuickAction('win')">Win</button>
    <button class="secondary" onclick="sendQuickAction('esc')">Esc</button>
    <button class="secondary" onclick="sendQuickAction('enter')">Enter</button>
  </div>
</div>

<script>
const KEYCODES = {
  'a':0x04,'c':0x06,'v':0x19,'x':0x1B,'s':0x16,
  'enter':0x28,'esc':0x29,'tab':0x2B,'space':0x2C,
  'up':0x52,'down':0x51,'left':0x50,'right':0x4F,
};

function sendText(text, mode, flags) {
  const encoded = new TextEncoder().encode(text);
  ws.send(new Uint8Array([0x20, encoded.length, mode, flags, ...encoded]));
}

function sendKeyPress(keycode, modifiers) {
  ws.send(new Uint8Array([0x10, keycode, modifiers]));
}

function sendKeyRelease(keycode, modifiers) {
  ws.send(new Uint8Array([0x11, keycode, modifiers]));
}

function sendQuickAction(action) {
  const parts = action.split('+');
  let mods = 0;
  let key = parts[parts.length - 1];
  for (const p of parts) {
    if (p === 'ctrl') mods |= 0x01;
    else if (p === 'shift') mods |= 0x02;
    else if (p === 'alt') mods |= 0x04;
    else if (p === 'win') mods |= 0x08;
  }
  const keycode = KEYCODES[key] || 0x04;
  sendKeyPress(keycode, mods);
  setTimeout(() => sendKeyRelease(keycode, mods), 50);
}

function sendEmergencyStop() {
  ws.send(new Uint8Array([0xF0]));
}

document.getElementById('send-text').addEventListener('click', () => {
  const text = document.getElementById('text-input').value;
  const mode = parseInt(document.querySelector('input[name="type-mode"]:checked').value);
  const flags = mode === 0 ? 0x01 : mode === 1 ? 0x04 : 0x03;
  sendText(text, 0, flags);
});

document.getElementById('clear-text').addEventListener('click', () => {
  document.getElementById('text-input').value = '';
});

document.getElementById('emergency-stop').addEventListener('click', sendEmergencyStop);
</script>
```

### Step 3: Verify and Commit

```bash
git status
git diff
git add -A
git commit -m "feat(ui): implement browser keyboard input

- Text input with humanized/fast/net-zero modes
- Quick action buttons (Ctrl+C, Alt+Tab, etc.)
- Emergency stop button
- Binary protocol messages (0x10, 0x11, 0x20, 0xF0)
- Fixes FEATURE-005"
```

### Step 4: Push and Merge to Dev

```bash
git push origin feature/feature-005-browser-keyboard
git checkout dev
git pull origin dev
git merge feature/feature-005-browser-keyboard
git push origin dev
```

### Step 5: Reset to Dev

```bash
git checkout dev
git pull origin dev
```
