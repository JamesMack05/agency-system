# 01_lead_qualifier — examples

Four comparative pairs. Each surfaces a decision boundary where a plausible-looking qualification path produces output that downstream specialists cannot safely consume. Real-model voice per CALL-010.

---

## Pair 1 — "Stated budget" is not a budget without pre-approval

### Input artifact

Inbound envelope from `00_orchestrator`:

```yaml
payload:
  inbound_channel: text
  inbound_text: |
    Hi! Saw a listing on Westgate that we love. We're looking 500-650k,
    hoping to move sometime this summer — kids start school in August.
    When can we tour?
  sender_handle: +1-512-555-0288
  subject_classification: qualification
content_provenance: anonymous_inbound
```

01's qualification call captures: intent = buy (move-up), timeline = 30-90 days, budget range = 500-650k, **pre-approval status = none ("we haven't talked to a lender yet")**, geography = Westgate/Southwest Austin, current rep = none.

### Bad output

```yaml
from: 01_lead_qualifier
to: 02_property_research              # ← routes to showing-prep
confidence: high                      # ← bad: budget stated, ready to go
handoff_reason: forward_normal
payload:
  qualified_lead:
    intent: buy
    sub_type: move_up
    timeline: 30-90_days
    budget: {range: "500-650k", preapproval_status: none, preapproval_lender: null}
    geography: ["Westgate", "Southwest Austin"]
    current_representation_status: none
    buyer_rep_agreement_status: not_yet_signed
  preferred_channel: text
next_action: "Pull comps + neighborhood for Westgate target band 500-650k; prep showing list."
```

### Good output

```yaml
from: 01_lead_qualifier
to: 03_client_communication           # ← nurture route, not showing-prep
agent_on_deal: team_lead              # ← sentinel — no assignment-table rule matches pre-conversion (CALL-022)
confidence: low                       # ← three-question filter incomplete
handoff_reason: forward_normal
payload:
  qualified_lead:
    intent: buy
    sub_type: move_up
    timeline: 30-90_days
    budget: {range: "500-650k_unverified", preapproval_status: none, preapproval_lender: null}
    geography: ["Westgate", "Southwest Austin"]
    current_representation_status: none
    buyer_rep_agreement_status: not_yet_signed
    iabs_delivered_flag: true
  preferred_channel: text
  notes: |
    Stated 500-650k but no pre-approval — budget unverified.
    August move + school timeline = real urgency; lender intro
    is the unblocking action. Buyer-rep not signed (TRELA
    §1101.563 blocks showings + offer-writing until signed).
next_action: "First-touch follow-up + lender introduction. Do not route to 02 for showing-prep until pre-approval + buyer-rep signed."
```

### Diagnosis

The bad output reads "stated budget = budget" and routes to showing-prep. That ignores `01/rules.md` §2 Hard rule: *"Pre-approval required before any showing-prep routing. A stated budget without pre-approval routes forward as `confidence: low`."* The downstream cost is concrete: `02_property_research` pulls comps anchored to a 500-650k band, `03_client_communication` drafts a "here are some Westgate listings to tour" follow-up, the agent invests showing time — then the lender conversation discloses a 380k pre-approval ceiling, and the entire showing list and tone has to be rebuilt with the family already emotionally anchored to homes they cannot finance.

The good output uses 03 for a *different* job: first-touch follow-up + lender intro. Confidence is `low` because the three-question filter is incomplete (pre-approval missing). The TRELA §1101.563 buyer-rep status is also flagged — even with pre-approval, showings cannot start until the buyer-rep agreement is signed (per `01/identity.md` §2 and Real Estate Domain Research §3 row 1). The notes field surfaces the unblocking action (lender intro) so 03 can frame the follow-up around that, not around touring.

`payload.qualified_lead.budget.range` is suffixed `_unverified` rather than just `500-650k` — small detail, but downstream specialists reading the payload at a glance see the gap without having to read the notes field.

---

## Pair 2 — Existing representation: hard refuse, do not capture

### Input artifact

Inbound envelope from `00_orchestrator`:

```yaml
payload:
  inbound_channel: voice_transcript
  inbound_text: |
    Hey Diana, this is Mike Reyes — Tom Walsh over at Coldwell has been
    helping me look in the Tarrytown area but honestly I'd rather work
    with your team. Can we set something up to look at that listing on
    Hartford?
  sender_handle: +1-512-555-0411
  subject_classification: qualification
content_provenance: anonymous_inbound
```

01's qualification call confirms: lead is currently signed with another agent at Coldwell Banker on a buyer-rep agreement.

### Bad output

```yaml
from: 01_lead_qualifier
to: 02_property_research
confidence: high                      # ← bad: "good lead, wants to switch"
handoff_reason: forward_normal
payload:
  qualified_lead:
    intent: buy
    sub_type: move_up
    timeline: 0-30_days
    budget: {range: "900-1100k", preapproval_status: pre_approved, preapproval_lender: "Texas Capital"}
    geography: ["Tarrytown", "78703"]
    current_representation_status: working_with_other_agent   # ← captured but not acted on
    buyer_rep_agreement_status: refused
  notes: "Wants to switch from Tom Walsh @ Coldwell — eager to start."
next_action: "Research Hartford listing + Tarrytown comps for showing prep."
```

