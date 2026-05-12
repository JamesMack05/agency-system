# 02_property_research — examples

Four comparative pairs. The decision boundary in research is consistently *what counts as a sourced, framed-not-opined fact*. Real-model voice per CALL-010.

---

## Pair 1 — Every quote names its source, or it doesn't ship

### Input artifact

```yaml
payload:
  research_request:
    subject: "MLS-9182734, 2412 Hartford Rd, 78703 — comps + neighborhood"
    purpose: showing_prep
  qualified_lead:
    geography: ["78703 Tarrytown"]
    budget: {range: "750-850k", preapproval_status: pre_approved}
```

### Bad output

```yaml
from: 02_property_research
to: 03_client_communication
confidence: high                          # ← bad: no sources to back it
handoff_reason: forward_normal
payload:
  brief_subject: "MLS-9182734, 2412 Hartford Rd, 78703"
  agent_pull_quotes:
    - "Recent Tarrytown comps in the 78703 area have closed in the $790k-$865k range."
    - "Eanes ISD elementary is Casis, which is well-regarded."
    - "No HOA on this block."
    - "Flood zone is fine for this property."
  comp_count: 4
  comp_window_days: 90
  sources_named: []                       # ← empty
  representation_context: represented
```

### Good output

```yaml
from: 02_property_research
to: 03_client_communication
confidence: high
handoff_reason: forward_normal
payload:
  brief_subject: "MLS-9182734, 2412 Hartford Rd, 78703"
  agent_pull_quotes:
    - "Recent comps in Tarrytown 0.5mi: $792k (3/2/1842sqft, 47 DOM), $821k (3/2.5/1980sqft, 12 DOM), $864k (4/2/2103sqft, 31 DOM), $805k (3/2/1755sqft, 22 DOM). Past 90 days. [Sources: UnlockMLS 9165501, 9171002, 9180445, 9166890]"
    - "Eanes ISD: elementary Casis (rated 8/10), middle O. Henry, high Austin High. [Source: eanes.k12.tx.us elementary boundary lookup, austin-hs.austinisd.org attendance map]"
    - "MUD: none (austincad.org parcel 0118040502 confirms no special district). HOA: none. [Sources: austincad.org parcel 0118040502, MLS-9182734 listing detail]"
    - "Flood zone X — minimal risk; no insurance carrier flag. [Source: FEMA Map Service Center, panel 48453C0470J]"
  comp_set:
    - {address: "2401 Hartford Rd", close_date: 2026-04-23, close_price: 792000, sqft: 1842, beds: 3, baths: 2, dom: 47, source_mls_id: 9165501}
    - {address: "2418 Hartford Rd", close_date: 2026-05-02, close_price: 821000, sqft: 1980, beds: 3, baths: 2.5, dom: 12, source_mls_id: 9171002}
    - {address: "2405 Niles Rd",    close_date: 2026-04-11, close_price: 864000, sqft: 2103, beds: 4, baths: 2, dom: 31, source_mls_id: 9180445}
    - {address: "2422 Hartford Rd", close_date: 2026-03-28, close_price: 805000, sqft: 1755, beds: 3, baths: 2, dom: 22, source_mls_id: 9166890}
  comp_count: 4
  comp_window_days: 90
  sources_named:
    - "UnlockMLS listings: 9182734 (subject), 9165501, 9171002, 9180445, 9166890"
    - "austincad.org parcel 0118040502 — MUD/PID lookup"
    - "eanes.k12.tx.us elementary boundary"
    - "FEMA Map Service Center, panel 48453C0470J"
  representation_context: represented
```

### Diagnosis

The bad output reads cleanly — four declarative facts, all plausible for the Tarrytown subject. The trap is that none of them are traceable. `02/rules.md` §2 names this as a hard rule: *"Every fact in `payload.agent_pull_quotes` must trace to an entry in `payload.sources_named`. Quotes without sources are quarantined and refused."* The working-realtor practice surfaced in Real Estate Domain Research §2.3 is the same: *"I never quote a comp I haven't pulled up in MLS."*

The failure mode in `02/identity.md` §6 is concrete: agent passes a figure to the client, client asks where it came from, team has no traceable source, trust damage. The team can recover from being wrong about a comp — they can't easily recover from being unable to say which comp.

Notice the bad output's confidence: high. That's misleading regardless of whether the facts happen to be correct — confidence-high without sources is confidence-high about *the wrong thing*. The sourcing is the audit surface. `02`'s `confidence` field is calibrated against source-density + comp-count + MUD/PID/HOA resolution (per `02/rules.md` §3 + `handoff.md` §5), not against how-strongly-the-specialist-believes-the-numbers.

