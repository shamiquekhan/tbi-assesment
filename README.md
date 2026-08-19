# Drift-Proof Countdown Timer

**Candidate:** Shamique Khan
**Registration Number:** 25bai10187
**Email:** shamique.25bai10187@vitbhopal.ac.in
**Date:** 18 August 2026

---

## Part 1 — Bug Fixes

### Bugs Identified
1. **Multiple Interval Leak:** The original code never stores the `setInterval` ID, so each click of "Start" spawns an additional timer loop. After three clicks the counter decrements 3× per second.
2. **Missing Zero-Padding:** When `seconds < 10`, the display shows `9:5` instead of `9:05` because the number is concatenated directly.
3. **No Termination Condition:** `timeLeft` decrements past zero into negative values. The timer never stops.

### Fixes Applied
- Stored `timerInterval` and guard with `if (timerInterval !== null) return;`
- Used `seconds.toString().padStart(2, '0')` for consistent formatting
- Added `if (timeLeft <= 0)` block that clears the interval and clamps the display to `0:00`

---

## Part 2 — Features & Architecture

### Pause / Resume
Instead of a naive counter (`timeLeft--`), the timer is **deadline-driven**:
- `endTime = clock() + remainingMs`
- **Pause** stops the animation loop and snapshots `remainingMs = endTime - clock()`
- **Resume** computes a fresh `endTime` based on the current monotonic clock plus the saved remainder

This guarantees that an intentional pause does not cost the speaker a single second of stage time.

### Drift-Proofing Strategy
| Risk | Mitigation |
|------|------------|
| Browser throttles background tabs | `requestAnimationFrame` pauses automatically when hidden; recalculation on `visibilitychange` ensures the display is correct the instant the tab is viewed again |
| System clock skew | `performance.now()` is immune to NTP updates or user clock changes; `Date.now()` would jump and drift |
| Scheduler/frame delay | The timer never decrements per frame. It always calculates `remaining = endTime - clock()` on demand, so delayed or dropped frames cannot accumulate timing error |

### Why `requestAnimationFrame` over `setInterval`?
- Syncs with the display refresh rate (no dropped frames, smooth UI)
- Naturally pauses in background tabs (battery-friendly)
- The deadline architecture makes the renderer irrelevant to accuracy; `rAF` is only a rendering scheduler, not the clock

### OS Sleep Limitation
Browser/platform behavior for `performance.now()` during OS sleep is **not uniform**. The specification expects the monotonic clock to advance through sleep, but real implementations differ across browsers and operating systems (particularly on Linux/macOS). This implementation does not claim a universal sleep-proof guarantee; instead, it documents the limitation explicitly. A platform-independent sleep-proof web timer would require an additional wall-clock reconciliation strategy.

### Timing Model
The timer is deadline-driven rather than frame-driven.

**While running:**
```
remaining = endTime - clock()
```

**When paused:**
```
remaining is snapshotted; no elapsed time accumulates
```

**On resume:**
```
endTime = clock() + remaining
```

**Why this avoids normal drift:**
The timer never decrements once per frame or once per interval. The scheduler only determines when the UI is refreshed. Therefore delayed animation frames do not accumulate one-frame-per-frame timing error.

### Clock Choice
`performance.now()` provides a monotonic clock that is not affected by normal system wall-clock adjustments such as NTP synchronization or manual clock changes.

### Background Tabs
Browsers may throttle or suspend `requestAnimationFrame()` in background tabs. Because the timer is deadline-driven, the next visible update recalculates the remaining duration instead of attempting to compensate for missed frames.

### Explicit State Machine
The timer is modeled as a finite-state machine with four states:

```
READY ──Start──> RUNNING
RUNNING ──Pause──> PAUSED
PAUSED ──Resume──> RUNNING
RUNNING ──deadline──> FINISHED
PAUSED ──Reset──> READY
RUNNING ──Reset──> READY
FINISHED ──Reset──> READY
```

Button actions are valid only for particular states. Double-click protection is inherent in the state guards.

### Clock Abstraction
The timing source is injectable via a `clock` function:
```js
function createTimerEngine(clock = () => performance.now()) { ... }
```
This makes the engine testable with a fake clock without waiting real time.

### Edge Cases Handled
- **Double-click protection:** All state transitions guard against duplicate clicks
- **Pause at zero:** If Pause is pressed exactly at the deadline, the state transitions to `FINISHED`, not `PAUSED`
- **Resume after finish:** Disabled; Reset must be pressed first
- **Tab switching:** `visibilitychange` listener triggers immediate recalculation when the tab returns to focus
- **Color warnings:** Visual feedback turns orange at ≤1 min and red at ≤10 sec
- **rafId hygiene:** Animation frame ID is nulled after cancellation, maintaining the invariant `RUNNING → rafId != null`, `NOT RUNNING → rafId == null`

### Invariants
1. `remainingMs >= 0`
2. `state === RUNNING  =>  endTime != null`
3. `state !== RUNNING  =>  no active animation loop`
4. `remainingMs === 0  =>  state === FINISHED`
5. **Running never decrements `remainingMs` per frame.** Instead: `remainingMs = endTime - currentTime`