### Good output

```yaml
from: 01_lead_qualifier
to: 00_orchestrator                  # ← back-handoff, not forward
back_to: 00_orchestrator
handoff_reason: back_compliance_block
confidence: high                      # ← high confidence in the *refusal*, not the lead
payload:
  refusal_reason: existing_representation
  refusal_detail: |
    Lead confirmed mid-call he is currently signed on a buyer-rep
    agreement with Tom Walsh at Coldwell Banker. Qualification halted
    per Hard rule; lead data NOT captured (would constitute interference
    with existing representation under REALTOR® Code Article 16 +
    IABS exception 2).
  raw_inbound: |
    Hey Diana, this is Mike Reyes — Tom Walsh over at Coldwell has been
    helping me look in the Tarrytown area...
next_action: "REALTOR® Code Article 16 + IABS exception 2: cannot interfere with existing representation. Escalate to broker for confirmation; do not capture lead data; do not draft outreach."
```

### Diagnosis

The bad output captures the qualification data, flags `current_representation_status: working_with_other_agent`, and routes forward anyway. The trap is treating representation status as *just another field* — but `01/rules.md` §2 names it as a **hard refusal**: *"If the lead admits working with another agent, do not continue qualifying. Back-handoff with `back_compliance_block`."* The bad output also captures `budget.range: "900-1100k"` and `preapproval_lender: "Texas Capital"` — that data was disclosed in confidence to the qualifier, and forwarding it once the refusal trigger fired creates downstream exposure: a research brief is now anchored to a lead Diana's team cannot legally pursue, and the trail records that the brokerage knew details about a competitor's client.

The good output stops the call at the disclosure, writes no qualified_lead payload, and routes the refusal to `00_orchestrator` with `back_compliance_block`. Notice: `confidence: high` in the good output refers to the *refusal*, not the lead — the qualifier is highly confident this is a compliance block, not uncertain. The `payload.refusal_reason` and `payload.refusal_detail` give the orchestrator + broker the diagnostic they need without preserving the captured-then-untouchable lead data.

This is the canonical `back_compliance_block` site for 01: Real Estate Domain Research §2.2 names existing-representation as one of the two hard-refuse triggers (the other being already-in-negotiation). The orchestrator's downstream behaviour on receipt is documented in `00/handoff.md` §6: escalate to broker.

---

## Pair 3 — CMA request is not a qualification

### Input artifact

Inbound envelope from `00_orchestrator` (mis-routed — see `00_orchestrator/examples.md` Pair 4 for the routing-side version):

```yaml
payload:
  inbound_channel: web_form
  inbound_text: |
    Thinking about selling next year, can you tell me what
    my house at 2412 Hartford might go for in this market?
  sender_handle: form_submission_id_72814
  subject_classification: qualification     # ← orchestrator mis-classified
  inbound_property_ref: "2412 Hartford Rd"
content_provenance: anonymous_inbound
```

### Bad output

```yaml
from: 01_lead_qualifier
to: 02_property_research
confidence: med
handoff_reason: forward_normal
payload:
  qualified_lead:
    intent: sell                         # ← inferred from "thinking about selling"
    sub_type: downsize                   # ← guessed from "house" + "next year"
    timeline: 90_plus_days
    budget: {range: null, preapproval_status: na_seller, preapproval_lender: null}
    geography: ["78703 Tarrytown"]
    property_type: SFH
    current_representation_status: unknown    # ← never asked
    buyer_rep_agreement_status: na_seller
    iabs_delivered_flag: false
  preferred_channel: email                # ← guessed from web_form channel
  raw_inbound: "Thinking about selling next year..."
  notes: "Seller lead — CMA needed for listing presentation prep."
next_action: "Pull CMA on 2412 Hartford for listing presentation."
```

### Good output

```yaml
from: 01_lead_qualifier
to: 00_orchestrator
back_to: 00_orchestrator
handoff_reason: back_scope_mismatch
confidence: high                        # ← high confidence in the refusal
payload:
  refusal_reason: cma_request_misrouted_as_qualification
  refusal_detail: |
    Inbound is a CMA / valuation request ("what's my house at 2412 Hartford
    worth"). This is property research, not qualification. No qualification
    work performed; no qualified_lead payload produced.
  raw_inbound: "Thinking about selling next year, can you tell me what my house at 2412 Hartford might go for in this market?"
  suggested_route:
    to: 02_property_research
    research_request:
      subject: "2412 Hartford Rd, 78703 — CMA + neighborhood profile"
      purpose: cma_only
next_action: "CMA request — re-route to 02_property_research. Qualification can follow IF lead engages after receiving the brief."
```

### Diagnosis

