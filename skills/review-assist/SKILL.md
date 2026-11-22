---
name: review-assist
description: Build a reading plan for a change since a fixed point, so a human can review it line by line in an order that makes sense.
disable-model-invocation: true
metadata:
  credits:
    builds-on: code-review (fixed-point input)
    author: Matt Pocock
    url: "https://github.com/mattpocock/skills"
---

Build a **reading plan** for the diff between `HEAD` and a fixed point the user supplies: a markdown checklist that tells a human reviewer which files to read, in which order. Files are grouped by **reading layer**, so the reviewer reads the shapes everything depends on before the code that uses them.

Read `GLOSSARY.md` at the repo root if it exists, and use its vocabulary when describing changes.

## Process

### 1. Pin the fixed point

Whatever the user passed is the fixed point (a branch, SHA, tag, `HEAD~n`, etc.). If they passed nothing, ask for it.

Before any other work, check both of these, and stop with the quoted message if either fails:

- The ref resolves: `git rev-parse --verify --quiet "<fixed-point>^{commit}"`. On failure: "`<fixed-point>` doesn't resolve to a commit. Pass a branch, SHA, tag or `HEAD~n`."
- The diff is non-empty: `git diff --name-status <fixed-point>...HEAD` (three-dot, so the comparison is against the merge-base). On empty output: "No changes between the merge-base of `<fixed-point>` and `HEAD`. Nothing to review."

The `--name-status` output is the **file list**: every path on it must appear in the plan exactly once. For renames (`R…`) use the new path and note the old one without backticks ("renamed from old/path"); for deletions (`D`) keep the path and mark it deleted.

Read the full diff with `git diff <fixed-point>...HEAD` and the commits with `git log <fixed-point>..HEAD --oneline`.

### 2. Sort files into reading layers

Put each file in exactly one layer. Use the first layer that fits, in this order:

1. **Skim**: generated code, lockfiles (`package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `Cargo.lock`, `poetry.lock`, `go.sum`, …), snapshots (`__snapshots__/`, `*.snap`), vendored code (`vendor/`, `third_party/`), build output, and any file whose header says it is generated.
2. **Contracts**: migrations, database schemas, API definitions (OpenAPI, GraphQL, protobuf), shared types and interfaces, event or message payloads.
3. **Domain**: core business logic that doesn't talk to the outside world.
4. **Adapters**: code that talks to the outside world: database access, HTTP clients, queues, filesystem, third-party SDKs.
5. **Entry points**: HTTP handlers, routes, CLI commands, UI pages and components, jobs, the user-facing surface.
6. **Wiring**: config, dependency injection, build and CI files, infra, Dockerfiles, environment files, package manifests.

Judge by what the file does, not only by its path. A test file goes in the layer of the code it tests. Docs (`*.md`) that describe behaviour go with the code they describe; standalone docs go in **Wiring**.

### 3. Write the plan

Name the file after the current branch (`git branch --show-current`), with `/` replaced by `-`. On a detached `HEAD`, use the short SHA. Write it to `.scratch/reviews/<name>.md`, creating the folder if needed and overwriting any earlier plan with the same name.

Use this shape. Layers appear in the order contracts → domain → adapters → entry points → wiring → skim. Omit any layer with no files.

```markdown
# Reading plan: <branch>

`<fixed-point>...HEAD`, <n> files, <n> commits

<one or two sentences on what the change does, from the commits and diff>

## Contracts

- [ ] `path/to/file.ts`: <one line on what changed>

## Domain

- [ ] `path/to/file.ts`: <one line on what changed>

…

## Skim

<details>
<summary><n> files: generated, lockfiles, snapshots, vendored</summary>

- [ ] `package-lock.json`: <one line, e.g. "deps updated for X">

</details>
```

### 4. Check the plan

Run this check against the plan file. It prints every path from the file list that doesn't appear exactly once:

```bash
git diff --name-only <fixed-point>...HEAD | while read -r f; do
  n=$(grep -cF "\`$f\`" .scratch/reviews/<name>.md)
  [ "$n" -eq 1 ] || echo "$n $f"
done
```

If it prints anything, fix the plan and run it again until it prints nothing.

### 5. Report in the terminal

Print a short version of the plan: the path to the plan file, then one line per layer present, with its file count and the most important file in it. Example:

```
Reading plan: .scratch/reviews/feature-login.md (12 files)
  Contracts     2  db/migrations/004_sessions.sql
  Domain        3  src/auth/session.ts
  Entry points  4  src/routes/login.ts
  Skim          3  (lockfile, snapshots)
```
