---
id: skills-index
created: 2026-06-02
project: status-android-battery-research
---

# Skills Index — Status Android Battery Research

Skills extracted during investigation of [#21045](https://github.com/status-im/status-app/issues/21045).
All files follow the fieldcraft atomic recipe schema (YAML frontmatter + Problem/Recipe/Why/See-also).

## Quick Lookup

| Skill file | Use when |
|------------|----------|
| `bayesian-hypothesis-scoring.md` | Assigning and updating hypothesis probabilities from code evidence |
| `android-energy-code-smells.md` | Scanning source for known battery anti-patterns before device tests |
| `android-battery-measurement-toolkit.md` | Choosing the right ADB/Perfetto tool for a given measurement |
| `android-controlled-battery-experiment.md` | Designing statistically valid N=5 experiments, Joule conversion |
| `battery-research-interim-reporting.md` | Triggering and formatting interim GitHub reports mid-research |

## File Descriptions

### `bayesian-hypothesis-scoring.md`
Evidence-based probability scoring (0–100%) using a point system for code evidence types.
Includes Bayesian update rules (measurement × prior), expected impact formula (probability × drain if true),
and calibrated priors for common Android patterns (e.g., QTimer no stop = 70% prior).
- Source: arxiv:2212.13773, arxiv:1811.05422

### `android-energy-code-smells.md`
10-pattern catalogue of Android battery anti-patterns, each with detection bash script and fix.
Key patterns: Durable WakeLock, unconstrained QTimer, unguarded network callback, Qt event loop
not suspended, radio-keeping pattern, HashMap GC, signal cascade without background guard.
Status-specific instances noted for each pattern.
- Source: PMC11479295, Nature s41598-024

### `android-battery-measurement-toolkit.md`
Tool decision tree: when to use `dumpsys battery` vs `batterystats` vs `top` vs Perfetto vs Battery Historian.
ADB command reference with thresholds. Perfetto SQL queries for wakelock root cause.
Status-specific notes: two-process architecture, `/proc/net/dev` preferred over tcpdump on stock Samsung.

### `android-controlled-battery-experiment.md`
Protocol for statistically valid Android battery experiments: N=5 runs, controlled variables checklist,
Mann-Whitney U test (scipy), Joule conversion (1% ≈ 623J on S20 FE), Status-specific thresholds
(UI process >2% CPU = Qt loop running, >50KB/min network = Waku active).
- Source: arxiv:2604.25587 (879 configs, 15 runs each)

### `battery-research-interim-reporting.md`
Trigger conditions for posting interim findings before research completes. GitHub comment template
with risk classification (Low/Medium/High). 8-point definition-of-done checklist.
Escalation path when all hypotheses are rejected.

## Status-Specific ADB Quick Reference

*Two-process architecture, process names, and monitoring commands: `research/status-android-skills.md § Architecture Facts`.*

```bash
# Verify WakuV2 = Light before every test (Relay is expected to drain)
# In app: Advanced Settings > WakuV2 options

# Network bytes per interface
adb shell cat /proc/net/dev | grep wlan0

```
