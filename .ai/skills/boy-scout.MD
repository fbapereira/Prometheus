# Boy Scout review

Review the changes on the current branch against the base branch and leave the codebase a little better than you found it. The output is one finding file per bug or improvement, plus uncommitted fixes for standard violations in the files the diff touches.

## Inputs

- **Base branch:** `main`, unless the user names another.
- **Diff:** `git diff main...HEAD` for committed changes, plus `git status` for uncommitted work. Only files in this diff are in scope.
- **Standards:** every file in `boy-scout/maps/` (standards and patterns to follow and check). Read all of them before reviewing. If the folder is missing or empty, say so and continue with bugs and improvements only.

## Process

1. List the changed files. Read each one in full, not only the hunks, so you see the surrounding context.
2. Read the standards in `boy-scout/maps/`.
3. Review each changed file for three kinds of issue:
   - **Bug:** behaviour that is wrong or will fail. Examples: logic errors, unhandled null or error paths, race conditions, broken contracts with callers, security problems.
   - **Improvement:** working code that could be clearer, safer or easier to maintain.
   - **Standard violation:** a departure from a map in `boy-scout/maps/`.
4. Do not report problems in code the diff did not touch. If a bug in changed code depends on untouched code, mention that dependency inside the bug file as context.
5. Before writing a file, check `boy-scout/findings/` for an existing finding on the same topic and do not duplicate it.

## Output

Write findings to `boy-scout/findings/`:

- Bug: `bug-<slug>.md`
- Improvement: `improvement-<slug>.md`

The slug is lowercase, hyphen-separated and at most six words, for example `bug-null-session-on-logout.md`.

### Bug file

The file must contain:

- **Title:** a one-line summary of the bug.
- **Description** in Jira wiki markup, with these sections: Summary, Steps to reproduce, Expected result, Actual result, and Affected files with line numbers.
- **Reproducing unit test:** a test that fails on the current code. Include the file path and the test code. Run it and confirm it fails. If it passes, the finding is wrong: correct the finding or drop it. If the bug cannot be unit tested, say why and describe the manual reproduction instead.
- **Initial fix prompt:** a self-contained prompt another agent could use to fix the bug, including the test that must pass afterwards.

### Improvement file

The file must contain:

- **Title:** a one-line summary of the improvement.
- **Description** in Jira wiki markup, with the affected files and line numbers.
- **Initial fix prompt:** a self-contained prompt another agent could use to make the change.
- **Reasoning:** why the new code is better, stated concretely (readability, risk, performance, or consistency with a map in `boy-scout/maps/`). Generic statements such as "cleaner code" are not enough.

### Standard violations

Fix these directly in the files the diff touches, in the working tree. Do not create a finding file for them. Rules:

- Do not commit anything. Leave every change uncommitted.
- Edit only files in the diff. Do not reformat untouched code.
- If the project has a linter or test command for the touched files, run it and report any failures.

## Report

Finish with a short summary:

- Number of bugs and improvements, with the file paths written.
- Standard fixes made, listed by file, all uncommitted.
- Anything you could not check, such as a test that could not run or a bug that could not be reproduced in a unit test.
