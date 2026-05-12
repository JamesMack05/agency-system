# 04_transaction_coordinator — examples

Eight comparative pairs. Sized 8 (not the 3-4 default) per ADR-003 deviation: Real Estate Domain Research §2.5 catalogues 8+ distinct failure modes for the deal-active phase, and a smaller pair-budget cannot exhibit the decision boundaries readers need to internalise. Real-model voice per CALL-010.

Each pair surfaces a Texas-domain trap that a national-generic TC system would miss — calendar-day arithmetic, two distinct deposit types, dual financing deadlines, regulatory pre-conditions, UPL boundaries.

---

## Pair 1 — Option Fee and Earnest Money are two different fees

### Input artifact

Inbound from `03_client_communication` on executed contract:

```yaml
payload:
  deal_state_change: contract_executed
  executed_contract_date: 2026-05-19
  contract_form: TREC_20-15
  closing_date: 2026-06-23
  buyer_rep_agreement: {signed_date: 2026-05-15, signed_pre_showing: true}
  iabs_delivered_flag: true
  # Fees: client wired $8,850 total to Heritage Title on 2026-05-19;
  # the contract specifies a $350 option fee + $8,500 earnest money.
```

### Bad output

```yaml
payload:
  deadlines:
    - {deadline_name: "Option period end", due_at: 2026-05-26T17:00:00-05:00, status: pending}
    - {deadline_name: "Earnest money delivery", due_at: 2026-05-22, status: pending}
  documents:
    - {doc_name: "Deposit ($8,850 total)", owner: heritage_title, status: received, received_at: 2026-05-19}   # ← bad: collapsed
  notes: "Buyer wired $8,850 to Heritage Title — option fee + earnest money combined."
confidence: high
```

### Good output

```yaml
payload:
  deadlines:
    - {deadline_name: "Option period end", due_at: 2026-05-26T17:00:00-05:00, status: pending, rule: "calendar_days; no weekend/holiday extension"}
    - {deadline_name: "Earnest money delivery", due_at: 2026-05-22, status: met, rule: "3 calendar days from execution; rolls forward to next business day if weekend/holiday"}
    - {deadline_name: "Option fee delivery", due_at: 2026-05-22, status: met, rule: "3 calendar days from execution"}
  fees:
    option_fee:
      amount: 350
      delivered_to: "Heritage Title Co"
      delivered_date: 2026-05-19
      refundable: false                                # ← non-refundable always
      credit_at_closing: true                          # ← credited to sales price if deal closes
      period_end: 2026-05-26T17:00:00-05:00
    earnest_money:
      amount: 8500
      delivered_to: "Heritage Title Co (escrow agent)"
      delivered_date: 2026-05-19
      refundable: true                                 # ← refundable on legitimate termination
      becomes_damages_on_default: true                 # ← liquidated damages on buyer default
      escrow_agent: "Heritage Title Co"
  notes: |
    Buyer wired $8,850 total to Heritage Title on 2026-05-19.
    Allocated per contract: $350 Option Fee + $8,500 Earnest Money.
    Tracked as two distinct line items per TREC convention — different
    refundability rules, different downstream behaviour on termination.
confidence: high
```

### Diagnosis

This is the **catastrophic failure mode** from `04/identity.md` §6: *"Conflates Option Fee with Earnest Money — different fees, different delivery targets (both title co, but tracked separately), different refundability rules (Option Fee non-refundable always; Earnest Money refundable on legitimate termination)."* Real Estate Domain Research §3 row 4 names the two as distinct fees that national-generic systems collapse into "earnest money."

The downstream cost is concrete. If the buyer terminates during the option period (her unrestricted right), the bad output's collapsed deposit produces wrong client guidance: either the team tells her she gets the full $8,850 back (wrong — the $350 option fee is non-refundable, always), or they tell her she gets nothing back (wrong — the $8,500 earnest money is refundable on option-period termination). Either way, the next conversation requires a correction the team should not have had to make. The good output separates the two from the moment of contract execution — `option_fee.refundable: false`, `earnest_money.refundable: true` — and downstream `forward_urgent` escalations to `03` reference the right field by name.

