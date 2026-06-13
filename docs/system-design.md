# System Design Deep-Dive

> **Disclaimer**: This repository is a technical case study. The original implementation is proprietary and owned by the employer. No confidential source code, credentials, or sensitive business information is included.

---

## 1. Problem Statement

A growing astrology services business relied on a third-party WhatsApp BSP (Business Service Provider) that charged per-agent monthly subscriptions and provided no custom AI integration. Customer support response times averaged **4–12 hours**, and there was no way to inject the company's proprietary knowledge base into chat conversations.

**Design Goals:**
- Eliminate BSP subscription costs by integrating directly with Meta's WhatsApp Cloud API
- Reduce median response time to **< 30 seconds** using AI auto-responders
- Maintain brand voice consistency via a self-hosted RAG knowledge base
- Give human agents a shared real-time inbox with full AI assistance features
- Implement complete observability over AI costs, queue health, and delivery status

---

## 2. Architecture Philosophy

### Async-First Design

The most critical constraint in WhatsApp webhook processing is Meta's **2-second response window** — if the server doesn't respond within 2 seconds, Meta retries the delivery (causing duplicate processing).

**Solution**: The webhook controller performs only two synchronous operations:
1. Validate the HMAC signature
2. Push the raw payload to a Redis queue

Everything else — contact resolution, AI inference, database writes, outbound dispatch — runs in decoupled BullMQ background workers.

```
Webhook Controller → (< 2ms) → Redis Enqueue → HTTP 200 → Meta
                                         ↓
                                  BullMQ Worker (async, unlimited time)
```

### Event-Driven State Synchronization

All state mutations (new messages, status updates, conversation updates) are broadcast via Socket.io **from the worker**, not the HTTP controller. This keeps the real-time inbox perfectly synchronized across all connected agents without polling.

---

## 3. Database Design

### MongoDB — Primary Relational Store

MongoDB was selected over a relational SQL database for the following reasons:
- WhatsApp message payloads are semi-structured JSON — document storage maps naturally
- Flexible `metadata` (Mixed) field on Conversation allows storing arbitrary Zoho data without schema migrations
- Native ObjectId references provide sufficient relational integrity for this scale

#### Index Strategy

| Collection | Index | Purpose |
|-----------|-------|---------|
| `Contact` | `{ phoneNumber: 1 }` unique | Fast phone lookup on every inbound message |
| `Contact` | `{ email: 1 }` | Zoho Books invoice lookup by email |
| `Message` | `{ conversationId: 1 }` | Load chat history for a thread |
| `Message` | `{ metaWamid: 1 }` unique sparse | Idempotency — prevent duplicate message saves |
| `Message` | `{ status: 1 }` | Filter draft/pending messages |
| `Conversation` | `{ contactId: 1 }` | Resolve active thread for incoming contact |
| `Conversation` | `{ lastMessageAt: -1 }` | Sort inbox list by most recent |
| `CostTracking` | `{ createdAt: 1, provider: 1 }` compound | Time-series cost aggregation queries |
| `AuditLog` | `{ userId: 1, createdAt: -1 }` compound | Paginated audit trail per operator |

### Qdrant — Vector Database

The PRD initially specified Pinecone (cloud-managed). During implementation, the decision was made to **migrate to self-hosted Qdrant** for:
- **Zero vector query egress costs**
- **No external dependency** for the RAG pipeline
- **Full control** over collection configuration and similarity thresholds

**Collection configuration:**
```
Name: drishti-knowledge
Vector size: 1536 floats (OpenAI text-embedding-3-small output)
Distance metric: Cosine similarity
Similarity threshold: 0.3 (configurable via GlobalConfig)
```

**Chunk deduplication:** Each text chunk is sanitized (emoji surrogate removal), MD5-hashed, then converted to a UUID for use as the Qdrant point ID. This prevents duplicate chunk uploads on re-ingestion of the same document.

### Redis — Queue Backplane + Cache Layer

Redis serves two distinct roles:

