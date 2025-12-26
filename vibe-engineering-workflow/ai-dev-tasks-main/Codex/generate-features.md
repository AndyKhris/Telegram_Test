---
description: Enumerate platform/feature/ops slices and write feature placeholders under /tasks/features
argument-hint: [<file_or_folder_paths...>]
---

# generate-features.md
> Step 0 of the 4-file workflow. Enumerate capabilities and save one **feature placeholder** per slice.
> This step does **not** create PRDs or task lists and does **not** modify any other rule files.

---

## GOAL
Turn a product idea + architecture into a concrete set of **vertical slices** that can be handed off to `create-prd.md`.
Each slice gets a tiny placeholder in `/tasks/features/`, dovetailing with:
`create-prd.md` → `generate-tasks.md` → `process-task-list.md`.

---

## INPUTS (from repo context)

Treat **all positional arguments** as paths (files or folders) relative to repo root:
$ARGUMENTS

- If a path is a **file**, read it.
- If a path is a **folder**, scan it (up to 2 levels deep; do not follow symlinks) and read high-signal files first.
  - Ignore: `.git/`, `node_modules/`, `.venv/`, `venv/`, `dist/`, `build/`, `.next/`, `__pycache__/`
  - Prefer file types in this order: `.md`, `.txt`, `.dot`, `.mmd`, `.csv`, `.png`(use OCR only if available; otherwise skip + report)
  - If > 25 candidates: prefer filenames containing `charter`, `requirements`, `architecture`, `design`, `plan`, `spec`, `notes` and read up to ~12.

Also use repo context if present: Product charter/goals, architecture notes/diagrams, constraints, and existing `/tasks/features/` placeholders.

> If some inputs are missing, proceed best-effort. If `$ARGUMENTS` is empty, ask the user for a directory or files (or a short pasted product description).

---

## PROCESS
1) **Ingest** the product idea, architecture, and constraints.
   - Read inputs from: $ARGUMENTS (files and/or folders per the rules above)
   - Then skim any other relevant repo context (charter, architecture docs, constraints).

2) **Ideate** feature slices using this taxonomy:
   - **Platform** — cross-cutting capabilities that enable features (keep **thin**, only what the next feature needs).  
   - **Feature** — **vertical slices** a user can feel (UI → API → worker → delivery)
   - **Ops** — delivery modes, observability, cost/quotas, storage/CDN, admin tooling

3) **Propose** a numbered list of 6–12 slices (names + one-line outcome), in **recommended build order**:
   - Start with a thin **Platform** slice that provides the core plumbing needed for the first vertical feature (e.g., basic webhook / API endpoint, minimal job handling).
   - Then define a thin **walking-skeleton Feature**: one end-to-end vertical slice that exercises the main path (e.g., request in → core processing → result out) using that platform plumbing.
   - Plan additional **Platform** slices as just-enough enablers placed **immediately before** the vertical Features that depend on them, instead of a big “platform-only” phase.
   - Introduce **Ops** slices after at least one vertical Feature exists, to add logging, metrics, delivery modes, storage, and other operational hardening.
   - **Include a final Ops slice** for **Release Smoke/E2E Suite** (post-deploy verification) as the last item in the list.
     - The PRD for this slice should define: environments (local/staging/prod), safety constraints, and pass/fail criteria.
     - The tasks for this slice should produce a runnable smoke entrypoint (e.g., `scripts/smoke.*`) and tests under `tests/smoke/` and/or `tests/e2e/` with a single command that exits non-zero on failure.
     - The suite must be safe-by-default (no spamming public chats, no destructive actions) and must not log secrets.
   - If the repo has no explicit test layout, default to:
     - Unit: `tests/unit/`
     - Integration: `tests/integration/`
     - Smoke: `tests/smoke/`
     - E2E: `tests/e2e/`
   - In general, favor shipping vertical **Features** early, extending and hardening Platform and Ops as later Features demand it.

   **Ask:** “I’ve drafted the high-level features and a suggested build order (thin platform plumbing first, then a walking-skeleton vertical feature, followed by further vertical slices with just-enough platform and ops to support them). Reply **Go** to create files, or suggest edits.”

4) **Wait** for user confirmation. If the user replies **Go**, continue; otherwise, revise the list and re-ask.

5) **Create placeholders** for each approved slice:
   - Name each in **kebab-case** (e.g., `bot-core-webhook`, `generate-images`)
   - Save to `/tasks/features/[n]-feature-[feature-name].md` (see numbering)

6) **Write index** (recommended): `/tasks/features/index.md` as a table:
   `ID | Type | Name | Outcome | Dependencies | Status | File`
   The rows should follow the **recommended build order** from step 3.

7) **Stop.** Do not create PRDs or task lists.

---

## OUTPUT
- **One feature file per slice** `/tasks/features/[n]-feature-[feature-name].md`
- `n` is a **zero-padded 4-digit** feature sequence only (PRDs reuse the same n as the feature placeholder).  
  Scan existing files matching `\d{4}-feature-*.md`; pick `max(n)+1`, else start `0001`.
- **Do not overwrite** existing placeholders unless explicitly instructed.
- **Optional** `/tasks/features/index.md` listing all slices (helpful, but not required by the rest of the system).

> PRDs are always created later by `create-prd.md` as `/tasks/[n]-prd-[feature-name].md`.

---

## OTHER INSTRUCTIONS
- Only write files under `/tasks/features/`.
- Do not create or edit anything else under `/tasks/`.
- If `/tasks/features/` is missing, create it.
- Keep each placeholder ≤ ~1–2 pages.

---

## OUTPUT FORMAT (for each placeholder file)

```markdown
# <Feature Name>  (<Type: Platform|Feature|Ops>)
ID: <F-000x> | Status: placeholder | Priority: <P1/P2/P3> | Size: <S/M/L>
Links: <architecture doc>, <design>, <related issues>

## Problem
<1–2 sentences: who is blocked and why>

## Outcome (Success Criteria)
<1 sentence with an observable behavior or metric>

## Scope
**In:**
- <bullet 1>
- <bullet 2>

**Out:**
- <explicitly not included>

## Flow (happy path)
1) <actor/event>
2) <system reaction>
3) <completion behavior>

## Dependencies
**Upstream:** <platform/infra needed first>  
**Downstream:** <who consumes this output next>

## Interfaces / Contracts (seeds)
**Inputs (shape):** `<name>` → <summary>  
**Outputs (shape):** `<name>` → <summary>  

## Acceptance Seeds (Given/When/Then)
- Given …
- When …
- Then …

## Non-Functional / Telemetry (seeds)
- <performance, security/privacy, logging/metrics notes>

## Risks & Open Questions
- <key risks or things we are unsure about>

## PRD Handoff
Use `@create-prd.md` with:
"Here’s the feature I want to build: @/tasks/features/<ID>-feature-<kebab-name>.md"
