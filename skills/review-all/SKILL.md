---
name: review-all
description: One command for a full review of a change (fixed point or PR number) - Matt's code-review and a peer review by Codex in parallel, reported side by side, then a reading plan with every finding pinned to the step where that code is read.
disable-model-invocation: true
metadata:
  credits:
    skill: code-review
    author: Matt Pocock
    url: "https://github.com/mattpocock/skills"
---

Run the host review and the **peer review** of a change together, report them side by side, then build a **reading plan** with each finding pinned to the step where the reviewer reads that code. A thin wrapper: it adds nothing to either review and never edits their findings.

It follows two sibling skills of this plugin, by reading their files: peer-review and review-assist are user-invoked, so the Skill tool can't call them. Their paths are relative to this skill's base directory.

Use the vocabulary in `GLOSSARY.md`.

## 1. Pick the change

The argument is a **fixed point** (branch, SHA, tag, `HEAD~n`) or a **PR number**, plus an optional `--out <folder>`. Take `--out <folder>` out first; it goes to review-assist in step 3. With no change given, ask: "Which change should I review: a fixed point (branch, SHA, tag, `HEAD~n`) or a PR number?"

## 2. Both reviews

Read [`../peer-review/SKILL.md`](../peer-review/SKILL.md) and follow it for this change in diff mode. It runs Matt's `code-review` and the external reviewer in parallel, and reports `## Standards`, `## Spec` (from `code-review`), then `## Peer review`. Keep `code-review`'s two sections verbatim; combine and rerank nothing.

If peer-review stops at its change checks (a ref that doesn't resolve, no merge-base, an empty diff), stop with its message. Two other stops are overridden, because nothing has been sent and the host agent's review can still run:

- The user answers **Cancel** at the confirmation, or Codex isn't installed or isn't logged in. Call the Skill tool for `mattpocock-skills:code-review` with the same fixed point, diffing against peer-review's `<head>` in PR mode. Report its `## Standards` and `## Spec`, then `## Peer review` as one line: "Skipped: <you declined | Codex isn't installed | Codex isn't logged in, run `codex login`>." In PR mode, delete peer-review's fetched ref: `git update-ref -d refs/peer-review/pr-<n>`.
- The external run fails after it was sent: peer-review already reports "External review failed" and keeps code-review's report. Carry on.

Keep the **findings** for step 4: every finding in Standards and Spec, and every finding in the peer review's External only group, with its verdict and kind tag. (Found by both and Host only repeat Standards and Spec findings, so they add none.)

## 3. Reading plan

Read [`../review-assist/SKILL.md`](../review-assist/SKILL.md) and follow it for the same change through its step 7 (the plan check), passing `--out <folder>` if given. It picks the plan folder (asking on the first run in a repo), writes the plan with its change surface, and checks it. Hold its step 8 (terminal report, and deleting its PR ref) until step 5 here: step 4 runs the plan check again, which needs that ref.

## 4. Pin the findings

A **pinned finding** sits under the plan step whose file entries include the finding's file. When that file appears in more than one step, use the step whose line ranges contain the finding's line, else the one nearest it. Add a `Findings:` list as the step's last sub-item (after `Check:`, or after the file entry for a skim or deleted file):

```markdown
  - Findings:
    - [Standards] src/auth/session.ts:42: <finding>
    - [Spec] src/auth/session.ts:60: <finding>
    - [Peer review · Standards] **CONFIRMED** src/auth/session.ts:88: <finding>. Reason: <verdict reason>
```

Write every file path in a finding without backticks, including any the reviewer wrapped in backticks, so the exactly-once plan check still counts each file once.

A finding with no file, or whose file isn't in the plan (a Spec finding about something missing, a file the change didn't touch), goes in one trailing step after the last layer:

```markdown
---

## Unpinned

- [ ] **<next step number>. Findings with no step**
  - Findings:
    - [Spec] <finding>
```

Leave out `## Unpinned` when every finding was pinned. Leave the change surface and every existing step untouched apart from the added `Findings:` lists.

Run review-assist's plan check (its step 7) again against the plan. It must print nothing.

## 5. Report

After the three review sections, do review-assist's step 8: its terminal report, which gives the plan's path, and in PR mode deleting its fetched ref.