The good output's structure also exposes the `comp_set` as a structured array, not just prose. That's not redundant with `agent_pull_quotes` — `agent_pull_quotes` is the paste-ready form for `03_client_communication` to use in client-facing drafts; `comp_set` is the structured form for `04_transaction_coordinator` (if mid-deal recheck happens) or human review.

---

## Pair 2 — MUD/PID always resolved: "I didn't check" is a failure, not a caveat

### Input artifact

```yaml
payload:
  research_request:
    subject: "1428 Cassia Ln, 78747 (Pilot Knob area) — neighborhood profile + comps"
    purpose: offer_prep
  qualified_lead:
    geography: ["78747 Pilot Knob"]
    budget: {range: "400-450k", preapproval_status: pre_approved, preapproval_lender: "Frost Bank"}
```

### Bad output

```yaml
from: 02_property_research
to: 03_client_communication
confidence: high                          # ← bad
handoff_reason: forward_normal
payload:
  brief_subject: "1428 Cassia Ln, 78747"
  agent_pull_quotes:
    - "Comps in Pilot Knob 0.5mi past 90 days: $412k, $428k, $441k (3 closed). [Sources: MLS 9201441, 9205102, 9209887]"
    - "Del Valle ISD; new construction (~2022); HOA $45/mo per MLS listing detail."
    - "Flood zone X. [Source: FEMA panel 48453C0680K]"
  comp_count: 3
  comp_window_days: 90
  neighborhood_profile:
    mud_status: "not_specified_in_listing"     # ← bad: assumed not present
    pid_status: "not_specified_in_listing"
    hoa_status: "present, $45/mo"
  sources_named: ["MLS 9201441, 9205102, 9209887, 9182441 (subject)", "FEMA panel 48453C0680K"]
  representation_context: represented
```

### Good output

```yaml
from: 02_property_research
to: 03_client_communication
confidence: high
handoff_reason: forward_normal
payload:
  brief_subject: "1428 Cassia Ln, 78747"
  agent_pull_quotes:
    - "Comps in Pilot Knob 0.5mi past 90 days: $412k, $428k, $441k (3 closed). [Sources: MLS 9201441, 9205102, 9209887]"
    - "Del Valle ISD; new construction (~2022). [Source: MLS 9182441 listing detail]"
    - "HOA: yes, $45/mo. [Source: MLS 9182441 listing detail]"
    - "**MUD: yes — Travis County MUD #11 (Pilot Knob). 2025 tax rate 0.85/$100, assessed value $432k → ~$3,672/year MUD assessment additional to standard property tax. [Source: austincad.org parcel 0445010812, Travis County MUD #11 rate filing]**"
    - "PID: no. [Source: austincad.org parcel 0445010812 special-district lookup]"
    - "Flood zone X. [Source: FEMA panel 48453C0680K]"
  comp_count: 3
  comp_window_days: 90
  neighborhood_profile:
    mud_status: "present — Travis County MUD #11, 2025 rate 0.85/$100, est. $3,672/year"
    pid_status: "absent — confirmed via austincad.org parcel lookup"
    hoa_status: "present, $45/mo"
  sources_named:
    - "UnlockMLS: 9182441 (subject), 9201441, 9205102, 9209887"
    - "austincad.org parcel 0445010812 — MUD/PID special-district lookup"
    - "Travis County MUD #11 tax rate filing (2025)"
    - "FEMA Map Service Center, panel 48453C0680K"
  representation_context: represented
  notes: |
    MUD assessment materially affects effective monthly payment.
    On a 90% LTV $430k purchase, MUD adds ~$306/mo. Surface in
    first-touch + offer prep — buyer must understand total carry.
```

### Diagnosis

The bad output records MUD/PID status as `not_specified_in_listing`. That's a phrasing trick — it sounds rigorous ("we checked the listing") but documents the wrong source. MLS listings frequently omit MUD/PID disclosures, especially on new construction (Pilot Knob is one of the named at-risk areas in Real Estate Domain Research §2.3 / §3). The authoritative source is austincad.org via the parcel ID, not the MLS listing detail. Defaulting to `not_specified_in_listing` and forwarding is the canonical failure mode in `02/identity.md` §6: *"Reports neighborhood without resolving MUD / PID → buyer makes offer based on quoted $ taxes; closes; discovers $3,200/year MUD assessment they weren't told about."*

