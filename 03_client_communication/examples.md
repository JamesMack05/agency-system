# 03_client_communication — examples

Four comparative pairs. 03 is the **primary site** for `back_compliance_block` (UPL refusals) and the canonical degrade-with-flag specialist (voice-file-missing per CALL-020). Decision boundaries cluster around refusal-vs-draft, personalisation-vs-anonymity, and queue-vs-send. Real-model voice per CALL-010.

---

## Pair 1 — UPL refusal: do not interpret, draft the refusal

### Input artifact

Inbound envelope from `00_orchestrator` (client text mid-option-period):

```yaml
case_id: CASE-2026-0042
agent_on_deal: agent_diana
content_provenance: verified_client
case_type: residential_buyer_side
payload:
  inbound_channel: text
  inbound_text: |
    Hey — quick question. If the option period is up Friday and I haven't
    decided, can I still get my earnest money back if I terminate over
    the weekend? Or am I locked in once option period ends?
  sender_handle: +1-512-555-0142
  subject_classification: communication_draft
  draft_request:
    purpose: reactive_response
    channel_preference: text
```

### Bad output

```yaml
from: 03_client_communication
to: END
confidence: high                                # ← bad: confidently wrong
handoff_reason: forward_normal
payload:
  channel: text
  recipient_handle: +1-512-555-0142
  body_draft: |
    Hi Sarah — great question! Once the option period ends Friday at 5pm,
    your unrestricted termination right ends with it, but your earnest
    money is still refundable if you terminate for any reason covered
    by the contract (financing falling through, title issues, etc.).
    Just not for "I changed my mind" reasons. So yes, you'd still likely
    get it back if there's a real cause — let me know if you want to talk
    through specifics.
  agent_review_required: true                   # ← present but doesn't save the draft
  voice_file_used: voice/diana.md
  do_not_send_yet: false
```

### Good output

```yaml
from: 03_client_communication
to: 00_orchestrator
back_to: 00_orchestrator
handoff_reason: back_compliance_block
confidence: low                                 # ← refusal-language draft, low confidence in send-as-is
payload:
  refusal_reason: upl_legal_interpretation_requested
  refusal_detail: |
    Client asked whether earnest money is recoverable post-option-period
    and whether they're "locked in." Both are contract-interpretation /
    termination-rights questions. UPL: 03 cannot interpret. Drafted the
    standard UPL recommendation in Diana's voice for her to review + send.
  proposed_draft:
    channel: text
    recipient_handle: +1-512-555-0142
    body_draft: |
      Hey Sarah — good question and an important one. That's a contract-
      interpretation call that needs to come from your attorney or
      Heritage Title (our title company), not me — I'd give you the wrong
      answer trying to summarise it. Want me to loop in Heritage now?
      They can answer same-day. Or I can connect you with the attorney
      we recommend if you'd prefer that route.
    agent_review_required: true
    voice_file_used: voice/diana.md
    do_not_send_yet: true                       # ← UPL-adjacent; agent verifies before send
    do_not_send_yet_reason: refusal_recommendation_pending_agent_review
    tcpa_window_check: sent_within_window: true
next_action: "UPL — legal interpretation refused; agent must review refusal language + escalate to attorney or title. Surface to Diana in next digest from 04 (deal is mid-option-period)."
```

### Diagnosis

The bad output drafts an *answer* — and the answer is partially right (financing/title contingencies do preserve earnest money refundability) and partially wrong (the phrasing "if there's a real cause" is exactly the kind of approximation that breaks under contract scrutiny). The structural failure isn't that the draft is wrong; it's that 03 produced a substantive legal answer at all. `03/rules.md` §2 names this hard refusal: *"UPL hard refuse — legal interpretation. When a client asks for interpretation of contract enforceability, termination rights, earnest-money refundability, title-dispute reading, or any 'is this legal / can I' question, do not draft a substantive answer."*

The system-level cost is concrete: every draft that ships with `agent_review_required: true` BUT contains a substantive legal answer puts the agent in the position of either (a) sending an answer she didn't write and may not stand behind, or (b) catching the UPL leak during review and rewriting from scratch. Option (b) means 03 did negative work — the agent had to draft a refusal anyway, but now from a state where the original draft has already framed the client's expectation that an answer is coming. Real Estate Domain Research §2.4 names this as the canonical malpractice exposure for non-attorneys.

The good output ships the verbatim UPL refusal language from `03/rules.md` §2 in Diana's voice — the *agent* (not the system) recommends counsel via title or attorney. The refusal preserves rapport ("good question and an important one") and offers a concrete next step ("Want me to loop in Heritage now? They can answer same-day"). `do_not_send_yet: true` is set because UPL-adjacent drafts always pass through agent review before send — even when the draft IS the prescribed refusal language, the agent confirms title vs attorney routing and timing.