**1. BullMQ Queue Backplane (primary use)**

| Queue | Worker | Concurrency | Retry Policy |
|-------|--------|-------------|--------------|
| `inbound-msg-queue` | InboundWorker | Configurable | Exponential backoff |
| `outbound-msg-queue` | OutboundWorker | Configurable | 3 attempts, 1s delay |
| `vector-ingest-queue` | VectorWorker | 1 (sequential) | Standard retry |
| `status-msg-queue` | StatusWorker | Configurable | Standard retry |

All queues are configured with `removeOnFail: { age: 86400, count: 1000 }` to prevent Redis memory accumulation from failed job retention.

**2. Zoho API Response Cache**

| Cache Key | TTL | Rationale |
|-----------|-----|-----------|
| `zoho:books:invoices:{email}` | 15 minutes | Invoice data changes infrequently; rate limit protection |
| Zoho CRM Contact + Deals | None (disabled) | Real-time sync required for deal stage accuracy in AI context |

---

## 4. API Design

### REST Design Principles

- **Resource-oriented URLs** — `/conversations/:id/messages`, `/conversations/:id/toggle-ai`
- **HTTP semantics** — `POST` for mutations, `GET` for reads, `DELETE` for removals
- **Unified response envelope** — `{ success: boolean, data?: any, message?: string }`
- **Correlation IDs** — every request generates a correlation ID for distributed tracing

### Authentication Architecture

```
Browser → POST /auth/login → JWT issued (HttpOnly, Secure, SameSite=Lax cookie)
                           → If twoFactorEnabled: returns requires2fa flag
       → POST /auth/login/2fa (TOTP token) → JWT issued
       → All subsequent requests: JWT cookie parsed by authenticateJwt middleware
```

**Security properties:**
- Passwords hashed with bcrypt (10 rounds)
- JWT contains only `{ id, role, email }` — no sensitive data in token payload
- HttpOnly cookies prevent XSS token theft
- TOTP 2FA via `otplib` + QR code generation for authenticator app enrollment
- Session cleared server-side on logout (cookie cleared)

### Webhook Security

Meta webhook signatures use HMAC-SHA256. The implementation:
1. Reads the raw request body buffer (not parsed JSON) before JSON middleware processes it
2. Computes `hmac-sha256(body, META_APP_SECRET)`
3. Compares against `x-hub-signature-256` header using **`crypto.timingSafeEqual`** to prevent timing attacks

### Role-Based Access Control

| Endpoint Category | `agent` | `admin` |
|------------------|---------|---------|
| View conversations, send messages | ✅ | ✅ |
| Toggle AI mode | ✅ | ✅ |
| View Zoho CRM sidebar | ✅ | ✅ |
| Request refund (Zoho CRM write) | ✅ | ✅ |
| Register new users | ❌ | ✅ |
| View analytics / cost dashboard | ❌ | ✅ |
| Configure AI providers / system prompt | ❌ | ✅ |
| Manage queue jobs (retry/delete) | ❌ | ✅ |
| Upload knowledge base documents | ❌ | ✅ |
| Sync and broadcast templates | ❌ | ✅ |

### External API Integration Design

**Meta WhatsApp Cloud API:**
- All outbound dispatches use message type-specific payload schemas (text, template, image, interactive)
- Template broadcasts compile `{{1}}` placeholder variables with provided parameters before saving to DB
- Outbound worker implements exponential backoff (3 attempts, 1s initial delay) for transient Meta API failures

**Zoho OAuth:**
- Access tokens stored in `ZohoToken` collection with `expiresAt` timestamp
- `zohoAuth.ts` implements automatic token refresh on expiry
- India DC endpoint (`zoho.in`) required for accounts in the Indian data center

---

## 5. Scalability Considerations

### Current Architecture Limits