The bad output fabricates a qualified_lead payload from inferences: intent inferred from "thinking about selling", sub_type guessed, preferred_channel guessed from inbound channel, representation status never even asked. Each individual inference might be plausible, but the aggregate is a confident-looking checklist built on nothing the lead actually said. Downstream `02_property_research` would consume this and produce a *seller-listing-prep* CMA — a different deliverable shape than a *neutral CMA for a prospect* (per `02/rules.md` §3: representation context drives confidence and framing).

The good output recognises the misclassification and back-handoffs via `back_scope_mismatch` per `01/rules.md` §3: *"CMA / valuation requests — not a qualification. Back-handoff to `00_orchestrator` with `back_scope_mismatch`, `next_action: 'CMA request — re-route to 02_property_research.'"* The `suggested_route` field in the refusal payload is a courtesy to `00_orchestrator` — naming the right destination so the re-route is a one-hop fix rather than a re-classification investigation.

Note the asymmetry: `01` does NOT produce a qualified_lead payload here, even a partial one. That's deliberate — capturing inferred data and forwarding it would constitute the failure mode `01/identity.md` §6 names: *"Treats a CMA / valuation request as a qualification — produces a fake qualified-lead payload from a research-class inbound."*

---

## Pair 4 — Preferred channel missing: forward gaps explicitly, don't guess

### Input artifact

Inbound envelope from `00_orchestrator`. Qualification call captured timeline + pre-approval + budget + geography in full — but the lead ended the call abruptly ("oh hold on, I have to grab the kids") before channel preference was asked. Lead has not yet signed a buyer-rep agreement.

```yaml
content_provenance: verified_client    # ← flipped during qual call before drop
payload:
  qualified_lead:
    intent: buy
    sub_type: first_time
    timeline: 30-90_days
    budget: {range: "350-425k", preapproval_status: pre_approved, preapproval_lender: "PrimeLending"}
    geography: ["Pflugerville", "Round Rock"]
    property_type: SFH
    current_representation_status: none
    buyer_rep_agreement_status: not_yet_signed
    iabs_delivered_flag: true
```

01 needs to route forward to `03_client_communication` for a first-touch acknowledgment, but `preferred_channel` is missing.

### Bad output

```yaml
from: 01_lead_qualifier
to: 03_client_communication
confidence: high                          # ← bad
handoff_reason: forward_normal
payload:
  qualified_lead: { ... }
  preferred_channel: email                # ← guessed (lead originally inbound via web_form)
  raw_inbound: "..."
next_action: "Draft first-touch follow-up email; introduce Diana, propose tour scheduling."
```

### Good output

```yaml
from: 01_lead_qualifier
to: 03_client_communication
agent_on_deal: team_lead                  # ← sentinel — first-touch follow-up, not yet agent-assigned (CALL-022)
confidence: med                           # ← gap flagged, not guessed
handoff_reason: forward_normal
payload:
  qualified_lead: { ... }
  preferred_channel: null                 # ← explicitly null; do not guess
  channel_inference_hint: "Inbound arrived via web_form; lead's prior touches in CRM (if any) may show channel preference."
  raw_inbound: "..."
  notes: |
    Qualification call ended abruptly before preferred_channel captured.
    Three-question filter complete + pre-approval verified + geography
    specific. Buyer-rep not yet signed (TRELA §1101.563 blocks showings
    until signed — surface in first-touch).
next_action: "First-touch follow-up. preferred_channel unset; if 03 cannot proceed, back-handoff with back_data_missing and 01 will re-contact lead to capture."
```

### Diagnosis

The bad output guesses `preferred_channel: email` based on the inbound channel and forwards with `confidence: high`. The trap is invisible: downstream `03_client_communication` produces an email draft, the agent sends, the lead never replies because she actually prefers text, and the team reads the silence as disinterest rather than channel mismatch.

The good output sets `preferred_channel: null` explicitly and flags the gap in `payload.notes`. Confidence drops to `med` because one filter dimension is genuinely incomplete. This is the configuration that triggers the T2 trajectory (per `decisions/trajectories.md`): if 03 cannot proceed without preferred_channel (e.g., the first-touch purpose requires a specific channel decision), 03 back-handoffs to 01 with `back_data_missing`, and 01 re-contacts the lead to fill the gap. That round-trip is *the system working as designed* — better one back-handoff than a sent message into a channel the lead doesn't watch.

Notice the `channel_inference_hint` field — it surfaces information that *could* inform a guess (web_form inbound suggests email-comfort), without forcing the qualifier to commit to that inference. 03 may use the hint in its `do_not_send_yet` reasoning ("draft email but flag for agent review re: channel"), or may back-handoff. Either way, the gap is named explicitly rather than papered over.

The buyer-rep status surfaced in `payload.notes` matters here too: 03's first-touch should include the buyer-rep agreement ask so showings can start once the lead is ready, per TRELA §1101.563. That detail propagates because the gap was made explicit rather than smoothed.
