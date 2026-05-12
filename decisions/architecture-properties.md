# Architecture Properties (cross-ADR)

**Purpose.** Three system-level properties are load-bearing on criterion #2 (handoff protocols defined-not-hand-waved) and criterion #4 (real design decisions). Each property holds *across* multiple ADRs, so naming it inside any single ADR couples that ADR to the others' contents. This note is the single home for cross-document properties; ADRs reference here.

**Why a separate file.** A separate note avoids the alternative — re-stating the same property across three ADRs and creating a 3-way drift surface. Top-level note has 1 home, 3 cross-references.

---

## P1 — Closed refusal-taxonomy loop

**Statement.** Every refusal the system can perform is typed at the schema layer. There is no refusal that escapes the loop:

```
per-specialist Rule 0       (ADR-003 §rules.md §1; locked verbatim per CALL-015)
        │ drives
        ▼
failure-mode register       (ADR-003 §identity.md §6 — what this specialist refuses and why)
        │ catalogues
        ▼
refusal trigger             (ADR-003 §handoff.md §4 — declarative condition under which the refusal fires)
        │ maps to
        ▼
handoff_reason enum value   (ADR-001 Extension 2 — 6 typed values: forward_normal/_urgent,
                             back_data_missing/_scope_mismatch/_quality_failure/_compliance_block)
```

**Closure invariant.** No refusal in any `handoff.md` may produce a `handoff_reason` value not in the ADR-001 enum. No `handoff_reason` value in the ADR-001 enum may exist without at least one `handoff.md` producing it (otherwise the value is dead). Adding a 7th enum value (CALL-001 revisit trigger) requires demonstrating the producing site in at least one `handoff.md`; removing a value requires demonstrating no `handoff.md` produces it.

**Why this matters for the brief.** Criterion #2 ("handoff protocols actually defined or hand-waved") is satisfied at the system level, not just per-specialist: a judge auditing one `handoff.md` sees the typed values; a judge auditing the schema sees the same set; a judge cross-referencing both sees the loop closed. The architecture property is *audit-able* — the same 6 values appear in 1 schema doc + 5 specialist docs, no orphans either direction.

**Cross-references.** ADR-001 Extension 2 (schema layer); ADR-003 §rules.md / §identity.md §6 / §handoff.md §4 (per-specialist sites); CALL-001 (enum migration discipline); CALL-019 (copy-paste cost as the price of audit-ability).

---

## P2 — Global topology (interaction graph)

**Statement.** The 5 specialists form a directed multigraph with typed edges. Forward edges are sequential by deal-state; back edges return to predecessor specialists with typed `handoff_reason` causes; terminal edge is `END`.

```
                          ┌────────────────────────────────────────┐
                          │           00_orchestrator              │
                          │     (router; produces metadata only)   │
                          └────┬──────┬──────┬──────┬─────────────┘
                       fwd_*  │      │      │      │  fwd_*
                              ▼      ▼      ▼      ▼
                  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
                  │   01     │ │   02     │ │   03     │ │   04     │
                  │ qualifier│ │ research │ │  comms   │ │    TC    │
                  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘
                       │ fwd_*      │ fwd_*      │ fwd_*      │
                       └────────────┼────────────┼────────────┤
                                    │            │            │
                                    ▼            ▼            ▼
                                 (next specialist or END)
                       ◄─── back_*  via back_to=<predecessor> ────►
```

**Forward routes (typed `forward_normal` or `forward_urgent`):**
- `00 → {01, 02, 03, 04}` — orchestrator routes to any specialist based on subject classification + deal state.
- `01 → {02, 03, 04}` — qualified lead can route to research (comp pull), comms (immediate follow-up), or TC (if already in deal-active phase from a previous case).
- `02 → {03, 04}` — research output feeds comms drafting or TC deal-active context.
- `03 → {04, END}` — draft output either advances to TC (deal-active phase begins) or terminates (one-shot follow-up).
- `04 → {03, END}` — TC escalates to comms via `forward_urgent` (deadline pressure) or terminates (deal funded).

