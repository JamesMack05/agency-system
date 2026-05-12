# 00_orchestrator — handoff

Payload contract for `00_orchestrator`. Consumes `INTAKE.md`; emits a single envelope per inbound. Specialist execution loop: Think → Act → Observe (per `decisions/architecture-properties.md` P3).

## 1. Inputs

**Required-in:**
- `INTAKE.md` file present at expected location with non-empty content (raw inbound: text / email / voice transcript / web-form).

**Optional-in:**
- Sender hint (channel, named handle, or known case reference embedded in inbound).
- Urgency markers in inbound text ("ASAP", "deadline today", explicit T-N hours).

`00` is the entry point — no upstream envelope to consume. Input arrives as a file, not as an envelope.

## 2. Outputs

**Required-out (every emitted envelope):**
- `schema_version` (current: `1.1.0`)
- `case_id` (`CASE-YYYY-NNNN` format; reused if matching existing open case, generated fresh otherwise)
- `from: 00_orchestrator`
- `to: <one of 01_lead_qualifier / 02_property_research / 03_client_communication / 04_transaction_coordinator>`
- `back_to: null` (00 only forwards)
- `timestamp` (ISO 8601)
- `agent_on_deal: null` — 00 always emits null; producing site is `01_lead_qualifier` per CALL-022
- `payload`: shape per below
- `required_fields_present: true`
- `confidence` (`high` / `med` / `low` per rules.md §3 calibration)
- `next_action` (declarative instruction to receiving specialist)
- `trail`: single entry with the routing event
- `parent_envelope_id: null` (00 envelopes are first-in-case)
- `content_provenance` (`anonymous_inbound` / `verified_client` / `agent_authored`)
- `case_type` (one of 6 values from envelope schema)
- `linked_case_ids: []` unless inbound references known related cases
- `handoff_reason: forward_normal` (or `forward_urgent` per soft rules)
- `intermediary_status: null` unless jurisdiction requires (TX TRELA — set explicit `yes` / `no` based on team's representation of both parties)

**Payload shape:**
- `inbound_channel` (text / email / voice / web_form / zillow_relay)
- `inbound_text` (raw, verbatim from `INTAKE.md`)
- `sender_handle` (best contact handle if known)
- `subject_classification` (qualification / property_research / communication_draft / deal_active)
- `inbound_property_ref` (MLS ID or address if specified)
- `urgency` (normal / urgent / null)

## 3. Routing destinations

Forward `to`:
- `01_lead_qualifier` — new lead inbounds, qualification questions, returning leads with stale qualification.
- `02_property_research` — CMA requests, neighborhood research questions, comp queries on specific properties.
- `03_client_communication` — drafting requests for known clients (e.g., "send Sarah a thank-you for the showing").
- `04_transaction_coordinator` — deal-active events (executed contract, title-co update, lender milestone, deadline change).

`back_to: null` — `00` only forwards. Back-handoffs from downstream are received by `00` (see §6).

## 4. Refusal triggers

Per CALL-019, refusal triggers map verbatim to ADR-001 Extension 2 enum values:

| Condition | `handoff_reason` |
|---|---|
| `INTAKE.md` missing or empty | (no envelope produced — Rule 0; processing halts) |
| Subject classification ambiguous (>1 plausible specialist, no tie-breaker) | (no envelope produced — escalate to human; not a back-handoff because there's no upstream specialist) |
| Inbound references property the team has no listing/buyer-rep on | (no envelope produced — escalate to human via system log) |
| Inbound names a counterparty already represented by another agent | `back_compliance_block` — but `00` is the entry point so this surfaces as a refusal-to-route (logged, escalated to broker). Pattern: emit a routing-refusal envelope `to: 00_orchestrator` (self) with `handoff_reason: back_compliance_block`, `next_action: 'IABS exception 2 + REALTOR® Code Article 16: cannot interfere with existing representation. Escalate to broker.'` |

**Enum reference (full set, per ADR-001 Extension 2):**
`forward_normal` / `forward_urgent` / `back_data_missing` / `back_scope_mismatch` / `back_quality_failure` / `back_compliance_block`

## 5. Confidence calibration

- `high` — sender identity verified against existing case (handle/number/email match).
- `med` — sender named in inbound but no case match (new lead with stated identity).
- `low` — sender identity inferred from channel only (anonymous Zillow lead, web-form with no name field, voicemail with no callback).

## 6. Back-handoff sources

`00_orchestrator` is the universal back-target. Any specialist may `back_to: 00_orchestrator` for the following reasons:

| From | `handoff_reason` | Trigger |
|---|---|---|
| `01_lead_qualifier` | `back_scope_mismatch` | Lead is in negotiation (deal-active); needs re-routing or human escalation. |
| `01_lead_qualifier` | `back_compliance_block` | Lead admits existing representation; escalate to broker. |
| `01_lead_qualifier` | `back_scope_mismatch` | CMA / valuation request misrouted to qualifier; re-route to `02`. |
| `02_property_research` | `back_scope_mismatch` | Subject too broad ("research Austin housing"); needs specific subject. |
| `02_property_research` | `back_compliance_block` | UPL on legal interpretation of HOA covenants / easements. |
| `03_client_communication` | `back_compliance_block` | UPL — client requested legal interpretation; surface to agent. |
| `03_client_communication` | `back_data_missing` | Missing channel preference, voice file, or other required draft input. |
| `04_transaction_coordinator` | `back_compliance_block` | TRELA §1101.563 violation (no buyer-rep agreement) or UPL on advise-to-terminate. |
| `04_transaction_coordinator` | `back_data_missing` | Required deal-state field missing from incoming envelope. |

On receipt of a back-handoff, `00` either re-routes (if a different specialist owns the input) or escalates to human (if no specialist can take it).
