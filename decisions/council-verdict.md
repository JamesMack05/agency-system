# Council Verdict — Comp 4 ADRs (2026-05-12)

## Verdict
**ITERATE-WITH-FIXES**

Verdict-rule clause: *"Iterate-with-fixes — at least one role flagged a blocker; fix the blockers and re-run if budget allows."* Principle-Purist flagged one block (P8 AgentBench + Mistake 7 — no trajectory eval named before build). Skeptic and Optimist flagged no blocks but raised multiple iterate-grade findings, several convergent with Purist's iterates. The architecture itself is not structurally wrong; the gap is verification rigor and a small number of contract-text defects.

> **CLOSURE NOTE (added 2026-05-12 post-drafting):** All iterate findings + the P8 BLOCK closed within the same compressed drafting cycle. BLOCK closure: `decisions/trajectories.md` (3 trajectories with shared 3-layer pass criteria; T1 hand-walked end-to-end). Iterate closures: ADR-001 §Extension 2 + §Layering principle + §Absence-vs-value (C2 / Skeptic Issues 1 + 3); ADR-003 §examples.md Pair-count deviation + §Per-specialist Rule 0 demoted to CALL-015 (Optimist Finding 3 + Skeptic ADR-003 finding); `decisions/architecture-properties.md` standalone artifact (C1); `03_client_communication/rules.md` §3 soft-degrade (CALL-020). The "Before Thursday" / "Before Saturday" fix-priority labels below are historical priority markers — all are now closed. No re-council run required (per §Loop decision below).

## Severity tally

| Role | Block | Iterate | Nit |
|------|------:|--------:|----:|
| Skeptic | 0 | 7 (3 top + 4 other) | 2 |
| Optimist | 0 | 3 | 3 |
| Principle-Purist | 1 | 2 | several pass/N/A |

## Convergent findings (>=2 roles agreed)

### C1 — Cross-document/system-level architectural property is under-surfaced
**Roles:** Optimist Finding 1 (closed refusal-taxonomy loop never named) + Principle-Purist P4/P6 partial (T/A/O cycle + global topology implicit, not declared) + Skeptic "Other finding" on ADR-002 boundary test having no enforcement.
**Severity:** iterate (convergent → high confidence).
**Fix priority:** Before Thursday. Cheap (3-5 line additions); high payoff for criterion #2 / #4.
**Fix:** Add cross-ADR property block naming (a) the 4-document refusal-taxonomy loop (ADR-001 Ext 2 ↔ ADR-003 §rules.md / §identity.md §6 / §handoff.md §4), and (b) the global topology + specialist execution loop (4-line T/A/O contract + 1-paragraph topology in ADR-002 or new sub-ADR).

### C2 — `handoff_reason` enum design under-claims or under-covers domain reality
**Roles:** Skeptic Issue 2 (missing `back_compliance_block` for UPL/TRELA) + Optimist Finding 2 (`forward_urgent` is the TREC calendar-day handle, currently buried).
**Severity:** iterate. Different angles on the same enum — Skeptic says a value is missing; Optimist says an existing value's domain rationale is under-claimed. Both rest on the same root: enum design and domain research aren't fully reconciled in the artefact.
**Fix priority:** Before Thursday. Both fixes touch the same paragraphs in ADR-001 Extension 2.
**Fix:** (a) Add `back_compliance_block` with worked examples from `decisions/domain-research.md` §2.2/§2.4/§3, increment `schema_version`; OR argue in Options why compliance refusals roll into `back_scope_mismatch` and commit to the conflation. (b) Add the TREC calendar-day rationale sentence under `forward_urgent`.

## Single-role findings

### Skeptic-only

