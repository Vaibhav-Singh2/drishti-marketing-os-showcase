# System Architecture

> **Disclaimer**: This repository is a technical case study. The original implementation is proprietary and owned by the employer. No confidential source code, credentials, or sensitive business information is included.

---

## 1. High-Level System Architecture

The system is structured as a **Turborepo monorepo** containing two applications — a Next.js 16 web frontend and a Bun/Express API backend — backed by three data stores (MongoDB, Qdrant, Redis) and integrated with Meta's WhatsApp Cloud API and Zoho's India DC APIs.

```mermaid
graph TB
    subgraph External["External Services"]
        META["Meta WhatsApp\nCloud API v20.0"]
        ZOHO["Zoho India DC\nCRM + Books"]
        LLM["AI Providers\nClaude · GPT-4o · Gemini"]
        R2["Cloudflare R2\nMedia Storage"]
        ECR["Amazon ECR\nContainer Registry"]
    end

    subgraph Monorepo["Drishti Marketing OS — Turborepo Monorepo"]
        subgraph Frontend["apps/web — Next.js 16"]
            LOGIN["/login\nAuth + 2FA"]
            INBOX["/inbox\nShared Chat Console"]
            DASH["/dashboard\nAnalytics"]
            AISETTINGS["/ai-settings\nModel Config + KB Upload"]
            QUEUES["/queues\nBullMQ Monitor"]
            TEMPLATES["/templates\nBroadcast Campaigns"]
        end

        subgraph Backend["apps/api — Express + Bun"]
            WEBHOOK["Webhook Controller\nHMAC Verification"]
            AUTH["Auth Controller\nJWT + 2FA TOTP"]
            CHAT["Chat Controller\nInbox API"]
            CONFIG["Config Controller\nGlobal AI Settings"]
            MEDIA["Media Controller\n16MB Upload + R2 Presign"]
            ANALYTICS["Analytics Controller\nToken Cost Aggregation"]

            subgraph Queues["BullMQ Background Workers"]
                IQ["inbound-msg-queue\nInboundWorker"]
                OQ["outbound-msg-queue\nOutboundWorker"]
                VQ["vector-ingest-queue\nVectorWorker"]
                SQ["status-msg-queue\nStatusWorker"]
            end

            subgraph AILayer["AI Services"]
                RAG["RAG Pipeline\nQdrant Semantic Search"]
                SWITCH["AI Switchboard\nProvider Router"]
                TOOLS["Agent Tool Loop\nFunction-Calling"]
            end

            SOCKET["Socket Service\nWebSocket Emitter"]
        end

        subgraph DataStores["Data Stores"]
            MONGO["MongoDB Atlas\nPrimary Database"]
            QDRANT["Qdrant\nVector Store\n1536-dim Cosine"]
            REDIS["Redis\nQueue Backplane\n+ Cache"]
        end
    end

    %% Frontend ↔ Backend
    INBOX <-->|"HTTP + Cookie Auth"| CHAT
    INBOX <-->|"WebSocket / Socket.io"| SOCKET
    LOGIN -->|"POST /auth/login"| AUTH
    AISETTINGS -->|"POST /documents/upload"| VQ
    QUEUES -->|"GET /queues"| QUEUES
    TEMPLATES -->|"POST /templates/broadcast"| OQ

    %% Meta Webhook flow
    META -->|"POST /messages/webhook\nHMAC Signed"| WEBHOOK
    WEBHOOK -->|"Enqueue payload"| IQ
    WEBHOOK -->|"200 EVENT_RECEIVED < 2s"| META

    %% Inbound Worker
    IQ -->|"Process job"| RAG
    IQ -->|"Invoke LLM"| SWITCH
    SWITCH --> TOOLS
    TOOLS --> ZOHO
    IQ -->|"Emit events"| SOCKET
    IQ -->|"Enqueue reply"| OQ

    %% Outbound Worker
    OQ -->|"POST /messages"| META

    %% Vector Worker
    VQ -->|"Upsert 1536-dim vectors"| QDRANT

    %% Data store connections
    Backend <-->|"Mongoose ODM"| MONGO
    RAG <-->|"REST Client"| QDRANT
    Backend <-->|"ioredis"| REDIS

    %% External AI
    SWITCH -->|"Completions API"| LLM

    %% Media
    MEDIA -->|"Pre-signed PUT"| R2

    %% CI/CD
    ECR -->|"docker pull"| Backend
```

