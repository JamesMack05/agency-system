# Trajectories + Pass Criteria

**Purpose:** verify the architecture by hand-walking concrete end-to-end trajectories through ADR-001 / ADR-003 / ADR-002 before specialist drafting commits. Three trajectories locked with explicit pass criteria; T1 hand-walked end-to-end.

**Good-enough floor:** *2+ real trajectories pass on primary use cases without catastrophic failure.* Three trajectories below cover happy-path, back-handoff (data-missing), and forward-urgent (deadline pressure). T1 hand-walked here; T2/T3 envelope-sequenced and pass-criteria-checked.

---

## Pass criteria (shared across all trajectories)

A trajectory passes if **every envelope at every hop** satisfies all three layers:

### Layer A — ADR-001 envelope well-formed
1. **Rule 0 holds:** `schema_version` present, `case_id` present. If `content_provenance: anonymous_inbound`, envelope is quarantined for sender-identity verification before processing. (Per-specialist preconditions — e.g. `agent_on_deal` required for 03 — are checked at Layer B, not Layer A. Envelope = processable; per-specialist = performable.)
2. All 16 core fields populated or explicitly null with a documented reason. `content_provenance` absence is itself a Layer-B (per-specialist) violation, not a Layer-A (Rule 0) one.
3. `handoff_reason` is one of the 6 enum values (`forward_normal` / `forward_urgent` / `back_data_missing` / `back_scope_mismatch` / `back_quality_failure` / `back_compliance_block`).
4. `parent_envelope_id` references the previous envelope in the trajectory (CAS chain unbroken).
5. `trail` is append-only — every prior hop appears.

### Layer B — ADR-003 refusal triggers fire correctly
1. The receiving specialist's Rule 0 (per locked wording in ADR-003) is satisfied — or, if not, the specialist back-handoffs with the correct `handoff_reason`.
2. Refusal triggers in the receiving specialist's `handoff.md` §4 map to the produced `handoff_reason` value verbatim.
3. Confidence calibration in `handoff.md` §5 matches the `confidence` field set by the producing specialist.
4. Back-handoff sources in `handoff.md` §6 of the destination specialist include the producing specialist as a valid `back_to` source.

### Layer C — ADR-002 analogy doesn't leak
1. Envelope `payload` field names use real-model vocabulary (no `binder_slot`, `sticky_tab`, `chain_of_custody_log`).
2. `next_action` free-text uses real-model vocabulary.
3. Handoff occurs entirely in the real-model zone (envelope is never "translated" via analogy mid-trajectory).

---

## T1 — Happy-path forward (00 → 01 → 02 → 03 → 04 → END)

**Scenario.** Sarah Chen, buyer-side, anonymous inbound via Zillow on a listing in Tarrytown (78703). Pre-approved $850k, 60-day timeline, downsizer. Through qualification, neighborhood + comp research, agent-voice intro email, signed buyer-rep + executed contract, 35-day TC phase to funding.

### Envelope sequence

```
HOP 1 — Inbound capture (no envelope yet; raw_inbound captured by 00)
HOP 2 — 00 → 01    (forward_normal)
HOP 3 — 01 → 02    (forward_normal)
HOP 4 — 02 → 03    (forward_normal)
HOP 5 — 03 → 04    (forward_normal — after off-system signed buyer-rep + executed contract)
HOP 6 — 04 → END   (forward_normal — funded, deal closed)
```

### Hand-walk

**HOP 2: 00 → 01 (forward_normal)**

All envelopes below use `schema_version: 1.1.0` — the version with the 6-value `handoff_reason` enum.

