# Retro Log — Status Android Battery Research

<!-- Append wins/fails here during sessions via /log win and /log fail -->

## Session 7 — 2026-06-03/04 (APK Build, Soaks, PR Review Fixes, Closure)

### Wins

- [process] Docker APK build environment cracked after 5 sequential failures — each one distinct: (1) root permission for jenkins uid, (2) git safe.directory whitelist, (3) cache volumes routed to wrong HOME (/root vs /home/jenkins), (4) version.sh shallow clone fix (git tag HEAD), (5) mobileui Maven publish step missing. Each fix was additive, no backtracking.
- [process] Reviewer feedback to fold SetAppBackground into existing ToBackground/ToForeground was correct — we accepted it immediately, refactored cleanly, removed 18 lines net.
- [process] PausableMessenger wrapper pattern implemented correctly — parallel to PausableMediaServer, wires Pause()/Resume() to ToBackground()/ToForeground() so Java PauseServices/ResumeServices covers Messenger without new API surface.
- [process] Closed PRs professionally: closing comment on #7508 preserved all measurement data in a table, linked to Jo's clean replacement. Good record for git blame readers.
- [project] T5b soak with all fixes (R1b+R2+R3): 15.8% modem duty cycle vs 55% baseline, even at weak signal (-120 dBm). Mailserver sync storm eliminated.
- [project] Jo's #7516 confirmed our R2 approach was right; his cleanup (SetPaused/isPaused instead of separate backgroundMode) is architecturally cleaner — no new field needed when existing paused atomic already tracks the state.

### Fails

- [process] T5b soak ran 88 min but PIDs were empty throughout — get_pid couldn't find processes, so all CPU% values were blank. Root cause: didn't verify PID capture with a test run before committing to the 70-min soak.
- [process] Left unused `errors` import in services/ext/api.go after removing SetAppBackground — Copilot flagged it. Root cause: didn't run gofmt/vet after edit, and the build error was masked by pre-existing missing-migration failures.
- [project] PROJECT_KNOWLEDGE.md had wrong entry: "PauseServices excludes messaging explicitly — use separate backgroundMode flag." Jo's #7516 disproves this — isPaused() works fine once Messenger is registered via PausableMessenger. Entry needs correction.
- [process] Docker cache volume paths set to /root/.cache initially (assumed default HOME) but container runs with -e HOME=/home/jenkins. Root cause: didn't check where Go writes its cache when HOME is overridden.
