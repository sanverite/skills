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
| `/sanverite-skills:review-assist <fixed-point>` | Builds a reading plan for the change since a fixed point (branch, SHA, tag, `HEAD~n`), grouped by reading layer: contracts → domain → adapters → entry points → wiring → skim. Writes it to `.scratch/reviews/<branch>.md` and prints a short version. |

More skills (peer-review, review-all, architect) are planned; see the [issues](https://github.com/sanverite/skills/issues).
