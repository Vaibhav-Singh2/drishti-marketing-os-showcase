# My Engineering Contributions

> **Disclaimer**: This repository is a technical case study. The original implementation is proprietary and owned by the employer. No confidential source code, credentials, or sensitive business information is included.

---

## Overview

I was the **sole engineer** responsible for designing and implementing Drishti Marketing OS end-to-end — from architecture decisions to production deployment. This document details what I personally built, owned, and shipped.

---

## 1. System Architecture Design

**What I did:**

Designed the entire system architecture from scratch, replacing the business's dependency on third-party WhatsApp BSP subscriptions (WATI/AISensy) with a custom-built platform. This involved:

- Evaluating and selecting the Turborepo monorepo structure to manage two interconnected applications with shared configurations
- Designing the async-first message pipeline using BullMQ + Redis to meet Meta's 2-second webhook constraint
- Choosing MongoDB as the primary database and designing all collection schemas, indexes, and relationships
- Making the vendor decision to migrate the RAG vector store from Pinecone to self-hosted Qdrant (cost reduction + zero external latency)
- Designing the role-based access control model (admin vs agent) and the JWT + 2FA security stack

**Impact**: The architecture supports unlimited concurrent AI conversations with a single deployment, replacing a per-agent billing model that capped support capacity.

---

## 2. AI Pipeline & Multi-Provider Switchboard

**What I built:**

The complete AI inference engine that powers all automated responses:

- **`aiSwitchboard.ts`**: Abstract provider router implementing `IAIProvider` interface for Claude 3.5, GPT-4o, and Gemini 1.5 Pro. Resolves provider from per-conversation `AISession` overrides falling back to `GlobalConfig` defaults. Implements multi-provider fallback chaining.

- **`agentTools.ts`**: The native function-calling tool registry. Defined 10 tools including `lookupCustomerByPhone`, `lookupCustomerByEmail`, `getDealById`, `verifyPayment`, `searchKnowledgeBase`, `shareReportLink`, `requestRefund`, and `escalateToHuman`. Implemented the ownership verification logic inside `shareReportLink` and `shareCorrectionLink` — the server-side privacy guard that prevents unauthorized URL exposure.

- **5-Part Context Assembly**: Designed and implemented the prompt assembly pipeline that combines brand guidelines + RAG chunks + conversation history + Zoho CRM context + user query into a structured multi-part system prompt.

- **Cost Tracking**: Implemented per-call token usage logging to `CostTracking` with configurable per-million-token pricing rates for each provider, enabling real-time cost analytics.

---

## 3. RAG Knowledge Base Pipeline

**What I built:**

End-to-end Retrieval-Augmented Generation system:

- **`ragPipeline.ts`**: Vector similarity search using Qdrant REST client. Generates 1536-dimension query embeddings with OpenAI `text-embedding-3-small`. Applies configurable cosine similarity threshold (default 0.3, stored in `GlobalConfig`). Falls back to human takeover escalation if no relevant chunks found.

- **`vectorWorker.ts`**: Background ingestion worker that:
  - Validates uploaded files (rejects 0-byte files immediately)
  - Extracts text from PDFs via `pdf-parse` and plain-text via UTF-8 conversion
  - Sanitizes text to remove emoji surrogate characters (prevents Qdrant upsert failures)
  - Generates embeddings and upserts to Qdrant with MD5-hash-derived UUIDs for deduplication
  - Implements differentiated file cleanup — preserves temp file across retries, deletes only on success or final failure

- **`qdrantClient.ts`**: Qdrant client wrapper with collection auto-initialization on boot, point upsert, similarity search, and UUID conversion utility (`toUUID()`).

---

## 4. WhatsApp Webhook Processing Engine

**What I built:**

The complete inbound/outbound message processing pipeline:

- **`webhookController.ts`**: Webhook entry point with HMAC-SHA256 signature verification using `crypto.timingSafeEqual` (timing-attack resistant). Designed the raw body capture approach — preserving `req.rawBody` buffer before JSON parsing middleware, since HMAC verification requires the original bytes.

- **`inboundWorker.ts`**: The most complex component in the system. Handles:
  - `metaWamid` idempotency checking to prevent duplicate processing
  - Contact creation via Zoho CRM lookup (for new phone numbers)
  - Conversation find-or-create with AI mode preservation on thread reopen
  - Full 7-layer pipeline coordination (RAG → Switchboard → Draft/Autonomous dispatch)
  - Media download from Meta CDN using streaming responses

- **`outboundWorker.ts`**: Meta Graph API dispatch worker with:
  - Template parameter compilation (`{{1}}` placeholder replacement)
  - Local EC2 media file priority with R2 bucket fallback
  - Draft message status promotion on approval (`draft` → `pending` → `sent`)
  - MetaWamid registration for delivery status tracking

- **Status tracking**: Designed the `status-msg-queue` pipeline that maps Meta's async `delivered`/`read` status webhooks back to `metaWamid` indexes in MongoDB, triggering real-time WebSocket updates to update inbox checkmark indicators.

---

## 5. Real-Time WebSocket Synchronization

**What I built:**

The bidirectional real-time communication layer:

- **`socketService.ts`**: Socket.io server integration with room-based event targeting. Implements `emitToRoom(conversationId, event, data)` for conversation-specific broadcasts and `emitGlobal(event, data)` for inbox-wide updates. Supports `join_room` for agents subscribing to specific conversation namespaces.

- **`store.ts` (Zustand)**: Frontend state machine with mutation handlers for all WebSocket events:
  - `handleIncomingMessage` — appends new messages, renders draft cards, increments unread badges
  - `handleStatusUpdate` — mutates message status in place (`sent → delivered → read`)
  - `handleConversationUpdate` — re-sorts thread list by `lastMessageAt` descending
  - `handleChatClear` / `handleConversationDelete` — instant local state cleanup