---

## 2. Request Flow — Inbound Message Lifecycle

Every inbound WhatsApp message traverses a 7-layer pipeline designed for sub-2s acknowledgement, idempotency, and grounded AI response generation.

```mermaid
sequenceDiagram
    autonumber
    actor Customer as WhatsApp Customer
    participant Meta as Meta Cloud API
    participant Webhook as Webhook Controller
    participant Redis as Redis / BullMQ
    participant InWorker as Inbound Worker
    participant RAG as RAG Pipeline
    participant LLM as AI Switchboard
    participant DB as MongoDB
    participant Socket as Socket Service
    actor Agent as Support Agent (Next.js)
    participant OutWorker as Outbound Worker

    Customer->>Meta: Sends Message
    Meta->>Webhook: POST /messages/webhook (HMAC Signed)
    Note over Webhook: Constant-time HMAC SHA256 verification
    Webhook->>Redis: Enqueue → inbound-msg-queue
    Webhook-->>Meta: HTTP 200 EVENT_RECEIVED (< 2s)

    Redis->>InWorker: Process job
    Note over InWorker: metaWamid idempotency check
    InWorker->>DB: Find/Create Contact + Conversation
    InWorker->>DB: Save Message (direction: inbound)
    InWorker->>Socket: Emit message_new + conversation_update
    Socket-->>Agent: Real-time UI update

    Note over InWorker: Check conversation.isAiActive
    InWorker->>RAG: retrieveContext(query)
    RAG->>LLM: Generate embedding (text-embedding-3-small)
    RAG->>DB: Cosine search Qdrant (threshold > 0.3)
    InWorker->>DB: Fetch last 10 messages (conversation history)
    InWorker->>LLM: invokeLLM(5-part context)
    LLM->>DB: Log tokens + cost → CostTracking

    alt Supervised Draft Mode
        InWorker->>DB: Save response (status: draft)
        InWorker->>Socket: Emit message_new (draft)
        Socket-->>Agent: Show Suggested Reply card
        Agent->>Webhook: POST /messages/:id/approve
        Webhook->>Redis: Enqueue → outbound-msg-queue
    else Autonomous Mode
        InWorker->>DB: Save response (status: pending)
        InWorker->>Redis: Enqueue → outbound-msg-queue
    end

    Redis->>OutWorker: Process job
    OutWorker->>Meta: POST /messages (Bearer token)
    Meta-->>OutWorker: Return wamid
    OutWorker->>DB: Update status → sent + metaWamid
    OutWorker->>Socket: Emit message_status (sent)
    Socket-->>Agent: Update double-checkmarks

    Meta->>Webhook: Async status update (delivered/read)
    Webhook->>Redis: Enqueue → status-msg-queue
    Redis->>InWorker: Process status job
    InWorker->>DB: Update message status
    InWorker->>Socket: Emit message_status
    Socket-->>Agent: Render delivered/read indicators
```

---

## 3. Component Interaction Diagram

