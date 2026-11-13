# AGENTS.md

Instructions for AI agents working in Prometheus.

## Task board

Tasks live in `tasks/` as one Markdown file each. A task's folder is its status. To change status, move the file to another folder.

| Folder | Meaning |
|--------|---------|
| `0-NewBugs` | Bugs that were just reported and nobody has looked at yet |
| `1-ToGroom` | Needs details: scope, acceptance criteria, open questions |
| `2-ToValidate` | Groomed and waiting for a human to approve it |
| `3-ToExecute` | Approved and ready to work on |
| `4-Review` | Work is done and waiting for review |
| `5-PendingExternal` | Blocked on someone or something outside the team |
| `6-Done` | Finished and accepted |

## Rules

- Name files `YYYY-MM-DD-short-slug.md`, for example `2026-11-12-login-timeout.md`.
- Only work on tasks in `3-ToExecute`. Don't move a task out of `2-ToValidate` without human approval.
- When you finish a task, move it to `4-Review`, never straight to `6-Done`.
- When a task is blocked, move it to `5-PendingExternal` and write down what it is waiting for.
- Add a dated line to the task's `## Log` every time you move it.

## Task template

```markdown
# <Title>

**Type:** bug | feature | chore
**Created:** YYYY-MM-DD

## Description
What happens and what should happen instead.

## Acceptance criteria
- [ ] ...

## Notes

## Log
- YYYY-MM-DD: created in 0-NewBugs
```
