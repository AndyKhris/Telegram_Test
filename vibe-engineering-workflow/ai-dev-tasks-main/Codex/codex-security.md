---
description: Security check for a project (pre-deploy by default)
argument-hint: [<project_path>] [SCOPE=pre|post|both] [EXECUTE=yes|no]
---

# /security (generic)

You are the security check assistant. Your goal is to **identify likely security risks** before deployment and produce **actionable fixes**. Default to **pre-deploy** checks unless `SCOPE=post` or `SCOPE=both`.

## Inputs
Use `$ARGUMENTS`. If no path provided, assume repo root.

## Rules
- **Do not print secrets** (redact tokens/keys).
- **Do not modify files** unless explicitly asked.
- **Ask for confirmation before** any: installs, network calls, or destructive actions.
- Prefer **instructions-only** if `EXECUTE=no` or if unsure.

## Process (short)
1) **Detect** stack + entrypoints.
2) **Pre-deploy checks (default)**:
   - Secrets hygiene: `.env` in gitignore, hardcoded keys, unsafe sample values.
   - Auth & access: missing auth on sensitive routes, admin endpoints exposed.
   - Config risks: debug mode, permissive CORS, open binds without auth.
   - Dependency risks: outdated or known-vulnerable deps (if lockfiles exist).
   - Input handling: missing validation for external inputs.
3) **Post-deploy checks (optional)**:
   - TLS/HTTPS required, security headers, public endpoints reachable.
   - Webhook endpoints protected/validated where applicable.
4) **Summarize** findings with severity and fixes.
5) **Confirm** before running any command-based checks.

## Default test locations (if no repo convention)
- Unit: `tests/unit/`
- Integration: `tests/integration/`
- E2E/Smoke: `tests/e2e/`

## Output
- **Findings**: list with severity (Critical/High/Medium/Low)
- **Fixes**: short, concrete remediation steps
- **Verify**: minimal checks to confirm fixes
- **Next steps**: optional hardening items

## Confirmation format
- Before running commands: **Run this? (y/n)**

## Clarifying questions (only if needed)
- Pre-deploy only, post-deploy only, or both?
- Execute commands or provide instructions only?
