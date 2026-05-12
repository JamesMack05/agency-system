# Diana's Team — Architecture

The engineering view of the system. The realtor-facing guide is in [`README.md`](README.md); read that first if you haven't.

> **Target reader.** You are reviewing this system for design soundness, or extending it for your own team, or curious about why specific choices were made. You already know what the system does end-to-end; this document is *why* it's built this way.

> **A note on vocabulary.** The README uses an old-school manila-binder analogy to introduce the system without imposing schema-speak on the realtor. From here on, this document uses the real vocabulary the system actually runs on: the deal binder is the **envelope**; the sticky-tab note is the **`handoff_reason`** field; the front-cover checklist is **Rule 0**; the binder's contents and structure live in the **envelope schema**. Same concepts the realtor uses; schema-typed names. The full schema — 16 fields, both Rule 0 layers, and the six `handoff_reason` values — is in [`HANDOFF_SCHEMA.md`](HANDOFF_SCHEMA.md) at the repo root.

---

## Architecture

The five specialists are folders in this repo. Each folder contains four files: `identity.md` (role, responsibilities, failure modes), `rules.md` (Rule 0 + Hard rules + Soft rules), `examples.md` (comparative pairs showing the decision boundary), and `handoff.md` (input/output contract against the envelope schema).

```
00_orchestrator/         — router (subject classification + case_id resolution)
01_lead_qualifier/       — first-touch qualification (3-question filter, TRELA §1101.563)
02_property_research/    — sourced CMAs + neighborhood briefs (austincad.org, UnlockMLS)
03_client_communication/ — drafts in the licensed realtor's voice; never sends; UPL refusal site
04_transaction_coordinator/ — deal-active phase, deadline arithmetic, T-24h escalation
```