```mermaid
graph LR
    subgraph FE["Frontend (Next.js 16)"]
        AUTH_STORE["authStore.ts\nZustand"]
        INBOX_STORE["store.ts\nZustand"]
        SIDEBAR_STORE["sidebarStore.ts\nZustand"]
        SOCKET_CLIENT["socket.ts\nSocket.io-client"]
        INBOX_LAYOUT["InboxLayout.tsx"]
        COMPOSER["ChatComposer.tsx"]
    end

    subgraph BE["Backend (Express + Bun)"]
        AUTH_C["authController"]
        CHAT_C["chatController"]
        WEBHOOK_C["webhookController"]
        CONFIG_C["configController"]
        SOCKET_S["socketService"]
        AI_SWITCH["aiSwitchboard"]
        RAG_P["ragPipeline"]
        ZOHO_CRM["zohoCRM"]
        ZOHO_BOOKS["zohoBooks"]
        INBOUND_W["inboundWorker"]
        OUTBOUND_W["outboundWorker"]
        VECTOR_W["vectorWorker"]
    end

    INBOX_LAYOUT --> AUTH_STORE
    INBOX_LAYOUT --> INBOX_STORE
    INBOX_LAYOUT --> SOCKET_CLIENT
    COMPOSER --> CHAT_C

    SOCKET_CLIENT <-->|WebSocket| SOCKET_S
    AUTH_STORE <-->|HTTP| AUTH_C
    INBOX_STORE <-->|HTTP| CHAT_C

    WEBHOOK_C --> INBOUND_W
    INBOUND_W --> AI_SWITCH
    INBOUND_W --> RAG_P
    INBOUND_W --> ZOHO_CRM
    INBOUND_W --> ZOHO_BOOKS
    INBOUND_W --> OUTBOUND_W
    INBOUND_W --> SOCKET_S

    AI_SWITCH -->|Claude| LLM1["Anthropic API"]
    AI_SWITCH -->|GPT-4o| LLM2["OpenAI API"]
    AI_SWITCH -->|Gemini| LLM3["Google GenAI"]

    RAG_P <-->|REST| QDRANT_DB["Qdrant\nVector DB"]
    INBOUND_W <-->|Mongoose| MONGO_DB["MongoDB"]
    VECTOR_W --> QDRANT_DB
```

---

## 4. Deployment Architecture

```mermaid
graph TB
    subgraph CI["GitHub Actions CI/CD Pipeline"]
        PUSH["git push → main"]
        LINT["Lint + Type Check"]
        BUILD["Docker BuildKit\nmulti-stage builds"]
        PUSH_ECR["Push to\nAmazon ECR"]
        SSH["SSH Deploy\nto EC2"]
    end

    subgraph AWS["AWS EC2 — Mumbai Region (ap-south-1)"]
        NGINX["Nginx Reverse Proxy\nport 80 / 443\nclient_max_body_size 16M"]

        subgraph Docker["Docker Compose Network (drishti-network)"]
            API_C["api container\nBun Runtime\n:5005"]
            WEB_C["web container\nNode.js Standalone\n:3005"]
            REDIS_C["redis container\nPersistent Volume"]
            QDRANT_C["qdrant container\nqdrant_prod_data volume"]
        end

        HEALTH["/healthz\nHTTP 200 if DB+Redis ready\nHTTP 503 otherwise"]
    end

    subgraph Managed["Managed Services"]
        MONGO_ATLAS["MongoDB Atlas\nManaged Cluster"]
        CF_R2["Cloudflare R2\nMedia Bucket"]
        CLOUDWATCH["AWS CloudWatch\nLog Aggregation"]
    end

    PUSH --> LINT --> BUILD --> PUSH_ECR --> SSH
    SSH -->|"docker compose pull api web\ndocker compose up -d"| Docker

    NGINX --> API_C
    NGINX --> WEB_C

    API_C --> REDIS_C
    API_C --> QDRANT_C
    API_C --> MONGO_ATLAS
    API_C --> CF_R2
    API_C -->|"stdout JSON logs"| CLOUDWATCH
```

---

## 5. Database Relationship Diagram

