# Failure Modes Register

Compiled from each specialist's `identity.md` §6 failure-mode register. Cross-document view of every distinct failure mode the system exists to prevent — organised by specialist (matching the §6 source), then re-grouped by taxonomy at the bottom.

**Why this is its own file.** Each `identity.md` §6 lives next to that specialist's responsibilities (per ADR-003 + CALL-017 — co-location with responsibilities prevents the canonical failure-mode rot pattern). This file is the *cross-cutting* view — useful for criterion #2 review ("handoff protocols actually defined or hand-waved") and for system-level auditing. The per-specialist §6 entries remain canonical; this file mirrors them. Any divergence is a bug — discovered via cross-document review during periodic maintenance audits.

---

## By specialist

### 00_orchestrator — routing failures

1. **Routes anonymous inbound straight to a downstream specialist without setting `content_provenance: anonymous_inbound`** → downstream treats unverified identity as verified; may draft personalised content for a spoofed/fake sender. See `00_orchestrator/examples.md` Pair 1.
2. **Generates a fresh `case_id` for a returning client** → orphans the existing case, splits the deal trail across two records, breaks the CAS chain (`parent_envelope_id` no longer reaches the original). See `00_orchestrator/examples.md` Pair 2.
3. **Routes inbound naming a property the team has no listing or buyer-rep on** → 02 produces neutral research, 03 then drafts client-facing content implying representation that doesn't exist. TRELA / IABS exposure.
4. **Routes a TC-class message ("title called, closing pushed to Friday") with no resolved `case_id`** → 04 has no deal state to update; either silently noops or back-handoffs with no diagnostic. See `00_orchestrator/examples.md` Pair 3.
5. **Misclassifies subject (e.g. routes a CMA request to `01` instead of `02`)** → wrong specialist's Rule 0 fires, the back-handoff cycle wastes a hop and surfaces to the team as a system error rather than a routing error. See `00_orchestrator/examples.md` Pair 4 + `01_lead_qualifier/examples.md` Pair 3.

### 01_lead_qualifier — qualification failures

1. **Forwards a lead with no `raw_inbound` text captured** → downstream specialists have no provenance trail; system cannot demonstrate where the qualified-lead data came from if challenged. (Rule 0 violation — `01_lead_qualifier/rules.md` §1.)
2. **Captures a stated budget without pre-approval and forwards as `confidence: high`** → downstream property research treats budget as actionable; team wastes hours on showings the lead can't close. See `01_lead_qualifier/examples.md` Pair 1.
3. **Misses the existing-representation question and forwards a buyer already working with another agent** → REALTOR® Code Article 16 violation, IABS exception 2 ignored, broker exposure. (Canonical `back_compliance_block` site — see `01_lead_qualifier/examples.md` Pair 2.)
4. **Forwards a lead already in negotiation ("I have an offer in, can you help me think about it")** → wrong specialist; should hard-refuse + back-handoff to `00` for human escalation. Re-qualifying someone under contract is malpractice-adjacent.
5. **Treats a CMA / valuation request as a qualification ("what's my house worth")** → produces a fake qualified-lead payload from a research-class inbound. Back-handoff to `00` for re-routing to `02`. See `01_lead_qualifier/examples.md` Pair 3.
6. **Forwards to `02_property_research` without populating `payload.research_request.subject`** → 02's Rule 0 fires `back_data_missing` on every showing-prep route. `subject` derived from `inbound_property_ref` (carried from `00`) or composed from top `geography` + `property_type` with a band suffix; `purpose` selected from {`cma_only`, `showing_prep`, `neighborhood_brief`, `valuation_comparison`}. See `01_lead_qualifier/handoff.md` §2 payload shape.
7. **Forwards with `agent_on_deal: null` on a route to 02 / 03 / 04** → 03's Rule 0 fires `back_data_missing` on every emission that reaches it (directly or via carry-through). `agent_on_deal` must be set per the assignment table at `decisions/assumptions.md` § A15 (named agent if rule matches) or to the `team_lead` sentinel as default. Per CALL-022.

### 02_property_research — research failures

1. **Quotes a comp without naming the source MLS ID** → agent passes the figure to a client; client asks for the source; team has none; trust damage. See `02_property_research/examples.md` Pair 1.
2. **Reports neighborhood without resolving MUD / PID** → buyer makes offer based on quoted $ taxes; closes; discovers $3,200/year MUD assessment they weren't told about. Buyer's right to terminate may have been triggered if MUD was on the SDN and not disclosed. See `02_property_research/examples.md` Pair 2.
3. **Produces a single-point price estimate ("worth $812,000")** instead of a price band → agent uses the number in client comms; market moves; band would have absorbed the move; single point now reads as wrong-prediction. See `02_property_research/examples.md` Pair 3.
4. **Accepts an over-broad subject ("research Austin market") and produces a brief anyway** → output is generic, not actionable; agent can't use any of it; team time wasted. (Rule 0 violation — `02_property_research/rules.md` §1.)
5. **Opines on whether a listing is fairly priced ("comps suggest this is overpriced by 5%")** → strays into agent interpretation; if agent disagrees, conflict; if agent accepts and shares, may inadvertently signal valuation opinion to client (agent territory, not specialist territory). See `02_property_research/examples.md` Pair 4.

