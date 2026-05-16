# Design Calls Log

Tracks implicit scoping/format decisions made during ADR drafting that did not rise to ADR-worthy on their own but are still load-bearing. Each entry: the call, engineering reasoning at the time, revisit trigger.

Use when: hitting a blockade during execution and unsure *why* a thing is shaped the way it is — read here before reopening the underlying ADR.

Companion to: `decisions/open-questions.md` (parked uncertainty). This log is the inverse — decisions closed but still worth being able to retrieve.

---

## ADR-001 — Envelope schema

### CALL-001 — Two extensions (Rule 0 + `handoff_reason`), not one
*Date:* 2026-05-12 (revised same-day evening during adversarial review)
*Status:* Locked (revised from v1 single-extension after pushback; revisit trigger exercised → enum now 6 values)
*Context:* ADR-001 §Decision

**The call:** Ship two extensions on top of Austin's 16 fields — envelope-level Rule 0 *and* `handoff_reason` enum. v1 draft proposed Rule 0 only.

**Engineering reasoning at the time:** v1's argument against `handoff_reason` was "redundant with `back_to` + `next_action`." That argument was wrong: `next_action` is free-text, and types are definition / free-text is hand-waving. Criterion #2 explicitly demands handoff protocols defined-not-hand-waved at the schema layer. Four observed back-handoff semantics in Austin's own design (data missing, scope mismatch, quality failure, time-critical) are encoded only as English prose in his `next_action`. Typing them is real engineering work.

**Revisit trigger (was):** A specialist requires a 6th `handoff_reason` value not covered by the current 5. That's the schema migration signal — increment `schema_version` and update this log.

**Revisit trigger exercised (2026-05-12 evening — same drafting cycle).** Adversarial review of the v1 5-value schema flagged that UPL / IABS / TRELA refusals — surfaced by the parallel domain research run — did not fit any of the 5 values. Compliance refusals have downstream behaviour distinct from `back_scope_mismatch` (route to human, frequently terminate system involvement entirely; not "wrong specialist, try the other one"). Conflating them would have forced the orchestrator to disambiguate at the routing layer rather than the schema layer — losing the dispatch-precision rationale that justified typing the enum in the first place.

**Migration shape:**
- Added `back_compliance_block` as 6th value with worked examples (UPL: 03 client requests enforceability opinion; TRELA: 04 deal-active without buyer-rep agreement post-2026-01-01; IABS: 00 inbound to represented counterparty).
- Bumped `schema_version: 1.0.0 → 1.1.0`.
- All 5 specialist `handoff.md` files (per CALL-019 copy-paste convention) inherit the new value on drafting day — see CALL-019 update.

**Updated revisit trigger:** A 7th value not covered by the current 6. Next migration → `schema_version: 1.2.0`. Migration cadence-tracker: 5→6 took ~6 hours (drafting cycle). 6→7 should similarly fit within a single drafting/review cycle if needed; if migration ever requires more than that, the enum has likely outgrown its design intent and should be re-architected (e.g. typed sub-categories rather than flat enum).

---

### CALL-002 — `intermediary_status` as conditional, not core 17th field
*Date:* 2026-05-12
*Status:* Locked
*Context:* ADR-001 §Schema

**The call:** Treat `intermediary_status` (TX TRELA flag) as conditional — present only when jurisdiction requires — rather than counting it among the core fields.

**Engineering reasoning at the time:** Matches Austin's framing. Counting it as core would inflate field count for a value that is `null` outside Texas. Conditional fields are a real schema pattern (sparse columns); flattening them into the core schema misrepresents the contract for non-TX deployments.

**Revisit trigger:** A second jurisdiction-specific flag emerges (CA, NY equivalent). At that point the conditional field pattern needs a named convention rather than being a one-off.

---

### CALL-003 — Rejected `next_deadline` envelope field
*Date:* 2026-05-12
*Status:* Locked
*Context:* ADR-001 §Options considered

**The call:** Do not add `next_deadline` to the envelope schema. Deadlines live in case state (`cases/CASE-YYYY-NNNN.md`); specialists query it.

**Engineering reasoning at the time:** Null for ~80% of case lifecycle (pre-contract specialists have no firm deadlines). Real estate has multiple concurrent deadlines per case (option period, financing contingency, inspection, closing) — single field can't represent the structure. Duplicating case state into envelope violates Austin's "one fact, one location" principle and creates a sync burden.

**Revisit trigger:** A specialist needs cross-case deadline awareness in a context where reading case state is too expensive (e.g. batch routing decisions). At that point, an envelope-resident deadline-pointer might be justified.

---

### CALL-004 — Rejected `provenance_proof` field
*Date:* 2026-05-12
*Status:* Locked
*Context:* ADR-001 §Options considered

**The call:** Do not add cryptographic provenance signatures. The 4-value `content_provenance` enum (`anonymous_inbound` / `verified_client` / `agent_authored` / `system_generated`) is sourcing-by-attestation.

