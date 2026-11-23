# ai-skills

Personal skills that sit between the steps of Matt Pocock's main flow (grill-with-docs → to-spec → to-tickets → implement → code-review → retro) to fill gaps it leaves.

## Language

### Review

**Host agent**:
The agent running the skill (e.g. Claude in Claude Code), as opposed to the external reviewer.
_Avoid_: main model, Claude (when the host could be another agent)

**External reviewer**:
An AI agent from a different vendor or model than the host agent (e.g. Codex, CodeRabbit), run on the user's explicit authorization.
_Avoid_: second model, other AI, peer

**Peer review**:
A review of one artifact (spec, tickets, or diff) by an external reviewer, done blind, then reconciled against the host agent's own findings.
_Avoid_: cross-review, second opinion

**Blind review**:
A review in which the external reviewer never sees the host agent's findings for the same artifact.
_Avoid_: independent review

**Reconciliation**:
Sorting peer-review findings into three buckets: found by both, found only by the external reviewer, found only by the host agent.
_Avoid_: merge, dedupe

**Verdict**:
The host agent's label on a finding only the external reviewer raised: CONFIRMED or DISPUTED, each with a one-line reason. Disputed findings are still reported.
_Avoid_: false positive, rejected

**Reading plan**:
An ordered list of steps a human reviewer follows to read a change line by line, grouped by reading layer, ordered by imports within a layer, and flagged by risk. Each test is read just before the code it tests.
_Avoid_: review summary, walkthrough

**Change surface**:
The outward-facing things a change touches, directly or through their callers: APIs, exported functions, pages, components, hooks, stores, the API clients it calls, CLI commands, jobs, schemas, config keys. Listed at the top of a reading plan.
_Avoid_: affected list, impact summary

**Reading layer**:
One band of a reading plan: contracts, domain, adapters, entry points, wiring, then skim. A repo's own layering overrides these defaults.
_Avoid_: tier, phase

### Design

**Requirement**:
A functional or non-functional need (with rough numbers where non-functional) recorded in the architecture doc. Every stack choice must trace back to at least one.
_Avoid_: feature, ask

**Constraint**:
An existing, already-decided choice (from prior ADRs, the stack table, or the code) that the architect treats as fixed rather than re-deciding.
_Avoid_: legacy, given

**Stack table**:
The single table recording easily reversible technology choices (choice, alternatives, why, requirement). One-way-door choices get an ADR instead.
_Avoid_: tech radar, stack list
