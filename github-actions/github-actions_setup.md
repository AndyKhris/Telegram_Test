# GitHub Actions setup: Codex Review trigger + auto-merge

This doc captures (1) the end goal, (2) how we implemented it in this repo, and (3) **every issue we hit along the way** and how we resolved it.

> Repo: `AndyKhris/Telegram_Test`
>
> Key files:
> - `.github/workflows/codex-request-review.yml`
> - `.github/workflows/codex-automerge.yml`
> - `.github/workflows/ci-tests.yml`
> - `.github/workflows/ci-tests-autofix-needed.yml`

---

## 1) Goal and constraints

### Goal
Integrate Codex Review into the repo so the workflow is:

1. Developer opens a PR and applies label `automerge`.
2. A workflow posts a **human-authored** PR comment: `@codex review`.
3. The `chatgpt-codex-connector` GitHub App responds on the PR.
4. A CI workflow runs lint + tests on the PR in a clean environment.
5. A workflow detects the Codex response and enables auto-merge (squash).
6. GitHub merges automatically once required checks are green.
7. If Codex finds issues (or doesn't clearly pass), the workflow applies label `autofix-needed` so the developer/agent can fix and re-request review.
8. If CI fails, the workflow applies labels `autofix-needed` + `ci-failed` so the developer/agent can fix and push.

### Constraints / requirements
- Must use: **Codex CLI**, **GitHub MCP** (from Codex CLI), **Codex Review (GitHub connector)**, **GitHub Actions**.
- No Claude-based automation.
- Git operations in this environment are constrained (repo `.git` protected), so we used **GitHub MCP write tools** to create branches/PRs and push changes.
- We want a safe default: only auto-merge PRs we authored, from the same repo (no forks), and only when explicitly labeled `automerge`.

---

## 2) Final architecture (what each workflow does)

### `codex-request-review` (request a Codex review)
File: `.github/workflows/codex-request-review.yml`

Purpose:
- When a PR is ready **and** labeled `automerge`, post **one** comment that contains `@codex review`.

Why it exists:
- The Codex connector can be configured to review PRs automatically, but in practice we wanted a deterministic "request review now" trigger.
- Also: **Codex often ignores comments made by `github-actions[bot]`**, so the comment must come from a human account (via a PAT).

Key behaviors:
- Trigger: `pull_request_target` on events `[opened, reopened, ready_for_review, labeled, synchronize]`.
- Security note:
  - `pull_request_target` runs in the context of the **base branch** and can access secrets.
  - To avoid secret exfiltration, this workflow **does not** check out or run PR code; it only calls GitHub APIs.
  - It also refuses to run on forked PRs (`head.repo.full_name == github.repository`).
- Safety checks:
  - PR is **not draft**
  - PR head repo is the same repo (no forks): `github.event.pull_request.head.repo.full_name == github.repository`
  - PR has label `automerge`
  - Comment is only posted once per PR (dedupe by searching existing comments for `@codex review`)
- Auth for the comment: `CODEX_REVIEW_TOKEN` (a PAT stored as an Actions secret).

### `codex-automerge` (merge after Codex passes)
File: `.github/workflows/codex-automerge.yml`

Purpose:
- When `chatgpt-codex-connector[bot]` posts feedback indicating "pass", merge the PR automatically.
- When the feedback does **not** indicate pass, apply label `autofix-needed` and do **not** merge.

Key behaviors:
- Triggers (the connector can post in different PR surfaces):
  - `issue_comment` (regular PR comments)
  - `pull_request_review` (review summaries)
  - `pull_request_review_comment` (inline review comments / threads)
- Filter: only run when the author is `chatgpt-codex-connector` or `chatgpt-codex-connector[bot]`.
- Safety gates before merging:
  - PR has label `automerge`
  - PR is not draft
  - PR author allowlist (currently `["AndyKhris"]`)
  - Not a fork (`pr.head.repo.full_name === owner/repo`)
- Pass detection:
  1) Prefer a structured block:
     - A `CODEX_AUTOMERGE_V1` block with `RESULT: PASS`, `P0_COUNT: 0`, `P1_COUNT: 0`, `AUTOMERGE: ENABLE`
  2) Fallback (because Codex connector did not reliably follow the requested output format):
     - Regex match for the phrase "didn't find any major issues" (handles straight `'` and curly apostrophes, and "did not").
