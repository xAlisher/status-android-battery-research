# Status Android — Battery/CPU Research Skills
**Repo:** status-im/status-app
**Created:** 2026-06-02

---

## Architecture Facts

### Two Processes
Status runs TWO Android processes when in background:
```
app.status.mobile          ← Qt/Nim UI process (QML engine, timers, animations)
app.status.mobile:statusgo ← Separate foreground service (libstatus.so, Waku, status-go)
```
Always measure **both** separately. Most devs only look at one.

*Full lifecycle call chain with footnoted sources: `report.md § 2`.*

### What Gets Paused vs Not
- **Paused in background:** all pausable status-go services (except messaging)
- **NOT paused:** Waku light client receive (event-driven, always runs)
- **NOT paused:** "messaging" service (intentional — needed for push notifications)
- **Unknown/suspect:** Qt event loop in UI process — NO pause code found

### Key Files
| File | Purpose |
|------|---------|
| `StatusQtActivity.java:48-61` | onResume/onPause → setUiVisible |
| `StatusGoStub.java:47-58` | Bridge to Binder client |
| `StatusGoServiceClient.java:279-337` | Async Binder dispatch for setUiVisible |
| `StatusGoService.java:160-214` | PauseServices/ResumeServices dispatch |
| `StatusGoService.java:161` | Comment: messaging excluded from pause |
| `StatusGoService.java:188-191` | Comment: Waku excluded from pause |

---

## ADB Commands for Status-Specific Testing

### Identify both processes
```bash
adb shell ps -A | grep -E "app.status|statusgo"
```

### Monitor CPU per process in background
```bash
# UI process (Qt event loop suspect)
adb shell top -b -n 5 -d 10 | grep "app.status.mobile "

# status-go process (Waku/status-go)
adb shell top -b -n 5 -d 10 | grep ":statusgo"
```

### Check PauseServices actually fired
```bash
adb logcat -s StatusGoService -v time | grep -E "PauseServices|ResumeServices|error"
```

### Watch Waku network activity (per-minute)
```bash
# First: identify the active interface (wlan0=WiFi, rmnet_data0=mobile data on Samsung)
# adb shell cat /proc/net/dev | grep -v "lo:\|dummy\|sit\|p2p" | awk '{print $1, $2}'
IFACE=wlan0  # change to rmnet_data0 for mobile data tests (H2/T2)
NET1=$(adb shell cat /proc/net/dev | grep $IFACE | awk '{print $2}')
sleep 60
NET2=$(adb shell cat /proc/net/dev | grep $IFACE | awk '{print $2}')
echo "RX bytes in 60s: $((NET2 - NET1))"
```

### Battery stats per-app after soak
```bash
adb shell dumpsys batterystats --charged | grep -A10 "app.status"
```

### Force unplug (required for all battery tests)
```bash
adb shell dumpsys battery unplug
adb shell dumpsys batterystats --reset
# ... run test ...
adb shell dumpsys battery reset  # restore when done
```

---

## Known Gotchas

- **QML a11y tree empty until first user tap** — always tap screen before `uiautomator dump`
- **Qt bottom bars** — adb tap + MCP Click both fail, user must tap manually
- **Nav bar: y~2328** — never tap y > 2270 (system nav bar)
- **WiFi ADB setup** — requires USB first: `adb tcpip 5555`, then disconnect USB
- **Status app package:** `app.status.mobile`
- **WakuV2 options** — in Advanced Settings. Must be **Light mode** for normal users (Relay mode will always drain battery — expected)