- **[iterate] Issue 1 — Envelope-level Rule 0 names a specialist folder (`03_client_communication`) inside a folder-agnostic invariant.** Layering defect: the 4th predicate duplicates ADR-003's per-specialist Rule 0 and contradicts the processable/performable distinction the ADRs draw. Fix priority: **Before Thursday** (2-line predicate rewrite; cheap; strengthens criterion #2).
- **[iterate] Issue 3 — Rule 0's `content_provenance` predicate inverts the driver.** Quarantines on field-absence rather than on `anonymous_inbound` value. Contract layer fails to implement what field 14 claims to drive. Fix priority: **Before Thursday** (2-line rewrite).
- **[iterate] ADR-002 boundary test has no reviewer-side enforcement.** Concrete fix: add a banned-words grep as pre-submission QA (lexical-rules-as-final-QA, not lexical-rules-as-design). Fix priority: **Before Saturday** (pre-submission script).
- **[iterate] ADR-002 sequential-reading mitigation depends on README structure that is out of scope per CALL-013.** Either add a narrow README-link-placement constraint or reframe the consequence. Fix priority: **Before Saturday**.
- **[iterate] ADR-003 lock of 3-4 comparative pairs ignores domain content volume** — Research §2.5 names 8+ failure modes for `04_transaction_coordinator`; pair count cannot exhibit them. Fix priority: **Before Thursday** (let pair count vary per specialist, or add comparative-table section for `04`).
- **[iterate] ADR-003's >5-items failure-mode-register revisit trigger fires immediately for `04_transaction_coordinator`.** Same root as above. Fix priority: **Before Thursday** (acknowledge in the ADR or pre-split `failures.md`).
- **[nit] `03_` Rule 0 misses the soft-degrade voice-file mode.** Drafting note for Thursday, not an ADR change. Fix priority: **Before Thursday drafting**.
- **[nit] Quick-reference card for `handoff_reason` enum** would reduce CALL-019 copy-paste tax. Belt-and-braces. Fix priority: **Post-submission**.

### Optimist-only