---

## Manual Test Matrix

| Test | Steps | Expected |
|------|-------|----------|
| **1. Repeated Start** | Click Start 6 times rapidly | Only one countdown loop runs |
| **2. Zero-padding** | Wait until 9:09, then 0:09 | Displays `9:09` and `0:09`, never `9:9` |
| **3. No negative** | Wait past 0:00 | Stays at `0:00`, shows "Time Up" |
| **4. Pause/Resume** | Start → wait 5s → Pause → wait 20s → Resume | Display holds during pause; continues from paused value |
| **5. Repeated Pause** | Pause twice | Second click has no effect |
| **6. Repeated Resume** | Resume 5 times rapidly | Exactly one countdown loop |
| **7. Background tab** | Start → switch tab 30s → return | ~30s elapsed; display correct |
| **8. System clock change** | Start → change OS clock ±5min → return | Timer unaffected by wall-clock adjustment |
| **9. Laptop sleep** | Start at 2:00 → sleep 30s → wake | Observe result; document browser/OS combo |

---

## Architecture Diagram (Text)

```
┌──────────────────┐
│  Timer State     │
│ READY            │
│ RUNNING          │
│ PAUSED           │
│ FINISHED         │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Deadline Engine  │
│                  │
│ endTime - now()  │
└────────┬─────────┘
         │
         ▼
┌────────────────────┐
│ requestAnimationFrame │
│      renderer      │
└─────────┬──────────┘
         │
         ▼
   ┌─────────────┐
   │  DOM display │
   └─────────────┘

Clock ≠ Renderer
```

The single sentence `Clock ≠ Renderer` captures the entire design: `performance.now()` is the clock; `requestAnimationFrame` is only the rendering scheduler.

---

## How to Run
1. Clone the repo
2. Open `part1.html` or `part2.html` in any modern browser
3. For Part 2: try starting the timer, switching tabs for 30 seconds, and returning — the time will be exactly correct

---

## Interview Prep — Key Talking Points

**Q: "Why `performance.now()` instead of `Date.now()`?"**
> `Date.now()` is wall-clock time. If the OS syncs with NTP or the user adjusts the clock while the laptop sleeps, `Date.now()` can jump forward or backward, making the timer drift or go negative. `performance.now()` is monotonic — it only moves forward at a steady rate, independent of system clock changes.

**Q: "What happens if I mash the Start button in Part 2?"**
> Nothing. The state machine returns early if the current state doesn't allow the transition (e.g., `READY` → `RUNNING` only works from `READY` or `PAUSED`). The state machine prevents invalid transitions.

**Q: "Does `requestAnimationFrame` really solve the sleep problem?"**
> It solves the *drift* problem from frame suspension. When the laptop sleeps, the animation loop pauses. When it wakes, the next frame calculates `endTime - clock()`, which reflects real elapsed time — **provided the browser's `performance.now()` advanced during sleep**. Browser implementations differ here; Chrome/Chromium and Firefox generally advance it on Windows, but behavior on Linux/macOS varies. This is why the README documents the limitation rather than claiming a universal guarantee.

**Q: "Why not use `setInterval` with a 100 ms tick?"**
> You could, but `requestAnimationFrame` is more efficient (matches monitor refresh rate) and self-throttles in background tabs. A 100 ms `setInterval` wastes CPU cycles when no one is looking. Crucially, neither approach measures time — the deadline architecture does that.

**Q: "What if `performance.now()` pauses during sleep on some platforms?"**
> That's a valid platform limitation. In a mission-critical production app I'd add a `Date.now()` sanity check inside the `visibilitychange` handler: if the wall-clock gap and monotonic gap diverge by more than a few seconds, I'd log a warning or fall back to wall-clock time. For this assessment, `performance.now()` covers the vast majority of modern devices and the architecture cleanly isolates the clock for such an extension.

---

## Quick Checklist Before Submitting
- [ ] `original.html` is **exactly** the buggy snippet, untouched
- [ ] `part1.html` fixes all 3 bugs and is testable by opening in a browser
- [ ] `part2.html` has Start, Pause, Resume, and Reset buttons
- [ ] `part2.html` uses `performance.now()` (or equivalent monotonic source)
- [ ] `part2.html` handles `visibilitychange` for tab-switching
- [ ] `part2.html` uses explicit state machine (READY/RUNNING/PAUSED/FINISHED)
- [ ] `part2.html` has clock abstraction for testability
- [ ] `part2.html` centralizes animation-frame scheduling and UI rendering
- [ ] `part2.html` includes accessibility attributes (aria-live, aria-label, role)
- [ ] `README.md` explains reasoning, not just what the code does
- [ ] `README.md` documents OS sleep limitation honestly
- [ ] `README.md` includes manual test matrix
- [ ] `README.md` includes architecture diagram and invariants
- [ ] You can explain every line live in the interview