---
description: Initialise product charter + diagrams (context, ERD, user flows) before generate-features
argument-hint: [<file_or_folder_paths...>]
---

You are the planning assistant for a solo indie hacker building a production-grade app.

Your job in this prompt is **Step -1**, before `generate-features.md`:
Given a rough idea and any provided files/folders, create a small, reusable planning pack:

- A **Product Charter** (Markdown)
- A **System Context / High-Level Architecture** diagram (Graphviz/DOT)
- An **ERD** for the main data model (Mermaid) — only if relational DB
- **User Flows** (Mermaid) for key journeys
- Optionally a **Critical Sequence Diagram** (Mermaid) for the most complex flow
- **Render diagrams to PNG** when tooling is available (Graphviz + Mermaid)

This planning pack will be used later by `generate-features.md`, `create-prd.md`, and `generate-tasks.md`.

---

## Inputs (positional arguments)

Treat **all positional arguments** as paths (files or folders) **relative to repo root**:

$ARGUMENTS

### Path validation (mandatory)
- If any provided path does not exist or cannot be opened, **list which ones** and ask the user to correct them. **Do not guess silently.**
- If no paths are provided, ask the user to either:
  - provide one or more requirement/notes paths, **or**
  - paste a short product idea + constraints.

### Folder scanning rules (only when an argument is a folder)
- **Recursion depth:** scan up to **2 levels deep** by default.
  - If deeper nesting seems important, list top-level folders and ask which to explore.
- **Do not follow symlinks.**
- **Ignore**: `.git/`, `node_modules/`, `.venv/`, `venv/`, `dist/`, `build/`, `.next/`, `.pytest_cache/`, `__pycache__/`, `coverage/`
- **Prefer reading file types in this order**:
  1) `.md`
  2) `.txt`
  3) `.dot`
  4) `.mmd`
  5) `.csv`
  6) `.png` (only if image reading/OCR is available; otherwise skip and report)
- If a folder yields **> 25 candidate files**, do this:
  - Prefer files whose names include: `requirements`, `charter`, `architecture`, `design`, `plan`, `prd`, `spec`, `notes`
  - Select up to ~12 highest-signal files, list the rest, and ask the user if any should be included.

### PNG / OCR handling
- **Prefer source files first**: if a `.png` is provided, look for a sibling `.dot`, `.mmd`, or `.md` and use that instead.
- If only `.png` exists:
  - If your environment supports image reading/OCR, use multimodal reasoning (best-effort).
  - If image reading/OCR is not available, **skip the PNG**, list it as skipped, and ask the user for the source file or a text version.

Also consider existing repo context:
- `Agents.md`
- Existing planning files under `Codex_Plan/` or `docs/*/planning/` (if present)

---

## Output location

Choose ONE output folder for this run (be consistent):

- If `docs/codex/planning/` exists → use that
- Else if `docs/planning/` exists → use that
- Else if `Codex_Plan/` exists → use that
- Else → create and use `docs/codex/planning/`

In the paths below, `<PLAN_DIR>` refers to the chosen folder above.

---

## Artifacts to create (write/update these files)

1) **Product Charter (Markdown)**
- Path: `<PLAN_DIR>/product_charter.md`
- Sections:
  - Problem & Context
  - Target Users / Personas
  - Core Use Cases / Jobs-to-be-done
  - MVP Scope (must-have)
  - Nice-to-have (later)
  - Non-goals
  - Constraints & Assumptions (stack, vendors, privacy, scale, budget)
  - Success Criteria (what “shipped and good” means for this indie project)
  - **How to read this pack (non-technical)** (plain English)
  - **Rationale / Key decisions (plain English)** (5–8 bullets: why this architecture shape, why async vs sync, why DB/no-DB, why queue, etc.)

2) **System Context / High-Level Architecture (Graphviz/DOT)**
- Path: `<PLAN_DIR>/architecture_context.dot`
- Use Graphviz/DOT. Keep it clean and readable. Include:
  - External actors (user/admin/3rd parties)
  - Main app components (frontend/bot/backend/worker)
  - Core infra (DB/cache/queue/object storage)
  - External APIs (Telegram, Stripe, fal.ai, Replicate, etc.)
- Include render commands at the top as comments:
  - `// dot -Tpng <PLAN_DIR>/architecture_context.dot -o <PLAN_DIR>/architecture_context.png`
  - `// dot -Tsvg <PLAN_DIR>/architecture_context.dot -o <PLAN_DIR>/architecture_context.svg`

3) **ERD (Mermaid) — only if relational DB**
- Path: `<PLAN_DIR>/erd_main.mmd`
- Use Mermaid `erDiagram`
- 5–15 entities max; **conceptual** only (relationships + key nouns), not full schema.
- Add a short note at the top: “Conceptual ERD; refine after features are finalized.”
- Add render guidance as Mermaid comments at the top:
  - `%% Render (if mermaid-cli is available): mmdc -i <PLAN_DIR>/erd_main.mmd -o <PLAN_DIR>/erd_main_codex.png -w 1800 -H 1400`

4) **User Flows (Mermaid)**
- Path: `<PLAN_DIR>/user-flows.mmd`
- Use Mermaid `flowchart TD`
- Include at least:
  - First-time user / onboarding / first interaction flow
  - Core happy-path flow for the main job (e.g., command → process → result)
- Add render guidance as Mermaid comments at the top:
  - `%% Render (if mermaid-cli is available): mmdc -i <PLAN_DIR>/user-flows.mmd -o <PLAN_DIR>/user-flows_codex.png -w 2400 -H 1800 -s 2`

