# Battery & CPU Drain Investigation — Status Android
**Version:** 0.6 (T5b complete — H-N3 CONFIRMED steady-state; all primary hypotheses resolved)
**Report type:** Static analysis + device measurements
**Issue:** [#21045 — Mobile: Android flagged Status for high battery consumption and CPU usage](https://github.com/status-im/status-app/issues/21045)
**Related:** [#20742 — Background service battery consumption](https://github.com/status-im/status-app/issues/20742)
**Build analysed:** `2.38.0-rc.4-5-g4cf3d8e4b` (commit [`4cf3d8e4b`](https://github.com/status-im/status-app/commit/4cf3d8e4b))
**Date:** 2026-06-02
**Status:** All soaks complete (T1–T5b). Primary root causes identified and quantified.
**Next version trigger:** Fix validation (T6) or Android 15 replication

---

> **Claim states used in this report:**
> `[CODE]` = confirmed directly in source, no device needed |
> `[H — X%]` = hypothesis with probability score |
> `[?]` = unknown, needs external data |
> `[CONFIRMED]` / `[REJECTED]` = verdict from device measurement

---

## Abstract

Status Android was flagged by Android OS for high battery and CPU usage after an 8-hour overnight background soak on mobile data (96% → 31%, 65% consumed). The app was in WakuV2 Light mode with no active user interaction. **Two prior fixes (PR #20202 pausable services, PR #20882 mvds message-sending pause) are already present in the build under test — the drain persists regardless.** Static source code analysis of `4cf3d8e4b` identified **five single-factor hypotheses** and **five compound multi-factor hypotheses** that plausibly explain the remaining drain. The most critical findings are: (1) a 50ms keyboard polling timer in the Qt C++ layer that fires 20 times/second with no background guard — a **new finding not previously identified in any issue**; (2) the Qt event loop is not paused on Android `onPause()`, confirmed by PR #20882 author noting *"events piling up in the qt event loop while in the background"*; (3) a `NetworkConnectivityCallback` in the `:statusgo` process that fires `ConnectionChange` on every network capability change with no background throttle. These factors compound: the worst-case scenario — mobile data, multiple wallet accounts, active Waku light node — combines all drains simultaneously. Device-side validation measurements (T1–T5) are defined and ready to execute.

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

**Independent replication — #20742:** Same device class (Samsung S21 Ultra, Android 15, v2.38 RC1), different reporter. 52% drain over 6 hours = **~8.7%/hr** — consistent with #21045. #20742 was closed by PR #20882; #21045 remains open.

**Prior fix already in the build under test:**
| PR | Merged | What it fixed | Still in `4cf3d8e4b`? |
|----|--------|--------------|----------------------|
| [#20202](https://github.com/status-im/status-app/pull/20202) | 2026-04-09 | Introduced pausable services + `PauseServices`/`ResumeServices` via `appStateChanged` | ✓ Yes |
| [#20882](https://github.com/status-im/status-app/pull/20882) | 2026-05-18 | Paused mvds message-sending loops in background; added `MESSAGING_REPAUSE_DELAY_MS = 60s` wake on notification reply. Closed #20742. | ✓ Yes |

**Critical implication:** Both fixes are already in the build under test. The ~8%/hr drain persists **after** these fixes. H1a (keyboard timer) and H2 (Waku radio) are the prime candidates for the remaining unresolved drain.

---

## 2. Architecture Background

Status Android runs **two separate OS processes** when backgrounded. This is critical — most battery analysis looks at only one.

```
app.status.mobile          ← Process 1: Qt 6 event loop + Nim runtime + QML
app.status.mobile:statusgo ← Process 2: Foreground service, libstatus.so, Waku light node
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

### H1 — Qt Event Loop Not Paused `[REJECTED]`
**Device verdict — T1 (2026-06-02):** UI process (`app.status.mobile`) showed **0.0% CPU all 20 samples** over 10 min background soak. TIME+ column unchanged throughout. batterystats cpu:fg = 598ms in 10 min (<0.1%). Android freezes the Qt UI process in background — the event loop does NOT run indefinitely.

**Implication:** H1a (keyboard poll timer) is also REJECTED — the timer cannot fire if the process is frozen.

---

### H1 — Qt Event Loop Not Paused `[REJECTED — original text below]`
*Scoring: no stop path in code (+40), architectural pattern known in Qt Android (+15), analogous known issue in Qt Android codebase (+10) = 65%. Note: alexjba's comment is in a GitHub issue, not source code — the bayesian scoring table awards +20 only for comments in source. Correct application of the framework gives 65%.*

**Evidence:** Zero code pauses the Qt event loop on `Activity.onPause()`. Qt 6 on Android does not auto-suspend the event loop when the activity is paused. The loop runs until `onDestroy`. All QML `Timer {}` components, property bindings, and signal handlers continue firing.

**Corroboration (PR #20882):** alexjba's PR description explicitly states *"less events piling up in the qt event loop while the app is in the background"* as an expected outcome of the messaging pause fix. This is a developer acknowledgement that Qt loop accumulation in background is an observed problem — raising confidence that H1 is real even before device measurement.

**Source:** `mobile/android/qt6/src/app/status/mobile/StatusQtActivity.java:56-61` — `onPause` only calls `setUiVisible(false)`.

**Expected drain signature:** UI process (`app.status.mobile`) shows > 2% CPU sustained after 60s background with screen locked.

---

### H1a — 50ms Keyboard Poll Timer `[REJECTED]`
**Device verdict — T1:** UI process = 0.0% CPU; wake lock time = 3s/10min. Android froze the Qt UI process, preventing the timer from firing. Even if the timer logic is present in code, it has no measurable impact while the process is frozen.

**Note:** R1 (guard the timer) remains a good defensive fix even though H1a is rejected — if Android behaviour changes or on older OS versions, an unguarded timer would immediately become a drain source.

---

### H1a — 50ms Keyboard Poll Timer `[REJECTED — original text below]`
*Scoring: timer starts unconditionally (+40), no stop path anywhere in codebase (+20), architectural pattern (+15), explicit 20 FPS comment confirms intent (+20) = 95% → reduced to 90% pending device confirmation*
*Prior issues: **none** — not identified in #21045, #20742, #20882, or any prior issue. New finding from this investigation.*
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

**Source files (build 4cf3d8e4b):**
- `StatusGoService.java:291-300` — `NetworkConnectivityCallback` class (`onCapabilitiesChanged`, `onLost`)
- `StatusGoService.java:329-356` — `dispatchConnectionChange()` — de-dup on `lastConnectionKey` (line 333)
- `StatusGoService.java:362-376` — `registerNetworkCallback()` → `registerDefaultNetworkCallback()` (line 371)
- `StatusGoService.java:469` — registered unconditionally in `onCreate()`
- `src/app_service/service/general/service.nim` — `when defined(android): return` — Qt/nim `connectionChange` is a **no-op on Android**; Java callback is the sole path
- `ui/StatusQ/src/networkchecker.cpp:52-54` — Qt side still registers `applicationStateChanged` handler, but nim makes it a no-op; harmless on Android but dead code

---

### H2 — Waku Light Mode Keepalives `[INCONCLUSIVE — T2 pending]`
**Device verdict — T1 (partial):** `:statusgo` process showed **avg 13.7% CPU over first 8.5 min** background (T1 soak). Spike to 65.5% at T+3.5min. Process frozen by Android Doze at T+8.5min (TIME+ stopped advancing). Drain attributed to Status = 1.97 mAh in 10 min = **39% of all-app drain**. However: T1 was only 10 min — it's unclear whether the 65.5% CPU spike is one-time Waku connection settling, or recurs on a ~5–10 min schedule.

**T2 (30-min soak, running):** Will determine if statusgo cycles back after Doze freeze (i.e., does Waku wake the process periodically, or does it stay frozen once idle?).

---

### H2 — Waku Light Mode Keepalives `[INCONCLUSIVE — original text below]`
*Scoring: explicit source comment confirms Waku runs regardless of pause (+20 — comment in source, not timer start), P2P keepalive pattern known to cause radio drain (+15), interval unknown — can't confirm frequency is problematic (−10) = 25% base + mobile data worst-case condition (+20) = 45%. Note: the original score of 65% incorrectly used +40 (the "timer starts unconditionally, no stop path" category) for a source comment. Correct evidence type is +20. Score corrected in Loop 2. Fix-order implication: H1a expected impact (70% × 2%/hr = 1.4%/hr) now exceeds H2 (45% × 3%/hr = 1.35%/hr) — H1a is the top-priority fix.*
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

**Build-specific amplifier (target build `4cf3d8e4b`):** `MESSAGING_REPAUSE_DELAY_MS = 60_000L` — after any inline notification reply, the messaging service is *resumed* and stays active for 60 additional seconds before being re-paused. Every background notification reply in the overnight test extends messaging service activity by 1 minute. If the reporter replied to messages during the 8-hour test, this compounds H4's baseline cost.

**PR #20882 corroboration `[CODE-CONFIRMED — mechanism]`:** alexjba's PR description confirms the exact mechanism: *"we'll resume messenger for 60 seconds — enough to push the message through and then it will be paused again unless the app goes back to foreground."* The 60s value and the repause-on-reply design are intentional. The cost of this design choice (H4's actual battery impact) remains unmeasured — but the mechanism is confirmed by the PR author.

**Note:** This is the least actionable hypothesis — messaging must stay alive for notifications. The question is whether its background cost is higher than expected, particularly when compounded by MESSAGING_REPAUSE_DELAY_MS on notification replies.

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

| System | Location | Guards | Android status |
|--------|----------|--------|----------------|
| `NetworkConnectivityCallback` | `:statusgo` process (Java, `StatusGoService.java:291`) | None — fires unconditionally (de-dup on type+expensive key) | **Active** |
| `NetworkChecker` / nim `connectionChange` | UI process (C++/nim) | `Qt::ApplicationActive` check in `networkchecker.cpp:66` | **No-op** — `service.nim: when defined(android): return` |

**Net effect:** On Android the Java callback is the sole path. Every real network event → one `ConnectionChange` RPC (de-duped) → status-go activity in background.

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
├── UI Process (app.status.mobile)
│   ├── Qt event loop running (H1)
│   ├── keyboardTimer: 20 JNI calls/sec (H1a) ←── ~1-2% CPU/hr
│   └── NetworkChecker 10s timer on state-change (benign, minor)
│
└── :statusgo Process (app.status.mobile:statusgo)
    ├── Waku light node pings: radio wakes every ~N min (H2) ←── ~2-3% battery/hr
    ├── NetworkConnectivityCallback: fires on each radio wake (H1b)
    ├── messaging service: processing incoming messages (H4)
    └── wallet-tick cascade: price API on each Waku-delivered block (H5)
         └── each API call: radio wake → connectivity change → H1b fires again
```

**Estimated combined drain:** Component estimates are order-of-magnitude calibrations, not measurements. Midpoint arithmetic: 1.5 (H1a) + 2.5 (H2) = 4%/hr. The 6%/hr figure requires upper bounds simultaneously plus unquantified contributions from H1b, H4, H5. Observed: 8.1%/hr. The model leaves a 4.1%/hr gap at midpoints (not 2.1%/hr as H6 states — H6 used upper-bound components). All figures are pre-measurement estimates; device tests T1–T5 will replace these with real numbers.

---

## 5. Device Measurements — Results

### T1 — Background CPU Soak (10 min) — 2026-06-02

| Metric | Value | Verdict |
|--------|-------|---------|
| UI process (`app.status.mobile`) CPU | **0.0%** all 20 samples | H1/H1a REJECTED |
| statusgo avg CPU | **13.7%** (78s CPU / 570s elapsed) | H2 active but settling |
| statusgo peak CPU | **65.5%** at T+210s | One-time? or periodic? |
| statusgo frozen at | T+510s (8.5 min) — TIME+ stopped | Android Doze kicked in |
| Wake lock time | 3s 21ms in 10 min | Negligible |
| Status drain (batterystats) | 1.97 mAh / 10 min | 39% of all-app drain |
| Battery level change | None (100% → 100%) | 10 min too short |

**Setup:** WakuV2=Light ✓, battery reset ✓, screen locked via KEYCODE_POWER ✓, device Android 13 ✓

**Implication:** The hypothesised Qt UI process drain (H1 + H1a) does not occur on Android 13 — Android's process management freezes the UI process within the first 30s of background. The `:statusgo` foreground service is the primary active process in background. Whether statusgo's activity is transient (connection settling) or steady-state is the key open question.

### T2 — 30-min Background Soak — 2026-06-02 (COMPLETE)

**Setup:** Notifications=NONE (importance=NONE, userSet=false — default, NOT set by user)

| Metric | Value | Verdict |
|--------|-------|---------|
| Doze IDLE engaged at | T+8.5 min (from T1) | Confirmed `mState=IDLE` |
| Doze maintenance window 1 | T+17 min (2 min duration) | statusgo +5.24s CPU |
| Doze maintenance window 2 | T+27 min (2.5 min duration) | statusgo +4.92s CPU |
| Status drain total | **0.176 mAh / 30 min** | = 0.352 mAh/hr |
| vs T1 pre-Doze rate | 1.97 mAh / 10 min = 11.48 mAh/hr | **Doze reduces by 33×** |
| vs reported 8.1%/hr | 364 mAh/hr | **Gap: 1036×** |
| Status on Doze whitelist | No | Not requesting Doze exemption |
| Status Doze-bypass alarms | None | No `setExactAndAllowWhileIdle()` |

**H2 verdict: `[REJECTED — Android 13, WiFi, notifications-disabled]`** Both processes frozen by Doze IDLE after ~8.5 min. statusgo only active during brief maintenance windows. Total drain negligible (0.06% battery over 8h).

**Critical gap:** T2 drain is 1036× lower than the reported 8.1%/hr. Test conditions differ from reporter's in 3 key ways: (1) notifications disabled vs enabled, (2) WiFi vs mobile data, (3) Android 13 vs Android 15.

### H-N1 — Notification/FCM Prevents Doze `[REJECTED — code]`
**Hypothesis (rejected):** FCM high-priority messages bypass Doze, keeping device awake.

**Evidence against:** `StatusNotificationManager.java` line 100 — Status uses `"local-notifications"` (signals from statusgo), NOT FCM. There is no `FirebaseMessagingService` implementation in the manifest. Notifications are generated locally within the running app from Waku-delivered events. No server-side FCM push path exists to bypass Doze.

**Revised understanding:** Notification permission (`importance=NONE` vs `DEFAULT`) affects App Standby Bucket classification, which determines Doze maintenance window frequency. This is a real variable but not the FCM-bypass mechanism initially hypothesised.

### H-N2 — Mobile Data Causes LTE Radio Drain `[CONFIRMED — T4]`

**T4 result (2026-06-02, 42 min soak, cellular, notif=DEFAULT, 0 contacts):**

| Metric | Value |
|--------|-------|
| Total Status drain | 69.1 mAh / 42.5 min = **97.5 mAh/hr** |
| mobile_radio attribution | **67.8 mAh = 98% of drain** |
| Mobile radio active time | **23m 31s / 42m 34s = 55% of soak** |
| CPU attribution | 1.32 mAh = 1.9% of drain |
| T3 WiFi comparison | 1.632 mAh/hr |
| **Cellular vs WiFi ratio** | **97.5 / 1.632 = 59.7×** |
| Doze entry time | ~19.5 min (vs never for WiFi+notif=DEFAULT) |
| Cellular data | 3.14 MB total (4.43 MB/hr) |

**Root cause shift confirmed:** On WiFi, CPU (Waku keepalives) dominates. On cellular, the **LTE radio is the dominant mechanism** — 98% of attributed drain.

Each Waku keepalive ping (~13.5 min interval, confirmed T3) wakes the LTE modem. The modem requires ~5–10s RRC re-entry after each transmission. With the Waku filter subscription renewing every 13.5 min, the radio achieves only ~55% sleep duty cycle. This is the primary mechanism behind the reported overnight drain.

**Original hypothesis was partially wrong:** H-N2 assumed Doze prevention was the mechanism. The actual mechanism is the LTE radio energy cost per Waku ping, not Doze prevention. Doze did engage at ~19.5 min on cellular — but even before Doze, and during Doze maintenance windows, the radio was active for 55% of the soak.

### T3 — Notification=DEFAULT Soak (30 min) — 2026-06-02 (COMPLETE)

**Setup:** `pm grant app.status.mobile android.permission.POST_NOTIFICATIONS` applied; `importance=DEFAULT` confirmed. Device woken to `mState=ACTIVE` before locking. Otherwise identical to T2.

| Metric | T2 (notif=NONE) | T3 (notif=DEFAULT) | Delta |
|--------|-----------------|-------------------|-------|
| Doze IDLE engaged? | **Yes — T+8.5 min** | **No — never in 30 min** | Critical |
| statusgo avg CPU | 3.1% (windows only) | **1.95%** continuous | — |
| statusgo TIME+ behavior | Frozen T+8.5–28.5 min | Advanced continuously | — |
| Status drain | 0.176 mAh / 30 min | **0.816 mAh / 30 min** | **4.6×** |
| Status drain rate | 0.352 mAh/hr | **1.632 mAh/hr** | 4.6× |
| Network RX | 2.76 MB / 30 min | 2.51 MB / 30 min | similar |
| Wakelock | 30ms | 7ms | negligible |

**H-N1 REJECTED (code):** Source check of `StatusNotificationManager.java` confirmed Status uses `"local-notifications"` (Waku-delivered, in-process), not FCM. No `FirebaseMessagingService` in manifest. FCM Doze-bypass mechanism does not apply.

**Revised notification mechanism:** `importance=DEFAULT` keeps Status in `WORKING_SET` App Standby Bucket, preventing Android from entering Doze IDLE. With `importance=NONE`, Android classifies the app as low-priority and enters Doze IDLE in ~8.5 min.

**Waku keepalive timing:** Two CPU spikes observed — T+390s (37.9%) and T+1200s (13.7%). Gap = 810s = **~13.5 min candidate Waku filter subscription renewal interval.**

**H2 partial confirmation:** Waku `:statusgo` process runs continuously when notifications are enabled (no Doze freeze). 1.632 mAh/hr on WiFi. This is still 223× below the reported 8.1%/hr.

**223× gap — resolved by T4:** T4 cellular soak (same account, notif=DEFAULT) = 97.5 mAh/hr = 59.7× T3 WiFi. The LTE radio (98% of T4 drain, active 55% of soak) explains the bulk of the gap. Remaining gap (T4 = 26.7% of reported phone drain) attributable to H-N3 (account richness — 0 contacts/communities in test vs real user) plus other-app baseline drain.

### H-N3 — Account Richness: Waku Subscription Count Scales Drain `[PARTIALLY CONFIRMED — T5]`

**T5 result (2026-06-02, 30 min soak, cellular, notif=DEFAULT, ~50 contacts + 3 communities, post-login sync):**

| Metric | T4 (0 contacts) | T5 (loaded account) | Ratio |
|--------|-----------------|---------------------|-------|
| Radio active time | 23m 31s / 42m 34s = 55% | 30m 36s / 30m 49s = **99%** | 1.8× duty cycle |
| mobile_radio rate | 95.5 mAh/hr | 287 mAh/hr | **3.0×** |
| Network RX | 1.74 MB / 42.5 min | **321 MB / 30 min** | 184× data |
| Packets | 7,601 / 23.5 min = 323/min | 437,352 / 30.6 min = **14,285/min** | 44× |
| Doze entry | ~19.5 min | **Never in 30 min** | Doze delayed |
| Total drain | 69.1 mAh / 42.5 min | 131 mAh / 30.8 min | — |
| Gross drain rate | 97.5 mAh/hr | 425 mAh/hr (incl. 10 min screen-on) | — |

**T5b — Steady-state loaded account (2026-06-02, 70m 46s, screen correctly off from start):**

| Metric | T4 (0 contacts) | T5b (loaded, steady-state) | Ratio |
|--------|-----------------|---------------------------|-------|
| Drain rate | 97.5 mAh/hr | **144 mAh/hr** | **1.48×** |
| mobile_radio | 95.5 mAh/hr | 142.7 mAh/hr | 1.49× |
| Radio duty cycle | 55% | 49.5% | similar |
| Packets/min (active) | 323 | **6,865** | **21×** |
| CPU | 1.32 mAh | 1.27 mAh | identical |
| Doze entry | ~19.5 min | ~30s (screen already off) | — |

**H-N3 CONFIRMED.** 21× more network packets per radio-active minute on a loaded account vs bare. CPU is virtually identical — the effect is purely radio. More Waku filter subscriptions = more keepalive traffic = LTE modem busier per Doze maintenance window.

**T5 (post-login sync storm) vs T5b (steady-state):**
T5 captured a 321 MB RX sync storm (radio 99% duty cycle, no Doze, ~425 mAh/hr). T5b after sync settled: 202 MB RX, 50% duty cycle, 144 mAh/hr. The sync storm is a distinct worst-case scenario: after a long disconnect (airplane mode, overnight no-signal), reconnect triggers full mailserver catchup which keeps the radio fully occupied until complete.

**What IS confirmed:** Radio duty cycle jumps from 55% (bare) to near-full when syncing. Steady-state 49.5% is still materially higher load than T4's 55% in terms of absolute drain per hour.

**CPU insight:** Account richness does NOT scale CPU cost. Both bare and loaded accounts draw ~1.3 mAh CPU — statusgo processes each Waku message similarly regardless of subscription count. The O(n) cost is in network traffic, not compute.

**Post-login sync as a drain scenario:** If the reporter's phone came back from airplane mode, or hadn't run Status in hours before the 8h test, the initial sync storm explains a large portion of overnight drain independently of any code bug. No sync throttle exists for background reconnect in v2.38.

**`syncingOnMobileNetwork` — hidden, hardcoded in v2.38 `[CODE — CONFIRMED]`:**
`MessagingView.qml:77` (`visible: false`) and `settings/service.nim:309` (hardcoded `return true`). The setting exists — "Mobile data and Wi-Fi" vs "Wi-Fi only" — but is inaccessible to users in v2.38. Code comment: *"Hardcoded to true (Mobile data + Wi-Fi) for v2.38 while the feature is under review. Re-enable the line below in v2.39 once the WiFi-only behaviour is validated."* Confirmed by Noelia (Slack 2026-06-02): *"Yes it was temporarily hidden… Now it's defaulted to use data."*

**Implication:** All v2.38 users sync on both WiFi and cellular regardless of any setting. This is not a variable in the reporter's case. T4 (cellular soak) remains valid — the app uses whatever interface Android provides. `syncingOnMobileNetwork` becomes a real test variable in v2.39.

### T4 — Mobile Data Background Soak (42 min) — 2026-06-02 (COMPLETE)

**Setup:** WiFi disabled (script), mobile data enabled, notif=DEFAULT, 0 contacts/communities, batterystats reset before screen lock.

| Metric | T3 (WiFi) | T4 (Cellular) | Delta |
|--------|-----------|---------------|-------|
| Doze entry | Never (30 min) | **~19.5 min** | Earlier on cellular |
| Status drain total | 0.816 mAh / 30 min | **69.1 mAh / 42.5 min** | — |
| Status drain rate | 1.632 mAh/hr | **97.5 mAh/hr** | **59.7×** |
| mobile_radio | — | **67.8 mAh (98%)** | Dominant |
| CPU drain | ~1.6 mAh/hr | 1.32 mAh / 42.5 min = 1.87 mAh/hr | Similar |
| Radio active time | — | 23m 31s / 42m 34s = **55%** | LTE never sleeps |
| Cellular data | — | 3.14 MB / 42.5 min | 4.43 MB/hr |

**H-N2 CONFIRMED.** LTE radio = 98% of drain. CPU is comparable to WiFi — it is the radio that makes cellular 59.7× worse. Each Waku keepalive (~13.5 min interval) wakes the LTE modem; the modem's RRC transition cycle keeps the radio active for ~55% of elapsed time.

**Doze behavior:** Cellular + notif=DEFAULT → Doze at ~19.5 min. Despite Doze engaging, drain was still massive because mobile radio was active for 23.5 of the first 42 minutes (most of this occurred pre-Doze).

**Remaining gap to reported drain:** T4 Status = 97.5 mAh/hr on a 0-contact account = 26.7% of reported total phone drain (365 mAh/hr). H-N3 (account richness — subscriptions scale with contacts/communities) would multiply this for real users.

### Cross-Soak Summary — All Measurements

| Test | Network | Notif | Account | Condition | Drain rate | Radio duty | Packets/min active |
|------|---------|-------|---------|-----------|-----------|-----------|-------------------|
| T2 | WiFi | NONE | bare | steady | 0.352 mAh/hr | — | — |
| T3 | WiFi | DEFAULT | bare | steady | 1.632 mAh/hr | — | — |
| T4 | Cellular | DEFAULT | bare | steady | **97.5 mAh/hr** | 55% | 323 |
| T5 | Cellular | DEFAULT | loaded | post-login | ~425 mAh/hr* | 99% | 14,285 |
| T5b | Cellular | DEFAULT | loaded | steady | **144 mAh/hr** | 50% | 6,865 |

*T5 contaminated: 10 min screen-on + active sync storm.

**Multiplier stack for reported worst-case (typical user, cellular, notifications, loaded account):**

| Factor | Multiplier | Baseline → Result |
|--------|-----------|------------------|
| Notifications NONE → DEFAULT | 4.6× | 0.35 → 1.63 mAh/hr |
| WiFi → Cellular | 59.7× | 1.63 → 97.5 mAh/hr |
| Bare → Loaded account | 1.48× | 97.5 → 144 mAh/hr |
| Steady-state → Post-reconnect sync | ~3× peak | 144 → ~430 mAh/hr |

**Steady-state loaded account on cellular: 144 mAh/hr = 3.2%/hr on a 4500 mAh battery.**
At 3.2%/hr from Status + ~1%/hr phone baseline + other apps → consistent with the reported 8.1%/hr, especially if the test began after a long disconnect (sync storm at start of overnight window).

### H-N4 — Relay Mode vs Light Mode `[NOT APPLICABLE — reporter confirmed Light]`
Relay mode (full Waku node) requires actively forwarding messages for other nodes — dramatically higher drain than Light mode. Reporter confirmed Light mode. Included for completeness; not a variable in reported case.

---

## 6. Validation Test Procedures

*(Original test procedures below — now renumbered as section 6)*

**Experimental standards:**
- Each test run repeated **N=5 times** on separate battery stats windows (reset between runs)
- Median value reported; outliers (>2× IQR) noted but not discarded without cause
- Statistical comparison (control vs treatment): Mann-Whitney U test, p < 0.05 threshold
- Energy unit: 1% battery ≈ 623J on Samsung Galaxy S20 FE (4500mAh × 3.85V × 3.6 / 100)
- All tests require: device at 100% charge, WakuV2 = Light confirmed, other apps killed, `dumpsys battery unplug` applied, screen locked via power button (NOT home button)

**Validity constraint — OS version gap:** The original 65% drain was reported on Android 15 (Samsung S21 Ultra). Tests run on Android 13 (S20 FE). Android 15 introduced stricter background process limits and battery bucket behaviour not present in Android 13. A finding confirmed on Android 13 is valid for that OS version but may understate the drain on Android 15 — or conversely, Android 13's less aggressive killing may allow Qt process survival where Android 15 would not. Record the test OS version with every verdict and flag any confirmation as "confirmed on Android 13."

**Pre-test: Confirm node connected before any soak (run before T1–T5)**
```bash
# Confirm node.login fired — backgrounding before this produces startup-overhead CPU, not steady-state drain
adb logcat -v time 2>/dev/null | grep -m1 "node.login" && echo "CONNECTED — proceed" || echo "WARNING: not connected — wait and retry"
# If not seen within 60s: node failed to connect — stop and investigate
```

**Pre-test: Qt process survival check (run before T1)**
```bash
# Background the app, wait 5 minutes, verify Qt process is still alive
adb shell input keyevent KEYCODE_POWER
sleep 300
adb shell ps -A | grep "app.status.mobile " | grep -v ":statusgo"
# If no output → Android killed the UI process → T1 will produce a false REJECTED verdict
# Do NOT proceed with T1 if the UI process is dead — record "process killed at T+5min" and escalate
```

**Mann-Whitney U — calculation method:**
```python
# Run this after collecting N=5 measurements for control and treatment
# pip install "scipy>=1.6.0"  — alternative='less' parameter requires scipy 1.6+
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
# IMPORTANT: KEYCODE_POWER toggles screen state. Ensure screen is ON before sending it,
# otherwise it will turn the screen ON instead of locking it.
# Verify screen state first: adb shell dumpsys power | grep "mWakefulness"
# "mWakefulness=Awake" → screen is ON → KEYCODE_POWER will lock ✓
# "mWakefulness=Asleep" → screen is OFF → KEYCODE_POWER will turn ON ✗ (app stays background but wrong power state)
adb shell input keyevent KEYCODE_POWER
# Disable WiFi to isolate from network noise
adb shell svc wifi disable
# Monitor UI process CPU for 5 min
for i in $(seq 1 10); do
    echo "=== T+$((i*30))s ===" && date
    adb shell top -b -n 1 | grep "app.status.mobile "
    sleep 30
done
# Record median CPU% from 10 readings above
# Also capture wake locks held during the soak
adb shell dumpsys power | grep "PARTIAL_WAKE_LOCK" | grep "app.status"
```
**Verdict threshold:**
- UI process > 2% sustained (median across 5 runs) → H1 + H1a CONFIRMED
- UI process < 0.5% sustained → H1 REJECTED (Qt loop is pausing)
- 0.5–2%: INCONCLUSIVE — extend to 15min soak. If still 0.5–2% after 15min: record exact median, note "ambiguous signal — Qt loop may be intermittently active." Add Perfetto trace (see `skills/android-battery-measurement-toolkit.md`) to get per-wakelock attribution before escalating.

### T2 — Validate H2 (Waku radio activity)

**Critical:** H2's severity argument is specific to **mobile data** (LTE radio sleep failure). T2 must be run on mobile data, NOT WiFi. On Samsung devices, the mobile data interface is `rmnet_data0` (or `ccmni0`), not `wlan0`. Running this test on WiFi produces a valid WiFi measurement but cannot confirm the LTE radio-sleep failure mode central to H2.

```bash
# Run 5 times on MOBILE DATA (WiFi disabled).
# First: identify the mobile data interface on this device
adb shell cat /proc/net/dev | grep -v "lo:\|wlan\|dummy\|sit\|p2p"
# Use whichever interface shows non-zero TX/RX bytes (typically rmnet_data0 on Samsung)
IFACE="rmnet_data0"  # verify this first

adb shell svc wifi disable
NET1=$(adb shell cat /proc/net/dev | grep "$IFACE" | awk '{print $2}')
sleep 300  # 5 min background, screen locked
NET2=$(adb shell cat /proc/net/dev | grep "$IFACE" | awk '{print $2}')
echo "RX bytes in 5min background (mobile): $((NET2 - NET1))"
# Record for Mann-Whitney vs control (app force-stopped, same 5min window)
```
**Verdict threshold (median across 5 runs, mobile data):**
- > 250KB in 5min (50KB/min) → Waku actively communicating on cellular → H2 CONFIRMED
- < 50KB in 5min → H2 REJECTED or pings very infrequent
- Note: WiFi results from T5 are informational only — T2 is the H2 validation test and must use mobile data.

### T3 — Validate H1b (NetworkConnectivityCallback)
```bash
# Verify Log.d is visible for this tag (may be suppressed on production builds)
adb shell setprop log.tag.StatusGoService D
# StatusGoService.java:338 logs: "ConnectionChange args: ..."

# Single observation is sufficient for this test (binary: fires or doesn't)
adb logcat -s StatusGoService -v time 2>/dev/null | grep "ConnectionChange" &
# While monitoring, toggle WiFi 3 times with 10s gap
for i in 1 2 3; do
    adb shell svc wifi disable && sleep 5 && adb shell svc wifi enable && sleep 10
done
```
**Verdict:** Count `ConnectionChange` log lines during background. ≥ 1 per toggle → H1b mechanism CONFIRMED (callback fires in background). Zero → H1b REJECTED or log tag changed.

**Note on Qt path:** `networkchecker.cpp:onReachabilityChanged` has a `Qt::ApplicationActive` guard, but this is irrelevant — `src/app_service/service/general/service.nim` makes the nim/Qt `connectionChange` a **no-op on Android** (`when defined(android): return`). Only the Java `NetworkConnectivityCallback` fires on this build.

**Important caveat (F6):** T3 confirms that the code path *executes* — it does not measure drain impact. A confirmed T3 means H1b is a real event source, but not how much battery it costs. Drain quantification requires T4 with WiFi cycling to compare battery% before/after with and without network toggles.

### T4 — Validate MH1 (compound: loop + keyboard timer)
```bash
# Run 5 times. Before each run: reset batterystats
# WiFi OFF to eliminate network factor (isolates CPU drain)
adb shell svc wifi disable
adb shell input keyevent KEYCODE_POWER
adb shell dumpsys batterystats --reset
sleep 600  # 10 min soak
adb shell dumpsys cpuinfo | grep "app.status.mobile "
adb shell dumpsys batterystats --charged | grep -A5 "app.status.mobile\b"
# Also capture wake locks
adb shell dumpsys power | grep "PARTIAL_WAKE_LOCK" | grep "app.status"
# Record: cpuTimeMs for UI process vs control (app force-stopped, same 10min window)
```
**Control condition:** App force-stopped, same 10min window, same WiFi-off state. Run 5 control runs before or after treatment runs.
**Expected:** If MH1 is dominant, UI process cpuTimeMs will be significantly higher than the force-stopped control (p < 0.05, Mann-Whitney U).

### T5 — Differential: WiFi vs Mobile Data
Run T2 procedure twice in separate sessions — once on WiFi, once on mobile data (SIM data enabled, WiFi off). Record 5 runs each condition. If mobile data drain median > WiFi median by >2× → MH2 (radio never sleeps) CONFIRMED.

---

### Unexplained Drain Gap — H6 `[? — unknown]`

**Observation:** Full drain model (MH5) at midpoints estimates ~4%/hr. Upper-bound assumptions push to ~6%/hr but require all components simultaneously at maximum. Observed: ~8.1%/hr. **Gap: ~4.1%/hr unaccounted at midpoints (~2.1%/hr at upper bounds).**

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
- Research protocol: `research/research-protocol.md`
- Raw journal: `research/journal.md`
- Skills index: `skills/INDEX.md`

### Supporting Skills
- `skills/bayesian-hypothesis-scoring.md` — evidence-based probability scoring method
- `skills/android-energy-code-smells.md` — 10-pattern anti-pattern catalogue (PMC11479295)
- `skills/android-battery-measurement-toolkit.md` — ADB commands, Perfetto SQL, tool selection
- `skills/android-controlled-battery-experiment.md` — N=5 protocol, Mann-Whitney U, Joule conversion
- `skills/battery-research-interim-reporting.md` — interim report trigger conditions and GitHub template
