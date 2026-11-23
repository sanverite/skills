---
name: review-assist
description: Build a reading plan for a change (uncommitted work, a fixed point, or a PR number), so a human can review it line by line in an order that makes sense, with the change surface and cURLs to test affected endpoints by hand.
disable-model-invocation: true
metadata:
  credits:
    skill: code-review
    author: Matt Pocock
    url: "https://github.com/mattpocock/skills"
---

Build a **reading plan** for a change: a markdown checklist of steps that tells a human reviewer what to read, in which order, and what to check. Steps are grouped by **reading layer**, so the reviewer reads the shapes everything depends on before the code that uses them. The plan opens with the **change surface**: what the change affects, with cURLs for the HTTP endpoints it touches, so the reviewer can test it by hand.

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

Wrap a file path in backticks **only** in its file entry. Elsewhere in the plan, name files without backticks, so the check in step 7 can count entries.

### 5. Map the change surface

List the outward-facing things the change touches. Kinds: APIs (HTTP routes, GraphQL operations, gRPC methods, webhooks the app receives), exported functions, pages/routes, components, hooks, stores, API clients called, CLI commands, jobs, queue consumers and WebSocket handlers, schemas, config keys. Skip any kind with nothing in it.

- **Direct**: the item's own code changed. Only exported or public functions and methods; private helpers are covered by the steps.
- **Indirect**: the item's code didn't change, but it calls a changed symbol. Follow callers up from each changed symbol to the entry points (routes, pages, commands, jobs) and mark each `via <symbol>`. Per changed symbol, list at most 5 indirect items, then "+N more".
- **Removed**: a deleted endpoint, page or export is marked `removed`. A renamed or moved route shows the old path as removed and the new one as direct, with cURLs. Search the repo for callers of anything removed and list them in the terminal report's "Worth a look".
- **Read in**: the step holding the item's code (`Step 3`); for indirect items, the step holding the changed symbol.

Name symbols and routes in backticks (`` `calculateTax()` ``, `` `POST /api/orders` ``), but never a file path, so the check in step 7 still counts each file once.

**cURLs.** Decide per item, never by repo type: write cURLs only for HTTP endpoints this codebase serves, whether the repo is a backend, a full-stack app (e.g. Next.js `app/api/*`) or a monorepo. An API client that calls someone else's endpoint gets no cURL. Removed endpoints get none.

- Open with one `export` block: `BASE_URL` with the port taken from the repo (`.env.example`, config, the dev script), else `http://localhost:3000`; `TOKEN` only if any endpoint needs auth, as `<your token>`. Never read real secrets from `.env` files.
- Per endpoint: a happy-path call, plus one call for the edge case the change introduces (a new validation error, auth rule, status code) when there is one. Two at most. Comment each with the expected status.
- Build bodies from the endpoint's schema or validator (zod, pydantic, DTO, OpenAPI) with realistic values that use any field the change adds or alters. With no schema, infer from the handler and add `# inferred`.
- GraphQL: a `POST` with the operation and variables. gRPC: `grpcurl`. Webhooks: a cURL that simulates the sender, with signature headers set to `<signature>` and marked `# placeholder`. Jobs, queue consumers and WebSocket handlers: no command, a one-line hint in their table's Trigger column instead.

**Test hints.** Each page gets a one-line manual test hint: where to go and what to do to exercise the change ("open /checkout, apply an expired coupon, expect an error toast").

If nothing outward-facing changed (docs, lockfiles, refactors of private code), the section is the single line "No outward-facing changes."

### 6. Write the plan

