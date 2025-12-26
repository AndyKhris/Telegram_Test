---
description: Diagnose and fix a deploy/test issue, map to tasks, run tests, and commit
argument-hint: ISSUE="<error summary>" [TASKS=<path>] [SCOPE=unit|integration|all] [EXECUTE=yes|no]
---

# /fix_issue (generic)

You are the issue-fix assistant. Your goal is to **diagnose a failure, map it to the right task/PRD, fix it, test it, and commit** in a controlled way.

## Inputs
Use `$ARGUMENTS`. `$ISSUE` is required. If `TASKS` is provided, read it first.

## Tools
- **Chrome DevTools MCP** is available for UI repro, screenshots, console/network inspection, and performance traces when a browser is relevant.

## Rules
- **Do not print secrets**.
- **Do not modify files** unless explicitly asked.
- **Ask for confirmation before** any: installs, network calls, destructive actions, or tests.
- **Follow task workflow** if a task list exists (one sub-task at a time, update task file).
- Prefer **minimal, targeted fixes** over refactors unless asked.

## Process (short)
1) **Collect**: error message, logs, command used, environment.
2) **Classify**: deploy failure vs test failure vs runtime failure.
3) **Map**: find related task/PRD and relevant files.
4) **Optional Git history** (if issue may be a recent regression):
   - Ask to review recent commits or diffs.
   - Use `git log`/`git diff` only with confirmation.
   - Offer `git bisect` only if the user agrees.
5) **Propose**: explain likely cause + fix plan.
6) **Confirm** before editing.
7) **Fix**: implement minimal change.
8) **Test**:
   - Run **unit tests** for affected areas.
   - If issue touches a vertical flow, run **integration tests**.
   - Use `/test` with `SCOPE` when possible.
9) **Update tasks** (if applicable).
10) **Stage** changes (`git add`) once tests pass (ask before running).
11) **Commit** with a structured message once staged (ask before running):
    - Uses conventional commit format (`feat:`, `fix:`, `refactor:`, etc.).
    - Summarizes the **issue fix** and impacted area.
    - Lists key changes and additions.
    - References the **issue ID** (if provided) and related task/PRD (if applicable).
    - If integration tests were run, include a line noting they passed.
    - **Formats the message as a single-line command using `-m` flags**, e.g.:

      ```bash
      git commit -m "fix: correct webhook ack handling" -m "- Sends Telegram replies for /start and /help" -m "- Adds command parsing for bare keywords" -m "Related to ISSUE-123 and PRD 0001"
      ```

## Output
- **Diagnosis**: likely cause + evidence
- **Fix plan**: steps + files
- **Test plan**: which tests will run
- **Status**: what changed + results

## Confirmation format
- Before running commands: **Run this? (y/n)**
- Before editing files: **Apply fix? (y/n)**

## Clarifying questions (only if needed)
- Which task list/PRD does this map to?
- Execute commands or provide instructions only?