The envelope back-handoffs to `00_orchestrator` with `back_compliance_block` per `03/handoff.md` §4 — `03` is named there as the **primary site** for this enum value. The orchestrator's downstream behaviour: surface to Diana (the agent on the deal) via the deal-active digest from `04_transaction_coordinator`, which is currently tracking the option period this question pertains to.

The `proposed_draft` lives inside the refusal envelope rather than being emitted as a forward-normal envelope to END — because the draft *is* the artifact of the refusal, not an independent client comm. Conceptually: 03's output here is "I refused, and here's the refusal language for Diana to send." That single envelope captures both.

---

## Pair 2 — Anonymous inbound: neutral salutation, no name extraction

### Input artifact

Inbound envelope from `00_orchestrator` → routed via `01_lead_qualifier` (first-touch follow-up for an unqualified-yet web-form lead):

```yaml
case_id: CASE-2026-0091
agent_on_deal: agent_diana
content_provenance: anonymous_inbound           # ← critical flag
case_type: residential_buyer_side
payload:
  inbound_text: |
    Hi! Saw your Westgate listing on Zillow — my name's Mia and we're
    just starting to look. Can someone reach out? Email is fine.
  subject_classification: communication_draft
  draft_request:
    purpose: first_touch_follow_up
    channel_preference: email
  qualified_lead: null                          # ← not yet qualified (web-form, no call yet)
```

### Bad output

```yaml
from: 03_client_communication
to: END
confidence: high
handoff_reason: forward_normal
payload:
  channel: email
  recipient_handle: <web_form_email_placeholder>
  subject_line: "Hi Mia — thanks for reaching out about the Westgate listing"   # ← bad
  body_draft: |
    Hi Mia,

    Thanks so much for reaching out about the Westgate listing! I'd love
    to chat about what you're looking for. Are you available for a quick
    call this week — Tuesday or Thursday work well for me. Also, do you
    have a sense of timeline and have you spoken with a lender yet?

    Best,
    Diana
  agent_review_required: true
  voice_file_used: voice/diana.md
  do_not_send_yet: false
```

### Good output

```yaml
from: 03_client_communication
to: END
confidence: med                                 # ← anonymous inbound → cap at med
handoff_reason: forward_normal
payload:
  channel: email
  recipient_handle: <web_form_email_placeholder>
  subject_line: "Thanks for reaching out about the Westgate listing"
  body_draft: |
    Thanks for reaching out about the Westgate listing — happy to help.

    Quick to get started: a few questions if you don't mind sharing —
    what's your timeline looking like, have you started the lender
    pre-approval conversation, and what range are you working in? That'll
    help me point you at the right places to look in Westgate (or
    nearby — there are a few neighbourhoods that might also fit).

    Happy to set up a quick call too if that's easier — Tue/Thu afternoons
    work for me this week.

    Best,
    Diana
  agent_review_required: true
  voice_file_used: voice/diana.md
  do_not_send_yet: false
  notes: |
    Inbound named "Mia" in text but content_provenance: anonymous_inbound
    (web-form, no identity verification yet). Salutation is neutral per
    03 Hard rule §2; first name will be used in follow-up after qualification
    call confirms identity.
```

### Diagnosis

The bad output extracts "Mia" from the inbound text and uses it in both the subject line and the salutation. That feels like the obvious move — the lead *gave* her name — but `03/rules.md` §2 names this as a Hard rule: *"No personalised salutation for anonymous inbound. When `content_provenance: anonymous_inbound`, draft uses neutral salutation ('Thanks for reaching out', 'Hi there') — never 'Hi [first name]' or any name extracted from the inbound text."*

The reasoning is in the failure-mode register (`03/identity.md` §6): *"Drafts a personalised message ('Hi Sarah') for an anonymous inbound with `content_provenance: anonymous_inbound` → spoofed/fake-sender risk; the system speaks as if it knows the client when it doesn't."* A web-form lead is not identity-verified; the name field is whatever the submitter typed. If it's a real lead, the neutral salutation costs nothing — Mia is unlikely to be offended that the first email opened with "Thanks for reaching out." If it's a spoofed submission (competitor scraping the form, bot, mistyped name), the personalised salutation makes the team look like it can be tricked. The downstream cost of always-personal is occasional embarrassment; the cost of always-neutral-until-verified is zero.

The good output also caps `confidence` at `med` rather than `high` — per `03/handoff.md` §5, `high` requires `content_provenance: verified_client`. The anonymous flag is doing exactly what it's designed to do: cascading downstream caution without blocking the draft.

