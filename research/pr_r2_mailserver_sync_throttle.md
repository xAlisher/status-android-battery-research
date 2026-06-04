# PR Draft — R2: Defer mailserver history sync when app is in background

**Target repo:** `status-im/status-go` (primary) + `status-im/status-app` (optional signal)
**Related issue:** status-im/status-app#21045
**Measured evidence:** `xAlisher/status-android-battery-research` (T5 soak)

---

## Problem

When Status reconnects after a period of no connectivity (airplane mode, dead zone, phone off overnight), the mailserver service immediately requests full message history for all active chats via `requestAllHistoricMessagesWithRetries`. This catchup sync runs regardless of whether the app UI is visible.

Measured in T5 (30 min soak, loaded account, post-reconnect):
- **321 MB RX in 30 min** (avg 10.7 MB/min)
- Radio active **99% of soak duration** (vs 55% in steady state)
- Estimated drain rate: **~425 mAh/hr** during sync peak

The sync storm is invisible to the user if the app is backgrounded — they get no benefit until they open the app, but the battery cost is paid immediately. A user who has airplane mode overnight, then restores connectivity before sleeping, will drain significant battery before ever opening Status again.

---

## Proposed change

### status-go (primary fix)

The history request is triggered inside status-go when the active mailserver changes or reconnects. The fix adds a background mode check before executing `requestAllHistoricMessagesWithRetries`:

```go
// In the mailserver reconnect / active mailserver changed handler:

if api.service.backgroundMode {
    // Schedule deferred sync for when the UI returns to foreground.
    // Mark a flag so onForeground() will trigger the sync.
    api.service.pendingHistorySync = true
    log.Info("app is backgrounded — deferring mailserver history sync")
    return
}

// Proceed with immediate sync (foreground case, unchanged behavior).
api.service.requestAllHistoricMessagesWithRetries(false)
```

Add a corresponding `onForeground` hook that runs the deferred sync when `SetWakuBackgroundMode(false)` is called (or a new `SetAppForeground` RPC):

```go
func (s *Service) onForeground() {
    if s.pendingHistorySync {
        s.pendingHistorySync = false
        go s.requestAllHistoricMessagesWithRetries(false)
    }
}
```

The `backgroundMode` flag is set/cleared by the same RPC proposed in R1 (`SetWakuBackgroundMode`), or a dedicated `SetAppForeground(bool)` RPC. If both R1 and R2 are implemented together, they share a single `SetAppForeground` call to avoid redundant RPCs.

**Threshold:** Only defer reconnect syncs that occur while `backgroundMode=true`. First-launch / explicit login syncs are unaffected (those happen during the foreground onboarding flow).

---

### status-app (optional, for visibility)

The existing `scheduleBackendLifecycleUpdate` in `StatusGoService.java:194` already communicates background state to status-go for `PauseServices`/`ResumeServices`. No additional status-app code is needed if status-go's `SetWakuBackgroundMode`/`SetAppForeground` RPC handles both R1 and R2.

If a separate signal is preferred, add to `scheduleBackendLifecycleUpdate`:

```java
// Signal status-go to defer history sync in background.
nativeCall("SetAppForeground", "[" + visible + "]");
```

This call should appear after the existing `PauseServices`/`ResumeServices` call, within the same `lifecycleExecutor` task.

---

## Expected impact

From T5 measurements:

| Scenario | Before | After |
|----------|--------|-------|
| Background reconnect drain | ~425 mAh/hr peak (30 min) | Deferred to foreground — no background cost |
| History sync timing | Immediately on reconnect, regardless of UI state | On next foreground, triggered by `onForeground()` |
| User experience | Background sync (invisible) | First open after reconnect triggers sync — user sees messages loading in UI |

The deferred sync user experience is strictly better: the user sees the history arrive while the app is open, rather than having it silently arrive (and drain battery) while the app is locked.

---

## Scope

**In scope:**
- Reconnect-triggered sync (airplane mode, dead zone recovery, network switch)
- Background-only: no change to sync behavior when app is foregrounded

**Out of scope:**
- First login / account creation (always foreground, unaffected)
- Foreground reconnects (user opens app → immediate sync, unaffected)
- Gap fills (`fillGaps` for missed messages within a session — these are low-volume and can remain immediate)

---

## Test gates

**G1 — Sync is deferred in background:**
1. Background the app. Confirm `backgroundMode=true` is set.
2. Toggle airplane mode off (or simulate reconnect via `adb shell svc wifi enable`).
3. Observe logcat: `requestAllHistoricMessagesWithRetries` should NOT be called. `pendingHistorySync=true` log should appear.
4. Foreground the app. Confirm `onForeground()` fires and `requestAllHistoricMessagesWithRetries` IS called within 1s.

**G2 — Radio duty cycle during background reconnect:**
1. Run equivalent of T5 soak: loaded account, background, after reconnect, screen off from start.
2. `adb shell dumpsys batterystats --charged` → `mobile_radio` should be < 5 mAh over 30 min (vs ~100+ mAh in T5).
3. RX bytes via `TrafficStats` should be < 5 MB during 30 min background soak (vs 321 MB in T5).

**G3 — Messages arrive on foreground:**
1. While backgrounded with sync deferred, have another device send messages.
2. Foreground the app. Messages should appear within a few seconds (deferred sync runs).
3. OS notification for the message should still arrive via Waku push (not affected by deferred mailserver sync).

---

## Files to change

**status-go:**
- The mailserver service file where `requestAllHistoricMessagesWithRetries` is triggered on reconnect (search: `requestAllHistoricMessages`, `activeMailserverChanged`, `mailserverConnected`)
- `services/ext/api.go` — add `backgroundMode` state and `SetAppForeground` RPC (or reuse `SetWakuBackgroundMode` from R1)

**status-app:**
- `mobile/android/qt6/src/app/status/mobile/ipc/StatusGoService.java:194-213` — add `SetAppForeground` call inside `scheduleBackendLifecycleUpdate` (if not already covered by R1's `SetWakuBackgroundMode` call)

---

## Relation to R1

R1 and R2 both require status-go to track background/foreground state. They can share a single RPC (`SetAppForeground(bool)`) rather than two separate calls. Recommended: implement R1 and R2 as a single status-go PR with one shared state flag, and a single status-app call site in `scheduleBackendLifecycleUpdate`.

---

## Notes

- `requestAllHistoricMessagesWithRetries` Nim wrapper: `src/backend/mailservers.nim:23` — this is a passthrough to the status-go RPC; no Nim-side change is needed.
- The `connectionChange` path (`AppMain.qml` → `main/controller.nim:651` → `general/service.nim:91` → `status_go.connectionChange`) is the reconnect signal. This is separate from the mailserver history sync — `connectionChange` notifies status-go of the new network, but the sync is triggered internally by status-go's mailserver reconnect logic.
- `active mailserver changed` comment in `main/controller.nim:173-176` already has a placeholder for `requestAllHistoricMessagesResult` — this is the right place to add a foreground guard on the status-app side if needed.
