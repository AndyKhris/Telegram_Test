---
description: Deploy any project - detect stack, start services, verify health, configure integrations
argument-hint: "[<file_or_folder_paths...>] [--dry-run]"
---

# Deploy: Start, Verify & Operate

## Goal

Given a project (any language/framework), detect its components, start all services in correct order, verify health, configure external integrations (webhooks, callbacks), and produce an operational runbook.

This is an **action-oriented** prompt - it executes commands and verifies results, not just generates documentation.

---

## Inputs

Use `$ARGUMENTS`. If no path provided, assume repo root.

### Scanning Rules

- Scan **2 levels deep** max
- **Ignore:** `.git/`, `node_modules/`, `.venv/`, `venv/`, `__pycache__/`, `dist/`, `build/`, `target/`, `.next/`, `vendor/`
- **Prioritize reading:**
  - Dependency files: `package.json`, `requirements.txt`, `go.mod`, `Cargo.toml`, `Gemfile`, `pom.xml`, `composer.json`
  - Config files: `.env.example`, `.env`, `config.*`, `settings.*`
  - Deployment: `Dockerfile`, `docker-compose.yml`, `Makefile`, `Procfile`, `fly.toml`, `railway.json`
  - Entry points: `main.*`, `app.*`, `server.*`, `index.*`, `manage.py`

### Optional Flags

| Flag | Effect |
|------|--------|
| `--dry-run` | Show commands without executing |

---

## Process

### Phase 1: Detect & Analyze

**Step 1.1: Detect Technology Stack**

| Indicator | Stack |
|-----------|-------|
| `package.json` | Node.js |
| `requirements.txt`, `pyproject.toml`, `setup.py` | Python |
| `go.mod` | Go |
| `Cargo.toml` | Rust |
| `Gemfile` | Ruby |
| `pom.xml`, `build.gradle` | Java/Kotlin |
| `composer.json` | PHP |
| `*.csproj`, `*.sln` | .NET |
| `mix.exs` | Elixir |

**Step 1.2: Detect Runnable Components**

Find entry points by scanning for:
- Web server / API entry
- Background workers / job processors
- Scheduled tasks / cron jobs
- Database migrations
- CLI commands

**Step 1.3: Detect External Dependencies**

Look for connection strings, URLs, or client initialization for:
- Databases (PostgreSQL, MySQL, MongoDB, SQLite, Redis)
- Message queues (RabbitMQ, Kafka, SQS)
- Object storage (S3, GCS, Cloudinary)
- Third-party APIs (look for API key patterns in `.env.example`)

**Step 1.4: Detect Deployment Tooling**

Check if project already has:
- `Dockerfile` → Docker deployment available
- `docker-compose.yml` → Multi-container setup
- `Makefile` → Make commands available
- `Procfile` → Heroku-style process definitions
- `scripts/` or `deployment/` → Existing scripts
- CI/CD configs (`.github/workflows/`, `.gitlab-ci.yml`)

---

### Phase 2: Pre-Flight Validation

Present findings and validate environment:

```markdown
## Pre-Flight Check

### Detected Stack
- **Language:** [detected]
- **Framework:** [detected]
- **Package Manager:** [detected]

### Components Found
| Component | Entry Point | Type |
|-----------|-------------|------|
| [name] | [file:line] | API / Worker / Cron |

### External Dependencies
| Service | Required For | Status |
|---------|--------------|--------|
| [service] | [feature] | Checking... |

### Environment Variables
| Variable | Required | Status |
|----------|----------|--------|
| [VAR] | Yes/No | Set / Missing / Placeholder |

### Potential Issues
- [issue]: [description] → [fix]
```

**Check for blockers:**
- Missing `.env` file → Stop, ask user to create
- Missing dependencies (packages not installed) → Offer to install
- Port already in use → Ask: kill existing or use different port?
- Required services unreachable → Warn, ask to proceed or fix

**If blockers exist:** Stop and present fixes. Resume after user confirms fixed.

**If warnings only:** Present and ask "Proceed? (y/n)"

---

### Phase 3: Start Services

Start in dependency order:

```
1. External services (if locally managed)
2. Primary application (API/web server)
3. Secondary processes (workers, schedulers)
4. Tunnel / reverse proxy (if needed for webhooks)
5. Post-start configuration (set webhooks, register callbacks)
```

**For each component:**

1. **Announce:** "Starting [component]..."
2. **Command:** Show exact command being run
3. **Execute:** Run command (unless `--dry-run`)
4. **Wait:** Pause for service initialization
5. **Verify:** Confirm process running (check PID, port, or health endpoint)
6. **Report:** ✅ Success with details, or ❌ Failure with error

**Output format:**

```markdown
## Starting Services

### 1. [Component Name]
**Command:**
```[shell]
[exact command]
```
**Status:** ✅ Running
**Details:** PID [pid], Port [port]

### 2. [Next Component]
...
```

---

### Phase 4: Verification

After all services started, verify deployment works end-to-end:

**Step 4.1: Health Checks**

- Local endpoint (e.g., `/health`, `/healthz`, `/api/health`, `/`)
- External endpoint (via tunnel if applicable)