The qualification questions in the body are doing double duty — they're the natural first-touch ask (timeline, pre-approval, budget = the three-question filter from `01_lead_qualifier`), but they also serve as identity-verification context. When Mia replies, `01_lead_qualifier`'s next pass will flip `content_provenance` from `anonymous_inbound → verified_client`, and the *next* draft (cadence check-in or showing prep) can open with her actual name.

---

## Pair 3 — Voice file missing: degrade-with-flag, never block (CALL-020)

### Input artifact

Inbound envelope from `04_transaction_coordinator` (weekly status update during option period):

```yaml
case_id: CASE-2026-0107
agent_on_deal: agent_jane                       # ← new agent, voice file not yet authored
content_provenance: verified_client
case_type: residential_buyer_side
payload:
  case_state:
    deal_phase: option_period_active
    option_period_end: 2026-05-23T17:00:00-05:00
    inspection_window_end: 2026-05-23
    inspector_assigned: "Frank Davis, Davis Home Inspections"
    inspection_scheduled: 2026-05-20T10:00:00-05:00
  draft_request:
    purpose: status_update
    channel_preference: email
  context_for_draft: |
    Weekly status: inspection scheduled Wed 10am with Frank Davis; option
    period ends Friday 5pm; lender (Frost Bank) on track for buyer approval
    by next Tue. No client action needed this week.
```

03 attempts to resolve `voice/agent_jane.md` — file does not exist.

### Bad output

```yaml
from: 03_client_communication
to: 00_orchestrator
back_to: 00_orchestrator
handoff_reason: back_data_missing               # ← bad: blocks instead of degrading
confidence: high                                # ← high confidence in the wrong response
payload:
  refusal_reason: voice_file_missing
  refusal_detail: |
    voice/agent_jane.md does not exist; cannot draft in Jane's voice.
    Re-route after voice file is created.
next_action: "Block status-update draft until voice file authored."
```

### Good output

```yaml
from: 03_client_communication
to: END
confidence: low                                 # ← degrade-with-flag → cap at low
handoff_reason: forward_normal
payload:
  channel: email
  recipient_handle: <client_email_on_case_record>
  subject_line: "Weekly update — option period ending Friday"
  body_draft: |
    Hi [Client],

    Quick weekly update on where we are:

    - Inspection is scheduled with Frank Davis (Davis Home Inspections)
      for Wednesday at 10am — should take 2-3 hours.
    - Option period ends Friday at 5pm local. We'll review inspection
      results Wed/Thu and decide on next steps before the deadline.
    - Lender (Frost Bank) is tracking toward buyer-approval next Tuesday;
      nothing needed from you on that side right now.

    Nothing for you to action this week — I'll touch base Wednesday after
    the inspector's done. Let me know if anything comes up.

    Best,
    Jane
  agent_review_required: true
  voice_file_used: null                         # ← explicit: no voice file used
  do_not_send_yet: true                         # ← agent verifies voice match before send
  do_not_send_yet_reason: voice_missing
  tcpa_window_check: na_email
  notes: |
    voice/agent_jane.md does not exist — drafted in baseline house tone.
    Jane reviews + adjusts to her voice before send. Surface to broker
    that Jane's voice profile is missing; should be authored before her
    next deal-active phase.
next_action: "Voice asset missing — draft is baseline house tone; agent (Jane) review must verify voice match before send. Onboarding gap: voice/agent_jane.md needs to be created."
```

### Diagnosis

The bad output blocks via back-handoff, treating voice-file-missing as data-missing-equivalent. That violates CALL-020 directly: *"When `03_client_communication` receives a draft request and the resolved voice file is missing, the specialist **degrades-with-flag rather than blocks**. Drafts in baseline house voice and emits envelope with `voice_file_used: null` + `do_not_send_yet: true` + `confidence: low`."* The architectural reason is in CALL-020's reasoning: *"Degrading-with-flag preserves throughput (Diana's team can still acknowledge the inbound) without lying about voice fidelity."*

