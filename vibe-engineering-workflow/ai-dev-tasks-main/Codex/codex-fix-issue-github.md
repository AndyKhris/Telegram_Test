---
description: Diagnose and fix a deploy/test issue, map to tasks, run tests, commit, then open/update a PR and iterate until GitHub auto-merges
argument-hint: ISSUE="<error summary>" [TASKS=<path>] [SCOPE=unit|integration|all] [EXECUTE=yes|no] [BASE=main] [PR=<number>]
---


# /fix_issue (GitHub PR + auto-merge)

You are the issue-fix assistant. Your goal is to **diagnose a failure, map it to the right task/PRD, fix it, test it, and commit** in a controlled way — and then **publish the fix via a PR** that uses the repo’s GitHub Actions + Codex review + auto-merge workflow.

## Inputs
Use `$ARGUMENTS`. `$ISSUE` is required.
- If `TASKS` is provided, read it first.
- If `PR=<number>` is provided, treat it as the PR to update (push to that PR branch).
- `BASE` defaults to `main`.
- `EXECUTE` defaults to `yes`.
- `SCOPE` defaults to `all`.

## Execution modes
- If `EXECUTE=no`: do NOT edit files or run commands. Provide diagnosis + plan only.
- If `EXECUTE=yes`: you MAY edit files and run commands inside the workspace without extra confirmation, except for the “Confirmation gates” below.

## Confirmation gates (always ask first)
Ask “Run this? (y/n)” before:
- Installing dependencies (pip/npm/apt/brew/etc.)
- Any network calls that are not already required for normal repo operation
- Destructive operations (deleting files, reset/clean, rewriting history)
- Any build/deploy packaging steps (docker build, npm build, python -m build, etc.)
- Git publishing actions: `git commit`, `git push`, PR creation/label changes via GitHub MCP/UI

## Tools
- **Chrome DevTools MCP** is available for UI repro, screenshots, console/network inspection, and performance traces when a browser is relevant.

## Rules
- Do not print secrets.
- Prefer minimal targeted fixes over refactors.
- If a task list exists, follow it one sub-task at a time and update the task file accordingly.
- Keep a short “evidence trail”: commands run + key output snippets (no secrets).

## Process
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

12) **Publish to GitHub**
    - If `$PR` is provided (fixing an existing PR):
      - Push commits to the same PR branch.
    - If `$PR` is not provided (new fix):
      - Push your branch to GitHub.
      - Create a PR targeting `$BASE` and ensure it is **not draft**. 
      - Ensure the PR has label `automerge`.
    - Ask before any `git push` / PR creation actions.

13) **Request review + auto-merge loop**
    - Trigger/ensure a Codex review request exists:
      - The repo’s `codex-request-review` workflow posts `@codex review` **only once per PR**.
      - After pushing new commits, you must post `@codex review` again to re-trigger the connector.
    - Poll PR status via GitHub MCP (`mcp__github__pull_request_read(method=get)`) until:
      - `merged == true` → stop (done), OR
      - PR has label `autofix-needed` → proceed to fix loop:
        1) Read the latest Codex connector feedback (reviews/comments/review threads).
        2) Apply fixes locally.
        3) Run tests.
        4) Commit + push to the same PR branch.
        5) Post `@codex review` again.
      - A timeout / max attempts threshold is reached → stop and ask user.

## Output
- **Diagnosis**: likely cause + evidence
- **Fix plan**: steps + files
- **Test plan**: which tests will run
- **GitHub plan**: PR/branch actions and how the automerge loop will proceed
- **Status**: what changed + results

## Confirmation format
- Before running commands: **Run this? (y/n)**
- Before editing files: **Apply fix? (y/n)**

## Clarifying questions (only if needed)
- Which task list/PRD does this map to?
- Is this fix meant for an existing PR (`PR=<number>`) or should we open a new PR?
- Execute commands or provide instructions only?
