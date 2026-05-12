# 04_transaction_coordinator — handoff

Payload contract for `04_transaction_coordinator`. Consumes deal-state events from `00_orchestrator` (deal-active inbounds) or `03_client_communication` (executed-contract handoff). Emits deal-state digests, deadline updates, and `forward_urgent` escalations to `03`. Specialist execution loop: Think → Act → Observe (per `decisions/architecture-properties.md` P3).

## 1. Inputs

**Required-in:**
- `payload.executed_contract_date` — required to initialise deal-active phase. Rule 0.
- `payload.contract_form` (TREC 20-15 / TREC 23 / TREC 30 / other)
- `payload.buyer_rep_agreement.signed_pre_showing` (true / false) — verified at receipt per CALL-015 resolution; if false, immediate `back_compliance_block` per Hard rule.
- `case_id`
- `agent_on_deal`

**Optional-in:**
- `payload.option_fee` { amount, delivered_to, delivered_date, period_end }
- `payload.earnest_money` { amount, delivered_to, delivered_date, escrow_agent }
- `payload.financing` { type, lender, buyer_approval_deadline, property_approval_deadline }
- `payload.inspection_window_end`
- `payload.title_company`
- `payload.deal_state_change` (contract_executed / inspection_complete / financing_approved / closing_disclosure_delivered / funded / other)
- `payload.case_state` (full deal state object — used on subsequent envelopes after initialisation)

## 2. Outputs

**Required-out (every emitted envelope):**
- `schema_version: 1.1.0`
- `case_id`
- `from: 04_transaction_coordinator`
- `to: 03_client_communication` (escalation / agent comms) or `END` (funded) or `back_to` target on refusal
- `back_to`: null on forward; specialist on back-handoff
- `timestamp`
- `agent_on_deal`
- `payload`: shape per below
- `required_fields_present: true`
- `confidence` per §5
- `next_action` (declarative — typically "Track deadlines + daily digest" or "Voice-call client immediately re: [deadline]" for forward_urgent)
- `trail`: append TC event
- `parent_envelope_id`
- `content_provenance`: typically `agent_authored` (TC events are logged by agent) or `system_generated` (deadline-tracking output)
- `case_type`
- `linked_case_ids`
- `handoff_reason`: `forward_normal` / `forward_urgent` (per soft rules) / back-handoff value per §4
- `intermediary_status`

**Payload shape (deal-state digest or escalation):**
- `deal_state_change` (contract_executed / option_period_ending / inspection_complete / financing_approved / closing_disclosure_delivered / funded / other)
- `deadlines[]`: array of { deadline_name, due_at, status (pending / met / missed / blocked), blocker_if_any }
- `documents[]`: array of { doc_name, owner, status (received / pending / overdue), received_at }
- `urgency_reason` (when `forward_urgent`: option_period_T_minus_24h / financing_contingency_T_minus_24h / closing_T_minus_72h / other)
- `deadline_at` (when `forward_urgent`: ISO 8601)
- `escalation_required` (when `forward_urgent`: voice_contact_first / email_only / either)
- `context_for_draft` (when forwarding to `03`: free-text summary of what the draft needs to address)
- `digest_summary` (daily/weekly digest body for agent review)
- `notes`

## 3. Routing destinations

Forward `to`:
- `03_client_communication` — `forward_urgent` for T-24h escalations (voice contact required); `forward_normal` for routine status-update drafts (weekly digest, milestone announcements).
- `END` — terminal envelope on funding (`deal_state_change: funded`).

Back `back_to`:
- `00_orchestrator` — for hard refusals (no buyer-rep, UPL on advise-questions).
- `02_property_research` — for mid-deal research needs (comp-recheck for repair-credit ask, neighborhood disclosure deepening).
- `03_client_communication` — when `04` requested a draft from `03` and `03` refused (compliance, quality) and the refusal needs `04`-side handling rather than escalation.

## 4. Refusal triggers

Per CALL-019, refusal triggers map verbatim to ADR-001 Extension 2 enum values:

| Condition | `back_to` | `handoff_reason` |
|---|---|---|
| `payload.executed_contract_date` missing | `00_orchestrator` | `back_data_missing` |
| `payload.buyer_rep_agreement.signed_pre_showing: false` (or absent) | `00_orchestrator` | `back_compliance_block` (TRELA §1101.563 — per CALL-015 resolution) |
| Required deal-state field missing for the requested operation (e.g., `inspection_complete` event with no `inspector` or `inspection_findings`) | `00_orchestrator` | `back_data_missing` |
| Asked to advise on terminating, earnest-money refundability, breach interpretation, or any "should I" question | `03_client_communication` (for refusal-language drafting via UPL recommendation) OR `00_orchestrator` (for routing escalation) | `back_compliance_block` |
| Mid-deal research needed beyond what's in envelope | `02_property_research` | `back_data_missing` |
| Initial research brief was `confidence: low` but deal-active phase needs higher confidence | `02_property_research` | `back_quality_failure` |

**Enum reference (full set, per ADR-001 Extension 2):**
`forward_normal` / `forward_urgent` / `back_data_missing` / `back_scope_mismatch` / `back_quality_failure` / `back_compliance_block`

## 5. Confidence calibration

- `high` — all contract dates resolved, all addenda accounted for, both fee fields populated with delivery confirmations, financing dual-deadline set, HOA + SDN status verified, lender + title + inspector named.
- `med` — dates resolved but one or more documents not yet on file (e.g., SDN delivered but inspector not yet selected), OR HOA present but resale certificate not yet requested, OR financing tracked but property-approval deadline still inferred (not yet set in writing).
- `low` — financing details ambiguous (e.g., cash deal flagged as conventional in error), OR contract addenda referenced but text not on file, OR pre-execution requirements (SDN, buyer-rep) only partially verified, OR county-specific resources require cross-reference (Hays/Williamson) and tax-roll cannot be confirmed.

## 6. Back-handoff sources

Specialists that may `back_to: 04_transaction_coordinator`:

| From | `handoff_reason` | Trigger |
|---|---|---|
| `03_client_communication` | `back_compliance_block` | `04` requested a draft from `03` that crosses UPL (e.g., draft a client interpretation of an inspection report); `03` refused and the refusal needs `04`-side handling (TC adjusts deadline/escalation plan accordingly). |
| `03_client_communication` | `back_data_missing` | `04`'s `forward_urgent` envelope to `03` lacked context for the draft (e.g., missing `context_for_draft.decision_required`); `04` re-sends with the missing context. |

`04` does not typically receive back-handoffs because it sits at the deal-active terminus (only `END` is downstream). Most exception flow is `04`-originated forward / back to upstream specialists, not back-to-`04` from elsewhere.
