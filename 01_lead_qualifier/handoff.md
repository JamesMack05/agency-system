# 01_lead_qualifier — handoff

Payload contract for `01_lead_qualifier`. Consumes routed inbounds from `00_orchestrator`; emits qualified-lead payload. Specialist execution loop: Think → Act → Observe (per `decisions/architecture-properties.md` P3).

## 1. Inputs

**Required-in (from `00_orchestrator`):**
- `payload.inbound_text` (raw lead inbound — Rule 0)
- `payload.inbound_channel`
- `payload.subject_classification: qualification`
- `content_provenance` (drives identity-trust calibration)
- `case_id` (new or returning)

**Optional-in:**
- `payload.sender_handle` (for follow-up routing)
- `payload.urgency` (passes through to qualified-lead payload)
- `linked_case_ids` (if returning client with related cases)

## 2. Outputs

**Required-out (every emitted envelope):**
- `schema_version: 1.1.0`
- `case_id` (carried through from inbound)
- `from: 01_lead_qualifier`
- `to: 02_property_research` (showing-prep route) or `03_client_communication` (immediate-follow-up route) or `back_to` target on refusal
- `back_to`: null on forward; specialist on back-handoff
- `timestamp`
- `agent_on_deal`: **required on every forward emission to 02 / 03 / 04; null acceptable only on back-handoffs to 00.** Set to a named agent (e.g., `agent_diana`) when the team's assignment table at `decisions/assumptions.md` § Team Assignment Table maps the qualified lead to one (typically by `qualified_lead.sub_type` + top `geography`); otherwise default to the `team_lead` sentinel (resolves to `voice/team_lead.md` — Diana's team-default voice file). Never null on forward — 03's Rule 0 demands a value downstream. Per CALL-022.
- `payload`: shape per below
- `required_fields_present: true` (after three-question filter complete)
- `confidence` per §5
- `next_action`
- `trail`: append qualifier event
- `parent_envelope_id`: prior envelope ID (CAS chain)
- `content_provenance`: typically flips `anonymous_inbound → verified_client` after qualification call confirms identity
- `case_type`: residential_buyer_side / residential_seller_side / residential_both / unknown (set on qualification)
- `linked_case_ids`: carried through
- `handoff_reason`: `forward_normal` (or back-handoff value per §4)
- `intermediary_status`: set to `yes`/`no` if known by qualification; otherwise null

**Payload shape (qualified-lead):**
- `qualified_lead`:
  - `intent` (buy / sell / both / unsure) + `sub_type` (first_time / move_up / downsize / relocate / investor)
  - `timeline` (0-30_days / 30-90_days / 90_plus_days / exploratory)
  - `budget`: { `range`, `preapproval_status` (none / pre_qualified / pre_approved / cash), `preapproval_lender` (named) }
  - `geography` (array of Austin metro sub-areas + ISD priorities)
  - `property_type` (SFH / condo / townhome / new_construction)
  - `current_representation_status` (none / working_with_other_agent / working_with_us)
  - `buyer_rep_agreement_status` (signed / not_yet_signed / refused) + `signed_pre_showing` (true / false / N/A)
  - `iabs_delivered_flag` (true / false)
- `preferred_channel` (text / email / voice / no_preference) — required for downstream `03` drafting
- `raw_inbound` (carried through verbatim)
- `notes` (free-text for caveats, e.g. investor edge cases per A5)
- `research_request`: { `subject`, `purpose` } — **required when `to: 02_property_research`**. `subject` is derived from `inbound_property_ref` (carried through from `00_orchestrator` when the inbound named a specific property), else composed from top `geography` + `property_type` with a band suffix (e.g. `"Tarrytown SFH band — 800-950k"`). `purpose` ∈ {`cma_only`, `showing_prep`, `neighborhood_brief`, `valuation_comparison`} — chosen by the qualifier from the qualification call's stated need. Forwarding to `02_property_research` without this field fires 02's Rule 0 (back-handoff with `back_data_missing`). Not emitted on routes other than `02_property_research` (null when forwarding to `03_client_communication` or back-handing off).

