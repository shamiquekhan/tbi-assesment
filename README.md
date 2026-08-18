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
- `endTime = performance.now() + remainingMs`
- **Pause** stops the animation loop and snapshots `remainingMs = endTime - performance.now()`
- **Resume** computes a fresh `endTime` based on the current monotonic clock plus the saved remainder

This guarantees that a 5-minute pause does not cost the speaker a single second of stage time.

### Drift-Proofing Strategy
| Risk | Mitigation |
|------|------------|
| Browser throttles background tabs | `requestAnimationFrame` pauses automatically when hidden; recalculation on `visibilitychange` ensures the display is correct the instant the tab is viewed again |
| Laptop sleeps | Because remaining time is derived from `performance.now()` (monotonic clock), the math is correct regardless of how many animation frames were skipped |
| System clock skew | `performance.now()` is immune to NTP updates or user clock changes; `Date.now()` would jump and drift |

### Why `requestAnimationFrame` over `setInterval`?
- Syncs with the display refresh rate (no dropped frames, smooth UI)
- Naturally pauses in background tabs (battery-friendly)
- Forces us to recalculate from the monotonic clock on every frame, making drift architecturally impossible

### Edge Cases Handled
- **Double-click protection:** All state transitions guard against duplicate clicks
- **Resume after finish:** Disabled; Reset must be pressed first
- **Tab switching:** `visibilitychange` listener triggers an immediate `updateDisplay()` when the tab returns to focus
- **Color warnings:** Visual feedback turns orange at ≤1 min and red at ≤10 sec

---

## How to Run
1. Clone the repo
2. Open `part1.html` or `part2.html` in any modern browser
3. For Part 2: try starting the timer, switching tabs for 30 seconds, and returning — the time will be exactly correct