`04/rules.md` §2 names this as a Hard rule: *"Option Fee and Earnest Money tracked separately. Two distinct fields, two distinct delivery targets (both title co, separately tracked), two distinct refundability rules. Never collapse into one 'deposit' field."* The downstream consumers (`03` drafting client-comms; agent reading digests) depend on the field-level separation, not on prose-level qualification.

---

## Pair 2 — Calendar days, not business days

### Input artifact

Executed contract 2026-05-19 with option period of 7 days. The 7th calendar day lands on Saturday 2026-05-26 (5pm local).

### Bad output

```yaml
deadlines:
  - {deadline_name: "Option period end", due_at: 2026-05-29T17:00:00-05:00, status: pending}   # ← bad: business-day extension to Tuesday
    rule: "business_days; weekend extension to Mon/Tue"
```

### Good output

```yaml
deadlines:
  - {deadline_name: "Option period end", due_at: 2026-05-26T17:00:00-05:00, status: pending}   # ← Saturday 5pm local
    rule: "calendar_days; option period does NOT extend for weekend/holiday per TREC 23-19 §5"
    escalation: "T-24h on 2026-05-25 17:00 local triggers forward_urgent to 03 (voice contact required)"
```

### Diagnosis

The bad output applies business-day arithmetic — a national-generic assumption. TREC contract deadlines run **calendar days**, with one exception: earnest-money 3-day delivery rolls forward to next business day if it lands on weekend/holiday. Option period termination at 5pm local on the last day does *not* extend. Real Estate Domain Research §3 row 5 cites Paragon Realtors on the TREC arithmetic, with the option period named explicitly.

`04/rules.md` §2 Hard rule: *"Calendar-day arithmetic, not business-day. All TREC contract deadlines run calendar days. Exception: earnest-money 3-day delivery rolls forward if it lands on weekend/holiday. Most other deadlines (option period in particular — 5pm local on the last day) do *not* extend."* The failure mode in `04/identity.md` §6 names the consequence concretely: *"option period misses by 2-3 days routinely; client may believe they have until Monday when termination right expired Saturday at 5pm local."*

The good output also pre-computes the T-24h escalation point (Friday 5pm local) and notes the `forward_urgent` to `03_client_communication` that will fire then. That's the T3 trajectory in `decisions/trajectories.md` — pre-anchored at deal initialisation so the escalation is queued, not surprised.

---

## Pair 3 — Financing tracked as two deadlines, not one

### Input artifact

TREC 40-11 Third-Party Financing Addendum attached to contract. Closing date 2026-06-23. Parties set buyer-approval contingency at 21 days from execution.

### Bad output

```yaml
deadlines:
  - {deadline_name: "Financing contingency", due_at: 2026-06-09, status: pending}   # ← bad: single deadline
    rule: "21 days from execution"
```

### Good output

```yaml
deadlines:
  - deadline_name: "Buyer-approval deadline (TREC 40-11 §B(1))"
    due_at: 2026-06-09
    rule: "Variable; set by parties at 21 days from execution. Buyer's right to terminate if not obtained."
    status: pending
  - deadline_name: "Property-approval / appraisal deadline (TREC 40-11 §B(2))"
    due_at: 2026-06-20                                                 # ← closing - 3
    rule: "Fixed at closing_date - 3 calendar days. Appraisal-contingency lapses if not exercised by this date."
    status: pending
```

### Diagnosis

TREC 40-11 has **two** financing deadlines, not one. `04/rules.md` §2 Hard rule: *"Financing tracked as two deadlines: `buyer_approval_deadline` (variable, set by parties — typical 21 days) and `property_approval_deadline` (fixed at `closing_date - 3`). Single-deadline tracking is a hard violation."* Real Estate Domain Research §3 row 6 cites the Texas REALTORS® FAQ confirming the two-deadline structure.

The failure mode in `04/identity.md` §6: *"Financing contingency tracked as single deadline — property-approval (fixed at `closing_date - 3`) gets ignored; appraisal contingency lapses unnoticed."* The two deadlines do different things: buyer-approval is about the **buyer's qualification** (income, credit, debt-to-income); property-approval is about the **appraisal** (does the property support the loan amount). A buyer who clears buyer-approval at day-21 can still lose the deal at day-property-approval if the appraisal comes in low — and the property-approval deadline is the buyer's window to terminate without losing earnest money on appraisal grounds.

