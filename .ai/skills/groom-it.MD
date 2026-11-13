---
name: groomer
description: Groom every task file waiting in tasks/1-ToGroom into an executable spec — verified context with file:line anchors, locked decisions, explicit non-goals, a design and a test plan — then move each one to 2-ToValidate. Files every bug found along the way into tasks/0-NewBugs. Runs until 1-ToGroom is empty. Use when asked to groom the backlog, groom pending tasks, or prepare tasks for execution.
---

# Groomer

Turns a two-line bug note into a spec someone can execute without re-doing the research.
  
A groomed task is judged by one question: **could an implementer open this file and start
writing code, without opening the codebase to answer a question the groomer should have
answered?** If not, it is not groomed.

## The pipeline

```
0-NewBugs  →  1-ToGroom  →  2-ToValidate  →  3-ToExecute  →  4-Review  →  6-Done
                                                    ↘  5-PendingExternal
```

This skill owns exactly one hop: `1-ToGroom` → `2-ToValidate`. It never moves anything
further down, and never grooms a file from another directory.

**The directory names contain trailing spaces and the numeric prefixes get renumbered.**
Never hardcode a path. Resolve them every run:

```bash
ls -b tasks/                     # -b makes trailing spaces visible as '\ '
TOGROOM=$(find tasks -maxdepth 1 -type d -name '*ToGroom*')
TOVALIDATE=$(find tasks -maxdepth 1 -type d -name '*ToValidate*')
NEWBUG=$(find tasks -maxdepth 1 -type d -name '*NewBug*')
for d in "$TOGROOM" "$TOVALIDATE" "$NEWBUG"; do [ -d "$d" ] || { echo "missing task dir"; exit 1; }; done
```

If any of the three is empty, **stop and report it**. An empty `$NEWBUG` turns
`"$NEWBUG"/x.md` into `/x.md` — the bug gets written outside the repo and is lost.

Always quote the expansions. An unquoted `tasks/1-ToGroom /x.MD` silently becomes two
arguments and the write lands nowhere — `cat >` will report `No such file or directory` at a
line number that looks unrelated.

## Loop

1. `ls "$TOGROOM"`. **Empty → stop.** That is the only stop condition; report what was groomed
   and end. Do not go looking for more work in other directories.
2. Take the **lowest-numbered** file. Read it.
3. Research it (below). Rewrite the file in full, in the house format (below).
4. Rename to a descriptive slug: `<ticket>-<what-the-fix-does>.MD` — the *fix*, not the
   symptom. `418-trucation-of-fields.MD` became
   `418-show-full-value-in-tooltip-when-field-is-truncated.MD`.
5. `mv` it to `"$TOVALIDATE"`.
6. Report one line for that task, then go to 1.

One file at a time, start to finish. Do not read all of them first and do not batch the
research — the whole value is in per-task depth, and a batched pass produces summaries.

Re-run `ls "$TOGROOM"` at the top of every iteration. The directories are shared and can change
underneath you mid-run; a file can appear or be renamed while you work.

## Research: verify, do not infer

Every factual claim in a groomed file must have been executed, not remembered. The house format
has a section literally called **Context (verified)** — everything under it carries a
`file:line` link, and if you did not open that line, it does not go in.

Work outward from the symptom:

```bash
grep -rn "<the data-e2e hook or class from the ticket>" src/    # where does this actually render?
```

Then answer, for the thing you are about to change:

- **How many render sites are there really?** One hit means fixing it in one place covers every
  consumer. Say so — it collapses the scope.
- **Who reaches this code path?** Trace the props. A branch behind `styleValueAsTag = false` is
  reached by a handful of callers; enumerate them with links.
- **Where does the offending behaviour actually come from?** Frequently not the component named
  in the ticket. #418's truncation lives in the *parent's* `:deep(.input-label)` rule, not in
  `FieldView` — which is exactly why the fix had to measure at runtime instead of taking a prop.
  Getting this wrong produces a plausible design that cannot work.
- **What is the real API of the component you plan to use?** For a base component, read its
  source in `src/base-components/` and quote what you found. That kind of read is what surfaced `hbp-tooltip` rendering `tooltipText` through **`innerHTML`** — a stored-XSS
  sink on customer data, invisible from the call site.