### 03_client_communication — drafting failures

1. **Drafts when `agent_on_deal` is null** → produces voice-anonymous output that misrepresents the team member supposedly speaking. (Rule 0 violation — `03_client_communication/rules.md` §1.) Note: `agent_on_deal: team_lead` (CALL-022 sentinel) is NOT a Rule 0 violation — sentinel resolves to `voice/team_lead.md`, a real voice file, and Rule 0 is satisfied.
2. **Drafts a personalised message ("Hi Sarah") for an anonymous inbound** with `content_provenance: anonymous_inbound` → spoofed/fake-sender risk; the system speaks as if it knows the client when it doesn't. See `03_client_communication/examples.md` Pair 2.
3. **Interprets a contract clause for a client** ("looks like the option period expires Friday so you have until then to terminate") → UPL violation; one of the canonical malpractice exposures for non-attorneys in real estate. (Primary `back_compliance_block` site — see `03_client_communication/examples.md` Pair 1.)
4. **Sends autonomously** (any path that emits client-facing content without `agent_review_required: true`) → defeats the entire UPL defence. The agent-review gate IS the system's legal cover.
5. **Drafts using a missing voice file by inferring tone from prior messages** → produces baseline-voice content that *looks* like the agent's voice without the flag. Future divergence is invisible until a client notices. (Mitigated by CALL-020 degrade-with-flag — see `03_client_communication/examples.md` Pair 3.)

### 04_transaction_coordinator — deal-active failures

1. **Conflates Option Fee with Earnest Money** → catastrophic. Different fees, different delivery targets (both title co, but tracked separately), different refundability rules (Option Fee non-refundable always; Earnest Money refundable on legitimate termination). Confusing them produces wrong client guidance and risks deal collapse. See `04_transaction_coordinator/examples.md` Pair 1.
2. **Day-count using business days instead of calendar days** → option period misses by 2-3 days routinely; client may believe they have until Monday when termination right expired Saturday at 5pm local. See `04_transaction_coordinator/examples.md` Pair 2.
3. **Financing contingency tracked as single deadline** → property-approval (fixed at `closing_date - 3`) gets ignored; appraisal contingency lapses unnoticed. See `04_transaction_coordinator/examples.md` Pair 3.
4. **HOA resale certificate not requested at deal-active initialisation** → §207 Property Code 10-day delivery missed; buyer's right to terminate triggered by late docs. See `04_transaction_coordinator/examples.md` Pair 4.
5. **Seller's Disclosure Notice §5.008 not verified on file at execution** for non-exempt residential resale → exemption-class unexamined; later discovery may give buyer rescission grounds. See `04_transaction_coordinator/examples.md` Pair 5.
6. **Initialises deadline tracking on a deal where buyer-rep agreement is not signed pre-showing** → TRELA §1101.563 (post-2026-01-01) violation by upstream specialists not caught at the TC layer; broker exposure. (Canonical `back_compliance_block` site — see `04_transaction_coordinator/examples.md` Pair 6 + CALL-015 resolution.)
7. **Advises on terminating when client asks "should I exercise option period?"** → UPL. Surface options to agent; agent advises. See `04_transaction_coordinator/examples.md` Pair 7.
8. **Treats Hays / Williamson County deals with Travis County resources** (TraviCAD, AISD, etc.) → wrong tax roll, wrong recording rules, wrong ISD attendance. County-specific resource swap is a Soft rule. (Mitigated by `04_transaction_coordinator/rules.md` §3 county-specific resource-swap rule.)
9. **Misses CFPB 3-business-day closing disclosure delivery** → closing delayed; lender re-issues CD; closing pushed; client confidence impact + cascading deadlines.
10. **Advises client to file homestead exemption** (post-close) → administrative overstep. Flag the deadline (Travis County: any time after purchase if owner-occupied as of Jan 1) for the agent to mention; the agent recommends; the client files. See `04_transaction_coordinator/examples.md` Pair 8.

---

## By taxonomy