The good output drafts the status update in a neutral baseline tone — informative, scannable, no marketing language, no agent-specific voice signatures (no Diana-style "happy to chat anytime" or whatever Jane's verbal tics will eventually be). The draft is *useable*: Jane reviews, makes 2-3 voice-adjustments, sends. The team's weekly cadence holds. The system flags the underlying onboarding gap (voice profile missing) without making it a blocker — that's the asymmetry CALL-020 calls out.

`voice_file_used: null` is explicit, not omitted. That signals to the agent (and to audit) that this draft is *known* to be baseline-voice rather than accidentally-baseline-voice. The `do_not_send_yet: true` + `do_not_send_yet_reason: voice_missing` are the safety belt — even though the draft is sendable, the agent must positively confirm voice match (or rewrite) before the send action. `confidence: low` cascades downstream — any next-step decision built on this draft factors in the voice uncertainty.

The `notes` field surfaces the operational gap to whoever reviews the deal-active digest: *Jane's voice profile is missing; should be authored before her next deal-active phase.* That's the maintenance ticket the system creates implicitly through the degrade event.

---

## Pair 4 — Outside TCPA window: queue, don't send

### Input artifact

Inbound envelope from `04_transaction_coordinator` (financing-update notification, received at 22:48 local):

```yaml
case_id: CASE-2026-0042
agent_on_deal: agent_diana
content_provenance: agent_authored
timestamp: 2026-05-21T22:48:00-05:00
case_type: residential_buyer_side
payload:
  case_state:
    deal_phase: option_period_active
    update_event: "Frost Bank confirmed buyer-approval received 22:30 local."
  draft_request:
    purpose: status_update
    channel_preference: text                    # ← TCPA-restricted
  context_for_draft: |
    Confirm to Sarah that buyer-approval came through. No urgent action,
    no T-24h pressure — just an FYI she'll want to hear.
```

### Bad output

```yaml
from: 03_client_communication
to: END
confidence: high
handoff_reason: forward_normal
payload:
  channel: text
  recipient_handle: +1-512-555-0142
  body_draft: |
    Hey Sarah — quick FYI, Frost Bank confirmed your buyer-approval
    tonight. One less thing to worry about. Talk tomorrow!
  agent_review_required: true
  voice_file_used: voice/diana.md
  do_not_send_yet: false                        # ← bad: queues for immediate send
  tcpa_window_check: sent_within_window: true   # ← bad: incorrect flag
```

### Good output

```yaml
from: 03_client_communication
to: END
confidence: med                                 # ← outside window → med, not high
handoff_reason: forward_normal
payload:
  channel: text
  recipient_handle: +1-512-555-0142
  body_draft: |
    Hey Sarah — quick FYI, Frost Bank confirmed your buyer-approval
    tonight. One less thing to worry about. Talk tomorrow!
  agent_review_required: true
  voice_file_used: voice/diana.md
  do_not_send_yet: true                         # ← queued
  do_not_send_yet_reason: outside_send_window
  tcpa_window_check: sent_within_window: false  # ← honest flag
  scheduled_for_send_window: 2026-05-22T08:00:00-05:00
  notes: |
    Drafted at 22:48 local — outside 8am-9pm TCPA window for texts.
    Queued for 08:00 next morning (no T-24h pressure; no forward_urgent).
    Agent reviews + releases at next-window. Email channel would not
    have triggered the queue (na_email), but client preference is text.
next_action: "Agent review and release at 2026-05-22 08:00 local."
```

### Diagnosis

The bad output drafts the message and immediately releases it (`do_not_send_yet: false`), marking `tcpa_window_check: sent_within_window: true` — which is a factual lie at 22:48 local. The Soft rule from `03/rules.md` §3 is unambiguous: *"Outbound outside 8am-9pm local → TCPA flag for texts. Queue with `do_not_send_yet: true` until next valid window unless `forward_urgent` overrides."* TCPA (Telephone Consumer Protection Act) restricts unsolicited commercial texts outside an 8am-9pm window in the recipient's time zone; sending a "FYI" text at 22:48 to a client who didn't explicitly ask for after-hours updates is a soft TCPA violation. Not a likely enforcement event for a single deal — but a systematic pattern across a team's deals starts to look like one.

The good output queues the draft. Two structural calls: (1) `do_not_send_yet: true` with reason `outside_send_window` flags the gate explicitly so the agent doesn't fight the system on send, and (2) `scheduled_for_send_window` names the next valid window so the agent doesn't have to compute it. `tcpa_window_check` honestly records `sent_within_window: false` — that's audit-grade truth, not a justification for sending anyway.

The escape hatch is `forward_urgent`. If `04_transaction_coordinator` had marked this `forward_urgent` (e.g., T-24h pressure on an option-period decision, where the client needs to know *tonight* to make a morning call), the TCPA queue is overridden per the same Soft rule. But this envelope is not `forward_urgent` — the `context_for_draft` explicitly says "No urgent action, no T-24h pressure — just an FYI." The Soft rule fires.

Confidence drops from `high` to `med` per `03/handoff.md` §5 — and the cap is `med` here, not `low`. CALL-021 (Locked 2026-05-12) sub-types the `do_not_send_yet: true` cap by `do_not_send_yet_reason`: content-degraded reasons (`voice_missing` / `refusal_recommendation_pending_agent_review` / `sensitive_content_pending_agent_decision`) cap at `low` because the agent may want to rewrite; timing-only gates (`outside_send_window`) cap at `med` because the draft is ready and only the send window is closed. This Pair 4 case is `outside_send_window`-only — clean draft, gated timing — so `med` is the right colour.
