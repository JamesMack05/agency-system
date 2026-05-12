# Diana's Team — Five-Specialist Workflow System

A teachable, paper-shaped operating model for a boutique Austin residential real estate team. Five specialists, one shared envelope contract, no software to install. The agent reviews every client-facing message before it sends.

> **Target reader.** You are joining Diana's team next week. You know the difference between a buyer-rep agreement and an earnest money deposit, but you have never seen this system before. This README should make you operational by end of day.

---

## What this is, in three sentences

Diana's team handles ~70 residential transactions a year with four people. Inbound work — a Zillow lead, a returning client text, an executed contract from a title company — flows through five specialists in sequence: a **router** (who triages), a **qualifier** (who runs the three-question filter), a **research** specialist (who pulls comps and neighborhood facts), a **comms drafter** (who writes in the agent's voice), and a **transaction coordinator** (who owns every deadline once a contract is executed). Each specialist hands the work off to the next using a shared **case file** — and any specialist can hand a case *back* with a sticky-tab note if something is missing.

That's the whole system.

## The deal binder

Picture an old-school manila binder. One per deal. The case number is on the spine. Inside is every document about that case — the original inbound, the qualifier's notes, the research brief, drafts that went out, the executed contract, the deadline tracker. On the front cover, a **sign-in / sign-out log** shows who has held the binder, when, and why. Inside the cover, a short **checklist** says: *before you do anything with this binder, confirm the case number is on the spine, the source of the inbound is stamped, and the previous specialist signed off.*

The binder physically moves between teammates. When Maria (the research specialist) is done pulling comps for 2412 Hartford Rd, she signs out, drops the binder on Diana's desk for drafting the first-touch email, and Diana signs in. When Diana's drafting and notices Maria forgot to pull the school-district data, Diana doesn't fix it herself — she walks the binder back to Maria with a **sticky-tab note** ("missing: ISD attendance + Casis rating") and Maria fills it in.

A few stamps on the cover indicate where the inbound came from:

- **[UNVERIFIED]** — anonymous inbound (Zillow relay, web-form with no name, voicemail with no callback). The team doesn't yet know who the person is.
- **[CLIENT]** — identity confirmed against an existing case or via a qualification call.
- **[TEAM]** — a team member logged this themselves (an inspection report attached, a status update typed in).

These stamps matter because the rules change downstream. A binder with the **[UNVERIFIED]** stamp cannot get a personalised "Hi Sarah" email drafted on it — neutral salutation only — until somebody confirms who Sarah actually is.

## Try the smallest version

Imagine one inbound moving through two hops.

**Step 1 — Inbound capture.** A web-form lead arrives at 14:32:

> *"Hi, saw your Tarrytown listing on Zillow, can someone reach out? — Sarah"*

The **router** opens a fresh binder, writes case number `CASE-2026-0042` on the spine, drops the inbound inside, stamps the cover **[UNVERIFIED]** (Zillow relay — the team has no idea who actually typed that), and signs the cover log: *"14:32 — Router — handing to Qualifier."*

**Step 2 — Qualification.** The qualifier (Tom) picks up the binder, opens it, sees the inbound, calls the listed number, and runs the three-question filter: *timeline, pre-approval, budget.* Sarah is real, has a pre-approval letter from Frost Bank for $850k, wants to move in 60 days, is downsizing from Westlake. Tom updates the cover stamp: **[CLIENT]** (identity confirmed during the call). He files the qualification notes inside the binder, signs the cover log — *"14:48 — Qualifier — handing to Research, agent_on_deal: Diana"* — and drops the binder on Maria's desk.

**A near-miss.** Suppose Tom had reached Sarah, run the call, but Sarah had hung up before he asked her preferred contact channel. Tom can't hand the binder to the *drafter* yet — the drafter needs to know whether to email or text. So Tom puts a sticky tab on the binder: *"missing: preferred channel"* and hands it back. The drafter never sees an incomplete binder; he sees binders that are ready to act on or sticky-tabbed for fixes. The team never wastes a hop on something that won't work.

That's the entire system in two hops plus one near-miss: **case file moves forward when complete, moves back with a typed sticky-tab note when not.**

---

## How the rest of this README reads