**Step 4.2: Integration Checks**

- If webhook-based: Verify webhook registered (e.g., `getWebhookInfo` for Telegram)
- If database-backed: Verify connection (query works)
- If queue-backed: Verify worker consuming (check logs or queue length)

**Step 4.3: Smoke Test**

Suggest ONE minimal test the user can perform:
- "Send [command] to your bot"
- "Open http://localhost:[port] in browser"
- "Run: `curl [endpoint]`"

**Output format:**

```markdown
## Verification

| Check | Status | Details |
|-------|--------|---------|
| Local health | ✅ / ❌ | [response or error] |
| External health | ✅ / ❌ | [response or error] |
| [Integration] | ✅ / ❌ | [details] |

### Smoke Test
[Instruction for user to manually verify]
```

---

### Phase 5: Failure Handling

If any step fails:

**Step 5.1: Collect Diagnostics**
- Capture command output (stdout + stderr)
- Check logs if log files exist
- Check port/process status

**Step 5.2: Analyze & Present**

```markdown
## Deployment Failed

### Failed Step
[Component name] - [Step description]

### Error Output
```
[captured output]
```

### Likely Cause
[Analysis based on error patterns]

### Suggested Fix
```[shell]
[fix command]
```

### After Fixing
Reply "retry" to attempt again, or "skip" to continue without this component.
```

**Step 5.3: Retry Protocol**
- Never auto-retry
- Wait for user confirmation that issue is fixed
- Re-run only the failed step (not entire deployment)

---

### Phase 6: Rollback / Stop

Provide commands to stop everything:

```markdown
## Stop / Rollback

### Stop All Services
```[shell]
[commands to stop all running components]
```

### Stop Individual
| Component | Stop Command |
|-----------|--------------|
| [name] | [command] |

### Cleanup (if applicable)
- Remove webhook: [command]
- Reset state: [notes about what resets on restart]

### Rollback Notes
- [Any data considerations]
- [How to restore previous state if needed]
```

---

## Output

Produce four distinct output blocks:

### 1. Runbook

Exact commands in correct order, copy-pasteable:

```markdown
## Runbook

### Prerequisites
[One-time setup commands]

### Start (in order)
1. [command]
2. [command]
...

### Post-Start
[webhook configuration, etc.]

### Verify
[health check commands]
```

### 2. Status Block

Current state of deployment:

```markdown
## Status

| Component | State | PID | Port | URL |
|-----------|-------|-----|------|-----|
| [name] | Running/Stopped | [pid] | [port] | [url] |
```

### 3. Verify Block

Health check results:

```markdown
## Verification

- Local: [status] - [details]
- External: [status] - [details]
- [Integration]: [status] - [details]
```

### 4. Next Steps

What to do after successful deployment:

```markdown
## Next Steps

### Immediate
- [ ] [Test specific feature]
- [ ] [Monitor logs for errors]

### Before Sharing
- [ ] [Security hardening]
- [ ] [Rate limiting]
- [ ] [Error tracking]

### Production
- [ ] [Persistent storage]
- [ ] [Auto-restart setup]
- [ ] [Monitoring]
```

---

## Clarifying Questions

If detection is ambiguous, ask:

1. **Multiple entry points found - which to start?**
   - List detected options with descriptions
   - Allow "all" or custom selection

2. **Tunnel needed?**
   - "Does this app receive incoming webhooks or callbacks from external services?"
   - If yes: "What tunnel tool? (cloudflared / ngrok / localtunnel / other)"

3. **Existing deployment found:**
   - "Found [Dockerfile/Makefile/scripts]. Use existing, or generate new commands?"

4. **Port conflict detected:**
   - "Port [X] in use by PID [Y]. Kill it, use different port, or cancel?"

5. **Missing environment variables:**
   - "These variables are required but not set: [list]. Set them now, or proceed without?"

---

## Platform Handling

Detect OS and shell, adapt commands:

| Detected | Adaptations |
|----------|-------------|
| Windows + PowerShell | Use `.ps1` syntax, `\` paths |
| Windows + WSL/Git Bash | Use bash syntax, `/` paths |
| macOS / Linux | Use bash syntax, `/` paths |

For cross-platform projects, prefer commands that work in both (or provide both variants).

---

## Known Patterns by Stack

### Python
- Check for venv activation
- PYTHONPATH may need setting
- `.env` loading varies by framework

### Node.js
- Check node version against `.nvmrc` or `engines`
- `npm start` vs `node index.js` vs framework CLI

### Go
- May need `go build` first
- Binary in `./` or `./bin/`

### Ruby
- Bundle exec prefix
- Rails has specific commands

### Docker
- `docker-compose up` for multi-service
- Health checks in compose file

---

## Security Reminders

During deployment, check and warn:

- `.env` not in `.gitignore` → Warn
- Secrets in command output → Redact in runbook
- Default passwords detected → Warn to change
- Debug mode enabled → Warn for production

---

## Integration

**Use after:** `@process-task-list` when implementation complete

**Common follow-ups:**
- Set up CI/CD for automated deployment
- Configure monitoring and alerting
- Document operational procedures
