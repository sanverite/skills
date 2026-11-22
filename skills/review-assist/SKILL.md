---
name: review-assist
description: Build a reading plan for a change (uncommitted work, a fixed point, or a PR number), so a human can review it line by line in an order that makes sense.
disable-model-invocation: true
metadata:
  credits:
    skill: code-review
    author: Matt Pocock
    url: "https://github.com/mattpocock/skills"
---

Build a **reading plan** for a change: a markdown checklist of steps that tells a human reviewer what to read, in which order, and what to check. Steps are grouped by **reading layer**, so the reviewer reads the shapes everything depends on before the code that uses them.

Read `GLOSSARY.md` at the repo root if it exists, and use its vocabulary in every description.

## Process

### 1. Pick the change

Run every command from the repo root (`git rev-parse --show-toplevel`), so paths, links and the plan location all line up.

Choose the mode from what the user passed:

- **Nothing passed.** If `git status --porcelain` prints anything, use **uncommitted** mode. If it prints nothing, ask: "No uncommitted changes. Which change should I plan: a fixed point (branch, SHA, tag, `HEAD~n`) or a PR number?"
- **A PR number** (`#` then digits, or bare digits that `gh pr view` finds): **PR** mode. Bare digits that aren't a PR but resolve to a commit are a fixed point.
- **Anything else**: **fixed-point** mode.

Each mode settles the same five things. Use them in every later step:

| | Uncommitted | Fixed point | PR |
| --- | --- | --- | --- |
| Base | `HEAD` | `<ref>` | `<baseRefOid>` |
| Head | working tree | `HEAD` | `refs/review-assist/pr-<n>` |
| File list | `git diff --name-status HEAD`, plus `git ls-files --others --exclude-standard` (marked added) | `git diff --name-status <base>...<head>` | same as fixed point |
| Plan name | `<branch>-uncommitted` | `<branch>` | `pr-<n>` |
| Link target | `../../<path>#L<line>` | `../../<path>#L<line>` | `<repo url>/blob/<headRefOid>/<path>#L<line>` |

`<branch>` is `git branch --show-current`, or the short SHA on a detached `HEAD`. In every plan name, replace `/` with `-`. Three-dot diffs compare against the merge-base.

**PR setup.** Read the tracker setup in `docs/agents/issue-tracker.md` if present. On GitHub:

1. `gh pr view <n> --json number,title,body,url,baseRefOid,headRefOid`. On failure, stop: "PR #<n> not found, or `gh` isn't signed in (`gh auth login`)."
2. Fetch both sides from the PR's own repo (`<repo url>` is the PR url without `/pull/<n>`), without touching the working tree: `git fetch <repo url>.git <baseRefOid> "+pull/<n>/head:refs/review-assist/pr-<n>"`. `baseRefOid` is the base tip (at merge time, for a merged PR), so merged PRs still have a diff.
3. Use the PR title and body for the overview. Read file contents with `git show <head>:<path>`.

**Checks.** Before any other work, stop with the quoted message if one fails (fixed-point and PR modes):

- The base resolves: `git rev-parse --verify --quiet "<base>^{commit}"`. On failure: "`<base>` doesn't resolve to a commit. Pass a branch, SHA, tag, `HEAD~n` or a PR number."
- It shares history with the head: `git merge-base <base> <head>`. On failure: "`<base>` shares no history with `<head>`, so there's no merge-base to compare against."
- The file list is non-empty. On failure: "No changes between the merge-base of `<base>` and `<head>`. Nothing to review."

The **file list** must appear in the plan exactly once per path. For renames (`R…`) use the new path and note "renamed from old/path" without backticks. Keep deleted files (`D`), marked deleted.

Read the full diff (`git diff HEAD` in uncommitted mode, reading untracked files whole; `git diff <base>...<head>` otherwise) and the commits (`git log <base>..<head> --oneline`; none in uncommitted mode).

### 2. Sort files into reading layers

**Repo layering first.** If the repo has its own layering, use it in place of the defaults below, keeping skim last. Signs: a layering or architecture section in `CLAUDE.md`, `AGENTS.md`, `ARCHITECTURE.md` or `docs/adr/`, or workspace packages (`packages/*`, `apps/*`, `crates/*`) that depend on each other. For packages, read lower packages (those others depend on) first, and name each layer after its package.

**Default layers.** Otherwise check each file against these in the order listed, and put it in the first that fits. Skim is checked first but always read last:

