# Prior art for `architect`: wayfinder and brainstorming

Researched 2026-11-24 against the primary sources below.

## Question

Do mattpocock's `wayfinder` skill or obra/superpowers' `brainstorming` skill already do what our planned `architect` skill (issue #1, stories 41–59) does? If they do, building architect reinvents the wheel.

## Sources

- [W] https://raw.githubusercontent.com/mattpocock/skills/main/skills/engineering/wayfinder/SKILL.md
- [B] https://raw.githubusercontent.com/obra/superpowers/main/skills/brainstorming/SKILL.md
- [P] https://raw.githubusercontent.com/obra/superpowers/main/skills/writing-plans/SKILL.md
- [G] mattpocock-skills 1.3.1, local: `skills/engineering/grill-with-docs/SKILL.md`, `skills/productivity/grilling/SKILL.md`, `skills/engineering/domain-modeling/ADR-FORMAT.md`
- [I] GitHub issue #1 in this repo (`gh issue view 1`): architect summary, stories 41–59, Implementation Decisions, Out of Scope

## wayfinder (mattpocock)

What it does: plans an effort too big for one session as a map issue plus child "decision tickets" on the issue tracker, then resolves them one per session.

- Scope is "more than one agent session can hold" (frontmatter description) [W].
- It is user-invoked only: `disable-model-invocation: true` [W].
- Domain-agnostic: "engineering work, course content, whatever fits the shape" [W].
- "Plan, don't do": by default it produces "decisions, not deliverables". A map's Notes can override this [W].
- Ticket types are `research`, `prototype`, `grilling` and `task`. Grilling tickets call the Skill tool for "grilling" and "domain-modeling" [W].
- The map's Notes hold "skills every session should consult". When working a ticket it calls "whichever skills the `## Notes` block names" [W].
- If charting surfaces no fog ("small enough for one session"), "you don't need a map. Stop and ask the user" [W].
- No system-design content: no requirement or NFR phases, capacity, storage, failure modes, principles reference, or output docs beyond the tracker issues [W, read in full].

Verdict: it is an orchestrator for oversized efforts, not a design method. It complements architect and does not replace it. Issue #1 already plans this: suggest `/wayfinder` when the work exceeds one session, "with each remaining area becoming a ticket that runs architect for that area" (story 57) [I]. The Notes mechanism in [W] is how a ticket would call architect.

## brainstorming (obra/superpowers)

What it does: a generic intent interview ("what they actually want, and why") for "any creative work", in or out of software [B].

- Before implementing: lay out approaches, write a design doc, get approval, then invoke `superpowers:writing-plans` [B].
- What it covers: why now, success, goals/non-goals/anti-goals, prior art, current state, details. This is "territory, not a script", with no fixed phases [B].
- It does touch the stack. Under "How it gets made" it makes or proposes the "platform, programming language, and libraries" calls, but it decides them from how the user talks, not from recorded requirements [B].
- Design doc sections: intent, goals, constraints, approaches, decided parts of the how, what's left to the builder. It is saved to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` [B].
- A HARD-GATE allows only one next step after approval: "invoke superpowers:writing-plans and no other skill" [B]. writing-plans saves to `docs/superpowers/plans/` and expects subsystems to have been split "during brainstorming" [P].
- It has no NFR numbers, capacity estimates, data/storage/communication/failure phases, principles reference, ADRs, stack table or requirement tracing (absent in [B], read in full).
- It is model-invoked with a broad trigger: "You MUST use this before any creative work" [B].

Verdict: it does not cover architect, and its hard gate to writing-plans conflicts with the Matt chain (grill-with-docs or architect → to-spec → to-tickets). Issue #1 already rules out "a generic brainstorming or ideation skill" [I].

## grill-with-docs / grilling (mattpocock)

- grill-with-docs is only "Call the Skill tool twice, for 'grilling' and 'domain-modeling'", with `disable-model-invocation: true` [G].
- grilling supplies the interview engine: a design tree, rounds, a frontier, numbered questions each with a recommended answer, and facts looked up by sub-agents rather than asked [G].
- domain-modeling supplies the ADR format and the three-criteria test (hard to reverse, surprising, real trade-off). Status and Considered Options are optional there, and there is no Date, benchmark or requirement-link field [G].
- None of it has system-design phases or principles [G].

Verdict: this is the base architect composes on, as planned in issue #1 ("driven by the grilling primitive", ADRs in "Matt's format and three-criteria test") [I].

## Comparison

| Capability | wayfinder | brainstorming | grill-with-docs | architect (planned [I]) |
|---|---|---|---|---|
| Interview format | via grilling tickets [W] | open questions, one at a time [B] | rounds + recommended answers [G] | grilling rounds (story 47) |
| Fixed system-design phase order | no [W] | no ("territory, not a script") [B] | no [G] | FR → NFR → capacity → data → storage → comms → failure → stack |
| NFRs with numbers, capacity estimates | no | no | no | yes (stories 43–44) |
| Principles reference (DDIA + 6 books) | no | no | no | yes (stories 48–49) |
| Stack choice | no | yes, from user's framing [B] | no | yes, only after requirements, traced (46) |
| ADRs | no | no (one design doc) [B] | yes, Matt format [G] | yes, one-way doors only, strict profile: Status, Date, Considered Options, requirement link, benchmark (53, 59a) |
| Architecture doc + stack table | no | `docs/superpowers/specs/` doc [B] | GLOSSARY + ADRs [G] | `docs/architecture.md`, `docs/stack.md` (52, 54) |
| Requirement traceability | no | no | no | every ADR/stack row (55) |
| Brownfield constraints loaded first | n/a | "Look Before You Ask" [B] | not specified [G] | existing ADRs, stack, code (56) |
| Multi-session work | core purpose [W] | splits into sub-projects [B] | no | hands off to wayfinder (57) |
| Next step | decisions on map [W] | writing-plans only [B] | none stated [G] | to-spec (per context/milestone) → to-tickets; replaces grill-with-docs for new systems (58) |
| Model-invokable | no [W] | yes, broad trigger [B] | no [G] | yes (59) |

## Conclusion

Neither skill covers architect. wayfinder is a domain-agnostic multi-session planner with no design content [W]. brainstorming is a generic intent interview that picks a stack without requirement tracing and hard-gates into its own writing-plans flow [B, P]. Architect's specific value, from issue #1, is the phase order, numeric NFRs and capacity, the principles reference, and requirement-traced ADRs and stack table. That value is absent from all three skills.

Build architect as a thin layer: grilling as the engine and domain-modeling for ADRs and terms, plus phases, principles and output docs. Use wayfinder only for oversized work, with architect named in each map's Notes or tickets. Don't depend on brainstorming.

Risk: if superpowers is installed alongside architect, brainstorming's "MUST use this before any creative work" trigger [B] will compete with architect's "I want to build X" trigger (story 59) [I].
