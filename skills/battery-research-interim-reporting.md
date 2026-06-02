---
id: battery-research-interim-reporting
title: Release confirmed findings progressively while research continues
phase: ops
type: pattern
severity: high
severity_reason: Waiting for a complete paper means developers sit idle while a one-line fix could ship today
source: extracted-from-practice + fieldcraft discipline
last_used: 2026-06-02
created: 2026-06-02
status: active
---

## Problem
Performance research takes hours or days. A single confirmed hypothesis may already be enough to ship a fix. Without an interim reporting protocol, actionable findings sit in a journal while developers wait for the "final paper."

## Rule

**As soon as one hypothesis reaches `[CONFIRMED]`, post an interim finding — do not wait.**

## Trigger Conditions

| Condition | Action |
|-----------|--------|
| Any H confirmed by device measurement | Immediate interim post to issue |
| Any H rejected (narrows search space) | Post "ruled out" note to issue |
| 2+ hours into testing with no verdict | Post progress note with inconclusive data |
| All hypotheses rejected | Escalation post — scope needs expanding |

## Interim Report Format (GitHub comment)

```markdown
## Interim Finding — [date] — [H number]

**Status:** CONFIRMED / REJECTED / INCONCLUSIVE

**Hypothesis:** [one sentence]

**Evidence:**
- Command: `[exact adb command]`
- Output: `[raw value]`
- Threshold: [what the threshold was and whether it was crossed]

**Recommended fix:**
- File: `[path:line]`
- Change: [one sentence description]
- Risk: Low / Medium / High
- Reason risk is [level]: [one sentence]

**Research continuing:**
Tests T[X]–T[Y] still running. Next interim update in ~[time].

**Does not block:** [other hypotheses being tested in parallel]
```

## Risk Classification for Fixes

Before recommending any fix in an interim report, classify it:

| Risk | Definition | Example |
|------|-----------|---------|
| **Low** | Self-contained, no user-visible behaviour change, easily reverted | Add `timer.stop()` in `applicationStateChanged` |
| **Medium** | Touches shared infrastructure, may affect other features | Guard `dispatchConnectionChange()` with `uiVisible` flag |
| **High** | Changes protocol behaviour, may affect reliability | Modify Waku ping interval |

Only recommend Low-risk fixes for immediate action in interim reports. Medium/High risk fixes go in the final report after full validation.

## Definition of Done — Final Report

Research is complete (report reaches v1.0) when ALL:

1. [ ] T1–T5 all have `[CONFIRMED]` or `[REJECTED]` verdicts with numeric values
2. [ ] All hypotheses in report updated from `[H]` to measured state
3. [ ] At least one interim report posted per confirmed hypothesis
4. [ ] Raw data (logcat, `top`, `batterystats`) appended to `journal.md`
5. [ ] Recommendations reflect confirmed findings only — not hypotheses
6. [ ] Findings compared with parallel investigators (mag, Sale) — discrepancies noted
7. [ ] Report versioned and posted to GitHub issue
8. [ ] Fails log reviewed — any lessons for future protocol improvement

## Escalation When All Hypotheses Rejected

1. Do not conclude "no bug found" — 65% overnight drain is real
2. Check: was WakuV2 actually in Light mode? (verify in app, not reporter's word)
3. Check: were there screen-on periods during the test? (`dumpsys batterystats` logs screen state)
4. Expand: run `adb bugreport` and inspect full wakelock timeline in Battery Historian
5. Expand: check for unreported foreground wake events (notification interactions)
6. Post escalation note to issue: "All identified hypotheses rejected — expanding scope to full bugreport analysis"

## Peer Review Before v1.0

- Share draft with mag and Sale before final post
- Ask specifically: "Does your parallel investigation contradict anything here?"
- Min 24h for review
- Discrepancies → surface in report, do not resolve unilaterally

## See also
- `bayesian-hypothesis-scoring.md`
- `android-controlled-battery-experiment.md`
