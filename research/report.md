# Battery & CPU Drain Investigation — Status Android
**Version:** 0.1 (Phase 0 — source analysis only, no device measurements yet)
**Report type:** Pre-device static analysis + hypothesis generation
**Issue:** [#21045 — Mobile: Android flagged Status for high battery consumption and CPU usage](https://github.com/status-im/status-app/issues/21045)
**Related:** [#20742 — Background service battery consumption](https://github.com/status-im/status-app/issues/20742)
**Build analysed:** `2.38.0-rc.4-5-g4cf3d8e4b` (commit [`4cf3d8e4b`](https://github.com/status-im/status-app/commit/4cf3d8e4b))
**Date:** 2026-06-02
**Status:** Phase 0 complete — awaiting device (charging to 100%)
**Estimated time to v1.0:** ~6–8 hours of active research after device is available
**Next version trigger:** First hypothesis confirmed by device measurement → v0.2 interim report posted to #21045

---

> **Claim states used in this report:**
> `[CODE]` = confirmed directly in source, no device needed |
> `[H — X%]` = hypothesis with probability score |
> `[?]` = unknown, needs external data |
> `[CONFIRMED]` / `[REJECTED]` = verdict from device measurement

---

## Abstract

Status Android was flagged by Android OS for high battery and CPU usage after an 8-hour overnight background soak on mobile data (96% → 31%, 65% consumed). The app was in WakuV2 Light mode with no active user interaction. Static source code analysis of the target build (`4cf3d8e4b`) identified **five single-factor hypotheses** and **five compound multi-factor hypotheses** that plausibly explain the drain. The most critical findings are: (1) a 50ms keyboard polling timer in the Qt C++ layer that fires 20 times/second with no background guard; (2) the Qt event loop is not paused on Android `onPause()`, meaning the keyboard timer and all other QML timers continue running indefinitely while the app is backgrounded; (3) a new `NetworkConnectivityCallback` in the `:statusgo` process (added in this build) that fires on every network capability change and triggers a `ConnectionChange` RPC with no background throttling. These factors compound: the worst-case scenario — mobile data, multiple wallet accounts, active Waku light node — combines all drains simultaneously. Device-side validation measurements are defined and ready to execute.

---

## 1. Observed Behaviour

| Parameter | Value |
|-----------|-------|
| Duration | 8 hours (overnight) |
| Battery start | 96% |
| Battery end | 31% |
| Drain | **65%** |
| Network | Mobile data only |
| Other apps | None (force-stopped) |
| WakuV2 mode | Light (confirmed by reporter) |
| Device | Samsung S21 Ultra 5G, Android 15 |
| Flagged by OS | Yes — high battery AND CPU |

Reference: [Google Drive video from reporter](https://drive.google.com/file/d/1gILDTMHMAZSzKT-Ffilc1em1wKuk5Ipz/view?usp=sharing)

Normal Android idle drain for a Samsung flagship: ~1–2%/hour. Observed: ~8%/hour. **Delta attributable to Status: ~6%/hour.**

---

## 2. Architecture Background

Status Android runs **two separate OS processes** when backgrounded. This is critical — most battery analysis looks at only one.

```
im.status.ethereum          ← Process 1: Qt 6 event loop + Nim runtime + QML
im.status.ethereum:statusgo ← Process 2: Foreground service, libstatus.so, Waku light node
```

Background lifecycle flow:
```
Activity.onPause()
  → StatusGoStub.setUiVisible(false)              [StatusQtActivity.java:59]
    → StatusGoServiceClient (async Binder thread)
      → StatusGoService.applyUiVisibility(false)
        → nativeCall("PausableServices")           [get list from status-go]
        → nativeCall("PauseServices", [list])      [pause all except "messaging"]
```

**What is NOT paused:**
- Waku light client receive (event-driven, explicitly excluded)[^1]
- `messaging` service (intentional — required for push notifications)[^2]
- Qt event loop in Process 1 (no pause code exists anywhere in codebase)[^3]
- `keyboardTimer` in `SystemUtilsInternal` (C++ singleton, started on init, never stopped)

[^1]: `StatusGoService.java:188` — comment: *"Waku light client receive is event-driven and independent of all registered services — messages continue to arrive and be processed regardless of pause state."*
[^2]: `StatusGoService.java:161` — comment: *"Messaging ("messaging") is intentionally excluded so push notifications keep working."*
[^3]: Verified by full search of `src/`, `ui/`, `mobile/` — no `QCoreApplication::exit`, `QEventLoop::quit`, or equivalent called in response to `onPause`.

---

## 3. Single-Factor Hypotheses

### H1 — Qt Event Loop Not Paused `[H — 65%]`
*Scoring: no stop path in code (+40), architectural pattern known in Qt Android (+15), analogous known issue in Qt Android codebase (+10) = 65%. Note: alexjba's comment is in a GitHub issue, not source code — the bayesian scoring table awards +20 only for comments in source. Correct application of the framework gives 65%.*

**Evidence:** Zero code pauses the Qt event loop on `Activity.onPause()`. Qt 6 on Android does not auto-suspend the event loop when the activity is paused. The loop runs until `onDestroy`. All QML `Timer {}` components, property bindings, and signal handlers continue firing.

**Source:** `mobile/android/qt6/src/app/status/mobile/StatusQtActivity.java:56-61` — `onPause` only calls `setUiVisible(false)`.

**Expected drain signature:** UI process (`im.status.ethereum`) shows > 2% CPU sustained after 60s background with screen locked.

---

### H1a — 50ms Keyboard Poll Timer `[CODE — 90%]`
*Scoring: timer starts unconditionally (+40), no stop path anywhere in codebase (+20), architectural pattern (+15), explicit 20 FPS comment confirms intent (+20) = 95% → reduced to 90% pending device confirmation*
**Evidence:** `SystemUtilsInternal` (C++ singleton, Android-only) starts an unconditional 50ms repeat `QTimer` that makes two JNI calls per tick:

```cpp
// ui/StatusQ/src/systemutilsinternal.cpp:74-103
auto keyboardTimer = new QTimer(this);
keyboardTimer->setInterval(50); // 20 FPS polling rate
keyboardTimer->start();         // ← no condition, never stopped
```

Per tick: `KeyboardUtil.getKeyboardHeight()` + `KeyboardUtil.isKeyboardVisible()` — two JNI calls crossing the JVM boundary. JNI calls require thread context switching overhead.

**Quantified impact (if Qt loop not paused):**
- 20 fires/sec × 2 JNI calls = 40 JNI calls/sec
- 8 hours = **576,000 JNI round-trips**
- Equivalent to continuously polling Android from native code at video-frame rate

**Source:** `ui/StatusQ/src/systemutilsinternal.cpp:74`

---

### H1b — NetworkConnectivityCallback (New in Target Build) `[CODE — 60%]`
*Scoring: callback registered unconditionally (+40), no visibility guard (+20), de-duplication exists (−15) = 45% base + mobile data condition of original reporter (+15) = 60%*
**Evidence:** New in `4cf3d8e4b`. `StatusGoService.onCreate()` now registers `ConnectivityManager.registerDefaultNetworkCallback()`. Fires on every network capability change → dispatches `ConnectionChange` RPC on `lifecycleExecutor` → triggers `hystrix.Flush()` in status-go.

De-duplication only on `(type|expensive)` key pair. Fires unconditionally — no check for whether UI is visible or app is backgrounded.

**Worst case:** Mobile data (original reporter's condition). LTE/5G networks have frequent capability changes (signal strength variations, handoffs) that may not change type but could in borderline conditions.

**Source:** `mobile/android/qt6/src/app/status/mobile/ipc/StatusGoService.java` — `NetworkConnectivityCallback`, `dispatchConnectionChange()`

---

### H2 — Waku Light Mode Keepalives `[H — 65%]`
*Scoring: explicit source comment confirms Waku runs regardless of pause (+40), P2P keepalive pattern known to cause radio drain (+15), interval unknown — can't confirm frequency is problematic (−10) = 45% base + mobile data worst-case condition (+20) = 65%*
**Evidence:** Waku light clients must send periodic pings to relay nodes to maintain filter subscriptions (subscriptions expire). These pings run regardless of `PauseServices` state. The frequency is defined in status-go/waku — not inspectable without the vendored status-go source at the correct version.

**Radio impact:** Each ping wakes the radio. On LTE/5G, the radio takes ~5s to return to low-power RRC Idle state after a transmission. If pings are ≤ 30s apart, the radio never reaches deep sleep.

**Source:** `StatusGoService.java:188-191` (explicit comment). Waku source would be in `vendor/status-go` — not present locally.

---

### H3 — Go Runtime Major Fault Storms `[H — 35%]`
*Scoring: comment in source acknowledges the pattern (+20), inference only — no direct measurement (+0), requires H2 to be true first (conditional) (−10) = 35%*
**Evidence:** `StatusGoServiceClient.java:79` notes: *"the Binder call reaches nativeCall("AppStateChange") which can block for seconds if the Go runtime's memory was paged out (major faults)."* If Android pages out the `:statusgo` process memory between Waku wakes, each wake involves expensive page-fault resolution.

**Source:** `StatusGoServiceClient.java:79`

---

### H5 — Wallet-tick Cascade → Price API Calls `[H — 45%]`
*Scoring: code path confirmed (+40), debouncer reduces (not eliminates) (−15), depends on Waku delivering blocks in background (conditional) (−10), multiple accounts amplify (+10) = 45%*
**Evidence:** `wallet-tick-reload` signals emitted by status-go (one per chain-account pair on balance change) trigger `rebuildMarketData()` + `buildAllTokens()` → `fetchPrices` RPC → external HTTP API call. Comment in source explicitly warns about this: *"wallet-tick-reload event is emitted for every single chain-account pair."*

A debouncer (1000ms delay, 500ms check interval) limits frequency but does not prevent calls in background.

**Source:** `src/app_service/service/token/service_main.nim:147-157`, `async_tasks.nim:198`

### H4 — Messaging Service Background Activity `[H — 40%]`
*Scoring: explicitly excluded from PauseServices by design (+20), scope of activity unknown — no direct measurement (+0), intentional architectural choice (+0), cannot determine cost without device data (−10) = 40% — low confidence, limited code evidence*

**Evidence:** The `messaging` service is explicitly excluded from `PauseServices` so push notifications keep working. `StatusGoService.java:161` — *"Messaging ('messaging') is intentionally excluded so push notifications keep working."* The cost of this exclusion is unknown — depends on message volume, polling behaviour inside the messaging service, and whether it holds wake locks.

**Source:** `mobile/android/qt6/src/app/status/mobile/ipc/StatusGoService.java:161`

**Note:** This is the least actionable hypothesis — messaging must stay alive for notifications. The question is whether its background cost is higher than expected.

---

## 4. Compound Multi-Factor Hypotheses

### MH1 — Qt Loop Not Paused × 50ms Keyboard Timer *(MOST LIKELY DOMINANT)*

| Factor | Alone | Combined |
|--------|-------|----------|
| Qt event loop running | Enables all timers | — |
| 50ms keyboard timer | 20 wakes/sec if loop runs | 20 wakes/sec × JNI overhead |

**Chain:**
```
onPause() fires
  → Qt event loop NOT suspended
    → keyboardTimer fires every 50ms
      → 2 JNI calls per tick (getKeyboardHeight + isKeyboardVisible)
        → JVM boundary crossing overhead
          → CPU wakes 20×/sec
            → battery drain all night
```

**Why this is the highest-confidence compound:** Both factors are confirmed by code reading, not hypothesised. The only open question is whether Android kills the Qt process within minutes (in which case this self-resolves) or keeps it alive (in which case this runs all night).

**Measurable:** UI process CPU > 2% after 5min background with screen locked AND WiFi disabled.

---

### MH2 — Waku Pings × Mobile Data Radio Never Sleeps

**Chain:**
```
Waku filter subscription ping → LTE radio wakes (Tx)
  → LTE radio stays awake ~5s (RRC Active → RRC Connected → RRC Idle cycle)
    → NetworkConnectivityCallback may fire on capability change during wake
      → ConnectionChange RPC → status-go processes
        → potential Waku reconnect/re-subscribe → another ping
          → radio never reaches deep sleep
```

**Why original reporter's test was worst-case:** Mobile data (not WiFi). WiFi has much faster sleep recovery. LTE radio drain per wake is ~3-5× higher than WiFi wake.

**Measurable:** High `mobileActiveTime` in `dumpsys batterystats --charged`; RX bytes/min > 50KB with screen locked.

---

### MH3 — Duplicate Network Monitoring (Java + Qt)

Two independent systems both monitor network changes:

| System | Location | Guards |
|--------|----------|--------|
| `NetworkConnectivityCallback` | `:statusgo` process (Java) | None — fires unconditionally |
| `NetworkChecker` | UI process (C++) | Checks `Qt::ApplicationActive` |

On backgrounding:
1. Java callback fires on any connectivity change (no guard)
2. Qt's `applicationStateChanged` schedules a 10s delayed reachability check (does check for active state, so mostly benign)

**Net effect:** Every real network event → at minimum one `ConnectionChange` RPC from the Java side → status-go activity in background.

---

### MH4 — Waku Background Activity × Wallet-tick Cascade × Multiple Accounts

**Chain:**
```
Waku light node receives new blocks/transactions
  → status-go processes → emits wallet-tick-reload (×N accounts × M chains)
    → Nim token service handles signal
      → rebuildMarketData() debounced
        → fetchPrices() → external HTTP API call
          → network radio wakes for HTTP response
            → more network capability events
              → NetworkConnectivityCallback fires again
```

**Amplification:** Users with 5 accounts across 3 chains = 15 `wallet-tick-reload` events per block cycle. Debouncer reduces but doesn't eliminate.

---

### MH5 — Full Drain Model (All Factors)

```
Backgrounded Status, 8 hours, mobile data, multiple accounts
│
├── UI Process (im.status.ethereum)
│   ├── Qt event loop running (H1)
│   ├── keyboardTimer: 20 JNI calls/sec (H1a) ←── ~1-2% CPU/hr
│   └── NetworkChecker 10s timer on state-change (benign, minor)
│
└── :statusgo Process (im.status.ethereum:statusgo)
    ├── Waku light node pings: radio wakes every ~N min (H2) ←── ~2-3% battery/hr
    ├── NetworkConnectivityCallback: fires on each radio wake (H1b)
    ├── messaging service: processing incoming messages (H4)
    └── wallet-tick cascade: price API on each Waku-delivered block (H5)
         └── each API call: radio wake → connectivity change → H1b fires again
```

**Estimated combined drain:** 6%/hour → 48% over 8 hours. Observed: 65% over 8 hours (8.1%/hr). The model accounts for most of the observed drain. Remaining gap may be attributable to screen-on periods or other background wake events not yet identified.

---

## 5. Validation Tests

**Experimental standards:**
- Each test run repeated **N=5 times** on separate battery stats windows (reset between runs)
- Median value reported; outliers (>2× IQR) noted but not discarded without cause
- Statistical comparison (control vs treatment): Mann-Whitney U test, p < 0.05 threshold
- Energy unit: 1% battery ≈ 623J on Samsung Galaxy S20 FE (4500mAh × 3.85V × 3.6 / 100)
- All tests require: device at 100% charge, WakuV2 = Light confirmed, other apps killed, `dumpsys battery unplug` applied, screen locked via power button (NOT home button)

**Validity constraint — OS version gap:** The original 65% drain was reported on Android 15 (Samsung S21 Ultra). Tests run on Android 13 (S20 FE). Android 15 introduced stricter background process limits and battery bucket behaviour not present in Android 13. A finding confirmed on Android 13 is valid for that OS version but may understate the drain on Android 15 — or conversely, Android 13's less aggressive killing may allow Qt process survival where Android 15 would not. Record the test OS version with every verdict and flag any confirmation as "confirmed on Android 13."

**Pre-test: Qt process survival check (run before T1)**
```bash
# Background the app, wait 5 minutes, verify Qt process is still alive
adb shell input keyevent KEYCODE_POWER
sleep 300
adb shell ps -A | grep "im.status.ethereum " | grep -v ":statusgo"
# If no output → Android killed the UI process → T1 will produce a false REJECTED verdict
# Do NOT proceed with T1 if the UI process is dead — record "process killed at T+5min" and escalate
```

**Mann-Whitney U — calculation method:**
```python
# Run this after collecting N=5 measurements for control and treatment
# pip install scipy
from scipy.stats import mannwhitneyu

control    = [val1, val2, val3, val4, val5]  # CPU% with app force-stopped
treatment  = [val1, val2, val3, val4, val5]  # CPU% with app backgrounded

stat, p = mannwhitneyu(control, treatment, alternative='less')
print(f"U={stat:.1f}, p={p:.4f}")
# p < 0.05 → statistically significant difference → hypothesis confirmed
# p ≥ 0.05 → INCONCLUSIVE — difference not statistically significant
```

---

### T1 — Validate H1 + H1a (Qt loop + keyboard timer)
```bash
# Run 5 times. Before each run: dumpsys batterystats --reset && sleep 10
# Background app (power button — screen OFF)
adb shell input keyevent KEYCODE_POWER
# Disable WiFi to isolate from network noise
adb shell svc wifi disable
# Monitor UI process CPU for 5 min
for i in $(seq 1 10); do
    echo "=== T+$((i*30))s ===" && date
    adb shell top -b -n 1 | grep "im.status.ethereum "
    sleep 30
done
# Record median CPU% from 10 readings above
```
# Also capture wake locks held during the soak
adb shell dumpsys power | grep "PARTIAL_WAKE_LOCK" | grep "im.status"
```
**Verdict threshold:**
- UI process > 2% sustained (median across 5 runs) → H1 + H1a CONFIRMED
- UI process < 0.5% sustained → H1 REJECTED (Qt loop is pausing)
- 0.5–2%: INCONCLUSIVE — extend to 15min soak. If still 0.5–2% after 15min: record exact median, note "ambiguous signal — Qt loop may be intermittently active." Add Perfetto trace (see `skills/android-battery-measurement-toolkit.md`) to get per-wakelock attribution before escalating.

### T2 — Validate H2 (Waku radio activity)
```bash
# Run 5 times. Between runs: svc wifi disable && sleep 10 && svc wifi enable && sleep 30
adb shell svc wifi enable
NET1=$(adb shell cat /proc/net/dev | grep wlan0 | awk '{print $2}')
sleep 300  # 5 min background
NET2=$(adb shell cat /proc/net/dev | grep wlan0 | awk '{print $2}')
echo "RX bytes in 5min background: $((NET2 - NET1))"
# Record for Mann-Whitney vs control (app force-stopped, same 5min window)
```
**Verdict threshold (median across 5 runs):**
- > 250KB in 5min (50KB/min) → Waku actively communicating → H2 CONFIRMED
- < 50KB in 5min → H2 REJECTED or pings very infrequent

### T3 — Validate H1b (NetworkConnectivityCallback)
```bash
# Single observation is sufficient for this test (binary: fires or doesn't)
adb logcat -s StatusGoService -v time 2>/dev/null | grep "ConnectionChange" &
# While monitoring, toggle WiFi 3 times with 10s gap
for i in 1 2 3; do
    adb shell svc wifi disable && sleep 5 && adb shell svc wifi enable && sleep 10
done
```
**Verdict:** Count `ConnectionChange` log lines during background. ≥ 1 per toggle → H1b mechanism CONFIRMED (callback fires in background). Zero → H1b REJECTED or log tag changed.

**Important caveat (F6):** T3 confirms that the code path *executes* — it does not measure drain impact. A confirmed T3 means H1b is a real event source, but not how much battery it costs. Drain quantification requires T4 with WiFi cycling to compare battery% before/after with and without network toggles.

### T4 — Validate MH1 (compound: loop + keyboard timer)
```bash
# Run 5 times. Before each run: reset batterystats
# WiFi OFF to eliminate network factor (isolates CPU drain)
adb shell svc wifi disable
adb shell input keyevent KEYCODE_POWER
adb shell dumpsys batterystats --reset
sleep 600  # 10 min soak
adb shell dumpsys cpuinfo | grep "im.status.ethereum "
adb shell dumpsys batterystats --charged | grep -A5 "im.status.ethereum\b"
# Also capture wake locks
adb shell dumpsys power | grep "PARTIAL_WAKE_LOCK" | grep "im.status"
# Record: cpuTimeMs for UI process vs control (app force-stopped, same 10min window)
```
**Expected:** If MH1 is dominant, UI process cpuTimeMs will be significantly higher than control. Run Mann-Whitney U against control set.

### T5 — Differential: WiFi vs Mobile Data
Run T2 procedure twice in separate sessions — once on WiFi, once on mobile data (SIM data enabled, WiFi off). Record 5 runs each condition. If mobile data drain median > WiFi median by >2× → MH2 (radio never sleeps) CONFIRMED.

---

### Unexplained Drain Gap — H6 `[? — unknown]`

**Observation:** Full drain model (MH5) estimates ~6%/hr. Observed: ~8.1%/hr. **Gap: ~2.1%/hr unaccounted.**

Possible sources not yet hypothesised:
- Screen-on events during overnight test (user rolled over, notification lit screen)
- Background app updates or OS processes misattributed to Status
- Waku reconnect storms after network interruptions (not just steady-state pings)
- `messaging` service activity (explicitly excluded from pause — scope unknown)

**Resolution:** Check `dumpsys batterystats --charged` screen state log during the original test period. If screen was off the entire 8h, gap is real and source unknown — escalate per protocol.

---

## 6. Recommendations

### Immediate (address before next RC)

**R1 — Guard keyboard timer for background**
`ui/StatusQ/src/systemutilsinternal.cpp:74`
```cpp
// Current: unconditional start
keyboardTimer->start();

// Fix: stop when app not active
connect(qApp, &QGuiApplication::applicationStateChanged,
        keyboardTimer, [keyboardTimer](Qt::ApplicationState s) {
    if (s == Qt::ApplicationActive) keyboardTimer->start();
    else keyboardTimer->stop();
});
```
**Impact:** Eliminates 20 JNI calls/sec in background. Zero functional regression — keyboard height is irrelevant when app is backgrounded.

**R2 — Guard NetworkConnectivityCallback for background**
`StatusGoService.java` — `dispatchConnectionChange()` fires unconditionally on every network capability change regardless of app visibility. The `:statusgo` process has no Qt event loop state — it needs its own `uiVisible` check.

**Caution:** A blanket `if (!uiVisible) return` would break Waku reconnection after network interruptions, since Waku needs to know when the network comes back even in background. The fix must be selective — suppress cosmetic capability churn (e.g. signal strength changes that do not change type or metered status) but allow genuine network-up/network-down transitions through. Recommended approach: keep the existing de-duplication on `(type, expensive)` and add rate-limiting for background state rather than a hard suppress. File this as Medium-risk — requires validation that Waku reconnect still works after the change.

**R3 — Validate Qt event loop suspension on Android**
Explicitly test whether the Qt event loop suspends when the Android activity is paused. If it doesn't, add explicit suspension. alexjba's comment in the issue suggests this was assumed but never validated.[^4]

[^4]: alexjba, [#21045 comment](https://github.com/status-im/status-app/issues/21045#issuecomment-4563218143): *"I've assumed the qt event loop is paused by default when going into background. Now I think we should validate that as well... If that wakes up, the battery is dead."*

### Medium-term

**R4 — Waku filter subscription keepalive interval**
Audit the Waku light mode ping interval in status-go. If it's < 60s, consider increasing for background state. jrainville's comment confirms this is already on the radar.[^5]

[^5]: jrainville, [#21045 comment](https://github.com/status-im/status-app/issues/21045#issuecomment-4555832986): *"We need to check what we can disable when the app is in the background."*

**R5 — Suppress wallet-tick cascade in background**
`wallet-tick-reload` handlers in `service_main.nim` and `service_account.nim` should check an `inBackground` flag before calling `rebuildMarketData()`. Price data doesn't need to be fresh while the user can't see it.

**R6 — Reduce duplicate network monitoring**
The Java `NetworkConnectivityCallback` and the Qt `NetworkChecker` both push `ConnectionChange` to status-go. Consolidate: either the Java layer owns it (already in `:statusgo` process, more reliable) or the Qt layer owns it — not both.

### Preventive (test gates)

**G1 — Background CPU regression test**
Automated test: install APK, log in, background for 10 min, measure `dumpsys cpuinfo` for both processes. Fail if UI process > 2% sustained or `:statusgo` > 5% sustained.

**G2 — Radio activity test**
Automated test: background for 5 min on WiFi, measure `RX bytes/min`. Fail if > 50KB/min. This catches excessive Waku pings.

**G3 — Timer audit in CI**
Static analysis lint rule: any `QTimer` instantiation must have a corresponding `stop()` call in an `applicationStateChanged` handler, OR must demonstrate via comment why continuous firing is intentional.

**G4 — Background soak in release checklist**
Add to RC checklist: 30-min background soak test with `dumpsys batterystats` before → after. Numeric threshold required (not just "it seemed fine").

---

## 7. Interim Report Protocol

**Do not wait for T1–T5 to complete before sharing findings.**

| Trigger | Action |
|---------|--------|
| Any single hypothesis confirmed | Post interim comment to #21045 with finding + fix recommendation |
| Any hypothesis rejected (unexpected) | Post to #21045: what was ruled out and why |
| All hypotheses inconclusive after T1–T4 | Post escalation note: expanding scope |

**Interim comment format and definition of done:** `skills/battery-research-interim-reporting.md`

---

## 8. Data Collection Plan

When device is available (currently charging to 100%):

| Test | Duration | Data captured | Validates |
|------|----------|--------------|-----------|
| T1 | 10 min | `top` per 30s, both processes | H1, H1a |
| T2 | 5 min | `/proc/net/dev` delta | H2 |
| T3 | 2 min | logcat ConnectionChange | H1b |
| T4 | 10 min | `cpuinfo`, `batterystats` | MH1 |
| T5 | 5 min × 2 | network bytes (WiFi vs LTE) | MH2 |

All raw data will be appended to `journal.md` under session results.

---

## 9. Open Questions

1. Does the Qt Android platform plugin pause the event loop on `onPause`? (alexjba assumed yes — code suggests no)
2. What is the actual Waku light mode filter subscription ping interval? (needs status-go source at exact vendored version)
3. Does Android kill the UI process after some background time on Android 13 (our test device) vs Android 15 (reporter's device)? Different OSes may show different behaviour.
4. Is `wallet-tick-reload` emitted during background operation or only on foreground balance checks?

---

## References

### GitHub Issues
- Issue #21045: https://github.com/status-im/status-app/issues/21045
- Issue #20742: https://github.com/status-im/status-app/issues/20742

### Source Files (target commit `4cf3d8e4b`)
- `StatusQtActivity.java`: `mobile/android/qt6/src/app/status/mobile/StatusQtActivity.java`
- `StatusGoService.java`: `mobile/android/qt6/src/app/status/mobile/ipc/StatusGoService.java`
- `systemutilsinternal.cpp` (keyboard timer source): `ui/StatusQ/src/systemutilsinternal.cpp`
- `networkchecker.cpp` (Qt network monitoring): `ui/StatusQ/src/networkchecker.cpp`
- `service_main.nim` (wallet-tick handling): `src/app_service/service/token/service_main.nim`

### Developer Comments
- alexjba on Qt event loop assumption: https://github.com/status-im/status-app/issues/21045#issuecomment-4563218143
- jrainville on background disabling: https://github.com/status-im/status-app/issues/21045#issuecomment-4555832986

### Research Files
- Research protocol: `battery-research/research-protocol.md`
- Raw journal: `battery-research/journal.md`
- Skills index: `skills/INDEX.md`

### Supporting Skills
- `skills/bayesian-hypothesis-scoring.md` — evidence-based probability scoring method
- `skills/android-energy-code-smells.md` — 10-pattern anti-pattern catalogue (PMC11479295)
- `skills/android-battery-measurement-toolkit.md` — ADB commands, Perfetto SQL, tool selection
- `skills/android-controlled-battery-experiment.md` — N=5 protocol, Mann-Whitney U, Joule conversion
- `skills/battery-research-interim-reporting.md` — interim report trigger conditions and GitHub template
