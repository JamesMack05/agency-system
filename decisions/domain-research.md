# Real Estate Domain Research (Austin TX residential)

**Purpose:** seed the 5 × identity.md + 5 × rules.md drafts and prime `decisions/assumptions.md`. **Not architecture.** All architecture calls happen separately.

---

## §1 Bottom line / TL;DR

What this research **changes** for the architecture work:

1. **Written buyer-rep agreement is now a hard pre-condition for showing/offer-writing in Texas (TRELA §1101.563, effective 2026-01-01).** This is brand new for Comp 4 — older Austin Johnson reference may not reflect it. It belongs explicitly in `01_lead_qualifier`'s required-out payload and in `04_transaction_coordinator`'s deal-active checklist. Without it, the agent cannot legally tour the property or present an offer for the buyer. [TREC SB 1968 article](https://www.trec.texas.gov/article/what-changes-2026-about-buyertenant-representation-texas).
2. **Texas has *two* fees not one** — Option Fee (3-day delivery to title co, non-refundable, buys unrestricted right to terminate during the option period) and Earnest Money (3-day delivery to escrow agent, refundable on legitimate termination, sets default damages). National-generic systems collapse them into "earnest money." A specialist that confuses the two is wrong by design. [TREC option period article](https://www.trec.texas.gov/we-are-selling-our-house-and-buyer-never-paid-option-fee-what-happens-now), [TREC earnest money timeline](https://www.trec.texas.gov/how-long-does-agent-have-deposit-earnest-money-once-binding-contract-has-been-negotiated).
3. **Two financing deadlines in TREC 40-11, not one.** Buyer-approval contingency: variable, set by parties. Property-approval (appraisal): fixed at **3 days before Closing**. `04_transaction_coordinator` must track them separately. [Texas REALTORS® financing-addendum FAQ](https://www.texasrealestate.com/members/legal-and-ethics/resources/legal-faq/trec-third-party-financing-addendum/).
4. **TREC contracts run on calendar days, not business days, and weekends/holidays do *not* extend most internal deadlines** (option period in particular — only earnest-money 3-day delivery rolls forward to next business day). National TC checklists assume business days; that assumption is wrong here. [Paragon Realtors article](https://paragonrealtors.com/blog/posts/2025/11/03/technically-speaking-termination-option-period/).
5. **Unauthorized Practice of Law (UPL) is the canonical refusal trigger for `03_client_communication`.** Drafting contract addenda, opining on enforceability, advising on title disputes — all UPL. Article 13 of the REALTOR® Code of Ethics requires *recommending counsel*, not providing it. This becomes the verbatim refusal language for the drafting specialist. [NAR UPL article](https://www.nar.realtor/magazine/real-estate-news/law-and-ethics/what-constitutes-the-unauthorized-practice-of-law).
6. **Intermediary status is a Texas anomaly worth flagging in the envelope.** Austin Johnson already includes `intermediary_status` as a top-level envelope field — that wasn't a stylistic choice, it's load-bearing. In TX a broker can act as intermediary between the firm's own buyer and seller *only* with written consent listing prohibited conduct in conspicuous bold/underlined print. [Texas REALTORS® intermediary FAQ](https://www.texasrealestate.com/members/legal-and-ethics/resources/legal-faq/intermediary/).

What this research **confirms** (already assumed correctly by upstream calibration):

- The 5-specialist breakdown maps cleanly to real workflow stages — first-contact → research → drafting → deal-active. Boutique teams in fact do split roles this way at ~4-person scale (lead-handler + buyer-agent + listing-agent + transaction-coordinator is industry standard). [Tom Ferry team-build guide](https://www.tomferry.com/blog/real-estate-team-hiring-employees/).
- Confidence-driven downstream behaviour (`02_property_research` blocks `03_client_communication` on empty sources) maps to real fiduciary-duty + UPL constraints. Not over-engineering.
- Sequential rather than parallel handoff is correct at 4-person scale (Austin Johnson's argument). Confirmed independently — boutique team workflow guides describe sequential handoff with one TC coordinating the deal-active phase. [Dotloop teams guide](https://www.dotloop.com/blog/real-estate-teams-ultimate-guide/).
- Diana's "doesn't want software, wants a system she can teach in a week" framing matches how small Austin teams operate in practice. The CRM-tool research shows most boutique teams resist platform consolidation — they want narrow tools (Follow Up Boss, dotloop) plus operational checklists. [ihomefinder team CRM article](https://www.ihomefinder.com/blog/uncategorized/best-real-estate-crm-for-teams/).

**Two things the research could not resolve, parked for the assumption ledger:** (i) does Diana's team use TREC forms exclusively or also Texas REALTORS® TXR-versions (TXR-1601 etc.); (ii) which title company is the default escrow agent — both affect handoff details but neither blocks identity/rules drafting.

---

## §2 Per-specialist findings

### 2.1 `00_orchestrator/` — router

**Real-world workflow.** In a 4-person boutique team, the human equivalent is the team lead or a designated lead-handler triaging inbound. They distinguish: (a) new lead vs. existing client; (b) buyer-side vs. seller-side; (c) general inquiry vs. property-specific vs. deal-active. Most inbound in a 60-80 transaction/year team is repeat-business + referral, not cold lead-gen, so the router's primary job is **routing returning clients to whichever specialist owns their current deal-state**, not cold qualification.

**Core fields captured at this stage.** Inbound channel (email/text/voice/web-form), sender identity (name + best contact handle if known), subject classification (qualification / property-research / communication-draft / deal-active), existing case_id if known, urgency flag. Crucially: did this come from a *current client* or an *anonymous inbound* (e.g., Zillow lead) — drives `content_provenance` downstream.

**Standard deliverables / handoff content.** Routing decision (which specialist receives the envelope), case_id (new if none exists, existing if recognised), content_provenance enum, urgency tag. The orchestrator does *not* produce domain content — it produces routing metadata.

**Common failure modes / refusal triggers.**
- **Inbound with no identifiable sender** → cannot route a buyer-rep flow without a counterparty. Refuse: "no identified sender, no routing." Escalate to human.
- **Inbound that looks like a transaction-coordinator message but no case_id exists** → ambiguous: either it's a new deal that should have been pre-flagged, or it's misrouted. Refuse with diagnostic.
- **Inbound that names a property the team has no listing/buyer-rep agreement for** → cannot route to property research without representation context.

**Domain assumptions baked in.** Assumes all inbound is for *residential, in-Texas* transactions. A commercial inquiry or an out-of-state buyer would need a different routing taxonomy. Diana's brief is residential-Austin so this is fine; flag for re-locale onboarding.

**Evidence gap.** No public source describes how AI-orchestrators specifically handle boutique-team triage. The patterns above are inferred from team-structure guides + Austin Johnson's repo pattern. Working knowledge, not industry-cited.

### 2.2 `01_lead_qualifier/` — first contact

**Real-world workflow.** Standard practice: **within the first two minutes of any first call/text, get an answer on three dimensions — timeline, pre-approval status, budget**. If a lead cannot answer those three, they are not qualified yet; they go on a nurture cadence, not into active showing pipeline. The qualifier also surfaces *intent* (first-time buyer vs. relocate vs. investor vs. downsizer) and *motivation* (job change / family change / lifestyle / pure investment) because these change how showings get sequenced. [Freedom Trail Realty article](https://www.bostonrealestateclass.com/posts/the-new-real-estate-agents-guide-to-qualifying-your-leads/), [NextPhone lead qualification article](https://www.getnextphone.com/blog/real-estate-lead-qualification).

**Core fields captured at this stage.**
- **Intent:** buy / sell / both / unsure. Sub-type: first-time / move-up / downsize / relocate / investor.
- **Timeline:** 0-30 days / 30-90 days / 90+ days / exploratory.
- **Budget:** stated range AND pre-approval status (none / pre-qualified / pre-approved / cash). Pre-approval is the #1 filter — a stated budget without it is not a budget.
- **Geography:** Austin metro sub-area preferences (downtown / east / south / westlake / round rock / cedar park / etc.), school district priorities if any.
- **Property type:** SFH / condo / townhome / new-construction.
- **Representation status:** are they already working with another agent? (TRELA bars contact if yes — IABS exception #2.)
- **Buyer-representation agreement signed?** Post 2026-01-01, this is now required *before* showing or offer-writing. If not signed, the lead can still be qualified, but the specialist must flag that showing/offers are blocked pending signature.

**Standard deliverables / handoff content.** Qualified-lead payload mirroring Austin Johnson's 8-field shape but with TRELA 2026 additions: `intent`, `timeline`, `budget_with_preapproval_status`, `geography`, `property_type`, `current_representation_status`, `buyer_rep_agreement_status`, `iabs_delivered_flag`, plus `confidence` (low/med/high), plus the seed `raw_inbound` quote for provenance.

**Common failure modes / refusal triggers.**
- **No raw_inbound text** → Voiceprint-style Rule 0: "no inbound, no qualification." Refuse.
- **Lead admits they're working with another agent** → IABS exception 2 + Article 16 (no interference with existing relationships). Hard refuse, do not capture.
- **Lead refuses to discuss pre-approval** → not a refusal, but a confidence-low flag. Downstream property research blocks showing requests until pre-approval clarified.
- **Lead asks the qualifier to "tell me what my house is worth"** → CMA request, not a qualification. Back-handoff to orchestrator → property research.
- **Lead in negotiation phase** (e.g., "I have an offer in, can you advise") → this is deal-active. Hard refuse + escalate, never re-qualify someone already under contract.

**Domain assumptions baked in.** Assumes intent maps cleanly to one of buy/sell/both. Investor inquiries (BRRRR, 1031 exchange) need different qualification dimensions (rental yield, depreciation strategy, etc.) — Diana's brief is "mostly residential" so investors are edge cases, but if 10%+ of her volume is investor, the field set is too narrow. **Flag.**

**Evidence gap.** The "two-minute three-question filter" is industry folk-wisdom found across multiple lead-qual guides; no single TX-specific authoritative source. Working knowledge.

### 2.3 `02_property_research/` — deep research

**Real-world workflow.** Two distinct sub-jobs that look similar but aren't: (a) **CMA / pricing research** for listing presentation or buyer offer prep — find 3-5 closed comps in past 90 days within 0.5-1 mile, similar sqft/beds/baths/condition, adjust for differences, produce a price band; (b) **neighborhood research** — schools (Eanes ISD / Round Rock ISD / Austin ISD), MUD/PID status (austincad.org lookup), HOA presence and fees, commute, amenities. Both can feed showing prep but require different data and different deliverable shape. [HAR Instant CMA](https://cms.har.com/instantcma/), [Neuhaus Realty MUD/PID guide](https://neuhausre.com/guides/mud-pid-special-districts-guide-austin/).

**Core fields captured / produced.**
- For each comp: address, close date, close price, sqft, beds/baths, lot size, year built, days on market, condition adjustment, source (MLS / public records).
- For neighborhood: ISD + specific elementary/middle/high, MUD presence + tax rate from austincad.org, PID presence + assessment, HOA (yes/no, fee/cadence), flood zone, age of construction, walk score / school score.
- **Source citations are mandatory.** Austin Johnson already enforces `sources_named` — this maps directly to the working-realtor practice of "I never quote a comp I haven't pulled up in MLS."

**Standard deliverables / handoff content.** `brief_subject` (the specific property or neighborhood), `agent_pull_quotes` (3-5 facts the agent can use verbatim in client comms), `sources_named` (MLS listing IDs, austincad.org URL, TraviCAD URL, ISD pages, etc.) — Austin's pattern is correct and matches practice. Plus `confidence: low/med/high` driven by **how many comps were found** and **how recent**.

**Common failure modes / refusal triggers.**
- **Subject too broad** ("research Austin housing market") → back-handoff to orchestrator. Rule 0: "no specific subject, no brief."
- **Subject is a property the team has no listing or buyer-rep on** → can produce neutral research, cannot produce client-facing CMA framing. Confidence flag.
- **Fewer than 3 viable comps in 90 days** → confidence: low, must flag.
- **MUD/PID present but tax impact not pulled** → blocking. Austin MUD taxes add $1,250-$7,500/year (4-7% of effective payment for $450k home). A research brief that omits MUD is materially misleading. [CLR Sales Group MUD article](https://www.theclrsalesgroup.com/blog/2026/5/6/mud-taxes-in-austin-texas-the-hidden-cost-every-new-construction-buyer-needs-to-know).
- **Asked for valuation opinion** ("is this overpriced?") → soft refuse. CMAs frame data, agents (humans) interpret. Specialist can produce a price band, not an opinion.

**Domain assumptions baked in.**
- Assumes MLS access (UnlockMLS for Austin) exists and is the authoritative source. A team without MLS would need different sourcing.
- Assumes Travis County. Hays County and Williamson County have their own appraisal districts (hayscad.com, wcad.org) — same shape, different URLs. Re-locale flag.
- Assumes residential. Commercial CMA uses cap rate not comp adjustment — completely different methodology.
- Assumes the team uses TraviCAD for tax research. If Diana uses a paid tool (CoreLogic, MLS-integrated CMA) the source-naming convention shifts.

**Evidence gap.** No public source quantifies how a 4-person team divides CMA vs. neighborhood research between specialists vs. TC. Working knowledge: CMA usually lives with the listing/buyer agent; neighborhood research often the TC or a marketing coordinator. Diana's brief collapses these into one specialist — defensible but worth flagging.

### 2.4 `03_client_communication/` — drafts emails, texts, follow-ups in agent's voice

**Real-world workflow.** Two-mode operation: (a) **proactive cadence** — repeat every 3 days during "no news" periods of an active deal to prevent cold-feet; thank-you within 24h of a showing; weekly status during the option period and deal-active phase; (b) **reactive drafting** — response to a client question, draft for the agent to review and send. **Texting wins by volume** — 62% of buyers prefer it (NAR), 98% open rate vs. 20-25% for email. [Hometrack templates article](https://www.hometrack.net/100--real-estate-text-message-templates-that-actually-convert), [Matterport real estate communication](https://matterport.com/blog/real-estate-communication).

**Core fields captured / produced.**
- Inbound: `client_handle`, `inbound_channel` (text/email/voice transcript), `inbound_text`, `case_id`, `deal_state` (qualifying / shopping / under-contract / option-period / closing / post-close).
- Outbound draft: `channel` (text/email), `recipient_handle`, `subject_line` (email only), `body_draft`, `agent_review_required: true`, `voice_file_used` (which voice profile), `do_not_send_yet_flag` if voice file missing or content sensitive.

**Standard deliverables / handoff content.** Drafted message + flag indicating whether agent review is mandatory before send. **Never sends autonomously.** Austin Johnson's pattern: specialist never blocks — drafts in house baseline if voice file missing and flags `do_not_send_yet: true`. Correct pattern, keep it.

**Common failure modes / refusal triggers.**
- **Agent_on_deal field empty** → "no agent_on_deal, no draft" (verbatim Rule 0 candidate). Refuse.
- **Client asks for legal interpretation** ("can I terminate without losing earnest money?") → **UPL refusal.** Verbatim: *"That's a question for your attorney or our title company — I can flag it for [Agent], but I'm not licensed to interpret contract clauses for you."* Article 13 of the REALTOR® Code of Ethics. [NAR UPL article](https://www.nar.realtor/magazine/real-estate-news/law-and-ethics/what-constitutes-the-unauthorized-practice-of-law).
- **Client asks for opinion on enforceability** (Texas-specific: option period expired, can seller still terminate?) → same UPL refusal.
- **Asked to draft a contract addendum** → UPL hard refuse. Real estate professionals may insert factual data (party names, prices, dates) into attorney-approved blank forms but cannot draft substantive contract changes. [Avenue Legal Group UPL article](https://avenuelegalgroup.com/re-agents-unauthorized-practice-of-law/).
- **Outbound time-of-day outside 8am-9pm** → TCPA constraint on texts. Soft flag, draft for queue not immediate send.
- **Voice file missing** → degrade-with-flag, never block (steal from Austin's pattern). Draft in baseline voice + `do_not_send_yet: true`.
- **Client confidence-low on identity** (anonymous inbound from Zillow not yet matched to a verified client record) → quarantine. Do not draft a "Hi [first name]" personal message to an unverified identity.

**Domain assumptions baked in.**
- Assumes client preferred-channel is captured upstream (text vs. email vs. call). If missing, defaults to client's last-used channel.
- Assumes the agent reviews every draft before send. **This is the differentiator from "send-it-yourself" autoresponders** and is what makes UPL refusal defensible — the specialist drafts, the agent sends.
- Assumes voice file lives at `voice/<agent>.md` per Austin's pattern. Git-ignored. Per-agent.

**Evidence gap.** Cadence numbers (every-3-days) are folk-wisdom from one mortgage-broker blog; not industry-cited. Could be Diana-team-specific.

### 2.5 `04_transaction_coordinator/` — deal-active phase

**Real-world workflow.** The TC owns everything from "contract executed" through "post-close follow-up," typically 10-15 hours per transaction. Texas residential timelines run **30-45 days from executed contract to funding** (cash deals faster). The TC's primary deliverable is **deadline tracking** against the executed contract: option period, earnest money delivery, financing contingency (buyer approval *and* property appraisal as separate deadlines), inspection, repair amendment, survey, HOA docs, title commitment review, lender milestones, walk-through, closing disclosure delivery (federal CFPB 3-business-day rule), funding. [Paperless Pipeline TC checklist](https://www.paperlesspipeline.com/blog/real-estate-transaction-coordinator-checklist), [Daughtrey Law Texas deadlines article](https://daughtreylaw.com/2024/12/23/essential-real-estate-deadlines-for-texas-success/), [Be Happy TC closing process](https://www.behappytc.com/blog/texas-real-estate-closing-process).

**Core fields captured / produced.**
- `executed_contract_date` (drives every downstream deadline).
- `option_fee_amount`, `option_fee_delivered_date`, `option_fee_delivered_to` (title co per TREC), `option_period_end` (= executed_date + N days, *calendar days*, weekends *do not* extend).
- `earnest_money_amount`, `earnest_money_delivered_date`, `earnest_money_escrow_agent` (title co).
- `financing_type` (cash / conventional / FHA / VA / jumbo), `buyer_approval_deadline` (variable, from TREC 40-11 §B(1)), `property_approval_deadline` (fixed: closing_date - 3 days).
- `inspection_window_end` (typically within option period or first 7-10 days).
- `closing_date`, `funding_date` (Texas: same as closing for cash, day-of or next day for financed).
- `title_company`, `lender`, `inspector`, `appraiser` (vendor handles).
- `deadlines[]` — array of {deadline_name, due_at, status, blocker_if_any}.
- `documents[]` — array of {doc_name, owner, status, received_at}.

**Standard deliverables / handoff content.** Daily/weekly deal-status digest to agent. Urgent back-handoff to `03_client_communication` at T-24h to any contingency deadline (Austin's pattern: must include voice contact, not only text). Back-handoff to orchestrator on deal-state-change events.

**Common failure modes / refusal triggers.**
- **Contract not yet executed** → "no signed contract, no deadline tracking" (Rule 0 candidate). Refuse.
- **Option period and earnest money confused** → catastrophic. Verbatim rule: *Option Fee buys the right to terminate during the option period — non-refundable, never returned to buyer, credited to sales price at closing if deal closes. Earnest Money is a good-faith deposit — refundable on legitimate termination per TREC contract, becomes liquidated damages on buyer default.* Two different fees, two different rules, two different delivery targets (title co for both, but tracked separately). [Paragon option period article](https://paragonrealtors.com/blog/posts/2025/11/03/technically-speaking-termination-option-period/).
- **Day-count math using business days instead of calendar days** → Texas TREC contracts use calendar days. Exception: earnest money 3-day delivery rolls forward if it lands on weekend/holiday. Most other deadlines (option period in particular) do not. [TREC option period guidance](https://www.trec.texas.gov/we-are-selling-our-house-and-buyer-never-paid-option-fee-what-happens-now).
- **Financing contingency tracked as single deadline** → wrong. Buyer-approval (variable, set by parties) and property-approval (closing_date - 3 days, fixed) are separate.
- **HOA documents not requested early** → HOA resale certificate has its own timeline (typically 10 days in Texas under §207 of Property Code). Late request → late delivery → buyer's right to terminate.
- **Seller's Disclosure Notice not on file for non-exempt residential resale** → required pre-execution under §5.008 Property Code. Exceptions: new construction, foreclosure, family transfers, court-appointed fiduciaries (estate sales). [Texas REALTORS® disclosure chart](https://www.texasrealestate.com/wp-content/uploads/disclosurecharts.pdf).
- **Asked to advise on terminating** → UPL. Hard refuse, escalate to attorney/agent/title.
- **Asked about homestead exemption** → not UPL but post-close advisory. TC can flag the deadline (Travis County: any time after purchase if owner-occupied as of Jan 1) but cannot file. [TraviCAD homestead exemptions](https://traviscad.org/homesteadexemptions/), [Moreland 2026 filing guide](https://moreland.com/blog/2026-guide-to-filing-your-homestead-exemption-in-travis-county).

**Domain assumptions baked in.**
- Assumes TREC One-to-Four Family Residential Resale (TREC 20-15 or current version) is the contract form. New construction uses TREC 23, condo TREC 30. Form-version assumption — verify.
- Assumes title company closes (Texas is a title-company-closes state, not attorney-closes). [Texas residential closing process](https://www.freedom-res.com/post/texas-real-estate-closing-process/).
- Assumes residential — commercial has no promulgated form and very different timelines.
- Assumes financed deal default; cash deals collapse buyer-approval/property-approval to N/A and shorten closing by ~2 weeks.
- Assumes Travis County recording rules. Other counties (Hays, Williamson) similar but not identical.

**Evidence gap.** "198 tasks per transaction" claim (Paperless Pipeline) is industry marketing not authoritative; real number varies widely. Don't quote it as fact in identity.md.

---

## §3 Texas-specific gotchas

A national-generic system gets these wrong. Each one is a load-bearing detail for at least one specialist:

| Gotcha | Detail | Affects |
|---|---|---|
| **TRELA §1101.563 (effective 2026-01-01)** | Written buyer-representation agreement required *before* showing residential property or presenting an offer. Must specify services, termination date, exclusive/non-exclusive, compensation. ["Conspicuous language that broker compensation is not set by law and is fully negotiable" required.](https://www.trec.texas.gov/article/what-changes-2026-about-buyertenant-representation-texas) | `01_lead_qualifier`, `04_transaction_coordinator` |
| **IABS form (TREC IABS 1-0)** | Required at first substantive communication about specific property. Exceptions: lease <1yr no sale, party already represented, open house on that same property. Revised form mandatory since 2025-04-01. [TREC IABS form page](https://www.trec.texas.gov/information-about-brokerage-services-form) | `00_orchestrator`, `01_lead_qualifier` |
| **Intermediary status (TRELA)** | Broker can represent both buyer and seller only as intermediary, *not* as dual agent. Requires written consent with prohibited-conduct list in conspicuous bold/underlined print. [Texas REALTORS® intermediary FAQ](https://www.texasrealestate.com/members/legal-and-ethics/resources/legal-faq/intermediary/) | Envelope-level field (Austin already has `intermediary_status` — keep it) |
| **Option Fee ≠ Earnest Money** | Two fees. Option Fee: bought right to terminate for any reason during option period; 3-day delivery to title co; non-refundable; never returned. Earnest Money: good-faith deposit; 3-day delivery to escrow; refundable on legit termination; becomes damages on default. [TREC option fee Q&A](https://www.trec.texas.gov/we-are-selling-our-house-and-buyer-never-paid-option-fee-what-happens-now) | `04_transaction_coordinator` |
| **Calendar days, not business days** | All TREC contract deadlines are calendar days. Exception: earnest-money 3-day delivery rolls forward if weekend/holiday. Most other deadlines (option period termination at 5pm local) do not extend. [Paragon termination option article](https://paragonrealtors.com/blog/posts/2025/11/03/technically-speaking-termination-option-period/) | `04_transaction_coordinator` |
| **TREC 40-11 financing has two deadlines** | Buyer-approval contingency: variable, set by parties (typical 21 days). Property-approval contingency: fixed at closing_date - 3 days. [Texas REALTORS® 40-11 FAQ](https://www.texasrealestate.com/members/legal-and-ethics/resources/legal-faq/trec-third-party-financing-addendum/) | `04_transaction_coordinator` |
| **Seller's Disclosure Notice §5.008** | Required for previously-occupied single-family resale. Exemptions: new construction, foreclosure, family transfer, court-appointed fiduciary, multi-unit, commercial. Must be on file before contract executed. [Texas REALTORS® disclosure chart](https://www.texasrealestate.com/wp-content/uploads/disclosurecharts.pdf) | `04_transaction_coordinator`, `02_property_research` |
| **Title company closes, not attorney** | Texas is a title-state. No attorney required. Title company holds escrow, issues commitment + policy, prepares closing disclosure, conducts closing. National TC scripts often assume attorney handoffs. [Texas closing process](https://www.freedom-res.com/post/texas-real-estate-closing-process/) | `04_transaction_coordinator`, `03_client_communication` |
| **MUDs and PIDs in Austin metro** | Austin MUD taxes add $1,250-$7,500/year. PIDs assess additional. Lookup at austincad.org. New-construction in MUD areas (Pilot Knob, Travis MUD #22, etc.) commonly misrepresented in listings. [Neuhaus MUD/PID guide](https://neuhausre.com/guides/mud-pid-special-districts-guide-austin/) | `02_property_research` |
| **Homestead exemption (residential)** | Travis County: apply between Jan 1 and April 30 normally, but new homeowners can apply any time after purchase. Driver's license must reflect property address. [TraviCAD homestead page](https://traviscad.org/homesteadexemptions/) | `04_transaction_coordinator` (post-close advisory) |
| **UPL — Article 13 REALTOR® Code of Ethics** | Cannot give legal advice, opinion on enforceability, advise on title disputes, or draft substantive contract changes. Can insert factual data into blank attorney-approved forms. *Must recommend counsel, not provide it.* [NAR UPL article](https://www.nar.realtor/magazine/real-estate-news/law-and-ethics/what-constitutes-the-unauthorized-practice-of-law) | `03_client_communication` (primary), all specialists (secondary) |

---

## §4 Assumption ledger seed

For `decisions/assumptions.md`. Tag legend: **[evidence: strong]** = cited source above, **[evidence: medium]** = inferred from one or two sources, **[evidence: weak]** = working knowledge or single-blog source, no authoritative confirmation.

- **A1. The team operates under TRELA and uses TREC promulgated forms.** [evidence: strong] Diana is in Austin TX, brief specifies residential. Revisit if she expands out of state.
- **A2. Diana's team has MLS (UnlockMLS) access for comp research.** [evidence: medium] Industry-standard for 60-80 transaction/year teams. Verify with Diana before going live.
- **A3. The team has a default title company relationship.** [evidence: medium] Standard for boutique teams. Affects whose contact info goes into `04_transaction_coordinator`'s deal-active workflow. **Open question.**
- **A4. Diana's team predominantly handles Travis County, with some Hays / Williamson spillover.** [evidence: medium] Austin metro. CAD lookups and ISD references parameterised by county. Re-locale if she expands.
- **A5. ~5-10% of transactions are investor/cash-buyer, ~90% financed owner-occupant.** [evidence: weak] Austin 2026 market data shows financed-majority but doesn't break out by team type. Drives default assumptions in `01_lead_qualifier` field set. **Open question for Diana.**
- **A6. The team uses one transaction coordinator (in-house or contracted) per deal, not parallel TCs.** [evidence: medium] Industry standard at 4-person scale per Dotloop/Tom Ferry guides. Revisit at 15+ agents (Austin Johnson's threshold).
- **A7. Communication preference defaults to text for buyer leads <40 years old, email for sellers >50.** [evidence: weak] Folk-wisdom from communication guides; specific cutoffs are guessed. Should be captured per-client and stored in CRM, not hard-coded in `03_client_communication`.
- **A8. Diana's voice is captured per-agent (one file per agent) at `voice/<agent>.md`.** [evidence: strong — adopting Austin's pattern] Per-agent voice profile drives `03_client_communication` drafts.
- **A9. The agent reviews every drafted message before send.** [evidence: strong — UPL constraint] This is what makes the system defensible against UPL claims. **Non-negotiable; mark as such in `03_client_communication/identity.md`.**
- **A10. The team uses calendar reminders or a CRM (Follow Up Boss / dotloop / similar) — not custom software.** [evidence: medium] Diana's brief says "doesn't want software." `04_transaction_coordinator`'s output is markdown/text/CSV not API calls.
- **A11. Repeat-business + referral is the dominant lead source (>50%), not cold ads.** [evidence: medium] Industry pattern for 4-person 60-80-transaction/year boutique teams. Affects orchestrator's primary-task framing. **Open question for Diana.**
- **A12. Diana's team works exclusively with adult, non-vulnerable clients.** [evidence: weak] Edge cases (probate sales, divorce, foreclosure, vulnerable-elder) need different handling. Austin Johnson's `case_type` enum covers some — adopt that enum verbatim.
- **A13. The system writes drafts in English only.** [evidence: strong] Austin has a substantial Spanish-speaking buyer population; multilingual draft generation is a re-locale concern, not v1.
- **A14. The team does not currently use intermediary status (each agent represents one side per deal).** [evidence: weak] Many boutique teams do, many don't. Schema field `intermediary_status` exists in envelope (Austin) — defaults to `no`, must flip to `yes` with written-consent confirmation. **Open question for Diana.**

---

## §5 Sources fetched

| URL | Status | Key takeaway |
|---|---|---|
| https://www.trec.texas.gov/article/what-changes-2026-about-buyertenant-representation-texas | fetched + searched | TRELA §1101.563 effective 2026-01-01 — written buyer-rep required before showing or offer-writing residential |
| https://www.texasrealestate.com/members/legal-and-ethics/resources/legal-faq/intermediary/ | searched | Intermediary requires written consent, prohibited-conduct list in conspicuous bold/underlined; no dual agency in TX |
| https://www.trec.texas.gov/we-are-selling-our-house-and-buyer-never-paid-option-fee-what-happens-now | searched | Option fee 3-day delivery to title co; non-refundable; credited at closing or forfeited |
| https://paragonrealtors.com/blog/posts/2025/11/03/technically-speaking-termination-option-period/ | searched | Termination notice by 5pm local on last day of option period; TREC uses calendar days |
| https://www.trec.texas.gov/information-about-brokerage-services-form | searched | IABS required at first substantive communication; revised form mandatory 2025-04-01 |
| https://www.trec.texas.gov/what-are-agency-disclosure-requirements-real-estate-license-holder | listed | IABS exceptions: <1yr lease, party already represented, open-house-on-that-property |
| https://www.bostonrealestateclass.com/posts/the-new-real-estate-agents-guide-to-qualifying-your-leads/ | searched | Lead qualification: intent, budget+pre-approval, timeline within first 2 minutes |
| https://www.getnextphone.com/blog/real-estate-lead-qualification | searched | Pre-approval is the #1 filter; "timeline, pre-approval, budget" three-question rule |
| https://www.paperlesspipeline.com/blog/real-estate-transaction-coordinator-checklist | searched | TC owns post-contract phase; deadline tracking is primary deliverable |
| https://daughtreylaw.com/2024/12/23/essential-real-estate-deadlines-for-texas-success/ | searched | Texas closing timeline 30-45 days financed; key deadlines option/financing/closing |
| https://www.behappytc.com/blog/texas-real-estate-closing-process | searched | Title-company-closes state; no attorney required; closing disclosure 3-business-day rule |
| https://neuhausre.com/guides/mud-pid-special-districts-guide-austin/ | searched | MUD vs PID definitions, lookup at austincad.org, tax-rate impact |
| https://www.theclrsalesgroup.com/blog/2026/5/6/mud-taxes-in-austin-texas-the-hidden-cost-every-new-construction-buyer-needs-to-know | searched | Austin MUD taxes $1,250-$7,500/year; reduces borrowing capacity materially |
| https://www.texasrealestate.com/members/legal-and-ethics/resources/legal-faq/trec-third-party-financing-addendum/ | searched | TREC 40-11 buyer-approval (variable) and property-approval (closing-3) are separate deadlines |
| https://www.texasrealestate.com/wp-content/uploads/disclosurecharts.pdf | searched | SDN exemptions: new construction, foreclosure, family transfer, court-fiduciary, multi-unit, commercial |
| https://www.nar.realtor/magazine/real-estate-news/law-and-ethics/what-constitutes-the-unauthorized-practice-of-law | searched | UPL: no legal advice, no contract drafting, no enforceability opinions; Article 13 requires recommending counsel |
| https://avenuelegalgroup.com/re-agents-unauthorized-practice-of-law/ | searched | Agents may insert factual data into attorney-approved blank forms; cannot make substantive changes |
| https://traviscad.org/homesteadexemptions/ | searched | Travis County homestead exemption deadline Jan 1 - April 30; new owners can apply anytime |
| https://moreland.com/blog/2026-guide-to-filing-your-homestead-exemption-in-travis-county | searched | 2026 filing guide for Travis County homestead — current cadence |
| https://www.freedom-res.com/post/texas-real-estate-closing-process/ | searched | Texas residential closing process step-by-step; title-company-led |
| https://www.tomferry.com/blog/real-estate-team-hiring-employees/ | searched | Team-build guide: TC + marketing + buyer agents + listing agents is standard 4-role structure |
| https://www.dotloop.com/blog/real-estate-teams-ultimate-guide/ | searched | Boutique team workflow patterns; sequential handoff standard |
| https://matterport.com/blog/real-estate-communication | searched | Communication best practices, verbal recap + email recap pattern |
| https://www.hometrack.net/100--real-estate-text-message-templates-that-actually-convert | searched | 62% buyers prefer text (NAR); 98% open rate vs 20-25% email |
| https://cms.har.com/instantcma/ | listed | HAR Instant CMA tool — Austin MLS-integrated comp puller |
| https://www.noradarealestate.com/blog/austin-real-estate-market/ | searched | Austin 2026 market: median $582,500, 28-78 DOM range, 5.5 months inventory |
| https://www.texasrealestate.com/members/legal-and-ethics/resources/legal-faq/earnest-money/ | searched | Earnest money 3-day delivery to escrow agent; weekend roll-forward exception |
| https://www.trec.texas.gov/how-long-does-agent-have-deposit-earnest-money-once-binding-contract-has-been-negotiated | searched | Earnest money 3-day delivery rule (the only deadline that rolls weekends) |

---

*Research time-boxed to ~40 min effective. §2 per-specialist findings fed into the 5 × `identity.md` + `rules.md` drafts; §4 ledger fed into `decisions/assumptions.md`; §3 gotchas fed into per-specialist refusal triggers and `decisions/failure-modes.md`. No architecture changes were required.*