Single-deadline tracking either misses the property-approval window entirely (buyer loses termination right silently) or conflates the two milestones so the agent can't tell which contingency is at risk at any given point.

---

## Pair 4 — HOA resale certificate: request at initialisation, not when needed

### Input artifact

Executed contract on a property in an HOA neighborhood (Falconhead West, Lakeway). HOA confirmed in `02_property_research` brief. Resale certificate not in the team's possession at execution.

### Bad output

```yaml
documents:
  - {doc_name: "HOA resale certificate", owner: buyer_team, status: pending}   # ← bad: not requested yet
notes: "Will request HOA resale certificate when needed for closing review."
```

### Good output

```yaml
documents:
  - doc_name: "HOA resale certificate"
    owner: hoa_management_company
    status: requested
    requested_date: 2026-05-19                                                 # ← day of execution
    expected_delivery: 2026-05-29                                              # ← §207 Property Code 10-day limit
    notes: "§207 Texas Property Code: HOA must deliver within 10 days of written request. Late delivery → buyer's right to terminate triggered."
deadlines:
  - {deadline_name: "HOA resale certificate delivery", due_at: 2026-05-29, status: pending, rule: "§207 Property Code 10-day; buyer terminate-right on late delivery"}
```

### Diagnosis

`04/rules.md` §2 Hard rule: *"HOA resale certificate requested at initialisation if HOA present (per `02_property_research` brief). §207 Property Code 10-day delivery; late request is the team's failure, not the seller's."* The failure mode in `04/identity.md` §6: *"HOA resale certificate not requested at deal-active initialisation → §207 Property Code 10-day delivery missed; buyer's right to terminate triggered by late docs."*

The trap is treating the resale certificate as a closing artifact rather than an option-period artifact. The buyer's right-to-terminate on late HOA docs is what makes the timing load-bearing: if the team requests the certificate 5 days before closing and it doesn't arrive until 2 days after, the buyer can walk with earnest money refunded — even if everything else about the deal is fine. The team owes itself a 10-day buffer; the 10-day countdown starts on *written request*, which means the team has to make the request, not wait.

The good output records the request date, the HOA's delivery deadline under §207, and the deadline that fires if late. This isn't optional record-keeping — it's the input to the agent digest that surfaces "HOA docs due Friday, no response from HOA yet" three days early so a call to the HOA management company isn't a panic.

---

## Pair 5 — Seller's Disclosure Notice §5.008: verify exemption, don't assume one applies

### Input artifact

Executed contract on a single-family resale (built 2018, owner-occupied 6 years). SDN not in case file at execution.

### Bad output

```yaml
documents:
  - {doc_name: "Seller's Disclosure Notice", owner: seller, status: not_required, notes: "Property is new construction (exempt)."}   # ← bad: wrong exemption claim
```

### Good output

```yaml
documents:
  - doc_name: "Seller's Disclosure Notice (Property Code §5.008)"
    owner: seller_team
    status: required_and_missing                                               # ← non-exempt; should have been on file pre-execution
    exemption_claim: none
    exemption_check: |
      §5.008 exemptions: new construction, foreclosure, family transfer,
      court-fiduciary, multi-unit, commercial. Subject is owner-occupied
      single-family resale (built 2018, occupied 6 years) — NONE of the
      exemptions apply.
    surface_to_agent: immediate
    notes: |
      Pre-execution requirement was not met. Surface to agent_on_deal in
      next digest: SDN required for non-exempt residential resale per
      §5.008 Property Code; absence at execution may give buyer rescission
      grounds. Request SDN from seller's agent immediately; document
      timeline.
```

### Diagnosis

The bad output claims an exemption that doesn't apply. `04/rules.md` §2 Hard rule: *"Seller's Disclosure Notice §5.008 verified on file at execution for non-exempt residential resale. Exemptions: new construction, foreclosure, family transfer, court-fiduciary, multi-unit, commercial."* The trap is treating "built 2018" as "new construction" — Real Estate Domain Research §3 row 7 names new construction as an exemption, but the exemption is for *first sale* properties, not for properties built recently and resold.

