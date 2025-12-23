# Repository Guidelines

## Project Structure & Module Organization
- Planning assets live in `Codex_Plan/` and `ClaudeCode_Plan/` (PRDs, Graphviz/Mermaid diagrams). The latest iteration is `Codex_Plan/v3/`.
- When code is added, use:
  - `src/` — FastAPI app and modules
    - `src/telegram/` (webhook, parsing)
    - `src/jobs/` (queue + job store)
    - `src/providers/` (Replicate, fal adapters via common interface)
    - `src/config/` (settings via BaseSettings)
  - `tests/` — pytest suites mirroring `src/` layout
  - `assets/` — static files, exported diagrams

## Build, Test, and Development Commands
- Run dev API: `uvicorn src.app:app --reload` (once code exists).
- Tests: `pytest -q` (unit and integration).
- Lint/format (optional): `ruff check .` and `black .`.
- Render diagrams (Graphviz):
  - Windows: `"C:\\Program Files\\Graphviz\\bin\\dot.exe" -Tpng Codex_Plan/v3/architecture_v3.dot -o Codex_Plan/v3/architecture_v3.png`
  - Cross‑platform: `dot -Tpng <in.dot> -o <out.png>`

## Coding Style & Naming Conventions
- Python, 4‑space indent, type hints required.
- Files and functions: `snake_case`; classes: `PascalCase`.
- Keep modules small and focused; avoid side effects on import.
- Public interfaces: `ImageProvider.generate(...)`, `ImageProvider.edit(...)`.
- Configuration via `pydantic.BaseSettings`; no hard‑coded secrets.

## Testing Guidelines
- Use `pytest`; place tests under `tests/` with names like `test_webhook.py`, `test_replicate_adapter.py`.
- Prefer unit tests for parsing, adapters, and job transitions; mock network calls.
- Aim for clear assertions over exhaustive coverage.

## Commit & Pull Request Guidelines
- Use concise, action‑oriented commits (e.g., `feat: add fal edit adapter`).
- PRs must include: purpose, summary of changes, how to test, and updated docs/diagrams when architecture changes.
- Link issues and add screenshots or sample commands when relevant.

## Review guidelines
- test workflow 3
- test workflow token 4
- test workflow token 5
- Always include the following fixed block in agent responses (final output):
  ```
  CODEX_AUTOMERGE_V1
  RESULT: PASS|FAIL
  P0_COUNT: <int>
  P1_COUNT: <int>
  AUTOMERGE: ENABLE|SKIP
  ```

## Security & Configuration Tips
- Required env vars (when code exists): `TELEGRAM_BOT_TOKEN`, `REPLICATE_API_TOKEN`, `FAL_API_KEY`.
- Never commit secrets; avoid logging prompts with PII; enforce SSRF guards when downloading Telegram media.

## Architecture Overview
- Non‑blocking webhook acks quickly; work runs in background jobs.
- Provider adapters normalize calls to Replicate (generate) and fal (edit).
- Idempotency (dedupe on `update_id`), rate limits, and structured logs are mandatory in `src/`.
