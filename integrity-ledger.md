# Design + Rationale Ledger
**Repo:** `xAlisher/status-android-battery-research`
**Purpose:** Track design decisions, removals, and trade-offs across integrity loop iterations.

---

## Repo Structure

See `README.md` for the annotated file tree.

---

## Design Decisions and Rationale

| ID | Decision | Rationale |
|----|----------|-----------|
| D1 | `report.md` and `research-protocol.md` are separate files | Report = this investigation's findings. Protocol = reusable for any Android battery investigation. Different audiences, different update cadence. |
| D2 | `journal.md` kept in repo | Contains fails log and raw session data. Fails log is load-bearing — future researchers need to know what didn't work and why. |
| D3 | Skills live in `skills/` separate from `research/` | Skills are domain-agnostic and reusable across projects. Research files are Status-specific. |
| D4 | `status-android-skills.md` lives in `research/` not `skills/` | It is Status-app-specific (process names, test profile password, nav bar coordinates) — not portable. |
| D5 | `INDEX.md` contains a quick ADB reference section | Reduces need to open 5 skill files just to find a command. Entry-point convenience. |
| D6 | Bayesian probability scores on every hypothesis | Enables prioritisation by expected impact, not subjective labels. Evidence: arxiv:2212.13773 |
| D7 | N=5 runs + Mann-Whitney U in validation tests | Single-run measurements are not statistically valid for battery research. Evidence: arxiv:2604.25587 |
| D8 | Mann-Whitney U python snippet included inline in report.md | Researcher must be able to execute the analysis without hunting for the method. |
| D9 | Interim report protocol included in both report.md and skills/battery-research-interim-reporting.md | report.md version = condensed trigger table. Skill file = full format with examples. Different granularity. |
| D10 | Time estimates in research-protocol.md | Protocol is the operational doc; estimates belong there, not in the scientific report. |
| D11 | H4 (messaging service) included as hypothesis | Referenced in MH5 diagram — undefined H4 would be a broken reference. Low confidence (40%) but honest. |
| D12 | H6 (unexplained drain gap) included as [? — unknown] | Model predicts 6%/hr, observed 8.1%/hr. Honest accounting — gap must be documented, not ignored. |
| D13 | `research-protocol.md` includes Perfetto commands | Perfetto is the modern replacement for Battery Historian. Researchers need it inline, not in a separate skill. |
| D14 | `skills/INDEX.md` includes a duplicate ADB quick reference | Skills are meant to be opened individually. INDEX convenience section prevents needing to open toolkit file for basic commands. |

---

## Loop Log

### Loop 0 — Baseline
- No changes yet
- Integrity issues fixed pre-commit: H4 undefined, ghost file ref, wrong project name in frontmatter, absolute path in protocol

### Loop 1 — Redundancy + Adversarial pass

**Redundancy agent (R1–R20): 20 findings**
**Adversarial agent (F1–F6, M1–M6, LB1–LB5): 6 failures, 6 missing, 5 load-bearing**

**Supervisor decisions — executed:**

| Item | Decision | Rationale |
|------|----------|-----------|
| R1 | Remove journal.md "Two Processes Running" block | Pure restatement; report.md §2 is canonical. LB3 protects status-android-skills.md copy — kept. |
| R2, R3 | Remove journal.md lifecycle chain + Waku block | Pure restatement. alexjba analysis (session reasoning) kept. |
| R4 | Remove keyboard timer code block from journal FINDING A | Restatement. Replaced with summary + pointer. |
| R5 | journal.md MH1 JNI figure → cross-reference | README.md headline kept; journal derivation replaced with pointer. |
| R7 | Mann-Whitney snippet in protocol → pointer | Duplicate of skill. Pointer already existed immediately below. |
| R8 | Scoring table in protocol → pointer | Skill file is more complete (+Bayesian updates, calibration table). |
| R9 | Interim format block in report §7 and protocol → pointer | Skill file is canonical. Trigger table in report §7 kept (project-specific). |
| R10 | DoD lists in report §7 and protocol → pointer | Skill file has 8-point version. No project-specific additions in removed lists. |
| R11 | Escalation list in protocol → pointer | Skill file has 6-point version with escalation note item. |
| R12 | Peer review section in protocol → removed | Skill file covers it. |
| R13 | Mann-Whitney snippet duplicate in protocol → single pointer | Pointer already existed on same page. |
| R16 | File tree in ledger → pointer to README | README is the public entry point. |
| R17 | journal.md H1b restatement → pointer | Promoted to report.md §3. |
| R18 | journal.md MH3 → pointer | Promoted to report.md §4. |
| R19 | journal.md H4 block → pointer | Promoted to report.md §3. |
| R20 | tcpdump line in journal → replaced with /proc/net/dev | Factual conflict with anti-patterns section. |
| R14 | journal.md H2 in Hypotheses list → merged with H1b/H5 removal | All three simple hypothesis restatements removed; session-specific chain reasoning (MH2) kept. |
| R15 | journal.md H5 restatement → pointer on FINDING C | |
| F2 | H1 score corrected 75% → 65% | Adversarial agent correctly identified alexjba's comment is in GitHub issue, not source code. Scoring framework awards +20 for source comments only. |
| F3 | R2 recommendation rewritten | Blanket suppress would break Waku reconnect. Code snippet removed. Replaced with selective rate-limit approach description. Risk reclassified Medium. |
| F5 | Joule formula fixed 3800mAh×4.4V → 4500mAh×3.85V | S20 FE has 4500mAh. Old formula gave wrong capacity with coincidentally correct result. |
| F6 | T3 caveat added | T3 confirms mechanism (code path fires), not drain impact. Added explicit note. |
| M3 | Pre-T1 Qt process survival check added | Without this, killed process produces false REJECTED verdict for H1/H1a. |
| M5 | Wake lock capture added to T1 and T4 | Foreground service wake locks don't appear in CPU% measurements. |
| M6 | INCONCLUSIVE path specified for 0.5–2% zone | Added concrete next step: extend to 15min, then Perfetto trace. |

