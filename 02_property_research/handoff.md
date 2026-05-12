# 02_property_research — handoff

Payload contract for `02_property_research`. Consumes research requests from `00_orchestrator` (direct CMA inbound) or `01_lead_qualifier` (qualified lead needing showing-prep research) or `04_transaction_coordinator` (mid-deal research for a client question). Emits sourced brief. Specialist execution loop: Think → Act → Observe (per `decisions/architecture-properties.md` P3).

## 1. Inputs

**Required-in:**
- `payload.research_request.subject` — specific property (MLS ID, street address, parcel number) OR specific neighborhood (named sub-area, ISD attendance zone, zip + sub-area). Rule 0.
- `payload.research_request.purpose` — `cma_only` / `showing_prep` / `neighborhood_brief` / `valuation_comparison`. Enum aligned with `01_lead_qualifier/handoff.md` §2 emit set; describes the *kind of research* requested, not the downstream comm purpose (downstream comm purpose surfaces in `payload.draft_request.purpose` on 02→03 emission).
- `case_id`

**Optional-in:**
- `payload.qualified_lead` (carried through from `01` if research is for a qualified lead — drives confidence calibration on whether brief is for represented client or neutral).
- `payload.case_state` (carried through from `04` if research is mid-deal — drives whether to focus on current contract context).
- `payload.geography_context` (county, sub-area, ISD).

## 2. Outputs

**Required-out (every emitted envelope):**
- `schema_version: 1.1.0`
- `case_id`
- `from: 02_property_research`
- `to: 03_client_communication` (research feeds drafting) or `04_transaction_coordinator` (research informs deal-active context) or `back_to` target on refusal
- `back_to`: null on forward; specialist on back-handoff
- `timestamp`
- `agent_on_deal` (carried through; required for downstream 03 drafting)
- `payload`: shape per below
- `required_fields_present: true` (after sources_named populated)
- `confidence` per §5
- `next_action`
- `trail`: append research event
- `parent_envelope_id`
- `content_provenance`: typically inherited from incoming envelope (research is data assembly, not source-changing)
- `case_type`: carried through
- `linked_case_ids`
- `handoff_reason`: `forward_normal` (or back-handoff value per §4)
- `intermediary_status`: carried through

**Payload shape (research brief):**
- `brief_subject` — exact MLS ID + street address, OR named neighborhood + ISD
- `agent_pull_quotes` — 3-5 facts the agent can paste verbatim into client comms (each item: declarative sentence + source reference)
- `sources_named` — array of source identifiers (MLS listing IDs, austincad.org parcel URLs, TraviCAD/HaysCAD/WCAD URLs, ISD attendance pages, etc.)
- `comp_set` (when CMA): array of {address, close_date, close_price, sqft, beds, baths, lot_size, year_built, days_on_market, condition_adjustment, source_mls_id}
- `comp_count` + `comp_window_days`
- `neighborhood_profile` (when applicable): { isd_elementary, isd_middle, isd_high, mud_status, pid_status, hoa_status, flood_zone, age_of_construction, walk_score }
- `representation_context` (represented / neutral) — neutral when team has no listing/buyer-rep on subject
- `notes` (free-text for caveats, county-boundary edge cases, missing data flags)
- `draft_request` (when forwarding to 03): { purpose, channel_preference }

## 3. Routing destinations

Forward `to`:
- `03_client_communication` — research feeding immediate client comms (first-touch follow-up, showing prep summary, offer-prep talking points).
- `04_transaction_coordinator` — research informing deal-active context (e.g., comp recheck mid-option-period, neighborhood disclosure for SDN review).

Back `back_to`:
- `00_orchestrator` — for over-broad subjects, valuation-opinion refusals.
- `01_lead_qualifier` — for missing qualification context (e.g., budget-anchored CMA without budget).

## 4. Refusal triggers

Per CALL-019, refusal triggers map verbatim to ADR-001 Extension 2 enum values:

| Condition | `back_to` | `handoff_reason` |
|---|---|---|
| `payload.research_request.subject` over-broad ("research Austin housing market") | `00_orchestrator` | `back_scope_mismatch` |
| `payload.research_request.subject` missing entirely | `00_orchestrator` | `back_data_missing` |
| Asked for valuation opinion ("is this overpriced?") | `00_orchestrator` | `back_scope_mismatch` |
| Asked for legal interpretation of HOA covenant / easement / title issue | `00_orchestrator` | `back_compliance_block` |
| CMA requested but `payload.qualified_lead.budget` missing (cannot anchor low/med/high framing) | `01_lead_qualifier` | `back_data_missing` |
| Fewer than 3 viable comps in 90 days AND deal-active context (i.e., cannot produce confidence: low brief because deal phase needs higher confidence) | `00_orchestrator` | `back_quality_failure` |

**Enum reference (full set, per ADR-001 Extension 2):**
`forward_normal` / `forward_urgent` / `back_data_missing` / `back_scope_mismatch` / `back_quality_failure` / `back_compliance_block`

## 5. Confidence calibration

- `high` — 4+ comps within 60 days, all sources named, MUD/PID/HOA all resolved, condition adjustments documented, represented context.
- `med` — 3 comps within 90 days, OR MUD present but tax-rate from secondary source, OR comp set requires unusual condition adjustments, OR neutral context (team has no listing/buyer-rep).
- `low` — fewer than 3 viable comps in 90 days, OR MUD/PID/HOA not fully resolved, OR subject is novel construction (no comparable closed sales), OR sources include unverified secondary references. Forward with `confidence: low` and explicit gap callout in `payload.notes`.

## 6. Back-handoff sources

Specialists that may `back_to: 02_property_research`:

| From | `handoff_reason` | Trigger |
|---|---|---|
| `03_client_communication` | `back_data_missing` | Draft requires research context that isn't in the envelope (e.g., draft needs school-district detail for a buyer with kids; comp set has it but neighborhood profile section was omitted). |
| `04_transaction_coordinator` | `back_data_missing` | Mid-deal question requires deeper research than initially pulled (e.g., comp-recheck for repair-credit ask after inspection). |
| `04_transaction_coordinator` | `back_quality_failure` | Initial brief flagged `confidence: low` but deal-active phase needs higher confidence; re-research with expanded comp window or wider geographic radius. |

On receipt of a back-handoff, `02` re-runs the brief with the requested expansion or fills the missing section, then re-routes forward.
