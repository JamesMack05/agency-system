# HANDOFF_SCHEMA.md — Inter-specialist envelope contract

The single structured document that travels with every handoff between specialists. 16 core fields from Austin Johnson's `eccentricnode/agency-icm` (published 2026-05-12; attribution preserved), plus two engineering extensions: an envelope-level Rule 0 invariant and a typed `handoff_reason` enum.

**Current `schema_version`:** `1.1.0`.

Live in: every `<specialist>/handoff.md` consumes and produces envelopes shaped by this schema.

For the full design justification of the 16-field choice + the two extensions, see `decisions/ADR-001-envelope-schema.md`. This doc is the contract reference — terse, readable in 2 minutes, no rationale.

---

## The 16 core fields

| # | Field | Purpose | Required? | Example value |
|---|---|---|---|---|
| 1 | `schema_version` | Migration anchor (semver) | Always | `"1.1.0"` |
| 2 | `case_id` | Stable identity for the deal across hops; never reassigned | Always | `"CASE-2026-0042"` |
| 3 | `from` | Producing specialist folder name | Always | `"01_lead_qualifier"` |
| 4 | `to` | Receiving specialist folder name, or `END` | Always | `"02_property_research"` |
| 5 | `back_to` | Populated on back-handoff; null on forward | Conditional | `null` (forward) or `"01_lead_qualifier"` (back) |
| 6 | `timestamp` | ISO 8601 instant of envelope production | Always | `"2026-05-12T14:48:21Z"` |
| 7 | `agent_on_deal` | Drives voice-file selection in `03_client_communication`. Producing site: `01_lead_qualifier` (per CALL-022). | Conditional | `"agent_diana"` (named, from assignment table A15) or `"team_lead"` (sentinel default, resolves to `voice/team_lead.md`) or `null` (only on 00→01 emission before producing site fires) |
| 8 | `payload` | Shape owned by producing specialist's `handoff.md` §2 | Always | (see per-specialist `handoff.md` files) |
| 9 | `required_fields_present` | Receiver-side validation result | Always | `true` |
| 10 | `confidence` | Drives downstream gating. On forward: confidence in the payload's primary claim (qualified_lead, research brief, draft). On back-handoff: confidence in the refusal cause. | Always | `"low"` / `"med"` / `"high"` |
| 11 | `next_action` | Declarative free-text instruction to receiver | Always | `"Pull 3-5 comps within 0.5mi past 90 days."` |
| 12 | `trail` | Append-only audit log of every hop | Always | (array — see below) |
| 13 | `parent_envelope_id` | CAS-style concurrency token; references the previous envelope in the chain | Conditional | `"ENV-2026-0042-002"` or `null` (first envelope) |
| 14 | `content_provenance` | 4-value enum driving quarantine behaviour | Always | `"anonymous_inbound"` / `"verified_client"` / `"agent_authored"` / `"system_generated"` |
| 15 | `case_type` | 6-value enum driving conditional logic | Always | `"residential_buyer_side"` / `"residential_seller_side"` / `"residential_both"` / `"estate"` / `"divorce"` / `"foreclosure"` |
| 16 | `linked_case_ids` | Cross-case references (concurrent buy+sell, etc.) | Always | `[]` or `["CASE-2026-0040"]` |

**Plus conditional jurisdiction field:**

- `intermediary_status` (TX TRELA flag) — present when jurisdiction requires; not part of core 16. Values: `"yes"` / `"no"` / `null` (out of jurisdiction).

---

## Extension 1 — Envelope-level Rule 0

> **Rule 0.** No `schema_version` → no processing. No `case_id` → no processing. An envelope where `content_provenance: anonymous_inbound` is quarantined for sender-identity verification before processing.

Rule 0 is checked **before** any other validation. A receiving specialist that processes an envelope failing Rule 0 has violated the contract.

