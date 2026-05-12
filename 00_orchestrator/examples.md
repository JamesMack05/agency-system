# 00_orchestrator — examples

Four comparative pairs. Each surfaces a decision boundary that the orchestrator must hold against pressure from a plausible-looking wrong answer. Pairs are real-model (envelope-vocabulary) per CALL-010 resolution.

---

## Pair 1 — Anonymous Zillow inbound: `content_provenance` is not a guess

### Input artifact

`INTAKE.md`:

```
channel: zillow_relay
from: sarah.chen.zillow.relay@zillow.example
received_at: 2026-05-12T14:32:08Z
body: |
  Hi, saw your Tarrytown listing on Zillow, can someone reach out?
  Thanks, Sarah
```

### Bad output

```yaml
schema_version: 1.1.0
case_id: CASE-2026-0042
from: 00_orchestrator
to: 01_lead_qualifier
content_provenance: verified_client     # ← bad
agent_on_deal: agent_diana              # ← bad (also)
confidence: high
next_action: "Qualify Sarah; she's an active lead."
payload:
  inbound_text: "Hi, saw your Tarrytown listing on Zillow..."
  sender_handle: sarah.chen.zillow.relay@zillow.example
  subject_classification: qualification
```

### Good output

```yaml
schema_version: 1.1.0
case_id: CASE-2026-0042
from: 00_orchestrator
to: 01_lead_qualifier
content_provenance: anonymous_inbound   # ← Zillow relay = unverified identity
agent_on_deal: null                     # ← 00 emits null because 01 is the producing site (per CALL-022); 01 sets this on case_type resolution
confidence: med                         # ← named sender but no case match
next_action: "Qualify lead — capture intent, timeline, budget+pre-approval, geography, current rep status, buyer-rep signed status. Identity verification during qual call."
payload:
  inbound_channel: zillow_relay
  inbound_text: "Hi, saw your Tarrytown listing on Zillow..."
  sender_handle: sarah.chen.zillow.relay@zillow.example
  subject_classification: qualification
  inbound_property_ref: MLS-9182734
  urgency: normal
```

### Diagnosis

The name "Sarah" appears in the text but the inbound arrived via Zillow relay — the orchestrator has no way to verify that the human typing the message is who she says she is. `content_provenance: anonymous_inbound` is a contract claim about what the system *knows*, not what the inbound *says*. Flipping it to `verified_client` would silently downgrade `03_client_communication`'s salutation guard (Rule 03/§2 Hard rule: anonymous inbound → neutral salutation only) and the system would draft "Hi Sarah" for a spoofable identity. Identity verification is `01_lead_qualifier`'s job during the qual call; the orchestrator's job is to flag the gap, not close it.

`agent_on_deal` is null at this stage for the same reason — Diana doesn't pick up a downsizer assignment until 01 confirms the lead is real and qualified. Setting it preemptively forces `03_client_communication` to draft in a specific voice for an unverified inbound.

---

## Pair 2 — Returning client: reuse `case_id`, don't orphan the existing case

### Input artifact

`INTAKE.md`:

```
channel: text
from: +1-512-555-0142
received_at: 2026-05-19T09:14:00Z
body: |
  Hey, quick one — is the inspector still good for tomorrow at 10?
```

Plus: open case `CASE-2026-0042` exists with `payload.qualified_lead.sender_handle: +1-512-555-0142` and `deal_state: option_period_active`.

### Bad output

```yaml
schema_version: 1.1.0
case_id: CASE-2026-0089              # ← bad: fresh ID
from: 00_orchestrator
to: 01_lead_qualifier                # ← bad: routes new-lead path
parent_envelope_id: null             # ← bad: orphans CASE-2026-0042's chain
content_provenance: anonymous_inbound
payload:
  subject_classification: qualification
  inbound_text: "Hey, quick one — is the inspector still good for tomorrow at 10?"
```

### Good output

```yaml
schema_version: 1.1.0
case_id: CASE-2026-0042              # ← reuse, matched on sender_handle
from: 00_orchestrator
to: 04_transaction_coordinator       # ← deal-active phase; TC owns inspector
parent_envelope_id: ENV-2026-0042-004    # ← prior envelope in case
content_provenance: verified_client  # ← number matches open case
confidence: high
agent_on_deal: agent_diana           # ← carried through from case state
payload:
  inbound_channel: text
  inbound_text: "Hey, quick one — is the inspector still good for tomorrow at 10?"
  sender_handle: +1-512-555-0142
  subject_classification: deal_active
  urgency: normal
next_action: "Confirm inspector schedule per case state; route response via 03 if status update needed."
```

### Diagnosis

The number matches an open case — that resolves three things at once: `case_id` (reuse), `content_provenance` (`verified_client`), and `agent_on_deal` (carried through). Generating `CASE-2026-0089` does the opposite of all three: it splits the deal trail across two records, the CAS chain (`parent_envelope_id`) no longer reaches the original, and `04_transaction_coordinator` would receive a fresh envelope with no prior state to update — silent noop or a back-handoff with no useful diagnostic.

Subject classification follows from case state, not from the inbound text alone: "is the inspector still good" looks like a qualification question if you read it cold, but the matched-case context shows this is mid-deal — deal_active routes to `04`. The orchestrator never produces domain content, but it *does* produce the routing decision, and routing-from-context is the whole job.

---

## Pair 3 — TC-class inbound, no resolved `case_id`: escalate, don't fabricate

### Input artifact

`INTAKE.md`:

```
channel: voice_transcript
from: unknown_caller_id
received_at: 2026-05-20T11:02:00Z
body: |
  Hi, this is Marcus over at Heritage Title — just wanted to flag that
  the closing on the Tarrytown deal is going to push to Friday because
  the lender re-issued the CD this morning. Call me back at 512-555-0900.
```

