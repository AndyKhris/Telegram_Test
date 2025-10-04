# Product Requirements Document: Telegram AI Image Bot

## 1. Overview

### 1.1 Product Vision
A Telegram bot that enables users to generate and edit images using state-of-the-art AI models through simple chat commands. The system provides a minimal, fast, and reliable MVP with a clear path to scale.

### 1.2 Goals
- Deliver image generation via `/generate` command using Replicate FLUX-LoRA
- Deliver image editing via `/edit` command using fal.ai nano-banana
- Provide fast, reliable responses directly in Telegram chat
- Build extensible architecture for future enhancements (storage, QA, credits, ComfyUI)

### 1.3 Non-Goals (MVP)
- User authentication/authorization beyond Telegram
- Persistent storage (S3/CDN)
- Credit/payment system
- Quality assurance scoring
- ComfyUI integration
- Advanced prompt engineering UI

## 2. User Stories

### 2.1 Core Functionality
- As a user, I can send `/generate <prompt>` to create a new image from text
- As a user, I can send `/edit <instruction>` (replying to an image) to modify an existing image
- As a user, I receive progress updates while my image is being processed
- As a user, I receive the generated/edited image directly in the chat
- As a user, I receive clear error messages if something goes wrong

### 2.2 Future Enhancements
- As a user, I can view my remaining credits
- As a user, I can see quality scores for my generated images
- As a user, I can access my image history
- As a user, I can use advanced editing with mask support

## 3. Technical Architecture

### 3.1 System Components

#### 3.1.1 Client Layer
- **Telegram Bot**: User interface for commands and image delivery
- **Commands**: `/start`, `/generate`, `/edit`, `/help`

#### 3.1.2 Mini Orchestrator (FastAPI)
- **Webhook Receiver**: Handles incoming Telegram updates
- **Command Router**: Routes commands to appropriate handlers
- **State Management**: In-memory cache (MVP) / Redis (production)
- **Core Functions**:
  - `generate_with_flux_lora(prompt: str) -> image_url`
  - `edit_with_nano_banana(image_url: str, instruction: str) -> image_url`

#### 3.1.3 Model Services
- **Replicate FLUX-LoRA**: Text-to-image generation
  - Expected latency: 30-60 seconds
  - Requires polling for completion
- **fal.ai nano-banana**: Fast image editing
  - Expected latency: 5-10 seconds
  - Good for quick edits, variable instruction adherence

#### 3.1.4 Future Extensions (Dashed in Architecture)
- **Storage/CDN**: S3/Cloudinary for image persistence
- **QA & Scoring**: ArcFace, Aesthetic, Safety scoring
- **Credits & Payments**: Supabase + LSQ/Stripe integration
- **ComfyUI Cloud**: Advanced workflows (masked edits, DSLRizer, FLUX gen)

### 3.2 Data Flow

#### 3.2.1 Generate Flow
1. User sends `/generate sunset over mountains`
2. Webhook receives message → Command router
3. Orchestrator calls `generate_with_flux_lora()`
4. Sends "Generating..." status to user
5. Polls Replicate API for completion
6. Returns image URL to Telegram
7. Bot sends photo to user

#### 3.2.2 Edit Flow
1. User replies to image with `/edit make it cyberpunk style`
2. Webhook receives message + photo reference
3. Orchestrator extracts image URL from Telegram
4. Calls `edit_with_nano_banana(image_url, instruction)`
5. Sends "Editing..." status to user
6. Returns edited image URL
7. Bot sends photo to user

### 3.3 Error Handling & Resilience

#### 3.3.1 API Failures
- Retry logic: 3 attempts with exponential backoff
- Timeout handling: 90s for generation, 30s for editing
- User notification: Clear error messages for failures
- Fallback: Log errors, notify user to try again

#### 3.3.2 State Management
- Session TTL: 1 hour (prevents memory leaks)
- Cleanup: Background task removes expired sessions
- Redis (production): Enables multi-instance deployment

#### 3.3.3 Rate Limiting
- Per-user: 5 requests per minute (prevents abuse)
- Global: 100 concurrent requests (prevents cost overrun)
- Queue system: Redis-based job queue for scaling (future)

## 4. API Specifications

### 4.1 Telegram Commands

#### `/start`
- **Description**: Initialize bot, show welcome message
- **Response**: Instructions and available commands

#### `/generate <prompt>`
- **Description**: Generate image from text prompt
- **Parameters**:
  - `prompt` (required): Text description of desired image