**Layering note.** Envelope-level Rule 0 governs whether an envelope is *processable*. ADR-003's per-specialist Rule 0 governs whether the specialist's task is *performable* on an otherwise-valid envelope. The two layers are not duplicative — see `decisions/ADR-001-envelope-schema.md` §Extension 1 for the full layering principle and `decisions/architecture-properties.md` §P3 for the system-level enforcement claim.

---

## Extension 2 — `handoff_reason` enum

Every envelope carries a typed `handoff_reason`. 6 values:

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
```

Receivers dispatch on `handoff_reason` deterministically; the `next_action` prose explains, the type decides.

**Closed taxonomy.** Every refusal a specialist can perform produces one of these 6 values. No `handoff.md` may produce a `handoff_reason` value not in this list; no value in this list may exist without a producing site. Closure is documented at `decisions/architecture-properties.md` §P1.

---

## `trail` shape

Append-only array. One entry per envelope-emit:

```yaml
trail:
  - {at: 2026-05-12T14:32:08Z, by: 00_orchestrator,      event: "routed forward_normal to 01_lead_qualifier"}
  - {at: 2026-05-12T14:48:21Z, by: 01_lead_qualifier,    event: "qualified high; routing to 02_property_research"}
  - {at: 2026-05-12T16:12:55Z, by: 02_property_research, event: "research complete; routing to 03_client_communication"}
```

Each producing specialist appends its own entry on emit. The receiver never modifies prior entries. The full chain is the deal's audit log.

---

## Minimal envelope example

A first-touch routing envelope from `00_orchestrator → 01_lead_qualifier`:

```yaml
schema_version: 1.1.0
case_id: CASE-2026-0042
from: 00_orchestrator
to: 01_lead_qualifier
back_to: null
timestamp: 2026-05-12T14:32:08Z
agent_on_deal: null              # 00 emits null because 01 is the producing site (per CALL-022); 01 will set this on case_type resolution during qualification
payload:
  inbound_channel: web_form
  inbound_text: "Hi, saw your Tarrytown listing on Zillow, can someone reach out?"
  sender_handle: sarah.chen.zillow.relay@zillow.example
  subject_classification: qualification
  inbound_property_ref: MLS-9182734
  urgency: normal
required_fields_present: true
confidence: med                  # anonymous inbound, identity unverified
next_action: "Qualify lead — capture intent, timeline, budget+pre-approval, geography, current rep status, buyer-rep signed status."
trail:
  - {at: 2026-05-12T14:32:08Z, by: 00_orchestrator, event: "routed forward_normal to 01_lead_qualifier"}
parent_envelope_id: null         # first envelope in case
content_provenance: anonymous_inbound
case_type: residential_buyer_side
linked_case_ids: []
handoff_reason: forward_normal
intermediary_status: null
```

Two more worked envelopes (back-handoff + `forward_urgent`) are in `decisions/trajectories.md` T2 and T3.

---

## Migration

Schema versions are semver (`MAJOR.MINOR.PATCH`). Migrations within `1.x.x` preserve receiver compatibility; a `2.0.0` bump is a breaking change.

Historic migrations:

- `1.0.0 → 1.1.0` (2026-05-12): Added `back_compliance_block` as 6th `handoff_reason` value (was 5). Triggered by adversarial review (Council-Verdict §C2) surfacing UPL/IABS/TRELA refusals that did not fit existing values. All 5 specialist `handoff.md` files re-validated; no breaking field changes — only enum extension.

For migration procedure on next bump (`1.2.0`+), see `decisions/design-calls.md` CALL-001.

---

## Attribution

The 16-field core schema is adopted verbatim from Austin Johnson's [`eccentricnode/agency-icm`](https://github.com/eccentricnode/agency-icm) (published 2026-05-12). Extensions 1 (Rule 0) and 2 (`handoff_reason` enum) are this submission's engineering additions. See `decisions/ADR-001-envelope-schema.md` for the full justification of each extension.
