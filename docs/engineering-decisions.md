# Engineering Decisions

> **Disclaimer**: This repository is a technical case study. The original implementation is proprietary and owned by the employer. No confidential source code, credentials, or sensitive business information is included.

---

This document captures the key technical decisions made during the design and implementation of Drishti Marketing OS, including the rationale, alternatives considered, and tradeoffs accepted.

---

## Decision 1: Bun Runtime Instead of Node.js

**Decision**: Run the Express API on the **Bun runtime** instead of vanilla Node.js.

**Context**: The backend is a TypeScript Express application. Node.js v20 was the default choice, but Bun 1.1 had reached production stability.

**Rationale**:
- Bun offers significantly faster startup times, which matters for container cold-start recovery
- Bun is fully Node.js-compatible for the CommonJS/ESM modules in use
- The Bun container image (`oven/bun:1.1-alpine`) is smaller than the equivalent Node Alpine image

**Tradeoff**: Bun compatibility with some npm packages (particularly `speakeasy` and `bcryptjs`) required verification. These packages were explicitly validated to run correctly under Bun's runtime, and swapping to native Node crypto alternatives was explicitly avoided to maintain stability.

**Implementation**: The Dockerfile uses a 3-stage build — TypeScript is compiled to JavaScript in a Node stage, then production artifacts are handed off to the `oven/bun:1.1-alpine` runner stage to execute `bun dist/server.js`.

---

## Decision 2: BullMQ + Redis for Asynchronous Message Processing

**Decision**: Introduce a Redis-backed BullMQ queue layer between webhook ingestion and message processing.

**Context**: Meta's WhatsApp Cloud API enforces a hard **2-second webhook response timeout**. If the server fails to respond in time, Meta retries delivery — creating duplicate processing risk.

**Rationale**:
- AI inference (3-5s for LLM completion), Zoho API calls (200-500ms), and database writes cannot complete within 2 seconds
- BullMQ provides durable job persistence (jobs survive server restarts), retry mechanisms, and visibility into job status
- Separating ingest from processing enables the worker to be independently scaled

**Alternatives Considered**:
- **In-memory async processing** — fails on server restarts; no durability, no retry
- **SQS** — adds AWS-specific dependency and additional cost; BullMQ achieves the same with already-available Redis

**4 Dedicated Queues**:

| Queue | Responsibility | Key Constraint |
|-------|---------------|----------------|
| `inbound-msg-queue` | Contact resolution + AI pipeline | Must be idempotent (`metaWamid` check) |
| `outbound-msg-queue` | Meta API dispatch | Exponential backoff on failures |
| `status-msg-queue` | Delivery status updates | High throughput, low compute |
| `vector-ingest-queue` | PDF/TXT embedding pipeline | Sequential (1 concurrency) to avoid Qdrant conflicts |

**Memory Management**: All queues configured with `removeOnFail: { age: 86400, count: 1000 }` to prevent unbounded Redis memory growth from failed job accumulation.

---

## Decision 3: Self-Hosted Qdrant Over Cloud-Managed Pinecone

**Decision**: Migrate the vector database from Pinecone (originally specified in PRD) to a **self-hosted Qdrant** instance.

**Context**: The original architecture document specified Pinecone for RAG vector storage.

**Rationale**:
- Pinecone charges per vector stored and per query — at scale this becomes a significant cost
- Qdrant is fully open-source with a permissive Apache 2.0 license
- Self-hosted Qdrant runs as a Docker container on the same EC2 instance, eliminating external API latency for vector queries
- Qdrant provides a REST client (`@qdrant/js-client-rest`) with a well-documented API

**Tradeoffs**:
- Operational responsibility for Qdrant uptime shifts to the team (managed via Docker Compose with persistent volume)
- No managed backup — vector data should be backed up separately (backlog item)

**Key Implementation Detail**: Chunk deduplication using MD5 hash → UUID conversion prevents re-uploading duplicate vectors when the same document is re-ingested. Text chunks are sanitized to remove emoji surrogate characters before hashing to prevent broken UTF-16 sequences from corrupting Qdrant upserts.

---

## Decision 4: Multi-Provider AI Switchboard Architecture

**Decision**: Implement an abstract `IAIProvider` interface with a unified `AISwitchboard` instead of hardcoding a single LLM provider.

**Context**: The business wanted flexibility to switch between Claude, GPT-4o, and Gemini based on cost, performance, and availability.

**Rationale**:
- A provider-agnostic interface allows adding new LLM providers without modifying business logic
- Per-conversation `AISession` overrides allow testing different providers on specific threads without affecting global defaults
- The `GlobalConfig.defaultProvider` setting allows administrators to change the default provider for new and/or existing conversations via the settings dashboard
- Multi-provider fallback chaining ensures continuity if one provider hits a rate limit or outage

**Provider Cost Comparison (as configured)**:

| Provider | Model | Input ($/1M tokens) | Output ($/1M tokens) |
|---------|-------|---------------------|----------------------|
| OpenAI | GPT-4o | $5.00 | $15.00 |
| Anthropic | Claude 3.5 Sonnet | $3.00 | $15.00 |
| Google | Gemini 1.5 Pro | $1.25 | $3.75 |

**Scoped Override System**: Administrators can choose to apply a new default provider to:
- `newonly` — future conversations only
- `oldonly` — existing active conversations only (bulk update of `AISession` records)
- `all` — both new and existing

---

## Decision 5: Native Tool-Calling Agent Loop

**Decision**: Implement a native **function-calling agent loop** in the AI switchboard instead of stuffing all context into a single system prompt.

**Context**: The AI needs to respond accurately to customer questions about orders, payments, report statuses, and refund requests — all of which require live data from Zoho CRM, Zoho Books, and the Knowledge Base.