```yaml
schema_version: 1.1.0
case_id: CASE-2026-0042
from: 00_orchestrator
to: 01_lead_qualifier
back_to: null
timestamp: 2026-05-12T14:32:08Z
agent_on_deal: null              # 00 emits null because 01 is the producing site (per CALL-022); 01 will set this on case_type resolution during qualification
payload:
  inbound_channel: web_form
  inbound_text: "Hi, saw your Tarrytown listing on Zillow, can someone reach out?"
  sender_handle: sarah.chen.zillow.relay@zillow.example
  subject_classification: qualification
  inbound_property_ref: MLS-9182734
  urgency: normal
required_fields_present: true
confidence: med                  # anonymous inbound, identity unverified
next_action: "Qualify lead — capture intent, timeline, budget+pre-approval, geography, current rep status, buyer-rep signed status."
trail:
  - {at: 2026-05-12T14:32:08Z, by: 00_orchestrator, event: "routed forward_normal to 01_lead_qualifier"}
parent_envelope_id: null         # first envelope in case
content_provenance: anonymous_inbound
case_type: residential_buyer_side
linked_case_ids: []
handoff_reason: forward_normal
intermediary_status: null         # not yet relevant; both sides not represented by Diana's brokerage
```

- **Layer A.** Rule 0: `schema_version` ✓, `case_id` ✓. `content_provenance: anonymous_inbound` is a quarantine trigger — but the trajectory passes through 00 BEFORE 01 receives, so 00 is the layer that holds the envelope while sender-identity verification proceeds (in this case: 01's qualification call doubles as identity verification). `handoff_reason: forward_normal` ✓ (one of 6). `parent_envelope_id: null` correct for first envelope ✓. `trail` has the 00 hop ✓.
- **Layer B.** 01's Rule 0 ("No `raw_inbound` content, no qualification") — `payload.inbound_text` populated ✓. 01's `handoff.md` §6 lists 00 as valid back-handoff source — N/A here, not a back. Confidence `med` matches anonymous-inbound calibration ✓. (Note: 01 receives a quarantined envelope but its qualification task IS the identity-verification step, so the quarantine resolves during 01's ACT step rather than blocking THINK.)
- **Layer C.** Field names are real-model (`inbound_channel`, `subject_classification`). `next_action` is real-model ("qualify lead", "capture intent"). No analogy leak ✓.

**HOP 3: 01 → 02 (forward_normal)**

```yaml
schema_version: 1.1.0
case_id: CASE-2026-0042
from: 01_lead_qualifier
to: 02_property_research
back_to: null
timestamp: 2026-05-12T14:48:21Z
agent_on_deal: "agent_diana"     # set by 01 per CALL-022 — assignment table A15 sample rule matched (sub_type: downsize + geography: 78703 Tarrytown → agent_diana)
payload:
  qualified_lead:
    intent: buy
    sub_type: downsize
    timeline: 30-90_days
    budget: {range: "750-850k", preapproval_status: pre_approved, preapproval_lender: "Frost Bank"}
    geography: ["78703 Tarrytown", "78704 Bouldin"]
    property_type: SFH
    current_representation_status: none
    buyer_rep_agreement_status: not_yet_signed
    iabs_delivered_flag: true
  research_request:
    subject: "MLS-9182734 (Tarrytown listing) — comp set + neighborhood profile"
    purpose: showing_prep
required_fields_present: true
confidence: high                 # all 3-question filter answered + pre-approval verified
next_action: "Pull 3-5 closed comps within 0.5mi past 90 days; pull neighborhood profile (ISD, MUD/PID, HOA, flood)."
trail:
  - {at: 2026-05-12T14:32:08Z, by: 00_orchestrator, event: "routed forward_normal to 01_lead_qualifier"}
  - {at: 2026-05-12T14:48:21Z, by: 01_lead_qualifier, event: "qualified high; routing to 02_property_research"}
parent_envelope_id: ENV-2026-0042-001
content_provenance: verified_client     # 01 verified Sarah's identity during qualification call
case_type: residential_buyer_side
linked_case_ids: []
handoff_reason: forward_normal
intermediary_status: no
```

- **Layer A.** All checks pass. Note: `content_provenance` flipped `anonymous_inbound → verified_client` after 01's qualification call confirmed identity. `intermediary_status: no` set explicitly (Diana's brokerage doesn't list MLS-9182734).
- **Layer B.** 02's Rule 0 ("No specific subject, no brief") — `payload.research_request.subject` names a specific MLS ID ✓. 02's `handoff.md` §1 lists `qualified_lead` as required input ✓. Confidence `high` calibrated to "all 3-question filter answered + pre-approval verified" ✓.
- **Layer C.** Real-model throughout. No "binder" or "sticky-tab" terminology. ✓.

**HOP 4: 02 → 03 (forward_normal)**

```yaml
schema_version: 1.1.0
case_id: CASE-2026-0042
from: 02_property_research
to: 03_client_communication
back_to: null
timestamp: 2026-05-12T16:12:55Z
agent_on_deal: "agent_diana"     # carried through from 01 (set per CALL-022 at HOP 3 — assignment table A15 match: downsize + 78703 → agent_diana)
payload:
  brief_subject: "MLS-9182734, 2412 Hartford Rd, 78703"
  agent_pull_quotes:
    - "Recent comps in Tarrytown 0.5mi: $792k (3/2/1842sqft, 47 DOM), $821k (3/2.5/1980sqft, 12 DOM), $864k (4/2/2103sqft, 31 DOM)."
    - "Eanes ISD elementary: Casis (8/10), middle: O. Henry, high: Austin High."
    - "MUD: none. HOA: none. Flood: zone X (minimal)."
  sources_named:
    - "UnlockMLS listings 9182734, 9165501, 9171002, 9180445"
    - "austincad.org parcel 0118040502"
    - "eanes.k12.tx.us elementary boundary lookup"
  comp_count: 4
  comp_window_days: 90
  draft_request:
    purpose: first_touch_follow_up
    channel_preference: email      # captured by 01 during qual call: "email me at sarah.chen.work@example.com"
required_fields_present: true
confidence: high                  # 4 comps, all <90d, MUD/PID resolved
next_action: "Draft first-touch email in agent_diana voice; flag agent_review_required: true."
trail:
  - {at: 2026-05-12T14:32:08Z, by: 00_orchestrator, event: "..."}
  - {at: 2026-05-12T14:48:21Z, by: 01_lead_qualifier, event: "..."}
  - {at: 2026-05-12T16:12:55Z, by: 02_property_research, event: "research complete; routing to 03_client_communication"}
parent_envelope_id: ENV-2026-0042-002
content_provenance: verified_client
case_type: residential_buyer_side
linked_case_ids: []
handoff_reason: forward_normal
intermediary_status: no
```

- **Layer A.** Rule 0: `schema_version` ✓, `case_id` ✓, `content_provenance: verified_client` (no quarantine trigger) ✓. All 16 fields shape-valid. `handoff_reason: forward_normal` ✓. (Note: the `agent_on_deal` precondition for 03 is checked at Layer B — per-specialist Rule 0 — not Layer A. Envelope is *processable* regardless of `agent_on_deal`; 03 then checks *performable* in its Rule 0.)
- **Layer B.** 03's per-specialist Rule 0 ("No `agent_on_deal`, no draft") — `agent_on_deal: "agent_diana"` populated ✓. This is the layer where the `agent_on_deal` precondition is enforced. 03's `handoff.md` §1 requires `voice_file_used`-eligible payload — `agent_on_deal` resolves to `voice/diana.md` per ADR-001 attribution + `decisions/domain-research.md` §2.4 / A8 ✓. Confidence `high` matches "all sources named + MUD/PID resolved" ✓.
- **Layer C.** `agent_pull_quotes` is real-model (working-realtor language). No analogy leak. `next_action` says "draft first-touch email", not "fill the binder slot" ✓.

**HOP 5: 03 → 04 (forward_normal)**

*Off-system gap:* 03 produced a draft email, agent_diana reviewed and sent, Sarah replied wanting to tour, agents toured, Sarah signed buyer-rep (TRELA §1101.563 satisfied), Diana wrote an offer using TREC 20-15, offer accepted, contract executed 2026-05-19. None of this happens inside the system — but the envelope picks up at 04 with the executed-contract event.

```yaml
schema_version: 1.1.0
case_id: CASE-2026-0042
from: 03_client_communication
to: 04_transaction_coordinator
back_to: null
timestamp: 2026-05-19T17:45:00Z
agent_on_deal: "agent_diana"
payload:
  deal_state_change: contract_executed
  executed_contract_date: 2026-05-19
  contract_form: TREC_20-15
  closing_date: 2026-06-23
  option_fee: {amount: 350, delivered_to: "Heritage Title Co", delivered_date: 2026-05-19, period_end: 2026-05-26}
  earnest_money: {amount: 8500, delivered_to: "Heritage Title Co", delivered_date: 2026-05-21, escrow_agent: "Heritage Title Co"}
  financing: {type: conventional, lender: "Frost Bank", buyer_approval_deadline: 2026-06-09, property_approval_deadline: 2026-06-20}
  inspection_window_end: 2026-05-26
  buyer_rep_agreement: {signed_date: 2026-05-15, signed_pre_showing: true}
  iabs_delivered_flag: true
  intermediary_status_confirmed: no
required_fields_present: true
confidence: high
next_action: "Initialise deal-active phase. Track all deadlines. Daily digest to agent_diana. Voice-escalation on T-24h to any contingency."
trail:
  - {at: 2026-05-12T14:32:08Z, by: 00_orchestrator, event: "..."}
  - {at: 2026-05-12T14:48:21Z, by: 01_lead_qualifier, event: "..."}
  - {at: 2026-05-12T16:12:55Z, by: 02_property_research, event: "..."}
  - {at: 2026-05-19T17:45:00Z, by: 03_client_communication, event: "off-system phase complete; contract executed; routing to 04_transaction_coordinator"}
parent_envelope_id: ENV-2026-0042-003
content_provenance: agent_authored      # the executed-contract event was logged by agent_diana, not auto-generated
case_type: residential_buyer_side
linked_case_ids: []
handoff_reason: forward_normal
intermediary_status: no
```

- **Layer A.** Rule 0 ✓ (schema, case_id valid; `content_provenance: agent_authored` is not a quarantine trigger). 16 fields shape-valid. `intermediary_status: no` confirmed.
- **Layer B.** 04's per-specialist Rule 0 ("No signed contract, no deadline tracking") — `payload.executed_contract_date` populated ✓. `payload.buyer_rep_agreement.signed_pre_showing: true` satisfies TRELA §1101.563 (`decisions/domain-research.md` §3) ✓. Option fee + earnest money tracked separately (not conflated — `decisions/domain-research.md` §2.5 catastrophic-failure mode avoided) ✓. Financing has separate `buyer_approval_deadline` and `property_approval_deadline` ✓. Confidence `high` matches "all required fields populated by agent" ✓.
- **Layer C.** Field names real-model (`executed_contract_date`, `option_fee`, `earnest_money` — no "binder closed" or "sticky-tab placed" language) ✓.

**HOP 6: 04 → END (forward_normal)**

```yaml
schema_version: 1.1.0
case_id: CASE-2026-0042
from: 04_transaction_coordinator
to: END
back_to: null
timestamp: 2026-06-23T15:30:00Z
agent_on_deal: "agent_diana"
payload:
  deal_state_change: funded
  funding_date: 2026-06-23
  closing_disclosure_delivered_date: 2026-06-20
  homestead_exemption_advisory_sent: true   # post-close advisory per decisions/domain-research.md §2.5
required_fields_present: true
confidence: high
next_action: null    # terminal
trail: [...]         # full chain
parent_envelope_id: ENV-2026-0042-004
content_provenance: system_generated
case_type: residential_buyer_side
linked_case_ids: []
handoff_reason: forward_normal
intermediary_status: no
```

- **Layer A/B/C.** Terminal envelope. Rule 0 satisfied; no receiver, so no per-specialist Rule 0 to check. Closing-disclosure delivery captured (CFPB 3-business-day rule per `decisions/domain-research.md` §2.5). Homestead exemption flagged (TraviCAD post-close advisory, A10 in assumption ledger). ✓.

### T1 verdict

**PASS.** All 5 envelope-producing hops well-formed under ADR-001, refusal triggers correctly N/A (no refusals on happy-path), no analogy leakage. The layering of envelope-level vs per-specialist Rule 0 holds: envelope-Rule-0 no longer references the `03_client_communication` folder; per-specialist Rule 0 in ADR-003 owns the `agent_on_deal` precondition. HOPs 4, 5, 6 commentary above reflects the locked layering.

---

## T2 — Back-handoff (03 → 01, back_data_missing)

**Scenario.** Mike Reyes, buyer-side, text inquiry. 01 qualifies on a partial call (timeline + budget yes, channel preference *not* captured before lead got distracted and ended call). 01 routes directly to 03 to send a "thanks for the call, here's a summary + next steps" message — but 03 needs `payload.preferred_channel` to know whether to draft a text or email. Back to 01 with `back_data_missing`.

### Envelope sequence

```
HOP 1 — Inbound capture (text)
HOP 2 — 00 → 01    (forward_normal)
HOP 3 — 01 → 03    (forward_normal — skipping 02 because too early-stage for property research)
HOP 4 — 03 → 01    (BACK, back_data_missing — preferred_channel unset)
HOP 5 — 01 → 03    (forward_normal — re-qualified with preferred_channel populated)
HOP 6 — 03 → END   (forward_normal — draft produced, do_not_send_yet: false, routed for agent review)
```

### Pass-criteria check

- **HOP 4 (the back-handoff).** Critical envelope under test:
  - `from: 03_client_communication`, `to: 01_lead_qualifier`, `back_to: 01_lead_qualifier`, `handoff_reason: back_data_missing`. ✓
  - `payload.missing_fields: ["preferred_channel"]` — names what's missing for re-qualification.
  - `next_action: "Re-contact lead; capture preferred_channel (text/email/voice). Re-route to 03 with channel populated."`
  - **Layer A.** Rule 0 ✓ (`schema_version`, `case_id` valid; `content_provenance: verified_client` not a quarantine trigger). `back_to` populated correctly. (The envelope layer does not care about `agent_on_deal` regardless of direction; performance preconditions live at Layer B.) ✓
  - **Layer B.** 03's `handoff.md` §4 lists `preferred_channel` missing → `back_data_missing` mapping. ✓ 01's `handoff.md` §6 lists 03 as valid back-handoff source. ✓ 03's per-specialist Rule 0 (`agent_on_deal`) is satisfied on hop 3: 01 emitted `agent_on_deal: team_lead` (sentinel per CALL-022 — Diana's team-default voice file, real resolution); only refusal driver is `payload.preferred_channel: null`. Rule 0 + payload-field-required do not collapse into a single ambiguous refusal cause. ✓
  - **Layer C.** Real-model field names. No analogy leak. ✓
- **HOP 5 (the re-route).** 01 calls Mike back, captures `preferred_channel: text`, re-routes to 03. `parent_envelope_id` chain shows ENV-001 → ENV-002 → ENV-003 (back) → ENV-004. CAS chain unbroken. ✓
- **HOP 6 (the resolved draft).** 03 produces draft text, `do_not_send_yet: false`, agent-review flag set per `decisions/domain-research.md` §2.4 / A9 (every draft reviewed before send). ✓

### T2 verdict

**PASS.** The back-handoff trajectory exercises `back_data_missing` correctly and demonstrates the back-and-forward CAS chain is preserved. One observation: the trajectory works *because* 03 has visibility into what fields are missing — relies on 03's `handoff.md` §1 (Inputs: required-in / optional-in) being explicit about which payload fields are required for which draft type. This is locked in ADR-003 §handoff.md. ✓

---

## T3 — Forward-urgent (04 → 03, forward_urgent)

**Scenario.** Sarah Chen (T1 case), now T-22h from option period end (2026-05-25 13:00 → option ends 2026-05-26 17:00 local). Inspection report just landed: foundation defect requiring $18k repair credit ask. Decision needed before option period closes; client must be reachable by voice (text won't carry the nuance + decision-deadline urgency). 04 → 03 with `forward_urgent`.

### Envelope sequence

```
HOP N — 04 → 03    (forward_urgent — T-22h to option period end, voice contact required)
HOP N+1 — 03 → END (forward_normal — voice-script + email-followup drafted, agent_review_required: true)
```

### Pass-criteria check

- **HOP N (the forward_urgent handoff).** Critical envelope under test:
  - `from: 04_transaction_coordinator`, `to: 03_client_communication`, `back_to: null`, `handoff_reason: forward_urgent`.
  - `payload.urgency_reason: option_period_T_minus_22h`
  - `payload.deadline_at: 2026-05-26T17:00:00-05:00`
  - `payload.escalation_required: voice_contact_first`
  - `payload.context_for_draft`:
    - inspection_findings: "Foundation: 1.5\" differential settlement on east side; structural engineer recommends pier support; estimated repair $14-22k."
    - decision_required: "Repair-credit ask ($18k mid-range) OR terminate during option period OR proceed without ask."
    - constraint: "Decision must be communicated to seller's agent by 2026-05-26 17:00 local."
  - `next_action: "Draft voice-call script for agent_diana to call Sarah immediately. Follow with email recap. Flag agent_review_required: true on both."`
  - `confidence: high` — TC has all the inspection data; urgency is unambiguous.
  - **Layer A.** Rule 0 ✓ (schema, case_id valid; `content_provenance: agent_authored` not a quarantine trigger — TC logged the inspection event from the inspector's report). `handoff_reason: forward_urgent` is one of 6 ✓. `parent_envelope_id` chain intact (extends T1's ENV-004) ✓.
  - **Layer B.** 03's per-specialist Rule 0 ("No `agent_on_deal`, no draft") — `agent_on_deal: "agent_diana"` populated ✓. 03's `handoff.md` §4 has `forward_urgent` mapped to "voice-call script + email recap" deliverable shape (per Austin's pattern + `decisions/domain-research.md` §2.4 — text wins on volume but voice required for high-stakes decisions). ✓
  - **Layer C.** Real-model: `urgency_reason`, `deadline_at`, `escalation_required: voice_contact_first`. No "sticky-tab on the binder" language. ✓
- **HOP N+1 (the produced draft).** 03 produces both voice-script + email-recap drafts. `do_not_send_yet: false` (urgent, can't queue). `agent_review_required: true` (UPL constraint per `decisions/domain-research.md` §2.4 / A9 — agent reviews every comm before send, even under urgency). ✓

### T3 verdict

**PASS.** `forward_urgent` correctly drives both the channel-escalation behaviour (voice first, not text) and the deliverable shape (voice-script alongside email). Demonstrates that under urgency, the architecture does *not* sacrifice the agent-review constraint — UPL discipline holds even at T-22h.

**Observation:** if the foundation issue had triggered a *legal* refusal (e.g., Sarah asks "can the seller force me to close if I don't repair-ask?"), 03's response would back-handoff with `back_compliance_block` (the 6th enum value), not engage with the legal interpretation. T3 doesn't exercise this because the decision is commercial not legal — but T3 sits adjacent to the compliance-block fix and motivates it.

---

## Summary

| Trajectory | Hops | Pass criteria result | Surfaces |
|---|---|---|---|
| T1 happy-path | 6 | PASS | Layering of envelope-level vs per-specialist Rule 0 holds end-to-end |
| T2 back-handoff | 6 | PASS | Confirms CAS chain preservation across back-then-forward |
| T3 forward-urgent | 2 | PASS | Demonstrates UPL discipline under urgency; motivates `back_compliance_block` |

**Good-enough floor met:** 3 trajectories pass on primary use cases without catastrophic failure.
