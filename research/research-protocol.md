# Autonomous Research Protocol — Android Battery/CPU Issues
**Version:** 1.0
**Created:** 2026-06-02
**Origin:** Issue #21045 investigation

This protocol is generic — apply it to any Android performance/battery issue.

---

## Epistemological Framework

Every claim in this research lives in exactly one state:

| State | Definition | Tag in report |
|-------|-----------|--------------|
| **Code-confirmed** | Directly visible in source — no device needed | `[CODE]` |
| **Hypothesis** | Code suggests it *could* cause the issue — needs measurement | `[H]` |
| **Unknown** | Cannot determine from source — requires external data | `[?]` |
| **Measured** | Device test completed, verdict recorded with number | `[CONFIRMED]` / `[REJECTED]` |

Never present a hypothesis as a fact. Never present an unknown as a hypothesis. A claim without a tag is incomplete.

---

## Principle: Researcher Mindset

> Stress test → observe → gather data → form hypothesis → prove or disprove → next step.
> No guesses. Every claim must have a measurement behind it.

---

## Phase 0: Framing (before touching device)

**Time budget: ~1 hour**

1. **Read the issue fully.** Extract:
   - Exact reproduction conditions (how long, what mode, what was running)
   - Expected vs actual (numbers, not descriptions)
   - Any developer hypotheses in comments

2. **Read related issues.** Map the problem space.

3. **Source code analysis.** Trace the relevant path:
   - Who calls what when the triggering event happens
   - What is and isn't paused/stopped
   - What runs in background, on what thread, at what frequency
   - Two-process architectures are easy to miss — check manifest

4. **Form ranked hypotheses.** Each must be:
   - Falsifiable with a measurement
   - Ranked by confidence (code evidence) and impact (if true)

5. **Write journal entry.** Record: hypotheses, evidence, open questions, measurement plan.

---

## Phase 1: Environment Setup

```bash
# 0. Charge phone to 100% before any test run.
#    Without a consistent start level, drain numbers across runs are not comparable.
adb shell dumpsys battery | grep level   # must show "level: 100"

# 1. Enable WiFi ADB (requires USB for first-time setup)
adb tcpip 5555
# Disconnect USB, get phone IP from Settings > About > Status > IP
adb connect <phone-ip>:5555
adb devices  # verify

# 2. Install the specific build under test
adb install -r <path-to-arm64.apk>
# Verify correct version is installed:
adb shell dumpsys package app.status.mobile | grep versionName

# 3. Kill all other background apps — tests must isolate Status drain
adb shell am kill-all   # kills cached background processes
# Verify only system processes remain:
adb shell ps -A | grep -v "system\|root\|shell\|radio" | wc -l

# 4. Verify WakuV2 mode BEFORE testing
#    In app: Advanced Settings > WakuV2 options — must be "Light" not "Relay"
#    Relay mode is expected to drain battery — testing it is pointless
#    Screenshot this setting and save to journal.

# 5. Mock cable unplug so Android tracks drain
adb shell dumpsys battery unplug
# To reset: adb shell dumpsys battery reset

# 6. Reset battery stats for clean measurement window
adb shell dumpsys batterystats --reset
echo "Stats reset at: $(date)"

# 7. Identify process names (app may use multiple processes)
adb shell ps -A | grep -E "status|statusgo"
```

---

## Phase 2: Control Measurement (app NOT running)

**Establish device idle drain rate. Without this, you cannot attribute drain to the app.**

```bash
# Force-stop the app
adb shell am force-stop app.status.mobile

# Lock screen (screen OFF = realistic background condition)
adb shell input keyevent KEYCODE_POWER

# Wait 10 minutes, then measure
sleep 600
adb shell dumpsys battery | grep level
adb shell dumpsys cpuinfo | head -10
```

Record: battery % drop over 10 min with app NOT running. This is your baseline drain rate.

---

## Phase 3: Baseline Measurements (app running, foreground)

**Run these BEFORE reproducing the issue. Capture in journal.**

