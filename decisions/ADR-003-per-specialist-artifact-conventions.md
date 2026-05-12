# ADR-003 — Per-specialist artifact conventions

**Status:** Accepted
**Date:** 2026-05-12
**Supersedes:** None
**Superseded by:** None

## Context

The brief mandates 4 files per specialist folder (`identity.md`, `rules.md`, `examples.md`, `handoff.md`) across all 5 specialists. Calibration §2 surfaced significant divergence in how the 4 reference repos shape these files: Voiceprint's 11-section identity-as-worksheet differs from Austin's ~20-line role-and-responsibilities; Voiceprint's Rule 0 framing differs from Austin's Hard/Soft split; Arjen has no separate `handoff.md` at all.

Without a consistent shape locked across the 5 specialists, the submission risks:
- Reader confusion when comparing specialists side-by-side (failure mode for criterion #3 stranger onboarding)
- Drift in how each specialist's failure modes are documented (failure mode for criterion #2 defined-not-hand-waved)
- `handoff.md` files that don't share a common contract pattern (failure mode for criterion #2 + criterion #4)

The decision: lock per-file structural shape across all 5 specialists, citing inherited patterns inline.

## Decision

**Per-file structure locked as follows.**

### `identity.md` — 6 sections, ~25-30 lines

1. **Role** — one sentence.
2. **Core responsibilities** — bulleted, 3-5 items.
3. **Out of scope** — bulleted. What this specialist refuses or escalates.
4. **Quality standards** — bulleted. What "good output" means here.
5. **Voice & approach** — short paragraph. How this specialist communicates.
6. **Failure-mode register** — bulleted, 3-5 items for 00 / 01 / 02 / 03; **8-10 items for `04_transaction_coordinator`** (justified deviation per below). The specific failure modes this specialist exists to prevent. Each item declarative, naming a concrete failure (e.g. "Drafts an email when `agent_on_deal` is null → produces voice-anonymous output that misrepresents the team member").

§1-5 inherit Austin's structure (calibration §1.4). §6 inherits Arjen's failure-mode register pattern (calibration §1.3, §5.3). No Comp-4-level reference repo has §6 — this is the load-bearing addition.

### `rules.md` — 3 sections, ~30-40 lines

1. **Rule 0** — single sentence stating the per-specialist refusal condition, followed by a verbatim refusal-language block (Voiceprint pattern, calibration §1.1 / §5.4). Distinct from envelope-level Rule 0 in ADR-001: envelope-level Rule 0 governs whether an envelope is *processable*; per-specialist Rule 0 governs whether this specialist's *task is performable* on a valid envelope.
2. **Hard rules** — bulleted, behaviours always exhibited or always refused (Austin's Hard Refusals pattern).
3. **Soft rules / calibration** — bulleted, context-dependent behaviours (e.g. confidence-driven; voice-file-dependent). Austin's Soft Rules + Confidence Calibration pattern.

**Per-specialist Rule 0 wordings.** See CALL-015 (`decisions/design-calls.md`) for the locked verbatim text. ADR-003's structural claim is that each specialist has a Rule 0 as §1 of its `rules.md`, and that Hard/Soft rules expand on Rule 0's refusal condition. The specific wordings are CALL-grade decisions: they may be revised by domain research (e.g. TRELA §1101.563 wording-impact on `04_transaction_coordinator`) without requiring an ADR amendment. Demoted from this ADR to CALL-015 on 2026-05-12 evening — keeps the ADR's structural commitment stable while letting the wordings track real-world drift.

### `examples.md` — comparative-pairs, 3-4 per specialist (6-8 for `04_transaction_coordinator`)

Each example is a 4-part block (Voiceprint pattern, calibration §1.1):
1. **Input artifact** — what the specialist received
2. **Bad output** — plausible-but-wrong response, with annotated failure
3. **Good output** — correct response
4. **Diagnosis** — why the bad/good split

Comparative pairs preferred over Arjen's full-dialogue format because pairs surface the *contrast* — the decision boundary between bad and good — which is what a newest-agent reader needs to internalise.

**Pair-count deviation for `04_transaction_coordinator`** (added 2026-05-12 evening): 3-4 pairs is the default for 00 / 01 / 02 / 03; `04` ships 6-8 pairs because `decisions/domain-research.md` §2.5 catalogues 8+ distinct failure modes (option-fee/earnest-money confusion, calendar-vs-business day arithmetic, financing dual-deadline tracking, HOA timeline, SDN exemptions, UPL on advise-on-terminating, homestead exemption advisory, etc.) — a 3-4 pair budget cannot exhibit the decision boundaries readers need to internalise. The deviation is a deliberate scaling-with-domain-volume choice, not a violation of the convention. Other specialists are not authorised to scale pair count without similar evidence (e.g. domain research showing the failure-mode count exceeds the default).

Voice convention (real-model vs binder analogy) **locked real-model across all 5 specialists** per CALL-010 (Locked 2026-05-12 during the compressed drafting cycle; no per-example metaphor override was demonstrably needed in any of the 24 pairs produced).

### `handoff.md` — 6 sections, payload contract

Inherits Austin's per-specialist payload-contract pattern (calibration §1.4, §3):

1. **Inputs** — required-in / optional-in fields from incoming envelope. References ADR-001 schema.
2. **Outputs** — required-out / optional-out fields in produced envelope.
3. **Routing destinations** — which specialist(s) can be `to` after this one.
4. **Refusal triggers** — conditions under which this specialist back-handoffs, mapped to `handoff_reason` enum values from ADR-001 Extension 2.
5. **Confidence calibration** — when this specialist sets `confidence` to `low` / `med` / `high`.
6. **Back-handoff sources** — which specialists can `back_to` this one and under what conditions.

Refusal-trigger-to-`handoff_reason` mapping is the load-bearing tie back to ADR-001 — every refusal trigger names which enum value it produces.

## Options considered

**(a) Voiceprint's 11-section identity-as-worksheet.** Rejected. Designed for an operator-buyer audience reading a single-specialist offering. At 5 specialists × 11 sections, a stranger onboarding into the team faces ~55 sections of identity content before reaching any actual rules. Fails the "operational in a day" test on volume alone.

**(b) Arjen's narrative `identity.md`.** Rejected. Narrative form makes failure modes implicit (buried in prose) rather than explicit. Criterion #2 rewards explicit. The *failure-mode register* from Arjen's pattern is what's worth keeping; the narrative wrapper isn't.

**(c) Three-full-dialogues `examples.md` (Arjen pattern).** Rejected. Dialogues show one trajectory each. Comparative pairs (bad/good with diagnosis) surface decision boundaries — which is the actual learning content. Pairs also compress better: 3-4 pairs in the space of 1-2 full dialogues.

**(d) Skip per-specialist Rule 0; rely on envelope-level Rule 0 from ADR-001.** Rejected. The two Rule 0s handle different failure cases. Envelope-level Rule 0 catches malformed envelopes (no `schema_version`, no `case_id`). Per-specialist Rule 0 catches valid envelopes that this specialist cannot act on (valid envelope to `01_lead_qualifier` with empty `raw_inbound` is well-formed but unactionable). Without per-specialist Rule 0, that failure mode has no documented refusal handle.

**(e) Adopt Austin's per-specialist structure verbatim; no failure-mode register.** Rejected. Calibration §3 identified the absence of a failure-mode register as a real engineering gap across all 4 reference repos — failures are mentioned in prose but not catalogued. Adding §6 to `identity.md` is the cheapest insertion point (lives where responsibilities live, stays in sync naturally).

**(f) Separate `failures.md` per specialist instead of §6 of `identity.md`.** Rejected. A separate file requires keeping responsibilities and failure-modes in sync across two files. §6 of `identity.md` keeps them physically adjacent. Revisit if the register exceeds 5 items per specialist (consequences section).

## Consequences

**Positive.**
- Consistent shape across 5 specialists makes side-by-side reading clean (helps criterion #3 stranger onboarding).
- Failure-mode register in `identity.md` ships a deliverable no reference repo has at Comp 4 level — direct engineering improvement, not packaging.
- Per-specialist Rule 0 + envelope-level Rule 0 = two-layer refusal contract. System invariants and task invariants are documented at the layer where each lives.
- `handoff.md` payload-contract structure consumes ADR-001 envelope directly. Refusal-trigger-to-`handoff_reason` mapping is auditable cross-document.

**Negative.**
- 25-30 line `identity.md` × 5 specialists = ~140 lines of identity content. Some redundancy across specialists in §5 (Voice & approach) if all 5 serve the same team. Mitigation: tolerate redundancy. It's a feature here — a reader landing on `03_client_communication/identity.md` in isolation shouldn't have to read all 5 to understand the voice context.
- Two Rule 0 layers can confuse a first-time reader. Mitigation: README architecture section names both layers and links ADR-001 + ADR-003.
- `handoff.md` requires cross-document discipline with ADR-001 — adding a `handoff_reason` enum value in ADR-001 means revisiting all 5 `handoff.md` files. Documented in ADR-001 revisit trigger.

**Revisit triggers.**
- A specialist genuinely needs a section not in the locked structure (e.g. `00_orchestrator` may need a "routing logic" section). Add to that specialist only; document the deviation in this ADR.
- `examples.md` voice convention resolved 2026-05-12 (CALL-010 Locked — real-model across all 5 specialists). Revisit this ADR only if a future example demonstrably requires per-example metaphor override.
- Failure-mode register accumulates >5 items for a specialist *other than* `04_transaction_coordinator` (which already ships 8-10 items as a justified deviation per §identity.md above). Either split into a separate `failures.md` (reversing option-f) or compress the register. The 04 deviation does not trigger this revisit because the deviation was pre-authorised on domain-volume grounds (`decisions/domain-research.md` §2.5) and split-vs-co-locate was reconsidered: co-locating 8-10 items with responsibilities still avoids bridge-doc rot, whereas splitting would re-introduce the canonical failure mode option-f rejected. The trigger fires when a *different* specialist accumulates >5 items without comparable domain-volume justification.
- Real-estate domain research returns evidence that contradicts an assumption baked into a locked per-specialist Rule 0 wording. Update the affected Rule 0 and log under design calls.

## Cross-references

- **ADR-001** — `handoff.md` inherits envelope schema; refusal triggers map to `handoff_reason` enum values.
- **ADR-002** — all four files in real-model zone (no binder analogy unless explicit per-example override in `examples.md`).
- **CALL-010** (`decisions/design-calls.md`) — `examples.md` voice locked real-model across all 5 specialists.
- **Reference-repo calibration analysis** — pattern sources for §identity.md (Austin Johnson's role-and-responsibilities), §rules.md (Voiceprint Rule 0 + Austin Hard/Soft split), §examples.md (Voiceprint comparative-pairs), §handoff.md (Austin payload-contract); failure-mode register §6 (Arjen specialist-builder).
- **`decisions/architecture-properties.md`** — this ADR's per-specialist artifact shapes are the per-specialist sites of three system-level properties: P1 closed refusal-taxonomy loop (rules.md §1 + identity.md §6 + handoff.md §4), P2 global topology (handoff.md §3 + §6 declares each specialist's edges), P3 specialist execution loop (identity.md §1-§5 + rules.md §1 + handoff.md §1/§2/§5 are the per-step contracts of Think → Act → Observe).
