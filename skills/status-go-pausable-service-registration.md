---
id: status-go-pausable-service-registration
title: Register a new service in status-go ServiceRegistry as Pausable
created: 2026-06-04
project: status-android-battery-research
---

## Problem
A service (e.g. `Messenger`) needs to respond to `PauseServices`/`ResumeServices` from the
mobile layer, but it doesn't implement `common.Pausable` and isn't in the `ServiceRegistry`.

## Recipe

1. **Create a wrapper** in `pkg/backend/node/service_registry.go` (follow `PausableMediaServer`):

```go
type PausableMessenger struct {
    common.PauseBroadcaster
    m *protocol.Messenger
}

func newPausableMessenger(m *protocol.Messenger) *PausableMessenger {
    p := &PausableMessenger{m: m}
    p.MarkStarted()
    return p
}

func (p *PausableMessenger) PausableName() string { return "messenger" }

func (p *PausableMessenger) Pause() error {
    p.m.SetPaused(true)
    p.MarkPaused()
    return nil
}

func (p *PausableMessenger) Resume() error {
    p.m.SetPaused(false)
    p.MarkResumed()
    return nil
}
```

2. **Register** in `populateServiceRegistry()` in `get_status_node.go`:

```go
if n.wakuV2ExtSrvc != nil {
    if m := n.wakuV2ExtSrvc.Messenger(); m != nil {
        n.serviceRegistry.Register(newPausableMessenger(m))
    }
}
```

3. Add `"github.com/status-im/status-go/protocol"` import to `service_registry.go`.

The Java layer calls `PausableServices` to get names, then `PauseServices`/`ResumeServices`
with the full list — once registered, the service is automatically included.

## Why
`PausableName()` is the map key used by `PauseMultiple()`. Name must be stable.
`common.PauseBroadcaster` provides `PausableState()` and subscriber broadcasting for free.
Do NOT name it "messaging" — that name is reserved by the `wakuV2ExtSrvc` itself.

## See also
- `android-energy-code-smells.md` — Pattern 5: service not registered as Pausable
- status-go PR #7516 (jrainville) — production reference