The financial delta is not marginal: Real Estate Domain Research §3 row 9 cites $1,250-$7,500/year on Austin MUD properties; this Pilot Knob example sits in the middle of that range at ~$3,672/year, or ~$306/month against a 90% LTV $430k purchase. On a buyer pre-approved for $450k, $306/month of unaccounted carry is 30-40% of their reserve margin. The buyer's offer + comfort with the closing payment + actual post-close cash flow are all anchored to that figure.

`02/rules.md` §2 is unambiguous: *"Pull austincad.org for any property in the Austin metro. State present-with-rate or absent-confirmed-by-parcel-lookup explicitly. Never default to 'assumed not present.'"* The good output cites the parcel ID for both directions — present (with rate + dollar estimate) for MUD, absent (with parcel-lookup confirmation) for PID. The `notes` field surfaces the offer-prep implication so 03's draft frames the MUD reality in client comms rather than burying it.

---

## Pair 3 — Price band, not single-point estimate

### Input artifact

```yaml
payload:
  research_request:
    subject: "MLS-9182734, 2412 Hartford Rd, 78703 — CMA for offer prep"
    purpose: offer_prep
  qualified_lead:
    budget: {range: "750-850k", preapproval_status: pre_approved}
  case_state:
    deal_phase: pre_offer
```

Comp set returned 4 viable comps (the same as Pair 1).

### Bad output

```yaml
from: 02_property_research
to: 03_client_communication
confidence: high
handoff_reason: forward_normal
payload:
  brief_subject: "MLS-9182734, 2412 Hartford Rd, 78703"
  cma_estimate: "$812,000"                # ← bad: single-point
  agent_pull_quotes:
    - "CMA suggests fair value of $812,000 based on 4 closed comps."
    - "Recommend offer at or slightly below ask."
  comp_set: [ ... 4 comps ... ]
  sources_named: ["UnlockMLS 9182734, 9165501, 9171002, 9180445, 9166890"]
```

### Good output

```yaml
from: 02_property_research
to: 03_client_communication
confidence: high
handoff_reason: forward_normal
payload:
  brief_subject: "MLS-9182734, 2412 Hartford Rd, 78703"
  price_band:                             # ← framing, not prediction
    low: 785000                           # ← bottom-quartile comp adjusted down
    mid: 820000                           # ← median of adjusted comps
    high: 855000                          # ← top-quartile comp adjusted up
    method: "Median of 4 closed comps (past 90 days, 0.5mi, $/sqft adjusted for condition + DOM). Band = ±$35k around mid; reflects spread observed in comp set, not predictive uncertainty."
  agent_pull_quotes:
    - "4 closed comps in Tarrytown 0.5mi past 90 days: $792k, $805k, $821k, $864k. Subject is 1,920sqft (3/2.5/well-maintained, no flag conditions). [Sources: UnlockMLS 9165501, 9166890, 9171002, 9180445]"
    - "Adjusted price band: $785k low / $820k mid / $855k high. Median DOM in comp set: 26 days."
    - "Spread is condition-driven — top comp at $864k had a recent kitchen renovation; bottom comp at $792k was a slightly smaller floor plan with original 1990s finishes."
  comp_set: [ ... 4 comps ... ]
  sources_named: ["UnlockMLS 9182734, 9165501, 9171002, 9180445, 9166890"]
  notes: |
    Listed at $829k. Band straddles list price (mid $820k, high $855k).
    Comp spread reflects condition-driven variance, not pricing strategy.
    Agent_on_deal interprets implications for offer; specialist provides
    data + framing only.
```

### Diagnosis

The bad output collapses the comp set into a single number ("$812,000") and then layers an opinion on top of it ("recommend offer at or slightly below ask"). Both moves violate the same rule from `02/rules.md` §2: *"Price band only — never single-point estimate. A CMA produces 'low / mid / high' framing the data; it does not produce a target number."* The opinion violates the adjacent hard rule: *"Hard refuse — valuation opinion. If the request asks 'should we offer below ask?' — back-handoff with `back_scope_mismatch`."*

Why this matters concretely: the market moves between brief production and offer submission. A 4-comp band of $785k / $820k / $855k absorbs ~$35k of market motion in either direction — the agent reading the band can decide where in it to position the offer based on whatever happened in the 48 hours since the brief was produced (a fresh listing, a fresh sale, a client conversation about urgency). A single-point $812,000 is a prediction; if it's wrong, the brief reads as a failed forecast and the team has to re-establish credibility before the next interaction. Bands are framing the data, points are predictions, and `02_property_research` produces framing, never predictions.

