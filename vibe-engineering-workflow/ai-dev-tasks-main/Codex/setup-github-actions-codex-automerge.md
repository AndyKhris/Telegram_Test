---
description: One-time setup for Codex Review + auto-merge GitHub Actions in this repo
argument-hint: [DEFAULT_BRANCH=main] [EXECUTE=yes|no]
---

# Setup: GitHub Actions (Codex review trigger + auto-merge)

Use this prompt to do the **one-time repo setup** required for the GitHub PR loop:
- `codex-request-review` posts a human-authored `@codex review` comment (once per PR) when label `automerge` is present.
- `codex-automerge` merges the PR when the Codex connector indicates a clean review; otherwise it adds label `autofix-needed`.
- `ci-tests` runs lint + tests on each PR in a clean environment (recommended default safety gate).

Reference (detailed runbook + historical issues): `github-actions/github-actions_setup.md`.

## Inputs
Use `$ARGUMENTS`.
- `DEFAULT_BRANCH` (default: `main`)
- `EXECUTE`:
  - `no` = plan/instructions only
  - `yes` = perform changes using available tools (ask for confirmation before GitHub writes)

## Rules
- Do not print secrets or tokens.
- Ask for confirmation before any GitHub write action (committing workflow files, creating PRs, labels, secrets instructions).
- Do not run PR code in `pull_request_target` workflows (security).

## Step 1) Pre-flight checks
1. Confirm GitHub identity and repo:
   - Use GitHub MCP to confirm the authenticated user and target repo.
2. Confirm the Codex GitHub App is installed:
   - The bot should appear as `chatgpt-codex-connector[bot]`.
   - Verify on any PR by commenting `@codex review` and seeing a reply (or a thumbs-up reaction when it has no suggestions).
3. Confirm GitHub Actions are enabled for the repo.

## Step 2) Required GitHub UI settings (manual)
These are one-time settings you must do in GitHub:

1. Add Actions secret `CODEX_REVIEW_TOKEN`
   - Create a fine-grained PAT for your human GitHub user:
     - Repository access: this repo only
     - Permissions (minimum):
       - Issues: Read and write (PR comments are "issue comments")
       - Pull requests: Read (workflow reads PR metadata before commenting)
   - Add it in: Repo -> Settings -> Secrets and variables -> Actions -> New repository secret:
     - Name: `CODEX_REVIEW_TOKEN`
     - Value: (the PAT)

2. Workflow permissions for `GITHUB_TOKEN`
   - Repo -> Settings -> Actions -> General -> Workflow permissions:
     - Select **Read and write permissions**
   - Without this, `codex-automerge` cannot apply labels or merge PRs.

3. Ensure label `automerge` exists
   - Repo -> Issues -> Labels:
     - Create label `automerge` if missing.
   - Note: labels `autofix-needed` and `ci-failed` are created automatically by workflows when needed.

4. Optional (recommended) repo settings
   - Allow auto-merge (if you want GitHub-native auto-merge behavior):
     - Repo -> Settings -> General -> Pull Requests -> enable "Allow auto-merge"
   - Automatically delete head branches (cleanup):
     - Repo -> Settings -> General -> Pull Requests -> enable "Automatically delete head branches"
   - Branch protection / required checks (recommended default):
     - Create a branch protection rule for `DEFAULT_BRANCH` with:
       - Require a pull request before merging
       - Require status checks to pass before merging
       - Select the CI check from `ci-tests.yml` (typically shown as `CI Tests / ci-tests`)
       - (Optional) Require 1 approval (Codex connector review counts if your setup enforces reviews)
     - With required checks enabled, `codex-automerge` should enable auto-merge after a clean Codex review, and GitHub will merge only after CI is green.


## Step 3) Add workflow files in the correct location
Ensure these exist in `.github/workflows/` on `DEFAULT_BRANCH`:
- `.github/workflows/codex-request-review.yml`
- `.github/workflows/codex-automerge.yml`
- `.github/workflows/ci-tests.yml`
- `.github/workflows/ci-tests-autofix-needed.yml`

Notes:
- `codex-request-review` must use `pull_request_target` and must not checkout PR code.
- `codex-request-review` must post `@codex review` using `CODEX_REVIEW_TOKEN` (the connector often ignores `github-actions[bot]`).
- `codex-request-review` should dedupe so it comments **only once** per PR.
- `codex-automerge` should listen to connector feedback across surfaces:
  - `issue_comment`, `pull_request_review`, and `pull_request_review_comment`
- `codex-automerge` should:
  - If non-pass: add label `autofix-needed` (create label if missing) and do not merge
  - If pass: remove `autofix-needed` (if present) and merge (or enable auto-merge)
- `ci-tests-autofix-needed` should:
  - When `CI Tests` fails on a PR labeled `automerge`, add labels `autofix-needed` and `ci-failed` (create labels if missing)

If `EXECUTE=yes`:
- Ask for confirmation, then write/update these files on `DEFAULT_BRANCH` (via git push or GitHub MCP write tools).

## Step 4) Verification (recommended)
Create two short-lived test PRs to validate both paths:

1. PASS path
   - Make a tiny safe change (e.g., a comment/doc update).
   - Open PR and add label `automerge`.
   - Expect:
     - `codex-request-review` posts `@codex review` (human-authored)
     - `chatgpt-codex-connector[bot]` replies
     - `ci-tests` runs and passes
     - `codex-automerge` enables auto-merge (and GitHub merges once required checks pass) and posts a `CODEX_AUTOMERGE_V1` summary comment

2. FAIL path
   - Open PR with an intentional issue that the connector will flag.
   - Add label `automerge`.
   - Expect:
     - `codex-automerge` adds label `autofix-needed` and does not merge
   - Then fix it and push to the same PR branch.
   - Important: re-request review by commenting `@codex review` again (the request workflow only posts once per PR).
   - Expect:
     - `autofix-needed` removed
     - PR merged

## Troubleshooting checklist
- `codex-request-review` didn't comment:
  - Confirm `CODEX_REVIEW_TOKEN` exists and has Issues: write for this repo.
  - Confirm PR has label `automerge` and is not draft.
- `codex-automerge` didn't merge / couldn't label:
  - Confirm Actions workflow permissions are set to Read+Write.
  - Confirm PR is from same repo (not fork) and author is allowlisted in the workflow.
- Connector only reacts with a thumbs-up (no comment/review):
  - Actions cannot trigger on reactions; you need the connector to post a comment/review for automation.

