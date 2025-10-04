# PRD v3 — Telegram Image Bot (Generate/Edit) with Non-Blocking Orchestrator

## Summary
- Telegram bot with two flows: /generate (text → image via Replicate) and /edit (image + instruction → edited image via fal.ai).
- Mini Orchestrator (FastAPI) replies fast to Telegram and runs slow work in background jobs.
- Backend controls all model parameters (“backend params”), not the user. No specific parameter names are exposed in UX or this PRD.
- Guardrails: idempotency (dedupe retries), simple rate limits/allowlist, safe Telegram media fetching, and structured logs/metrics.

## Goals (v1)
- Fast, reliable UX in Telegram: immediate “Working…” ack, then final images.
- Clear separation of concerns: webhook → policy → jobs → providers → delivery.
- Provider abstraction: map normalized inputs + backend params to each vendor.
- Operability: logs/metrics, retries/timeouts, simple status tracking.

## Non-Goals (v1)
- Exposing model parameters to users. All tuning remains backend-only.
- Persistent galleries or payments/credits (may be added later).
- Multi-node scaling requirements.

## Users & Commands
- Allowed users (allowlist) can use:
  - `/generate <prompt>` — create new images from text.
  - `/edit <instruction>` — reply to, or caption with, a photo to edit it.
- The bot immediately responds in chat with “Generating…” or “Editing…”.

## Flows (Plain English)
### /generate
1) User sends `/generate A cozy cabin in snow`.
2) Webhook validates the user, applies idempotency (dedupe on Telegram retry), and enqueues a job.
3) Bot sends “Generating…” quickly and returns HTTP 200 to Telegram.
4) Background worker processes the job: applies backend policy (params), maps inputs to the vendor, calls the provider, and waits for images.
5) Bot sends the image(s) back to the chat; job marked succeeded or failed.

### /edit
1) User uploads a photo with caption `/edit make it sunset`, or replies to a photo with that command.
2) Webhook validates, dedupes, and enqueues a job with the Telegram `file_id` and instruction.
3) Bot sends “Editing…” and returns HTTP 200.
4) Background worker resolves the `file_id` via Telegram getFile to a trusted URL (SSRF guard: only Telegram file hosts), applies backend policy, maps inputs to the vendor, calls the provider, and waits for results.
5) Bot sends the edited image(s) to the chat; job marked succeeded or failed.

## Architecture Components
- Webhook & Command Router: parses `/generate` and `/edit`; replies fast.
- Idempotency & Rate Limits: dedupe on Telegram update_id; allowlist + daily caps.
- Backend Params & Policy: central place for defaults and clamps (generic, not user-exposed).
- Provider Mapper: converts normalized inputs + backend params to each vendor’s API.
- Telegram Media Fetcher: resolves Telegram `file_id` → getFile → trusted URL; blocks unsafe hosts.
- Job Queue: executes slow work off the webhook path.
- Job Store: tracks job states (queued/running/succeeded/failed), timings, and errors.
- Result Delivery: chooses sendPhoto, sendMediaGroup, or sendDocument appropriately and posts to Telegram.
- Logs & Metrics: structured logs with job_id; counts, latency, retries, provider errors, basic cost estimate.

## Non-Blocking Behavior
- Webhook finishes quickly (parse → guard → enqueue → ack), avoiding Telegram timeouts.
- All slow operations (provider calls, downloads, uploads) run in background workers.
- Users see instant feedback and receive the final media when ready.

## Provider Abstraction
- Normalized operations:
  - Generate(prompt, backend params) → [ImageResult]
  - Edit(image, instruction, backend params) → [ImageResult]
- The mapper translates these into Replicate/fal.ai requests without exposing vendor details.

## Telegram Media Handling (/edit)
- Accept photo via captioned upload or as a reply target.
- Store only the Telegram `file_id` in the job; worker resolves it later.
- SSRF guard: allow only Telegram file hosts; block private IP ranges.

## Reliability & Safety
- Idempotency on Telegram update_id to prevent duplicate jobs/charges.
- Basic allowlist and per-user caps to prevent abuse.
- Timeouts and retries with exponential backoff for provider calls.
- Friendly error messages to users; detailed logs for operators.

## Observability
- Logs: structured JSON with request_id/job_id, user, provider, duration, outcome, error.
- Metrics: job counts, success/failure rates, latency, retry counts, rough cost estimate.

## API Surface
- `POST /telegram/webhook` — main entry, immediate 200 after enqueue + ack.
- `GET /healthz` — readiness.
- `GET /jobs/{id}` — optional debugging/status endpoint.

## Acceptance Criteria
- Webhook responds < 1s with a “Working…” message in chat.
- Background job completes and returns generated/edited image(s) to the same chat.
- Duplicate Telegram retries do not create duplicate jobs or charges (idempotency verified).
- Allowlist and daily caps enforced; errors are returned to users and logged with job_id.

## Future Extensions (optional)
- Storage/CDN for long-lived sharing; QA/scoring; credits/payments.
- ComfyUI provider for advanced workflows.