**Rationale**:
- **Static prompt stuffing** would require injecting all possible CRM data upfront, increasing token consumption dramatically and potentially exposing irrelevant customer data
- **Tool-calling** allows the LLM to selectively fetch only the data it needs for the specific query (1–3 roundtrips)
- The LLM itself decides which tools to invoke — no brittle keyword-matching required

**Tools Implemented**:

| Category | Tools |
|----------|-------|
| **Lookup** | `lookupCustomerByPhone`, `lookupCustomerByEmail`, `getDealById` |
| **Verification** | `verifyPayment`, `getInvoices` |
| **Knowledge** | `searchKnowledgeBase` |
| **Actions** | `getCheckoutLink`, `getPaymentLink`, `shareReportLink`, `shareCorrectionLink` |
| **Escalation** | `requestRefund`, `escalateToHuman` |

**Safety Guardrails**: `shareReportLink` and `shareCorrectionLink` enforce ownership verification — the deal must be linked to the caller's verified phone or confirmed email before any private URL is returned. The LLM cannot bypass this server-side check.

---

## Decision 6: Dual AI Mode — Supervised Draft vs Autonomous

**Decision**: Implement two AI response modes controlled per conversation thread via `isAiDraftMode`.

**Context**: The business needed flexibility — some customer segments warrant full automation while others require human review before sending.

**Rationale**:
- **Autonomous mode** (`isAiDraftMode: false`) — AI replies immediately; best for common FAQ queries
- **Supervised Draft mode** (`isAiDraftMode: true`) — AI generates a suggestion shown to the agent as a "Suggested Reply" card; agent can approve (with optional edits), or reject

**Approve Flow**:
```
AI response saved (status: draft) → WebSocket emit → Agent sees card
  → Approve: PUT /messages/:id/approve → status updated to "pending" → queued to outbound
  → Reject: DELETE /messages/:id → draft deleted
```

This gives operations teams fine-grained control over AI quality during rollout without disabling the AI entirely.

---

## Decision 7: Turborepo Monorepo Structure

**Decision**: Structure the project as a **Turborepo monorepo** with shared TypeScript and ESLint configs.

**Context**: The project consists of two apps (`api` and `web`) that share no runtime code but do share configuration.

**Rationale**:
- Turborepo provides intelligent build caching — unchanged apps skip re-compilation
- Shared `packages/` workspace for `eslint-config` and `typescript-config` enforces consistent standards
- A single `yarn install` at root installs all workspace dependencies
- `turbo.json` pipeline ensures `build` tasks run in dependency order

**CI Pipeline Integration**: Turborepo's task pipeline is integrated into the GitHub Actions workflow. The `check-types` and `lint` gates run before Docker builds to catch type errors early.

---

## Decision 8: Cloudflare R2 for Media Storage

**Decision**: Use **Cloudflare R2** (S3-compatible) for operator-uploaded media files with a local EC2 disk fallback.

**Context**: Operators can attach images and documents (up to 16MB) to outbound WhatsApp messages. These files need to be accessible to the Meta API for sending.

**Rationale**:
- Cloudflare R2 has **zero egress fees** (unlike AWS S3), which matters for media-heavy workflows
- R2's S3-compatible API allows using the standard `@aws-sdk/client-s3` SDK with minimal configuration changes
- Pre-signed PUT URLs enable direct client-to-R2 uploads, bypassing the API server entirely for large files

**Implementation**: The media controller generates a pre-signed PUT URL via `@aws-sdk/s3-request-presigner`. The frontend uploads directly to R2. The API receives only the resulting R2 URL for database storage. Local EC2 disk storage (under `/temp/media/`) is maintained as a fallback and for inbound media downloads from Meta.

---

## Decision 9: Vanilla CSS with Design Token System

**Decision**: Use **Vanilla CSS with CSS custom properties** (no Tailwind, no CSS-in-JS) for the frontend.

**Context**: The team had an explicit constraint against Tailwind CSS due to bundle size concerns and the desire for tight, readable stylesheets.

**Rationale**:
- CSS custom properties provide a full design token system (`--primary`, `--surface`, `--background`, `--bubble-inbound`, etc.) without framework overhead
- Pure CSS gives full control over the grid layout (4-column inbox grid: sidebar + thread list + chat + Zoho panel)
- Glassmorphism effects and dark mode are simpler to maintain in vanilla CSS
- No build-time processing required — CSS is loaded directly

**Design System Highlights**:
- Dark obsidian theme (`--background: #0B0F17`)
- Emerald active indicator (`--primary: #10B981`)
- 4-column responsive grid with CSS media query collapsing for iPad/mobile
- Custom `Outfit` font from Google Fonts

---

## Decision 10: GitHub Actions → ECR → SSH Deployment Pipeline

**Decision**: Implement a GitHub Actions CI/CD pipeline that builds Docker images, pushes to Amazon ECR, and deploys via SSH.

**Context**: The team needed automated deployment with zero-downtime container replacement.

**Key Design Choices**:

1. **Active run polling** — the workflow checks for concurrent runs and waits, preventing race conditions from simultaneous deployments

2. **`docker compose pull api web` (not `pull` without args)** — explicitly pulling only the app images avoids re-pulling the Qdrant image from Docker Hub. Docker Hub rate limits (100 pulls/6h on free tier) would abort the deployment if the Qdrant pull was triggered repeatedly

3. **`docker compose up -d --remove-orphans`** — starts updated containers and removes any orphaned containers from previous deploys, keeping the compose state clean

4. **BuildKit cache mounts** — Docker BuildKit layer caching dramatically reduces rebuild times for unchanged dependency layers
