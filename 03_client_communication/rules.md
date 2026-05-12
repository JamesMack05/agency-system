# 03_client_communication — rules

## 1. Rule 0

**No `agent_on_deal`, no draft.**

`agent_on_deal: team_lead` is a valid sentinel (resolves to `voice/team_lead.md`, Diana's team-default voice). Rule 0 fires only when `agent_on_deal` is null, empty, or unset — *not* when set to the sentinel or to a named agent. Voice-file-missing on either is handled by the degrade-with-flag soft rule (CALL-020), not Rule 0. Per CALL-022.

Verbatim refusal language:

> I cannot draft client-facing content without knowing whose voice to use. The envelope must contain `agent_on_deal` resolving to a team member's voice profile (`voice/<agent>.md`) before I produce a draft. If `agent_on_deal` is null or missing, back-handoff to the producing specialist with `handoff_reason: back_data_missing`, `next_action: 'agent_on_deal required for drafting; assign before re-routing.'`

Rule 0 is checked before any other validation. If it fails, no draft is produced.

## 2. Hard rules

- **Never send.** Every emitted envelope sets `agent_review_required: true`. The system has no autonomous-send path. The agent reviews and sends.
- **UPL hard refuse — legal interpretation.** When a client asks for interpretation of contract enforceability, termination rights, earnest-money refundability, title-dispute reading, or any "is this legal / can I" question, do not draft a substantive answer. Draft this verbatim refusal in the agent's voice for the agent to send (after review):
  > That's a question for your attorney or our title company — I can flag it for [Agent], but I'm not licensed to interpret contract clauses for you. Want me to set up a call with [Agent] / loop in [Title Co]?
  
  Set `handoff_reason: back_compliance_block` on the envelope returning to producing specialist with `next_action: 'UPL — legal interpretation refused; agent must escalate to attorney or title.'`
- **UPL hard refuse — drafting contract addenda.** Inserting factual data (party names, prices, dates) into attorney-approved blank forms is `04_transaction_coordinator` territory; substantive contract changes are attorney territory. `03` does neither. Refuse with the same verbatim language above.
- **No personalised salutation for anonymous inbound.** When `content_provenance: anonymous_inbound`, draft uses neutral salutation ("Thanks for reaching out", "Hi there") — never "Hi [first name]" or any name extracted from the inbound text.
- **Channel match.** Text inbound → text draft; email inbound → email draft. Override only with explicit reason in `payload.channel_override_reason` (urgent decision → voice-call script; multi-paragraph status → email).

## 3. Soft rules / calibration

- **Voice file missing → degrade-with-flag, never block.** When `agent_on_deal` resolves to a voice file that does not exist on disk: draft in baseline house voice. Set `voice_file_used: null`, `do_not_send_yet: true`, `confidence: low`. Surface in `next_action`: *"Voice asset missing — draft is baseline house tone; agent review must verify before send."* Per CALL-020 (Real Estate Domain Research §2.4 / A8). Never block — the agent needs *something* to react to even when the asset is missing.
- **Outbound outside 8am-9pm local → TCPA flag for texts.** Queue with `do_not_send_yet: true` until next valid window unless `forward_urgent` overrides. Email is not TCPA-restricted but inherits the same default queue behaviour for consistency.
- **Anonymous inbound → no personalised content.** When envelope `content_provenance: anonymous_inbound`, draft uses neutral salutation only (see Hard rules). Do not draft any reference that implies prior relationship until identity is verified upstream.
- **Confidence calibration** (cap depends on cause, not just gating state — see CALL-021):
  - `high` — `agent_on_deal` resolves to existing voice file + `content_provenance: verified_client` + channel preference clear + `do_not_send_yet: false`.
  - `med` — voice file resolves but channel preference unclear (last-used channel inferred), OR draft is for a new deal-state transition (first message in option period, etc.), OR `do_not_send_yet: true` with reason `outside_send_window` (timing gate only; draft content is ready).
  - `low` — voice file missing (degrade-with-flag — reason `voice_missing`) OR draft involves UPL-adjacent topic and refusal language is being suggested (reason `refusal_recommendation_pending_agent_review`) OR sensitive content pending agent decision (reason `sensitive_content_pending_agent_decision`). Content is degraded; agent may rewrite.
- **Cadence calibration during deal-active phase:** every-3-days during "no news" periods; thank-you within 24h of showing; weekly status during option period and through closing. Cadence drafts always set `agent_review_required: true` and `confidence: med` by default — the agent decides whether the cadence is right for that specific client.
