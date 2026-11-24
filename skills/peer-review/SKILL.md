---
name: peer-review
description: Send a diff to an external reviewer (Codex) as a blind review, then reconcile its findings against Matt's code-review, with a CONFIRMED or DISPUTED verdict on every finding only the external reviewer raised.
disable-model-invocation: true
metadata:
  credits:
    skill: code-review
    author: Matt Pocock
    url: "https://github.com/mattpocock/skills"
---

Get a second opinion on a change from a different vendor's model. The **external reviewer** reviews the diff **blind** (it never sees the host agent's findings), against the same two axes as Matt's `code-review`: Standards and Spec. Then **reconciliation** sorts every finding into found by both, external only, and host only, and each external-only finding gets a **verdict**.

Nothing leaves the machine without the user's OK on this run.

Use the vocabulary in `GLOSSARY.md`.

## 1. Pick the change

Run every command from the repo root (`git rev-parse --show-toplevel`).

The argument is a **fixed point** (branch, SHA, tag, `HEAD~n`) or a **PR number** (`#` then digits, or bare digits that `gh pr view` finds). With no argument, ask: "Which change should I review: a fixed point (branch, SHA, tag, `HEAD~n`) or a PR number?"

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

## 2. Load the external reviewer

If `docs/agents/peer-review.md` exists, read its settings and skip to the login check.

Otherwise this is the first run in the repo. Check which supported external reviewers are installed: `command -v codex`. Codex is the only one supported so far.

- None installed: stop. "No supported external reviewer found. Install Codex (`npm install -g @openai/codex`), run `codex login`, then run peer-review again."
- Otherwise ask: "Which external reviewer should peer-review use in this repo?", listing the installed ones. Then write `docs/agents/peer-review.md`, creating `docs/agents/` if needed:

  ```markdown
  # peer-review

  External reviewer: Codex
  Review command: codex exec -s read-only --ephemeral -o <output file> -
  Login check: codex login status
  Login command: codex login

  The review brief is passed on stdin. Edit these lines to change the reviewer or its command, or delete this file to be asked again on the next run.
  ```

  Remind the user in the final report to commit it.

**Login check.** If `command -v codex` fails, stop: "Codex isn't installed. Install it with `npm install -g @openai/codex`, then run `codex login`." If the login check exits non-zero, stop: "Codex isn't logged in. Run `codex login`, then run peer-review again."

## 3. Confirm

Ask on every run, before anything is sent, with options **Send** and **Cancel**:

"Send this change to Codex (OpenAI) for review? It runs in this repo in read-only mode, so it can read any file here, not only the diff of `<n>` files (`<base>...<head>`). It never sees Claude's findings."

On **Cancel**, stop: "Nothing was sent." Don't ask for permission again later in the run; this answer covers it.

## 4. Run both reviews

Call the Skill tool for `mattpocock-skills:code-review` (the fully qualified name: Claude Code ships its own `code-review`). Give it `<base>` as the fixed point. In PR mode, `<head>` stands in for `HEAD` everywhere it diffs or logs.

Follow it up to the point where it spawns its sub-agents. By then it has found the spec source and the standards files. **Before** spawning them, start the external reviewer in the background (Bash with `run_in_background`), so both reviews run at once and the brief cannot contain host findings:

```bash
out=$(mktemp -t peer-review.XXXXXX)
codex exec -s read-only --ephemeral -o "$out" - < <brief file>
```

Use the review command from `docs/agents/peer-review.md`, with `<output file>` set to `$out`. The brief file (written with `mktemp`) holds, in this order:

1. "You are reviewing a code change. Don't modify any file. Run `<diff command>` to see it, and `git log <base>..<head> --oneline` for its commits."
2. **Standards**: the standards files code-review found, as paths to read, plus code-review's smell baseline pasted in full, with its two rules (the repo overrides; smells are judgement calls).
3. **Spec**: the spec code-review found (a path to read, or the fetched contents pasted), or "No spec available: skip the Spec axis."
4. The output format: "Report every finding as one line: `- [Standards|Spec] <path>:<line> · <hard|judgement> · <finding, under 30 words>`. Group nothing, rank nothing. If there are none, write `No findings.`"

Then let code-review finish: its sub-agents, and its `## Standards` and `## Spec` report.

Wait for the background run to finish (you are notified; don't poll). Read `$out`. If the run failed or `$out` is empty, report the last lines of its output under `## Peer review` as "External review failed", and stop after code-review's report.

## 5. Reconcile

The **host findings** are every finding in code-review's Standards and Spec reports. The **external findings** are the lines in `$out`.

Match a host finding and an external finding when they describe the same issue at the same place (same file, overlapping lines, same problem in different words). Every finding lands in exactly one group:

- **Found by both**: matched pairs.
- **External only**: external findings with no match. For each one, open the code (and the spec, for a Spec finding) and check the claim. Give a verdict: **CONFIRMED** (the claim holds) or **DISPUTED** (it doesn't), with a one-line reason pointing at what you read. Disputed findings are reported like the rest.
- **Host only**: host findings with no match.

## 6. Report

After code-review's report, add:

```markdown
## Peer review

External reviewer: Codex · <base>...<head> · both <n> · external only <n> (<n> confirmed, <n> disputed) · host only <n>

### Found by both

- [Standards] `src/auth/session.ts:42`: <finding>

### External only

- **CONFIRMED** [Spec] `src/orders/create.ts:88`: <finding>. Reason: <one line>
- **DISPUTED** [Standards] `src/cart.ts:12`: <finding>. Reason: <one line>

### Host only

- [Spec] `src/orders/create.ts:20`: <finding>
```

An empty group reads "None." Keep each finding's axis tag. Rank nothing across groups.

In PR mode, delete the fetched ref when done: `git update-ref -d refs/peer-review/pr-<n>`.