```bash
# Battery baseline
adb shell dumpsys battery | grep -E "level|status|plugged|temperature"

# CPU baseline — all processes
adb shell top -b -n 1 -H | head -30

# App-specific CPU
adb shell top -b -n 1 | grep -E "app.status|statusgo"

# Wake locks — what's keeping the CPU awake
adb shell dumpsys power | grep -E -A2 "Wake Locks:|PARTIAL_WAKE_LOCK"

# Network baseline — bytes before test
adb shell cat /proc/net/dev

# Memory
adb shell dumpsys meminfo app.status.mobile 2>/dev/null | head -30

# Thermal
adb shell dumpsys thermalservice 2>/dev/null | head -20
```

Journal template:
```
## Baseline — [timestamp]
- Battery: X%
- CPU (UI process): X%
- CPU (statusgo process): X%
- Active wake locks: [list]
- Network RX bytes (verify interface — wlan0 for WiFi, rmnet_data0 for mobile): X
```

---

## Phase 3b: Reproduce the Issue (Minimal)

**Goal:** Shortest possible reproduction that still shows the drain.

For background battery drain:
```bash
# 1. Open app, log in, let it connect
adb shell monkey -p app.status.mobile -c android.intent.category.LAUNCHER 1

# 2. Wait for connection — confirm node.login before proceeding
#    Testing a disconnected node is not representative of real usage
adb logcat -v time 2>/dev/null | grep -m1 "node.login" && echo "CONNECTED — proceed"
# If this doesn't appear within 60s, the node failed to connect — stop and investigate

# 3. Lock screen (power button) — NOT just home button
#    Screen-on vs screen-off has significant Android power management differences
#    Tests MUST be run with screen locked
adb shell input keyevent KEYCODE_POWER

# 4. Start measurement loop (10-30 min soak)
START_NET=$(adb shell cat /proc/net/dev | grep wlan0)
echo "Test started: $(date)"
echo "Network baseline: $START_NET"

for i in $(seq 1 20); do
    sleep 30
    echo "=== T+$((i*30))s ==="
    adb shell top -b -n 1 | grep -E "app.status|statusgo"
done

# 5. Final measurements
adb shell dumpsys battery | grep level
adb shell dumpsys power | grep -A3 "Wake Locks:"
adb shell cat /proc/net/dev | grep wlan0
adb shell dumpsys batterystats --charged | grep -E "app.status|wakelock|cpu" | head -30
```

---

## Phase 4: Validate Hypotheses

### H-template: Qt event loop running in background
```bash
# UI process CPU should drop to ~0% when backgrounded if event loop pauses
adb shell top -b -n 1 | grep "app.status.mobile "  # no suffix = UI process
# Threshold: > 2% CPU sustained after 60s background → Qt loop NOT pausing → H CONFIRMED
# Threshold: < 0.5% CPU → Qt loop is pausing → H REJECTED
# 0.5–2% = INCONCLUSIVE, measure for longer
```

### H-template: Waku ping frequency
```bash
NET1=$(adb shell cat /proc/net/dev | grep wlan0 | awk '{print $2}')
sleep 60
NET2=$(adb shell cat /proc/net/dev | grep wlan0 | awk '{print $2}')
echo "RX bytes/min: $((NET2 - NET1))"
# > 50KB/min in background with no user activity = suspicious

# Airplane mode comparison — isolates network drain from CPU drain:
# Run: adb shell svc wifi disable  (WiFi off, keep measuring CPU)
# If CPU drops significantly → drain is network-related (Waku)
# If CPU stays same → drain is local (Qt event loop, timers)
adb shell svc wifi disable
sleep 60
adb shell top -b -n 1 | grep -E "app.status|statusgo"
adb shell svc wifi enable
```

### H-template: Wake lock count
```bash
# Count PARTIAL_WAKE_LOCKs — each = keeps CPU running
adb shell dumpsys power | grep "PARTIAL_WAKE_LOCK" | wc -l
# > 3 unexplained wake locks = suspicious
```

### H-template: Battery stats per-process
```bash
adb shell dumpsys batterystats --charged | grep -A5 "app.status"
# Look for: wakelockTime, cpuTimeMs, mobileRxBytes, mobileActiveTime
```