A 2018-built property occupied 6 years is a standard residential resale, fully within §5.008's coverage. Defaulting to `not_required` because the property is "newish" produces the failure mode in `04/identity.md` §6: *"Seller's Disclosure Notice §5.008 not verified on file at execution for non-exempt residential resale → exemption-class unexamined; later discovery may give buyer rescission grounds."*

The good output names the exemption check explicitly, lists all 6 exemption classes, confirms none apply, and flags the missing SDN for immediate agent attention. `status: required_and_missing` is the right colour — the field exists for this reason, and the agent digest will surface this as a red item until resolved.

---

## Pair 6 — TRELA §1101.563 buyer-rep verification: back_compliance_block

### Input artifact

Inbound envelope from `00_orchestrator` (re-routed deal-active inbound):

```yaml
payload:
  deal_state_change: contract_executed
  executed_contract_date: 2026-05-19
  contract_form: TREC_20-15
  closing_date: 2026-06-23
  buyer_rep_agreement: {signed_date: null, signed_pre_showing: false}     # ← absent
```

### Bad output

```yaml
payload:
  deadlines: [...full deadline tracking initialised...]
  notes: "Buyer-rep agreement not signed; will request signature alongside other docs."
confidence: med
handoff_reason: forward_normal                            # ← bad: proceeded
```

### Good output

```yaml
to: 00_orchestrator
back_to: 00_orchestrator
handoff_reason: back_compliance_block                     # ← canonical TRELA refusal site
confidence: high                                          # ← high confidence in the refusal
payload:
  refusal_reason: trela_1101_563_violation
  refusal_detail: |
    Incoming envelope reports executed contract on 2026-05-19. Buyer-rep
    agreement `signed_pre_showing: false`. TRELA §1101.563 (effective
    2026-01-01) requires written buyer-rep agreement signed BEFORE showing
    or offer-writing for residential property. If contract was executed
    without a pre-showing buyer-rep agreement on file, an upstream step
    violated TRELA. Deal-active phase not initialised; deadlines not
    tracked; document tracking not begun.
next_action: |
  TRELA §1101.563 violated: deal-active phase requires buyer-rep
  agreement signed pre-showing. Escalate to broker immediately; do not
  initiate deadline tracking. Possible resolutions: (a) buyer-rep was
  signed but the flag was set incorrectly upstream — verify with 01;
  (b) buyer-rep was genuinely not signed pre-showing — brokerage
  exposure, broker handles disclosure to client + parties.
```

### Diagnosis

This is the canonical `back_compliance_block` site for `04` per CALL-015 resolution and Real Estate Domain Research §3 row 1. `04/rules.md` §2 Hard rule: *"TRELA §1101.563 buyer-rep verification on receipt (per CALL-015 resolution). Verify `payload.buyer_rep_agreement.signed_pre_showing: true` before initialising deal state. If absent or false: back-handoff with `handoff_reason: back_compliance_block`."*

CALL-015's reasoning is in the design log: Rule 0 stays single-condition ("No signed contract, no deadline tracking"), but buyer-rep verification lives as a Hard rule producing `back_compliance_block`. This split keeps Rule 0 readable — Rule 0 is the contract-layer refusal (no executed contract = nothing to track); the buyer-rep check is the regulatory-layer refusal (TRELA bar). Different failure modes, different fixes upstream, different downstream behaviour.

The bad output proceeds with deadline tracking and quietly adds a "request signature alongside other docs" item. That defeats the whole point of TRELA §1101.563 — the agreement is required *pre-showing*, not pre-closing. If the showing already happened without a buyer-rep agreement, the violation is in the past, and continuing to operate the deal as if it can be cured by a late signature is a second-order compliance failure on top of the first. The broker has to know.

The good output halts at the gate, names TRELA §1101.563 explicitly, and proposes two resolution branches in `next_action` — (a) upstream data error (flag was set wrong; correctable in 01), (b) genuine violation (broker exposure; legal escalation). Either way, `04` does not initialise.

