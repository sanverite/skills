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

All skills below are user-invoked: they run only when you type them.

| Skill | What it does |
| --- | --- |
| `/sanverite-skills:review-assist [fixed-point \| PR number] [--out <folder>]` | Builds a reading plan for a change: your uncommitted work (no argument), the change since a fixed point (branch, SHA, tag, `HEAD~n`), or a PR. Steps are grouped by reading layer (contracts → domain → adapters → entry points → wiring → skim), ordered by imports, with each test read just before its code. Every step has a risk flag, clickable `file:line` links, what changed and what to check. The plan opens with the change surface (affected APIs, functions, pages, components…), with cURLs for every affected endpoint the repo serves and test hints for pages. Writes `<plan folder>/<name>.md` and prints a short version. On the first run in a repo it asks for the plan folder (default `.scratch/reviews/`), offers to gitignore it, and saves the choice to `docs/agents/review-assist.md`; pass `--out <folder>` to override it for one run. |

More skills (peer-review, review-all, architect) are planned; see the [issues](https://github.com/sanverite/skills/issues).