Write to `.scratch/reviews/<plan name>.md`, creating the folder if needed and overwriting any earlier plan with that name. Layers go in reading order (contracts → domain → adapters → entry points → wiring → skim, or the repo's own layers with skim last); omit any layer with no files. Use exactly this shape:

````markdown
# Reading plan: <branch, PR "#<n> <title>", or "<branch> (uncommitted)">

> [!NOTE]
> **<base>...<head>** · <n> files · <n> commits · <n> steps · 🔴 <n> 🟡 <n> 🟢 <n>
>
> <one or two sentences: what the change does and why>

<optional: when the change spans 3+ steps, a small diagram in a code block showing how the main pieces connect, e.g. `Session type ─▶ newSession() ─▶ login route`>

## Change surface

### APIs

| Endpoint | Impact | Read in |
| --- | --- | --- |
| `POST /api/orders` | direct | Step 4 |
| `GET /api/invoices/:id` | via `calculateTax()` | Step 2 |
| `DELETE /api/carts/:id` | removed | Step 5 |

<details><summary>cURLs</summary>

```bash
export BASE_URL=http://localhost:3000
export TOKEN=<your token>

# POST /api/orders: happy path, expect 201
curl -X POST "$BASE_URL/api/orders" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"items":[{"sku":"TSHIRT-M","qty":2}],"couponCode":"WELCOME10"}'

# POST /api/orders: expired coupon, expect 422
curl -X POST "$BASE_URL/api/orders" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"items":[{"sku":"TSHIRT-M","qty":1}],"couponCode":"EXPIRED"}'

# GET /api/invoices/:id: expect 200 with tax recalculated
curl "$BASE_URL/api/invoices/inv_123" -H "Authorization: Bearer $TOKEN"
```

</details>

### Functions

| Function | Impact | Read in |
| --- | --- | --- |
| `calculateTax()` | direct | Step 2 |

### Pages

| Route | Impact | Read in | Test |
| --- | --- | --- | --- |
| `/checkout` | via `applyCoupon()` | Step 3 | Apply an expired coupon, expect an error toast |

---

## Contracts

- [ ] **1. <title>** · 🟡 medium
  - [`db/migrations/004_sessions.sql:1-12`](../../db/migrations/004_sessions.sql#L1) · [`src/types.ts:3-9`](../../src/types.ts#L3)
  - Changed: <one line>
  - Check: <one line>

---

## Domain

- [ ] **2. <title>** · 🔴 high
  - [`src/auth/session.test.ts:1-30`](../../src/auth/session.test.ts#L1) (test) · [`src/auth/session.ts:12-40`](../../src/auth/session.ts#L12)
  - Changed: <one line>
  - Check: <one line>

…

---

## Skim

<details><summary><n> files: <kinds present, e.g. "lockfile, snapshots"></summary>

- [ ] `package-lock.json`: <a few words>

</details>
````

Other kinds (components, hooks, stores, API clients called, CLI commands, schemas, config keys) get the same three columns, one `###` per kind present. Jobs, queue consumers and WebSocket handlers get a fourth "Trigger" column instead of cURLs. Put a horizontal rule (`---`) before each layer heading. Steps stay checkbox lists, never tables; use no emojis besides the risk dots.

In uncommitted mode, the callout's first line reads `**HEAD + uncommitted** · <n> files · <n> steps · …`.

### 7. Check the plan

Run this against the plan. It prints every path in the file list that doesn't appear exactly once as a file entry. Set `files` to the mode's file list as paths only: `{ git diff --name-only HEAD; git ls-files --others --exclude-standard; }` in uncommitted mode, `git diff --name-only <base>...<head>` otherwise.

```bash
plan=.scratch/reviews/<plan name>.md
<files> | while read -r f; do
  n=$(( $(grep -oF "\`$f\`" "$plan" | wc -l) + $(grep -oF "\`$f:" "$plan" | wc -l) ))
  [ "$n" -eq 1 ] || echo "$n $f"
done
```

If it prints anything, fix the plan and run it again until it prints nothing. Then confirm by reading the plan: layers in order, each test just before its code, no file before a changed file it imports, every surface row's "Read in" step holds its code, and cURLs only for endpoints this codebase serves.

### 8. Report in the terminal

Print the path to the plan, then one line per layer present, with its step count, highest risk and its most important file, then one line counting the change surface by kind. Then a short "Worth a look" list: the checks that didn't fit in the plan's one-line Check fields, callers of anything removed, and anything you noticed across steps.

```
Reading plan: .scratch/reviews/feature-login.md (8 files, 5 steps)
  Contracts     1  🟡  db/migrations/004_sessions.sql
  Domain        1  🔴  src/auth/session.ts
  Entry points  2  🔴  src/routes/login.ts
  Skim          1      (lockfile)
  Surface       APIs 3 (1 direct) · Pages 1 · Functions 1
```

In PR mode, delete the fetched ref when done: `git update-ref -d refs/review-assist/pr-<n>`.
