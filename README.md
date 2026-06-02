# Status Android — Battery & CPU Drain Research

Autonomous investigation of [#21045 — Android: high battery consumption and CPU usage](https://github.com/status-im/status-app/issues/21045).

**Build under test:** `2.38.0-rc.4-5-g4cf3d8e4b`
**Device:** Samsung Galaxy S20 FE (SM-G780G), Android 13
**Status:** Phase 0 complete (source analysis) — awaiting device measurements

---

## What's in here

```
research/
  report.md              ← Scientific report: hypotheses, evidence, recommendations
  journal.md             ← Session-by-session research log, raw data, fails log
  research-protocol.md   ← Generic autonomous research protocol for Android battery issues
  status-android-skills.md ← Status-specific ADB commands and gotchas

skills/
  INDEX.md                              ← Start here — skill descriptions and quick lookup
  bayesian-hypothesis-scoring.md        ← Evidence-based probability scoring for hypotheses
  android-energy-code-smells.md         ← 10-pattern catalogue of Android battery anti-patterns
  android-battery-measurement-toolkit.md ← Tool selection: ADB vs Perfetto vs batterystats
  android-controlled-battery-experiment.md ← N=5 protocol, Mann-Whitney U, Joule conversion
  battery-research-interim-reporting.md ← Interim finding format and trigger conditions
```

---

## Key Findings (Phase 0 — source analysis only)

All claims below are hypotheses until confirmed by device measurement.

| ID | Hypothesis | Probability | Impact if true |
|----|-----------|-------------|----------------|
| H1a | 50ms keyboard polling QTimer fires 20×/sec with no background guard | `[CODE — 90%]` | ~1–2%/hr CPU drain |
| H1 | Qt event loop not paused on Android `onPause()` — all timers continue | `[H — 75%]` | Enables H1a and all QML timers |
| H2 | Waku light mode keepalives run regardless of pause state, wake radio | `[H — 65%]` | ~2–3%/hr battery |
| H1b | New `NetworkConnectivityCallback` fires on network changes, no background guard | `[CODE — 60%]` | Compounds with H2 |
| H5 | `wallet-tick-reload` cascade triggers HTTP price API calls in background | `[H — 45%]` | Amplified by account count |

**Most critical single fix:** `ui/StatusQ/src/systemutilsinternal.cpp:74` — add `applicationStateChanged` handler to stop `keyboardTimer` when app is backgrounded. Zero functional regression; eliminates 576,000 JNI calls over an 8-hour overnight session.

---

## Validation Tests (ready to run)

T1–T5 defined in `research/report.md` § 5. Requires: device at 100%, WakuV2 = Light, `dumpsys battery unplug`, N=5 runs, Mann-Whitney U for statistical comparison.

---

## Skills

The `skills/` directory contains reusable, domain-agnostic research recipes extracted from this investigation. Each follows the fieldcraft atomic recipe schema and is applicable to any Android performance/battery issue — not just Status.

See `skills/INDEX.md` for the full skill guide.

---

## Epistemological Framework

All claims carry a state tag (`[CODE]`, `[H — X%]`, `[CONFIRMED]`, `[REJECTED]`, `[INCONCLUSIVE]`, `[?]`). Full definitions: `research/report.md` header block. A claim without a tag is not a finished claim.