```mermaid
erDiagram
    User {
        ObjectId _id PK
        string email UK
        string passwordHash
        string name
        string role "admin | agent"
        string twoFactorSecret
        bool twoFactorEnabled
        bool isActive
    }

    Contact {
        ObjectId _id PK
        string phoneNumber UK
        string name
        string email
        string zohoContactId
        bool isActive
    }

    Conversation {
        ObjectId _id PK
        ObjectId contactId FK
        bool isAiActive
        bool isAiDraftMode
        string status "open | closed"
        date lastMessageAt
    }

    Message {
        ObjectId _id PK
        ObjectId conversationId FK
        ObjectId senderId FK
        string direction "inbound | outbound"
        string senderType "user | bot | contact"
        string text
        string messageType
        string status "sent | delivered | read | draft | pending | failed"
        string metaWamid UK
    }

    AISession {
        ObjectId _id PK
        ObjectId conversationId FK,UK
        string provider "claude | openai | gemini"
        string modelName
        number maxHistoryMessages
        number temperature
    }

    CostTracking {
        ObjectId _id PK
        ObjectId conversationId FK
        string provider
        string modelName
        number inputTokens
        number outputTokens
        number calculatedCostUSD
    }

    AuditLog {
        ObjectId _id PK
        ObjectId userId FK
        string action
        string description
        string clientIp
    }

    KnowledgeDocument {
        ObjectId _id PK
        string fileName
        string status "processing | completed | failed"
        number chunkCount
    }

    GlobalConfig {
        ObjectId _id PK
        string key UK
        string defaultProvider
        string defaultSystemPrompt
        number ragSimilarityThreshold
    }

    Contact ||--o{ Conversation : "has"
    Conversation ||--o{ Message : "contains"
    User ||--o{ Message : "sends"
    Conversation ||--|| AISession : "has"
    Conversation ||--o{ CostTracking : "tracks"
    User ||--o{ AuditLog : "generates"
```

---

## 6. AI Agent Tool-Calling Loop

```mermaid
flowchart TD
    START([Inbound Message Received]) --> CHECK{isAiActive?}
    CHECK -->|No| IDLE[Wait for human agent]
    CHECK -->|Yes| ASSEMBLE[Assemble 5-Part Context]

    ASSEMBLE --> BRAND[1. Brand Guidelines\n& Safety Instructions]
    ASSEMBLE --> RAG_CTX[2. RAG Semantic Chunks\nfrom Qdrant]
    ASSEMBLE --> HISTORY[3. Conversation History\nlast 10 messages]
    ASSEMBLE --> QUERY[4. Current User Query]
    ASSEMBLE --> ZOHO_CTX[5. Zoho CRM + Books\nLive Records]

    BRAND & RAG_CTX & HISTORY & QUERY & ZOHO_CTX --> INVOKE[Invoke AI Switchboard]

    INVOKE --> TOOL_CALL{LLM needs\ntool call?}
    TOOL_CALL -->|Yes| TOOLS["Execute Tool\n• lookupCustomerByPhone\n• lookupCustomerByEmail\n• getDealById\n• verifyPayment\n• searchKnowledgeBase\n• getCheckoutLink\n• shareReportLink\n• requestRefund\n• escalateToHuman"]
    TOOLS --> VERIFY{Ownership\nverified?}
    VERIFY -->|Yes| INJECT[Inject secure link\ninto context]
    VERIFY -->|No| WITHHOLD[Withhold link\nPrompt for Deal ID]
    INJECT & WITHHOLD --> INVOKE
    TOOL_CALL -->|No| RESPONSE[Generate Final Response]

    RESPONSE --> MODE{isAiDraftMode?}
    MODE -->|Supervised| DRAFT[Save as Draft\nEmit to Agent UI]
    MODE -->|Autonomous| QUEUE[Queue to outbound-msg-queue\nSend immediately]

    DRAFT --> AGENT_REVIEW{Agent Action}
    AGENT_REVIEW -->|Approve| QUEUE
    AGENT_REVIEW -->|Reject| DELETE[Delete Draft]
```