**Deferred to Loop 2 (load-bearing, needs second look):**
- R13 ADB quick ref trim in protocol — adversarial agent LB5 protects anti-patterns; trim boundary needs verification
- R6 README state tag table — adversarial LB1 says tag system is load-bearing; README version may be needed for entry-point readers

---

### Loop 2 — Redundancy + Adversarial pass

**Redundancy agent: 8 findings (R1–R8)**
**Adversarial agent: 5 failures, 5 missing, 5 load-bearing (F1–F5, M1–M5, LB1–LB5)**

**Supervisor decisions — executed:**

| Item | Decision | Rationale |
|------|----------|-----------|
| R3 | Remove journal.md "Background Lifecycle Path" ASCII chain | Report.md §2 is canonical. Replaced with pointer + footnote note. |
| R2 | Remove INDEX.md threshold comment block (CPU %>2%/5%, RX bytes/min >50KB) | Thresholds defined in android-controlled-battery-experiment.md and android-battery-measurement-toolkit.md. INDEX quick ref is for commands, not thresholds. |
| R7 | Remove inline tcpdump NOTE comment from research-protocol.md Waku ping code block | Anti-patterns section (line 445) already states "Don't use tcpdump on stock Samsung." Inline comment was duplicate. |
| R8 | Replace journal.md MH5 drain model prose with pointer to report.md §4 MH5 | report.md §4 MH5 is canonical. Journal version was unsourced arithmetic with wrong midpoint sum (claimed 6%/hr, actual 4%/hr). |
| F3 | H2 score corrected 65% → 45% | Used +40 (timer no stop path) for source comment. Correct category is +20. Fix-order inversion: H1a expected impact (70%×2%/hr = 1.4%/hr) now exceeds H2 (45%×3%/hr = 1.35%/hr). |
| F4 | T2 rewritten to use mobile data interface (rmnet_data0) | H2 is a cellular radio sleep hypothesis. wlan0 shows 0 bytes on cellular. Test was structurally invalid — would have produced false REJECTED for H2. |
| M5 | Pattern 4 fix corrected in android-energy-code-smells.md | processEvents(WaitForMoreEvents) blocks calling thread — causes CPU spin/deadlock, opposite of suspending. Replaced with applicationStateChanged handler guidance + explicit warning against WaitForMoreEvents. |
| F1 | Charging time added to time estimates table | N=5 runs require 4 recharges × ~75min = ~300min idle. Wall clock = ~11–15h, not ~8–10h active. |
| LB5 | MESSAGING_REPAUSE_DELAY_MS finding promoted to report.md H4 | Was only in journal.md DELTA 2 — would be lost if journal trimmed. Now load-bearing reference in H4 section. |
| R6 | README state tag table removed, replaced with pointer | Full table now in report.md header block (canonical). README kept "A claim without a tag is not a finished claim" + pointer. |
| F-scipy | scipy version pinned ≥1.6.0 | alternative='less' added in 1.6.0. Unpinned would silently fail on older envs. |
| F-H4 | H4 updated with MESSAGING_REPAUSE_DELAY_MS amplifier | Build-specific constant identified in journal DELTA 2. Explicit note: value may differ across builds. |
| F-screen | KEYCODE_POWER screen state note added to T1 setup | Screen-on vs screen-off has significant Android power management differences. Must be screen-locked for valid drain measurement. |

**Deferred to Loop 3 (load-bearing, unchanged from agent reports):**
- T2 no new deferrals — all Loop 2 items executed

---

## Removals Log

*See Loop 1 and Loop 2 supervisor decisions above.*

---

## Trade-offs Log

*(exit condition: loop ≥ 3 AND this section is the full output)*

**T1 — Android 13 vs Android 15 test device gap** (F1)
- Test device: S20 FE, Android 13. Reporter device: S21 Ultra, Android 15.
- Android 15 has stricter background process management. Findings on Android 13 may understate drain on Android 15 — or conversely, Android 13 may allow Qt process survival that Android 15 would prevent (producing a false negative on H1/H1a).
- **Cannot be resolved without a second device running Android 15.**
- Mitigation: all verdicts stamped "confirmed on Android 13" and cross-referenced to Open Question 3.

**T2 — Drain model uses uncalibrated component estimates** (F4)
- "~1–2% CPU/hr" and "~2–3%/hr" for H1a and H2 are order-of-magnitude estimates, not measurements.
- The model sums midpoints to ~4%/hr but the report claims ~6%/hr, implying upper-bound arithmetic or untracked contributions.
- H6 (2.1%/hr gap) is a direct consequence — it equals "observed minus model," and the model was not independently derived.
- **Cannot be resolved without device measurements (T1–T5).**

**T3 — H2 unactionable without vendored status-go source** (M1)
- R4 recommendation ("audit Waku ping interval") cannot be executed without the status-go source at the exact vendored commit.
- This is the second-highest expected-impact hypothesis (1.95%/hr).
- **Resolution requires: cloning status-go at the vendored commit hash in `go.sum`/`go.mod`.**
- Blocked by upstream access — outside scope of this repo.

**T4 — Peer review step requires human coordination** (M2)
- Definition of done requires cross-check with mag and Sale's parallel investigation.
- Neither person is identified in the repo beyond their GitHub usernames.
- **Cannot be automated or resolved within this repo.** Action item for Alisher to complete manually before v1.0.