- Non-pass behavior:
  - Ensure a repo label named `autofix-needed` exists (create it if missing) and add it to the PR.
  - Do not merge.
- Pass behavior:
  - Remove label `autofix-needed` if present.
- Merge behavior:
  1) Try GraphQL: `enablePullRequestAutoMerge`
  2) If that fails (common if repo auto-merge is disabled), fallback to merging immediately via REST (`pulls.merge`) using `squash` (and then `merge` as a backup).
- After action:
  - Post a comment with a `CODEX_AUTOMERGE_V1` summary block so the PR has a stable audit trail.

### `ci-tests` (lint + tests on PR)
File: `.github/workflows/ci-tests.yml`

Purpose:
- Run `ruff` and `pytest` on every PR update (same-repo branches) in a clean environment.

Notes:
- This workflow is intended to be configured as a **required status check** via branch protection.

### `ci-tests-autofix-needed` (label PR when CI fails)
File: `.github/workflows/ci-tests-autofix-needed.yml`

Purpose:
- When `CI Tests` completes for a PR labeled `automerge`:
  - If CI failed: add labels `autofix-needed` and `ci-failed`
  - If CI passed: remove label `ci-failed` (leave `autofix-needed` unchanged because it may be used by Codex review)

---

## 3) Step-by-step setup in GitHub UI

### Step 1 - Install / verify the Codex GitHub connector
You need the GitHub App installed and configured on the repo:
- App user shows as `chatgpt-codex-connector[bot]`
- It responds to PR comments containing `@codex review`

How to verify:
- Open any PR and comment `@codex review`
- Confirm the app replies on the PR thread (or reacts with a thumbs-up if it has no feedback)

> Important: the connector's reply format is **not guaranteed** to follow `AGENTS.md` exactly, even if you request it. This is why we added the phrase-based fallback in `codex-automerge`.

### Step 2 - Create a PAT for posting the `@codex review` comment
We must comment as a **human**, not `github-actions[bot]`.

Create a fine-grained PAT:
1. GitHub -> Settings -> Developer settings -> Fine-grained personal access tokens -> Generate new token
2. Repository access: select `AndyKhris/Telegram_Test`
3. Permissions (minimum required):
   - **Issues: Read and write** (PR comments are issue comments)
   - **Pull requests: Read** (workflow reads PR metadata before deciding to comment)
4. Save token value

### Step 3 - Add `CODEX_REVIEW_TOKEN` as a repo secret
1. Repo -> Settings -> Secrets and variables -> Actions
2. New repository secret:
   - Name: `CODEX_REVIEW_TOKEN`
   - Value: the PAT from Step 2

### Step 4 - Ensure Actions workflow token permissions allow writes
Repo -> Settings -> Actions -> General -> Workflow permissions:
- Select **Read and write permissions**

Why:
- `codex-automerge` merges PRs and comments using `GITHUB_TOKEN`, which needs write permissions.

### Step 5 - Ensure required labels exist
Repo -> Issues -> Labels:
- Ensure label `automerge` exists (create it if missing).

Note:
- Labels `autofix-needed` and `ci-failed` are created automatically by workflows when needed.

### Step 6 - Enable auto-merge (optional)
Repo -> Settings -> General -> Pull Requests:
- Enable "Allow auto-merge"

Note:
- Our workflow tries to enable auto-merge first, but it can still merge directly via REST if auto-merge isn't enabled.

### Step 7 - Require CI on `main` (recommended)
Configure GitHub so the **`CI Tests` workflow is a hard merge gate**.

