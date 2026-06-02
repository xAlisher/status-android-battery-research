---
id: android-energy-code-smells
title: Catalogue of Android energy anti-patterns to check during battery research
phase: discovery
type: pattern
severity: high
severity_reason: Missed anti-pattern = missed root cause = wrong fix recommendation
source: extracted-from-research
upstream_url: https://pmc.ncbi.nlm.nih.gov/articles/PMC11479295/
last_used: 2026-06-02
created: 2026-06-02
status: active
---

## Problem
Battery drain usually comes from a known class of anti-patterns. Checking against this catalogue during source analysis avoids missing obvious causes.

## Catalogue (literature-backed)

### 1. Durable WakeLock `[CODE — detectable by grep]`
**Pattern:** `WakeLock.acquire()` called without guaranteed `release()` — or with timeout so long it's effectively infinite.
**Impact:** CPU/screen stays on indefinitely. Most studied Android energy smell.
**Detection:**
```bash
grep -rn "acquire()" --include="*.java" --include="*.kt" | grep -v "acquire(long"
# Any acquire() without timeout is suspicious
```
**Fix:** `wl.acquire(60 * 1000 * 10L)` — always use timeout form.

### 2. Unconstrained Polling Timer `[CODE — detectable by grep]`
**Pattern:** `QTimer`, `Handler.postDelayed`, `AlarmManager`, or `Timer` started unconditionally with no background guard.
**Impact:** Timer fires continuously regardless of app visibility. At 50ms = 20 CPU wakes/sec.
**Detection:**
```bash
grep -rn "setInterval\|QTimer\|postDelayed\|AlarmManager" --include="*.cpp" --include="*.java"
# Check if timer.stop() is called in onPause/applicationStateChanged
```
**Fix:** Stop timer when app backgrounds, restart on resume.
**Status app instance:** `systemutilsinternal.cpp:74` — 50ms keyboard poll, never stopped.

### 3. Unguarded Background Network Callback `[CODE — detectable by grep]`
**Pattern:** `ConnectivityManager.registerDefaultNetworkCallback()` or similar registered without checking app visibility before dispatching work.
**Impact:** Every network state change (frequent on mobile data) wakes the `:statusgo` process.
**Detection:**
```bash
grep -rn "registerDefaultNetworkCallback\|registerNetworkCallback" --include="*.java" --include="*.kt"
# Check if handler checks a visibility flag before dispatching
```
**Fix:** Guard with `if (!uiVisible) return;` or rate-limit background dispatches.
**Status app instance:** `StatusGoService.java` — `NetworkConnectivityCallback`, no background guard.

### 4. Qt Event Loop Not Suspended on Mobile `[ARCHITECTURAL]`
**Pattern:** Qt for Android does not auto-suspend the event loop on `Activity.onPause()`. All QML Timers, bindings, and signal handlers keep running.
**Impact:** All Qt-side timers (including polling timers) run at full rate all night.
**Detection:** Search for `onPause` handler — does it call anything that suspends the event loop?
**Fix:** Add an `applicationStateChanged` handler on each timer that calls `timer->stop()` when state is not `Qt::ApplicationActive` and `timer->start()` on resume. Do NOT use `QCoreApplication::processEvents()` with `WaitForMoreEvents` — this blocks the calling thread waiting for events (the opposite of suspending) and will cause a CPU spin or deadlock. The only correct mitigation is guarding each timer individually or adding explicit event loop quiescing via `QEventLoop` with manual exec/quit control.
**Literature reference:** Known Qt Android issue — alexjba confirmed assumption was never validated.

### 5. Radio-Keeping Network Pattern `[ARCHITECTURAL]`
**Pattern:** Frequent small network requests (< 30s apart) prevent mobile radio from entering RRC Idle state.
**Impact:** LTE radio takes ~5s to return to low-power state after any Tx. If pings are < 30s apart, radio never sleeps.
**Detection:** Measure bytes/min in background. If > 50KB/min on cellular → radio likely never sleeping.
**Fix:** Batch requests, use exponential backoff, increase keepalive intervals for background state.
**Literature threshold:** Battery Historian flags wakeup alarms every ≤ 10s and syncs every ≤ 30s as red flags.