No case currently in deal-active phase matches "Tarrytown" + Heritage Title — there are three open Tarrytown buyer-rep cases and one uses a different title company.

### Bad output

```yaml
schema_version: 1.1.0
case_id: CASE-2026-0042                # ← bad: guessed match (one of three Tarrytown cases)
from: 00_orchestrator
to: 04_transaction_coordinator
content_provenance: agent_authored     # ← bad: caller is not the team
confidence: med                        # ← bad: confidence should be low, but the real fix is "don't guess"
payload:
  subject_classification: deal_active
  inbound_text: "Hi, this is Marcus over at Heritage Title..."
  inbound_property_ref: "Tarrytown (best-guess: CASE-2026-0042)"
next_action: "Push CASE-2026-0042 closing to Friday; update lender re-disclosure timeline."
```

### Good output

```yaml
# No envelope emitted. Escalation log entry:
event: routing_escalation
timestamp: 2026-05-20T11:02:30Z
reason: ambiguous_case_match
detail: |
  TC-class inbound from Heritage Title naming "Tarrytown deal" + CD re-issue.
  Three open Tarrytown buyer-rep cases (CASE-2026-0042 / 0058 / 0061);
  only one (CASE-2026-0042) uses Heritage Title — but Marcus did not name
  the buyer or the address. Cannot route to 04 without resolved case_id.
escalate_to: human (agent_diana — owns 0042; brokerage admin — owns 0058 + 0061)
suggested_resolution: |
  Diana or admin to confirm which deal Marcus is referring to before any
  envelope is emitted. Do not push deadlines on a guessed match.
```

### Diagnosis

The inbound looks routable — TC-class subject, named title company, clear deal-state event. But the orchestrator's `case_id` resolution depends on a unique match, and three Tarrytown cases are open. Guessing one and routing produces a confident-looking envelope built on a fabricated foundation: if the wrong case is picked, `04_transaction_coordinator` updates the wrong deal's deadlines, the wrong lender's re-disclosure timeline, the wrong client's closing. That's a silent-corruption failure mode — the system reports success while degrading state.

Escalation is the correct refusal here. Rules.md §2: "Escalate to human when subject classification is ambiguous (more than one plausible specialist with no tie-breaker) — back-handoff is for downstream confusion, escalation is for upstream confusion. Do not guess." The ambiguity is *which case*, not *which specialist*, but the rule generalises: when the orchestrator can't disambiguate, the right answer is to not produce an envelope.

`content_provenance: agent_authored` in the bad output is also wrong — Marcus is the title company's representative, not Diana's team. There's no enum value for "verified third party in the deal chain" because the system hasn't needed one yet (revisit trigger candidate, but not for this case).

---

## Pair 4 — CMA looks like qualification, but it's not

### Input artifact

`INTAKE.md`:

```
channel: web_form
from: form_submission_id_72814
received_at: 2026-05-13T10:08:00Z
fields:
  name: Tom Reyes
  address: 2412 Hartford Rd, Austin TX 78703
  message: |
    Thinking about selling next year, can you tell me what
    my house at 2412 Hartford might go for in this market?
```

### Bad output

```yaml
schema_version: 1.1.0
case_id: CASE-2026-0091
from: 00_orchestrator
to: 01_lead_qualifier                # ← bad: looks like a seller lead
content_provenance: anonymous_inbound
confidence: med
payload:
  subject_classification: qualification
  inbound_text: "Thinking about selling next year, can you tell me what my house at 2412 Hartford might go for in this market?"
  sender_handle: form_submission_id_72814
  inbound_property_ref: "2412 Hartford Rd"
next_action: "Qualify seller lead — intent, timeline, motivation."
```

### Good output

```yaml
schema_version: 1.1.0
case_id: CASE-2026-0091
from: 00_orchestrator
to: 02_property_research              # ← CMA request is research, not qualification
content_provenance: anonymous_inbound
confidence: med
payload:
  subject_classification: property_research
  inbound_text: "Thinking about selling next year, can you tell me what my house at 2412 Hartford might go for in this market?"
  sender_handle: form_submission_id_72814
  inbound_property_ref: "2412 Hartford Rd, Austin TX 78703"
  research_request:
    subject: "2412 Hartford Rd, 78703 — CMA + neighborhood profile"
    purpose: cma_only
  urgency: normal
next_action: "Produce CMA on 2412 Hartford (3-5 closed comps, 0.5mi, 90d). Neutral context (no representation yet)."
```

### Diagnosis

The inbound has qualification-shaped surface signals — stated intent ("thinking about selling"), stated timeline ("next year") — but the operative ask is a property valuation. If routed to `01_lead_qualifier`, the qualifier's Rule 0 holds (raw_inbound present) but its Soft rule fires immediately: *"CMA / valuation requests — not a qualification. Back-handoff to `00_orchestrator` with `back_scope_mismatch`."* The bad route wastes a hop and surfaces the misroute to the team as a system error rather than a routing error.

The right read is: this is a research request first; if `02_property_research` produces a brief and the seller later asks to engage, *then* a qualification follows. Routing taxonomy: CMA = `02`, even when wrapped in seller-intent language. Tie-breaker rule from `00/rules.md` §3 — *"prefer the most specific match"* — picks `02` because the specific ask is a price band, and `02`'s Rule 0 (specific subject named) is satisfied by `2412 Hartford Rd`.

`representation_context` will be set to `neutral` by `02` when it produces the brief (per `02/rules.md` §3), reflecting that the team has no listing or buyer-rep on the property yet. Confidence stays `med` at the orchestrator layer: sender named but no case match — the calibration is unchanged by the route correction.
