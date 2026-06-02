---
id: android-battery-measurement-toolkit
title: Choose and use the right Android battery measurement tool
phase: ops
type: pattern
severity: high
severity_reason: Wrong tool gives misleading data — e.g. measuring while charging, or missing the second process entirely
source: extracted-from-research + external-literature
upstream_url: https://developer.android.com/topic/performance/power/battery-historian
last_used: 2026-06-02
created: 2026-06-02
status: active
---

## Problem
Android offers multiple battery/CPU measurement tools. Picking the wrong one wastes a test run or produces data you cannot act on. Each tool has different overhead, availability, and resolution.

## Tool Decision Tree

```
Is the device a Pixel or Qualcomm Snapdragon?
├─ YES → Perfetto with ODPM (hardware power rails) — most accurate
└─ NO  → dumpsys batterystats — always available, sufficient for root cause

Is the drain > 5 min or overnight?
├─ YES → bugreport + Battery Historian (visual timeline) OR dumpsys --charged
└─ NO  → adb shell top + /proc/net/dev snapshots (fast, no overhead)

Do you need per-subsystem breakdown (CPU vs radio vs display)?
├─ YES → Perfetto systrace
└─ NO  → dumpsys cpuinfo + dumpsys power (wakelocks)
```

## Tool Reference

### 1. dumpsys batterystats — always available, start here
```bash
adb shell dumpsys batterystats --enable full-wake-history  # enable before test
adb shell dumpsys batterystats --reset                     # reset before test
# ... run test ...
adb shell dumpsys batterystats --charged > /tmp/stats.txt  # capture after
grep -E "im.status|wakelock|cpu|network" /tmp/stats.txt
```
**Red flags:** wakelock > 1h/session, radio more frequent than every 30s, JobScheduler < 30s intervals

### 2. adb shell top — fastest, no setup
```bash
# One snapshot
adb shell top -b -n 1 | grep -E "im.status|statusgo"

# 5-min continuous sampling (every 30s)
for i in $(seq 1 10); do
    echo "=== T+$((i*30))s ===" && date
    adb shell top -b -n 1 | grep -E "im.status|statusgo"
    sleep 30
done
```
**Threshold:** UI process > 2% sustained with screen locked = event loop running.

### 3. Perfetto — best for systemic root cause, Android 10+
```bash
# Start trace (30 second window)
adb shell perfetto -o /data/misc/perfetto-traces/trace.pftrace \
  -t 30s sched freq idle am wm gfx view binder_driver hal dalvik \
  camera audio video power

# Pull trace
adb pull /data/misc/perfetto-traces/trace.pftrace /tmp/
# Open at https://ui.perfetto.dev
```
**SQL to find wakelock root cause:**
```sql
SELECT t.name, SUM(dur)/1e9 as total_sec
FROM slice s JOIN track t ON s.track_id = t.id
WHERE t.name LIKE '%wakelock%'
GROUP BY t.name ORDER BY total_sec DESC
```
**SQL to find radio-hungry app:**
```sql
SELECT uid, SUM(packet_count) as total_packets
FROM network_packets GROUP BY uid ORDER BY total_packets DESC LIMIT 10
```

### 4. Battery Historian — visual timeline (deprecated but still useful)
```bash
adb bugreport /tmp/bugreport.zip
# Upload to https://bathist.ef.lc/ (community mirror) or run locally:
# docker run -p 9999:9999 gcr.io/android-battery-historian/stable:3.0 --port 9999
```
**Note:** No longer maintained by Google. Use Perfetto for new work.

### 5. /proc/net/dev — network bytes, no overhead
```bash
NET1=$(adb shell cat /proc/net/dev | grep wlan0 | awk '{print $2}')
sleep 300
NET2=$(adb shell cat /proc/net/dev | grep wlan0 | awk '{print $2}')
echo "RX bytes in 5min: $((NET2 - NET1))"
# > 250KB in 5min background = active network communication
```

### 6. dumpsys power — wake locks
```bash
adb shell dumpsys power | grep -A3 "Wake Locks:"
# PARTIAL_WAKE_LOCK = CPU stays on even with screen off
# Count > 3 unexplained = investigate
```

## Status-Specific Notes
- App runs TWO processes: `im.status.ethereum` (Qt/UI) AND `im.status.ethereum:statusgo` (Waku/Go)
- Always measure both separately — they have independent CPU profiles
- `tcpdump` NOT available on stock Samsung Android — use `/proc/net/dev` instead
- Trepn Profiler: requires Qualcomm Snapdragon AND Qualcomm's app from Play Store. S20 FE uses Snapdragon 865 — hardware is compatible. **[? — unverified: app must be installed and device must not have Play Store blocked. Do not rely on Trepn until confirmed working on the test device. Fallback: Perfetto is sufficient and always available.]**

## See also
- `android-controlled-battery-experiment.md`
- `android-energy-code-smells.md`
- `../research/status-android-skills.md` (Status-specific ADB commands)