The trajectory anchor: this is the exact failure path that would have surfaced during T1's HOP 5 hand-walk in `decisions/trajectories.md` if Sarah's case had been routed without `signed_pre_showing: true` — the verification gate would fire here, before any deadline tracking began.

---

## Pair 7 — UPL on advise-on-terminating

### Input artifact

Inbound envelope from `03_client_communication` carrying a client question that bypassed `03`'s own UPL filter (or the question came in via a channel that didn't route through `03` first — e.g., agent forwarded a voicemail transcript):

```yaml
payload:
  client_question: |
    Sarah called and asked: "If I exercise the option period because of
    the foundation issue, do I get my earnest money back, or just the
    option fee back? And can the seller try to keep my earnest money
    if they say I'm not exercising in good faith?"
  case_state: { deal_phase: option_period_active, inspection_findings: "foundation $14-22k repair" }
```

### Bad output

```yaml
payload:
  answer_for_agent: |
    Yes, Sarah gets her earnest money back in full if she terminates
    during the option period — that's the whole point of the option
    period, it's an unrestricted right to terminate. The seller can't
    contest "good faith" during the option period because the option
    period IS the no-good-faith-needed window. The option fee stays
    with the seller regardless. Confidence: high.
  notes: "Drafted answer for Diana to relay to Sarah."
```

### Good output

```yaml
to: 03_client_communication
back_to: 03_client_communication                          # ← UPL refusal handled via 03's draft path
handoff_reason: back_compliance_block
confidence: high                                          # ← high confidence in the refusal
payload:
  refusal_reason: upl_termination_rights_question
  refusal_detail: |
    Question asks for interpretation of termination rights and
    earnest-money refundability — both are contract-interpretation /
    legal-advice questions. UPL: 04 does not advise on these. Surface
    options to agent via digest; 03 drafts the UPL recommendation
    language for agent to send to Sarah.
  options_for_agent_digest:
    - "Option-period termination: client has unrestricted termination right until period_end (2026-05-26 17:00 local). Title company can confirm refundability mechanics."
    - "Repair-credit ask is an alternative path — agent_on_deal can present this option without 04 advising on which to choose."
    - "Recommend looping in Heritage Title (escrow agent) for refundability mechanics + attorney for any 'good faith' interpretation question."
  context_for_03:
    purpose: refusal_with_recommendation
    upl_recommend: "title + attorney"
    decision_for_sarah: "client decides; agent presents options; 04 + 03 do not interpret"
next_action: |
  UPL — client decision question; surface options in agent digest, do
  not advise. 03 drafts UPL refusal language in Diana's voice
  (recommending Heritage Title + attorney) for Diana's review + send.
```

### Diagnosis

`04/rules.md` §2 Hard rule: *"UPL hard refuse on advice questions. 'Should I terminate?', 'Will I get earnest money back?', 'Is the seller in breach?', 'Can I get the option period extended?' — all UPL. Back-handoff to `03_client_communication` with `handoff_reason: back_compliance_block`."* The failure mode in `04/identity.md` §6: *"Advises on terminating when client asks 'should I exercise option period?' — UPL. Surface options to agent; agent advises."*

The bad output looks tempting because the *answer is mostly correct* — option-period termination does preserve earnest-money refundability, and the option fee is non-refundable. But the right answer from the wrong source is still wrong: 04 is not a licensed attorney or a title officer, and the client question explicitly asks for contract interpretation ("can the seller try to keep my earnest money if they say I'm not exercising in good faith"). That's a legal-defensibility question. Real Estate Domain Research §2.4 names this exact pattern: termination-rights interpretation is one of the canonical UPL exposures.

The good output back-handoffs to `03_client_communication` with `back_compliance_block`. `03` is the primary site for UPL refusals (per `03/handoff.md` §4) — `04` recognises the question crossed UPL territory and routes the refusal-language drafting to the specialist that owns that drafting. The `options_for_agent_digest` field surfaces what *is* in scope: the structural facts (period end date, repair-credit alternative path) that the agent can present to the client without crossing into interpretation. `context_for_03` primes 03 to draft Diana's voice for the standard UPL recommendation language.

