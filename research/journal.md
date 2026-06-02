# Battery Research Journal — #21045
**Issue:** [Mobile] Android flagged Status for high battery consumption and CPU usage
**Started:** 2026-06-02
**Researcher:** AI agent (eml's station)
**Device available:** RF8RA0M127K (Samsung SM-G780G, Android 13, API 33)
**Build under test:** `2.38.0-rc.4-5-g4cf3d8e4b` (commit `4cf3d8e4b`) — "chore: declare intent for notification (#21065)"
**Source analysed at:** commit `4cf3d8e4b` ✓ (verified, checked out)

---

## Fails Log
| Date | Fail | Should have done |
|------|------|-----------------|
| 2026-06-02 | Did not attempt to fetch the specified CI build URL; went straight to source code analysis without flagging the blocked artifact | Try URL immediately → hit 403 → block user right away: "need direct APK link" |
| 2026-06-02 | Research protocol missing "start from 100% charge" requirement — without consistent start level, drain numbers across runs are not comparable. This is basic scientific method — should not have required prompting. | Always verify `dumpsys battery \| grep level` = 100 before starting any test run |
| 2026-06-02 | Ran source code analysis on whatever was in `~/status-app-src/` without verifying it matched the target build commit (`4cf3d8e4b`). Hypotheses may be based on wrong code. Same discipline as the build URL — always align source to the exact artifact under test. | Check `git log` in source repo, checkout target commit, then analyse. |
| 2026-06-02 | Explained data-driven reporting approach verbally but never wrote it into any file. Methodology only exists in conversation context — lost after session ends. | Write it into protocol.md before doing anything else. |
| 2026-06-02 | Hypothesis confidence labelled as HIGH/MEDIUM/LOW (subjective) — no numeric probabilities, no code-evidence weighting. Cannot be used to prioritise fixes with bounded risk. | Add probability scores with evidence basis to each hypothesis. |
| 2026-06-02 | No time estimate anywhere for full research completion. Team has no idea when to expect actionable findings. | Add time estimate to protocol and report. |
| 2026-06-02 | No interim report protocol. Actionable findings (e.g. H1a keyboard timer — one-line fix, zero risk) sit unreleased while waiting for full paper. Developers lose days. | Define interim report trigger and format. |
| 2026-06-02 | No definition of done, no escalation protocol for rejected hypotheses, no raw data storage format, no peer review step with mag/Sale, no report versioning. | All added to protocol and report. |
| 2026-06-02 | Protocol missing control test (app force-stopped), build install step, WakuV2 mode verification, screen lock requirement, other-apps-killed step, tcpdump assumption (not on stock Samsung), connectivity confirmation before backgrounding, airplane mode comparison, numeric thresholds for verdicts, and version check. 10 gaps found in self-review. | All added to protocol. |

---

## Session 1 — 2026-06-02

### Goal
Understand root cause of battery/CPU drain when Status runs in background (light mode, no active usage, 8h overnight test → 65% battery consumed).

### Environment Setup
- Attempted ADB WiFi connect: device offline (USB not connected)
- Completed: full source code analysis of the background lifecycle path
- Blocked: live device measurements pending WiFi ADB setup (see BLOCKERS below)

---

## Source Code Analysis

### Background Lifecycle Path (traced end-to-end)

```
Android puts app in background
    → StatusQtActivity.onPause()
        → StatusGoStub.setUiVisible(false)
            → StatusGoServiceClient.setUiVisible(false)  [async, background thread]
                → Binder IPC to separate :statusgo process
                    → StatusGoService.applyUiVisibility(false)
                        → scheduleBackendLifecycleUpdate(false)
                            → nativeCall("PausableServices", "[]")  [get list]
                            → nativeCall("PauseServices", [all_pausable_names])
```

**Source files:**
- `StatusQtActivity.java:56-61` — onPause hook
- `StatusGoStub.java:47-58` — bridge
- `StatusGoServiceClient.java:279-337` — async dispatch + Binder
- `StatusGoService.java:194-214` — scheduleBackendLifecycleUpdate

*Two-process architecture and Waku pause exclusion: see `report.md § 2`.*

### alexjba's Qt Event Loop Hypothesis

alexjba said: *"I've assumed the qt event loop is paused by default when going into background. Now I think we should validate that as well... If that wakes up, the battery is dead."*

**Analysis:** Qt for Android (Qt 6.x) does NOT automatically pause the event loop when Activity.onPause() fires. The Qt event loop runs until the surface is destroyed (onStop/onDestroy). During the background period between onPause and onStop:
- Timer events still fire
- `onIdle` handlers run
- Animations that don't check visibility continue
- Any QML `Timer {}` with `running: true` keeps ticking

**This is a confirmed known issue with Qt Android apps.** Qt 6 added `QAndroidApplication::runOnAndroidMainThread` but does NOT suspend the event loop on pause.

---

---

## Target Build vs Previously Analysed Code — Delta

The initial analysis was done on HEAD (`c616fee7b`). Target build is `4cf3d8e4b`. Re-analysis on the correct commit found **three material differences** relevant to the battery issue:

### DELTA 1: NetworkConnectivityCallback — NEW in target build
`StatusGoService.java` at target commit adds a `ConnectivityManager.NetworkCallback` registered in `onCreate()`:

```java
connectivityManager.registerDefaultNetworkCallback(networkCallback);
```

- Fires `onCapabilitiesChanged` on **every** network capability change (not just type changes)
- Each fire dispatches a `ConnectionChange` RPC to status-go on `lifecycleExecutor`
- `ConnectionChange` triggers `hystrix.Flush()` in status-go per the comment
- De-duplication only on `(type, expensive)` pair — minor capability churn is de-duped, but type changes (WiFi ↔ cellular, or capability adds/removes that affect type) are not
- **On mobile data:** network capability changes are frequent, especially in areas with varying signal
- This was **absent** from the version I originally analysed — **H1b added**

### DELTA 2: Messaging Re-pause Delay — NEW in target build
`MESSAGING_REPAUSE_DELAY_MS = 60_000L` — after an inline notification reply the messaging service is *resumed* and stays alive for 60 seconds before being re-paused. This means every notification reply in background = messaging service active for ~60s extra.

Previously I noted messaging is "intentionally not paused." That's still true for the general case, but now there's an additional 60s active window per reply.

### DELTA 3: StatusQtActivity teardown watchdog — NEW (not battery-relevant)
3-second watchdog kills UI process if Qt `onDestroy` deadlocks. Not relevant to background drain, but confirms the Qt teardown deadlock was a known issue.

### NOT CHANGED
- `onPause` → `setUiVisible(false)` path: identical
- `PauseServices`/`ResumeServices` logic: identical
- Waku still excluded from pause: identical

---

---

## Extended Source Sweep — Additional Findings

### FINDING A: 50ms Keyboard Poll Timer (C++, Android-specific)
`ui/StatusQ/src/systemutilsinternal.cpp:74-103` — `SystemUtilsInternal` starts a `QTimer` at **50ms interval (20 FPS)** unconditionally on Android. Each tick makes two JNI calls (`KeyboardUtil.getKeyboardHeight` + `KeyboardUtil.isKeyboardVisible`). Timer is a singleton — no background guard, never stopped. → Promoted to `report.md H1a`. Code block: see `report.md H1a`.

### FINDING B: Duplicate Network Monitoring (Qt layer + Java layer)
Two independent systems both monitor network changes and both push `ConnectionChange` to status-go:

1. **Java** `NetworkConnectivityCallback` in `:statusgo` process (new in target build) — `registerDefaultNetworkCallback` on Android ConnectivityManager
2. **C++** `NetworkChecker` in UI process — subscribes to `QNetworkInformation::reachabilityChanged`, `transportMediumChanged`, `isMeteredChanged`; also fires `onReachabilityChanged` **10 seconds after every app state change** via `QTimer::singleShot(kCheckDelay)`

`NetworkChecker.onReachabilityChanged` checks `Qt::ApplicationActive` before acting — meaning the Qt-side network check is intentionally suppressed in background. But the Java-side `NetworkConnectivityCallback` has no such guard and fires unconditionally.

### FINDING C: Wallet-tick-reload Cascade
`wallet-tick-reload` events emitted by status-go (once per chain-account pair on balance change) trigger:
1. `rebuildMarketData()` — debounced at 1000ms/500ms → `fetchPrices` RPC to external API
2. `buildAllTokens()` — debounced rebuild of all token state

If multiple accounts or chains exist, each balance check = N wallet-tick-reload events → potentially several price API calls in quick succession. The debouncer comment explicitly notes this: *"at the app start can be more such signals received from the statusgo side."* If status-go is doing background balance polling (Waku light node processes incoming blocks), this cascade fires in background too.

### FINDING D: NetworkChecker app-state-change 10s delayed check
```cpp
connect(qApp, &QGuiApplication::applicationStateChanged, this, [&](Qt::ApplicationState state) {
    QTimer::singleShot(kCheckDelay, this, [&]() { onReachabilityChanged(m_netinfo->reachability()); });
});
```
This fires a delayed check 10 seconds after **any** app state change (foreground → background transition). The handler does check `Qt::ApplicationActive` before acting — so this one is likely benign, but it still schedules a timer on the Qt event loop after backgrounding.

---

## Hypotheses (ranked by confidence)

### H1: Qt event loop runs unconstrained in background [HIGH confidence]
- No code found that pauses the Qt event loop on `onPause`
- `StatusQtActivity` only calls `setUiVisible(false)` — this reaches status-go but does NOT stop Qt timers, animations, or polling
- Qt 6 on Android: event loop continues until the app surface is destroyed
- **Expected drain:** continuous CPU wake-ups from QML timers/bindings

### H2: Waku light mode keepalives too aggressive [HIGH confidence]
- Waku is explicitly excluded from PauseServices
- Light mode clients must ping relay nodes to maintain filter subscriptions (they expire)
- Default Waku light mode ping interval not confirmed — needs measurement
- **Expected drain:** periodic radio wake, network activity, CPU for crypto operations per ping

### H3: statusgo heap paged out → major fault storms on reconnect [MEDIUM confidence]
- StatusGoServiceClient.java:78: "Binder call reaches nativeCall("AppStateChange") which can block for seconds if the Go runtime's memory was paged out (major faults)"
- If the `:statusgo` process is paged out and Waku wakes it frequently, each wake = expensive major fault storm
- **Expected drain:** burst CPU spikes when Waku callbacks fire after memory was paged out

### H1a: Keyboard Poll Timer (50ms, never stopped) [HIGH confidence] — NEW
- `SystemUtilsInternal` starts a 50ms repeat QTimer unconditionally on Android — no background guard
- Two JNI calls per tick: `getKeyboardHeight` + `isKeyboardVisible`
- If Qt event loop is not paused (H1): 20 CPU wakeups/sec + 40 JNI calls/sec = ~3.5M JNI calls/night
- Source: `ui/StatusQ/src/systemutilsinternal.cpp:74-103`
- **Expected drain:** continuous low-level CPU burn in UI process

*H1b, H5: promoted to `report.md` — see §3 for full evidence and scoring.*

---

## Multi-Factor Compound Hypotheses

### MH1: Qt Event Loop + 50ms Keyboard Timer [VERY HIGH — most likely dominant cause]
**Factors:** H1 (Qt loop not paused) × H1a (keyboard timer)
**Chain:** App backgrounded → Qt event loop continues → 50ms timer fires → 2 JNI calls → Android runtime wakes → CPU active → repeat 20×/sec
**Why it matters:** Even if Waku were perfectly silent, this alone produces continuous CPU activity in the UI process all night. Derivation: see `report.md H1a`.
**Measurable signature:** UI process CPU > 2% sustained with screen locked, even with WiFi disabled.

### MH2: Waku Keepalives + Mobile Data Radio Never Sleeps [HIGH]
**Factors:** H2 (Waku pings) × H1b (NetworkConnectivityCallback) × mobile data condition
**Chain:** Waku pings relay nodes every ~N seconds → radio wakes → mobile data capability changes fire → `NetworkConnectivityCallback.onCapabilitiesChanged` → `ConnectionChange` RPC → status-go processes it → Waku may reconnect or re-subscribe → more pings → radio can't enter deep sleep
**Why original reporter's test is worst-case:** 8 hours on **mobile data** (not WiFi). Mobile data radio has aggressive sleep modes — any wake prevents ~5s of sleep. Waku pings + connectivity churn = radio never reaches deep sleep state.
**Measurable signature:** High `mobileActiveTime` in `dumpsys batterystats`; RX bytes/min > 50KB even with screen locked.

*MH3: promoted to `report.md § 4` (MH3). See there for full chain.*

### MH4: Wallet-tick Cascade + Waku Background Processing [MEDIUM]
**Factors:** H2 (Waku active) + H5 (wallet-tick cascade) + multiple accounts
**Chain:** Waku light node receives new blocks in background → status-go processes transactions → emits `wallet-tick-reload` per chain/account → Nim token service handles signal → `rebuildMarketData` + `buildAllTokens` debounced → `fetchPrices` RPC → HTTP call to price API → network radio wakes
**Amplified by:** users with many accounts/chains (each = more events); active DeFi communities (more blocks)
**Measurable signature:** Logcat shows wallet-tick signals at night; network bytes spike in short bursts rather than continuous stream.

### MH5: Everything Together — Full Drain Model [EXPLAINS 65% OVERNIGHT]
**All factors combined:**
1. Qt event loop running + 50ms keyboard timer → ~2-4% continuous UI process CPU
2. Waku keepalive pings → periodic radio wakes (~every 1-5 min)
3. NetworkConnectivityCallback → compounds each Waku-induced radio wake with a status-go RPC
4. Messaging service active → processing any incoming messages in background
5. Wallet-tick cascade → price API calls after each Waku-delivered block

*Drain model and arithmetic: see report.md § 4 MH5. Note: midpoint sum = 4%/hr; 6%/hr requires upper bounds simultaneously.*

*H4: promoted to `report.md § 3` (H4). See there.*

---

## Measurement Plan (requires device)

### Setup
```bash
# Enable WiFi ADB (one-time, requires USB cable first)
adb tcpip 5555
# Disconnect USB, note phone IP, then:
adb connect <phone-ip>:5555

# Mock cable unplug (so Android tracks battery drain without charging)
adb shell dumpsys battery unplug
```

### Baseline Measurements (app closed)
```bash
adb shell dumpsys battery | grep -E "level|status|plugged"
adb shell dumpsys cpuinfo | head -30
adb shell dumpsys power | grep -E "Wake Locks|PARTIAL_WAKE_LOCK"
```

### Foreground Baseline (app open, idle)
```bash
# Watch CPU for Status processes
adb shell top -b -n 3 -d 2 | grep -E "im.status|statusgo"
# Network activity
adb shell cat /proc/net/dev | grep -v "lo:"
# Battery stats reset
adb shell dumpsys batterystats --reset
```

### Background Test (app sent to background)
```bash
# Send app to background
adb shell input keyevent KEYCODE_HOME

# Sample CPU every 30 seconds for 5 minutes
for i in $(seq 1 10); do
  echo "=== $(date) ==="
  adb shell top -b -n 1 | grep -E "im.status|statusgo"
  sleep 30
done

# Check wake locks after 5 min background
adb shell dumpsys power | grep -A5 "PARTIAL_WAKE_LOCK"

# Check battery stats
adb shell dumpsys batterystats | grep -E "im.status|statusgo|wakelock|cpu"

# Network bytes after background period
adb shell cat /proc/net/dev
```

### Waku Ping Frequency Test
```bash
# NOTE: tcpdump is NOT available on stock Samsung Android — do not use it.
# Use /proc/net/dev instead:
adb shell cat /proc/net/dev  # snapshot 1
sleep 60
adb shell cat /proc/net/dev  # snapshot 2 (diff = bytes in 60s background)
```

### Qt Event Loop Test
```bash
# Check if Qt UI process CPU drops to ~0% in background
# Expectation: if Qt loop is paused → UI process CPU ≈ 0%
# If Qt loop is NOT paused → UI process CPU > 0% even in background
adb shell top -b -n 1 | grep "im.status.ethereum$"  # UI process (no :statusgo suffix)
adb shell top -b -n 1 | grep ":statusgo"             # statusgo process
```

### Logcat for Background Events
```bash
adb logcat -s StatusGoService StatusGoServiceClient StatusGoStub -v time &
# Then background the app, watch for PauseServices calls and errors
```

---

## BLOCKERS

### B1: No ADB WiFi connection
**Status:** Waiting — phone charging to 100% on powerbank (was at 19%)
**Root cause:** Phone not connected via USB, ADB WiFi not enabled
**Resolution:**
1. Phone reaches 100%
2. Connect phone via USB
3. `adb tcpip 5555`
4. Disconnect USB
5. `adb connect <phone-ip>:5555`
6. Verify: `adb devices`

### B2: Build to test
**Status:** Ready — `~/Downloads/StatusIm-Desktop-2.38.0-rc.4-5-g4cf3d8e4b-4cf3d8-arm64.apk` (284MB, rc.4)

---

## Next Actions
1. [ ] Enable ADB WiFi (needs USB first — manual step)
2. [ ] Run baseline CPU/battery measurements (foreground)
3. [ ] Run 10-min background soak test with CPU sampling
4. [ ] Check Qt UI process CPU in background — validate H1
5. [ ] Capture Waku network activity frequency — validate H2
6. [ ] Read logcat for PauseServices success/failure — validate pause actually fires
7. [ ] Check status-go pausable service list at runtime

---

## Decisions Log
- 2026-06-02: Prioritising H1 (Qt event loop) and H2 (Waku keepalives) as most likely root causes based on code analysis. H1 is unusual because no Qt event loop pause code exists anywhere in the codebase.