**Engineering reasoning at the time:** Diana's 4-person team has no PKI infrastructure. Cryptographic proof would add maintenance burden no consumer can use. Right fidelity for the use case is attestation-based, not cryptographic.

**Revisit trigger:** A future client *does* require provenance proof (e.g. legal/compliance environment, or a multi-party deployment where attestation isn't trusted). Out of scope for Comp 4.

---

### CALL-005 — Rejected field compression
*Date:* 2026-05-12
*Status:* Locked
*Context:* ADR-001 §Options considered

**The call:** Do not compress any field pairs. Specifically rejected: `from`+`to`+`back_to` → `route` object; `content_provenance`+`case_type` → `flags`; `parent_envelope_id`+`trail` merged; `confidence`+`required_fields_present` merged.

**Engineering reasoning at the time:** Each candidate pair has distinct consumers and distinct change cadences. `content_provenance` is injection-defence (drives quarantine); `case_type` is business classification (drives conditional logic). `parent_envelope_id` is a CAS pointer (concurrency check); `trail` is history (audit log). `confidence` is semantic; `required_fields_present` is structural. Merging flattens distinct quality signals into one and loses dispatch precision.

**Revisit trigger:** A schema simplification pass post-Comp 4 where some fields prove unused in practice. Compression candidate becomes valid if the merged consumer never disambiguates the two sub-cases.

---

### CALL-006 — Attribution stated in 3 places
*Date:* 2026-05-12
*Status:* Locked
*Context:* ADR-001 §Attribution + §Decision + §Context

**The call:** Austin's authorship cited in three locations in ADR-001 (context, decision, standalone attribution section) and again in `HANDOFF_SCHEMA.md` + README prior-art.

**Engineering reasoning at the time:** A judge skimming the ADR should encounter attribution before "we extended it" framing. Three locations is belt-and-braces against the skim-then-misread failure mode.

**Revisit trigger:** Post-Comp 4 if Austin or other reviewers indicate attribution density is excessive. Cosmetic.

---

### CALL-007 — File location: `decisions/ADR-001-envelope-schema.md`
*Date:* 2026-05-12
*Status:* Locked
*Context:* ADR-001 file path

**The call:** ADRs land in `decisions/` folder, named `ADR-NNN-<short-slug>.md`. Lowercase, hyphen-separated.

**Engineering reasoning at the time:** Standard ADR convention. `decisions/` matches calibration §4 — the differentiation surface no reference repo has. Numbering preserves chronology of decisions.

**Revisit trigger:** Folder structure changes for any reason. Format-only.

---

### CALL-008 — Tone: competition-artifact, not internal memo
*Date:* 2026-05-12
*Status:* Locked
*Context:* ADR-001 voice

**The call:** ADRs written assuming judges read them. Tight, scannable, no internal-team shorthand.

**Engineering reasoning at the time:** Comp 4 criterion #4 ("real design decisions vs template-filling") is judged via reading these artifacts. Internal-memo tone would obscure the decision quality.

**Revisit trigger:** If we extend this system post-Comp 4 for actual team use, tone may shift toward internal memo. Format-only.

---

## ADR-002 — Hybrid disclosure model

### CALL-009 — Translation paragraph, not bridge doc
*Date:* 2026-05-12
*Status:* Locked
*Context:* ADR-002 §Transition mechanism

**The call:** Bridge between analogy zone and real-model zone is a 5-7 line paragraph embedded in the README — not a separate `BRIDGE.md` artifact.

**Engineering reasoning at the time:** Upstream research named "bridge-doc rot" as the canonical failure mode of explicit two-tier hybrids. A separate file must stay in sync with both the analogy and the real model, becomes the de-facto onboarding doc, and decays first. Paragraph-in-README has bounded surface (<10 lines), single location, and is naturally co-located with both surfaces it bridges. The README is touched on every system change anyway.

**Revisit trigger:** Translation paragraph exceeds 10 lines. At that point it has become a doc, not a paragraph, and the bridge-doc rot risk reactivates. Either extract to `BRIDGE.md` with explicit ownership, or compress.

---

### CALL-010 — `examples.md` voice
*Date:* 2026-05-12 (resolved same-day during compressed drafting cycle)
*Status:* **Locked — real-model voice across all 5 specialists, no per-example metaphor override**
*Context:* ADR-002 §Open question

**The call:** Default `examples.md` to real-model voice. Decide per-specialist during examples.md drafting if a given example genuinely fails to read without analogy framing.

**Engineering reasoning at the time:** Comparative-pairs examples (Voiceprint pattern) work in real-model voice. Binder analogy is for entry surfaces, not deep examples. But: insufficient evidence to lock now — wait until specialists are concrete enough to test each example individually.

**Resolution (2026-05-12 — drafting cycle):** All 5 × `examples.md` drafted in pure real-model voice. **No per-example metaphor override was demonstrably needed in any of the 24 pairs total** (4 × 00 / 4 × 01 / 4 × 02 / 4 × 03 / 8 × 04). The comparative-pairs structure (Input artifact → Bad output → Good output → Diagnosis) consumes envelope YAML in Input/Bad/Good and natural-language Diagnosis prose; the envelope YAML is unambiguously real-model territory (per ADR-002 §Scoping rules — `handoff.md`, `identity.md`, `rules.md`, and `examples.md` all sit in the real-model zone by default), and the Diagnosis prose references envelope fields by name (`content_provenance`, `handoff_reason`, `payload.option_fee`, etc.), which renders cleanly without metaphor. Where analogies would have been candidates (Pair 1 of `00` — content_provenance as identity-trust; Pair 1 of `04` — option fee vs earnest money as two-fee taxonomy), the real-model framing was actually *clearer* because the boundary at issue is the schema-level distinction.

**Revisit trigger:** A future specialist or future example genuinely fails the read-test in real-model voice (a reader needs metaphor to understand the decision boundary). Authorise per-example metaphor override under ADR-002 §Scoping rules ("only where a specific example demonstrably needs the metaphor to be readable") and amend this CALL with the cited example. To date, zero cases of this surfaced.

---

### CALL-011 — Boundary test as scoping enforcement
*Date:* 2026-05-12
*Status:* Locked
*Context:* ADR-002 §Scoping rules

**The call:** Enforcement rule for which zone a doc belongs to is the boundary test ("can this doc be read standalone by its target reader?"), not a banned-words list or lexical match.

**Engineering reasoning at the time:** Lexical rules (banned words) are too brittle — they fail on edge cases like quoted source material and miss conceptual leakage. Semantic test ("does the target reader understand this in isolation?") is the right level of abstraction for a 5-day comp. A banned-words list could be derived later if maintenance discipline weakens.

**Revisit trigger:** A scoping leak makes it into the repo and the boundary test failed to catch it. At that point, augment with a lexical check.

---

### CALL-012 — Smallest-Version demo specifics not in ADR-002
*Date:* 2026-05-12
*Status:* Locked
*Context:* ADR-002 scoping

**The call:** What the "Try The Smallest Version" demo *is* (folder pair, single forward handoff) is an implementation detail, not an ADR decision. Lives in the README directly.

**Engineering reasoning at the time:** ADRs capture decisions with consequences across multiple docs. The demo's shape doesn't constrain other artifacts — changing it requires only a README edit. Not ADR-worthy.

**Revisit trigger:** If the demo's shape becomes load-bearing on other docs (e.g. it demonstrates an architectural property that needs justification), it gets its own ADR.

---

### CALL-013 — No README structure mockup in ADR-002
*Date:* 2026-05-12
*Status:* Locked
*Context:* ADR-002 scoping

**The call:** Inherit Austin's three-tier scaffold (narrative → smallest-version → architecture → deep-dive → where-to-go-next) without locking the section list in an ADR.

**Engineering reasoning at the time:** Same reasoning as CALL-012. Section list is presentation, not architecture. Inheriting from Austin's tested pattern + the analogy/real-model scoping rule is the load-bearing constraint. Concrete section ordering is a README-edit decision.

**Revisit trigger:** Stranger-onboarding final QA shows the section ordering itself is the failure point (e.g. reader bounces between architecture and deep-dive). At that point, README structure becomes ADR-worthy.

---

## ADR-003 — Per-specialist artifact conventions

### CALL-014 — One ADR for all 4 file types, not four ADRs
*Date:* 2026-05-12
*Status:* Locked
*Context:* ADR-003 scope

**The call:** Lock identity.md, rules.md, examples.md, handoff.md shapes in a single ADR rather than splitting into ADR-003/004/005/006.

**Engineering reasoning at the time:** Consistency across the 4 sibling file types is the load-bearing property — they will be read side-by-side (5 specialists × 4 files = 20 docs total). Splitting decisions across 4 ADRs invites drift, because pattern-A in identity.md and pattern-B in rules.md can be locally consistent but globally inconsistent. One ADR forces a single coherent shape decision.

**Revisit trigger:** A future file type (e.g. `failures.md` per option-f rejection) becomes load-bearing enough to deserve its own ADR. At that point, extract.

---

### CALL-015 — 5 per-specialist Rule 0 wordings (canonical home)
*Date:* 2026-05-12 (relocated from ADR-003 same-day evening during adversarial review)
*Status:* Locked
*Context:* ADR-003 §rules.md (structural claim) + CALL-015 (canonical wordings)

**The call:** Canonical text for each specialist's Rule 0 lives here, not in ADR-003. ADR-003 retains the structural commitment (each specialist has a Rule 0 as §1 of `rules.md`); this CALL owns the wordings.

**Locked wordings (drafting uses these verbatim):**

| Specialist | Rule 0 |
|---|---|
| `00_orchestrator` | "No `INTAKE.md`, no envelope." |
| `01_lead_qualifier` | "No `raw_inbound` content, no qualification." |
| `02_property_research` | "No specific subject in the query, no brief." |
| `03_client_communication` | "No `agent_on_deal`, no draft." |
| `04_transaction_coordinator` | "No signed contract, no deadline tracking." |

**Engineering reasoning at the time:** Rule 0 wording constrains what the rest of `rules.md` says — Hard rules and Soft rules expand on Rule 0's refusal condition. Locking Rule 0 now prevents re-deriving each specialist's refusal logic from scratch during drafting (and risking inconsistency across the 5). The wordings are short, derivable from the envelope schema (ADR-001), and don't require domain research to validate.

**Why CALL-grade, not ADR-grade.** The wordings are revisable by domain research (`decisions/domain-research.md` §3 already flagged that `04_transaction_coordinator`'s "No signed contract, no deadline tracking" may need refinement for TRELA §1101.563 — buyer-rep-agreement-pre-showing requirement adds a deadline-tracking dimension that pre-dates contract execution). Treating wording revision as an ADR amendment would mean ADR-003 churns on every domain edit; treating it as a CALL update keeps the ADR stable.

**Revisit trigger:** Domain research returns evidence that a per-specialist refusal condition is misaligned with how the role actually works. Update the affected Rule 0 wording HERE; re-derive Hard/Soft rules from it during drafting; do NOT amend ADR-003 (its structural claim is unaffected).

**04 Rule 0 — TRELA extension resolved 2026-05-12.** Considered three options:
- (A) Status quo: "No signed contract, no deadline tracking." Relies on upstream `01_lead_qualifier` to enforce TRELA §1101.563 (no buyer-rep, no qualification → fork before showing/offer). Risk: a quiet upstream gap means 04 misses a legal precondition.
- (B) Composite Rule 0: "No signed contract AND no signed buyer-rep agreement, no deadline tracking." Defence-in-depth via Rule 0. Risk: Rule 0s should be single-condition (readable, easy to violate-check); compounding two distinct preconditions blurs which one fired on rejection.
- (C) **Chosen.** Single-condition Rule 0 retained: "No signed contract, no deadline tracking." Buyer-rep verification lives as a **Hard rule** in `04/rules.md` §2: "On receipt, verify `payload.buyer_rep_agreement.signed_pre_showing: true`. If absent or false → back-handoff with `handoff_reason: back_compliance_block`, `next_action: 'TRELA §1101.563 violated: deal-active phase requires buyer-rep agreement signed pre-showing. Escalate to broker; do not initiate deadline tracking.'"

**Why C wins on engineering grounds:**
1. Rule 0 stays single-condition (matches the pattern of the other 4 specialists — easy to mentally check).
2. Buyer-rep verification produces the new `back_compliance_block` enum value introduced under C2 — exercises the value at its canonical use site, not just in worked examples.
3. Defence-in-depth still holds: if upstream 01 lets a no-buyer-rep case through (or upstream is bypassed entirely — direct routing 00→04 on a wrongly-classified inbound), 04 catches it before initialising the deadline machine. Same protection as B, cleaner separation of concerns.
4. Failure mode is diagnosable at the right layer: contract-missing → Rule 0 (system contract); buyer-rep-missing → Hard rule (regulatory contract). Different fixes upstream.

**Drafting note for 04/rules.md:** Hard rules block per above. Reference `decisions/domain-research.md` §1.1 (TRELA effective 2026-01-01) + §2.5 + §3 (gotcha row 1) for the regulatory anchor.

---

### CALL-016 — Comparative-pairs for examples.md, not full-dialogues
*Date:* 2026-05-12
*Status:* Locked
*Context:* ADR-003 §examples.md

**The call:** Adopt Voiceprint's comparative-pairs pattern (input → bad output → good output → diagnosis) for `examples.md`, not Arjen's full-dialogue pattern.

**Engineering reasoning at the time:** Pairs surface the *decision boundary* between bad and good — that's the learning content for a newest agent. Full dialogues show one trajectory and bury the failure mode in narrative. Pairs also compress better: 3-4 pairs fit in the space of 1-2 full dialogues, meaning more decision-boundaries per page of `examples.md`.

**Revisit trigger:** A specialist has a task where the failure mode is *temporal* (only visible across multi-turn interaction). At that point, that specialist's `examples.md` may need at least one full dialogue alongside the pairs.

---

### CALL-017 — Failure-mode register as §6 of identity.md, not separate failures.md
*Date:* 2026-05-12
*Status:* Locked
*Context:* ADR-003 §identity.md, Options (f)

**The call:** Failure-mode register lives as §6 of each specialist's `identity.md`, not in a separate `failures.md` file per specialist.

**Engineering reasoning at the time:** A separate file requires keeping responsibilities (identity.md §2) and failure-modes in sync across two files — the canonical failure-mode rot pattern. Co-locating them at §6 of identity.md keeps them physically adjacent: any change to responsibilities surfaces the relevant failure modes in the same edit. Reading-cost is negligible (§6 is at the end of identity.md, after the reader has the context to interpret it).

**Revisit trigger:** Failure-mode register accumulates >5 items per specialist. At that point, identity.md becomes top-heavy with failures relative to responsibilities. Split into `failures.md` or compress the register.

---

### CALL-018 — Tolerate Voice & approach redundancy across 5 identity.md files
*Date:* 2026-05-12
*Status:* Locked
*Context:* ADR-003 §Consequences

**The call:** §5 (Voice & approach) of `identity.md` may repeat similar content across the 5 specialists (all serve the same team / voice context). Do not extract to a shared `voice.md` to dedupe.

**Engineering reasoning at the time:** A reader landing on `03_client_communication/identity.md` in isolation should be able to understand the voice context without cross-referencing. Standalone-readability is the load-bearing property here. A shared `voice.md` introduces a bridge-doc dependency (read voice.md AND identity.md) and triggers the bridge-doc rot pattern already rejected in ADR-002. Redundancy in §5 is a feature, not a bug.

**Revisit trigger:** A specialist needs a fundamentally different voice from the others (e.g. transaction_coordinator is internal-team-facing while client_communication is client-facing). At that point, divergence is the signal — not a reason to extract; instead, document the divergence in that specialist's §5.

---

### CALL-019 — handoff.md refusal triggers copy-paste from ADR-001 handoff_reason enum
*Date:* 2026-05-12 (cost-of-copy-paste re-evaluated same-day evening during adversarial review)
*Status:* Locked
*Context:* ADR-003 §handoff.md §4

**The call:** Each `handoff.md` lists its refusal triggers with explicit mapping to ADR-001's `handoff_reason` enum values — by copy-paste of the value name, not by reference to an enum file.

**Engineering reasoning at the time:** A referenced enum file (e.g. `decisions/handoff-reason-enum.md`) would centralise the values but force every `handoff.md` reader to chase the reference to understand the refusal trigger. Copy-paste keeps the handoff.md self-contained — a developer can read it standalone and know which enum values it produces. Cost of copy-paste: 5 `handoff.md` files must be touched when a 6th enum value is added in ADR-001 (already documented as ADR-001 revisit trigger).

**Cost-of-copy-paste exercised (2026-05-12 evening).** The 5→6 enum migration under CALL-001 (revisit-trigger update) requires the new `back_compliance_block` value to land in the relevant `handoff.md` files on drafting day. Per-specialist mapping (initial cut, refine during drafting against `decisions/domain-research.md` §2):

| Specialist | `back_compliance_block` applies? | Triggering condition |
|---|---|---|
| `00_orchestrator` | Yes | Inbound names a counterparty already represented by another agent → IABS exception 2 + Article 16 bar (refuse routing, escalate to human) |
| `01_lead_qualifier` | Yes | Lead admits existing representation mid-qualification → cannot continue qualifying without interfering with existing relationship |
| `02_property_research` | No (default) | Research is data assembly, not advice. Edge case: if asked for an "opinion on enforceability" of an HOA covenant, refuse via `back_compliance_block` (UPL — soft refuse, escalate to attorney). Add as Soft rule, not primary `back_compliance_block` route. |
| `03_client_communication` | Yes (primary site) | Client requests legal interpretation, contract addendum drafting, or enforceability opinion → UPL refusal per NAR Article 13 + `decisions/domain-research.md` §2.4. This is the canonical `back_compliance_block` use case. |
| `04_transaction_coordinator` | Yes | Deal-active phase request implies showing/offer-writing without buyer-rep agreement on file (TRELA §1101.563 post-2026-01-01); or asks TC to advise on terminating (UPL). |

Cost: 4 of 5 `handoff.md` files materially touched (02 only as edge case, optional). Acceptable — the 4 primary refusal sites match where the domain research already pointed. Maintenance burden bounded.

**Revisit trigger:** Enum grows beyond 7 values, or refusal-trigger semantics become complex enough that copy-paste descriptions diverge from the canonical ADR-001 definition. Extract to a referenced enum file.

---

### CALL-020 — `03_client_communication` soft-degrade pattern for missing voice file
*Date:* 2026-05-12 (added evening during adversarial review, queued for drafting)
*Status:* Locked (drafting note — implements existing decision rather than opening new design surface)
*Context:* Drafting of `03_client_communication/rules.md` §3 Soft rules / calibration

**The call:** When `03_client_communication` receives a draft request and the resolved voice file (e.g. `voice/diana.md` per `agent_on_deal: agent_diana`) is missing, the specialist degrades-with-flag rather than blocks. Drafts in baseline house voice and emits envelope with `payload.voice_file_used: null` + `payload.do_not_send_yet: true` + `confidence: low`. Agent review (always required per UPL constraint) catches the voice mismatch before send.

**Why a soft rule, not Rule 0.** Rule 0 ("No `agent_on_deal`, no draft") fires when the *anchor* is missing — without `agent_on_deal` the system has no way to choose any voice file, baseline or otherwise, and the draft would be voice-anonymous in a way that misrepresents the team. Voice-file-missing is different: `agent_on_deal` IS set, the system knows whose voice to mimic, but the voice asset is absent. Degrading-with-flag preserves throughput (Diana's team can still acknowledge the inbound) without lying about voice fidelity. This is Austin Johnson's pattern (`decisions/domain-research.md` §2.4) lifted directly — confirmed correct by domain research.

**Drafting concrete.** `03/rules.md` §3 ships at minimum:
- Soft rule: "Voice file missing → degrade-with-flag, never block. Set `voice_file_used: null`, `do_not_send_yet: true`, `confidence: low`. Surface in `next_action`: 'Voice asset missing — draft is baseline house tone; agent review must verify before send.'"
- Soft rule: "Outbound time outside 8am-9pm local → TCPA flag for texts. Queue with `do_not_send_yet: true` until next valid window unless `forward_urgent` overrides."
- Soft rule: "Anonymous inbound (envelope quarantined per ADR-001 envelope-Rule-0 `content_provenance: anonymous_inbound`) → do not draft personalised salutation. Use neutral 'Thanks for reaching out' until identity verified upstream."

**Engineering reasoning at the time:** This is not a new design decision — it's drafting fidelity. The soft-degrade pattern was implicit in `decisions/domain-research.md` §2.4 / A8 / A9 but never made it into ADR-003's structural lock (because §3 Soft rules content is per-specialist by design, not ADR-locked). Adversarial review surfaced the gap; capturing here so drafting doesn't miss it.

**Revisit trigger:** A different soft-degrade pattern is needed for a different specialist (e.g. `02_property_research` degrade when MLS access is intermittent). At that point, generalise to a "soft-degrade taxonomy" CALL covering all specialists.

---

### CALL-021 — `03_client_communication` confidence cap for `do_not_send_yet: true` depends on the cause
*Date:* 2026-05-12 (surfaced during `03/examples.md` Pair 4 drafting; resolved same-day)
*Status:* **Locked — Option A**
*Context:* `03/handoff.md` §5 Confidence calibration + `03/rules.md` §3 calibration

**The call surfaced:** `03/handoff.md` §5 previously mapped `do_not_send_yet: true` for any reason → `confidence: low`. That calibration was written for the voice-missing / UPL-adjacent cases (Pair 3 + Pair 1 of `03/examples.md`) where the draft *content* is degraded. For the outside-TCPA-window case (Pair 4 of `03/examples.md`), the draft content is high-quality and ready to send; only the timing is gated. Capping at `low` over-signals degradation; `med` better represents "ready draft, gated timing."

**Options considered:**
- **(A) Chosen.** Sub-type the confidence cap by `do_not_send_yet_reason`. `voice_missing` / `sensitive_content_pending_agent_decision` / `refusal_recommendation_pending_agent_review` → `low`; `outside_send_window` → `med` (timing gate only).
- **(B) Rejected.** Leave the calibration as `low` for all `do_not_send_yet: true` cases. Treat the outside-window case as an outlier where downstream consumers override per-case. Loses dispatch precision — downstream cannot distinguish "draft degraded" from "draft fine, timing gated" without re-reading the reason.
- **(C) Rejected.** Drop the `do_not_send_yet: true` → confidence-cap rule entirely. Loses the useful "anything gated → at-least-cap" property; introduces a fourth confidence-determining axis.

**Engineering reasoning (Option A):** Confidence is consumed by downstream gating logic; if every gated draft is `low`, downstream cannot distinguish "draft is degraded" from "draft is fine, timing is gated." The two cases warrant different downstream behaviour — the first asks the agent to consider rewriting; the second is purely a queue-release decision. Sub-typing the cap by `do_not_send_yet_reason` (already a typed field in the payload schema) preserves the signal at no additional schema cost.

**Files updated 2026-05-12:**
- `03/handoff.md` §5 — calibration table re-stated with cause-based caps; added preamble paragraph explaining cause-vs-gating-state distinction.
- `03/rules.md` §3 — mirror calibration table updated with reason-specific cause names.

**Bridging text added (handoff.md §5 preamble):** *"Confidence reflects the *draft's send-readiness*, not just the gating state. Two drafts can both carry `do_not_send_yet: true` and warrant different confidence — content-degraded drafts cap at `low` (the agent may want to rewrite); timing-gated drafts cap at `med` (the draft is ready, only the send window is closed). Cause is captured in `do_not_send_yet_reason`; confidence dispatches on cause."*

**Revisit trigger:** A new `do_not_send_yet_reason` is introduced (e.g. recipient-blocked, channel-failure). At that point, classify the new reason as content-degraded (→ `low`) or operational-gate-only (→ `med`) and append to the table. If a reason resists classification, the binary content-vs-gating model is wrong and the calibration needs re-design.

---

### CALL-022 — `agent_on_deal` producing site + `team_lead` sentinel for pre-conversion drafts
*Date:* 2026-05-12 (surfaced during artifact-review sub-agent pass; resolved same-day)
*Status:* **Locked — Option A**
*Context:* `01_lead_qualifier/handoff.md` §2 + `03_client_communication/identity.md` §5 + `03_client_communication/rules.md` §1

**The call surfaced:** Artifact-review (sub-agent triage of staged drafts before submission) found `agent_on_deal` has no producing site across the 5 specialists. 03's Rule 0 demands it. 00 said "set downstream"; 01 said "pre-assigned by orchestrator policy" (hand-wave); 02 said "carried through." Trajectory T1 hop 4 hand-waved `# set by 00 on conversion`. Closed-loop invariant P1 held *in spec* (refusal taxonomy was closed) but failed *in derivation* — no upstream specialist could produce a valid value to satisfy Rule 0 in pre-conversion paths.

**The complication:** T1 (happy-path post-qualification) and T2 (early-touch back-handoff before lead converts) want different behaviors. T1 wants a named agent (e.g., Diana takes downsizers personally). T2 sends a thanks-for-call before any agent has claimed the case.

**Options considered:**
- **(A) Chosen.** Sentinel + team-voice file. 01 sets `agent_on_deal: team_lead` by default; named agent when an assignment rule (geography + sub_type → agent) matches. 03 has a real voice file for `team_lead` (Diana's team-default voice). Rule 0 satisfied either way.
- **(B) Rejected.** Null + accept Rule 0 loop. 01 emits `agent_on_deal: null` pre-conversion; 03 fires `back_data_missing`; 01 fills from heuristic; re-routes. Honest about the system's structure but adds an extra hop on every pre-conversion path — pure-tax behaviour for a common case.
- **(C) Rejected.** Two modes in 03 (`payload.draft_mode: agent_voice / team_voice`). Most expressive but Rule 0 semantics shift, payload schema gains a field, 03's structure changes across identity/rules/examples/handoff, and the mode flag bleeds into 04 (the `04→03` deadline-decision route — what mode?). Disproportionate invasiveness vs. (A).

**Engineering reasoning (Option A):** The sentinel pattern preserves Rule 0's closed-loop invariant at the schema layer (no null case) without forcing a new field through the payload schema. `team_lead` is a real voice resolution (real file, real voice), not a placeholder — so the existing CALL-020 voice-file-missing degrade-with-flag handles the residual edge case (sentinel set, voice file absent) without modification. T2 narrative simplifies: hop 3 emits `agent_on_deal: team_lead` (sentinel set), Rule 0 passes, the only refusal trigger is `preferred_channel: null` → `back_data_missing` — single clean cause, not a Rule-0-collapse.

**Files updated 2026-05-12:**
- `01_lead_qualifier/handoff.md` §2 — replaced the `agent_on_deal: null unless pre-assigned by orchestrator policy` line with the assignment-rule-or-sentinel-default rule.
- `01_lead_qualifier/identity.md` §6 — added failure mode for `agent_on_deal: null` emission on forward routes.
- `01_lead_qualifier/examples.md` Pair 1 + Pair 4 Good outputs — envelopes now show `agent_on_deal: team_lead`.
- `03_client_communication/identity.md` §5 — added paragraph on `team_lead` sentinel + team-default voice file.
- `03_client_communication/rules.md` §1 — sentinel-acceptance clarification before the verbatim refusal language.
- `03_client_communication/handoff.md` §1 — Required-in clarified: sentinel and named agent both satisfy Rule 0.
- `decisions/trajectories.md` T2 — narrative annotated: hop 3 emits `agent_on_deal: team_lead` (sentinel); Rule 0 satisfied.
- `decisions/trajectories.md` T1 (post-verification-pass) — HOP 2 comment updated (00→01 emission null because producing site is 01, not orchestrator); HOP 3 envelope updated to `agent_on_deal: "agent_diana"` (assignment table A15 sample rule matched: downsize + 78703 → agent_diana); HOP 4 comment updated to reflect carry-through from 01 (not "set by 00 on conversion").
- `decisions/failure-modes.md` — picked up new 01 + 03 failure modes (compiled view).
- `decisions/assumptions.md` — Team Assignment Table stub added (minimal at submission; expected to be filled with Diana on intake).
- `HANDOFF_SCHEMA.md` (post-verification-pass) — row 7 (`agent_on_deal`) Example column extended to include `team_lead` sentinel and the null-only-on-00→01-emission rule; minimal-envelope example comment updated to match.
- `02_property_research/handoff.md` §1 (post-verification-pass) — `payload.research_request.purpose` enum aligned to 01's emit set (`cma_only` / `showing_prep` / `neighborhood_brief` / `valuation_comparison`); was previously a draft-purpose enum that didn't match 01's research-purpose emit. Per verification-S1 finding.

**Assignment table policy:** The named-agent assignment table at `decisions/assumptions.md` § Team Assignment Table is intentionally minimal at submission. The defensible default is the sentinel; named-agent rules are filled with Diana on intake (Comp-4-as-template — the system is teachable to Diana's team in a week, so onboarding includes the assignment table fill). This avoids inventing team members the brief doesn't name.

**Revisit trigger:** A specialist other than 03 introduces a voice-anchored Rule 0 (e.g., 04 adds a recipient-voice-required draft mode). At that point, generalise the sentinel-pattern as a system-level invariant rather than a 03-specific accommodation. Or, if Diana's team finds the sentinel-default is firing on most leads through first-touch and into research, tighten the assignment table at intake.

---

### CALL-023 — `01_lead_qualifier` routing tie-breaker when both research and first-touch apply
*Date:* 2026-05-12 (surfaced during T1 end-to-end integration test; resolved same-day)
*Status:* **Locked**
*Context:* `01_lead_qualifier/handoff.md` §3 Routing destinations

**The call surfaced:** End-to-end integration test (HOP 3 of T1, run in Claude Chat with the full 01 spec + a Sarah-Chen-style inbound where the inbound_property_ref named a team-held listing) found that 01's §3 routing destinations under-specifies what to do when a qualified lead needs BOTH research and first-touch outreach. The trajectory's worked example (T1 HOP 3) routes `01→02_property_research` directly. The LLM under integration test routed `01→03_client_communication` first, reasoning that the intermediary-status concern (team holds the listing → dual-rep requires written consent under TRELA) and the unsigned buyer-rep (TRELA §1101.563 blocks showings) both demand pre-research disclosure. Both choices are defensible. The spec didn't disambiguate.

**Options considered:**
- **(A) Chosen.** Default to `02_property_research` first (research-informed first-touch is more substantive); exception routes to `03_client_communication` first when intermediary concern, buyer-rep gap with showing intent, or IABS-not-delivered with substantive discussion imminent.
- **(B) Rejected.** Default to `03_client_communication` first always (disclosure-first). Loses the research-informed first-touch quality in the common case; over-corrects for the edge cases.
- **(C) Rejected.** Leave it as a judgment call ("qualifier decides"). Under-specification is the failure mode the spec exists to prevent; making the call explicit makes the system testable.

**Engineering reasoning (Option A):** The default exists because most leads do not name team-held listings and have IABS already delivered — research-first is the common path and produces stronger first-touch comms. The exceptions name the structural conditions under which disclosure MUST precede research; they're enumerable and testable, not vibe-based. The rule covers the compliance edge cases without over-bureaucratising the common path.

**Files updated 2026-05-12:**
- `01_lead_qualifier/handoff.md` §3 Routing destinations — tie-breaker rule appended after the Forward `to` list.

**Revisit trigger:** A fourth structural condition surfaces where disclosure must precede research (e.g., a new regulation post-CALL-023 date), OR if the exception path fires on >30% of forward emissions (suggesting the default is wrong and exceptions should be the default).

---

### CALL-024 — Trajectory T1 HOP 3 confidence value contradicted 01/handoff.md §5 calibration rule
*Date:* 2026-05-12 (surfaced during T1 end-to-end integration test; resolved same-day)
*Status:* **Locked — trajectory patched**
*Context:* `decisions/trajectories.md` T1 HOP 3 envelope vs. `01_lead_qualifier/handoff.md` §5 confidence calibration

**The call surfaced:** T1 HOP 3 envelope was specified with `confidence: high`. 01/handoff.md §5 explicitly caps confidence at `med` when `buyer_rep_agreement_status: not_yet_signed` (even when lead is willing to sign). The trajectory's hand-walked example contradicted its own rule. The LLM under integration test correctly applied the rule (emitted `med`), exposing the contradiction.

**The resolution:** Trajectory patched to `confidence: med`. Calibration rule in 01/handoff.md §5 was correct; the trajectory was wrong.

**Why this matters beyond the trajectory file:** This is a cross-file consistency bug that survived `/council` adversarial review, three artifact-review sub-agents (stranger-onboarding, handoff-protocol, voice/template), and a post-fix verification sub-agent. None of those reviewers caught it because all of them READ artifacts; none EXECUTED them. The LLM under integration test had to DERIVE the confidence value from the rule applied to the lead's state — and immediately surfaced the contradiction. **Read-based review can never substitute for execution-based testing.** Captured as a meta-lesson in `04 Resources/Skill Patterns/skill-feedback-log.md` and `~/.claude/projects/.../memory/feedback_planning_stages.md`.

**Files updated 2026-05-12:**
- `decisions/trajectories.md` T1 HOP 3 — `confidence: high` → `confidence: med`; comment updated to reference 01/handoff.md §5 calibration.

**Revisit trigger:** A future trajectory hand-walk produces an envelope value that doesn't satisfy a per-specialist calibration rule — at that point, formalise the rule that trajectories must be derivable from per-specialist rules (no free-form envelope content in trajectories).

---

## Add new calls here as drafting continues
