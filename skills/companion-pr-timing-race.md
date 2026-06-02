---
id: companion-pr-timing-race
title: Update companion PR submodule before or immediately after pushing to avoid Check timing race
phase: review
type: gotcha
severity: medium
severity_reason: status-go Check workflow triggers within seconds of a push and will fail with sha_mismatch if the companion PR still points to the previous commit
modules: ["status-go", "status-app"]
last_used: "2026-06-02"
created: "2026-06-02"
status: active
---

## Problem

The `Check` workflow on `status-im/status-go` PRs runs `scripts/check-companion-pr.sh`
within ~15s of a push. It looks up the companion status-app PR and verifies that PR's
`vendor/status-go` submodule SHA matches the just-pushed status-go commit. If you push
to status-go first and update the companion PR submodule minutes later, the Check will
have already run, seen a stale SHA, and failed.

## Recipe

```bash
# Pattern A — update companion first, then push status-go
cd ~/status-app-src/vendor/status-go && git checkout <new-sha>
cd ~/status-app-src && git add vendor/status-go && git commit -m "chore: bump status-go to <sha>"
git push origin <companion-branch>

cd ~/status-go-src && git push origin <branch>   # Check now sees correct companion SHA

# Pattern B — pushed status-go already, Check failed
gh run rerun <run-id> --repo status-im/status-go --failed
# Then immediately update companion PR and push BEFORE the rerun reaches check-companion-pr
```

## Why

`check-companion-pr.sh` reads the companion PR's HEAD commit and inspects `vendor/status-go`
at that tree — it does not poll; it's a point-in-time check triggered by the status-go
push event. There is no retry; a failed Check must be manually re-run.

## See also

- `go-mod-replace-sum-tidy`
