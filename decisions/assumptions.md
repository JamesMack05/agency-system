# Assumptions

Documents assumptions baked into the architecture and per-specialist designs. Each has evidence and a revisit trigger. Assumptions marked **[OPEN]** are tracked in `decisions/open-questions.md` until resolved with Diana or shipped with defensible defaults.

**Evidence legend:**
- **strong** — cited authoritative source (TREC, TRELA, NAR, Texas REALTORS®)
- **medium** — inferred from one or two industry-practice sources
- **weak** — working knowledge, single-blog source, or industry folk-wisdom

---

## System-level assumptions

### A1 — Regulatory framework

**Assumption.** The team operates exclusively under TRELA (Texas Real Estate License Act) and uses TREC promulgated forms.

**Evidence.** Strong. Diana's team is in Austin TX; brief specifies residential. TRELA + TREC are the regulatory framework (`decisions/domain-research.md` §3).

**Revisit trigger.** Diana expands out of Texas, or onto a contract type not covered by TREC promulgated forms (e.g. commercial).

---

### A6 — Single transaction coordinator per deal

**Assumption.** Each deal has one transaction coordinator (in-house or contracted), not parallel TCs.

**Evidence.** Medium. Industry standard at 4-person scale per Dotloop and Tom Ferry team-build guides (`decisions/domain-research.md` §1).