### H-template: Perfetto trace (wakelock root cause)
```bash
# 1. Start Perfetto trace (30s window)
adb shell perfetto \
  -c - --txt -o /data/misc/perfetto-traces/trace.pb <<EOF
buffers { size_kb: 65536 }
data_sources { config { name: "android.power" android_power_config {
  battery_poll_ms: 1000
  collect_power_rails: true
  collect_energy_estimation_breakdown: true
}}}
data_sources { config { name: "linux.process_stats" }}
duration_ms: 30000
EOF

# 2. Pull trace
adb pull /data/misc/perfetto-traces/trace.pb /tmp/status-trace.pb

# 3. Query wakelock attribution (run in Perfetto UI or with trace_processor)
# Open https://ui.perfetto.dev — drag /tmp/status-trace.pb
# SQL to find top wakelock holders:
# SELECT process_name, SUM(dur)/1e9 AS wakelock_seconds
# FROM android_wakelock_wake_period
# JOIN process USING (upid)
# GROUP BY process_name ORDER BY wakelock_seconds DESC LIMIT 10
```
**When to use:** After T1–T5 if any result is INCONCLUSIVE — Perfetto identifies exact wakelock source. Note: Perfetto is available on Android 9+ (S20 FE = Android 13 ✓).

### Statistical comparison (Mann-Whitney U)
Full method and worked example: `skills/android-controlled-battery-experiment.md`.

---

## Phase 5: Logcat Capture

**Always capture logcat during reproduction. Filter by relevant tags.**

```bash
# Start capturing
adb logcat -v threadtime \
  -s StatusGoService StatusGoServiceClient StatusGoStub \
  -s QtCore QtAndroid \
  > /tmp/status-logcat-$(date +%Y%m%d-%H%M%S).txt &

# After test:
kill %1
grep -E "ERROR|WARN|Exception|PauseServices|ResumeServices" /tmp/status-logcat-*.txt
```

**What to look for:**
- Did `PauseServices` actually get called after backgrounding?
- Any errors in the background lifecycle path?
- Frequency of any periodic log messages (reveals polling intervals)
- Any `FATAL` or `ANR` entries

---

## Phase 6: Analysis

For each hypothesis:
- **CONFIRMED**: measurement shows expected signature → document what was measured, exact value
- **REJECTED**: measurement contradicts hypothesis → document what was expected vs actual
- **INCONCLUSIVE**: measurement was noisy or wrong metric → note what better metric to use

Write in journal:
```
## H1 Validation — [date]
**Hypothesis:** Qt event loop runs in background
**Measurement:** `top` showed UI process at 8% CPU after 2min background
**Verdict:** CONFIRMED
**Next:** quantify how much drain this causes vs statusgo process
```

---

## Phase 7: Skill Extraction

After completing research, extract:

1. **ADB commands that worked** → `research/status-android-skills.md`
2. **App-specific measurement notes** → (e.g., "top shows two processes: UI and :statusgo")
3. **What didn't work and why** → prevents wasted effort next session

---

## Quick Reference: ADB Commands

```bash
# Battery
adb shell dumpsys battery
adb shell dumpsys battery unplug          # mock unplug
adb shell dumpsys battery reset           # restore
adb shell dumpsys batterystats --reset    # reset stats
adb shell dumpsys batterystats --charged  # full stats

# CPU
adb shell top -b -n 1                     # one snapshot
adb shell top -b -n 1 -H                  # include threads
adb shell dumpsys cpuinfo                 # per-process rolling avg

# Wake locks
adb shell dumpsys power | grep -E "Locks|WAKE"
adb shell dumpsys wakelocks               # sometimes available

# Network
adb shell cat /proc/net/dev               # cumulative bytes per interface
adb shell dumpsys netstats                # app-level network stats

# Logcat
adb logcat -v time -s TAG1 TAG2           # filtered
adb logcat -c                             # clear buffer

# Process info
adb shell ps -A | grep status            # list processes
adb shell am force-stop app.status.mobile  # force stop

# App control
adb shell input keyevent KEYCODE_HOME     # background app
adb shell monkey -p app.status.mobile -c android.intent.category.LAUNCHER 1  # launch
```

---

## Time Estimates

Use these when planning research sessions:

| Phase | Activity | Estimate |
|-------|----------|---------|
| 0 | Issue reading + related issues | 30 min |
| 0 | Source code analysis (1 component) | 30–60 min |
| 0 | Hypothesis formation + journal entry | 30 min |
| 1 | Environment setup (first time, WiFi ADB) | 15 min |
| 1 | Environment setup (repeat session) | 5 min |
| 2 | Control measurement (app force-stopped) | 15 min |
| 3 | Baseline measurements | 10 min |
| 3 | Single test run (10 min soak + analysis) | 25 min |
| 3 | Full T1–T5 suite (single run each) | ~3 hours |
| 3 | Full T1–T5 suite (N=5 runs, statistical) | ~4–5 hours active |
| 3 | **Charging time between N=5 runs** (device back to 100%) | **+3–5 hours idle** |
| 6 | Analysis + journal update | 30 min |
| 7 | Interim report (1–2 confirmed findings) | 30 min |
| 7 | Final report update | 1 hour |
| **Total (single runs)** | **Phase 0 + setup + T1–T5 + reporting** | **~6–8 hours** |
| **Total (N=5 runs, active)** | **Phase 0 + setup + T1–T5 + reporting** | **~8–10 hours active** |
| **Total (N=5 runs, wall clock)** | **Including charging waits between runs** | **~11–15 hours wall clock** |

How the totals are computed:
- Single run path: 30+45+30+15+15+10+180+30+30+60 = 445 min ≈ 7.5h (rounded to 6–8h)
- N=5 path: T1–T5 expands to ~250min active; charging to 100% takes 60–90 min per run on S20 FE → 4 recharges × ~75min = ~300min charging idle time
- Wall clock for N=5: 515 min active + ~300 min charging ≈ 815 min ≈ 13.5h — plan across two sessions
- Range reflects: source code analysis variance (30–60min) and retries required when results are INCONCLUSIVE

---

## Hypothesis Probability Scoring

**Full skill:** `skills/bayesian-hypothesis-scoring.md`

When forming hypotheses, assign a numeric probability (0–100%) based on code evidence. Full scoring table, Bayesian update rules, and expected-impact formula: `skills/bayesian-hypothesis-scoring.md`.

---

## Interim Report Protocol

**Full skill:** `skills/battery-research-interim-reporting.md`

**Trigger:** Any single hypothesis reaches `[CONFIRMED]` status via device measurement.

**Do not wait for full T1–T5 suite.** If T1 confirms H1a (keyboard timer) in the first 10 minutes, release an interim report immediately. Developers can start a fix while T2–T5 continue.

Format, risk classification, definition of done, and escalation: `skills/battery-research-interim-reporting.md`. Post to GitHub issue as a comment. Tag jrainville and alexjba.

---

## Raw Data Storage

All measurements go directly into `journal.md` under the session they were collected. Format:

```
### Raw Data — T1 — [timestamp]
**Command:** `adb shell top -b -n 1 | grep "app.status.mobile "`
**Output:**
  PID USER ... CPU% ... ARGS
  1234 u0_a99 ... 8.2 ... app.status.mobile
**Verdict:** H1 CONFIRMED — 8.2% CPU sustained after 5min background with WiFi off
```

Never summarise without including the raw output. The summary can be wrong; the raw output cannot.

---

## Anti-Patterns (don't do these)

- **Don't run overnight tests without checkpoints.** 10-minute soak with measurements every 30s is more informative.
- **Don't screenshot instead of data.** `dumpsys batterystats` > Android UI screenshots.
- **Don't assume the fix is obvious.** Validate every hypothesis before concluding.
- **Don't test while charging.** Always `dumpsys battery unplug` first.
- **Don't use the same baseline twice.** Reset `batterystats` at the start of every test run.
- **Don't ignore the two-process architecture.** Measure both `app.status.mobile` AND `:statusgo` CPU separately.
- **Don't background with home button.** Use power button (screen lock). Screen-on affects power management.
- **Don't skip the control test.** App force-stopped baseline is required to attribute drain to Status.
- **Don't test in Relay mode.** Verify WakuV2 = Light before every test session.
- **Don't use tcpdump on stock Samsung.** Binary not present. Use `/proc/net/dev` and `dumpsys netstats`.
- **Don't claim a verdict without a threshold.** State the number and whether it crossed the bar.