- **Response**: Generated image sent to chat
- **Error Cases**: Invalid prompt, API failure, timeout

#### `/edit <instruction>`
- **Description**: Edit image (must reply to existing image)
- **Parameters**:
  - `instruction` (required): Description of desired edit
  - `image_url` (extracted from reply): Source image
- **Response**: Edited image sent to chat
- **Error Cases**: No image in reply, invalid instruction, API failure

#### `/help`
- **Description**: Show help information
- **Response**: Usage instructions and examples

### 4.2 Internal API Endpoints

#### `POST /webhook`
- **Description**: Receives Telegram updates
- **Request**: Telegram Update object
- **Response**: 200 OK (processed asynchronously)

#### `generate_with_flux_lora(prompt: str) -> str`
- **Description**: Generate image using Replicate FLUX-LoRA
- **Returns**: Image URL
- **Raises**: APIError, TimeoutError

#### `edit_with_nano_banana(image_url: str, instruction: str) -> str`
- **Description**: Edit image using fal.ai nano-banana
- **Returns**: Edited image URL
- **Raises**: APIError, TimeoutError

## 5. Configuration & Environment

### 5.1 Required Environment Variables
```
TELEGRAM_BOT_TOKEN=<bot_token>
TELEGRAM_WEBHOOK_URL=<public_url>/webhook
REPLICATE_API_KEY=<api_key>
FAL_AI_API_KEY=<api_key>
REDIS_URL=<redis_url>  # Optional for MVP
STATE_BACKEND=memory    # or 'redis'
RATE_LIMIT_PER_USER=5
RATE_LIMIT_WINDOW=60
```

### 5.2 Deployment Requirements
- Python 3.11+
- FastAPI + Uvicorn
- Public HTTPS endpoint (for Telegram webhook)
- Redis (optional for MVP, required for production)

## 6. Success Metrics (MVP)

### 6.1 Functional Metrics
- Command success rate > 95%
- Average generation time < 60s
- Average edit time < 15s
- Error rate < 5%

### 6.2 User Experience
- Clear progress indicators during processing
- Informative error messages
- Response within 1 minute for generation
- Response within 20 seconds for editing

## 7. Future Roadmap

### Phase 2: Persistence & Quality
- Image storage (S3/Cloudinary)
- Quality scoring (aesthetic, safety)
- User history and gallery

### Phase 3: Monetization
- Credit system
- Payment integration (Stripe/LSQ)
- Usage analytics

### Phase 4: Advanced Features
- ComfyUI integration for complex workflows
- Masked editing and inpainting
- Multiple model support
- Batch processing

### Phase 5: Scale
- Multi-region deployment
- CDN integration
- Advanced caching strategies
- Job queue with priority handling

## 8. Security & Privacy

### 8.1 MVP Considerations
- Webhook verification (Telegram secret token)
- API key protection (environment variables)
- Input validation (prompt length limits, content filtering)
- Rate limiting (prevent abuse)

### 8.2 Future Enhancements
- Content moderation (safety scoring)
- NSFW detection
- User blocking/reporting
- GDPR compliance (data deletion)

## 9. Risks & Mitigations

### 9.1 Technical Risks
| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Replicate API slow/down | High | Medium | Retry logic, user notification, consider backup provider |
| Telegram CDN URL expiration | Medium | High | Download and re-upload images, or implement storage early |
| Memory leaks from state | Medium | Medium | TTL-based cleanup, migrate to Redis |
| Cost overrun | High | Medium | Rate limiting, usage monitoring, budget alerts |

### 9.2 Product Risks
| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Poor image quality | High | Medium | Test prompts, add quality scoring, allow regeneration |
| Slow user adoption | Medium | Low | Clear onboarding, examples, tutorials |
| Abuse/spam | Medium | Medium | Rate limiting, user blocking, content moderation |

## 10. Success Criteria

### 10.1 MVP Launch Criteria
- [ ] `/generate` command functional with FLUX-LoRA
- [ ] `/edit` command functional with nano-banana
- [ ] Progress indicators during processing
- [ ] Error handling and user notifications
- [ ] Rate limiting implemented
- [ ] Deployed on public HTTPS endpoint
- [ ] Basic monitoring/logging

### 10.2 Definition of Done
- All commands respond correctly to valid inputs
- Error messages are clear and actionable
- System handles 10 concurrent users without degradation
- Documentation complete (setup, deployment, usage)
- Code reviewed and tested