# Telegram AI Image Bot - Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│                          TELEGRAM AI IMAGE BOT                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


┌──────────────────────┐
│                      │
│       CLIENT         │
│                      │
│   Telegram Bot       │
│   (/start            │
│    /generate         │
│    /edit             │
│    /help)            │
│                      │
└──────────┬───────────┘
           │
           │ message / photo
           │ sendPhoto / status
           │
           ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│              MINI ORCHESTRATOR (FastAPI)                                 │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  Webhook & Command Router                                      │    │
│  │  - Telegram Webhook Handler                                    │    │
│  │  - Command Parser & Validator                                  │    │
│  │  - Async Request Handler                                       │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  State Management                                              │    │
│  │  - In-Memory Cache (MVP)                                       │    │
│  │  - Redis (Production)                                          │    │
│  │  - Session TTL (1 hour)                                        │    │
│  │  - Cleanup Tasks                                               │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  Core Functions                                                │    │
│  │  - generate_with_flux_lora(prompt) → image_url                 │    │
│  │  - edit_with_nano_banana(image_url, instruction) → image_url   │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  Error Handling & Resilience                                   │    │
│  │  - Retry Logic (3 attempts, exponential backoff)               │    │
│  │  - Timeout Handling (90s gen, 30s edit)                        │    │
│  │  - User Notifications                                          │    │
│  │  - Error Logging                                               │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  Rate Limiting                                                 │    │
│  │  - Per-User: 5 req/min                                         │    │
│  │  - Global: 100 concurrent                                      │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└──────────┬────────────────────────┬──────────────────────────────────────┘
           │                        │
           │ Generate:              │ Edit:
           │ text prompt            │ image URL + instruction
           │                        │
           ▼                        ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│                        MODEL SERVICES                                   │
│                                                                         │
│  ┌─────────────────────────────┐   ┌─────────────────────────────┐    │
│  │                             │   │                             │    │
│  │      Replicate              │   │        fal.ai               │    │
│  │   FLUX-LoRA (Generate)      │   │  nano-banana (Edit)         │    │
│  │                             │   │                             │    │
│  │  - Text-to-Image            │   │  - Image-to-Image           │    │
│  │  - Latency: 30-60s          │   │  - Latency: 5-10s           │    │
│  │  - Polling Required         │   │  - Fast Edits               │    │
│  │                             │   │  - Variable Quality         │    │
│  └─────────────────────────────┘   └─────────────────────────────┘    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
           │                        │
           │ image URL              │ edited image URL
           │                        │
           └────────────┬───────────┘
                        │
                        ▼
                 (back to user)


═══════════════════════════════════════════════════════════════════════════
                       EXTEND LATER (Optional)
═══════════════════════════════════════════════════════════════════════════

┌─────────────────────────┐     ┌─────────────────────────┐
│  Storage / CDN          │     │  QA & Scoring           │
│  (S3 / Cloudinary)      │     │  (ArcFace, Aesthetic,   │
│                         │     │   Safety)               │
│  - upload/download      │     │                         │
│  - Image Persistence    │     │  - check/add credits    │
│  - CDN Delivery         │     │  - scores               │
└─────────────────────────┘     └─────────────────────────┘

┌─────────────────────────┐     ┌─────────────────────────┐
│  Credits & Payments     │     │  ComfyUI Cloud          │
│  (Supabase + LSQ/Stripe)│     │  (masked edits,         │
│                         │     │   DSLRizer, FLUX gen)   │
│  - Credit Tracking      │     │                         │
│  - Payment Processing   │     │  - Advanced Workflows   │
│  - Usage Analytics      │     │  - Complex Edits        │
└─────────────────────────┘     └─────────────────────────┘


═══════════════════════════════════════════════════════════════════════════
                              DATA FLOWS
═══════════════════════════════════════════════════════════════════════════

GENERATE FLOW:
──────────────
1. User → /generate sunset over mountains
2. Webhook → Command Router → generate_with_flux_lora()
3. Orchestrator → "Generating..." status → User
4. Orchestrator → Poll Replicate API
5. Replicate → Image URL → Orchestrator
6. Orchestrator → Image → Telegram → User