| Component | Current Design | Scale-Out Path |
|-----------|---------------|----------------|
| Webhook ingestion | Single EC2 instance | Add load balancer + multiple API instances |
| BullMQ workers | Single process | Scale worker concurrency; run workers on separate instances |
| MongoDB | Atlas managed cluster | Vertical scale; sharding by `contactId` if needed |
| Qdrant | Single container | Qdrant distributed cluster mode |
| Redis | Single container | Redis Cluster or Redis Sentinel for HA |
| WebSocket | Single Socket.io server | Redis adapter for multi-instance Socket.io |

### Horizontal Scaling Path

The architecture is designed to be **horizontally scalable** with known, well-defined upgrade steps:

1. **Multiple API instances** → Add Nginx upstream balancing
2. **Multi-node WebSocket** → Enable Socket.io Redis adapter (`@socket.io/redis-adapter`)
3. **Worker autoscaling** → Increase `concurrency` in BullMQ worker config; run dedicated worker processes
4. **Multi-tenant** → Scope all collections with a `tenantId` field

### Performance Optimizations Implemented

- **Bun runtime** for the Express API — faster startup and improved throughput over Node.js
- **Next.js standalone output** — self-contained production build, no `node_modules` copying
- **Redis caching** for Zoho Books invoices (15min TTL) to prevent API rate limit exhaustion
- **Compound indexes** on `CostTracking` and `AuditLog` for O(log n) aggregation queries
- **Docker BuildKit** with cache mounts for fast CI/CD image builds

---

## 6. Observability Design

### Logging

All backend logs are structured JSON objects emitted to `stdout`:

```json
{
  "service": "drishti-api",
  "operation": "inboundWorker",
  "correlationId": "uuid-v4",
  "level": "info",
  "message": "Message processed successfully",
  "timestamp": "2026-06-01T12:00:00.000Z"
}
```

In production, `NODE_ENV=production` disables file logging — all logs stream to stdout for collection by AWS CloudWatch or Promtail.

### Health Check

```
GET /healthz
→ 200 OK      if MongoDB state == "connected" AND Redis state == "ready"
→ 503 Service Unavailable  otherwise
```

### Queue Observability

Admin users can access the `/queues` dashboard to:
- View active, waiting, completed, and failed job counts per queue
- Inspect individual failed job payloads and error messages
- Bulk retry or purge failed jobs
- Delete individual problematic jobs

### Cost Tracking

Every LLM call records to `CostTracking`:
- Provider and model name
- Input + output token counts
- Calculated cost in USD (based on configurable per-million-token pricing rates)
- Conversation reference for attribution

The `/dashboard` analytics page aggregates these records to show cost trends over time, broken down by provider.

---

## 7. Security Design

| Layer | Control | Implementation |
|-------|---------|----------------|
| **Webhook ingestion** | HMAC-SHA256 signature verification | `crypto.timingSafeEqual` constant-time comparison |
| **API authentication** | JWT in HttpOnly Secure cookie | `authenticateJwt` middleware on all protected routes |
| **2FA** | TOTP via Google Authenticator | `otplib` + QR code enrollment flow |
| **Outbound API** | API key authentication | `x-api-key` header check |
| **Password storage** | bcrypt hashing | 10 rounds |
| **Sensitive link access** | Ownership verification | Deal ID + email cross-reference in Zoho CRM |
| **AI guardrails** | Tool-level ownership checks | `shareReportLink` verifies phone/email before URL injection |
| **Audit trail** | All admin actions logged | `AuditLog` collection with `userId`, `action`, `clientIp` |
| **CORS** | Origin whitelist in production | Bypassed only for webhook and external trigger endpoints |
| **Config validation** | Zod schema at boot | `process.exit(1)` if required env vars missing |

### Privacy Guard — Sensitive Link Protection

One of the most important security features: the system **never exposes private report URLs or correction links** unless the customer provides both:
1. Their exact **18–19 digit Zoho Deal ID** (extracted via regex from chat message)
2. Their **email address** (matching the Zoho CRM contact record)

Both values are cross-verified against Zoho CRM before any link is injected into the AI's context window. If verification fails, the AI is instructed to politely withhold the link and ask for the required identifiers.
