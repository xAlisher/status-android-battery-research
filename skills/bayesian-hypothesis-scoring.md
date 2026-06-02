---
id: bayesian-hypothesis-scoring
title: Score and update hypothesis probability using Bayesian updating during debugging research
phase: discovery
type: pattern
severity: medium
severity_reason: Subjective HIGH/MEDIUM/LOW labels cannot prioritise fix effort or communicate risk to developers
source: extracted-from-research + arxiv:1811.05422 + arxiv:2212.13773
upstream_url: https://arxiv.org/pdf/2212.13773
last_used: 2026-06-02
created: 2026-06-02
status: active
---

## Problem
Saying "HIGH confidence" is subjective and non-actionable. A developer cannot decide which hypothesis to fix first without a relative probability. Bayesian updating provides a principled way to assign and revise probabilities as evidence accumulates.

## Framework

### Step 1 — Assign prior probability (before any measurement)

Score each hypothesis based on code evidence:

| Evidence type | Points |
|--------------|--------|
| Timer/loop starts unconditionally, no stop path in code | +40 |
| Explicit comment in source confirms behaviour | +20 |
| Architectural pattern known to cause this class of problem | +15 |
| Analogous known issue confirmed elsewhere in same codebase | +10 |
| Guard code exists but may not fire (conditional) | −15 |
| Inference only — no direct code evidence | −10 |
| Contradicting code (explicit stop/check) | −25 |

Cap at 95% (measurement required to reach certainty). Floor at 5%.

**Example — keyboard timer (H1a):**
- Timer starts unconditionally, no stop path: +40
- No background guard found anywhere: +20 (architectural pattern)
- Analogous issue confirmed in Qt Android (known): +15
- No contradicting code: +0
- **Prior: 75%** → round to 70% for conservatism

### Step 2 — Update on measurement (Bayesian update)

```
P(H|evidence) = P(evidence|H) × P(H) / P(evidence)
```

In practice, use these update rules:

| Measurement result | Update |
|-------------------|--------|
| Measurement confirms predicted signature exactly | Prior × 1.5, cap at 95% |
| Measurement shows signal but weaker than predicted | Prior × 1.2 |
| Measurement is inconclusive / noisy | No change |
| Measurement contradicts prediction | Prior × 0.3 |
| Multiple independent measurements all confirm | Prior → 95% |

**Example — H1a post-measurement:**
- Predicted: UI process > 2% CPU with WiFi off, screen locked
- Measured: 8.2% CPU (much higher than threshold)
- Update: 70% × 1.5 = 95% → `[CONFIRMED — 95%]`

### Step 3 — Tag every claim

Use these tags in reports and journal entries:

| Tag | Meaning |
|----|---------|
| `[CODE — X%]` | Directly visible in source, probability from code evidence |
| `[H — X%]` | Hypothesis with prior probability |
| `[H — X% → Y%]` | Prior updated by measurement |
| `[CONFIRMED — X%]` | Measurement confirmed, posterior probability |
| `[REJECTED]` | Measurement contradicted hypothesis |
| `[INCONCLUSIVE]` | Measurement was noisy or wrong metric |
| `[? — unknown]` | Cannot determine without external data |

### Step 4 — Prioritise fixes by expected impact

```
Expected impact = probability × estimated drain (% battery/hr if confirmed)
```

Example ranking for Status #21045:

| Hypothesis | Probability | Est. drain if true | Expected impact |
|-----------|------------|-------------------|----------------|
| H1a (keyboard timer) | 70% | 2%/hr | **1.4%/hr** |
| H2 (Waku pings) | 65% | 3%/hr | **1.95%/hr** |
| H1b (net callback) | 55% | 1%/hr | **0.55%/hr** |
| H5 (wallet cascade) | 45% | 1%/hr | **0.45%/hr** |

Fix order: H2 → H1a → H1b → H5 (by expected impact).

Note: H1a has a higher probability but H2 has higher drain if true — prioritise by expected impact, not raw probability alone.

## Prior Probability Calibration Table

Use these as starting points for common Android battery patterns:

| Pattern | Calibrated prior |
|---------|-----------------|
| QTimer with no stop() in background | 70% causes drain |
| ConnectivityManager callback with no guard | 60% contributes to drain |
| Waku/P2P keepalive on mobile data | 65% causes radio drain |
| Background service not in PausableServices | 55% causes drain |
| HashMap in hot path | 30% measurable impact |
| GC pressure in background | 25% measurable impact |

## What This Is NOT

This is not rigorous Bayesian inference — it doesn't require computing exact likelihoods. It's a disciplined way to:
1. Force explicit probability estimates (vs vague labels)
2. Update estimates as data arrives (vs ignoring new evidence)
3. Prioritise by expected value (vs fixing the "most obvious" thing first)

## See also
- `android-controlled-battery-experiment.md`
- `android-energy-code-smells.md`