Prerequisite (important):
- GitHub may not show a workflow/job in the required checks picker until it has run at least once.
- If `CI Tests / ci-tests` is missing, **push a same-repo PR (or a commit) that triggers `.github/workflows/ci-tests.yml` once**, wait for it to complete, then refresh the settings UI.

✅ Option A: Using Branch rulesets (recommended as of 2025)
1. Repo -> Settings -> Branches
2. Click **Add branch ruleset**
3. Ruleset name: `main-protection` (or similar)
4. Target branches: `main`
5. Enable:
   - Require pull request before merging
   - Require status checks to pass before merging
6. Under required status checks, select the CI check:
   - Usually shown as `CI Tests / ci-tests` (workflow name + job name)
7. Save / Create

Option B: Classic branch protection rules (legacy)
1. Repo -> Settings -> Branches -> Branch protection rules
2. Add (or edit) a rule for `main`:
   - Enable "Require a pull request before merging"
   - Enable "Require status checks to pass before merging"
   - Select the check from `ci-tests.yml` (usually shown as `CI Tests / ci-tests`)

Why:
- With required checks enabled, `codex-automerge` can enable auto-merge after a clean Codex review, and GitHub will merge only after CI is green.

Important:
- `ci-tests.yml` currently installs dependencies from a repo-root `requirements.txt` via `pip install -r requirements.txt`.
  - If your repo uses Poetry/`pyproject.toml`, update the workflow install step accordingly.

---

## 4) How we push code / create PRs (Codex CLI + GitHub MCP)

Because `.git` operations can require escalation/approval in this environment, we prefer GitHub MCP write tools for "remote-first" changes:
- `create_branch` to create a branch on GitHub
- `create_or_update_file` to push a file change to that branch (or main)
- `create_pull_request` to open a PR
- `issue_write` to label the PR with `automerge`

This keeps the workflow remote-first (no local `git push` required), but local git still works when needed (see Issue L).

---

## 5) Full list of issues encountered (and fixes)

This section is intentionally exhaustive.

### Issue A - "Resource not accessible by integration" (403) when commenting `@codex review` from Actions
**What happened**
- Initial attempt was to have GitHub Actions comment `@codex review` using the default `GITHUB_TOKEN`.
- The workflow run failed with:

```
RequestError [HttpError]: Resource not accessible by integration
... POST https://api.github.com/repos/AndyKhris/Telegram_Test/issues/<PR_NUMBER>/comments
```

**Why it mattered**
- Even if we fixed permissions, we later discovered another blocker: the Codex connector typically ignores comments authored by `github-actions[bot]`.

**Fix**
- Stop trying to trigger Codex review using `GITHUB_TOKEN`.
- Introduce a PAT secret `CODEX_REVIEW_TOKEN` and use it in `codex-request-review` so the comment appears as the user account (`AndyKhris`) instead of `github-actions[bot]`.

### Issue B - Codex connector didn't respond when Actions posted `@codex review`
**What happened**
- When the comment was authored by `github-actions[bot]`, the connector did not reply.

**Fix**
- Same as Issue A: use a human-authored comment via `CODEX_REVIEW_TOKEN`.

### Issue C - Couldn't easily inspect Actions runs via CLI (TLS/SSPI error)
**What happened**
- From the PowerShell environment, direct calls to the GitHub REST API to list Actions workflow runs failed with:

```
The underlying connection was closed: An unexpected error occurred on a receive.
No credentials are available in the security package
```

**Why it mattered**
- The GitHub MCP toolset does not expose Actions run logs, so we needed another way to debug failing workflows.

**Fix**
- Use the Chrome DevTools MCP browser to open:
  - `https://github.com/AndyKhris/Telegram_Test/actions`
  - and the failing run pages under `/actions/runs/<id>`
- Even without signing in, the UI showed the key validation errors under **Annotations**.