## 3. Routing destinations

Forward `to`:
- `02_property_research` — qualified leads needing CMA / neighborhood research before agent can act (typical when client asks about a specific property or wants to compare options).
- `03_client_communication` — qualified leads needing immediate first-touch follow-up (e.g., new lead from web form expecting an acknowledgment within 24h).
- `04_transaction_coordinator` — extremely rare; only if a returning client's qualification surfaces a deal-active state that wasn't routed there by `00`.

**Tie-breaker when both research and first-touch apply** (per CALL-023). Default to `02_property_research` first — research-informed first-touch produces more substantive client comms (comp set, neighborhood facts) in the opening outreach. **Exception: route to `03_client_communication` first when ANY of the following holds**, because the disclosure must precede any showing-prep:
- The inbound names a property held by the team's brokerage (`payload.inbound_property_ref` resolves to a team listing — intermediary status requires written consent under TRELA before dual-rep can proceed).
- `payload.qualified_lead.buyer_rep_agreement_status: not_yet_signed` AND the lead has named a specific property or requested a showing window (TRELA §1101.563 blocks showing until signed; first-touch carries the buyer-rep ask and IABS delivery).
- `payload.qualified_lead.iabs_delivered_flag: false` AND substantive property-specific discussion is imminent (IABS must precede first substantive comms about a specific property).

In all other cases, default to `02_property_research`; 03 then produces the comp-anchored first-touch in the next hop.

Back `back_to`:
- `00_orchestrator` — for hard refusals (existing representation, deal-active, CMA-misrouted) and for inbounds that need re-routing.

## 4. Refusal triggers

Per CALL-019, refusal triggers map verbatim to ADR-001 Extension 2 enum values:

| Condition | `back_to` | `handoff_reason` |
|---|---|---|
| `payload.inbound_text` empty or missing | `00_orchestrator` | `back_data_missing` |
| Lead admits existing representation | `00_orchestrator` | `back_compliance_block` |
| Lead is in negotiation phase (active offer / contract / option period) | `00_orchestrator` | `back_scope_mismatch` |
| CMA / valuation request misrouted to qualifier | `00_orchestrator` | `back_scope_mismatch` |
| Required identity-verification step refused (lead won't share name/contact) and `content_provenance` cannot flip from `anonymous_inbound` | `00_orchestrator` | `back_quality_failure` |

**Enum reference (full set, per ADR-001 Extension 2):**
`forward_normal` / `forward_urgent` / `back_data_missing` / `back_scope_mismatch` / `back_quality_failure` / `back_compliance_block`

## 5. Confidence calibration

- `high` — three-question filter answered (timeline + pre-approval + budget); pre-approval verified with lender named; representation status resolved (`none` or `working_with_us`); buyer-rep agreement signed or scheduled; geography specific.
- `med` — three-question filter answered but pre-approval is `pre_qualified` (not `pre_approved`), OR buyer-rep agreement not yet signed (lead willing to sign), OR geography is sub-area-level only.
- `low` — three-question filter incomplete OR pre-approval refused OR buyer-rep status unclear OR investor edge case (current field set is residential-buyer-shaped). Forward only to `03` for nurture follow-up; never to `02` for showing prep.

## 6. Back-handoff sources

Specialists that may `back_to: 01_lead_qualifier`:

| From | `handoff_reason` | Trigger |
|---|---|---|
| `02_property_research` | `back_data_missing` | Qualification context insufficient for the research request (e.g. price band asked but no budget context to anchor "low/med/high" framing). |
| `03_client_communication` | `back_data_missing` | Required draft input missing — typically `payload.preferred_channel` (per `decisions/trajectories.md` T2). |
| `03_client_communication` | `back_data_missing` | Missing context for a personalised follow-up (e.g., draft requires lead's stated property type but qualification was incomplete). |

On receipt of a back-handoff, `01` re-contacts the lead to capture the missing field and re-routes forward.