- **Is there a precedent in this repo?** Search before designing. A working implementation of the
  same mechanism beats an invented one, and citing it shortens the review.
  `grep -rn "scrollWidth" src/` found the whole truncation-detection approach.
- **What exists already that you would otherwise rebuild?** `src/helpers/` and
  `src/composables/` are worth a look every time (`escapeHtml` in `SanitizeHtml.ts` was already
  there).
- **What tests touch this today?** `ls <Component>.spec.ts` and `grep -rn "<hook>" tests/`. "No
  spec exists" and "no e2e references this" are both load-bearing facts — they tell the
  implementer there is nothing to migrate.
- **Will the environment let the test observe this?** jsdom reports `scrollWidth === clientWidth
  === 0`; a globally-registered component gets stubbed and swallows its slot. If the obvious test
  cannot fail, say how to make it fail.

Do not stop at the first plausible cause. #418 looked like a one-line tooltip addition; the
research turned up an XSS sink, a layout-breaking wrapper `div`, a stray `tabindex` on every
field, and a second undocumented bug.

## Format

Two worked examples live in the pipeline — find them wherever they have reached by now:

```bash
find tasks -name '1198-*' -o -name '418-show-full-value*'
```

Read both before grooming anything. Title states the fix. Blockquote gives ticket number, the mechanism of the bug, and one sentence
on what is out of scope.

- **`## Goal`** — numbered, observable outcomes. Include what must *not* happen ("a value that
  fits shows no tooltip"). "Truncation itself is unchanged" is a goal, not a detail.
- **`## Decisions (locked)`** — a two-column table, question → decision. Every design question you
  hit and answered goes here **with its reason**, so review argues with the reasoning instead of
  re-deriving it. Anything a reasonable implementer could do two ways belongs in this table. Mark
  the ones that are non-negotiable (`**HTML escaping (required)**`).
- **`## Context (verified)`** — the findings, each anchored. Bold the claim, then the evidence.
  Where a fact drives a decision, say so inline: *"which is exactly why the tooltip must be driven
  by a runtime measurement rather than a prop."*
- **`## Non-goals`** — the real defence against scope creep. Each entry says *why* it is excluded,
  and links the code so a reader can see it was considered rather than missed.
- **`## Design`** — a concrete sketch, not prose. Show the actual template/script/style change.
  Close with **Implementation checks (verify, do not assume)**: the things you could not verify
  from reading, that the implementer must confirm in the running app.
- **`## Tests`** — numbered cases including the negative and the escaping case, plus a manual
  verification line for anything visual (`/run`). Name the file to create.

Two rules on tone: no hedging you did not earn, and no claim you did not run. If you could not
verify something, put it under Implementation checks — do not launder it into Context.

## Bugs found while grooming

Grooming one ticket routinely uncovers a *different* bug. When it does — **before moving on**:

1. Write `"$NEWBUG"/YYYY-MM-DD-<short-slug>.md` (today's date, per `AGENTS.md`). One file per
   bug — every bug found gets filed, including the second and third one in the same ticket.
2. Use the `AGENTS.md` task template with `**Type:** bug`. Under `## Description` give
   **Symptom**, **Why it happens** (verified, with `file:line`), **Why it is not covered by** the
   ticket you were grooming, and **Suggested fix** (cite the precedent). End `## Log` with
   `- YYYY-MM-DD: created in 0-NewBugs while grooming <ticket>`.
3. In the task you are grooming, list it under `## Non-goals` and **link the new file**.

The link matters. A bug that lives only as a "⚠️ flagged, confirm whether this should be a
follow-up" footnote inside someone else's task never enters the pipeline. Filing it is not
optional and it is not a question for the user — file it, then mention it in your report.

The tell that you have found one: you are writing a non-goal and the reason is "the ticket
doesn't mention it" rather than "it is genuinely a different concern".

## Report

Per task, one line: ticket, the fix in a phrase, and the one finding that changed the design.
Then any bugs filed, and confirmation that `1-ToGroom` is empty. Do not paste the groomed files
back — they are on disk and the user has them open.