The good output preserves the agent's decision space: `price_band.method` documents *how* the band was constructed (median + spread), so the human agent reading the brief knows whether the variance is structural (condition-driven) or noise (a single-outlier comp pulling the median). The `notes` field names the agent's role explicitly ("Agent_on_deal interprets implications for offer; specialist provides data + framing only"), which keeps the handoff to `03_client_communication` clean — `03` will draft talking points around the band, not around an opinion.

---

## Pair 4 — "Is this overpriced?" — refuse cleanly

### Input artifact

Inbound envelope from `04_transaction_coordinator` (mid-deal, repair-credit ask context):

```yaml
payload:
  research_request:
    subject: "MLS-9182734 — is the seller's $829k list defensible given the foundation issue?"
    purpose: mid_deal_question
  case_state:
    inspection_findings: "Foundation: 1.5\" differential settlement east side; structural engineer recommends pier support; est. $14-22k."
    deal_phase: option_period_active
```

### Bad output

```yaml
from: 02_property_research
to: 04_transaction_coordinator
confidence: high                          # ← bad
handoff_reason: forward_normal
payload:
  brief_subject: "MLS-9182734 — pricing defensibility post-inspection"
  agent_pull_quotes:
    - "Comp band $785-855k mid $820k. List at $829k is at the upper edge of justified."
    - "Foundation repair $14-22k drops effective value to $807-815k mid. List is overpriced by ~$15-20k post-inspection."
    - "Recommend $18k repair credit ask or termination if seller refuses."
  recommendation: repair_credit_ask
```

### Good output

```yaml
from: 02_property_research
to: 00_orchestrator
back_to: 00_orchestrator
handoff_reason: back_scope_mismatch
confidence: high                          # ← high confidence in the refusal
payload:
  refusal_reason: valuation_opinion_requested
  refusal_detail: |
    Request asks whether list price is "defensible" — that is a valuation
    opinion, which is agent territory under 02's hard rule. Research can
    produce updated price band reflecting inspection findings, but the
    interpretation (defensible? overpriced? credit-ask amount?) is the
    agent's call.
  available_deliverable:
    description: |
      Updated price band incorporating foundation repair cost adjustment.
      Specialist can produce band; specialist cannot frame it as defensible-
      or-not, justified-or-not, recommend-credit-ask-or-not.
    price_band_post_inspection:
      low: 763000      # ← original low $785k minus $22k high-end repair estimate
      mid: 803000      # ← original mid $820k minus $17k mid-point repair estimate
      high: 841000     # ← original high $855k minus $14k low-end repair estimate
      method: "Original band (Pair 3) minus structural-engineer repair estimate range $14-22k."
    sources_named: ["UnlockMLS 9182734, 9165501, 9171002, 9180445, 9166890", "Structural engineer report (case file)"]
next_action: "Valuation opinion is agent territory; specialist produces price band only. Re-route to 04 with updated band; let agent_on_deal interpret defensibility + repair-credit framing."
```

### Diagnosis

The bad output reads as if it's doing the agent a favour — it has the comps, it has the repair cost, the math is straightforward. But the words "defensible," "overpriced," and "recommend" all cross from data-framing into interpretation. `02/rules.md` §2: *"Hard refuse — valuation opinion. If the request asks 'is this overpriced?' / 'is this a fair price?' / 'should we offer below ask?' — back-handoff with `back_scope_mismatch`."* The failure mode in `02/identity.md` §6 names the consequence: *"Opines on whether a listing is fairly priced — strays into agent interpretation; if agent disagrees, conflict; if agent accepts and shares, may inadvertently signal valuation opinion to client (agent territory, not specialist territory)."*

The good output threads a useful needle: refuse the opinion, but offer the deliverable that *is* in scope. The `available_deliverable.price_band_post_inspection` is what `02` can legitimately produce — the comp band shifted by the engineer's repair estimate, with the methodology stated. The orchestrator (or the agent via the orchestrator) then has the structured data needed to make the call, without `02` having made the call for them.

This is also where `back_scope_mismatch` is the right enum value rather than `back_compliance_block`. UPL is for *legal* interpretations (enforceability of contract clauses, termination rights, refundability questions) — those route via `back_compliance_block` per Real Estate Domain Research §2.3 / §3. A valuation opinion isn't a legal question; it's an agent-judgment question. The two enum values reflect different downstream behaviour: `back_scope_mismatch` re-routes within the team (agent makes the call); `back_compliance_block` typically escalates outside the system entirely (attorney, title, broker). Naming the right enum value is the point of typing the taxonomy in the first place.