EDIT FLOW:
──────────
1. User → Reply to image with /edit make it cyberpunk
2. Webhook → Extract image URL → edit_with_nano_banana()
3. Orchestrator → "Editing..." status → User
4. Orchestrator → fal.ai API call
5. fal.ai → Edited Image URL → Orchestrator
6. Orchestrator → Image → Telegram → User

ERROR FLOW:
───────────
1. API Failure → Retry (3x with backoff)
2. Still Failing → Log Error
3. Send User-Friendly Message → User
4. Store State for Debugging


═══════════════════════════════════════════════════════════════════════════
                          DEPLOYMENT ARCHITECTURE
═══════════════════════════════════════════════════════════════════════════

                      ┌─────────────────┐
                      │   Telegram      │
                      │   Bot API       │
                      └────────┬────────┘
                               │
                               │ HTTPS Webhook
                               │
                               ▼
                      ┌─────────────────┐
                      │   FastAPI App   │
                      │   (Uvicorn)     │
                      │                 │
                      │   Port: 8000    │
                      └────────┬────────┘
                               │
                    ┌──────────┴──────────┐
                    │                     │
                    ▼                     ▼
           ┌─────────────────┐   ┌─────────────────┐
           │  In-Memory      │   │  External APIs  │
           │  Cache (MVP)    │   │  - Replicate    │
           │       OR        │   │  - fal.ai       │
           │  Redis (Prod)   │   │                 │
           └─────────────────┘   └─────────────────┘
```

## Key Architectural Decisions

### 1. **Webhook vs Polling**
- **Choice**: Webhook
- **Rationale**: Lower latency, more efficient, scalable
- **Implementation**: FastAPI POST endpoint `/webhook`

### 2. **State Management**
- **MVP**: In-memory cache (simple, fast)
- **Production**: Redis (distributed, persistent)
- **Rationale**: Start simple, migrate when scaling

### 3. **Async Processing**
- **Choice**: FastAPI async handlers
- **Rationale**: Non-blocking, handles concurrent requests efficiently
- **Implementation**: `async def` handlers with `httpx` for API calls

### 4. **Error Handling Strategy**
- **Retry Logic**: 3 attempts with exponential backoff
- **Timeouts**: 90s (generation), 30s (editing)
- **User Feedback**: Progress updates + clear error messages
- **Logging**: Structured logs for debugging

### 5. **Rate Limiting**
- **Per-User**: 5 requests/minute
- **Global**: 100 concurrent requests
- **Rationale**: Prevent abuse + cost control

### 6. **Image URL Handling**
- **Choice**: Pass URLs (not download/re-upload)
- **Rationale**: Faster, less bandwidth
- **Risk**: Telegram CDN URLs may expire
- **Future Mitigation**: Download and store in S3

### 7. **Extension Points**
- All future features designed as pluggable modules
- Clear interfaces for storage, QA, credits, ComfyUI
- Enables incremental development without refactoring

## Technology Stack

### Core
- **Python 3.11+**: Modern async support
- **FastAPI**: High-performance web framework
- **Uvicorn**: ASGI server
- **python-telegram-bot**: Telegram API wrapper
- **httpx**: Async HTTP client

### State Management
- **cachetools**: In-memory caching (MVP)
- **redis-py**: Redis client (production)

### API Clients
- **replicate**: Replicate API client
- **fal-client**: fal.ai API client

### Utilities
- **pydantic**: Data validation
- **tenacity**: Retry logic
- **structlog**: Structured logging

### Deployment
- **Docker**: Containerization
- **Railway/Render/Fly.io**: Hosting (MVP)
- **AWS/GCP**: Production deployment

## Security Considerations

1. **Webhook Verification**: Validate Telegram secret token
2. **API Key Protection**: Environment variables, never in code
3. **Input Validation**:
   - Prompt length limits (max 500 chars)
   - Command validation
   - Image URL validation
4. **Rate Limiting**: Prevent abuse and cost overrun
5. **Content Filtering**: (Future) NSFW detection, safety scoring

## Scalability Path

### Phase 1 (MVP): Single Instance
- In-memory state
- Direct API calls
- Simple deployment

### Phase 2: Horizontal Scaling
- Redis for state
- Multiple FastAPI instances
- Load balancer

### Phase 3: Queue-Based
- Redis/Celery job queue
- Worker pool for API calls
- Priority handling

### Phase 4: Microservices
- Separate services for generation/editing
- Independent scaling
- Advanced monitoring