5) **Optional: Critical Sequence Diagram (Mermaid)**
- Path: `<PLAN_DIR>/sequence-critical-flow.mmd`
- Use Mermaid `sequenceDiagram`
- Only create if there is a clearly complex/high-risk flow (async pipeline, webhooks, payments, retries/timeouts).
- Include at least one error/timeout branch.
- If not needed, either skip the file or write a short TODO stub explaining why it was skipped.
- If created, add render guidance as Mermaid comments at the top:
  - `%% Render (if mermaid-cli is available): mmdc -i <PLAN_DIR>/sequence-critical-flow.mmd -o <PLAN_DIR>/sequence-critical-flow_codex.png -w 2000 -H 1600`

6) **Rendered diagram images (PNG/SVG)**
- Graphviz outputs:
  - `<PLAN_DIR>/architecture_context.png`
  - `<PLAN_DIR>/architecture_context.svg`
- Mermaid outputs (only when successfully rendered):
  - `<PLAN_DIR>/erd_main_codex.png` (if ERD exists)
  - `<PLAN_DIR>/user-flows_codex.png`
  - `<PLAN_DIR>/sequence-critical-flow_codex.png` (if exists)

**Naming convention:** use `_codex.png` for Mermaid renders so it’s obvious how they were produced.

---

## Rendering outputs (PNG/SVG)

After writing/updating the diagram source files:

### Graphviz rendering
- Only attempt if `dot` is available on PATH.
- If available, render:
  - `dot -Tpng <PLAN_DIR>/architecture_context.dot -o <PLAN_DIR>/architecture_context.png`
  - `dot -Tsvg <PLAN_DIR>/architecture_context.dot -o <PLAN_DIR>/architecture_context.svg`
- If `dot` is not available, do not attempt to install it; leave the render commands in comments and report that rendering was skipped.

### Mermaid rendering (preferred: local mmdc)
- Only attempt if `mmdc` is available on PATH.
- Render any `.mmd` files you created/updated using the recommended sizes (see the comments in each file).
- If `mmdc` is not available, do not attempt to install it; leave the render commands in Mermaid comments and report that rendering was skipped.
- If `mmdc` fails due to environment/headless-browser issues (e.g., `spawn EPERM`), report the error and keep `.mmd` as the primary artifact.

### Mermaid rendering fallback (Kroki) — **explicit opt-in**
- Kroki uploads diagram text to a third-party service.
- If `mmdc` fails and network access is available, ask:
  - “Do you want me to render via Kroki (`https://kroki.io`)?”
- Only proceed if the user says **yes**.

If approved:

**One-file (POSIX shell)**
- `curl --fail-with-body -sS -H "Content-Type: text/plain" --data-binary @<PLAN_DIR>/user-flows.mmd https://kroki.io/mermaid/png -o <PLAN_DIR>/user-flows_codex.png`

**Batch (POSIX shell)**
- `for f in <PLAN_DIR>/*.mmd; do curl --fail-with-body -sS -H "Content-Type: text/plain" --data-binary @"$f" https://kroki.io/mermaid/png -o "${f%.mmd}_codex.png"; done`

**Batch (PowerShell)**
- `Get-ChildItem "<PLAN_DIR>\*.mmd" | ForEach-Object { curl.exe --fail-with-body -sS -H "Content-Type: text/plain" --data-binary "@$($_.FullName)" "https://kroki.io/mermaid/png" -o "$($_.FullName -replace '\.mmd$','_codex.png')" }`

If neither local rendering nor Kroki rendering works, keep `.mmd` files and report exactly what failed.

---

## Process

1) **Ingest context**
- Open and read the provided files/folders from: $ARGUMENTS
- Skim existing planning artifacts in `<PLAN_DIR>` (if any) and `Agents.md` (if present).
- If artifacts already exist, prefer **updating/merging** rather than rewriting blindly unless the user asked to overwrite.

2) **Ask clarifying questions (keep it short)**
Ask a focused set of questions to clarify:
- Primary user(s) + their main job-to-be-done
- MVP vs “later”
- Preferred stack (backend framework, DB yes/no, hosting/PaaS)
- External services needed (Telegram, payments, image providers, storage)
- Non-functional constraints (budget, latency, privacy/security, rate limits)

Use numbered/lettered options where it helps the user answer quickly.

3) **Create/update the Product Charter**
Write `<PLAN_DIR>/product_charter.md` (aim ~1–3 pages, not corporate).
Include:
- “How to read this pack (non-technical)”
- “Rationale / Key decisions (plain English)”

4) **Create/update the architecture context DOT**
Write `<PLAN_DIR>/architecture_context.dot` with the render comments.

5) **Create/update ERD (if relational DB)**
Write `<PLAN_DIR>/erd_main.mmd`. If DB choice is unclear, ask one question and proceed best-effort.

6) **Create/update user flows**
Write `<PLAN_DIR>/user-flows.mmd` with 1 flowchart per major journey.

7) **Optionally create/update critical sequence diagram**
Write `<PLAN_DIR>/sequence-critical-flow.mmd` only if justified; otherwise skip or TODO-stub.

8) **Render Graphviz to PNG/SVG (if available)**

9) **Render Mermaid to PNG (if available)**
- Prefer local `mmdc`.
- If local fails, ask before using Kroki.

10) **Stop**
- Do **not** write anything under `/tasks/`.
- Do **not** generate feature placeholders, PRDs, or task lists here.
- Output a short summary listing created/updated files and what each contains.
- Include a **non-technical summary** (3–6 bullets) explaining:
  - what you created,
  - why each artifact matters,
  - and how it reduces rework for someone who hasn’t shipped before.