Every handoff between specialists produces an **envelope** — 16 fields (Austin Johnson's schema, with attribution; see [`decisions/ADR-001-envelope-schema.md`](decisions/ADR-001-envelope-schema.md)) plus two engineering extensions: an envelope-level Rule 0 invariant, and a typed `handoff_reason` enum so back-handoffs have causes the system can dispatch on, not just prose to read.

The interaction graph (forward edges + typed back edges) is enumerated in [`decisions/architecture-properties.md`](decisions/architecture-properties.md) §P2 (Global topology). The execution loop every specialist runs (Think → Act → Observe) is in §P3.

## The five specialists at a glance

| # | Folder | Role | Rule 0 | Primary refusal type |
|---|---|---|---|---|
| 00 | `00_orchestrator/` | Triage every inbound, resolve `case_id`, set `content_provenance` | "No `INTAKE.md`, no envelope." | Escalate to the licensed realtor on ambiguous routing |
| 01 | `01_lead_qualifier/` | Three-question filter: timeline / pre-approval / budget | "No `raw_inbound`, no qualification." | `back_compliance_block` on existing-representation (Article 16 / IABS exception 2) |
| 02 | `02_property_research/` | Sourced comps + neighborhood profile; price band, not opinion | "No specific subject, no brief." | `back_scope_mismatch` on valuation opinion requests |
| 03 | `03_client_communication/` | Draft in the realtor's voice; the realtor reviews and sends | "No `agent_on_deal`, no draft." | `back_compliance_block` on UPL — primary site |
| 04 | `04_transaction_coordinator/` | Deadline tracking from execution to funding | "No signed contract, no deadline tracking." | `back_compliance_block` on TRELA §1101.563 violation |

(The schema field `agent_on_deal` is named for compatibility with Austin Johnson's prior art. It identifies the **licensed realtor** on the deal — the human whose voice the comms specialist drafts in.)

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

A back-handoff at any hop returns the envelope to a prior specialist with a typed `handoff_reason`. The full topology (which edges are forward, which are back, what causes which) is in [`decisions/architecture-properties.md`](decisions/architecture-properties.md) §P2; three worked trajectories with pass criteria are in [`decisions/trajectories.md`](decisions/trajectories.md).

## Where to read what

| Question | Read |
|---|---|
| What does a specialist do? | `<specialist>/identity.md` |
| When does a specialist refuse? | `<specialist>/rules.md` |
| What does refusal vs success look like, side by side? | `<specialist>/examples.md` |
| What's the input/output contract? | `<specialist>/handoff.md` |
| Why does the envelope have these 16 fields? | [`decisions/ADR-001-envelope-schema.md`](decisions/ADR-001-envelope-schema.md) |
| Why does the README use a binder analogy and this file use the real vocabulary? | [`decisions/ADR-002-hybrid-disclosure-model.md`](decisions/ADR-002-hybrid-disclosure-model.md) |
| Why does every specialist have the same four files in the same shape? | [`decisions/ADR-003-per-specialist-artifact-conventions.md`](decisions/ADR-003-per-specialist-artifact-conventions.md) |
| What system-level properties does the architecture claim? | [`decisions/architecture-properties.md`](decisions/architecture-properties.md) |
| What design calls happened during construction and why? | [`decisions/design-calls.md`](decisions/design-calls.md) |
| What did the adversarial review find? | [`decisions/council-verdict.md`](decisions/council-verdict.md) |
| What did we look up to build this and what's still uncertain? | [`decisions/domain-research.md`](decisions/domain-research.md) + [`decisions/assumptions.md`](decisions/assumptions.md) + [`decisions/open-questions.md`](decisions/open-questions.md) |
| What are the failure modes the system exists to prevent? | [`decisions/failure-modes.md`](decisions/failure-modes.md) (compiled from each `identity.md` §6) |

## Onboarding a reviewer or new maintainer

In order, roughly 90 minutes total:

1. **The realtor-facing [`README.md`](README.md).** Read this first even if you're a reviewer. It defines the user-experience contract this architecture exists to deliver. Anything in this document that contradicts the README is a defect.
2. **[`HANDOFF_SCHEMA.md`](HANDOFF_SCHEMA.md).** Two minutes. The terse envelope contract — 16-field table, the two Rule 0 layers, the six `handoff_reason` values. The rest of the docs reference these as defined terms.
3. **`00_orchestrator/identity.md`** + **`01_lead_qualifier/identity.md`.** The two specialists that touch every new inbound. Internalise their roles before reading anything else.
4. **Any one specialist's `examples.md`.** Read at least one `examples.md` end-to-end — the comparative pairs are how each specialist reasons. `03_client_communication/examples.md` is the densest on real-team-relevant patterns (UPL refusal, anonymous-inbound salutation, voice-file degrade, TCPA window).
5. **[`decisions/ADR-001-envelope-schema.md`](decisions/ADR-001-envelope-schema.md).** The 16-field contract with rationale. You read the contract in step 2; this is the *why* — the Rule 0 + `handoff_reason` extensions in full, plus the rejection reasoning for alternatives.
6. **[`decisions/trajectories.md`](decisions/trajectories.md) T1.** One worked happy-path with envelope YAML at every hop. Once T1 makes sense, the whole system makes sense.
7. **Your own specialist's four files.** When you have a specialist assignment, you read those four in detail and you're operational.

If something stops making sense at any step, the answer is in `decisions/` — the folder exists so the system explains itself.

## Prior art

Austin Johnson's [`eccentricnode/agency-icm`](https://github.com/eccentricnode/agency-icm) (published 2026-05-12) defines the 16-field envelope schema this system adopts verbatim. Without that prior art, this submission would have shipped a weaker contract; reinventing 16 schema fields under a 5-day deadline against a public artifact that already nails it would have been time spent in the wrong place. We extend Austin's schema with two engineering additions — an envelope-level Rule 0 and a typed `handoff_reason` enum — because the original's free-text `next_action` field hand-waves at handoff semantics that the brief asks to have defined. See [`decisions/ADR-001-envelope-schema.md`](decisions/ADR-001-envelope-schema.md) for the full justification.

Other reference repos consulted during calibration: Voiceprint (specialist Rule 0 framing + comparative-pairs examples pattern), Virgilio Customs Specialist (single-specialist artifact shape), Arjen specialist-builder (failure-mode register pattern). None of those have an inter-agent handoff design at Comp-4 scale; Austin's was the strongest in the public corpus by a significant margin.

## The `decisions/` folder

The `decisions/` folder is where the engineering work lives. Three ADRs (envelope schema, hybrid disclosure model, per-specialist artifact conventions); a cross-document architecture-properties note (three system-level invariants: closed refusal taxonomy, global topology, per-specialist execution loop); three worked trajectories with pass criteria; a design-calls log capturing every implicit scoping decision made during construction; an assumption ledger with strong/medium/weak evidence tags; an open-questions register; an adversarial-review verdict from a three-reviewer council; and a domain-research artifact citing the Texas-specific regulations the system holds against (TRELA §1101.563, TCPA, UPL via NAR Article 13, §5.008 Property Code, TREC arithmetic).

The folder is not packaging. Each file maps to a real decision with consequences that propagate to specific lines of specialist code. This document points; the `decisions/` folder shows the work.

## Where to go next

- **End user (realtor or new team member):** read [`README.md`](README.md), then try the quickstart.
- **Reviewer of this architecture:** start with [`decisions/ADR-001-envelope-schema.md`](decisions/ADR-001-envelope-schema.md) + [`decisions/architecture-properties.md`](decisions/architecture-properties.md), then read any one specialist's four files.
- **Realtor's team using this in production:** each specialist's four files are operating instructions. The `decisions/` folder is the audit trail when something behaves unexpectedly and you need to know *why*.
- **Maintainer extending the system:** every ADR has a "Revisit triggers" section. When a trigger fires, the ADR is the change site, not the specialist files. The specialist files inherit from ADRs and the design-calls log.

This system fits in a folder. The realtor can use it in a day. The architect can understand it in 90 minutes. That's the design intent.
