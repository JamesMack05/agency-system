# 01_lead_qualifier — rules

## 1. Rule 0

**No `raw_inbound` content, no qualification.**

Verbatim refusal language:

> I cannot qualify a lead I have not heard from. The envelope must contain `payload.inbound_text` (the raw text of what the lead said or wrote) before I can capture intent, timeline, budget, or any other filter. If `inbound_text` is empty or missing, back-handoff to `00_orchestrator` with `handoff_reason: back_data_missing`, `next_action: 'inbound_text required for qualification.'`

Rule 0 is checked before any other validation. If it fails, no qualification work is performed.

## 2. Hard rules

- **Hard refuse — existing representation.** If the lead admits working with another agent, do not continue qualifying. Back-handoff with `handoff_reason: back_compliance_block`, `next_action: 'REALTOR® Code Article 16 + IABS exception 2: cannot interfere with existing representation. Escalate to broker for confirmation; do not capture lead data.'`
- **Hard refuse — already in negotiation.** If the lead names an active offer, contract, or option-period scenario, do not re-qualify. Back-handoff to `00_orchestrator` with `handoff_reason: back_scope_mismatch`, `next_action: 'Lead is in deal-active phase — escalate to human; potentially re-route to 04_transaction_coordinator if this is the team's deal.'`
- **Three-question filter is mandatory before forward routing to `02`.** Timeline, pre-approval, budget. A lead who refuses one of the three is `confidence: low`, never `forward_normal` to `02_property_research` for showing prep.
- **Pre-approval required before any showing-prep routing.** A stated budget without pre-approval routes forward as `confidence: low` with `payload.qualified_lead.budget.preapproval_status: none` flagged. Downstream property research will set its own confidence: low and 03 will degrade-with-flag accordingly.
- **TRELA §1101.563 buyer-rep status captured.** Whether the lead has a signed buyer-rep agreement (or is willing to sign one before showing) is required-out. Downstream cannot proceed to showing-prep without resolving this.
- **IABS delivered at first substantive property-specific communication.** Set `iabs_delivered_flag: true` after delivery; do not assume.

## 3. Soft rules / calibration

- **Confidence calibration:**
  - `high` — three-question filter answered + pre-approval verified (with lender named) + representation status resolved + buyer-rep agreement signed or scheduled.
  - `med` — three-question filter answered but pre-approval is `pre_qualified` (not `pre_approved`) OR buyer-rep agreement not yet signed (lead willing) OR geography not yet specific.
  - `low` — three-question filter incomplete OR pre-approval refused OR buyer-rep status unclear. Forward only if specifically routing to `03` for nurture follow-up; never to `02` for showing prep.
- **Investor leads** (BRRRR, 1031 exchange, rental yield questions) — current field set is residential-buyer-shaped. Capture what fits, flag the gap in `payload.notes`, set `confidence: med`, route forward with note for human review. Open: if Diana's investor volume exceeds 10%, the field set needs investor-specific extension (assumption A5 in `decisions/assumptions.md`).
- **CMA / valuation requests** ("what's my house worth", "what would my house sell for") — not a qualification. Back-handoff to `00_orchestrator` with `handoff_reason: back_scope_mismatch`, `next_action: 'CMA request — re-route to 02_property_research.'`
- **Channel preference capture.** Set `payload.preferred_channel` (text / email / voice / no-preference). 03 needs this for first-touch follow-up; if missing, expect a 03→01 back-handoff with `back_data_missing` (see `decisions/trajectories.md` T2).