This pair exercises the cross-specialist UPL routing: `04` catches the question, `03` produces the refusal-language draft, the agent reviews + sends. Three specialists, one closed loop, no specialist crosses into advice.

---

## Pair 8 — Homestead exemption: flag the deadline, don't file

### Input artifact

Funded deal (`deal_state_change: funded` event, post-close advisory phase):

```yaml
payload:
  deal_state_change: funded
  funding_date: 2026-06-23
  closing_disclosure_delivered_date: 2026-06-20
  client_residency_status: owner_occupied
  county: travis
```

### Bad output

```yaml
to: END
payload:
  post_close_actions:
    - {action: "File homestead exemption application with TraviCAD", owner: 04_tc, status: completed, completed_at: 2026-06-24}   # ← bad: TC filed
```

### Good output

```yaml
to: 03_client_communication                               # ← agent-facing advisory; not END yet
confidence: high
handoff_reason: forward_normal
payload:
  deal_state_change: funded
  post_close_advisory_items:
    - advisory_type: homestead_exemption
      owner_action: "Client files with TraviCAD (driver's license must reflect property address)"
      timing: "Normally Jan 1 - April 30 each year; new homeowners can apply ANY TIME after purchase"
      surface_to_client_via: agent_on_deal
      source: "TraviCAD homestead exemption page (traviscad.org/homesteadexemptions)"
      tc_action: "Flag in agent digest; agent_on_deal mentions during post-close touch; client files."
      explicit_not_tc: "TC does not file on client's behalf — administrative overstep."
  digest_summary_for_03:
    purpose: post_close_touch
    items_to_surface:
      - "Homestead exemption available for filing — agent should mention to Sarah during post-close check-in."
      - "Travis County: any time after purchase since this is a new-owner application."
      - "Driver's license update to property address is the precondition."
next_action: "03 drafts post-close touch-base in Diana's voice mentioning homestead exemption (advisory only). Forward to END after sent."
```

### Diagnosis

`04/rules.md` §3 Soft rule: *"Homestead exemption — flag, don't file. Post-close (within ~30 days of funding for Travis County), surface in agent digest. Agent recommends; client files; TC does not file on the client's behalf."* The failure mode in `04/identity.md` §6: *"Advises client to file homestead exemption (post-close) — administrative overstep. Flag the deadline for the agent to mention; the agent recommends; the client files."*

The trap is "we know how to do this; we can save the client a step." TC filing on the client's behalf crosses two lines: (1) administrative overstep — the homestead application requires the client's signature and driver's-license confirmation, which TC cannot legitimately complete; (2) liability exposure — if TC files and the filing is rejected (wrong address, wrong residency status, wrong tax year), the client lost the exemption window through the team's action. The right posture is **advisory** — surface the action item to the agent, agent surfaces to client, client takes the action.

Real Estate Domain Research §2.5 names this as a "post-close advisory" rather than a TC deliverable; row 10 of §3 confirms the Travis County mechanic (any time after purchase for new owners, normally Jan 1 - April 30 for renewals). The good output routes through `03_client_communication` for the agent-voice draft that mentions the exemption during a post-close touch — that's how the advisory reaches the client, indirectly through the team's normal comms cadence.

`to: 03_client_communication` rather than `to: END` because the advisory has to be delivered (via agent draft) before the case truly terminates. The case will terminate on `03 → END` once the post-close touch is sent.

---

## A note on the pair-count deviation

ADR-003 authorises 6-8 pairs for `04_transaction_coordinator` (vs 3-4 default for the other specialists). Eight pairs land here because Real Estate Domain Research §2.5 + §3 catalogues 8+ distinct failure modes specific to Texas residential deal-active phase — each pair above exercises a different one (fee taxonomy / day arithmetic / dual-deadline financing / HOA timeline / SDN exemption / TRELA buyer-rep / UPL termination-rights / homestead overreach). Compressing to 4 pairs would force two omissions; both omissions are domain-load-bearing.

A second specialist authorising a similar deviation would need similar domain-volume evidence per ADR-003 §examples.md §Pair-count deviation. This is not a precedent for arbitrary scaling.
