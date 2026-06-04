# PR Draft — R1: Extend Waku filter subscription renewal interval in background

**Target repo:** `status-im/status-go`
**Related issue:** status-im/status-app#21045
**Measured evidence:** `xAlisher/status-android-battery-research` (T4, T5b soaks)

---

## Problem

On Android, the LTE radio accounts for **98% of Status battery drain** on cellular (T4 soak: `mobile_radio = 67.8 mAh / 69.1 mAh` over 42 min). The drain mechanism is Waku light mode filter subscription renewal: the renewal keepalive (~13.5 min interval, observed from CPU spike pattern in T3) wakes the LTE modem from RRC Idle each time, and the modem takes 5–10s to return to sleep. This produces a 55% radio active duty cycle even on a 0-contact account.

The renewal runs at the same rate whether the user is actively using the app or the phone has been locked for hours. At background rates, a typical loaded account (50 contacts, 3 communities) drains **144 mAh/hr** from the radio alone — **3.2% of a 4500 mAh battery per hour**, purely from keepalive traffic.

The app already communicates foreground/background transitions to status-go via `PauseServices`/`ResumeServices` (see `StatusGoService.java:194`). Waku is intentionally excluded from this mechanism to preserve push notifications. This PR adds a separate, lightweight background mode for Waku that **reduces renewal frequency without dropping subscriptions or notifications**.

---

## Proposed change

### status-go

**File:** `services/ext/api.go` (or wherever the Waku light node's filter subscription renewal scheduler lives)

Add a new exported RPC method:

```go
// SetWakuBackgroundMode reduces filter subscription renewal frequency
// when the app UI is not visible. Pass true when app goes to background,
// false when it returns to foreground.
//
// Background interval: 45 min (vs default ~13.5 min in foreground).
// This reduces LTE radio wake events from ~4/hr to ~1.3/hr in background.
// Subscriptions are maintained; existing filters are NOT dropped.
func (api *PublicAPI) SetWakuBackgroundMode(ctx context.Context, background bool) error {
    interval := defaultFilterRenewalInterval  // e.g. 14 * time.Minute
    if background {
        interval = 45 * time.Minute
    }
    return api.service.messenger.SetFilterSubscriptionRenewalInterval(interval)
}
```

The messenger's Waku light node already has a renewal ticker (the one producing the ~13.5 min CPU spikes observed in T3). This PR replaces the fixed ticker period with a mutable interval and adds the `SetWakuBackgroundMode` RPC to expose it.

**Suggested interval constants:**
- Foreground: keep existing default (observed ~13.5 min; leave unchanged so foreground latency is unaffected)
- Background: 45 min — reduces radio wakes by ~70% while staying under typical Doze maintenance window spacing (~1 hr on Android 13)

---

### status-app

**File:** `mobile/android/qt6/src/app/status/mobile/ipc/StatusGoService.java`

In `scheduleBackendLifecycleUpdate`, after the existing `PauseServices`/`ResumeServices` call, add:

```java
// Adjust Waku renewal interval for background/foreground.
// Waku is excluded from PauseServices (notifications must keep working),
// but slowing keepalives in background significantly reduces LTE radio drain.
final String wakuArgs = "[" + visible + "]";
nativeCall("SetWakuBackgroundMode", "[" + !visible + "]");
```

(The exact call site should be after the existing `PauseServices`/`ResumeServices` call completes, within the same `lifecycleExecutor` task, so ordering is guaranteed.)

---

## Expected impact

From T4 and T5b measurements:

| Scenario | Before | After (modeled) |
|----------|--------|-----------------|
| Radio wake rate (bare account, background) | ~4.4/hr (55% duty cycle) | ~1.3/hr (-70%) |
| Radio wake rate (loaded account, background) | ~4.4/hr (50% duty cycle) | ~1.3/hr (-70%) |
| Drain rate, loaded account, cellular | ~144 mAh/hr | ~50–70 mAh/hr (est.) |

The reduction in duty cycle is approximate. The exact savings depend on modem RRC tail timer (5–15s per carrier). The 45 min interval is conservative; even 30 min would cut radio wakes by 55%.

Notifications are unaffected: the filter subscription is maintained. Only the renewal keepalive frequency changes. Any message arriving between renewals is still received because the filter remains registered at the relay.

---

## Test gates

**G1 — Renewal interval changes on background:**
1. Attach ADB logcat, background the app.
2. Confirm a `SetWakuBackgroundMode(true)` call appears in logs within 1s of screen lock.
3. Confirm subsequent CPU spikes from the renewal ticker are ~45 min apart (vs ~13.5 min before).

**G2 — Radio duty cycle reduction (T4-equivalent soak):**
1. Run a 42-min background soak (cellular, bare account, same methodology as T4).
2. `adb shell dumpsys batterystats --charged` → `mobile_radio` mAh should be < 40 mAh (vs 67.8 mAh in T4).
3. Radio active fraction should be < 30% of soak duration (vs 55% in T4).

**G3 — Notifications unaffected:**
1. While backgrounded with background mode active, send a DM from a separate device.
2. Verify OS notification appears within the renewal interval + a small delta (not within the original 13.5 min window, unless the sender triggers a push event directly).
3. On foreground: verify `SetWakuBackgroundMode(false)` is called and renewal returns to normal cadence.

---

## Files to change

**status-go:**
- The file containing the Waku light node filter subscription renewal loop (search: `filterSubscriptionLoop`, `renewFilters`, or similar)
- `services/ext/api.go` — add `SetWakuBackgroundMode` RPC method

**status-app:**
- `mobile/android/qt6/src/app/status/mobile/ipc/StatusGoService.java:194-213` — add `nativeCall("SetWakuBackgroundMode", ...)` inside `scheduleBackendLifecycleUpdate`

---

## Notes

- The existing `PauseServices`/`ResumeServices` path (`StatusGoService.java:194`) is the right call site for the status-app change. It already runs on `lifecycleExecutor` with generation coalescing, so rapid foreground/background transitions are handled correctly.
- iOS: equivalent change would target the background fetch / BGTaskScheduler path. Not in scope here.
- If the Waku renewal is implemented as a `time.Ticker` in Go, replacing it with `time.NewTimer` + reset would allow interval changes without goroutine restart.