- **[iterate] Finding 3 — ADR-003 over-locks 5 verbatim per-specialist Rule 0 wordings; CALL-015 buys brittleness against a low-cost problem.** Domain research already flags `04_transaction_coordinator` wording may need update (TRELA §1101.563 buyer-rep). Fix priority: **Before Thursday** (move 5 wordings to non-normative appendix or CALL entry; keep ADR-003's structural claim).
- **[nit] ADR-002's three zones could collapse to one-sentence invariant** ("README is the only bilingual doc"). Fix priority: **Post-submission**.
- **[nit] `decisions/` folder is itself a criterion #4 exhibit — ADRs cite criterion #2 only.** One-line addition per ADR. Fix priority: **Before Saturday**.
- **[nit] Graceful-degradation as a first-class envelope state (typed via `back_quality_failure` + `do_not_send_yet`) is under-claimed.** Fix priority: **Post-submission**.

### Principle-Purist-only

- **[BLOCK] P8 (AgentBench) + Mistake 7 (build-before-using) — no trajectory eval named before build.** Three ADRs lock substantial up-front design; Diana stranger-onboarding test is final QA, not pre-build sanity check. The good-enough floor mandates "2+ real trajectories pass on primary use cases" as the gate. Fix priority: **Before Thursday's drafting** for trajectory specification; **hand-walk one before Saturday's build**.
- **[iterate] P4 (T/A/O cycle) + P6 (interaction topology) implicit, not specified.** Triangulates with Optimist Finding 1 — see C1 above.
- **[iterate / nit] P1 (MemGPT memory tiers) — working/archival/recall split implemented but not declared.** No compress/eviction rule for unbounded `trail[]`. Fix priority: **Before Saturday** (1-line tier declaration + `trail`-truncation revisit trigger), or accept N/A-by-type with justification.

## Fix priority

### Before Thursday's drafting (2026-05-14)
These block specialist drafting because the contract text or enum set will be referenced verbatim in `handoff.md` / `rules.md` files.

1. **Lock 2-3 trajectories + pass criteria** (Purist BLOCK). Happy-path forward-only; one 03→01 back-handoff; one 04→03 deadline back-handoff. Pass criteria per trajectory (envelope well-formed under ADR-001; refusal triggers fire correctly per ADR-003; analogy doesn't leak per ADR-002). Hand-walk closes both BLOCK and Mistake 7 partially.
2. **Decide `handoff_reason` enum coverage** (C2 / Skeptic Issue 2 / Optimist Finding 2). Either add `back_compliance_block` and bump `schema_version`, or commit-and-justify the conflation. Add the TREC `forward_urgent` rationale sentence either way.
3. **Fix Rule 0 contract-text defects** (Skeptic Issues 1 + 3). Pull `agent_on_deal`/`03_client_communication` predicate out of envelope Rule 0; rewrite the `content_provenance` predicate to fire on value not absence. ~4 lines combined.
4. **Demote 5 per-specialist Rule 0 wordings** (Optimist Finding 3) to non-normative appendix or CALL entry so domain-research-driven edits don't shape as ADR amendments.
5. **Acknowledge `04_transaction_coordinator`'s pre-fired revisit trigger** (Skeptic findings on pair count + register size). Either widen pair count for `04` or pre-split to `failures.md`.
6. **Drafting note: `03_` Rule 0 + voice-file degrade pattern** (Skeptic nit). One-line for Thursday's `rules.md` §3.

### Before Saturday's repo build (2026-05-16)
These can wait until artifact-shipping day.

1. **Add cross-ADR property block** (C1). 3-5 line subsection naming the closed refusal taxonomy + global topology + T/A/O specialist execution loop. Highest-leverage criterion #2/#4 framing improvement.
2. **Hand-walk one trajectory before build** (closes Purist BLOCK on the verification side). 30-60 min cost.
3. **ADR-002 boundary-test pre-submission grep** (Skeptic). Banned-words script for "binder"/"sticky-tab"/"chain-of-custody" in `decisions/` + `HANDOFF_SCHEMA.md`.
4. **ADR-002 README-link constraint OR consequence reframe** (Skeptic).
5. **One-line tier declaration + `trail`-truncation revisit trigger** (Purist P1).
6. **One-line criterion #4 claim per ADR** (Optimist nit).

### Post-submission revisit (after 2026-05-17)
1. ADR-002 one-sentence invariant collapse.
2. Quick-reference card for `handoff_reason` enum.
3. Graceful-degradation as first-class envelope state — claim as design property.
4. Surface Reflexion intellectual-lineage in README prior-art (Purist note).

## Meta-findings (if any)
None. All Phase-2 findings are on engineering grounds. Notable: Principle-Purist explicitly applies N/A-by-type justification (P2 Voyager / Mistake 1 / Mistake 3) rather than force-fitting principles — good rigor. Optimist's "criterion #4 framing" suggestions are surface-level wording changes that the Purist also touches under P5 attribution — they are about claiming earned engineering work, not adding optics.

## Loop decision

**Recommendation: Fix and proceed without re-running council.** Reasoning:

1. The single BLOCK (Purist P8/Mistake 7) is a verification-rigor gap, not an architecture defect. Fix is procedural (lock + run trajectories), not a redesign — re-running council on the same ADRs won't surface new information once the trajectory eval is committed.
2. Convergent iterate findings (C1, C2) are text-edit fixes (~5-15 lines total across three ADRs); the architecture they sit on isn't being renegotiated.
3. Skeptic Issues 1 + 3 are 4-line contract-text rewrites that should be made directly — re-review would just re-confirm them.
4. Loop budget context: this is Tuesday 2026-05-12; Thursday is drafting; submission deadline is the following Sunday. A second council pass on the same artefacts within this window would burn capacity that should go into Thursday's drafting + Saturday's build + the trajectory hand-walk.
5. The Purist's recursive boundary-test audit already showed the ADRs themselves pass ADR-002's own test (zero binder-analogy contamination). The high-confidence structural checks pass.

**Re-run council only if:** the trajectory hand-walk on Saturday surfaces a schema-vs-real-flow mismatch that forces ADR-001 or ADR-003 amendment beyond text fixes. At that point the artefact is materially different and adversarial review is warranted again.
