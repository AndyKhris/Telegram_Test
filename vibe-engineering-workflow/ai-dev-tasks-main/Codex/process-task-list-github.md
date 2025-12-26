---
description: Work through a task list one sub-task at a time, then open a PR and iterate until GitHub auto-merges
argument-hint: TASKS=<path> [BASE=main]
---

# Task List Management (GitHub PR + Auto-merge)

Guidelines for managing task lists in markdown files to track progress on completing a PRD, with a GitHub PR loop that uses:
- `codex-request-review` (posts `@codex review` once per PR when label `automerge` is present)
- `codex-automerge` (auto-merges when the Codex connector indicates pass; otherwise applies label `autofix-needed`)
- `ci-tests` (runs lint + tests on each PR)
- `ci-tests-autofix-needed` (adds label `autofix-needed` + `ci-failed` when CI fails on an `automerge` PR)
When invoked as a slash command, treat `$TASKS` (if provided) as the path to the task list markdown file (e.g., `@/tasks/tasks-0003-prd-generate-images.md`). Read that file first, then apply the guidelines below.

## Task Implementation

- **One sub-task at a time:** Do **NOT** start the next sub-task until you ask the user for permission and they say "yes" or "y".
- **Completion protocol:**
  1. When you finish a **sub-task**, immediately mark it as completed by changing `[ ]` to `[x]`.
  2. If **all** subtasks underneath a parent task are now `[x]`, follow this sequence:
     - **First**: Run the repo’s formatters/linters (stack-appropriate; e.g., `python -m black .` and `python -m ruff check src tests`) and fix any issues.
     - **Then**: Run the relevant **unit tests** for that parent task (or the full unit suite if unsure).
     - If this parent task is **Integration Tests** (final parent for a vertical feature), run the **integration tests** after unit tests.
     - **Only if style checks + tests pass**: Stage changes (`git add .`).
     - **Clean up**: Remove any temporary files and temporary code before committing.
     - **Commit**: Use a descriptive conventional commit message, formatted as a single-line command using `-m` flags, e.g.:

       ```bash
       git commit -m "feat: add payment validation logic" -m "- Validates card type and expiry" -m "- Adds unit tests for edge cases" -m "Related to T123 in PRD"
       ```

  3. Once all the subtasks are marked completed and changes have been committed, mark the **parent task** as completed.
- Stop after each sub-task and wait for the user's go-ahead.

## GitHub PR Loop (after commit)

### 4) Publish branch + create PR

After a commit succeeds (and you're ready to review/merge it):
1. **Push your branch** (if not already pushed):
   - Typical: `git push -u origin <branch>`
2. **Create a Pull Request** targeting `$BASE` (default `main`):
   - Prefer GitHub MCP: `mcp__github__create_pull_request`
   - Ensure the PR is **not draft** (`draft=false`).
3. **Enable the automerge workflow**:
   - Add label `automerge` to the PR (GitHub MCP: `mcp__github__issue_write` with `labels=["automerge"]`).
   - Then wait for `codex-request-review` to post `@codex review`; if it does not show up (missing/invalid `CODEX_REVIEW_TOKEN`), post `@codex review` yourself.

### 5) Poll + autofix until merged

After the PR exists, repeat the following loop until the PR is merged (or you stop):

1. Poll the PR using GitHub MCP `mcp__github__pull_request_read(method=get)`:
   - If `merged == true` -> **stop (done)**.
   - If the PR is closed without merge -> **stop and report**.
2. If the PR has label `autofix-needed`:
   - Fetch CI/check status first:
     - `mcp__github__pull_request_read(method=get_status)`
   - If label `ci-failed` is present (or `CI Tests` is failing), prioritize fixing CI:
     - Reproduce locally by running the same commands (typically `python -m ruff check ...` and `python -m pytest -q`, plus any formatter checks your repo enforces).
     - Fix the failing tests/lint, then re-run locally until green.
   - Fetch the latest Codex connector feedback:
     - `mcp__github__pull_request_read(method=get_review_comments)` (line-level discussions)
     - and/or `mcp__github__pull_request_read(method=get_comments)` / `method=get_reviews` (summary feedback)
   - Fix the issue(s) locally.
   - Run relevant tests locally.
   - Commit and push to the **same PR branch**.
   - Re-request review by posting `@codex review` again via `mcp__github__add_issue_comment`.
     - Important: `codex-request-review` only posts `@codex review` once per PR, so the agent must re-request it after updates.
   - Return to polling.
3. If the PR is not merged and does not have `autofix-needed`:
   - Also check CI/check status:
     - `mcp__github__pull_request_read(method=get_status)`
   - Keep polling until:
     - CI is green AND the Codex connector responds (then `codex-automerge` will merge or apply `autofix-needed`), or
     - A timeout is reached.

**Loop guards (recommended):**
- Don't spam: only post a new `@codex review` after you pushed a new commit.
- Set `MAX_AUTOFIX_ATTEMPTS` (e.g., 3) and `MAX_WAIT_MINUTES` (e.g., 30) to avoid infinite loops.
- Only run the loop for PRs you authored, same-repo (not forks), and with label `automerge`.


## Task List Maintenance

1. **Update the task list as you work:**
   - Mark tasks and subtasks as completed (`[x]`) per the protocol above.
   - Add new tasks as they emerge.

2. **Maintain the "Relevant Files" section:**
   - List every file created or modified.
   - Give each file a one-line description of its purpose.

## AI Instructions

When working with task lists, the AI must:

1. Regularly update the task list file after finishing any significant work.
2. Follow the completion protocol:
   - Mark each finished **sub-task** `[x]`.
   - Mark the **parent task** `[x]` once **all** its subtasks are `[x]`.
3. Add newly discovered tasks.
4. Keep "Relevant Files" accurate and up to date.
5. Before starting work, check which sub-task is next.
6. After implementing a sub-task, update the file and then pause for user approval.





