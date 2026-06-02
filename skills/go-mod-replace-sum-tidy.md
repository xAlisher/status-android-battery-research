---
id: go-mod-replace-sum-tidy
title: Remove orphaned go.sum entries after adding a replace directive
phase: build
type: gotcha
severity: medium
severity_reason: CI golangci-lint diffs go.sum after running its own tidy — orphaned entries fail the check silently locally but loudly in CI
modules: ["status-go"]
last_used: "2026-06-02"
created: "2026-06-02"
status: active
---

## Problem

After adding `replace github.com/X => github.com/Y vZ` to go.mod, the original module X's
two go.sum entries (`X h1:...` and `X/go.mod h1:...`) become orphaned. Local `go mod tidy`
may silently skip removing them if other tidy errors (e.g. missing generated migration
packages in a shallow clone) block full completion. CI golangci-lint runs its own tidy
and diffs go.sum — the orphaned entries cause a failure.

## Recipe

```bash
# 1. After editing go.mod, grep for orphaned entries
grep "github.com/X " go.sum    # look for the original module path

# 2. If found, remove both lines manually
# go.sum format: one h1 line + one /go.mod h1 line per version
grep -v "github.com/X " go.sum > go.sum.tmp && mv go.sum.tmp go.sum

# 3. Verify only the fork entries remain
grep -E "X|Y" go.sum

# 4. Commit go.sum together with go.mod
git add go.mod go.sum
git commit -m "chore(deps): add replace directive for X, remove orphaned checksum"
```

## Why

With a `replace` directive, Go resolves imports of X via Y. The original X hash is no
longer needed for verification and `go mod tidy` removes it. If local tidy fails for
unrelated reasons (shallow clone, generated file gaps), the removal is skipped and the
diff only surfaces in CI.

## See also

- `companion-pr-timing-race`
