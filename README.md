# sanverite-skills

Companion skills for [Matt Pocock's skills](https://github.com/mattpocock/skills). They sit between the steps of his main flow (grill-with-docs → to-spec → to-tickets → implement → code-review → retro) and fill gaps it leaves.

**Not an official Matt Pocock plugin.** It's a separate plugin by [sanverite](https://github.com/sanverite) that builds on his work by composition: it calls his skills by name and never copies or modifies them (see [ADR 0001](docs/adr/0001-extend-by-composition-not-fork.md)). Credit for the flow and the skills it extends goes to Matt Pocock.

## Requirements

- [Claude Code](https://claude.com/claude-code)
- `mattpocock-skills` **1.3.1 or later**, installed alongside:

  ```bash
  claude plugin install mattpocock-skills@claude-plugins-official
  ```

## Install

In Claude Code:

```
/plugin marketplace add sanverite/skills
/plugin install sanverite-skills@sanverite
```

## Skills

review-assist and peer-review are user-invoked: they run only when you type them. architect can also be picked up by the agent when you say you want to build something new.

| Skill | What it does |
| --- | --- |
| `/sanverite-skills:review-assist [fixed-point \| PR number] [--out <folder>]` | Builds a reading plan for a change: your uncommitted work (no argument), the change since a fixed point (branch, SHA, tag, `HEAD~n`), or a PR. Steps are grouped by reading layer (contracts → domain → adapters → entry points → wiring → skim), ordered by imports, with each test read just before its code. Every step has a risk flag, clickable `file:line` links, what changed and what to check. The plan opens with the change surface (affected APIs, functions, pages, components…), with cURLs for every affected endpoint the repo serves and test hints for pages. Writes `<plan folder>/<name>.md` and prints a short version. On the first run in a repo it asks for the plan folder (default `.scratch/reviews/`), offers to gitignore it, and saves the choice to `docs/agents/review-assist.md`; pass `--out <folder>` to override it for one run. |
| `/sanverite-skills:architect [what to build]` | Requirements-first system design for a new app, service, API or subsystem. Interviews you in a fixed order (functional requirements → non-functional requirements with numbers → capacity → data model → storage → communication → failure modes → stack), recommending answers from a bundled principles checklist that cites DDIA and seven other books. Writes `docs/architecture.md` (requirements, estimates, diagrams, data model, adapter boundaries, testing strategy), ADRs for hard-to-reverse choices and `docs/stack.md` for the rest, each with alternatives, date and the requirement it serves. Then points you to `/to-spec` → `/to-tickets`. |
| `/sanverite-skills:peer-review <fixed-point \| PR number \| spec \| tickets>` | Sends a diff, a spec or a set of tickets to an external reviewer (Codex, run read-only) as a blind review. A diff is reviewed against the same two axes as Matt's `code-review` (Standards and Spec), while `code-review` runs alongside. A spec (issue number or file) is checked for gaps, contradictions, ambiguities and missing user stories; tickets (issue numbers, a tickets folder, or a spec's sub-issues) for missing slices, wrong blocking edges, horizontal slices and tickets that aren't agent-ready, with the parent spec as context. For documents, a host sub-agent gets the identical brief in parallel. Asks before anything is sent, every run. Reconciles the findings into found by both, external only and host only, and checks every external-only finding against the code: CONFIRMED or DISPUTED, with a reason. Codex is the only external reviewer for now. On the first run in a repo it saves Codex's review command to `docs/agents/peer-review.md`, which you can edit. Needs Codex installed and logged in (`codex login`). |

More skills (review-all) are planned; see the [issues](https://github.com/sanverite/skills/issues).
