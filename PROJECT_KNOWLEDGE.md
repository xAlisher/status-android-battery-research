# PROJECT_KNOWLEDGE — Status Android Battery Research

Accumulated lessons from #21045 investigation and fix PRs.
Raw captures live in `~/status-test-day/retro-log.md`; this file holds permanent distilled wisdom.

---

## Architecture Facts

- **Two processes:** `app.status.mobile` (Qt/Nim UI — frozen by Android in background) and `app.status.mobile:statusgo` (Waku/libstatus foreground service — always running).
- **UI process is frozen** within ~30s of backgrounding on Android 13+ — H1 (Qt event loop) and H1a (keyboard timer) are moot on modern Android. Confirmed T1: 0.0% CPU all 20 samples.
- **statusgo is the active process** in background. All meaningful background battery cost comes from here.
- **PauseServices** excludes "messaging" explicitly — `isPaused()` is always false for the messaging service on Android. Use a separate `backgroundMode` flag.

## Measurement Numbers (S20 FE, Android 13, WakuV2 Light)

| Scenario | Drain | LTE duty cycle | Notes |
|---|---|---|---|
| WiFi, notif=NONE, bare account (T2) | 0.352 mAh/hr | — | Baseline |
| WiFi, notif=DEFAULT, bare account (T3) | 1.632 mAh/hr | — | Notifications add ~1.3 mAh/hr on WiFi |
| Cellular, notif=DEFAULT, bare account (T4) | **97.5 mAh/hr** | **55%** | LTE radio is the dominant cost |
| Cellular, notif=DEFAULT, loaded account, steady-state (T5b) | **144 mAh/hr** | ~49.5% | 21× more packets than bare, CPU identical |
| Cellular, notif=DEFAULT, loaded, post-login sync storm (T5) | ~425 mAh/hr | 99% | Mailserver catchup storm |
| Reported overnight drain (#21045) | ~8.1%/hr | — | ~365 mAh/hr phone drain |

- 1% battery ≈ 623 J on S20 FE (4500 mAh × 3.6 V)
- Normal Android idle: 1–2%/hr; Status delta: ~6%/hr
- Waku keepalive interval: **~13.5 min** (confirmed T3 — two CPU spikes 810s apart)
- Each keepalive wakes LTE modem → ~5–10s RRC re-entry → 55% duty cycle at bare minimum

## Root Cause Summary

**Primary confirmed driver:** Waku filter subscription renewal on cellular.
- ~13.5 min keepalive interval → LTE never fully sleeps
- Account richness scales linearly with packets (not CPU): loaded account = 21× more traffic

**Secondary confirmed driver:** Post-reconnect mailserver sync storm.
- After long disconnect, `asyncRequestAllHistoricMessages()` fires from two sites unconditionally
- Produces 321 MB RX, 99% LTE duty cycle until sync completes

**Rejected:** H1 (Qt event loop), H1a (keyboard timer) — Android freezes UI process within ~30s

## Fix Architecture (PRs)

### R1 — Waku filter renewal suppression
**go-waku** (`logos-messaging/logos-delivery-go#1304`): `Sub.backgroundMode atomic.Bool` + guards in `subscriptionLoop` ticker and closing cases + `SetBackgroundMode()` on `Sub` and `FilterManager`.

**status-go** (PR #7508 commit 2): Call chain —
```
Messenger.SetAppBackground(background)
  → m.messaging.SetFilterBackgroundMode(background)      // pkg/messaging/api.go
  → core.stack.Transport.SetFilterBackgroundMode(...)    // layers/transport/transport.go
  → t.waku.SetFilterBackgroundMode(...)                  // types.Waku interface + gowaku.go
  → w.filterManager.SetBackgroundMode(background)        // go-waku FilterManager
```

### R2 — Mailserver sync deferral
**status-go** (PR #7508 commit 1):
- `Messenger.backgroundMode atomic.Bool`
- `SetAppBackground(bool)` — defers `asyncRequestAllHistoricMessages()` until foreground
- Guards in `handleConnectionChange` and `checkForStorenodeCycleSignals`
- RPC exposed via `services/ext/api.go`

**status-app** (PR #21111): `StatusGoService.java` calls `nativeCall("SetAppBackground", "[" + !visible + "]")` in `scheduleBackendLifecycleUpdate`.

### R3 — WiFi-only sync toggle
**status-app** (PR #21110): Remove `visible: false` from `allowSyncingOnMobileNetwork` list item in `MessagingView.qml`. The Nim backend setting `getSyncingOnMobileNetwork` already works; only the UI hide was missing.

## Expected Post-Fix Numbers

- **R1 alone:** LTE duty cycle from 55% → near 0% after Doze engages. Drain: 97.5 mAh/hr → ~1.6 mAh/hr (WiFi baseline). ~60× improvement for cellular users.
- **R1+R2:** Loaded account steady-state from 144 mAh/hr → ~1–2 mAh/hr. Sync storm eliminated. ~70–100× combined improvement.
- **Tradeoff:** 1–2s subscription staleness on foreground return. Push notifications unaffected.

## Go status-go Patterns

### Adding a method to the Waku call chain
When adding a new method that must flow from `Messenger` down to `filterManager`:
1. `pkg/messaging/waku/types/waku.go` — add to `Waku` interface
2. `pkg/messaging/waku/gowaku.go` — implement on `*Waku`
3. `pkg/messaging/layers/transport/transport.go` — proxy through `t.waku`
4. `pkg/messaging/api.go` — expose on `API`
5. `protocol/messenger.go` — call from entry point

Verify with: `var _ types.Waku = (*waku.Waku)(nil)` in a `//go:build ignore` file.

### go.mod replace directive + go.sum
When adding a `replace github.com/X => github.com/Y` directive:
- The original module X's go.sum entries become orphaned
- `go mod tidy` should remove them, but may silently skip if other tidy errors (missing migration packages) block completion
- **Fix:** manually remove the two orphaned lines (`X h1:...` and `X/go.mod h1:...`) from go.sum and commit
- CI golangci-lint runs its own `go mod tidy` and diffs go.sum — it will catch this

### Companion PR timing
The status-go `Check` workflow runs within seconds of a push and verifies the companion status-app PR's `vendor/status-go` submodule matches the pushed SHA.
- **Rule:** update the companion PR submodule in the same terminal session immediately before or after pushing to status-go — not minutes later
- If you miss the window: `gh run rerun <run-id> --repo status-im/status-go --failed`

## CI Notes

- `golangci-lint` takes ~5 min; runs `go mod tidy` as first step and diffs go.sum
- `Check` = companion PR SHA check — runs in ~15s, timing-sensitive
- `jenkins/prs/tests` = full test suite — takes 30–60 min, don't wait for it before declaring CI green on minor changes
- `Check status-go submodule branch` on status-app requires submodule commit to be on `develop` — only passes after status-go PR merges to develop; expected failure until then
