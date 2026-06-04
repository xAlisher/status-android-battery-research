# PROJECT_KNOWLEDGE — Status Android Battery Research

Accumulated lessons from #21045 investigation and fix PRs.
Raw captures live in `~/status-test-day/retro-log.md`; this file holds permanent distilled wisdom.

---

## Architecture Facts

- **Two processes:** `app.status.mobile` (Qt/Nim UI — frozen by Android in background) and `app.status.mobile:statusgo` (Waku/libstatus foreground service — always running).
- **UI process is frozen** within ~30s of backgrounding on Android 13+ — H1 (Qt event loop) and H1a (keyboard timer) are moot on modern Android. Confirmed T1: 0.0% CPU all 20 samples.
- **statusgo is the active process** in background. All meaningful background battery cost comes from here.
- **PauseServices/ResumeServices** work for Messenger once it is registered as a `PausableMessenger` in the `ServiceRegistry`. Register in `populateServiceRegistry()` by wrapping `wakuV2ExtSrvc.Messenger()`. The earlier note about needing a separate `backgroundMode` flag was wrong — Jo's #7516 uses `isPaused()` directly.

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

### What landed in 2.38

**Sale's #7507 (MERGED 2026-06-03):**
- `fix(rpclimiter)`: pause per-chain 1s RPC limiter tickers in background (5 chains × 1 tick/s = 5 CPU wakeups/sec eliminated)
- `chore(ens)`: pause ENS verifier ticker in background

**Jo's #7516 / #21125 (in review):** Cleaner version of our R2 — mailserver sync deferral when backgrounded.
- Removed `ToBackground()`/`ToForeground()` from `Messenger` entirely
- Folded httpServer lifecycle + `asyncRequestAllHistoricMessages()` into `SetPaused()`
- `PausableMessenger` delegates `Pause()`→`SetPaused(true)`, `Resume()`→`SetPaused(false)`
- Guard in `handleConnectionChange`: `if online && !m.isPaused()` (uses existing `paused` atomic, no new field)
- No go-waku dependency

### Deferred to 2.39 / Logos Delivery

**R1 — Waku filter health-check ping suppression** (go-waku #1304):
- `FilterHealthCheckLoop` pings every ~1 min → keeps LTE radio active at 55% duty cycle
- Fix: `backgroundMode atomic.Bool` in `WakuFilterLightNode`, gate in `FilterHealthCheckLoop`
- Dropped from 2.38 — regression risk (silent subscription expiry) + go-waku → Logos Delivery migration in 2.39

**R3 — WiFi-only sync toggle** (status-app #21110):
- Backend `getSyncingOnMobileNetwork` was rolled back in #20469; UI toggle pointless without it

## Measured Post-Fix Numbers

| Scenario | Modem duty cycle | Drain |
|---|---|---|
| T4 baseline (cellular, bare) | 55% | 97.5 mAh/hr |
| T5b baseline (cellular, loaded) | ~50% | 144 mAh/hr |
| T5b with R1b+R2+R3 (weak signal -120 dBm) | **15.8%** | ~40 mAh/hr est |
| T5b with R1b only (previous session) | **5.6%** | ~14 mAh/hr est |

- 15.8% vs 5.6% gap: R1b (FilterHealthCheckLoop suppression) accounts for most of the difference. Without it, ~15% is the floor for 2.38.
- WiFi drain (T3): 1.6 mAh/hr — unaffected.

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