### Issue D - `codex-request-review` workflow was invalid: `secrets` not recognized in `if`
**What happened**
- GitHub Actions marked the workflow run as invalid with:

```
Invalid workflow file: .github/workflows/codex-request-review.yml#L1
(Line: 44, Col: 13): Unrecognized named-value: 'secrets'. Located at position 1 within expression: secrets.CODEX_REVIEW_TOKEN != ''
(Line: 134, Col: 13): Unrecognized named-value: 'secrets'. Located at position 1 within expression: secrets.CODEX_REVIEW_TOKEN == ''
```

This came from step-level `if:` conditions like:
- `if: ${{ secrets.CODEX_REVIEW_TOKEN != '' }}`

**Fix**
- Copy the secret into a job-level env var and gate on `env` instead:
  - Job env:
    - `CODEX_REVIEW_TOKEN: ${{ secrets.CODEX_REVIEW_TOKEN }}`
  - Step conditions:
    - `if: ${{ env.CODEX_REVIEW_TOKEN != '' }}`
    - `if: ${{ env.CODEX_REVIEW_TOKEN == '' }}`

After this change, `codex-request-review` successfully posted `@codex review` using the PAT.

### Issue E - `codex-automerge` workflow was invalid: `on: reaction` not supported
**What happened**
- We attempted to trigger auto-merge on thumbs-up reactions using:
  - `on: reaction: types: [created]`
- GitHub Actions rejected it with:

```
Invalid workflow file: .github/workflows/codex-automerge.yml#L1
(Line: 6, Col: 3): Unexpected value 'reaction'
```

**Fix**
- Remove the `reaction` trigger entirely.
- Rely on comment/review events from `chatgpt-codex-connector[bot]`:
  - `issue_comment`, `pull_request_review`, `pull_request_review_comment`.

