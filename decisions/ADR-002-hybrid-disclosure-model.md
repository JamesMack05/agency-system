# ADR-002 — Hybrid disclosure model: V1 skeleton + V2 content slots

**Status:** Accepted
**Date:** 2026-05-12
**Supersedes:** None
**Superseded by:** None

## Context

The submission must satisfy two success criteria that pull in opposite directions:

1. Criterion #2 ("handoff protocols actually defined") and #4 ("real design decisions") reward exposing real architecture with rigor.
2. Diana's "operational in a day" test under criterion #3 rewards onboarding a reader without a vocabulary cliff.

A pure approach to either fails the other:

- **Pure same-architecture** (Austin's pattern): newest agent encounters envelope / payload / CAS-token terminology on day 1 before she has any reason to care. Upstream research identified this as the "vocabulary cliff" failure mode.
- **Pure analogy-first**: failed the 2026-05-12 analogy stress-test gate. Strongest candidate (deal binder + chain-of-custody log) scored 3.5/5 — passes Q1 (back-handoff), Q2 (refusal), Q4 (content_provenance); marginal on Q3 (parallel work); fails Q5 (confidence-driven downstream gating, a software concept with no paper-world ancestor).

The decision: what disclosure model satisfies both criteria without introducing a maintenance burden that decays.

## Decision

**Hybrid disclosure model with strict scoping rules.**

The README scaffold mirrors Austin's three-tier pattern (Try-Smallest-Version → Architecture → Deep Dive). The two entry-level surfaces — **narrative opener** and **"Try The Smallest Version" demo** — use the deal-binder + chain-of-custody analogy that survived the gate on Q1, Q2, Q4 (the questions relevant to those surfaces). All deeper docs use real envelope vocabulary with zero analogy.

### Scoping rules (load-bearing — without these the model leaks)

**Analogy zone** (binder + chain-of-custody appears):

- README narrative opener
- README "Try The Smallest Version" demo
- `examples.md` only where a specific example demonstrably needs the metaphor to be readable (default off — see open question)

**Real-model zone** (envelope vocabulary, zero analogy):

- `HANDOFF_SCHEMA.md`
- All per-specialist docs (`identity.md`, `rules.md`, `examples.md`, `handoff.md`) across the 5 specialist folders
- All `decisions/` content (ADRs, failure modes, assumptions, council review)
- All README content after the transition paragraph

**Boundary test:** a doc is in the right zone if it can be read standalone by its target reader. Smallest-Version reader = newest agent expecting analogy. HANDOFF_SCHEMA.md reader = developer or maintainer expecting the real schema. Cross-contamination (analogy terms in HANDOFF_SCHEMA.md, or `payload` in the smallest-version demo) signals wrong zone.

### Transition mechanism

A **5-7 line vocabulary translation paragraph** at the boundary between Smallest-Version and Architecture in the README. Maps analogy terms ("binder slot", "sticky-tab", "chain-of-custody log") to real schema fields (`payload`, `content_provenance`, `trail`). This paragraph lives **inside the README**, not as a separate `BRIDGE.md` artifact.

Engineering reason for paragraph-not-doc: upstream research identified bridge-doc rot as the canonical failure mode of explicit two-tier hybrids. A separate bridge file must stay in sync with both the analogy and the real model, becomes the de-facto onboarding doc, and decays first. Embedding the translation in the README bounds the surface (<10 lines, single location, naturally co-located with both surfaces it bridges).

## Options considered

- **(a) Pure V1 — layered same-architecture (Austin's pattern).** Rejected. Same-architecture doesn't hide rigor; it shrinks it. The newest agent still meets envelope vocabulary in the smallest tier. Diana's "operational in a day" test asks for a mental model first, not three depths of the same mental model.

- **(b) Pure V2 — analogy-first.** Rejected by the 2026-05-12 gate. Q5 (confidence-driven downstream gating) has no paper-world ancestor; no analogy carries it. Forcing the strongest candidate (3.5/5) to cover Q5 produces a leaky abstraction that judges spot under criterion #4.

- **(c) Hybrid with explicit `BRIDGE.md`.** Rejected. Bridge-doc rot is the named failure mode. A third file synced against two others doubles maintenance burden, accumulates drift, and becomes the de-facto onboarding doc. Submission has no docs-rotation discipline to catch the drift.

- **(d) Persona-tailored entry points (spaCy pattern).** Rejected. Doubles writing effort under a 5-day timeline. Diana's brief specifies one newest-agent path, not a branched audience. Complexity unjustified by the use case.

- **(e) Hybrid with scoping rules (chosen).** Justification:
  - Scoping rules contain the analogy to surfaces where its 3.5/5 score is sufficient (Q1, Q2, Q4 only — Q3 marginal and Q5 fail never surface in entry docs).
  - Translation paragraph is bounded (<10 lines, README-resident, no separate artifact).
  - Both criteria satisfied: criterion #2 holds because real-model zone exposes the schema fully; criterion #3 helped by the analogy ramp in the entry surfaces.

## Consequences

**Positive.**

- Vocabulary ramp: newest agent encounters analogy in the docs she reads first; real model where she has already engaged.
- Analogy contained to surfaces where it survives the stress test. Q3 (parallel work) and Q5 (confidence-driven gating) appear only in docs using the real model — the analogy never carries weight it can't.
- No separate bridge artifact. Translation paragraph decays only when the README itself decays, which happens on every system change anyway.
- Criterion #2 (defined-not-hand-waved) preserved fully: every doc in the real-model zone exposes envelope vocabulary with no abstraction wrapper.

**Negative.**

- Two vocabularies coexist in the repo. A reader who jumps from narrative opener to `HANDOFF_SCHEMA.md` without passing through the translation paragraph sees an unexplained shift. Mitigation: README structure forces sequential reading through the translation paragraph before deep docs are linked.
- Scoping discipline must hold across all future edits. A change that puts "binder" terminology in `HANDOFF_SCHEMA.md` or `payload` in the smallest-version demo leaks the model. Mitigation: scoping rules documented here; future edits reference this ADR.
- Schema changes per ADR-001 revisit triggers (e.g. 6th `handoff_reason` value) may require a corresponding line in the translation paragraph. Small but real coupling.

**Revisit triggers.**

- Translation paragraph exceeds 10 lines. At that point it has become a doc, not a paragraph, and bridge-doc rot risk reactivates.
- Stranger-onboarding final-QA test shows the newest agent gets lost at the analogy / real-model boundary. Disclosure model needs rework.
- A schema change introduces a concept that has no analogy expression and is referenced in the smallest-version demo. Either the demo changes or the schema change is wrong for this disclosure model.

## Open question — RESOLVED 2026-05-12 (CALL-010)

`examples.md` per-specialist — should examples use binder terminology, real envelope terminology, or mixed? **Resolved real-model across all 5 specialists.** Voiceprint's comparative-pairs pattern (artifact → bad output → good output → diagnosis) works cleanly in real-model voice; zero per-example metaphor overrides were demonstrably needed across the 24 pairs produced (4 × 00 / 4 × 01 / 4 × 02 / 4 × 03 / 8 × 04). See CALL-010 in `decisions/design-calls.md` for the resolution rationale and revisit trigger.

## System-level properties

This ADR's analogy/real-model zoning is *outside* the three system-level architecture properties (P1/P2/P3 in `decisions/architecture-properties.md`) by design: the system-level properties live entirely in the real-model zone, where they can be claimed without translation. The disclosure model's job is to keep that zone uncontaminated; the architecture properties' job is to be claim-able once a reader is inside it.

Cross-reference: `decisions/architecture-properties.md`. The boundary test (this ADR §Scoping rules) defends the real-model zone where P1/P2/P3 are stated.
