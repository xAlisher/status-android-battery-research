---
id: android-controlled-battery-experiment
title: Run a scientifically valid controlled battery drain experiment on Android
phase: ops
type: pattern
severity: high
severity_reason: Uncontrolled tests produce unrepeatable numbers that cannot be used to confirm or reject hypotheses
source: extracted-from-research + arxiv:2604.25587 + MDPI:sensors-24-06469
upstream_url: https://arxiv.org/html/2604.25587v1
last_used: 2026-06-02
created: 2026-06-02
status: active
---

## Problem
Battery drain tests without controlled variables produce numbers that can't be compared across runs or used to isolate causes. A 5% difference between runs is meaningless noise if brightness, other apps, or charge level varied.

## Scientific Standards (from literature)

- **Minimum runs per condition:** 5 (literature uses 15; 5 is practical minimum for statistical validity)
- **Statistical test:** Mann-Whitney U (non-parametric — battery data is rarely normally distributed)
- **Significance threshold:** p < 0.05
- **Unit:** Convert % to Joules for comparison: `J = (mAh × V × 3600) / 1000`
  - Samsung Galaxy S20 FE voltage ≈ 3.85V, 4500mAh battery
  - 1% ≈ 45mAh ≈ 45 × 3.85 × 3600 / 1000 ≈ **623 Joules**
- **Single device per test series** — never compare runs across different phones

## Controlled Variables Checklist (set before every run)

```bash
# 1. Battery at 100%
adb shell dumpsys battery | grep "level:"   # must be 100

# 2. Mock cable unplug (Android tracks drain without charging)
adb shell dumpsys battery unplug

# 3. Screen brightness — fix at 0% (or 50% for realistic tests)
adb shell settings put system screen_brightness 0

# 4. Screen timeout — set long enough to not interfere
adb shell settings put system screen_off_timeout 3600000  # 1 hour

# 5. Kill all other apps
adb shell am kill-all

# 6. Verify no other apps running
adb shell ps -A | grep -v "system\|root\|shell\|radio\|zygote\|surfaceflinger" | wc -l

# 7. Fix network state (WiFi or mobile data — pick one per test series)
adb shell svc wifi enable    # OR: adb shell svc wifi disable

# 8. Reset battery stats
adb shell dumpsys batterystats --reset
echo "Run started: $(date) | Battery: $(adb shell dumpsys battery | grep level)"
```

## Run Template (N=5)

```bash
#!/bin/bash
# Run this script 5 times with identical setup for statistical validity

RUN=$1  # Pass run number: ./run_test.sh 1

echo "=== RUN $RUN START ===" >> /tmp/battery-test.log
echo "Time: $(date)" >> /tmp/battery-test.log
echo "Battery: $(adb shell dumpsys battery | grep level)" >> /tmp/battery-test.log

# Background app (screen lock)
adb shell input keyevent KEYCODE_POWER

# Sample every 60s for 10 minutes (adjust for your test duration)
for i in $(seq 1 10); do
    sleep 60
    {
        echo "--- T+${i}min ---"
        echo "Battery: $(adb shell dumpsys battery | grep level)"
        echo "CPU UI: $(adb shell top -b -n 1 | grep 'im.status.ethereum ')"
        echo "CPU SGo: $(adb shell top -b -n 1 | grep ':statusgo')"
        echo "Net RX: $(adb shell cat /proc/net/dev | grep ${IFACE:-wlan0} | awk '{print $2}')"  # set IFACE=rmnet_data0 for cellular tests
    } >> /tmp/battery-test-run$RUN.log
done

echo "=== RUN $RUN END ===" >> /tmp/battery-test-run$RUN.log
echo "Final battery: $(adb shell dumpsys battery | grep level)" >> /tmp/battery-test-run$RUN.log

# Reset for next run — also force-stop and cold-start the app for run independence
adb shell am force-stop im.status.ethereum
adb shell dumpsys battery reset
```

## Control Test (required baseline)

Run the same script with `adb shell am force-stop im.status.ethereum` before starting. This gives the device-only idle drain rate. Without it, you cannot attribute drain to Status.

```bash
adb shell am force-stop im.status.ethereum
adb shell am force-stop im.status.ethereum:statusgo 2>/dev/null
# Then run full measurement script
```

## Differential Tests (isolate factors)

| Test | Variable changed | What it isolates |
|------|-----------------|-----------------|
| WiFi off | `adb shell svc wifi disable` | CPU drain vs network drain |
| Mobile data vs WiFi | switch network type | radio sleep behaviour |
| App foregrounded | bring to front | background vs foreground delta |
| Dark mode | Settings > Display | UI rendering cost |

Run each differential test with N=5 and compare medians.

## Thresholds (Status-specific)

| Metric | Normal | Concerning | Critical |
|--------|--------|-----------|---------|
| UI process CPU (background) | < 0.5% | 0.5–2% | > 2% |
| `:statusgo` CPU (background) | < 1% | 1–5% | > 5% |
| Wakelock duration (10min test) | < 1min | 1–5min | > 5min |
| Network RX (5min background) | < 50KB | 50–250KB | > 250KB |
| Battery drain (10min background) | < 0.5% | 0.5–1% | > 1% |

## Data Recording Format

```
## Test Run [N] — [timestamp]
**Condition:** [describe: WiFi on/off, screen locked, app version]
**Battery start:** X%
**Battery end:** Y%
**Delta:** Z% over T minutes = W%/hour
**UI process CPU (median):** X%
**:statusgo CPU (median):** Y%
**Network RX bytes/min:** Z
**Wake locks held:** [list from dumpsys power]
**Raw data file:** /tmp/battery-test-run[N].log
```

## See also
- `android-battery-measurement-toolkit.md`
- `bayesian-hypothesis-scoring.md`