**Consequence**
- If the connector only reacts with a thumbs-up and does not comment, GitHub Actions cannot be triggered (there's no `reaction` event), so we can't auto-merge on "reaction-only" outcomes.
- In our testing the connector replied with a comment, so the flow worked.

### Issue F - Codex connector did not follow the requested structured `CODEX_AUTOMERGE_V1` block
**What happened**
- You requested that Codex respond in a strict format:
  - `CODEX_AUTOMERGE_V1`
  - `RESULT: PASS|FAIL`
  - `P0_COUNT`, `P1_COUNT`, `AUTOMERGE`
- The connector's actual response was typically a natural-language sentence like:
  - "Codex Review: Didn't find any major issues..."

**Fix**
- Keep support for the structured block (in case it appears), but add a phrase-based fallback:
  - Regex match for "didn't find any major issues" (case-insensitive; supports `'` and "did not").

### Issue K - `codex-automerge` didn't run because the connector posted review comments (not issue comments)
**What happened**
- Initially, `codex-automerge` only listened to `issue_comment`.
- The Codex connector sometimes posts feedback as:
  - PR review summaries (`pull_request_review`), and/or
  - inline review comments (`pull_request_review_comment`)
- In those cases, the workflow never fired, so no `autofix-needed` label was applied and nothing merged.

**Fix**
- Expand triggers and dispatch logic in `.github/workflows/codex-automerge.yml` to handle:
  - `issue_comment` + `pull_request_review` + `pull_request_review_comment`
- Normalize to a single `(prNumber, triggerBody)` inside the script so the same merge/label logic applies to all three.

### Issue G - `push_files` sometimes failed (422 "Tree SHA does not exist")
**What happened**
- While iterating on workflow files via GitHub MCP, pushing multiple files in one commit sometimes failed with:
  - `422 Tree SHA does not exist`

**Fix**
- Use `create_or_update_file` to update individual files (reliable for small edits), instead of batching via `push_files`.

### Issue L - `git push` rejected (non-fast-forward) when publishing the final workflows to `main`
**What happened**
- A `git push origin main` attempt failed with:

```
! [rejected]        main -> main (fetch first)
error: failed to push some refs
```

This happens when:
- `origin/main` has commits you don't have locally, and/or
- your local `main` has commits that are not on `origin/main` (diverged history).

**Fix**
- Fetch `origin/main`, then re-apply only the intended workflows/docs commit onto the current remote head:
  - `git fetch origin`
  - `git branch backup/local-main-pre-origin-sync-<date>`
  - `git reset --hard origin/main`
  - `git cherry-pick <commit_sha>`
  - `git push origin main`

**Why this fix**
- It avoids accidentally pushing unrelated local-only commits when your local `main` is not in sync with the remote.

### Issue H - Workflow appeared to run but didn't comment (root cause: invalid workflow file)
**What happened**
- Before we found Issue D, we observed:
  - PRs labeled `automerge`
  - No `@codex review` comment appeared
  - No diagnostics appeared in PR comments

**Fix**
- Once we inspected the Actions run UI, we discovered the workflow never ran correctly because it was invalid (`secrets` context issue).
- After fixing the workflow validation errors, the comment started appearing immediately.

### Issue I - External bot noise (CodeRabbit)
**What happened**
- CodeRabbit posted its own review comments and commit statuses, sometimes rate limiting reviews after many commits in a short time.

**Fix**
- No code change required; not part of the Codex workflow.
- Just be aware it can add noise when you're scanning PR comments to see whether Codex responded.

### Issue J - Label detection needed to be robust across PR event types
**What happened**
- We initially assumed `github.event.pull_request.labels` (or the `labeled` payload) would always be enough to decide whether a PR has the `automerge` label.
- In practice, different `pull_request_target` actions (`opened`, `synchronize`, `labeled`, etc.) can provide different payload shapes, and relying on the payload alone is fragile.

**Fix**
- `codex-request-review` calls `github.rest.pulls.get(...)` and inspects `pr.labels` from the API response (source of truth) before commenting.

---

## 6) Verification / test PRs (what we ran)

We validated the final setup by opening and merging test PRs:

- PR **#12** (2025-12-23):
  - `codex-request-review` posted `@codex review` as `AndyKhris`
  - `chatgpt-codex-connector[bot]` replied "Didn't find any major issues..."
  - `codex-automerge` merged (squash) and posted a `CODEX_AUTOMERGE_V1` summary

- PR **#13** (2025-12-23):
  - Same end-to-end success

This proves:
- PAT-based triggering works (`CODEX_REVIEW_TOKEN`)
- The connector replies when invoked
- Auto-merge triggers correctly on the connector's comment

- PR **#14** (2025-12-24):
  - Started with an intentionally broken file to force a non-pass review
  - `codex-automerge` applied label `autofix-needed`
  - After the fix was pushed and `@codex review` was re-requested, the connector replied "Didn't find any major issues..."
  - `codex-automerge` removed `autofix-needed` and merged (squash)

---

## 7) Day-to-day usage (how you should run it)

1. Do work in a feature branch (however you prefer: local git, GitHub MCP, etc).
2. Open a PR to `main`.
3. Apply label `automerge`.

What happens next:
- `codex-request-review` posts `@codex review` (once).
- Codex connector replies on the PR (comment).
- `codex-automerge` merges if the response indicates "pass".

If you do **not** want auto-merge:
- Don't apply the `automerge` label.

If the PR gets label `autofix-needed`:
- Fix the issue(s) on the same PR branch and push.
- Then re-request the review by commenting `@codex review` again (as the PR author).
  - `codex-request-review` intentionally does not post `@codex review` more than once per PR.
- Once the connector posts a "pass" signal, `codex-automerge` will remove `autofix-needed` and merge.

---

## 8) Notes / recommended cleanup

The current `codex-request-review` includes a diagnostic step:
- "Debug status (PR 11 only)"

It was used to prove the workflow started while debugging. It's safe (and `continue-on-error`), but you can remove it once you no longer need it.
