# ADR-001 — Adopt and extend Austin Johnson's 16-field envelope schema

**Status:** Accepted
**Date:** 2026-05-12
**Supersedes:** None
**Superseded by:** None

## Context

The Comp 4 brief requires 5 specialist agents that hand off work. Criterion #2 explicitly judges whether "handoff protocols are actually defined or hand-waved." Without a typed inter-specialist contract, the architecture collapses to ad-hoc message-passing and fails criterion #2.

Austin Johnson's `eccentricnode/agency-icm` (published 2026-05-12, ~4hrs before our calibration pass) defines a 16-field envelope schema in a single root-level `HANDOFF_SCHEMA.md`. Calibration analysis found this is the strongest handoff design in the public corpus by a significant margin. The three other reference repos investigated (Voiceprint, Virgilio Customs Specialist, specialist-builder) are single-specialist and have no inter-agent handoff pattern.

The decision: (a) reinvent from first principles, (b) adopt Austin's verbatim, (c) adopt + extend, or (d) adopt a subset.

## Decision

**Adopt Austin's 16-field envelope verbatim, with explicit attribution. Extend with two additions, both load-bearing on engineering grounds:**

1. An envelope-level Rule 0 stated as the contract's first invariant.
2. A `handoff_reason` enum that types the back-handoff taxonomy.

### Schema (verbatim from Austin, attributed)

| # | Field | Purpose |
|---|---|---|
| 1 | `schema_version` | Migration anchor (semver) |
| 2 | `case_id` (`CASE-YYYY-NNNN`) | Stable identity; never reassigned |
| 3 | `from` | Producing specialist folder |
| 4 | `to` | Receiving specialist folder (or `END`) |
| 5 | `back_to` | Populated on back-handoff; null on forward |
| 6 | `timestamp` | ISO 8601 |
| 7 | `agent_on_deal` | Drives voice-file selection |
| 8 | `payload` | Shape owned by producing specialist's `handoff.md` |
| 9 | `required_fields_present` | Receiver-side validation result |
| 10 | `confidence` (`low`/`med`/`high`) | Drives downstream gating |
| 11 | `next_action` | Declarative instruction to receiver |
| 12 | `trail` | Append-only audit log |
| 13 | `parent_envelope_id` | CAS-style concurrency token |
| 14 | `content_provenance` (4-value enum) | Drives quarantine behaviour |
| 15 | `case_type` (6-value enum) | Conditional logic on transaction class |
| 16 | `linked_case_ids` | Cross-case references (concurrent buy+sell) |

Plus conditional `intermediary_status` (TX TRELA flag) — present when jurisdiction requires; not part of core 16.

**Current schema version:** `1.1.0`. Bumped from `1.0.0` on 2026-05-12 evening: Extension 2's `handoff_reason` enum expanded from 5 to 6 values (added `back_compliance_block` for UPL/IABS/TRELA refusals). The migration trigger documented under CALL-001 (and ADR §Revisit triggers) was exercised within the same drafting cycle by adversarial review — flagging that the 5-value enum under-covered Texas-domain refusal causes that the system materially encounters in `03_client_communication` and `04_transaction_coordinator`.

### Extension 1 — Envelope-level Rule 0

> **Rule 0.** No `schema_version` → no processing. No `case_id` → no processing. An envelope where `content_provenance: anonymous_inbound` is quarantined for sender-identity verification before processing.
>
> Rule 0 is checked before any other validation. A receiving specialist that processes an envelope failing Rule 0 has violated the contract.

**Engineering rationale.** Austin's design distributes refusal logic across per-specialist `handoff.md` files. There is no system-contract invariant — only specialist-local refusals. Rule 0 fills that gap: it is the single check every receiving specialist runs before specialist-specific validation. Without it, a malformed envelope can reach a specialist's domain logic and produce domain-shaped errors (harder to diagnose) instead of contract-shaped rejections (immediately diagnosable). Borrows the Rule 0 framing from Voiceprint (where it operates inside a single specialist) and lifts it to the envelope level.