A second view of the same failures, grouped by *what kind* of failure each is. Useful for system-level review (criterion #2 + #4) and for stranger-onboarding (criterion #3 — "what kinds of mistakes can this system make"). Numbers below cross-reference the per-specialist register above.

### Data-integrity failures (contract layer)

Producing a well-formed envelope without the data needed for downstream specialists to act safely.

- **Missing source provenance.** Quoting facts without traceable sources: 02-1 (MLS ID not named), 02-4 (over-broad subject).
- **Missing required fields.** Forwarding without Rule 0 satisfaction: 01-1 (no raw_inbound), 01-6 (no research_request on 02-bound route → 02's Rule 0), 01-7 (no agent_on_deal on forward → 03's Rule 0), 03-1 (drafted with no agent_on_deal — symptom of 01-7), 04-Rule-0 (no executed contract).
- **Wrong field values.** Identity/state misrepresentation: 00-1 (wrong `content_provenance`), 00-2 (wrong `case_id` strategy), 04-1 (Option Fee/Earnest Money conflation), 04-3 (financing single-deadline).

### Compliance failures (regulatory layer)

Crossing or failing to detect a regulatory boundary.

- **UPL — Unauthorised Practice of Law.** Drafting or advising on legal interpretation, contract enforceability, termination rights. NAR Article 13. Primary site: `03_client_communication`. Failures: 03-3 (interpret contract clause), 04-7 (advise on terminating), 02-cross-ref-UPL (legal interpretation of HOA covenants).
- **TRELA §1101.563** (effective 2026-01-01). Buyer-rep agreement required pre-showing. Failures: 04-6 (deadline tracking without buyer-rep), 01-cross-ref-TRELA (qualification without surfacing buyer-rep status).
- **REALTOR® Code Article 16 / IABS exception 2.** No interference with existing representation. Failures: 01-3 (forwards lead already working with another agent), 00-cross-ref (routes inbound to a represented counterparty).
- **TCPA — Telephone Consumer Protection Act.** Outbound texts outside 8am-9pm local. Mitigated by `03_client_communication/rules.md` §3 soft-rule queue-with-flag pattern. Failures: 03-cross-ref-TCPA (release outside-window draft).
- **§5.008 Property Code — Seller's Disclosure Notice.** Required for non-exempt residential resale. Failures: 04-5 (missed exemption check).
- **§207 Property Code — HOA resale certificate.** 10-day delivery requirement. Failures: 04-4 (late request triggers buyer terminate-right).
- **CFPB 3-business-day rule.** Closing Disclosure delivery before close. Failures: 04-9 (missed re-disclosure cascade).

### Content failures (output quality)

Producing output that is technically well-formed but operationally wrong.

- **Voice-anonymous content.** 03-1 (no agent_on_deal — note CALL-022 sentinel `team_lead` is NOT voice-anonymous; it resolves to a real team-default voice file), 03-5 (missing voice file inferred from prior messages instead of degraded-with-flag).
- **Single-point predictions where bands are required.** 02-3 (single-point CMA estimate).
- **Opinions in research output.** 02-5 (valuation opinion), 02-cross-ref-fairness ("is this overpriced").
- **Personalisation without identity verification.** 03-2 (personalised salutation on anonymous inbound), 00-1 (downstream personalisation enabled by wrong `content_provenance`).
- **Autonomous send.** 03-4 (any path that bypasses `agent_review_required: true`).

### Topology failures (routing)

Routing work to the wrong specialist or no specialist.

- **Misclassification.** 00-5 (CMA request → 01 instead of 02), 01-5 (CMA request handled as qualification at 01 layer).
- **Ambiguous routing without escalation.** 00-3 (named property with no representation), 00-4 (TC inbound with no resolved case_id).
- **Skipped specialist.** 01-cross-ref (jumps to 02 without identity verification step that closes anonymous-inbound quarantine).
- **Wrong scope.** 01-4 (qualifies someone already under contract).

### Time-arithmetic failures (deal-active phase)

Specific to `04_transaction_coordinator` deadline reasoning.

- **Day-count.** 04-2 (business vs calendar days, option period in particular).
- **Dual-deadline tracking.** 04-3 (financing as single deadline).
- **Late upstream artifact requests.** 04-4 (HOA resale certificate timing).
- **Post-close advisory overreach.** 04-10 (TC files homestead instead of flagging for agent).

---

## What this register is not

This is not an exhaustive list of every possible bug. It's the catalogue of *failures the system was designed against* — each entry maps to a specific Rule 0, Hard rule, Soft rule, or refusal trigger that exists *because* this failure mode was anticipated. Adding a new failure mode means either (a) extending an existing rule to cover it, or (b) introducing a new rule and updating this file's per-specialist + by-taxonomy sections.

**Update discipline.** When a `<specialist>/identity.md` §6 entry is added/edited/removed, this file's corresponding §By-specialist entry is updated in the same change. The §By-taxonomy view is regenerated from the §By-specialist view; never edited independently. Otherwise this file accumulates drift from the canonical per-specialist §6 sources — the exact rot pattern CALL-017 was designed to prevent at the specialist layer.

---

## Cross-references

- Per-specialist canonical source: each `<specialist>/identity.md` §6.
- Refusal mapping: each `<specialist>/handoff.md` §4 (refusal triggers → `handoff_reason` enum values per CALL-019).
- Closed refusal taxonomy (system-level claim): `decisions/architecture-properties.md` §P1.
- Adversarial review findings on failure coverage: `decisions/council-verdict.md` (Principle-Purist P8 + Skeptic findings).
- Domain anchors for compliance failures: `decisions/domain-research.md` §3.