Everything from here uses the system's real vocabulary — the words the team uses inside the binders and across the `decisions/` folder. The deal binder is the **envelope**: a single structured document that travels with each handoff, carrying the case ID, the source-stamp (`content_provenance`), the sign-in log (`trail`), and the payload (whatever the producing specialist needs to pass on). The sticky-tab note is the **`handoff_reason`** field: one of six typed values (forward_normal, forward_urgent, back_data_missing, back_scope_mismatch, back_quality_failure, back_compliance_block). The front-cover checklist is **Rule 0**: a precondition gating whether the specialist can act — applied at two layers, an **envelope-level** invariant every specialist checks on receipt (case_id present, payload non-empty, `content_provenance` valid) and a **per-specialist** precondition unique to each role (03 won't draft without `agent_on_deal`; 04 won't track without an executed contract date). The full envelope schema — 16 fields, both Rule 0 layers, and the six `handoff_reason` values — is in `HANDOFF_SCHEMA.md` at the repo root. From here on, "envelope" not "binder," "handoff_reason" not "sticky-tab" — same concepts, schema names.

---

## Architecture

The five specialists are folders in this repo. Each folder contains four files: `identity.md` (role, responsibilities, failure modes), `rules.md` (Rule 0 + Hard rules + Soft rules), `examples.md` (comparative pairs showing the decision boundary), and `handoff.md` (input/output contract against the envelope schema).

```
00_orchestrator/         — router (subject classification + case_id resolution)
01_lead_qualifier/       — first-touch qualification (3-question filter, TRELA §1101.563)
02_property_research/    — sourced CMAs + neighborhood briefs (austincad.org, UnlockMLS)
03_client_communication/ — drafts in agent voice; never sends; UPL refusal site
04_transaction_coordinator/ — deal-active phase, deadline arithmetic, T-24h escalation
```

