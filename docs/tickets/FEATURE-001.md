# FEATURE-001: Add WebSocket Server

| Field | Value |
| --- | --- |
| **ID** | FEATURE-001 |
| **Priority** | 🔴 P0 (Critical) |
| **Epic** | Feature Expansion |
| **Estimated Effort** | 3 hours |
| **Dependencies** | None |
| **Branch** | `feature/feature-001-websocket-server` |

---

## Description

The current firmware only supports HTTP polling (500 ms – 2 s latency). Real-time mouse
control requires < 20 ms latency. This ticket adds a WebSocket server for bidirectional
low-latency communication.

## References

> **Related docs for this ticket:** `docs/ARCHITECTURE.md`, `docs/FINDINGS.md`, `docs/STEALTH_HARDENING.md`, `docs/FEATURE_EXPANSION.md`, `docs/TICKETS.md` — read the relevant sections listed above before coding.

- `docs/FEATURE_EXPANSION.md` §2, §3
- `docs/ARCHITECTURE.md` §10 (Web Dashboard)
- `docs/FINDINGS.md` §3.3 (HTTP-Only Networking)

## Files to Modify

| File | Action | Description |
| --- | --- | --- |
| `platformio.ini` | Modify | Add `WebSocketsServer` library dependency |
| `src/web_portal.cpp` | Modify | Add WebSocket server, event handler, integration |

## Objectives

1. Add `WebSocketsServer` library dependency
2. Initialize WebSocket server on port 81
3. Implement WebSocket event handler
4. Integrate WebSocket loop into `webPortalLoop()`
5. Handle connection/disconnection events

## Acceptance Criteria

- [ ] `WebSocketsServer` library is added to `platformio.ini`
- [ ] WebSocket server is initialized on port 81
- [ ] WebSocket event handler is implemented
- [ ] `webSocket.loop()` is called in `webPortalLoop()`
- [ ] Connection events are logged via `eventLogAdd()`
- [ ] Disconnection events are logged via `eventLogAdd()`
- [ ] WebSocket endpoint is at `ws://<ip>:81/ws`
- [ ] No compilation errors or warnings
- [ ] Existing HTTP functionality is not broken

## Unit Tests

Create file `test/test_websocket.cpp`:

```cpp
#include <unity.h>

void test_websocket_server_initialization() {
  // Verify WebSocket server is initialized on port 81
  TEST_ASSERT_TRUE_MESSAGE(true, "WebSocket server initialization verified");
}

void test_websocket_connection() {
  // Verify WebSocket connection is accepted
  TEST_ASSERT_TRUE_MESSAGE(true, "WebSocket connection verified");
}

void test_websocket_message_received() {
  // Verify WebSocket messages are received and parsed
  TEST_ASSERT_TRUE_MESSAGE(true, "WebSocket message reception verified");
}

void setup() {
  UNITY_BEGIN();
  RUN_TEST(test_websocket_server_initialization);
  RUN_TEST(test_websocket_connection);
  RUN_TEST(test_websocket_message_received);
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
git checkout -b feature/feature-001-websocket-server
```

### Step 2: Add Library Dependency

In `platformio.ini`, add:

```ini
lib_deps =
    USB
    links2004/WebSockets @ ^2.4.0
```

### Step 3: Add WebSocket Server to web_portal.cpp

```cpp
#include <WebSocketsServer.h>

static WebSocketsServer webSocket = WebSocketsServer(81);

void webSocketEvent(uint8_t num, WStype_t type, uint8_t* payload, size_t length) {
  switch (type) {
    case WStype_CONNECTED:
      eventLogAdd(String("WebSocket client #") + String(num) + " connected");
      break;
    case WStype_DISCONNECTED:
      eventLogAdd(String("WebSocket client #") + String(num) + " disconnected");
      break;
    case WStype_BIN:
      // Parse binary message (FEATURE-002)
      break;
    case WStype_TEXT:
      // Parse JSON config (optional)
      break;
  }
}
```

### Step 4: Integrate into webPortalSetup() and webPortalLoop()

```cpp
void webPortalSetup() {
  // ... existing setup ...
  webSocket.begin();
  webSocket.onEvent(webSocketEvent);
}

void webPortalLoop() {
  // ... existing loop ...
  webSocket.loop();
}
```

### Step 5: Verify and Commit

```bash
git status
git diff
git add -A
git commit -m "feat(network): add WebSocket server

- Add WebSocketsServer library dependency
- Initialize WebSocket server on port 81
- Implement connection/disconnection event handling
- Integrate into webPortalSetup() and webPortalLoop()
- Fixes FEATURE-001"
```

### Step 6: Push and Merge to Dev

```bash
git push origin feature/feature-001-websocket-server
git checkout dev
git pull origin dev
git merge feature/feature-001-websocket-server
git push origin dev
```

### Step 7: Reset to Dev

```bash
git checkout dev
git pull origin dev
```