- **AI thinking indicator**: Implemented real-time "AI thinking" visual state — a WebSocket event emitted when the switchboard starts processing, consumed by the Zustand store to display a pulsing indicator in the chat UI.

---

## 6. Zoho CRM/Books Integration Layer

**What I built:**

Complete read-write integration with Zoho India DC APIs:

- **OAuth token management** (`zohoAuth.ts`): Auto-refreshing token lifecycle with `ZohoToken` MongoDB store. Token refresh is mutex-guarded to prevent race conditions when multiple concurrent requests trigger refresh simultaneously.

- **CRM integration** (`zohoCRM.ts`): Contact search by phone number, deal lookup by contact ID with active filter, secure deal report URL extraction, and the whitelisted write operation for `Request Refund` (updates `Report_Status` to `"Asking Refund"` in Zoho CRM).

- **Books integration** (`zohoBooks.ts`): Customer invoice lookup by email with Redis 15-minute caching.

- **Sidebar UI** (`InboxLayout.tsx`): Built the Zoho CRM/Books sidebar panel showing deal cards with stage, amount, closing date, Report Status, and action buttons (Request Refund). Implemented the "Request Refund" button that calls the whitelisted backend endpoint with optimistic UI update.

- **Two-factor Deal ID verification**: Implemented the regex extraction (`/\b\d{18,19}\b/g` for Deal IDs, email pattern) and Zoho cross-reference logic in the AI switchboard for secure link sharing.

---

## 7. Authentication & Security System

**What I built:**

Full authentication stack from scratch:

- **`authController.ts`**: JWT-based login with bcrypt password verification, TOTP 2FA setup (secret generation + QR code export) and verification, HttpOnly secure cookie session management, and `/me` endpoint for browser session rehydration.

- **Middleware chain** (`middleware/`): `authenticateJwt` for cookie/Bearer token parsing with role extraction, `authenticateApiKey` for external trigger API protection, `verifyWebhookSignature` for HMAC-SHA256 Meta webhook validation with constant-time comparison.

- **`validateEnv.ts`**: Zod schema validation for all environment variables at server boot. Missing required variables cause `process.exit(1)` to prevent silent misconfiguration in production.

---

## 8. Frontend Web Inbox

**What I built:**

Complete Next.js 16 web application with real-time inbox:

- **`InboxLayout.tsx`**: The central workspace component — 4-column CSS grid layout, mount-time auth check with redirect, conversation list with open/closed status filtering, real-time WebSocket integration, AI draft card UI (Approve/Edit/Reject), media upload integration, and the 24-hour Meta reply window countdown timer.

- **`ChatComposer.tsx`**: Message composition component with the AI Copilot dropdown menu — brand-compliant tone refinement, message expansion, and bullet-point compression modes, all using the 5-part context system via `/assist` endpoint.

- **CSS Design System** (`globals.css`): Designed the full dark-mode design token system from scratch — deep obsidian backgrounds, emerald primary indicators, glassmorphism panels, custom Outfit font, and the 4-column responsive grid with CSS media query collapsing for iPad viewports.

- **All pages**: Designed and built `/login`, `/inbox`, `/dashboard`, `/settings`, `/ai-settings`, `/queues`, and `/templates` pages with their respective data fetching and UI logic.

---

## 9. DevOps & CI/CD Pipeline

**What I built:**

Complete containerization and automated deployment infrastructure:

- **Multi-stage Dockerfiles**: 3-stage backend Dockerfile (Builder → Dependencies → Bun Runner) and 2-stage frontend Dockerfile leveraging Next.js standalone output.

- **`docker-compose.prod.yml`**: Production container orchestration with bridge networking, persistent volumes for Qdrant and Redis, health check conditions, and service dependency ordering.

- **GitHub Actions workflow**: Full CI/CD pipeline with lint + type-check gates, Docker BuildKit build with ECR push, active run concurrency detection, and SSH deploy step with selective image pull strategy (avoiding Docker Hub rate limits by pulling only ECR-hosted images).

---

## 10. Observability & Admin Tooling

**What I built:**

- **Cost analytics dashboard**: Aggregation pipeline on `CostTracking` collection, broken down by provider and model, rendered as Recharts SVG graphs.

- **Queue management dashboard** (`/queues`): Real-time BullMQ metrics display with failed job inspection, bulk retry, bulk clean, and individual job retry/delete controls.

- **Template broadcast system** (`/templates`): Meta Graph API template sync, interactive template grid, CSV file parsing for bulk recipient lists, and campaign broadcast queue management.

- **Structured logging**: Winston configuration with correlation IDs (`service`, `operation`, `correlationId`) on every log entry, stdout JSON output in production for CloudWatch ingestion.

- **`/healthz` endpoint**: Composite health check validating both MongoDB connection state and Redis readiness, returning `503` on degraded state.

---

## Measurable Engineering Outcomes

| Area | Metric | Achievement |
|------|--------|-------------|
| **Response latency** | Median customer response time | 4–12 hours → < 30 seconds |
| **Scale** | Concurrent AI conversations | From ~50/day (human-limited) → Unlimited |
| **Reliability** | Webhook duplicate processing | Zero duplicate messages (idempotency via `metaWamid`) |
| **Security** | Private link exposure | Zero unauthorized URL exposure (ownership verification system) |
| **Costs** | BSP subscription | Eliminated (direct Meta Cloud API integration) |
| **Observability** | AI cost visibility | Per-conversation, per-model cost tracking in real time |
| **Deployment** | Docker Hub rate limits | Zero deployment failures (selective ECR-only image pulls) |
| **Memory** | Redis queue accumulation | Bounded (24h retention + 1000-job cap enforced) |