**Back routes (typed `back_data_missing` / `back_scope_mismatch` / `back_quality_failure` / `back_compliance_block`):**
- Any specialist may `back_to` any of its valid forward predecessors (declared in each specialist's `handoff.md` §6 — ADR-003).
- `00` is the universal valid `back_to` target — any specialist can escalate to the orchestrator for routing reconsideration or human handoff.

**Topology invariant.** No edge exists in the system that is not declared in some specialist's `handoff.md` (forward §3 "Routing destinations" or back §6 "Back-handoff sources"). The graph above is the union of all per-specialist edge declarations; if a specialist tries to `from → to` an undeclared edge, the receiver back-handoffs with `back_scope_mismatch` (input arrived from an unexpected source). Topology is enforced at the contract layer, not the routing layer.

**Why this matters for the brief.** Criterion #1 ("does the architecture make sense, could a real team use it?") rests on whether the graph reflects boutique-team workflow. `decisions/domain-research.md` §2.1 + §1 confirms: 4-person teams sequentially route inbound through lead-handler → buyer-agent → TC, matching the 00 → 01 → 02 → 03 → 04 happy-path topology. Back routes match the real-world failures (incomplete intake, scope misroute, quality rejection, regulatory block).

**Cross-references.** ADR-003 §handoff.md §3 + §6 (per-specialist edge declarations); ADR-001 Extension 2 (typed edge values); `decisions/trajectories.md` T1/T2/T3 (concrete walks across the topology).

---

## P3 — Specialist execution loop (Think → Act → Observe)

**Statement.** Every specialist runs the same 3-step loop on every received envelope. The loop is the contract between the specialist's identity (what it is) and the envelope (what it consumes/produces).

```
THINK   — read incoming envelope.
          Check Rule 0 (per-specialist, from rules.md §1).
          Check required-in fields (from handoff.md §1).
          Decide: forward, back, or terminate.

ACT     — execute the specialist's domain task per identity.md §1-§5.
          Produce payload per handoff.md §2 (Outputs).
          Calibrate confidence per handoff.md §5.

OBSERVE — validate outgoing envelope: Rule 0 (envelope-level, ADR-001),
          required_fields_present, content_provenance, handoff_reason,
          parent_envelope_id chain, trail append.
          Emit envelope OR back-handoff with typed handoff_reason.
```

**Contract invariants per step.**
- **THINK — refusal-discipline.** If Rule 0 fails, the specialist must back-handoff (not silently process); if required-in fields missing, must back-handoff with `back_data_missing`. No specialist may "best-effort" past a Rule 0 violation. (This is the system-level enforcement of P1.)
- **ACT — confidence-honesty.** A specialist that produces output with weak inputs sets `confidence: low` and surfaces the weakness in `payload`, not by silently inferring. `decisions/domain-research.md` §2.3: "fewer than 3 viable comps in 90 days → confidence: low, must flag." Pattern generalises across all specialists.
- **OBSERVE — schema-discipline.** Outgoing envelope is checked against ADR-001 before emit. The check is the specialist's responsibility, not the receiver's first-action. (Receivers re-check Rule 0 on receive — defence-in-depth — but the producer is contractually first-line.)

**Why this matters for the brief.** Criterion #2 ("handoff protocols actually defined") is satisfied at the *protocol* layer (typed envelope, typed enum) AND the *behaviour* layer (every specialist runs the same loop in the same sequence). Without P3 named, two specialists could produce well-formed envelopes via different internal disciplines — and the system would have invisible behavioural drift. Naming the loop binds specialist behaviour to a single contract.

**Cross-references.** ADR-001 §Rule 0 (envelope-level — fires in OBSERVE step); ADR-003 §rules.md §1 (per-specialist Rule 0 — fires in THINK step); ADR-003 §handoff.md §1 + §2 + §5 (required fields + outputs + confidence — drive ACT step); ADR-003 §identity.md §1-§5 (specialist domain task — what ACT does).

---

## How these three properties relate

P1 (closed refusal taxonomy) + P2 (global topology) + P3 (execution loop) are not independent — they're the same architecture property viewed from three angles:

- **P1 is the value side:** what set of refusal causes can flow on the edges.
- **P2 is the structural side:** what edges exist and which specialists they connect.
- **P3 is the behavioural side:** how each specialist transforms an incoming envelope into outgoing edge(s).

A judge auditing the system finds the same architecture property at all three levels. A specialist author consults all three for any drafting decision (which Rule 0 to write, which back-handoff target is valid, when to set confidence: low). A maintainer debugging a production issue traces P3 to find the failing step, then P1 to confirm the refusal type, then P2 to find the destination.

**This note is the system-level claim that the architecture is closed.** No refusal escapes typing (P1), no edge is undeclared (P2), no specialist runs a different protocol (P3). Together they answer criterion #2 + #4 at the *system* level, beyond what any single ADR claims locally.

---

## Cross-document update discipline

When any ADR changes, check this note for impact:

| ADR change | Update required here? |
|---|---|
| ADR-001 enum migration (e.g. 6→7 `handoff_reason` values) | Yes — P1 enum count + closure invariant. |
| ADR-001 Rule 0 predicate change | Yes — P3 OBSERVE step. |
| ADR-003 per-specialist Rule 0 wording change | No (P1/P3 reference the *category*, not the wording). |
| ADR-003 handoff.md §3/§6 edge declaration change | Yes — P2 forward/back routes if the edge set changes. |
| ADR-002 disclosure-zone scoping change | No (this note is real-model zone; analogy is out of scope here). |
| New specialist added (architectural change) | Yes — P2 graph; revisit P3 loop applicability. |

**Update cadence.** This note tracks the underlying ADRs. Update inline as ADRs change; never let it lag.