**Revisit trigger.** Team scales past 15 agents (Austin Johnson's threshold for revisiting sequential handoff in ADR-001).

---

### A8 — Per-agent voice files

**Assumption.** Diana's voice is captured per-agent (one file per team member) at `voice/<agent>.md`, git-ignored. Per Austin Johnson's pattern.

**Evidence.** Strong (pattern adoption). Drives `03_client_communication` drafts — without `agent_on_deal`, no voice file lookup, no draft (CALL-015 Rule 0).

**Revisit trigger.** A specialist needs voice context from someone other than the assigned `agent_on_deal` (e.g. team-shared boilerplate).

---

### A9 — Agent reviews every draft before send

**Assumption.** A human agent reviews every drafted message before it is sent to the client. The system never sends autonomously.

**Evidence.** Strong (UPL constraint). NAR Article 13 of the REALTOR® Code of Ethics requires recommending counsel, not providing legal interpretation. Without human review the system would commit UPL by drafting content a client might act on as legal advice (`decisions/domain-research.md` §2.4).

**Revisit trigger.** None acceptable. Non-negotiable; documented as a hard quality standard in `03_client_communication/identity.md`.

---

### A10 — No custom software

**Assumption.** The team uses calendar reminders or a narrow CRM (Follow Up Boss, dotloop, similar). The system's output is markdown / text / CSV — never API calls into a custom platform.

**Evidence.** Medium. Diana's brief explicitly states "doesn't want software, wants a system she can teach in a week." Confirmed by `decisions/domain-research.md` §1 — boutique teams resist platform consolidation.

**Revisit trigger.** Diana adopts a CRM with an API that the team wants the system to integrate with.

---

### A12 — Adult, non-vulnerable clients

**Assumption.** Diana's team works exclusively with adult, non-vulnerable clients.

**Evidence.** Weak. Edge cases — probate sales, divorce, foreclosure, vulnerable-elder — need different handling. Austin Johnson's `case_type` enum (ADR-001) covers some (`estate`, `divorce`, `foreclosure`) but per-specialist behaviour on each is currently default.

**Revisit trigger.** A specialist's first encounter with a non-default `case_type` envelope. At that point that specialist's `examples.md` needs a worked case for that type.

---

### A13 — English only

**Assumption.** The system drafts in English only.

**Evidence.** Strong (scope assumption). Austin has a substantial Spanish-speaking buyer population, but multilingual draft generation is a re-locale concern, not v1.

**Revisit trigger.** Diana adds Spanish-language clients to the team's regular caseload. At that point voice files need per-language variants.

---

## Per-specialist assumptions

### A11 — Lead-source mix (`00_orchestrator`) **[OPEN]**

**Assumption.** Repeat-business + referral is the dominant lead source (>50% of inbound), not cold ads or walk-ins.

**Evidence.** Medium. Industry pattern for 4-person 60-80-transaction/year boutique teams (`decisions/domain-research.md` §2.1).

**Defensible default if unresolved.** Frame `00_orchestrator/identity.md` around routing returning clients as the primary task; cold inbound as secondary.

**Revisit trigger.** Diana confirms otherwise, or quarterly inbound audit shows the mix has shifted.

**Open question:** OQ-5 in `decisions/open-questions.md`.

---

### A5 — Investor share (`01_lead_qualifier`) **[OPEN]**

**Assumption.** ~5-10% of Diana's transactions are investor / cash-buyer; ~90% are financed owner-occupants.

**Evidence.** Weak. Austin 2026 market data shows financed-majority but doesn't break out by team type (`decisions/domain-research.md` §2.2 + §5).

**Defensible default if unresolved.** Treat investor as an edge case in `01_lead_qualifier/examples.md`. Investor inquiries that don't fit the qualifier's field set trigger `back_scope_mismatch` (ADR-001 `handoff_reason`) and route to orchestrator.

**Revisit trigger.** Diana confirms investor share is >10%, or `01_lead_qualifier` accumulates more than 3 `back_scope_mismatch` events on investor inquiries in any quarter.

**Open question:** OQ-4 in `decisions/open-questions.md`.

---

### A15 — Team Assignment Table (`01_lead_qualifier`) **[OPEN]**

**Assumption.** The `01_lead_qualifier` assignment table maps qualified leads to a named agent (`agent_on_deal`) based on `qualified_lead.sub_type` + top `geography`. At submission, the table is intentionally minimal: every forward emission defaults to `agent_on_deal: team_lead` (CALL-022 sentinel). Diana fills the named-agent rules with the 3 other team members on intake.

**Evidence.** Weak. Brief doesn't name team members beyond Diana; the assignment rule structure (sub_type + geography → agent) is industry-standard for boutique teams, but the specific rules belong to Diana.

**Defensible default if unresolved.** Every forward emission uses `agent_on_deal: team_lead` (resolves to `voice/team_lead.md`, Diana's team-default voice file per A8). Diana's team-default voice carries all pre-conversion communications and any case where a named-agent rule doesn't match. Per CALL-022.

**Revisit trigger.** Diana fills the table with at least 3 named-agent rules during onboarding (typical: one specialty per non-Diana agent), OR `team_lead` sentinel fires on >80% of forward emissions in any 30-day window post-launch (indicates the table is under-populated and the team is sustaining team-voice on too many client-facing touches).

**Sample rule shape (filled with Diana on intake):**
```yaml
- { sub_type: downsize,   geography: [78703, 78704, 78746], agent: agent_diana }
- { sub_type: first_time, geography: [78747, 78748],        agent: agent_<name> }
- { sub_type: investor,   geography: any,                   agent: agent_<name> }
```

Until filled, the system ships with the sentinel default and no named-agent rules.

---

### A2 — MLS access (`02_property_research`)

**Assumption.** Diana's team has UnlockMLS (Austin metro MLS) access for comp research; MLS is the authoritative source for closed comps.

**Evidence.** Medium. Industry-standard for 60-80-transaction/year teams (`decisions/domain-research.md` §2.3).

**Revisit trigger.** Verify with Diana before going live. If she uses a paid alternative (CoreLogic, MLS-integrated CMA tool), source-naming convention in `02_property_research/handoff.md` shifts.

---

### A4 — Travis County primary, Hays / Williamson spillover (`02_property_research`, `04_transaction_coordinator`)

**Assumption.** Diana's team predominantly handles Travis County (Austin metro), with some Hays County and Williamson County spillover.

**Evidence.** Medium. Austin metro standard. CAD lookups parameterised by county (austincad.org, hayscad.com, wcad.org).

**Revisit trigger.** Diana expands beyond the Austin metro area, or a deal crosses into a fourth county.

---

### A7 — Communication channel defaults (`03_client_communication`)

**Assumption.** Communication preference defaults to text for buyer leads under ~40, email for sellers over ~50.

**Evidence.** Weak. Folk-wisdom from communication guides; specific cutoffs are guessed (`decisions/domain-research.md` §2.4 + §5).

**Defensible default.** Capture per-client preferred channel upstream (in `01_lead_qualifier` payload); never hard-code in `03_client_communication`. Default to last-used channel if preference not captured.

**Revisit trigger.** Drafting fails to match client preference more than 2 times in a single case_id.

---

### A3 — Default title company (`04_transaction_coordinator`) **[OPEN]**

**Assumption.** Diana's team has a primary title-company relationship for most deals.

**Evidence.** Medium. Standard for boutique teams in Austin (`decisions/domain-research.md` §2.5 + §5).

**Defensible default if unresolved.** Parameterise the title-company field in `04_transaction_coordinator/handoff.md`; require it to be populated per case_id at deal-execution time.

**Revisit trigger.** Diana confirms a specific default, or the team starts routing >2 different title companies per quarter without a clear pattern.

**Open question:** OQ-3 in `decisions/open-questions.md`.

---

## Envelope-level assumption

### A14 — Intermediary status default (ADR-001 envelope) **[OPEN]**

**Assumption.** Diana's team does not currently use intermediary status — each agent represents one side per deal. Envelope `intermediary_status` defaults to `false`.

**Evidence.** Weak. Boutique-team practice varies. TX intermediary rules require written consent + conspicuous bold/underlined prohibited-conduct disclosure (`decisions/domain-research.md` §3, TRELA).

**Defensible default if unresolved.** Default `intermediary_status: false`. Flipping to `true` requires Diana's explicit consent flow per TRELA. Documented in `01_lead_qualifier` and `04_transaction_coordinator` handoff payloads.

**Revisit trigger.** Diana practices intermediary representation on at least one deal in the next year, or TRELA changes the disclosure requirements.

**Open question:** OQ-6 in `decisions/open-questions.md`.

---

## Summary

15 assumptions documented. **5 are Open** (A3, A5, A11, A14 tracked in `decisions/open-questions.md`; A15 tracked in-place at this file). Defensible defaults are named for each open assumption; the system ships safely without Diana's input but should be confirmed before going live.

- **Strong evidence (4):** A1, A8, A9, A13. Non-negotiable.
- **Medium evidence (5):** A2, A4, A6, A10, A11. Industry-standard, low risk.
- **Weak evidence (6):** A3, A5, A7, A12, A14, A15. Highest-revisit priority post-launch.