**Layering principle (envelope-Rule-0 ≠ per-specialist Rule 0).** Envelope-Rule-0 governs whether an envelope is *processable* — a malformed envelope (no schema version, no case ID, unverified anonymous identity) fails this layer regardless of who receives it. ADR-003's per-specialist Rule 0 governs whether the specialist's task is *performable* on an otherwise-valid envelope (e.g., `01_lead_qualifier`: "no `raw_inbound`, no qualification"; `03_client_communication`: "no `agent_on_deal`, no draft"). The two layers are *not* duplicative: a draft-producing handoff to `03_client_communication` with missing `agent_on_deal` is *processable* (envelope is well-formed) but *not performable* (03 cannot draft without a voice-file anchor), so it is rejected at the per-specialist layer. Earlier drafts of this Rule 0 included an `agent_on_deal` predicate at the envelope layer — that was a layering defect (folder-coupling envelope contract to one specialist's task precondition); removed 2026-05-12 evening.

**Absence vs value (the `content_provenance` predicate).** Rule 0 fires on the *value* `anonymous_inbound`, not on field absence. Absence of `content_provenance` is itself a required-field violation handled at the per-specialist layer (ADR-003 `handoff.md` §1 — receiving specialists cannot set `required_fields_present: true` on an envelope where field 14 is unset). Earlier drafts of this predicate inverted the driver (fired on absence rather than value); fixed 2026-05-12 evening. The semantic distinction matters: an envelope arriving with `content_provenance: anonymous_inbound` is a *known quarantine case* (the system knows the inbound came from an unverified source); an envelope arriving with `content_provenance` unset is a *contract violation* (the producing specialist failed to set a required field). Different failure modes, different handlers.

### Extension 2 — `handoff_reason` enum

Austin's `next_action` is a free-text string. Criterion #2 demands handoff protocols **defined**, not hand-waved. Free-text fields are hand-waving in fancy clothes: they describe what should happen without typing why.

Calibration §3.1.4 surfaced four genuinely distinct back-handoff semantics in Austin's own design — encoded only as English prose in `next_action`. Typing them at the schema layer makes the failure taxonomy load-bearing.

```
forward_normal         — standard forward handoff
forward_urgent         — deadline pressure; receiver should escalate channels
                         (e.g. 04→03 at T-24h: voice contact required, not text)
back_data_missing      — couldn't process; required fields incomplete
                         (e.g. 01 back-handoff on empty raw_inbound)
back_scope_mismatch    — input valid but not my responsibility
                         (e.g. 02 back-handoff on too-broad subject)
back_quality_failure   — output exists but receiver rejects on quality grounds
                         (e.g. 03 sets do_not_send_yet on low-confidence draft)
back_compliance_block  — input would require crossing a regulatory boundary the
                         specialist cannot cross (UPL / IABS / TRELA / TCPA)
                         (e.g. 03 back-handoff when client requests a contract-
                         enforceability opinion → UPL refusal per
                         decisions/domain-research.md §2.4 + §3 / NAR Article 13;
                         04 back-handoff on missing buyer-rep agreement when
                         showing/offer-writing was implicit in the request →
                         TRELA §1101.563 (post-2026-01-01) bar;
                         00 back-handoff on inbound to a represented counterparty
                         → IABS exception 2 + Article 16 bar)
```

`next_action` is retained as the free-text instruction; `handoff_reason` anchors it with a typed cause. Receivers can dispatch on `handoff_reason` deterministically; the prose explains, the type decides.

**Why `back_compliance_block` is its own value, not a sub-case of `back_scope_mismatch`.** Compliance refusals have different downstream behaviour than scope refusals: a scope mismatch routes to whichever specialist *does* own the input; a compliance block routes to a human (attorney, agent, title company) and frequently terminates the system's involvement entirely (the system is not the failure point — the request itself is impermissible for any specialist). Conflating them would force `00_orchestrator` to disambiguate at the routing layer rather than at the schema layer, which loses dispatch precision (the specific anti-pattern §Options(d) below rejects).

**Why `forward_urgent` deserves a TREC-specific rationale.** TREC contract deadlines run on calendar days, not business days, and most internal deadlines (option period in particular — `decisions/domain-research.md` §3) do not extend on weekends/holidays. A T-24h escalation under TREC arithmetic can fall on a Saturday with no buffer; the receiver needs to know that the channel-escalation mandate (voice, not text) is non-negotiable because the calendar gives no slack. National-generic systems assume business days and would fall back to "I'll text them Monday" — wrong by design here. The `forward_urgent` value, paired with `04_transaction_coordinator`'s deadline arithmetic, makes the calendar-day constraint load-bearing at the schema layer.

## Options considered

- **(a) Reinvent from first principles.** Rejected. Reinventing 16 fields in 5 days under deadline against a public artifact that already nails it would ship a weaker schema. Time spent re-deriving Austin's solutions is time not spent on the meta-layer where the real gap exists.

- **(b) Adopt verbatim, no extension.** Rejected. Austin's design has two identifiable engineering gaps: no system-contract invariant (Rule 0 fills), and untyped handoff semantics (`handoff_reason` fills). Shipping verbatim inherits both gaps.

- **(c) Adopt + extend (chosen).** Justified by gap-analysis above. Each extension addresses a specific engineering deficiency, not a perception goal.

- **(d) Adopt a subset.** Rejected. Every field is load-bearing — calibration §3 traces each to a specific failure mode. Removing any field opens a failure mode that the schema currently defends against.

**Other extensions considered and rejected:**

- `next_deadline` field. Rejected: null for ~80% of case lifecycle (pre-contract specialists have no firm deadlines yet); real estate has multiple concurrent deadlines per case (option period, financing, inspection, closing) which a single field can't represent; case state already holds them and specialists can query it. Adding `next_deadline` to the envelope duplicates case state and violates "one fact, one location."

- `provenance_proof` field (cryptographic signatures). Rejected: Diana's 4-person team has no PKI infrastructure. The existing 4-value `content_provenance` enum is sourcing-by-attestation, which is the right fidelity for this use case. Cryptographic proof would add maintenance burden no specialist consumes.

- Field compression (`from`+`to`+`back_to` → `route` object; `content_provenance`+`case_type` → combined `flags`; `parent_envelope_id`+`trail` merged). Rejected: each candidate pair has distinct consumers and distinct change cadences. Merging flattens two quality signals into one and loses dispatch precision.

## Consequences

**Positive.**
- Criterion #2 ("handoff protocols actually defined") clears at-or-above Austin's bar. Typing handoff semantics is a stronger answer to the criterion than Austin's free-text encoding.
- Rule 0 makes contract violations diagnosable at the schema layer instead of the domain layer.
- `handoff_reason` enables deterministic dispatch on back-handoff cause — Lead Qualifier can react differently to `back_data_missing` vs `back_scope_mismatch` without parsing prose.
- Attribution preserves integrity (extending > competing).
- Zero schema invention cost on the 16 core fields preserves the 5-day timeline.

**Negative.**
- Architectural similarity to Austin's submission. The differentiation surface is the `decisions/` folder, not schema divergence. If a judge weights novelty over rigor, this design loses on novelty axis.
- Schema lock-in. The 6-value `handoff_reason` enum (post-C2; was 5) is a public commitment; adding a 7th value later is a schema migration. Mitigation: revisit trigger below. Note: the 5→6 migration was exercised within the drafting cycle itself (added `back_compliance_block`) and bumped `schema_version: 1.0.0 → 1.1.0`. The migration shape is now demonstrated, not just claimed.
- Two extensions to maintain — Rule 0 prose stays in sync with schema field list; `handoff_reason` values stay in sync with refusal taxonomies in each specialist's `handoff.md`. With 6 enum values now copy-pasted across 5 `handoff.md` files (CALL-019), maintenance cost has 6/5 of the baseline; still acceptable, revisit if the enum reaches 7+.

**Revisit triggers.**
- A specialist encounters a back-handoff cause that doesn't fit the 6 `handoff_reason` values. Add a 7th value, increment `schema_version` (next: 1.2.0). The 5→6 migration on 2026-05-12 evening already exercised this trigger — pattern documented under CALL-001.
- Austin retracts or significantly modifies his schema pre-submission.
- Team scales past Austin's 15-agent threshold (parallel handoffs become load-bearing; current schema's single `to` field becomes insufficient).
- Future Comp introduces a handoff requirement neither schema covers (e.g. multi-party concurrent edits, cryptographic provenance proofs).

## Attribution

`HANDOFF_SCHEMA.md` and this ADR cite Austin Johnson (`github.com/eccentricnode/agency-icm`, published 2026-05-12) as the 16-field schema's originator. The submission's README prior-art section names Austin's repo directly. Framing: Austin's 16 fields are the contract; Rule 0 and `handoff_reason` are this submission's engineering additions.

## System-level properties

This ADR is the schema layer of three system-level architecture properties documented in `decisions/architecture-properties.md`:

- **P1 (Closed refusal-taxonomy loop)** — Extension 2's `handoff_reason` enum is the typed-value home for every refusal cause produced by any `handoff.md` (ADR-003). No `handoff.md` may produce an enum value not declared here; no value declared here may exist without a producing site.
- **P2 (Global topology)** — `from`/`to`/`back_to` field semantics are the typed-edge layer that ADR-003's `handoff.md` §3 + §6 declares per-specialist; the union of those declarations forms the system's interaction graph.
- **P3 (Specialist execution loop)** — Rule 0 (this ADR) fires in the OBSERVE step of every specialist's Think → Act → Observe loop.

Cross-reference: `decisions/architecture-properties.md`.
