# Open Questions

Working register of open assumptions surfaced during architecture design + domain research. Each one needs either (a) ~30 seconds with Diana, or (b) a defensible default documented in `decisions/assumptions.md`.

Numbering continues from prior research: OQ-1 (tech onboarding patterns — resolved during disclosure-model selection; lean-hybrid recommendation informed ADR-002), OQ-2 (newest-agent disclosure-model preference — informally resolved by OQ-1).

Companion to: `decisions/design-calls.md` (closed implicit decisions). This file holds the open ones.

---

## OQ-3 — Default title company

**What we don't know.** Does Diana's team have a primary title-company relationship, or do they route per-client / per-deal?

**Why it matters.** Drives `04_transaction_coordinator`'s deal-active workflow — whose contact info appears in handoff content, where Option Fee and Earnest Money get delivered by default.

**Evidence.** `decisions/domain-research.md` §2.5 + A3 [evidence: medium]. Boutique teams in Austin typically have a primary relationship; not universal.

**Defensible default if Diana unavailable.** Parameterise the title-company field in `04_transaction_coordinator/handoff.md`; require it to be populated per case_id in the `cases/` folder at deal-execution time. Documented as a parameter, not a constant.

**Resolution.** 30-second ask to Diana, or ship the parameterised default.

**Status.** Open — non-blocking; shipped with defensible default. Resolve with Diana post-submission.

---

## OQ-4 — Investor / cash-buyer share of Diana's volume

**What we don't know.** What percentage of Diana's 60-80 transactions/year are investor or cash-buyer vs. financed owner-occupant?

**Why it matters.** Drives `01_lead_qualifier`'s field set. If >10% investor, the current intent-subtype list (`first-time / move-up / downsize / relocate / investor`) is too coarse — investors need BRRRR, 1031, depreciation, rental-yield dimensions that owner-occupants don't have.

**Evidence.** `decisions/domain-research.md` §2.2 + A5 [evidence: weak]. Working assumption: ~5-10% investor.

**Defensible default if Diana unavailable.** Treat investor as an edge case in `01_lead_qualifier/examples.md`. Investor inquiries that don't fit the qualifier's field set trigger `back_scope_mismatch` (ADR-001 `handoff_reason`) and route to orchestrator.

**Resolution.** 30-second ask to Diana, or ship with edge-case handling.

**Status.** Open — non-blocking; shipped with defensible default. Resolve with Diana post-submission.

---

## OQ-5 — Lead-source mix

**What we don't know.** Rough share of Diana's lead sources — referral, repeat business, online ads, walk-in, Zillow / Realtor.com.

**Why it matters.** Drives `00_orchestrator`'s primary-task framing. Repeat/referral-heavy → router's main job is *recognising returning clients* and routing them back to their specialist. Cold-ad-heavy → router's main job is *triage and qualification* of unknown inbound. These are different identity.md structures.

**Evidence.** `decisions/domain-research.md` §2.1 + A11 [evidence: medium]. Working assumption: >50% repeat + referral for a 4-person 60-80-tx/year boutique team in Austin.

**Defensible default if Diana unavailable.** Frame `00_orchestrator/identity.md` around routing returning clients as the primary task, with cold inbound as the secondary path. Document the assumption explicitly.

**Resolution.** 30-second ask to Diana, or ship with assumed mix.

**Status.** Open — non-blocking; shipped with defensible default. Resolve with Diana post-submission.

---

## OQ-6 — Intermediary status usage

**What we don't know.** Does Diana's team practice intermediary status (one broker representing both sides of a deal) or strict one-side-per-deal?

**Why it matters.** Drives the default value of `intermediary_status` envelope field (ADR-001). Wrong default could route handoffs in ways that violate TRELA without the required written consent + conspicuous bold/underlined prohibited-conduct disclosure.

**Evidence.** `decisions/domain-research.md` §3 + A14 [evidence: weak]. TX intermediary rules are strict.

**Defensible default if Diana unavailable.** Default `intermediary_status: false`. The envelope field is present (per Austin's schema, ADR-001); flipping to `true` requires Diana's explicit written-consent flow per TRELA.

**Resolution.** 30-second ask to Diana, or ship with `false` default + assumption note.

**Status.** Open — non-blocking; shipped with defensible default. Resolve with Diana post-submission.

---

## OQ-7 — Team Assignment Table rules

**What we don't know.** Which `qualified_lead.sub_type` + top `geography` combinations route to which named team member (beyond Diana herself).

**Why it matters.** Drives `01_lead_qualifier`'s forward emission. Until rules are filled, every lead defaults to `agent_on_deal: team_lead` (CALL-022 sentinel). Sustained sentinel firing means Diana's team-default voice carries every client-facing draft — over-burdening the team_lead voice file and masking under-population of the assignment table.

**Evidence.** A15 [evidence: weak]. Brief doesn't name team members beyond Diana; the assignment rule structure (sub_type + geography → agent) is industry-standard for boutique teams, but the specific rules belong to Diana.

**Defensible default if Diana unavailable.** Every forward emission uses `agent_on_deal: team_lead` (CALL-022 sentinel) → resolves to `voice/team_lead.md` (Diana's team-default voice per A8). System ships safely with no named rules.

**Resolution.** Diana fills the table with ≥3 named-agent rules during onboarding (typical: one specialty per non-Diana agent), or ship with the sentinel default.

**Status.** Open — non-blocking; shipped with defensible default. Resolve with Diana on intake.

---

## Triage

All five are non-blocking — shipped with defensible defaults documented in `decisions/assumptions.md`. Each can be resolved with Diana post-submission (OQ-3, OQ-4, OQ-5, OQ-6 in <30 seconds; OQ-7 during the onboarding table-fill).

**Priority order for Diana (if asked):** OQ-6 (compliance), OQ-3 (operational defaults), OQ-7 (assignment table), OQ-5 (orchestrator framing), OQ-4 (qualifier field set).