### 6. HashMap Usage → GC Pressure `[CODE]`
**Pattern:** `HashMap<Integer, Object>` instead of `SparseArray`/`ArrayMap` in hot paths.
**Impact:** Autoboxing + GC pauses = CPU spikes. Up to 27% battery improvement when fixed (MDPI study).
**Detection:**
```bash
grep -rn "HashMap<Integer\|HashMap<Long" --include="*.java" --include="*.kt"
```
**Fix:** Replace with `SparseArray<V>` (integer key) or `ArrayMap<K,V>`.

### 7. Background Service Without Pause Mechanism `[ARCHITECTURAL]`
**Pattern:** Service registered but not implementing `PausableService` interface — or implementing it but not being registered with status-go's pausable list.
**Impact:** Service runs at full rate in background consuming CPU and network.
**Detection:** Check `nativeCall("PausableServices", "[]")` output at runtime — are all expected services listed?
**Fix:** Implement and register `PausableService` interface in status-go.

### 8. Signal Cascade Without Background Guard `[CODE]`
**Pattern:** Event handler (e.g. `wallet-tick-reload`) triggers expensive work (price API fetch, token rebuild) without checking whether UI is visible.
**Impact:** Background balance updates trigger full market data rebuild + external HTTP calls.
**Detection:**
```bash
grep -rn "wallet-tick-reload\|balance.*update" --include="*.nim" | grep -v "inBackground\|uiVisible"
```
**Fix:** Add `if self.inBackground: return` guard at the top of background-irrelevant handlers.

### 9. Dual Network Monitor (Redundant Dispatch) `[ARCHITECTURAL]`
**Pattern:** Two independent systems both monitoring network state and both pushing the same RPC to the backend.
**Impact:** Every real network event causes double-processing. Compounds with H3 above.
**Status app instance:** Java `NetworkConnectivityCallback` + Qt `NetworkChecker` both push `ConnectionChange`.
**Fix:** Designate one owner. Since `NetworkConnectivityCallback` is in the `:statusgo` process (closer to where it's needed), prefer it and remove the Qt-side dispatch.

### 10. Internal Getter/Setter in Hot Paths `[CODE]`
**Pattern:** Virtual method calls for field access within the same class — more expensive than direct access on Android Dalvik/ART.
**Impact:** 9% battery improvement per MDPI study when corrected.
**Note:** Lower priority compared to architectural issues above.

## Quick Scan Script

```bash
# Run in project root — flags most common smells
echo "=== WakeLock without timeout ===" && grep -rn "\.acquire()" --include="*.java" --include="*.kt" | grep -v "acquire(long\|//\|test"
echo "=== Unconstrained QTimer ===" && grep -rn "\.start()" --include="*.cpp" | grep -i "timer" | grep -v "singleShot\|stop\|test"
echo "=== Network callbacks ===" && grep -rn "registerDefaultNetworkCallback\|registerNetworkCallback" --include="*.java"
echo "=== HashMap with Integer key ===" && grep -rn "HashMap<Integer\|HashMap<Long" --include="*.java" --include="*.kt"
echo "=== Wallet signals without background guard ===" && grep -rn "wallet-tick" --include="*.nim" | grep -v "inBackground\|background"
```

## Impact Reference (from literature)

| Fix | Battery improvement |
|-----|-------------------|
| Fix all three Java code smells | 36% capacity improvement |
| HashMap → SparseArray | 27% |
| Power saving mode on | 9% |
| Dark theme | 1.4% |
| Polling timer eliminated | varies — up to dominant cause |

## See also
- `android-battery-measurement-toolkit.md`
- `android-controlled-battery-experiment.md`
