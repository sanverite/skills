---
name: architect
description: "System design for a new app, service, API or subsystem: interview requirements first, derive the tech stack from them, and write the architecture doc, ADRs and stack table. Use when the user wants to build something new or asks which tech stack to use. For system design, prefer this over a generic brainstorming skill."
metadata:
  credits:
    skills: grilling, domain-modeling, codebase-design, research, prototype
    author: Matt Pocock
    url: "https://github.com/mattpocock/skills"
---

Design a system the way an experienced architect would: settle **requirements** first, then let every technology choice follow from them. Each choice traces to a requirement, and every decision is written down with what, why, when, and what was rejected.

The output is design, not implementation: decisions, numbers, diagrams and boundaries. Once code exists, the code and tests are the record of what it does, so nothing here describes individual tests, schemas or adapters.

Use the vocabulary in `GLOSSARY.md`: **requirement**, **constraint**, **stack table**.

## 1. Load what exists

Run from the repo root (`git rev-parse --show-toplevel`). Read, where present: `GLOSSARY.md` or `GLOSSARY-MAP.md`, `docs/architecture.md`, `docs/stack.md`, every ADR in `docs/adr/`.

Each decision found there is a **constraint**: treat it as fixed, never re-ask it.

Read [`PRINCIPLES.md`](PRINCIPLES.md) in full. It holds, per topic, the trade-off questions and requirement signals the interview draws on.

## 2. Interview

Call the Skill tool for `grilling` and `domain-modeling`. Grilling supplies the format: rounds, numbered questions, a recommended answer for each. Architect adds a fixed **phase order** on top. The frontier never reaches past the current phase.

| # | Phase | Settles | Principles topics |
|---|---|---|---|
| 1 | Functional requirements | What the system does, for whom, core flows | — |
| 2 | Non-functional requirements | Ranked quality attributes and SLOs, each with a number | Quality attributes; SLOs |
| 3 | Capacity estimates | Peak RPS, read/write ratio, storage per year, bandwidth | Capacity estimation |
| 4 | Data model | Entities, relationships, hot-path access patterns | Data models |
| 5 | Storage | Engine type, replication, partitioning, consistency and transactions | Storage engines; Replication; Partitioning; Transactions and consistency |
| 6 | Communication | Sync vs async, delivery guarantees, realtime transport, service boundaries | Streams and logs; Service boundaries and data ownership |
| 7 | Failure modes | What each dependency does when slow or down, overload behaviour | Stability patterns; Overload and load shedding |
| 8 | Stack | Architecture style, concrete technologies, adapter boundaries, testing strategy | Architecture styles |

A phase is done when every principles topic it lists has either a settled answer or a recorded reason it doesn't apply (no requirement touches it). Then record the phase (step 3) and move on.

**Requirements carry ids**: `FR-1`, `FR-2` for functional, `NFR-1` for non-functional. Every non-functional requirement has a number (p99 under 200 ms, 99.9% monthly, 10k concurrent users). An adjective is not a requirement yet: ask for the number.

**Every recommended answer from phase 2 on cites** the requirement ids it serves and the principle it rests on: `➡️ Single leader with async read replicas (NFR-2, NFR-4; DDIA 5)`.

**Stack phase.** Ask only questions whose answer serves a recorded requirement, and name the ids in the question. A choice no requirement asks for is either unneeded or a sign a requirement is missing: ask which. Then:

- Draw one **adapter boundary** per external technology (datastore, queue, third-party API): the port the core depends on, and the adapter behind it. Call the Skill tool for `codebase-design` for where seams go and how deep each module is.
- Settle the **testing strategy**: which levels exist (unit, integration, end-to-end, contract, load), what each one proves, and which requirements the load and failure tests check.

**Delegate**:

- Domain terms and **bounded contexts** go through `domain-modeling`. More than one context means a `GLOSSARY-MAP.md` with a GLOSSARY and `docs/adr/` per context.
- Facts that are uncertain or likely stale (current versions, managed-service limits, pricing, published benchmarks): call the Skill tool for `research`. Everything else comes from `PRINCIPLES.md` and the interview.
- A one-way-door choice that hinges on a number nobody has published (write throughput on this access pattern): call the Skill tool for `prototype` for a throwaway benchmark. Record the result in the ADR.

## 3. Record

Write as each phase settles, so nothing waits on the end of the session. Today's date comes from `date +%F`.

### `docs/architecture.md`

Sections, in this order, each filled as its phase settles:

1. **Requirements**: tables `| ID | Requirement | Number |` for FR and NFR.
2. **Capacity estimates**: the arithmetic, not only the results.
3. **Components**: a Mermaid component diagram.
4. **Flows**: a Mermaid flowchart or sequence diagram for each flow whose order or failure path isn't obvious.
5. **Data model**: entities, relationships, and the access patterns each one serves. Entities and fields, not DDL.
6. **Communication**: sync and async paths, delivery guarantees, realtime transport.
7. **Failure modes**: per dependency, what happens when it is slow or down, and the pattern that handles it.
8. **Adapter boundaries**: per external technology, its port and adapter.
9. **Testing strategy**: levels, what each proves, requirements covered.
10. **Decisions**: links to the ADRs and to `docs/stack.md`.

### ADRs

Each choice gets Matt's three-criteria test (in `domain-modeling`): hard to reverse, surprising without context, a real trade-off. If it passes all three, write an ADR. Number it after the highest in the target `docs/adr/` (system-wide decisions in the root folder, a single context's decisions in that context's folder).

Architect's ADRs are a stricter version of Matt's format. Every field is required, except Benchmark, which is required only when one was cited or run:

```md
---
status: accepted
date: YYYY-MM-DD
---

# {Short title of the decision}

{1-3 sentences: context, decision, why.}

**Requirements**: FR-n, NFR-n (docs/architecture.md)

## Considered Options

- **{Chosen}**: why it wins against the requirements.
- **{Alternative}**: why not.

## Benchmark

{Source and number, or the prototype run and its result.}
```

When a decision replaces an existing ADR, set the old one's status to `superseded by ADR-NNNN`.

### `docs/stack.md`

Every choice that fails the three-criteria test goes here, one row each:

```md
| Choice | Alternatives | Why | Requirements | Date | Status |
|---|---|---|---|---|---|
| Redis for session cache | Memcached, in-process | TTLs and persistence on restart | NFR-1, NFR-3 | YYYY-MM-DD | accepted |
```

A row that is replaced keeps its place, with status `superseded by <choice>`, and the new row goes below it.

## 4. Hand off

When phase 8 is recorded, check: every ADR and every stack row names at least one requirement, and every NFR is covered by at least one decision or marked as needing none. Fix any gaps before handing off.

Then tell the user what was written (paths) and the next step: run `/to-spec` in this session, once per bounded context or milestone, then `/to-tickets` on each spec. The interview covered what grill-with-docs would, so it isn't needed.