1. **Skim**: generated code, lockfiles (`package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `Cargo.lock`, `poetry.lock`, `go.sum`, …), snapshots (`__snapshots__/`, `*.snap`), vendored code (`vendor/`, `third_party/`), build output, and any file whose header says it is generated.
2. **Contracts**: migrations, database schemas, API definitions (OpenAPI, GraphQL, protobuf), shared types and interfaces, event or message payloads.
3. **Domain**: core business logic that doesn't talk to the outside world.
4. **Adapters**: code that talks to the outside world: database access, HTTP clients, queues, filesystem, third-party SDKs.
5. **Entry points**: HTTP handlers, routes, CLI commands, UI pages and components, jobs, the user-facing surface.
6. **Wiring**: config, dependency injection, build and CI files, infra, Dockerfiles, environment files, package manifests, standalone docs.

Judge by what the file does, not only by its path. Docs that describe one piece of code go with that code. A test goes in the layer of the code it tests.

### 3. Order and group into steps

- **Imports.** Within a layer, read each changed file's imports and order the files so a file comes before every changed file that imports it. Files in an import cycle go in the same step.
- **Steps.** Group each file with its test, and keep a few files together only when they make one change that can't be judged apart. Number steps 1, 2, 3… across the whole plan.
- **Tests first.** Within a step, list the test immediately before the code it tests, marked `(test)`, so the reviewer knows what the code should do before judging whether it does.

### 4. Write each step

Each step has:

- **Title**: a few words naming what the step changes, in domain language.
- **Risk**: 🔴 high, 🟡 medium or 🟢 low. High: auth or security, data migrations and deletes, money, concurrency, public API or schema changes, removed behaviour, or logic with no test. Low: renames, comments, formatting, mechanical moves. Medium: the rest.
- **Files**: each one a link (target from the table in step 1) whose text is the path and the changed line ranges in the new version, taken from the hunk headers of the diff with `-U0`: `` [`src/auth/session.ts:12-40,88`](../../src/auth/session.ts#L12) ``. The link points at the first range. Exceptions:
  - Untracked files: the whole file, `:1-<line count>`.
  - Pure renames and binary files (no hunks): link with the path alone as text, `` [`assets/logo.png`](../../assets/logo.png) ``.
  - Deleted files: no link, `` `old/path.ts` (deleted) ``.
  - Skim files: the path alone, `` `package-lock.json` ``.
- **Changed**: one line, under 20 words, on what changed and why.
- **Check**: one line, under 25 words, on the one to three things most worth verifying: edge cases, invariants, anything in the diff that looks off. Put the rest in the terminal report's "Worth a look" list.

Wrap a file path in backticks **only** in its file entry. Elsewhere in the plan, name files without backticks, so the check in step 6 can count entries.

### 5. Write the plan

Write to `.scratch/reviews/<plan name>.md`, creating the folder if needed and overwriting any earlier plan with that name. Layers go in reading order (contracts → domain → adapters → entry points → wiring → skim, or the repo's own layers with skim last); omit any layer with no files. Use exactly this shape:

```markdown
# Reading plan: <branch, PR "#<n> <title>", or "<branch> (uncommitted)">

> <base>...<head> · <n> files · <n> commits · <n> steps · 🔴 <n> 🟡 <n> 🟢 <n>

<one or two sentences: what the change does and why>

<optional: when the change spans 3+ steps, a small diagram in a code block showing how the main pieces connect, e.g. `Session type ─▶ newSession() ─▶ login route`>

## Contracts

- [ ] **1. <title>** · 🟡 medium
  - [`db/migrations/004_sessions.sql:1-12`](../../db/migrations/004_sessions.sql#L1) · [`src/types.ts:3-9`](../../src/types.ts#L3)
  - Changed: <one line>
  - Check: <one line>

## Domain

- [ ] **2. <title>** · 🔴 high
  - [`src/auth/session.test.ts:1-30`](../../src/auth/session.test.ts#L1) (test) · [`src/auth/session.ts:12-40`](../../src/auth/session.ts#L12)
  - Changed: <one line>
  - Check: <one line>

…

## Skim

<details><summary><n> files: <kinds present, e.g. "lockfile, snapshots"></summary>

- [ ] `package-lock.json`: <a few words>

</details>
```

In uncommitted mode, the header line reads `HEAD + uncommitted · <n> files · <n> steps · …`.

### 6. Check the plan

Run this against the plan. It prints every path in the file list that doesn't appear exactly once as a file entry. Set `files` to the mode's file list as paths only: `{ git diff --name-only HEAD; git ls-files --others --exclude-standard; }` in uncommitted mode, `git diff --name-only <base>...<head>` otherwise.

```bash
plan=.scratch/reviews/<plan name>.md
<files> | while read -r f; do
  n=$(( $(grep -oF "\`$f\`" "$plan" | wc -l) + $(grep -oF "\`$f:" "$plan" | wc -l) ))
  [ "$n" -eq 1 ] || echo "$n $f"
done
```

If it prints anything, fix the plan and run it again until it prints nothing. Then confirm by reading the plan: layers in order, each test just before its code, and no file before a changed file it imports.

### 7. Report in the terminal

Print the path to the plan, then one line per layer present, with its step count, highest risk and its most important file. Then a short "Worth a look" list: the checks that didn't fit in the plan's one-line Check fields, and anything you noticed across steps.

```
Reading plan: .scratch/reviews/feature-login.md (8 files, 5 steps)
  Contracts     1  🟡  db/migrations/004_sessions.sql
  Domain        1  🔴  src/auth/session.ts
  Entry points  2  🔴  src/routes/login.ts
  Skim          1      (lockfile)
```

In PR mode, delete the fetched ref when done: `git update-ref -d refs/review-assist/pr-<n>`.
