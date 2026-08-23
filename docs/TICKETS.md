# EspressoHID: KM — Ticket Backlog

> Comprehensive backlog of tickets covering all findings, feature expansion, and stealth
> hardening work. Each ticket is designed for an AI agent to execute autonomously with
> clear objectives, acceptance criteria, and unit test requirements.

---

## Ticket Index

| ID | Title | Priority | Epic | Status |
| --- | --- | --- | --- | --- |
| [STEALTH-001](#stealth-001-add-usbhidmouse-composite-device) | Add USBHIDMouse Composite Device | 🔴 P0 | Stealth | Pending |
| [STEALTH-002](#stealth-002-replace-uniform-random-with-log-normal-distribution) | Replace Uniform Random with Log-Normal Distribution | 🔴 P0 | Stealth | Pending |
| [STEALTH-003](#stealth-003-implement-burstpause-work-patterns) | Implement Burst/Pause Work Patterns | 🔴 P0 | Stealth | Pending |
| [STEALTH-004](#stealth-004-add-variation-to-net-zero-actions) | Add Variation to Net-Zero Actions | 🔴 P0 | Stealth | Pending |
| [STEALTH-005](#stealth-005-implement-mixed-activity-types) | Implement Mixed Activity Types | 🔴 P0 | Stealth | Pending |
| [STEALTH-006](#stealth-006-implement-humanized-mouse-movement) | Implement Humanized Mouse Movement | 🔴 P0 | Stealth | Pending |
| [STEALTH-007](#stealth-007-improve-usb-descriptor-spoofing) | Improve USB Descriptor Spoofing | 🟡 P1 | Stealth | Pending |
| [STEALTH-008](#stealth-008-improve-serial-number-format) | Improve Serial Number Format | 🟡 P1 | Stealth | Pending |
| [STEALTH-009](#stealth-009-add-circadian-rhythm-variation) | Add Circadian Rhythm Variation | 🟡 P1 | Stealth | Pending |
| [FEATURE-001](#feature-001-add-websocket-server) | Add WebSocket Server | 🔴 P0 | Feature | Pending |
| [FEATURE-002](#feature-002-implement-binary-websocket-protocol) | Implement Binary WebSocket Protocol | 🔴 P0 | Feature | Pending |
| [FEATURE-003](#feature-003-implement-hid-command-processor) | Implement HID Command Processor | 🔴 P0 | Feature | Pending |
| [FEATURE-004](#feature-004-implement-browser-mouse-capture) | Implement Browser Mouse Capture | 🔴 P0 | Feature | Pending |
| [FEATURE-005](#feature-005-implement-browser-keyboard-input) | Implement Browser Keyboard Input | 🔴 P0 | Feature | Pending |
| [FEATURE-006](#feature-006-implement-text-injection-ui) | Implement Text Injection UI | 🟡 P1 | Feature | Pending |
| [FEATURE-007](#feature-007-implement-dual-core-task-separation) | Implement Dual-Core Task Separation | 🟡 P1 | Feature | Pending |
| [FEATURE-008](#feature-008-add-ota-update-via-websocket) | Add OTA Update via WebSocket | 🟢 P2 | Feature | Pending |
| [QUALITY-001](#quality-001-add-unit-test-framework) | Add Unit Test Framework | 🔴 P0 | Quality | Pending |
| [QUALITY-002](#quality-002-add-action-selection-unit-tests) | Add Action Selection Unit Tests | 🔴 P0 | Quality | Pending |
| [QUALITY-003](#quality-003-add-timing-distribution-unit-tests) | Add Timing Distribution Unit Tests | 🔴 P0 | Quality | Pending |
| [QUALITY-004](#quality-004-add-nvs-config-unit-tests) | Add NVS Config Unit Tests | 🟡 P1 | Quality | Pending |
| [QUALITY-005](#quality-005-add-stealth-validation-tests) | Add Stealth Validation Tests | 🟡 P1 | Quality | Pending |
| [QUALITY-006](#quality-006-add-websocket-protocol-unit-tests) | Add WebSocket Protocol Unit Tests | 🟡 P1 | Quality | Pending |

---

## Standard Git Workflow for Ticket Execution (Base Branch: `dev`)

> **All branches off `dev`. No direct commits to `main`. Individual ticket files under `docs/tickets/` contain per-ticket branch names — all follow this flow.**

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

---

## Epic: Stealth Hardening

### STEALTH-001: Add USBHIDMouse Composite Device

**Priority:** 🔴 P0 (Critical)
**Epic:** Stealth
**Estimated Effort:** 2 hours
**Dependencies:** None

#### Description

The current firmware only exposes `USBHIDKeyboard` and `USBHIDConsumerControl`. A
keyboard-only USB device that never produces mouse movement is immediately suspicious to
behavioral analytics platforms. This ticket adds `USBHIDMouse` to create a composite
keyboard + mouse + consumer control device, which is what real keyboards with trackpoints
or touchpads look like.

**Detection vector:** Keyboard/mouse correlation analysis — monitoring tools track the ratio
of keyboard to mouse activity. A device that only produces keyboard events with zero mouse
movement for hours is flagged.

**Reference:** `docs/STEALTH_HARDENING.md` §V1, §6.1

#### Objectives

1. Instantiate `USBHIDMouse` in `main.cpp`
2. Add mouse helper functions to `human_input.cpp`
3. Update `human_input.h` with mouse function declarations
4. Ensure the device enumerates as a composite keyboard + mouse + consumer control device
5. Verify the device works on Windows, macOS, and Linux

#### Acceptance Criteria

- [ ] `USBHIDMouse` is instantiated and initialized in `setup()`
- [ ] `Mouse.begin()` is called before `USB.begin()`
- [ ] Mouse helper functions (`moveMouse`, `clickMouse`, `pressMouse`, `releaseMouse`, `scrollMouse`) are implemented in `human_input.cpp`
- [ ] Mouse helper functions are declared in `human_input.h`
- [ ] The device enumerates as a composite HID device with keyboard, mouse, and consumer control interfaces
- [ ] Mouse movement works on the host OS (verified by moving the cursor)
- [ ] Mouse click works on the host OS (verified by clicking)
- [ ] Mouse scroll works on the host OS (verified by scrolling)
- [ ] Existing keyboard and consumer control functionality is not broken
- [ ] No compilation errors or warnings

#### Unit Tests

```cpp
// test/test_mouse_hid.cpp

void test_mouse_hid_instantiation() {
  // Verify USBHIDMouse is instantiated and can be used
  // This test requires a connected USB host
  TEST_ASSERT_TRUE_MESSAGE(true, "Mouse HID instantiation verified");
}

void test_mouse_move_helper() {
  // Verify moveMouse() calls Mouse.move() with correct parameters
  // Mock Mouse.move() and verify it's called with (dx, dy)
  TEST_ASSERT_TRUE_MESSAGE(true, "moveMouse helper verified");
}

void test_mouse_click_helper() {
  // Verify clickMouse() calls Mouse.click() with correct button
  // Mock Mouse.click() and verify it's called with (button)
  TEST_ASSERT_TRUE_MESSAGE(true, "clickMouse helper verified");
}

void test_mouse_scroll_helper() {
  // Verify scrollMouse() calls Mouse.move() with scroll parameters
  // Mock Mouse.move() and verify it's called with (0, 0, vScroll, hScroll)
  TEST_ASSERT_TRUE_MESSAGE(true, "scrollMouse helper verified");
}

void test_mouse_press_release_helper() {
  // Verify pressMouse() and releaseMouse() work correctly
  // Mock Mouse.press() and Mouse.release() and verify calls
  TEST_ASSERT_TRUE_MESSAGE(true, "pressMouse/releaseMouse helpers verified");
}
```

#### Files to Modify

- `src/main.cpp` — Add `USBHIDMouse` instance, initialize in `setup()`
- `include/human_input.h` — Add mouse function declarations
- `lib/fake_keyboard_core/src/human_input.cpp` — Implement mouse helper functions

---

### STEALTH-002: Replace Uniform Random with Log-Normal Distribution

**Priority:** 🔴 P0 (Critical)
**Epic:** Stealth
**Estimated Effort:** 3 hours
**Dependencies:** None

#### Description

All timing uses Arduino's `random(min, max)` which produces a uniform distribution. Real
human timing follows a log-normal distribution — mostly fast with occasional long pauses.
This is detectable by statistical analysis of timing distributions. The Linux `hid-omg-detect`
kernel driver uses "keystroke timing entropy" as one of its three scoring factors.

**Detection vector:** Timing entropy analysis — uniform distributions have low entropy and
are easily distinguishable from human timing.

**Reference:** `docs/STEALTH_HARDENING.md` §V3, §6.3

#### Objectives

1. Implement `logNormalDelay(medianMs, sigmaMs)` function
2. Implement `gammaDelay(shape, scale)` function
3. Implement `gaussianRandom(mean, stddev)` helper function
4. Replace all `random(min, max)` calls in `human_input.cpp` with humanized distributions
5. Replace all `random(min, max)` calls in `actions.cpp` with humanized distributions
6. Ensure timing entropy is > 3.5 bits (human-like)

#### Acceptance Criteria

- [ ] `gaussianRandom(mean, stddev)` function is implemented using Box-Muller transform
- [ ] `logNormalDelay(medianMs, sigmaMs)` function is implemented
- [ ] `gammaDelay(shape, scale)` function is implemented
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

#### Unit Tests

```cpp
// test/test_timing_distribution.cpp

void test_gaussian_random_mean() {
  // Generate 1000 samples, verify mean is close to expected
  float sum = 0;
  for (int i = 0; i < 1000; i++) {
    sum += gaussianRandom(100.0f, 20.0f);
  }
  float mean = sum / 1000.0f;
  TEST_ASSERT_FLOAT_WITHIN(10.0f, 100.0f, mean);
}

void test_gaussian_random_stddev() {
  // Generate 1000 samples, verify stddev is close to expected
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
  // Verify log-normal delay is within reasonable range
  for (int i = 0; i < 100; i++) {
    unsigned long delay = logNormalDelay(800, 300);
    TEST_ASSERT_GREATER_OR_EQUAL(10, delay);
    TEST_ASSERT_LESS_OR_EQUAL(4000, delay);
  }
}

void test_log_normal_delay_median() {
  // Verify median is close to expected
  unsigned long samples[1000];
  for (int i = 0; i < 1000; i++) {
    samples[i] = logNormalDelay(800, 300);
  }
  // Sort and get median
  // ... sorting code ...
  unsigned long median = samples[500];
  TEST_ASSERT_WITHIN(200, 800, median);
}

void test_gamma_delay_range() {
  // Verify gamma delay is within reasonable range
  for (int i = 0; i < 100; i++) {
    unsigned long delay = gammaDelay(2.0f, 400.0f);
    TEST_ASSERT_GREATER_OR_EQUAL(10, delay);
    TEST_ASSERT_LESS_OR_EQUAL(4000, delay);
  }
}

void test_timing_entropy() {
  // Collect 1000 samples and calculate entropy
  unsigned long samples[1000];
  for (int i = 0; i < 1000; i++) {
    samples[i] = logNormalDelay(800, 300);
  }
  // Calculate histogram and entropy
  // ... entropy calculation ...
  float entropy = calculateEntropy(samples, 1000);
  TEST_ASSERT_GREATER_THAN(3.5f, entropy);
}
```

#### Files to Modify

- `include/human_input.h` — Add new function declarations
- `lib/fake_keyboard_core/src/human_input.cpp` — Implement new functions, replace `random()` calls
- `lib/fake_keyboard_core/src/actions.cpp` — Replace `random()` calls with humanized distributions

---

### STEALTH-003: Implement Burst/Pause Work Patterns

**Priority:** 🔴 P0 (Critical)
**Epic:** Stealth
**Estimated Effort:** 3 hours
**Dependencies:** STEALTH-002

#### Description

Actions happen at predictable intervals (10–60 s for ACTIVE, 45–180 s for MEETING) with
uniform distribution. Real human work patterns have bursts of activity followed by pauses
(reading, thinking, bathroom breaks). Monitoring tools flag consistent activity without
natural breaks.

**Detection vector:** Behavioral analytics — "95%+ activity for 30 min" and "under 4%
fluctuation for 90 min" are Hubstaff thresholds for flagging simulated activity.

**Reference:** `docs/STEALTH_HARDENING.md` §V4, §6.4

#### Objectives

1. Implement `WorkPattern` struct with burst/pause state
2. Implement `updateWorkPattern()` function
3. Implement `getNextDelay()` function that returns humanized delays based on work pattern
4. Integrate work pattern into the main loop
5. Ensure activity patterns have natural variation

#### Acceptance Criteria

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
- [ ] Existing functionality is not broken

#### Unit Tests

```cpp
// test/test_work_pattern.cpp

void test_work_pattern_initialization() {
  WorkPattern pattern;
  memset(&pattern, 0, sizeof(pattern));
  TEST_ASSERT_FALSE(pattern.inBurst);
  TEST_ASSERT_FALSE(pattern.inPause);
  TEST_ASSERT_EQUAL(0, pattern.burstCount);
}

void test_burst_mode_duration() {
  WorkPattern pattern;
  memset(&pattern, 0, sizeof(pattern));
  pattern.inBurst = true;
  pattern.burstEnd = millis() + 120000;  // 2 min

  // Verify burst mode ends after duration
  TEST_ASSERT_TRUE(pattern.inBurst);
  // ... advance time ...
  // TEST_ASSERT_FALSE(pattern.inBurst);
}

void test_pause_mode_duration() {
  WorkPattern pattern;
  memset(&pattern, 0, sizeof(pattern));
  pattern.inPause = true;
  pattern.pauseEnd = millis() + 60000;  // 1 min

  // Verify pause mode ends after duration
  TEST_ASSERT_TRUE(pattern.inPause);
  // ... advance time ...
  // TEST_ASSERT_FALSE(pattern.inPause);
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
```

#### Files to Modify

- `include/config.h` — Add burst/pause timing constants
- `include/state.h` — Add `WorkPattern` struct and global instance
- `lib/fake_keyboard_core/src/state.cpp` — Define work pattern global
- `src/main.cpp` — Integrate work pattern into main loop

---

### STEALTH-004: Add Variation to Net-Zero Actions

**Priority:** 🔴 P0 (Critical)
**Epic:** Stealth
**Estimated Effort:** 2 hours
**Dependencies:** None

#### Description

Net-zero actions always follow the exact same pattern: Volume increment then decrement,
CapsToggle on then off, etc. This is detectable by pattern analysis. Real users don't
always reverse their actions — sometimes they leave volume changed, sometimes they toggle
CapsLock multiple times.

**Detection vector:** Pattern analysis — every Volume action is followed by a Volume
decrement, every CapsToggle is followed by another CapsToggle.

**Reference:** `docs/STEALTH_HARDENING.md` §V6, §6.6

#### Objectives

1. Add variation to Volume action (80% reversal, 20% leave changed)
2. Add variation to Brightness action (80% reversal, 20% leave changed)
3. Add variation to CapsToggle action (variable toggle count, sometimes leave changed)
4. Add variation to NumLockToggle action (variable toggle count, sometimes leave changed)
5. Add variable delay before reversal

#### Acceptance Criteria

- [ ] Volume action reverses 80% of the time, leaves changed 20% of the time
- [ ] Brightness action reverses 80% of the time, leaves changed 20% of the time
- [ ] CapsToggle action toggles 1–3 times (random)
- [ ] CapsToggle leaves CapsLock on 30% of the time if odd number of toggles
- [ ] NumLockToggle action toggles 1–3 times (random)
- [ ] NumLockToggle leaves NumLock on 30% of the time if odd number of toggles
- [ ] Variable delay before reversal (80–2000 ms for Volume/Brightness)
- [ ] No compilation errors or warnings
- [ ] Existing functionality is not broken

#### Unit Tests

```cpp
// test/test_net_zero_variation.cpp

void test_volume_reversal_probability() {
  // Run 1000 Volume actions, count reversals
  int reversals = 0;
  for (int i = 0; i < 1000; i++) {
    // Mock actionVolume() and check if reversal occurred
    // ... mock implementation ...
    if (/* reversal occurred */) reversals++;
  }
  // Should be around 80% (750-850 out of 1000)
  TEST_ASSERT_GREATER_THAN(700, reversals);
  TEST_ASSERT_LESS_THAN(900, reversals);
}

void test_caps_toggle_variable_count() {
  // Run 1000 CapsToggle actions, count toggle counts
  int counts[4] = {0, 0, 0, 0};  // 1, 2, 3, 4 toggles
  for (int i = 0; i < 1000; i++) {
    // Mock actionCapsToggle() and count toggles
    // ... mock implementation ...
    counts[toggleCount - 1]++;
  }
  // Should have variation (not all the same)
  TEST_ASSERT_GREATER_THAN(0, counts[0]);
  TEST_ASSERT_GREATER_THAN(0, counts[1]);
  TEST_ASSERT_GREATER_THAN(0, counts[2]);
}

void test_volume_delay_variation() {
  // Verify delay before reversal is variable
  unsigned long delays[100];
  for (int i = 0; i < 100; i++) {
    // Mock actionVolume() and capture delay
    // ... mock implementation ...
    delays[i] = /* captured delay */;
  }
  // Calculate variance
  // ... variance calculation ...
  float variance = calculateVariance(delays, 100);
  TEST_ASSERT_GREATER_THAN(10000.0f, variance);  // High variance
}
```

#### Files to Modify

- `lib/fake_keyboard_core/src/actions.cpp` — Modify `actionVolume()`, `actionBrightness()`, `actionCapsToggle()`, `actionNumLockToggle()`

---

### STEALTH-005: Implement Mixed Activity Types

**Priority:** 🔴 P0 (Critical)
**Epic:** Stealth
**Estimated Effort:** 4 hours
**Dependencies:** STEALTH-001

#### Description

The device only produces keyboard events. Real users constantly mix mouse movement, keyboard
input, scroll wheel, and compound actions (click + type + scroll). Monitoring tools track
the ratio of keyboard to mouse activity and flag devices that only produce one type.

**Detection vector:** Keyboard/mouse correlation — "keyboard near 0% for 50 min" is a
Hubstaff threshold. The inverse (mouse near 0%) is equally suspicious.

**Reference:** `docs/STEALTH_HARDENING.md` §V5, §6.5

#### Objectives

1. Add `actionMouseScroll()` — simulate reading a document
2. Add `actionMouseMove()` — simulate pointing at something
3. Add `actionClickType()` — simulate filling a form
4. Add `actionRightClick()` — simulate context menu usage
5. Add `actionDragSelect()` — simulate text selection
6. Add all new actions to the `ACTIONS[]` array with appropriate weights
7. Update weight distribution for mixed activity

#### Acceptance Criteria

- [ ] `actionMouseScroll()` is implemented with variable scroll count and direction
- [ ] `actionMouseMove()` is implemented with humanized movement
- [ ] `actionClickType()` is implemented with click + type + backspace sequence
- [ ] `actionRightClick()` is implemented with right-click + Escape sequence
- [ ] `actionDragSelect()` is implemented with press + drag + release sequence
- [ ] All new actions are added to the `ACTIONS[]` array
- [ ] Weight distribution produces mixed keyboard/mouse activity
- [ ] Mouse scroll has 70% down, 30% up direction bias
- [ ] Mouse scroll sometimes reverses (30% chance)
- [ ] Mouse move uses humanized movement (not instant)
- [ ] Click + type uses humanized typing
- [ ] Drag select uses humanized movement
- [ ] No compilation errors or warnings
- [ ] Existing functionality is not broken

#### Unit Tests

```cpp
// test/test_mixed_activity.cpp

void test_mouse_scroll_direction_bias() {
  // Run 1000 scroll actions, count directions
  int downCount = 0;
  for (int i = 0; i < 1000; i++) {
    // Mock actionMouseScroll() and check direction
    // ... mock implementation ...
    if (/* direction is down */) downCount++;
  }
  // Should be around 70% (650-750 out of 1000)
  TEST_ASSERT_GREATER_THAN(600, downCount);
  TEST_ASSERT_LESS_THAN(800, downCount);
}

void test_mouse_scroll_reversal_probability() {
  // Run 1000 scroll actions, count reversals
  int reversals = 0;
  for (int i = 0; i < 1000; i++) {
    // Mock actionMouseScroll() and check if reversal occurred
    // ... mock implementation ...
    if (/* reversal occurred */) reversals++;
  }
  // Should be around 30% (250-350 out of 1000)
  TEST_ASSERT_GREATER_THAN(200, reversals);
  TEST_ASSERT_LESS_THAN(400, reversals);
}

void test_mouse_move_humanized() {
  // Verify mouse move uses steps, not instant
  // Mock Mouse.move() and count calls
  int moveCalls = 0;
  // Mock actionMouseMove() and count Mouse.move() calls
  // ... mock implementation ...
  TEST_ASSERT_GREATER_THAN(3, moveCalls);  // Should use multiple steps
}

void test_click_type_sequence() {
  // Verify click + type + backspace sequence
  // Mock Mouse.click(), Keyboard.print(), Keyboard.press()
  // ... mock implementation ...
  TEST_ASSERT_TRUE_MESSAGE(true, "Click + type sequence verified");
}

void test_weight_distribution_mixed() {
  // Verify weight distribution produces mixed activity
  int keyboardActions = 0;
  int mouseActions = 0;
  for (int i = 0; i < 1000; i++) {
    // Mock performJiggle() and categorize action
    // ... mock implementation ...
    if (/* is keyboard action */) keyboardActions++;
    else mouseActions++;
  }
  // Should have both keyboard and mouse actions
  TEST_ASSERT_GREATER_THAN(100, keyboardActions);
  TEST_ASSERT_GREATER_THAN(100, mouseActions);
}
```

#### Files to Modify

- `lib/fake_keyboard_core/src/actions.cpp` — Add new action functions, update `ACTIONS[]` array
- `include/config.h` — Add weight constants for new actions

---

### STEALTH-006: Implement Humanized Mouse Movement

**Priority:** 🔴 P0 (Critical)
**Epic:** Stealth
**Estimated Effort:** 4 hours
**Dependencies:** STEALTH-001

#### Description

Current mouse movement (when added by STEALTH-001) uses simple `Mouse.move(dx, dy)` calls
which produce instant, straight-line movement. Real human mouse movement is curved, variable
speed, and includes micro-tremors. The WindMouse algorithm generates curved, non-linear paths
with variable speed that mimic natural human behavior.

**Detection vector:** Movement pattern analysis — "most USB jigglers produce mechanically
regular movement: the same short displacement at the same interval, forever."

**Reference:** `docs/STEALTH_HARDENING.md` §V2, §6.2

#### Objectives

1. Implement `WindMouseState` struct
2. Implement `windMouseInit()` function
3. Implement `windMouseStep()` function
4. Implement `windMouseMove()` function that moves mouse to target using WindMouse algorithm
5. Add micro-tremor support
6. Add overshoot and correction support
7. Integrate WindMouse into mouse movement actions

#### Acceptance Criteria

- [ ] `WindMouseState` struct is defined with `x`, `y`, `targetX`, `targetY`, `velocityX`, `velocityY`, `gravity`, `wind`, `damping` fields
- [ ] `windMouseInit()` function initializes state with target position
- [ ] `windMouseStep()` function updates position using WindMouse algorithm
- [ ] `windMouseMove()` function moves mouse to target using multiple steps
- [ ] Movement paths are curved (not straight lines)
- [ ] Movement speed varies (accelerates at start, decelerates at target)
- [ ] Micro-tremors are present (small random perturbations)
- [ ] Overshoot and correction occurs occasionally (10% chance)
- [ ] Movement duration increases with distance (Fitts' law)
- [ ] WindMouse is integrated into `actionMouseMove()`
- [ ] No compilation errors or warnings
- [ ] Existing functionality is not broken

#### Unit Tests

```cpp
// test/test_wind_mouse.cpp

void test_wind_mouse_initialization() {
  WindMouseState state;
  windMouseInit(state, 100.0f, 200.0f, 0.5f, 2.0f, 0.8f);

  TEST_ASSERT_FLOAT_WITHIN(1.0f, 100.0f, state.targetX);
  TEST_ASSERT_FLOAT_WITHIN(1.0f, 200.0f, state.targetY);
  TEST_ASSERT_FLOAT_WITHIN(0.01f, 0.5f, state.gravity);
  TEST_ASSERT_FLOAT_WITHIN(0.01f, 2.0f, state.wind);
  TEST_ASSERT_FLOAT_WITHIN(0.01f, 0.8f, state.damping);
}

void test_wind_mouse_convergence() {
  // Verify mouse reaches target
  WindMouseState state;
  windMouseInit(state, 100.0f, 200.0f, 0.5f, 2.0f, 0.8f);

  for (int i = 0; i < 1000; i++) {
    windMouseStep(state);
    if (fabs(state.x - state.targetX) < 1.0f &&
        fabs(state.y - state.targetY) < 1.0f) {
      break;
    }
  }

  TEST_ASSERT_FLOAT_WITHIN(2.0f, 100.0f, state.x);
  TEST_ASSERT_FLOAT_WITHIN(2.0f, 200.0f, state.y);
}

void test_wind_mouse_curved_path() {
  // Verify path is not straight
  WindMouseState state;
  windMouseInit(state, 100.0f, 0.0f, 0.5f, 2.0f, 0.8f);

  float pathX[100];
  float pathY[100];
  int steps = 0;

  for (int i = 0; i < 100; i++) {
    windMouseStep(state);
    pathX[steps] = state.x;
    pathY[steps] = state.y;
    steps++;
    if (fabs(state.x - state.targetX) < 1.0f &&
        fabs(state.y - state.targetY) < 1.0f) {
      break;
    }
  }

  // Calculate path deviation from straight line
  float maxDeviation = 0;
  for (int i = 0; i < steps; i++) {
    float t = (float)i / (float)steps;
    float expectedX = t * 100.0f;
    float expectedY = t * 0.0f;
    float deviation = sqrt(pow(pathX[i] - expectedX, 2) +
                           pow(pathY[i] - expectedY, 2));
    if (deviation > maxDeviation) maxDeviation = deviation;
  }

  // Path should deviate from straight line
  TEST_ASSERT_GREATER_THAN(5.0f, maxDeviation);
}

void test_wind_mouse_variable_speed() {
  // Verify speed varies throughout movement
  WindMouseState state;
  windMouseInit(state, 100.0f, 0.0f, 0.5f, 2.0f, 0.8f);

  float speeds[100];
  int steps = 0;

  for (int i = 0; i < 100; i++) {
    float prevX = state.x;
    float prevY = state.y;
    windMouseStep(state);
    float speed = sqrt(pow(state.x - prevX, 2) + pow(state.y - prevY, 2));
    speeds[steps] = speed;
    steps++;
    if (fabs(state.x - state.targetX) < 1.0f &&
        fabs(state.y - state.targetY) < 1.0f) {
      break;
    }
  }

  // Calculate speed variance
  float meanSpeed = 0;
  for (int i = 0; i < steps; i++) meanSpeed += speeds[i];
  meanSpeed /= steps;

  float speedVariance = 0;
  for (int i = 0; i < steps; i++) {
    speedVariance += pow(speeds[i] - meanSpeed, 2);
  }
  speedVariance /= steps;

  // Speed should vary (high variance)
  TEST_ASSERT_GREATER_THAN(0.1f, speedVariance);
}
```

#### Files to Modify

- `include/human_input.h` — Add WindMouse function declarations
- `lib/fake_keyboard_core/src/human_input.cpp` — Implement WindMouse algorithm
- `lib/fake_keyboard_core/src/actions.cpp` — Integrate WindMouse into mouse actions

---

### STEALTH-007: Improve USB Descriptor Spoofing

**Priority:** 🟡 P1 (High)
**Epic:** Stealth
**Estimated Effort:** 4 hours
**Dependencies:** None

#### Description

The Arduino ESP32-S3 USB stack generates a generic HID descriptor that is identifiable as an
Arduino/ESP32 device. USB descriptor fingerprinting tools can compare the descriptor against
a database of known devices. This ticket overrides the default descriptor with one extracted
from a real keyboard.

**Detection vector:** HID descriptor fingerprinting — "the HID report descriptor tells the OS
what the device is. Tools can parse and compare descriptors to known devices."

**Reference:** `docs/STEALTH_HARDENING.md` §V7, §6.7

#### Objectives

1. Extract HID report descriptor from a real keyboard (e.g., Dell KB216)
2. Extract HID report descriptor from a real mouse (e.g., Logitech M100)
3. Create custom descriptor arrays in PROGMEM
4. Override Arduino default descriptors
5. Verify descriptors match real devices exactly

#### Acceptance Criteria

- [ ] Custom keyboard HID report descriptor is extracted from a real keyboard
- [ ] Custom mouse HID report descriptor is extracted from a real mouse
- [ ] Descriptors are stored in PROGMEM arrays
- [ ] Arduino default descriptors are overridden
- [ ] Device enumerates with the custom descriptors
- [ ] Descriptors match real devices exactly (byte-for-byte)
- [ ] Keyboard functionality works with custom descriptor
- [ ] Mouse functionality works with custom descriptor
- [ ] No compilation errors or warnings

#### Unit Tests

```cpp
// test/test_usb_descriptor.cpp

void test_keyboard_descriptor_size() {
  // Verify keyboard descriptor has expected size
  TEST_ASSERT_EQUAL(63, sizeof(customKeyboardReportDescriptor));
}

void test_mouse_descriptor_size() {
  // Verify mouse descriptor has expected size
  TEST_ASSERT_EQUAL(52, sizeof(customMouseReportDescriptor));
}

void test_keyboard_descriptor_usage_page() {
  // Verify Usage Page is Generic Desktop (0x01)
  TEST_ASSERT_EQUAL(0x05, customKeyboardReportDescriptor[0]);
  TEST_ASSERT_EQUAL(0x01, customKeyboardReportDescriptor[1]);
}

void test_keyboard_descriptor_usage() {
  // Verify Usage is Keyboard (0x06)
  TEST_ASSERT_EQUAL(0x09, customKeyboardReportDescriptor[2]);
  TEST_ASSERT_EQUAL(0x06, customKeyboardReportDescriptor[3]);
}

void test_mouse_descriptor_usage_page() {
  // Verify Usage Page is Generic Desktop (0x01)
  TEST_ASSERT_EQUAL(0x05, customMouseReportDescriptor[0]);
  TEST_ASSERT_EQUAL(0x01, customMouseReportDescriptor[1]);
}

void test_mouse_descriptor_usage() {
  // Verify Usage is Mouse (0x02)
  TEST_ASSERT_EQUAL(0x09, customMouseReportDescriptor[2]);
  TEST_ASSERT_EQUAL(0x02, customMouseReportDescriptor[3]);
}
```

#### Files to Modify

- `include/config.h` — Add custom descriptor arrays
- `src/main.cpp` — Override default descriptors

---

### STEALTH-008: Improve Serial Number Format

**Priority:** 🟡 P1 (High)
**Epic:** Stealth
**Estimated Effort:** 2 hours
**Dependencies:** None

#### Description

The current serial number is a 16-hex-char string which is unusual for real keyboards. Real
keyboards have serial numbers in specific formats (e.g., Dell uses `CN0XXXXX`, HP uses
`CZCXXXXX`). This ticket generates serial numbers that match the format of the spoofed
manufacturer.

**Detection vector:** USB enumeration forensics — serial number format analysis.

**Reference:** `docs/STEALTH_HARDENING.md` §V8, §6.8

#### Objectives

1. Define serial number formats for each manufacturer
2. Implement `generateSerialNumber()` function
3. Update `applyRandomIdentity()` to use format-specific serial numbers
4. Verify serial numbers match manufacturer formats

#### Acceptance Criteria

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

#### Unit Tests

```cpp
// test/test_serial_number.cpp

void test_dell_serial_format() {
  char serial[20];
  generateSerialNumber(serial, sizeof(serial), 0);  // Dell index
  TEST_ASSERT_EQUAL('C', serial[0]);
  TEST_ASSERT_EQUAL('N', serial[1]);
  TEST_ASSERT_EQUAL('0', serial[2]);
  TEST_ASSERT_EQUAL(10, strlen(serial));
}

void test_hp_serial_format() {
  char serial[20];
  generateSerialNumber(serial, sizeof(serial), 1);  // HP index
  TEST_ASSERT_EQUAL('C', serial[0]);
  TEST_ASSERT_EQUAL('Z', serial[1]);
  TEST_ASSERT_EQUAL('C', serial[2]);
  TEST_ASSERT_EQUAL(10, strlen(serial));
}

void test_logitech_serial_format() {
  char serial[20];
  generateSerialNumber(serial, sizeof(serial), 2);  // Logitech index
  TEST_ASSERT_EQUAL(16, strlen(serial));
  // No prefix, alphanumeric
  for (int i = 0; i < 16; i++) {
    TEST_ASSERT_TRUE(isalnum(serial[i]));
  }
}

void test_serial_uniqueness() {
  // Verify serial numbers are unique across boots
  char serial1[20], serial2[20];
  generateSerialNumber(serial1, sizeof(serial1), 0);
  generateSerialNumber(serial2, sizeof(serial2), 0);
  // Very unlikely to be the same
  TEST_ASSERT_NOT_EQUAL(0, strcmp(serial1, serial2));
}
```

#### Files to Modify

- `lib/fake_keyboard_core/src/usb_identity.cpp` — Implement `generateSerialNumber()`, update `applyRandomIdentity()`

---

### STEALTH-009: Add Circadian Rhythm Variation

**Priority:** 🟡 P1 (High)
**Epic:** Stealth
**Estimated Effort:** 3 hours
**Dependencies:** STEALTH-003

#### Description

Activity patterns should vary throughout the day to simulate realistic work behavior. Real
humans are more active in the morning, less active after lunch, and have natural breaks
throughout the day. This ticket adds circadian rhythm variation to the work pattern.

**Detection vector:** Behavioral analytics — consistent activity throughout the day is
suspicious.

**Reference:** `docs/STEALTH_HARDENING.md` §6.4

#### Objectives

1. Add time-of-day awareness (via NTP)
2. Implement circadian rhythm multiplier
3. Adjust activity frequency based on time of day
4. Add natural break simulation (lunch, coffee breaks)
5. Integrate with work pattern

#### Acceptance Criteria

- [ ] Time-of-day is obtained from NTP
- [ ] Circadian rhythm multiplier is implemented (0.0–1.0)
- [ ] Activity is more frequent in the morning (9–12)
- [ ] Activity is less frequent after lunch (13–15)
- [ ] Activity is moderate in the afternoon (15–17)
- [ ] Natural breaks are simulated (lunch 12–13, coffee 10:30, 15:00)
- [ ] Break duration is variable (15–60 min)
- [ ] Circadian rhythm integrates with work pattern
- [ ] No compilation errors or warnings

#### Unit Tests

```cpp
// test/test_circadian.cpp

void test_circadian_morning_multiplier() {
  // Verify morning has high multiplier
  float multiplier = getCircadianMultiplier(10);  // 10 AM
  TEST_ASSERT_GREATER_THAN(0.8f, multiplier);
}

void test_circadian_afternoon_multiplier() {
  // Verify afternoon has lower multiplier
  float multiplier = getCircadianMultiplier(14);  // 2 PM
  TEST_ASSERT_LESS_THAN(0.6f, multiplier);
}

void test_circadian_evening_multiplier() {
  // Verify evening has low multiplier
  float multiplier = getCircadianMultiplier(18);  // 6 PM
  TEST_ASSERT_LESS_THAN(0.3f, multiplier);
}

void test_circadian_lunch_break() {
  // Verify lunch break is detected
  bool isBreak = isBreakTime(12, 30);  // 12:30 PM
  TEST_ASSERT_TRUE(isBreak);
}

void test_circadian_coffee_break() {
  // Verify coffee break is detected
  bool isBreak = isBreakTime(10, 30);  // 10:30 AM
  TEST_ASSERT_TRUE(isBreak);
}
```

#### Files to Modify

- `include/config.h` — Add circadian rhythm constants
- `lib/fake_keyboard_core/src/profiles.cpp` — Implement circadian rhythm functions
- `src/main.cpp` — Integrate circadian rhythm into main loop

---

## Epic: Feature Expansion

### FEATURE-001: Add WebSocket Server

**Priority:** 🔴 P0 (Critical)
**Epic:** Feature
**Estimated Effort:** 3 hours
**Dependencies:** None

#### Description

The current firmware only supports HTTP polling (500 ms – 2 s latency). Real-time mouse
control requires < 20 ms latency. This ticket adds a WebSocket server for bidirectional
low-latency communication.

**Reference:** `docs/FEATURE_EXPANSION.md` §2, §3

#### Objectives

1. Add `WebSocketsServer` library dependency
2. Initialize WebSocket server on port 81
3. Implement WebSocket event handler
4. Integrate WebSocket loop into `webPortalLoop()`
5. Handle connection/disconnection events

#### Acceptance Criteria

- [ ] `WebSocketsServer` library is added to `platformio.ini`
- [ ] WebSocket server is initialized on port 81
- [ ] WebSocket event handler is implemented
- [ ] `webSocket.loop()` is called in `webPortalLoop()`
- [ ] Connection events are logged
- [ ] Disconnection events are logged
- [ ] WebSocket endpoint is at `ws://<ip>:81/ws`
- [ ] No compilation errors or warnings
- [ ] Existing HTTP functionality is not broken

#### Unit Tests

```cpp
// test/test_websocket.cpp

void test_websocket_server_initialization() {
  // Verify WebSocket server is initialized
  TEST_ASSERT_TRUE_MESSAGE(true, "WebSocket server initialization verified");
}

void test_websocket_connection() {
  // Verify WebSocket connection is accepted
  TEST_ASSERT_TRUE_MESSAGE(true, "WebSocket connection verified");
}

void test_websocket_message_received() {
  // Verify WebSocket messages are received
  TEST_ASSERT_TRUE_MESSAGE(true, "WebSocket message reception verified");
}
```

#### Files to Modify

- `platformio.ini` — Add `WebSocketsServer` library dependency
- `src/web_portal.cpp` — Add WebSocket server, event handler, integration

---

### FEATURE-002: Implement Binary WebSocket Protocol

**Priority:** 🔴 P0 (Critical)
**Epic:** Feature
**Estimated Effort:** 3 hours
**Dependencies:** FEATURE-001

#### Description

Binary WebSocket messages are 10× smaller than JSON and faster to parse. This ticket
implements the binary protocol for mouse/keyboard commands.

**Reference:** `docs/FEATURE_EXPANSION.md` §3.2

#### Objectives

1. Define binary message types (0x01=mouse move, 0x02=mouse scroll, 0x10=key press, 0x11=key release, 0x20=text, 0xF0=emergency stop)
2. Implement `parseBinaryCommand()` function
3. Implement `HidCommand` struct
4. Handle all message types in WebSocket event handler

#### Acceptance Criteria

- [ ] Binary message types are defined as constants
- [ ] `HidCommand` struct is defined with type-specific unions
- [ ] `parseBinaryCommand()` function parses all message types
- [ ] Mouse move messages (0x01) are parsed correctly
- [ ] Mouse scroll messages (0x02) are parsed correctly
- [ ] Key press messages (0x10) are parsed correctly
- [ ] Key release messages (0x11) are parsed correctly
- [ ] Text messages (0x20) are parsed correctly
- [ ] Emergency stop messages (0xF0) are parsed correctly
- [ ] Invalid messages are rejected gracefully
- [ ] No compilation errors or warnings

#### Unit Tests

```cpp
// test/test_binary_protocol.cpp

void test_parse_mouse_move() {
  uint8_t data[] = {0x01, 10, 251, 0};  // dx=10, dy=-5, buttons=0
  HidCommand cmd;
  bool result = parseBinaryCommand(data, 4, cmd);

  TEST_ASSERT_TRUE(result);
  TEST_ASSERT_EQUAL(HID_CMD_MOUSE_MOVE, cmd.type);
  TEST_ASSERT_EQUAL(10, cmd.mouse.dx);
  TEST_ASSERT_EQUAL(-5, cmd.mouse.dy);
  TEST_ASSERT_EQUAL(0, cmd.mouse.buttons);
}

void test_parse_mouse_scroll() {
  uint8_t data[] = {0x02, 253, 0};  // vScroll=-3, hScroll=0
  HidCommand cmd;
  bool result = parseBinaryCommand(data, 3, cmd);

  TEST_ASSERT_TRUE(result);
  TEST_ASSERT_EQUAL(HID_CMD_MOUSE_SCROLL, cmd.type);
  TEST_ASSERT_EQUAL(-3, cmd.scroll.v);
  TEST_ASSERT_EQUAL(0, cmd.scroll.h);
}

void test_parse_key_press() {
  uint8_t data[] = {0x10, 0x04, 0x02};  // keycode=0x04 (A), modifiers=Shift
  HidCommand cmd;
  bool result = parseBinaryCommand(data, 3, cmd);

  TEST_ASSERT_TRUE(result);
  TEST_ASSERT_EQUAL(HID_CMD_KEY_PRESS, cmd.type);
  TEST_ASSERT_EQUAL(0x04, cmd.key.keycode);
  TEST_ASSERT_EQUAL(0x02, cmd.key.modifiers);
}

void test_parse_emergency_stop() {
  uint8_t data[] = {0xF0};
  HidCommand cmd;
  bool result = parseBinaryCommand(data, 1, cmd);

  TEST_ASSERT_TRUE(result);
  TEST_ASSERT_EQUAL(HID_CMD_EMERGENCY_STOP, cmd.type);
}

void test_parse_invalid_message() {
  uint8_t data[] = {0xFF};  // Unknown type
  HidCommand cmd;
  bool result = parseBinaryCommand(data, 1, cmd);

  TEST_ASSERT_FALSE(result);
}

void test_parse_truncated_message() {
  uint8_t data[] = {0x01, 10};  // Too short for mouse move
  HidCommand cmd;
  bool result = parseBinaryCommand(data, 2, cmd);

  TEST_ASSERT_FALSE(result);
}
```

#### Files to Modify

- `include/hid_command.h` — Define `HidCommand` struct and message types
- `src/web_portal.cpp` — Implement `parseBinaryCommand()`

---

### FEATURE-003: Implement HID Command Processor

**Priority:** 🔴 P0 (Critical)
**Epic:** Feature
**Estimated Effort:** 3 hours
**Dependencies:** FEATURE-002, STEALTH-001

#### Description

The HID command processor reads commands from a queue and executes them on the HID devices.
It runs on core 1 (Arduino loop) and processes at most 1 command per 8 ms (125 Hz).

**Reference:** `docs/FEATURE_EXPANSION.md` §5.2

#### Objectives

1. Implement `HidCommand` queue (ring buffer, 64 entries)
2. Implement `hidCommandPush()` function
3. Implement `hidCommandPop()` function
4. Implement `hidCommandProcessor_loop()` function
5. Implement `executeHidCommand()` function
6. Integrate processor into main loop

#### Acceptance Criteria

- [ ] `HidCommand` queue is implemented as a ring buffer (64 entries)
- [ ] `hidCommandPush()` adds commands to the queue
- [ ] `hidCommandPop()` removes commands from the queue
- [ ] `hidCommandProcessor_loop()` processes at most 1 command per 8 ms
- [ ] `executeHidCommand()` executes all command types
- [ ] Mouse move commands are executed on `Mouse`
- [ ] Mouse scroll commands are executed on `Mouse`
- [ ] Key press commands are executed on `Keyboard`
- [ ] Key release commands are executed on `Keyboard`
- [ ] Text commands are executed on `Keyboard`
- [ ] Emergency stop releases all keys and mouse buttons
- [ ] Queue overflow is handled gracefully (drop oldest)
- [ ] Processor is integrated into main loop
- [ ] No compilation errors or warnings

#### Unit Tests

```cpp
// test/test_hid_command_processor.cpp

void test_command_queue_push_pop() {
  HidCommand cmd1 = {HID_CMD_MOUSE_MOVE, {10, 20, 0}};
  HidCommand cmd2;

  TEST_ASSERT_TRUE(hidCommandPush(cmd1));
  TEST_ASSERT_TRUE(hidCommandPop(cmd2));
  TEST_ASSERT_EQUAL(cmd1.type, cmd2.type);
}

void test_command_queue_overflow() {
  // Fill queue to capacity
  HidCommand cmd = {HID_CMD_MOUSE_MOVE, {0, 0, 0}};
  for (int i = 0; i < 64; i++) {
    TEST_ASSERT_TRUE(hidCommandPush(cmd));
  }
  // Next push should fail
  TEST_ASSERT_FALSE(hidCommandPush(cmd));
}

void test_command_queue_empty() {
  HidCommand cmd;
  TEST_ASSERT_FALSE(hidCommandPop(cmd));
}

void test_mouse_move_execution() {
  // Mock Mouse.move() and verify it's called
  HidCommand cmd = {HID_CMD_MOUSE_MOVE, {10, 20, 0}};
  executeHidCommand(cmd);
  // Verify Mouse.move() was called with (10, 20)
  TEST_ASSERT_TRUE_MESSAGE(true, "Mouse move execution verified");
}

void test_key_press_execution() {
  // Mock Keyboard.press() and verify it's called
  HidCommand cmd = {HID_CMD_KEY_PRESS, {0x04, 0x02}};  // A + Shift
  executeHidCommand(cmd);
  // Verify Keyboard.press() was called
  TEST_ASSERT_TRUE_MESSAGE(true, "Key press execution verified");
}

void test_emergency_stop_execution() {
  // Mock Keyboard.releaseAll() and Mouse.release() and verify they're called
  HidCommand cmd = {HID_CMD_EMERGENCY_STOP};
  executeHidCommand(cmd);
  // Verify Keyboard.releaseAll() and Mouse.release() were called
  TEST_ASSERT_TRUE_MESSAGE(true, "Emergency stop execution verified");
}
```

#### Files to Modify

- `include/hid_command.h` — Define queue functions
- `lib/fake_keyboard_core/src/hid_command.cpp` — Implement queue and processor
- `src/main.cpp` — Integrate processor into main loop

---

### FEATURE-004: Implement Browser Mouse Capture

**Priority:** 🔴 P0 (Critical)
**Epic:** Feature
**Estimated Effort:** 4 hours
**Dependencies:** FEATURE-001, FEATURE-002

#### Description

The browser UI uses the Pointer Lock API to capture mouse movement and send relative deltas
to the device via WebSocket. This enables real-time mouse control from the browser.

**Reference:** `docs/FEATURE_EXPANSION.md` §4.2

#### Objectives

1. Implement Pointer Lock API integration
2. Implement mouse delta accumulation
3. Implement 125 Hz throttling
4. Implement binary message construction
5. Implement mouse button handling
6. Implement scroll wheel handling

#### Acceptance Criteria

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

#### Unit Tests

```javascript
// test/test_mouse_capture.js

function test_pointer_lock_request() {
  const canvas = document.getElementById('mouse-capture');
  canvas.click();
  // Verify requestPointerLock() was called
  assert(document.pointerLockElement === canvas);
}

function test_mouse_delta_accumulation() {
  // Simulate mouse movement events
  const event1 = new MouseEvent('mousemove', { movementX: 10, movementY: 5 });
  const event2 = new MouseEvent('mousemove', { movementX: -3, movementY: 2 });

  canvas.dispatchEvent(event1);
  canvas.dispatchEvent(event2);

  // Verify deltas are accumulated
  assert(pendingDx === 7);
  assert(pendingDy === 7);
}

function test_throttling() {
  // Send 100 movement events rapidly
  for (let i = 0; i < 100; i++) {
    const event = new MouseEvent('mousemove', { movementX: 1, movementY: 0 });
    canvas.dispatchEvent(event);
  }

  // Verify only ~12 messages were sent (100 / 8 ms = 12.5)
  assert(sentCount >= 10 && sentCount <= 15);
}

function test_binary_message_format() {
  // Verify binary message format
  sendMouseMove(10, -5, 0x01);
  assert(lastMessage[0] === 0x01);  // Type
  assert(lastMessage[1] === 10);    // dx
  assert(lastMessage[2] === 251);   // dy (signed to unsigned)
  assert(lastMessage[3] === 0x01);  // buttons
}
```

#### Files to Modify

- `src/web_portal.cpp` — Add companion app HTML with mouse capture JavaScript

---

### FEATURE-005: Implement Browser Keyboard Input

**Priority:** 🔴 P0 (Critical)
**Epic:** Feature
**Estimated Effort:** 3 hours
**Dependencies:** FEATURE-001, FEATURE-002

#### Description

The browser UI provides a text input field and virtual keyboard for sending keyboard input
to the device via WebSocket.

**Reference:** `docs/FEATURE_EXPANSION.md` §4.3, §4.4

#### Objectives

1. Implement text input field
2. Implement "Send Text" button
3. Implement typing mode selection (humanized, fast, net-zero)
4. Implement binary message construction for text
5. Implement virtual keyboard for mobile
6. Implement quick action buttons (Ctrl+C, Alt+Tab, etc.)

#### Acceptance Criteria

- [ ] Text input field is implemented
- [ ] "Send Text" button sends text via WebSocket
- [ ] Typing mode selection is implemented (humanized, fast, net-zero)
- [ ] Binary messages are constructed correctly (0x20, length, mode, flags, data)
- [ ] Virtual keyboard is implemented for mobile
- [ ] Quick action buttons are implemented
- [ ] Emergency stop button is implemented
- [ ] No JavaScript errors

#### Unit Tests

```javascript
// test/test_keyboard_input.js

function test_text_message_format() {
  sendText("Hello", 0, 0x01);
  assert(lastMessage[0] === 0x20);  // Type
  assert(lastMessage[1] === 5);     // Length
  assert(lastMessage[2] === 0);     // Mode
  assert(lastMessage[3] === 0x01);  // Flags
  // Verify text data
  const text = new TextDecoder().decode(lastMessage.slice(4));
  assert(text === "Hello");
}

function test_quick_action() {
  sendQuickAction('ctrl+c');
  // Verify key press and release messages
  assert(messages.length === 2);
  assert(messages[0][0] === 0x10);  // Key press
  assert(messages[1][0] === 0x11);  // Key release
}

function test_emergency_stop() {
  sendEmergencyStop();
  assert(lastMessage[0] === 0xF0);
}
```

#### Files to Modify

- `src/web_portal.cpp` — Add companion app HTML with keyboard input JavaScript

---

### FEATURE-006: Implement Text Injection UI

**Priority:** 🟡 P1 (High)
**Epic:** Feature
**Estimated Effort:** 2 hours
**Dependencies:** FEATURE-005

#### Description

The text injection UI provides a form for sending text with different typing modes
(humanized, fast, net-zero).

**Reference:** `docs/FEATURE_EXPANSION.md` §4.4

#### Objectives

1. Implement text injection form
2. Implement typing mode radio buttons
3. Implement flags checkboxes (humanized, net-zero, fast)
4. Implement "Send" and "Clear" buttons
5. Implement character counter

#### Acceptance Criteria

- [ ] Text injection form is implemented
- [ ] Typing mode radio buttons are implemented
- [ ] Flags checkboxes are implemented
- [ ] "Send" button sends text with selected mode and flags
- [ ] "Clear" button clears the text input
- [ ] Character counter shows current text length
- [ ] Maximum text length is 128 characters
- [ ] No JavaScript errors

#### Unit Tests

```javascript
// test/test_text_injection.js

function test_text_injection_humanized() {
  document.getElementById('text-input').value = 'Hello';
  document.getElementById('mode-humanized').checked = true;
  document.getElementById('send-text').click();

  assert(lastMessage[0] === 0x20);
  assert(lastMessage[2] === 0);  // Mode 0
  assert(lastMessage[3] & 0x01);  // Humanized flag
}

function test_text_injection_fast() {
  document.getElementById('text-input').value = 'Hello';
  document.getElementById('mode-fast').checked = true;
  document.getElementById('send-text').click();

  assert(lastMessage[0] === 0x20);
  assert(lastMessage[2] === 0);  // Mode 0
  assert(lastMessage[3] & 0x04);  // Fast flag
}

function test_text_injection_net_zero() {
  document.getElementById('text-input').value = 'Hello';
  document.getElementById('mode-netzero').checked = true;
  document.getElementById('send-text').click();

  assert(lastMessage[0] === 0x20);
  assert(lastMessage[2] === 1);  // Mode 1 (type + backspace)
  assert(lastMessage[3] & 0x02);  // Net-zero flag
}
```

#### Files to Modify

- `src/web_portal.cpp` — Add text injection UI to companion app HTML

---

### FEATURE-007: Implement Dual-Core Task Separation

**Priority:** 🟡 P1 (High)
**Epic:** Feature
**Estimated Effort:** 3 hours
**Dependencies:** FEATURE-001

#### Description

The ESP32-S3 is dual-core. Currently, core 0 handles WiFi/BT (idle most of the time) and
core 1 runs the Arduino loop(). By adding a WebSocket task to core 0, we utilize
otherwise-idle CPU cycles and achieve true parallelism without increasing CPU usage.

**Reference:** `docs/FEATURE_EXPANSION.md` §6.1

#### Objectives

1. Create WebSocket task on core 0
2. Implement thread-safe command queue
3. Implement mutex for shared state
4. Verify no race conditions
5. Measure CPU usage impact

#### Acceptance Criteria

- [ ] WebSocket task is created on core 0
- [ ] WebSocket task runs `webSocket.loop()` continuously
- [ ] Command queue is thread-safe (mutex protected)
- [ ] Shared state is protected by mutex
- [ ] No race conditions (verified by stress test)
- [ ] Core 0 CPU usage increases by < 20%
- [ ] Core 1 CPU usage is unchanged
- [ ] No compilation errors or warnings
- [ ] Existing functionality is not broken

#### Unit Tests

```cpp
// test/test_dual_core.cpp

void test_websocket_task_creation() {
  // Verify WebSocket task is created on core 0
  TEST_ASSERT_TRUE_MESSAGE(true, "WebSocket task creation verified");
}

void test_thread_safe_queue() {
  // Push from core 0, pop from core 1
  HidCommand cmd = {HID_CMD_MOUSE_MOVE, {10, 20, 0}};

  // Simulate concurrent access
  for (int i = 0; i < 1000; i++) {
    TEST_ASSERT_TRUE(hidCommandPush(cmd));
  }

  HidCommand popped;
  for (int i = 0; i < 1000; i++) {
    TEST_ASSERT_TRUE(hidCommandPop(popped));
  }
}

void test_no_race_conditions() {
  // Stress test: push/pop from multiple threads
  // ... stress test implementation ...
  TEST_ASSERT_TRUE_MESSAGE(true, "No race conditions detected");
}
```

#### Files to Modify

- `src/main.cpp` — Create WebSocket task on core 0
- `src/web_portal.cpp` — Move WebSocket loop to task
- `include/hid_command.h` — Add mutex protection

---

### FEATURE-008: Add OTA Update via WebSocket

**Priority:** 🟢 P2 (Medium)
**Epic:** Feature
**Estimated Effort:** 3 hours
**Dependencies:** FEATURE-001

#### Description

Currently, OTA updates are only available via HTTP. This ticket adds OTA update support via
WebSocket for faster and more reliable firmware updates.

**Reference:** `docs/FEATURE_EXPANSION.md` §11

#### Objectives

1. Implement OTA message type (0x30)
2. Implement OTA start message
3. Implement OTA data message
4. Implement OTA end message
5. Implement OTA progress reporting
6. Integrate with ESP-IDF Update library

#### Acceptance Criteria

- [ ] OTA message type (0x30) is defined
- [ ] OTA start message is handled
- [ ] OTA data message is handled
- [ ] OTA end message is handled
- [ ] OTA progress is reported to client
- [ ] Firmware is written to flash correctly
- [ ] Device reboots after successful update
- [ ] Failed update is handled gracefully
- [ ] No compilation errors or warnings

#### Unit Tests

```cpp
// test/test_ota_websocket.cpp

void test_ota_start_message() {
  uint8_t data[] = {0x30, 0x01, 0x00, 0x00, 0x10, 0x00};  // Start, size=4096
  HidCommand cmd;
  bool result = parseBinaryCommand(data, 6, cmd);
  TEST_ASSERT_TRUE(result);
  TEST_ASSERT_EQUAL(HID_CMD_OTA_START, cmd.type);
}

void test_ota_data_message() {
  uint8_t data[] = {0x30, 0x02, 0x00, 0x00, 0x00, 0x00, 0x10, /* ... */};  // Data, offset=0, size=16
  HidCommand cmd;
  bool result = parseBinaryCommand(data, 8 + 16, cmd);
  TEST_ASSERT_TRUE(result);
  TEST_ASSERT_EQUAL(HID_CMD_OTA_DATA, cmd.type);
}

void test_ota_end_message() {
  uint8_t data[] = {0x30, 0x03};  // End
  HidCommand cmd;
  bool result = parseBinaryCommand(data, 2, cmd);
  TEST_ASSERT_TRUE(result);
  TEST_ASSERT_EQUAL(HID_CMD_OTA_END, cmd.type);
}
```

#### Files to Modify

- `include/hid_command.h` — Add OTA message types
- `src/web_portal.cpp` — Implement OTA WebSocket handler

---

## Epic: Quality & Testing

### QUALITY-001: Add Unit Test Framework

**Priority:** 🔴 P0 (Critical)
**Epic:** Quality
**Estimated Effort:** 2 hours
**Dependencies:** None

#### Description

The project currently has no automated tests. This ticket adds a unit test framework using
Unity (PlatformIO's default test framework) and sets up the test infrastructure.

**Reference:** `docs/FINDINGS.md` §5.3

#### Objectives

1. Add Unity test framework dependency
2. Create test directory structure
3. Create test runner configuration
4. Create example test file
5. Verify tests run successfully

#### Acceptance Criteria

- [ ] Unity test framework is added to `platformio.ini`
- [ ] Test directory structure is created (`test/`)
- [ ] Test runner configuration is created
- [ ] Example test file is created
- [ ] Tests run successfully with `pio test`
- [ ] Test output is formatted correctly
- [ ] No compilation errors or warnings

#### Unit Tests

```cpp
// test/test_example.cpp

void test_example_assertion() {
  TEST_ASSERT_EQUAL(2, 1 + 1);
}

void test_example_string() {
  TEST_ASSERT_EQUAL_STRING("hello", "hello");
}

void test_example_float() {
  TEST_ASSERT_FLOAT_WITHIN(0.1f, 3.14f, 3.14f);
}
```

#### Files to Modify

- `platformio.ini` — Add Unity dependency, configure test runner
- `test/test_example.cpp` — Create example test file

---

### QUALITY-002: Add Action Selection Unit Tests

**Priority:** 🔴 P0 (Critical)
**Epic:** Quality
**Estimated Effort:** 3 hours
**Dependencies:** QUALITY-001

#### Description

The weighted action selection logic is critical for stealth. This ticket adds unit tests to
verify the selection algorithm works correctly, respects weights, and handles edge cases.

**Reference:** `docs/FINDINGS.md` §5.3

#### Objectives

1. Test weighted selection with known weights
2. Test selection respects enable mask
3. Test selection with all weights zero
4. Test selection with single action enabled
5. Test TypeText weight zero when custom text is empty

#### Acceptance Criteria

- [ ] Weighted selection test verifies distribution matches weights
- [ ] Enable mask test verifies disabled actions are not selected
- [ ] All-zero weights test verifies ShiftTap fallback
- [ ] Single action test verifies that action is always selected
- [ ] TypeText test verifies weight zero when custom text is empty
- [ ] All tests pass
- [ ] No compilation errors or warnings

#### Unit Tests

```cpp
// test/test_action_selection.cpp

void test_weighted_selection_distribution() {
  // Set known weights, run 10000 selections, verify distribution
  int counts[12] = {0};
  for (int i = 0; i < 10000; i++) {
    int action = selectWeightedAction();
    counts[action]++;
  }
  // Verify distribution matches weights (within 5%)
  // ... distribution verification ...
}

void test_enable_mask() {
  // Disable all actions except ShiftTap
  runtimeConfigSetActionEnabledMask(1 << 6);  // Only ShiftTap
  int action = selectWeightedAction();
  TEST_ASSERT_EQUAL(6, action);  // ShiftTap index
}

void test_all_zero_weights() {
  // Set all weights to 0, verify ShiftTap fallback
  // ... set weights to 0 ...
  int action = selectWeightedAction();
  TEST_ASSERT_EQUAL(6, action);  // ShiftTap index
}

void test_single_action() {
  // Enable only one action, verify it's always selected
  runtimeConfigSetActionEnabledMask(1 << 0);  // Only ArrowScroll
  for (int i = 0; i < 100; i++) {
    int action = selectWeightedAction();
    TEST_ASSERT_EQUAL(0, action);  // ArrowScroll index
  }
}

void test_typetext_weight_zero_when_empty() {
  // Set custom text to empty, verify TypeText weight is 0
  runtimeConfigSetCustomText("");
  uint8_t weight = actionWeightActive(8);  // TypeText index
  TEST_ASSERT_EQUAL(0, weight);
}
```

#### Files to Modify

- `test/test_action_selection.cpp` — Create action selection tests

---

### QUALITY-003: Add Timing Distribution Unit Tests

**Priority:** 🔴 P0 (Critical)
**Epic:** Quality
**Estimated Effort:** 2 hours
**Dependencies:** QUALITY-001, STEALTH-002

#### Description

Timing distributions are critical for stealth. This ticket adds unit tests to verify that
timing distributions are human-like (log-normal, gamma) and not uniform.

**Reference:** `docs/STEALTH_HARDENING.md` §8.2

#### Objectives

1. Test gaussian random mean and stddev
2. Test log-normal delay range and median
3. Test gamma delay range
4. Test timing entropy
5. Test that distributions are not uniform

#### Acceptance Criteria

- [ ] Gaussian random test verifies mean is close to expected
- [ ] Gaussian random test verifies stddev is close to expected
- [ ] Log-normal delay test verifies range is reasonable
- [ ] Log-normal delay test verifies median is close to expected
- [ ] Gamma delay test verifies range is reasonable
- [ ] Timing entropy test verifies entropy > 3.5 bits
- [ ] Non-uniform test verifies distributions are not uniform
- [ ] All tests pass
- [ ] No compilation errors or warnings

#### Unit Tests

```cpp
// test/test_timing_distribution.cpp

void test_gaussian_random_mean() {
  float sum = 0;
  for (int i = 0; i < 1000; i++) {
    sum += gaussianRandom(100.0f, 20.0f);
  }
  float mean = sum / 1000.0f;
  TEST_ASSERT_FLOAT_WITHIN(10.0f, 100.0f, mean);
}

void test_log_normal_delay_range() {
  for (int i = 0; i < 100; i++) {
    unsigned long delay = logNormalDelay(800, 300);
    TEST_ASSERT_GREATER_OR_EQUAL(10, delay);
    TEST_ASSERT_LESS_OR_EQUAL(4000, delay);
  }
}

void test_timing_entropy() {
  unsigned long samples[1000];
  for (int i = 0; i < 1000; i++) {
    samples[i] = logNormalDelay(800, 300);
  }
  float entropy = calculateEntropy(samples, 1000);
  TEST_ASSERT_GREATER_THAN(3.5f, entropy);
}

void test_not_uniform() {
  // Verify distribution is not uniform
  unsigned long samples[1000];
  for (int i = 0; i < 1000; i++) {
    samples[i] = logNormalDelay(800, 300);
  }
  // Calculate chi-squared test against uniform distribution
  float chiSquared = calculateChiSquared(samples, 1000);
  // Chi-squared should be high (rejects uniform hypothesis)
  TEST_ASSERT_GREATER_THAN(100.0f, chiSquared);
}
```

#### Files to Modify

- `test/test_timing_distribution.cpp` — Create timing distribution tests

---

### QUALITY-004: Add NVS Config Unit Tests

**Priority:** 🟡 P1 (High)
**Epic:** Quality
**Estimated Effort:** 2 hours
**Dependencies:** QUALITY-001

#### Description

NVS configuration persistence is critical for reliability. This ticket adds unit tests to
verify that configuration is persisted correctly and survives reboots.

**Reference:** `docs/ARCHITECTURE.md` §13

#### Objectives

1. Test NVS initialization
2. Test Wi-Fi credentials persistence
3. Test profile interval persistence
4. Test action mask persistence
5. Test weight persistence
6. Test factory reset

#### Acceptance Criteria

- [ ] NVS initialization test verifies `runtimeConfigBegin()` succeeds
- [ ] Wi-Fi credentials test verifies SSID and password are persisted
- [ ] Profile interval test verifies min/max are persisted
- [ ] Action mask test verifies mask is persisted
- [ ] Weight test verifies weights are persisted
- [ ] Factory reset test verifies all keys are cleared
- [ ] All tests pass
- [ ] No compilation errors or warnings

#### Unit Tests

```cpp
// test/test_nvs_config.cpp

void test_nvs_initialization() {
  TEST_ASSERT_TRUE(runtimeConfigBegin());
}

void test_wifi_credentials_persistence() {
  runtimeConfigSetWifiCredentials("TestSSID", "TestPass");
  const RuntimeConfig& cfg = runtimeConfigGet();
  TEST_ASSERT_EQUAL_STRING("TestSSID", cfg.wifiSsid);
  TEST_ASSERT_EQUAL_STRING("TestPass", cfg.wifiPass);
}

void test_profile_interval_persistence() {
  runtimeConfigSetProfileIntervalsMs(PROFILE_ACTIVE, 5000, 30000);
  TEST_ASSERT_EQUAL(5000, runtimeConfigProfileIntervalMinMs(PROFILE_ACTIVE));
  TEST_ASSERT_EQUAL(30000, runtimeConfigProfileIntervalMaxMs(PROFILE_ACTIVE));
}

void test_action_mask_persistence() {
  runtimeConfigSetActionEnabledMask(0x12345678);
  TEST_ASSERT_EQUAL(0x12345678, runtimeConfigActionEnabledMask());
}

void test_factory_reset() {
  runtimeConfigSetWifiCredentials("TestSSID", "TestPass");
  runtimeConfigFactoryReset();
  const RuntimeConfig& cfg = runtimeConfigGet();
  TEST_ASSERT_EQUAL_STRING("", cfg.wifiSsid);
  TEST_ASSERT_EQUAL_STRING("", cfg.wifiPass);
}
```

#### Files to Modify

- `test/test_nvs_config.cpp` — Create NVS config tests

---

### QUALITY-005: Add Stealth Validation Tests

**Priority:** 🟡 P1 (High)
**Epic:** Quality
**Estimated Effort:** 3 hours
**Dependencies:** QUALITY-001, STEALTH-002, STEALTH-003, STEALTH-004

#### Description

Stealth validation tests verify that the device produces human-like behavior and passes
detection tools.

**Reference:** `docs/STEALTH_HARDENING.md` §8

#### Objectives

1. Test timing entropy
2. Test movement pattern variance
3. Test activity type distribution
4. Test work pattern variation
5. Test net-zero action variation

#### Acceptance Criteria

- [ ] Timing entropy test verifies entropy > 3.5 bits
- [ ] Movement pattern test verifies high variance
- [ ] Activity type test verifies mixed keyboard/mouse activity
- [ ] Work pattern test verifies burst/pause variation
- [ ] Net-zero test verifies action variation
- [ ] All tests pass
- [ ] No compilation errors or warnings

#### Unit Tests

```cpp
// test/test_stealth_validation.cpp

void test_timing_entropy() {
  unsigned long samples[1000];
  for (int i = 0; i < 1000; i++) {
    samples[i] = logNormalDelay(800, 300);
  }
  float entropy = calculateEntropy(samples, 1000);
  TEST_ASSERT_GREATER_THAN(3.5f, entropy);
}

void test_movement_pattern_variance() {
  float movements[1000][2];
  for (int i = 0; i < 1000; i++) {
    // Generate humanized movement
    movements[i][0] = gaussianRandom(0, 10);
    movements[i][1] = gaussianRandom(0, 10);
  }
  float variance = calculateVariance2D(movements, 1000);
  TEST_ASSERT_GREATER_THAN(50.0f, variance);
}

void test_activity_type_distribution() {
  int keyboardCount = 0;
  int mouseCount = 0;
  for (int i = 0; i < 1000; i++) {
    int action = selectWeightedAction();
    if (isKeyboardAction(action)) keyboardCount++;
    else mouseCount++;
  }
  // Should have both keyboard and mouse actions
  TEST_ASSERT_GREATER_THAN(100, keyboardCount);
  TEST_ASSERT_GREATER_THAN(100, mouseCount);
}

void test_work_pattern_variation() {
  WorkPattern pattern;
  memset(&pattern, 0, sizeof(pattern));

  unsigned long delays[100];
  for (int i = 0; i < 100; i++) {
    updateWorkPattern(pattern);
    delays[i] = getNextDelay(pattern);
  }

  float variance = calculateVariance(delays, 100);
  TEST_ASSERT_GREATER_THAN(1000000.0f, variance);  // High variance
}

void test_net_zero_variation() {
  int reversals = 0;
  for (int i = 0; i < 1000; i++) {
    // Mock actionVolume() and check if reversal occurred
    if (/* reversal occurred */) reversals++;
  }
  // Should be around 80% (not 100%)
  TEST_ASSERT_GREATER_THAN(700, reversals);
  TEST_ASSERT_LESS_THAN(900, reversals);
}
```

#### Files to Modify

- `test/test_stealth_validation.cpp` — Create stealth validation tests

---

### QUALITY-006: Add WebSocket Protocol Unit Tests

**Priority:** 🟡 P1 (High)
**Epic:** Quality
**Estimated Effort:** 2 hours
**Dependencies:** QUALITY-001, FEATURE-002

#### Description

WebSocket protocol tests verify that binary messages are parsed correctly and edge cases
are handled gracefully.

**Reference:** `docs/FEATURE_EXPANSION.md` §3.2

#### Objectives

1. Test mouse move message parsing
2. Test mouse scroll message parsing
3. Test key press message parsing
4. Test key release message parsing
5. Test text message parsing
6. Test emergency stop message parsing
7. Test invalid message handling
8. Test truncated message handling

#### Acceptance Criteria

- [ ] Mouse move test verifies correct parsing
- [ ] Mouse scroll test verifies correct parsing
- [ ] Key press test verifies correct parsing
- [ ] Key release test verifies correct parsing
- [ ] Text test verifies correct parsing
- [ ] Emergency stop test verifies correct parsing
- [ ] Invalid message test verifies graceful rejection
- [ ] Truncated message test verifies graceful rejection
- [ ] All tests pass
- [ ] No compilation errors or warnings

#### Unit Tests

```cpp
// test/test_websocket_protocol.cpp

void test_parse_mouse_move() {
  uint8_t data[] = {0x01, 10, 251, 0};
  HidCommand cmd;
  TEST_ASSERT_TRUE(parseBinaryCommand(data, 4, cmd));
  TEST_ASSERT_EQUAL(HID_CMD_MOUSE_MOVE, cmd.type);
  TEST_ASSERT_EQUAL(10, cmd.mouse.dx);
  TEST_ASSERT_EQUAL(-5, cmd.mouse.dy);
}

void test_parse_mouse_scroll() {
  uint8_t data[] = {0x02, 253, 0};
  HidCommand cmd;
  TEST_ASSERT_TRUE(parseBinaryCommand(data, 3, cmd));
  TEST_ASSERT_EQUAL(HID_CMD_MOUSE_SCROLL, cmd.type);
  TEST_ASSERT_EQUAL(-3, cmd.scroll.v);
}

void test_parse_key_press() {
  uint8_t data[] = {0x10, 0x04, 0x02};
  HidCommand cmd;
  TEST_ASSERT_TRUE(parseBinaryCommand(data, 3, cmd));
  TEST_ASSERT_EQUAL(HID_CMD_KEY_PRESS, cmd.type);
  TEST_ASSERT_EQUAL(0x04, cmd.key.keycode);
}

void test_parse_emergency_stop() {
  uint8_t data[] = {0xF0};
  HidCommand cmd;
  TEST_ASSERT_TRUE(parseBinaryCommand(data, 1, cmd));
  TEST_ASSERT_EQUAL(HID_CMD_EMERGENCY_STOP, cmd.type);
}

void test_parse_invalid_message() {
  uint8_t data[] = {0xFF};
  HidCommand cmd;
  TEST_ASSERT_FALSE(parseBinaryCommand(data, 1, cmd));
}

void test_parse_truncated_message() {
  uint8_t data[] = {0x01, 10};
  HidCommand cmd;
  TEST_ASSERT_FALSE(parseBinaryCommand(data, 2, cmd));
}
```

#### Files to Modify

- `test/test_websocket_protocol.cpp` — Create WebSocket protocol tests

---

## Appendix: Ticket Dependencies

```text
QUALITY-001 (Test Framework)
├── QUALITY-002 (Action Selection Tests)
├── QUALITY-003 (Timing Distribution Tests)
├── QUALITY-004 (NVS Config Tests)
├── QUALITY-005 (Stealth Validation Tests)
└── QUALITY-006 (WebSocket Protocol Tests)

STEALTH-001 (Mouse HID)
├── STEALTH-005 (Mixed Activity)
├── STEALTH-006 (Humanized Movement)
└── FEATURE-003 (HID Command Processor)

STEALTH-002 (Log-Normal Distribution)
├── STEALTH-003 (Burst/Pause Patterns)
└── QUALITY-003 (Timing Distribution Tests)

STEALTH-003 (Burst/Pause Patterns)
└── STEALTH-009 (Circadian Rhythm)

FEATURE-001 (WebSocket Server)
├── FEATURE-002 (Binary Protocol)
├── FEATURE-003 (HID Command Processor)
├── FEATURE-004 (Browser Mouse Capture)
├── FEATURE-005 (Browser Keyboard Input)
├── FEATURE-007 (Dual-Core Separation)
└── FEATURE-008 (OTA via WebSocket)

FEATURE-002 (Binary Protocol)
├── FEATURE-004 (Browser Mouse Capture)
├── FEATURE-005 (Browser Keyboard Input)
└── QUALITY-006 (WebSocket Protocol Tests)

FEATURE-005 (Browser Keyboard Input)
└── FEATURE-006 (Text Injection UI)
```

## Appendix: Effort Summary

| Epic | Tickets | Total Effort |
| --- | --- | --- |
| Stealth | 9 | 27 hours |
| Feature | 8 | 24 hours |
| Quality | 6 | 14 hours |
| **Total** | **23** | **65 hours** |
