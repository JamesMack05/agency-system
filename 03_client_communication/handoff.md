# 03_client_communication — handoff

Payload contract for `03_client_communication`. Consumes draft requests from any upstream specialist (`00`, `01`, `02`, `04`). Emits drafts; never sends. Specialist execution loop: Think → Act → Observe (per `decisions/architecture-properties.md` P3).

## 1. Inputs

**Required-in:**
- `agent_on_deal` — required for voice-file resolution. Rule 0. May be a named agent (e.g., `agent_diana`) or the `team_lead` sentinel (boutique-team default, resolves to `voice/team_lead.md`); both satisfy Rule 0 per CALL-022.
- `payload.draft_request.purpose` (first_touch_follow_up / showing_thank_you / cadence_check_in / status_update / urgent_decision / refusal_with_recommendation)
- `payload.draft_request.channel_preference` (text / email / voice — from `payload.preferred_channel` upstream, or explicitly overridden)
- `case_id`
- `content_provenance` (drives whether personalised salutation is permitted)

**Optional-in:**
- `payload.qualified_lead` (drives lead-context for first-touch drafts)
- `payload.agent_pull_quotes` from research (verbatim-usable facts from `02`)
- `payload.case_state` (deal-state from `04` for status updates)
- `payload.urgency_reason` + `payload.deadline_at` (when receiving `forward_urgent` from `04`)
- `payload.context_for_draft` — free-text from upstream specialist explaining what the draft needs to address

## 2. Outputs

**Required-out (every emitted envelope):**
- `schema_version: 1.1.0`
- `case_id`
- `from: 03_client_communication`
- `to: 04_transaction_coordinator` (deal-active follow-up) or `END` (one-shot follow-up complete) or `back_to` target on refusal
- `back_to`: null on forward; specialist on back-handoff
- `timestamp`
- `agent_on_deal` (carried through)
- `payload`: shape per below
- `required_fields_present: true` (after draft + flags populated)
- `confidence` per §5
- `next_action`: typically "Agent review and send" (or "Agent review and queue for next valid send window" when do_not_send_yet)
- `trail`: append draft-produced event
- `parent_envelope_id`
- `content_provenance`: carried through (typically `verified_client` or `agent_authored`; rare `anonymous_inbound` cases trigger neutral salutation)
- `case_type`
- `linked_case_ids`
- `handoff_reason`: `forward_normal` (or back-handoff value per §4)
- `intermediary_status`: carried through

**Payload shape (drafted message):**
- `channel` (text / email / voice_call_script)
- `recipient_handle`
- `subject_line` (email only)
- `body_draft` (the message itself, in `agent_on_deal`'s voice)
- `agent_review_required: true` (always)
- `voice_file_used` (path to voice profile, or `null` if degraded to baseline)
- `do_not_send_yet` (true / false)
- `do_not_send_yet_reason` (when true: voice_missing / outside_send_window / sensitive_content_pending_agent_decision / refusal_recommendation_pending_agent_review)
- `tcpa_window_check` (sent_within_window: true / false / na_email)
- `notes` (free-text caveats — UPL refusal context, voice-degrade flag, etc.)

## 3. Routing destinations

Forward `to`:
- `04_transaction_coordinator` — drafts that signal deal-state transitions (e.g., "client confirmed they want to make an offer" → 04 prepares to receive executed contract).
- `END` — one-shot drafts that complete a follow-up cycle (e.g., first-touch acknowledgment with no expected next-action from system).

Back `back_to`:
- `00_orchestrator` — for UPL refusals (escalate to broker / attorney via human routing).
- `01_lead_qualifier` — for missing draft context (preferred_channel, lead intent for personalisation).
- `02_property_research` — for missing research context (e.g., draft needs comp framing that wasn't in the brief).

## 4. Refusal triggers

Per CALL-019 — `03` is the **primary site** for `back_compliance_block` (UPL refusals). Refusal triggers map verbatim to ADR-001 Extension 2 enum values:

| Condition | `back_to` | `handoff_reason` |
|---|---|---|
| `agent_on_deal` null or missing | producing specialist | `back_data_missing` |
| Client requests legal interpretation, contract enforceability opinion, termination rights advice, or contract addendum drafting | `00_orchestrator` | `back_compliance_block` |
| `payload.preferred_channel` missing | `01_lead_qualifier` | `back_data_missing` |
| Personalisation requested but `content_provenance: anonymous_inbound` (cannot draft "Hi [name]" for unverified identity) | `01_lead_qualifier` | `back_data_missing` (re-route through identity verification) |
| Voice file missing AND `do_not_send_yet: true` is unacceptable to upstream (e.g., upstream demands send-now) | producing specialist | `back_quality_failure` |
| Draft would violate TCPA outbound time AND `forward_urgent` is not set | (no back — queue locally with `do_not_send_yet: true`; soft-rule, not refusal) | (N/A) |

**Enum reference (full set, per ADR-001 Extension 2):**
`forward_normal` / `forward_urgent` / `back_data_missing` / `back_scope_mismatch` / `back_quality_failure` / `back_compliance_block`

## 5. Confidence calibration

Confidence reflects the *draft's send-readiness*, not just the gating state. Two drafts can both carry `do_not_send_yet: true` and warrant different confidence — content-degraded drafts cap at `low` (the agent may want to rewrite); timing-gated drafts cap at `med` (the draft is ready, only the send window is closed). Cause is captured in `do_not_send_yet_reason`; confidence dispatches on cause (see CALL-021).

- `high` — `agent_on_deal` resolves to existing voice file + `content_provenance: verified_client` + channel preference clear + draft purpose matches a templatable case (first-touch, thank-you, status update) + `do_not_send_yet: false`.
- `med` — voice file resolves but channel preference unclear (last-used inferred), OR draft is for a new deal-state transition (first message in option period, etc.), OR cadence-class draft (every-3-days check-in), OR `do_not_send_yet: true` with `do_not_send_yet_reason: outside_send_window` (timing gate only — draft content is ready).
- `low` — voice file missing (degrade-with-flag per CALL-020 / Real Estate Domain Research §2.4 — `do_not_send_yet_reason: voice_missing`), OR draft involves UPL-adjacent topic and refusal language is being suggested (`do_not_send_yet_reason: refusal_recommendation_pending_agent_review`), OR sensitive content pending agent decision (`do_not_send_yet_reason: sensitive_content_pending_agent_decision`). Content is degraded; agent may rewrite, not just release.

## 6. Back-handoff sources

Specialists that may `back_to: 03_client_communication`:

`03` rarely receives back-handoffs because it is the most-downstream specialist before `END` for non-deal-active cases, and for deal-active cases `04` typically forwards (not back-handoffs) to `03` via `forward_urgent`. Documented sparseness:

| From | `handoff_reason` | Trigger |
|---|---|---|
| (none typical) | (N/A) | `03` is a draft-producing terminus; back-handoffs to `03` would be requests to re-draft, which are typically expressed as new forward envelopes (not back-handoffs). |

If a real back-handoff to `03` ever surfaces (e.g., `04` rejects a draft as off-message), it would carry `back_quality_failure` — but the architecture's expected pattern is for `04` to comment via agent and request a new draft via fresh forward envelope.
