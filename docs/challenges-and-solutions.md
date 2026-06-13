# Challenges & Solutions

> **Disclaimer**: This repository is a technical case study. The original implementation is proprietary and owned by the employer. No confidential source code, credentials, or sensitive business information is included.

---

This document details the real engineering challenges encountered during the design and development of Drishti Marketing OS, and the solutions implemented to address them.

---

## Challenge 1: Meeting Meta's 2-Second Webhook Acknowledgement Requirement

### The Problem

Meta's WhatsApp Cloud API sends webhook events for every inbound message. If the server doesn't respond with HTTP `200` within **2 seconds**, Meta treats the delivery as failed and retries — which could cause the same message to be processed multiple times, potentially sending the customer duplicate AI responses.

The actual message processing pipeline — HMAC verification, contact resolution, Zoho CRM lookup, vector embedding generation, LLM inference, database writes — takes **3–8 seconds** under normal conditions.

### The Solution

**Immediate enqueueing with BullMQ + Redis:**

1. Webhook controller validates the HMAC signature (constant-time, < 5ms)
2. Pushes the raw payload to `inbound-msg-queue` (Redis write, < 10ms)
3. Returns `HTTP 200 EVENT_RECEIVED` immediately
4. Background `InboundWorker` processes the job asynchronously with no time constraint

**Idempotency guard against duplicate delivery:**

Even with the < 2s response, Meta occasionally retries on transient network issues. The `metaWamid` field on the `Message` collection has a unique sparse index. Before processing, the worker checks if a message with that `wamid` already exists — if so, it skips processing silently. This prevents duplicate messages from being saved or duplicate AI replies from being sent.

```
Meta sends webhook → Controller enqueues (< 15ms) → 200 OK
                                    ↓
                            InboundWorker (async)
                                    ↓
                            Check metaWamid uniqueness
                                    ↓ if duplicate → skip
                            Process pipeline
```

**Result**: Zero Meta webhook retry failures caused by slow processing. Zero duplicate message sends observed in production.

---

## Challenge 2: Preventing Sensitive Link Exposure via AI Hallucination

### The Problem

The AI needs to share customer-specific report URLs and correction links — but only with the correct customer. A naive implementation would include all URLs for the primary phone number's associated deals in the system prompt, risking:

- Sharing a link with a customer calling from a different phone
- Sharing another customer's link if they claim to be someone else
- The LLM exposing links it shouldn't based on vague instructions

### The Solution

**Two-factor ownership verification protocol in the AI Switchboard:**

1. **Default behavior**: Private URLs (`Report_URL`, `Correction Link`) are **never included** in the AI's context — not even for the primary phone number's own deals
2. **Verification trigger**: The switchboard uses regex to extract email patterns and Zoho Deal IDs (18–19 digit numbers) from the customer's message
3. **Cross-reference check**: If both a Deal ID and email are extracted, the switchboard queries Zoho CRM to verify that:
   - The email belongs to a registered contact
   - One of that contact's deals matches the provided Deal ID
4. **Conditional injection**: Only if verification passes are the links for that specific deal injected into the context
5. **Withhold on failure**: If verification fails, the AI is instructed to politely prompt the customer to provide both their email and Report Deal ID

This is a **server-side** enforcement — the LLM cannot bypass it because the links are never present in the prompt unless server-side verification passes.

**Internal Status Masking**: Internal Zoho deal statuses (e.g., `In Progress`, `Prepared`) are never exposed to customers. The AI maps all non-final states to a customer-safe phase (`"in_preparation"`), preventing customers from knowing the internal workflow stage.

---

## Challenge 3: Emoji Surrogate Characters Breaking Qdrant Vector Upserts

### The Problem

When processing uploaded PDF or TXT files for the RAG knowledge base, the text extraction sometimes produced surrogate character sequences — malformed Unicode from emoji or special characters. These broke Qdrant's REST API upsert calls, causing the vector worker to fail after significant processing time.

### The Solution

A **pre-embedding sanitization step** in the `VectorWorker`:

1. After extracting text from PDF/TXT files, the text is passed through a sanitization function that removes emoji surrogate split characters using Unicode-aware regex
2. The sanitized text is used both for:
   - Generating embeddings (OpenAI API)
   - Computing the MD5 hash used as the Qdrant point UUID
3. The raw (unsanitized) text is still stored in the Qdrant payload for retrieval, but the embedding and ID generation use the clean version

This ensures Qdrant upserts never fail due to encoding issues, and chunk deduplication (MD5-based UUID) remains stable across re-ingestion of the same document.

---

## Challenge 4: Redis Memory Accumulation from Failed Job Retention

### The Problem

In early development, failed BullMQ jobs accumulated indefinitely in Redis. In a production environment with retry failures (e.g., transient Meta API outages), thousands of failed job records could accumulate, eventually causing Redis to run out of memory.

### The Solution

All four queues were configured with explicit retention limits:

```typescript
removeOnFail: {
  age: 24 * 3600,  // Remove failed jobs older than 24 hours
  count: 1000      // Keep maximum 1000 failed jobs per queue
}
```

This ensures:
- The Redis memory footprint remains bounded regardless of failure volume
- Recent failures are still visible for debugging in the queue dashboard
- The queue management dashboard (`/queues`) shows only actionable job records

---

## Challenge 5: File Cleanup Race Condition in the Vector Ingestion Pipeline

### The Problem

The `VectorWorker` processes uploaded PDF/TXT files from a temporary disk location. The naive approach — "delete the file when processing finishes" — creates a problem with retries:

- If processing fails midway, BullMQ retries the job
- If the file was deleted on the first failure, the retry finds no file and fails immediately
- This wastes the retry attempts that could have succeeded on a subsequent try

### The Solution

**Differentiated cleanup strategy based on job outcome:**

- **Success**: Delete the temporary file immediately after successful Qdrant upsert
- **Failure (non-final attempt)**: Do NOT delete the file — leave it for the next retry
- **Final failure (all retries exhausted)**: Delete the file to clean up disk space

```
Job attempt 1 → fails → file preserved → retry
Job attempt 2 → fails → file preserved → retry  
Job attempt 3 (final) → fails → file DELETED
Job attempt N → succeeds → file DELETED immediately
```

This ensures retry attempts always have the file available, while preventing orphaned files from accumulating on disk after all retries are exhausted.

---

## Challenge 6: Zoho India Data Center OAuth Header Mismatch

### The Problem

The original architecture document specified Zoho API calls using `Authorization: Bearer <accessToken>`. In testing, all Zoho API calls returned `401 Unauthorized`.

### The Root Cause

Zoho's India Data Center (`.in` domain) uses a different authorization header format from the global Zoho API documentation. The correct format for all India DC API calls is:

```
Authorization: Zoho-oauthtoken <accessToken>
```

This is an undocumented distinction in Zoho's regional API guides.

### The Solution

Updated the authorization header in both `zohoCRM.ts` and `zohoBooks.ts` to use the `Zoho-oauthtoken` format. Added this as a documented known issue to prevent future developers from being confused by the discrepancy between the architecture spec and the implementation.

---

## Challenge 7: Docker Hub Rate Limiting Breaking CI/CD Deployments

### The Problem

The production `docker-compose.prod.yml` includes three services: `api`, `web`, and `qdrant`. When the CI/CD pipeline ran `docker compose pull` (pulling all services), it triggered a pull of the `qdrant/qdrant` image from Docker Hub. Docker Hub's free tier limits unauthenticated pulls to 100 per 6 hours per IP — the EC2 instance's IP was being rate-limited, causing deployment failures.

### The Solution

Modified the SSH deployment step to pull **only the application images** (which come from Amazon ECR, not Docker Hub):

```bash
# Before (broken):
docker compose pull

# After (fixed):
docker compose pull api web
# qdrant image is already present on the host — no re-pull needed
```

This eliminates Docker Hub rate limit exposure entirely. The Qdrant image is pulled once during initial setup and never re-pulled during routine deployments since it's pinned to a specific version.

---

## Challenge 8: WebSocket State Desync Across Multiple Operators

### The Problem

When multiple support agents are connected to the inbox simultaneously, state updates (new messages, status changes, conversation reordering) need to propagate to all connected clients consistently. An early issue caused the thread list sort order to desync — one operator would see a different "most recent" conversation than another.

### The Solution

**Consistent sort-on-update in the Zustand store:**

Every `conversation_update` WebSocket event triggers a re-sort of the `conversations` array by `lastMessageAt` descending — on every connected client simultaneously. The sort key is the server-side `lastMessageAt` timestamp from MongoDB, ensuring all clients converge to the same ordering.

**Room-based message targeting:**

Socket.io rooms (`join_room` event keyed by `conversationId`) ensure that high-frequency `message_new` and `message_status` events for a specific conversation only broadcast to agents who have that conversation open — not all connected agents. This prevents unnecessary state mutations in unrelated stores.

---

## Challenge 9: Next.js Standalone Build Configuration for Docker

### The Problem

The default Next.js production build includes a large `node_modules` directory that inflates the Docker image size significantly. Building the Docker image required copying all `node_modules` into the container, resulting in 2GB+ images and slow pull times.

### The Solution

Enabled Next.js `output: "standalone"` mode in `next.config.js`, which:
1. Traces only the files actually required to run the production build
2. Packages them into a self-contained `.next/standalone` directory
3. The resulting Docker image copies only `standalone/`, `public/`, and `.next/static/`

The Dockerfile uses a 2-stage build:
- **Stage 1**: Full build environment with all dev dependencies
- **Stage 2**: `node:20-alpine` with only the standalone output

This reduced the production image size by ~70% and significantly improved pull times during CI/CD deployments.
