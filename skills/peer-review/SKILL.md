---
name: peer-review
description: Send a diff, a spec or a set of tickets to an external reviewer (Codex) as a blind review, then reconcile its findings against the host agent's own review, with a CONFIRMED or DISPUTED verdict on every finding only the external reviewer raised.
disable-model-invocation: true
metadata:
  credits:
    skill: code-review
    author: Matt Pocock
    url: "https://github.com/mattpocock/skills"
---

Get a second opinion from a different vendor's model on a diff, a spec or a set of tickets. The **external reviewer** reviews it **blind**: it never sees the host agent's findings. Both sides work from the same brief. Then **reconciliation** sorts every finding into found by both, external only, and host only, and each external-only finding gets a **verdict**.

Nothing leaves the machine without the user's OK on this run.

Use the vocabulary in `GLOSSARY.md`.

## 1. Pick the artifact

Run every command from the repo root (`git rev-parse --show-toplevel`). Read the tracker setup in `docs/agents/issue-tracker.md` if present; it says how to read issues, sub-issues and blocking edges.

Decide the **artifact** from the arguments, checking in this order:

1. **Nothing passed**: ask "What should I review: a fixed point or PR number (a diff), a spec, or tickets (issue numbers or a tickets folder)?"
2. **A path to a folder**: **tickets**, one per `*.md` file in it (the `.scratch/<feature>/issues/` layout from `to-tickets`).
3. **A path to a file**: a **spec** if it has `## Problem Statement` and `## User Stories`, otherwise one **ticket**.
4. **Several issue numbers** (`#4 #5 6`): **tickets**.
5. **One number** (`#` then digits, or bare digits):
   - `gh pr view <n>` finds a PR: a **diff** in PR mode.
   - Otherwise `gh issue view <n> --json number,title,body,comments`. Its body has `## Problem Statement` and `## User Stories`: a **spec**. Then check for sub-issues (`gh api "repos/{owner}/{repo}/issues/<n>/sub_issues" --jq length`, or the tracker doc's way). If it has any, ask: "Review the spec #<n>, or its <count> tickets?" Tickets means its sub-issues are the tickets, and #<n> is their parent spec.
   - An issue without those sections: one **ticket**.
   - Bare digits that are neither a PR nor an issue but resolve to a commit: a **diff** against that fixed point.
6. **Anything else**: a **diff** against that fixed point.

### Diff

| | Fixed point | PR |
| --- | --- | --- |
| Base | `<ref>` | `<baseRefOid>` |
| Head | `HEAD` | `refs/peer-review/pr-<n>` |

**PR setup.** `gh pr view <n> --json number,title,url,baseRefOid,headRefOid`; on failure stop: "PR #<n> not found, or `gh` isn't signed in (`gh auth login`)." Then fetch both sides without touching the working tree: `git fetch <repo url>.git <baseRefOid> "+pull/<n>/head:refs/peer-review/pr-<n>"`, where `<repo url>` is the PR url without `/pull/<n>`.

**Checks.** Stop with the quoted message if one fails:

- `git rev-parse --verify --quiet "<base>^{commit}"` → "`<base>` doesn't resolve to a commit. Pass a branch, SHA, tag, `HEAD~n` or a PR number."
- `git merge-base <base> <head>` → "`<base>` shares no history with `<head>`, so there's no merge-base to compare against."
- `git diff --name-only <base>...<head>` is non-empty → "No changes between the merge-base of `<base>` and `<head>`. Nothing to review."

The **diff command** for both reviewers is `git diff <base>...<head>`.

### Spec

Read the full spec: the issue body and comments, or the file.

### Tickets

For each ticket, read its title, full body and comments, and its **blockers**: the tracker's native blocking edges (tracker doc), else its `## Blocked by` section or `**Blocked by:**` line. Label each ticket by its issue number (`#7`) or file name (`03-checkout.md`).

Find the **parent spec**: the issue named in the tickets' `## Parent` section or `Part of #<n>` line, the spec issue the user picked them from, or a spec file (with `## Problem Statement` and `## User Stories`) in the folder above a tickets folder. Read it in full. It is context only: its contents aren't reviewed. With no parent spec, skip the coverage check and say so in the report.

## 2. Load the external reviewer

If `docs/agents/peer-review.md` exists, read its settings and skip to the login check.

Otherwise this is the first run in the repo. Codex is the only supported external reviewer. If `command -v codex` fails, stop: "peer-review needs Codex. Install it with `npm install -g @openai/codex`, run `codex login`, then run peer-review again." Otherwise write `docs/agents/peer-review.md`, creating `docs/agents/` if needed:

```markdown
# peer-review

External reviewer: Codex
Review command: codex exec -s read-only --ephemeral -o <output file> -
Login check: codex login status
Login command: codex login

The review brief is passed on stdin, for diffs, specs and tickets alike. Edit the review command to change Codex's flags (keep `-s read-only`), or delete this file to reset it on the next run.
```

Remind the user in the final report to commit it.

**Login check.** If `command -v codex` fails, stop: "Codex isn't installed. Install it with `npm install -g @openai/codex`, then run `codex login`." If the login check exits non-zero, stop: "Codex isn't logged in. Run `codex login`, then run peer-review again."

## 3. Confirm

Ask on every run, before anything is sent, with options **Send** and **Cancel**:

"Send <what> to Codex (OpenAI) for review? It runs in this repo in read-only mode, so it can read any file here, not only <what>. It never sees Claude's findings."

`<what>` is "the diff of `<n>` files (`<base>...<head>`)", "spec #<n>" (or the file name), or "`<count>` tickets (`#4`, `#5`…)" plus "and their parent spec #<n> as context" when there is one.

On **Cancel**, stop: "Nothing was sent." Don't ask for permission again later in the run; this answer covers it.

## 4. Run both reviews

Both reviews run at the same time, and the external reviewer starts before any host finding exists, so its brief cannot contain one.

**Starting the external reviewer.** Write the brief to a file (`mktemp`), then in the background (Bash with `run_in_background`):

```bash
out=$(mktemp -t peer-review.XXXXXX)
codex exec -s read-only --ephemeral -o "$out" - < <brief file>
```

Use the review command from `docs/agents/peer-review.md`, with `<output file>` set to `$out`.

### Diff: host side is Matt's `code-review`

Call the Skill tool for `mattpocock-skills:code-review` (the fully qualified name: Claude Code ships its own `code-review`). Give it `<base>` as the fixed point. In PR mode, `<head>` stands in for `HEAD` everywhere it diffs or logs.

Follow it up to the point where it spawns its sub-agents. By then it has found the spec source and the standards files. **Before** spawning them, start the external reviewer with this brief:

1. "You are reviewing a code change. Don't modify any file. Run `<diff command>` to see it, and `git log <base>..<head> --oneline` for its commits."
2. **Standards**: the standards files code-review found, as paths to read, plus code-review's smell baseline pasted in full, with its two rules (the repo overrides; smells are judgement calls).
3. **Spec**: the spec code-review found (a path to read, or the fetched contents pasted), or "No spec available: skip the Spec axis."
4. "Report every finding as one line: `- [Standards|Spec] <path>:<line> · <hard|judgement> · <finding, under 30 words>`. Group nothing, rank nothing. If there are none, write `No findings.`"

Then let code-review finish: its sub-agents, and its `## Standards` and `## Spec` report. Its findings are the **host findings**.

### Spec or tickets: host side is a sub-agent

Write the brief below, start the external reviewer with it, then spawn one sub-agent (Agent tool, foreground) whose prompt is the same brief, word for word. Its findings are the **host findings**.

The brief, in this order:

1. "You are reviewing <a spec | a set of tickets>. Don't modify any file. You may read the repo for context: the code, `GLOSSARY.md`, `docs/adr/`."
2. The artifact, pasted in full. For tickets: each one under a `### <label>: <title>` heading, with a `Blocked by:` line listing its blockers' labels (or "none").
3. For tickets with a parent spec: the spec pasted under "Parent spec, context only: don't review it."
4. What to look for:
   - **Spec**:
     - **Gap**: something the problem or solution implies that no user story or decision covers, or a question an implementer would have to stop and ask.
     - **Contradiction**: two parts that can't both hold.
     - **Ambiguity**: a statement two implementers would build differently. Give both readings.
     - **Missing story**: an actor, error path, empty state, permission case or edge case with no user story.
   - **Tickets**:
     - **Missing slice**: a parent-spec user story or decision no ticket delivers, or a ticket that delivers none of them. (Only with a parent spec.)
     - **Blocking edge**: a ticket that needs another but doesn't list it, lists one it doesn't need, or sits in a cycle.
     - **Horizontal slice**: a ticket that builds one layer and isn't demoable or verifiable on its own. Wide refactors sequenced as expand, migrate, contract are fine.
     - **Not agent-ready**: a ticket an agent couldn't finish without asking: vague "what to build", acceptance criteria nobody could check, missing context, or too big for one context window.
5. "Report every finding as one line: `- [<kind>] <where> · <finding, under 30 words>`, where `<kind>` is one of the kinds above and `<where>` is a section name or `Story <n>` for a spec, a ticket label or `Story <n>` for tickets. Group nothing, rank nothing. If there are none, write `No findings.`"

### Waiting

Wait for the background run to finish (you are notified; don't poll). Read `$out`: its lines are the **external findings**. If the run failed or `$out` is empty, report the last lines of its output under `## Peer review` as "External review failed", and stop after the host side's report.

## 5. Reconcile

Match a host finding and an external finding when they describe the same issue at the same place: same file and overlapping lines for a diff; same section, story or ticket for a spec or tickets; the same problem in different words. Every finding lands in exactly one group:

- **Found by both**: matched pairs.
- **External only**: external findings with no match. For each one, check the claim against its source: the code (and the spec, for a diff's Spec finding), the spec, or the tickets and parent spec. Give a verdict: **CONFIRMED** (the claim holds) or **DISPUTED** (it doesn't), with a one-line reason pointing at what you read. Disputed findings are reported like the rest.
- **Host only**: host findings with no match.

## 6. Report

For a diff, this comes after code-review's report. For a spec or tickets, it is the whole report:

```markdown
## Peer review

External reviewer: Codex · <what was reviewed> · both <n> · external only <n> (<n> confirmed, <n> disputed) · host only <n>

### Found by both

- [Standards] `src/auth/session.ts:42`: <finding>

### External only

- **CONFIRMED** [Missing slice] Story 12: <finding>. Reason: <one line>
- **DISPUTED** [Blocking edge] #7: <finding>. Reason: <one line>

### Host only

- [Ambiguity] Implementation Decisions: <finding>
```

An empty group reads "None." Keep each finding's kind tag and location. Rank nothing across groups. For tickets with no parent spec, add a line under the summary: "No parent spec found, so missing slices weren't checked."

In PR mode, delete the fetched ref when done: `git update-ref -d refs/peer-review/pr-<n>`.
