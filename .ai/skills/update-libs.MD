---
name: update-libs
description: Update the project's dependencies in three passes — all patch bumps in one commit, minor bumps (adopting any new feature the codebase can use), and a groomed upgrade plan in tasks/2-ToValidate for every major bump. Use when asked to update libs, bump dependencies, check for outdated packages, or run the dependency upgrade.
---

# Update libs

Brings dependencies up to date without ever leaving `main` red. Patches and minors are
applied and committed here; majors are **never installed** by this skill — they become a plan
for a human to approve.

## Step 0 — Prepare

1. Working tree must be clean (`git status --porcelain` empty). If not, stop and report.
2. Branch off: `git switch -c chore/update-libs-YYYY-MM-DD`. Never commit on `main`.
3. Detect the package manager from the lockfile and use it for every command below:
   `pnpm-lock.yaml` → pnpm, `yarn.lock` → yarn, `bun.lock*` → bun, otherwise npm.
4. Find the check commands in `package.json` `scripts`: the unit test script and the typecheck
   script (`typecheck`, `type-check`, or `tsc --noEmit` if TypeScript is installed). Run both
   **now**, before touching anything. If either fails on a clean tree, stop — you can't tell
   your breakage from existing breakage. If a script does not exist, say so in the report;
   do not invent one.

"Run the checks" below means: install, then the unit tests, then the typecheck. All must pass.

## Step 1 — Check

```bash
npm outdated --json        # or: pnpm outdated --format json / yarn outdated --json
```

For each package compare the **installed** version to **latest** and classify:

| Group | Rule | Example |
|-------|------|---------|
| Patch | only `z` changes in `x.y.z` | `1.4.2 → 1.4.7` |
| Minor | `y` changes, `x` same, `x ≥ 1` | `1.4.2 → 1.6.0` |
| Major | `x` changes | `1.4.2 → 2.0.0` |

Edge cases:

- **`0.x` packages:** a minor bump (`0.3 → 0.4`) is breaking under semver. Treat it as **Major**.
  A `0.x` patch bump stays a Patch.
- **Pre-releases** (`-alpha`, `-beta`, `-rc`, `-next`): skip. Only stable versions.
- **Multiple jumps:** a package going `1.4.2 → 2.1.0` is a Major. Do not also apply its
  latest `1.x` here; that is part of the major plan.

Print the three groups before changing anything.

## Step 2 — Patches (one commit)

1. Bump every Patch package to its latest patch in one go, updating both `package.json` and the
   lockfile.
2. Run the checks.
3. Pass → commit `chore(deps): update patch dependencies` listing each `name old → new` in the
   body.
4. Fail → find the culprit (revert half, re-run, repeat). Commit the passing ones; drop the
   culprit from this run and file it as a bug in `0-NewBugs` (see **Filing tasks**) with the
   failing output. A patch that breaks the build is a bug, not a plan.

## Step 3 — Minors

*Agent:* **minimalist** (`.ai/agents/minimalist.MD`) — read-only; it decides, you edit.

For each Minor package:

1. Read the changelog / release notes for every version between installed and latest.
2. Ask the minimalist: does any new feature let us **delete or simplify** code we already have
   (replace a helper, a workaround, a polyfill, a hand-rolled option)? It must point to the
   exact `file:line` it would change. "Could be nice to use" is not a yes.
3. Sort:
   - **Has an applicable feature** → its own commit: bump + adopt the feature, then run the
     checks. Message: `chore(deps): update <name> to <ver> and use <feature>`.
   - **Nothing applicable** → goes into the shared batch.
4. Bump the whole batch in one commit, run the checks, message
   `chore(deps): update minor dependencies` with `name old → new` in the body.

A minor that fails the checks: same as patches — isolate it, keep the rest, file the culprit.
Don't fix deprecations here unless the fix is trivial; otherwise file them.

## Step 4 — Majors (plan only, no install)

*Agent:* **planner** (`.ai/agents/planner.MD`) — read-only; it researches, you write the file.

For each Major package, write **one** task file in `2-ToValidate` (see **Filing tasks**) titled
`Upgrade <name> from <old> to <new>`, `**Type:** chore`. It must contain:

- **Breaking changes that hit us** — from the migration guide, each one with the `file:line`
  sites it affects (grep for them; don't guess). Breaking changes that do not touch this
  codebase get one line saying so.
- **How to migrate each site** — the concrete code change, and whether a codemod exists.
- **New features worth adopting** — same bar as Step 3: only ones that remove or simplify
  existing code, with `file:line`.
- **Peer / coupled upgrades** — packages that must move together (e.g. a framework and its
  plugins, `@types/*`). Group them into the same plan rather than one plan each.
- **Acceptance criteria** — checks pass, plus the manual verification the changes need.

Do not create a plan if one for the same package and target version already exists anywhere in
`tasks/`; update it instead.

## Filing tasks

Follow `AGENTS.md`: file name `YYYY-MM-DD-short-slug.md`, the task template, and a dated
`## Log` line (`- YYYY-MM-DD: created in 2-ToValidate by update-libs`). Resolve directories,
never hardcode them:

```bash
TOVALIDATE=$(find tasks -maxdepth 1 -type d -name '*ToValidate*')
NEWBUG=$(find tasks -maxdepth 1 -type d -name '*NewBug*')
for d in "$TOVALIDATE" "$NEWBUG"; do [ -d "$d" ] || { echo "missing task dir"; exit 1; }; done
```

Always quote the expansions.

## Report

- Patches: count, commit hash.
- Minors: one line per own-commit package (the feature adopted), then the batch count and hash.
- Majors: one line per plan file created.
- Anything skipped or filed as a bug, and why.

Do not push or open a PR unless asked.