Every handoff between specialists produces an **envelope** — 16 fields (Austin Johnson's schema, with attribution; see `decisions/ADR-001-envelope-schema.md`) plus two engineering extensions: an envelope-level Rule 0 invariant, and a typed `handoff_reason` enum so back-handoffs have causes the system can dispatch on, not just prose to read.

The interaction graph (forward edges + typed back edges) is enumerated in `decisions/architecture-properties.md` §P2 (Global topology). The execution loop every specialist runs (Think → Act → Observe) is in §P3.

## The five specialists at a glance

| # | Folder | Role | Rule 0 | Primary refusal type |
|---|---|---|---|---|
| 00 | `00_orchestrator/` | Triage every inbound, resolve `case_id`, set `content_provenance` | "No `INTAKE.md`, no envelope." | Escalate to human on ambiguous routing |
| 01 | `01_lead_qualifier/` | Three-question filter: timeline / pre-approval / budget | "No `raw_inbound`, no qualification." | `back_compliance_block` on existing-representation (Article 16 / IABS exception 2) |
| 02 | `02_property_research/` | Sourced comps + neighborhood profile; price band, not opinion | "No specific subject, no brief." | `back_scope_mismatch` on valuation opinion requests |
| 03 | `03_client_communication/` | Draft in agent's voice; agent reviews and sends | "No `agent_on_deal`, no draft." | `back_compliance_block` on UPL — primary site |
| 04 | `04_transaction_coordinator/` | Deadline tracking from execution to funding | "No signed contract, no deadline tracking." | `back_compliance_block` on TRELA §1101.563 violation |

## A typical case from inbound to close

```
00 (route)  →  01 (qualify)  →  02 (research)  →  03 (draft first-touch)
                                                          │
                              [off-system: showings, offer, contract execution]
                                                          │
                                                          ▼
                                                   04 (deal-active)
                                                          │
                                  back-and-forth to 03 on T-24h decisions
                                                          │
                                                          ▼
                                                        END
```

A back-handoff at any hop returns the envelope to a prior specialist with a typed `handoff_reason`. The full topology (which edges are forward, which are back, what causes which) is in `decisions/architecture-properties.md` §P2; three worked trajectories with pass criteria are in `decisions/trajectories.md`.

## Where to read what

| Question | Read |
|---|---|
| What does a specialist do? | `<specialist>/identity.md` |
| When does a specialist refuse? | `<specialist>/rules.md` |
| What does refusal vs success look like, side by side? | `<specialist>/examples.md` |
| What's the input/output contract? | `<specialist>/handoff.md` |
| Why does the envelope have these 16 fields? | `decisions/ADR-001-envelope-schema.md` |
| Why does the README use a binder analogy here but real-model language everywhere else? | `decisions/ADR-002-hybrid-disclosure-model.md` |
| Why does every specialist have the same four files in the same shape? | `decisions/ADR-003-per-specialist-artifact-conventions.md` |
| What system-level properties does the architecture claim? | `decisions/architecture-properties.md` |
| What design calls happened during construction and why? | `decisions/design-calls.md` |
| What did the adversarial review find? | `decisions/council-verdict.md` |
| What did we look up to build this and what's still uncertain? | `decisions/domain-research.md` + `decisions/assumptions.md` + `decisions/open-questions.md` |
| What are the failure modes the system exists to prevent? | `decisions/failure-modes.md` (compiled from each `identity.md` §6) |

## Onboarding a new team member in a day

In order, reading roughly 90 minutes total:

1. **This README, twice.** Once for the analogy. Once for the real vocabulary. The transition paragraph above is where the second read switches mode.
2. **`HANDOFF_SCHEMA.md`.** Two minutes. The terse envelope contract — 16-field table, the two Rule 0 layers, the six `handoff_reason` values. The rest of the docs reference these as defined terms, so reading the contract here makes everything downstream make sense.
3. **`00_orchestrator/identity.md`** + **`01_lead_qualifier/identity.md`.** The two specialists that touch every new inbound. Internalise their roles before reading anything else.
4. **Any one specialist's `examples.md`.** Read at least one `examples.md` end-to-end — the pairs are how the system reasons. `03_client_communication/examples.md` is the densest on real-team-relevant patterns (UPL refusal, anonymous-inbound salutation, voice-file degrade, TCPA window).
5. **`decisions/ADR-001-envelope-schema.md`.** The 16-field contract with rationale. You read the contract in step 2; this is the *why* — the Rule 0 + `handoff_reason` extensions in full, plus the rejection reasoning for alternatives.
6. **`decisions/trajectories.md` T1.** One worked happy-path with envelope YAML at every hop. Once T1 makes sense, the whole system makes sense.
7. **Your own specialist's four files.** When you have a specialist assignment, you read those four in detail and you're operational.

If something stops making sense at any step, the answer is in `decisions/` — the folder exists so the system explains itself.

## Prior art

Austin Johnson's [`eccentricnode/agency-icm`](https://github.com/eccentricnode/agency-icm) (published 2026-05-12) defines the 16-field envelope schema this system adopts verbatim. Without that prior art, this submission would have shipped a weaker contract; reinventing 16 schema fields under a 5-day deadline against a public artifact that already nails it would have been time spent in the wrong place. We extend Austin's schema with two engineering additions — an envelope-level Rule 0 and a typed `handoff_reason` enum — because the original's free-text `next_action` field hand-waves at handoff semantics that the brief asks to have defined. See `decisions/ADR-001-envelope-schema.md` for the full justification.

Other reference repos consulted during calibration: Voiceprint (specialist Rule 0 framing + comparative-pairs examples pattern), Virgilio Customs Specialist (single-specialist artifact shape), Arjen specialist-builder (failure-mode register pattern). None of those have an inter-agent handoff design at Comp-4 scale; Austin's was the strongest in the public corpus by a significant margin.

## The `decisions/` folder

The `decisions/` folder is where the engineering work lives. Three ADRs (envelope schema, hybrid disclosure model, per-specialist artifact conventions); a cross-document architecture-properties note (three system-level invariants: closed refusal taxonomy, global topology, per-specialist execution loop); three worked trajectories with pass criteria; a design-calls log capturing every implicit scoping decision made during construction; an assumption ledger with strong/medium/weak evidence tags; an open-questions register; an adversarial-review verdict from a three-reviewer council; and a domain-research artifact citing the Texas-specific regulations the system holds against (TRELA §1101.563, TCPA, UPL via NAR Article 13, §5.008 Property Code, TREC arithmetic).

The folder is not packaging. Each file maps to a real decision with consequences that propagate to specific lines of specialist code. The README points; the `decisions/` folder shows the work.

## Where to go next

- New team member: read the onboarding-in-a-day list above.
- Architect reviewing for adoption: start with `decisions/ADR-001` + `decisions/architecture-properties.md`, then read any one specialist's four files.
- Diana's team using this in production: each specialist's four files are operating instructions. The `decisions/` folder is the audit trail when something behaves unexpectedly and you need to know *why*.
- Maintainer extending the system: every ADR has a "Revisit triggers" section. When a trigger fires, the ADR is the change site, not the specialist files. The specialist files inherit from ADRs and CALLs.

This system fits in a folder. You can read it in a day. You can teach it to a colleague. That's the design intent.
