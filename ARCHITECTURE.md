# System Architecture Document (SAD)
## Local Commerce AI Assistant SaaS

**Document Version:** 1.0.0  
**Target Audience:** Software Engineering Team, Systems Architects, Backend Developers  
**System Status:** Ready for Engineering Implementation  

---

## 1. Executive Summary & Architectural Vision

The **Local Commerce AI Assistant SaaS** is a multi-tenant platform designed to automate commerce operations, customer support, and sales for local brick-and-mortar merchants (bakeries, barbershops, cafes, boutique clinics).

### Strategic Design Principles
1. **Zero Merchant Overhead (Mobile-First Operations):** Merchants do not log into complex web dashboards or install proprietary mobile apps. Their operational command center and CRM is a dedicated **Telegram Supergroup with Forum Topics**.
2. **Frictionless Customer Access:** End customers interact exclusively through **WhatsApp**, sending conversational bursts of text, voice notes, and images.
3. **Zero-JSON Relational Storage:** State and entity memory are maintained in a high-performance **PocketBase (SQLite in WAL mode)** database with zero JSON columns, ensuring relational integrity, indexed queries, and straightforward auditability.
4. **Resilient AI Gateway via OmniRoute:** Inference is routed through a local self-hosted **OmniRoute AI Gateway**, offering ultra-low latency (<400ms TTFT) via Groq Cloud (Llama 3.3 70B & Whisper Large v3) paired with quota-aware automatic fallback to Google Gemini models (Gemini 2.0 Flash) for Tool Calling, trait extraction, and multimodal image analysis.
5. **Empirical Fintech Hardening:** Direct integration with PayClip APIs respects production rules: domain isolation, GET-before-PATCH catalog stock protection, and Zero-Trust Webhook Readback validation.

---

## 2. High-Level System Topology

```
                  ┌────────────────────────────────────────────────────────┐
                  │                 END CUSTOMERS (WHATSAPP)               │
                  └──────────────────────────┬─────────────────────────────┘
                                             │ WhatsApp Web Protocol (WS)
                                             ▼
┌───────────────────────────────────────────────────────────────────────────────────────────┐
│ BACKEND APPLICATION CORE (Node.js LTS / TypeScript or ESM)                                 │
│                                                                                           │
│   ┌───────────────────────────────────────────────────────────────────────────────────┐   │
│   │ WHATSAPP GATEWAY (Baileys Native Socket Pool)                                     │   │
│   │ - Multi-tenant session isolation (./storage/sessions/{tenant_id})                 │   │
│   │ - Organic human behavior simulation (random read delay, dynamic typing status)   │   │
│   └────────────────────────────────────────┬──────────────────────────────────────────┘   │
│                                            │ Raw Events                                   │
│                                            ▼                                              │
│   ┌───────────────────────────────────────────────────────────────────────────────────┐   │
│   │ 30-SECOND SLIDING DEBOUNCE BUFFER (MessageAggregator)                             │   │
│   │ - In-memory rolling window per (tenant_id:customer_id)                            │   │
│   │ - Flushes concatenated text, transcribed audio batch, and vision payload          │   │
│   └────────────────────────────────────────┬──────────────────────────────────────────┘   │
│                                            │ Consolidated Prompt Payload                  │
│                                            ▼                                              │
│   ┌───────────────────────────────────────────────────────────────────────────────────┐   │
│   │ AGENT RE-ACT ENGINE & TOOL ORCHESTRATOR                                           │   │
│   │ - Layered Prompt Compiler (System Guardrails + Preset + Identity + Rules + EAV)   │   │
│   │ - Max Iterations Guardrail: 5 iterations                                          │   │
│   │ - Stop Conditions: FINAL_ANSWER, ESCALATE, MAX_ITERATIONS                         │   │
│   └───────────────┬────────────────────────┬──────────────────────────┬───────────────┘   │
│                   │                        │                          │                   │
│                   │ Tool Calling           │ Webhook / API Readback   │ Native Bridge     │
│                   ▼                        ▼                          ▼                   │
│   ┌────────────────────────┐  ┌────────────────────────┐  ┌───────────────────────────┐   │
│   │ OMNIROUTE AI GATEWAY   │  │ PAYCLIP FINTECH CORE   │  │ TELEGRAM CRM BRIDGE       │   │
│   │ http://localhost:20128 │  │ - Checkout v2 (api)    │  │ - Grammy Framework        │   │
│   │ - Primary: Groq Llama  │  │ - F2F Catalog (io)     │  │ - Supergroup Forum Topics │   │
│   │ - Fallback: Gemini     │  │ - Cash TPV (api-gw)    │  │ - In-place msg mutations  │   │
│   └────────────────────────┘  └────────────────────────┘  └─────────────┬─────────────┘   │
│                                                                         │                 │
│                                            ┌────────────────────────────┘                 │
│                                            ▼                                              │
│                            ┌───────────────────────────────────┐                          │
│                            │ POCKETBASE DATA LAYER (Zero-JSON) │                          │
│                            │ SQLite in WAL Mode + Litestream   │                          │
│                            └───────────────────────────────────┘                          │
└───────────────────────────────────────────────────────────────────────────────────────────┘
                                             │ Telegram Bot API (MTProto)
                                             ▼
                  ┌────────────────────────────────────────────────────────┐
                  │            MERCHANTS & STAFF (TELEGRAM TOPICS)         │
                  │  - #Dudas-Clientes  |  #Mi-Catálogo  |  #Ventas-y-Caja │
                  └────────────────────────────────────────────────────────┘
```

---

## 3. Component Details & Subsystems

### 3.1 WhatsApp Gateway Subsystem (`@whiskeysockets/baileys`)
- **Direct Native Sockets:** Eliminates overhead by bypassing wrapper frameworks (such as BuilderBot). Runs raw `@whiskeysockets/baileys` directly over WebSockets, consuming ~35-45 MB RAM per active session.
- **Session Lifecycle & Storage:**
  - Active credentials reside locally in `./storage/sessions/{tenant_id}/` via `useMultiFileAuthState`.
  - On shutdown or credential rotation, sessions are encrypted (AES-256-GCM) and backed up to PocketBase `tenant_credentials`.
- **Organic Human-like Emulation (Anti-Ban Guard):**
  - Messages trigger a realistic reading delay (800ms - 1500ms) before emitting `sock.readMessages([key])`.
  - During response generation, the bot emits `sock.sendPresenceUpdate('composing', jid)` with duration proportional to response length (~60ms per character, clamped between 2s and 7s) before dispatching the final payload.
  - User-Agent handshake emulation uses current stable browser signatures (e.g., `['Ubuntu', 'Chrome', '124.0.0.0']`).

### 3.2 In-Memory 30-Second Sliding Debounce Buffer (`MessageAggregator`)
Local commerce customers typically send 3-5 rapid messages ("Good afternoon", "Do you have strawberry cake?", [photo of cake]). An immediate response to the first message results in broken context and duplicate API executions.
- **Key Partitioning:** Unique key per active thread: `${tenantId}:${customerPhone}`.
- **Debounce Window:** 30,000 milliseconds rolling timer (`clearTimeout` on incoming event + `setTimeout(30000)`).
- **Batch Processing on Flush:**
  1. Voice notes (`audio/ogg; codecs=opus`) are sent concurrently to Groq Whisper (`whisper-large-v3`, `language: 'es'`).
  2. Text chunks are merged in chronological order: `[14:02:01] Text 1 \n [14:02:15] Transcribed Audio \n [14:02:28] Text 2`.
  3. Image attachments are packaged into data URIs or temporary buffers for multimodal inspection by Gemini 2.0 Flash via OmniRoute.
  4. The aggregated payload is submitted to the ReAct Engine as a single user turn.

### 3.3 AI Orchestration via OmniRoute Gateway
The system connects to a local, self-hosted OmniRoute instance (`http://localhost:20128/v1`), exposing a standard OpenAI-compatible API:
- **Routing & Model Combos:**
  - **Text & Tool Calling Primary:** `groq/llama-3.3-70b-versatile` (<400ms time-to-first-token).
  - **Automatic Quota/Rate-Limit Fallback:** `google/gemini-2.0-flash-001` or `google/gemini-1.5-flash`.
  - **Multimodal Vision:** Customer images route to `google/gemini-2.0-flash`.
  - **Speech-to-Text:** `groq/whisper-large-v3`.
- **ReAct Execution Parameters:**
  - Maximum internal iterations: **5 iterations** (`MAX_ITERATIONS`).
  - Stop conditions:
    1. `FINAL_ANSWER`: Synthesized conversational text sent to WhatsApp customer.
    2. `ESCALATE`: Tool `escalate_to_human` invoked -> updates PocketBase session status to `PAUSED_FOR_HUMAN`, creates ticket in `#Dudas-Clientes`, alerts merchant in Telegram.
    3. `MAX_ITERATIONS` Exceeded: Guardrail auto-pause -> notifies merchant of reasoning deadlock and hands off to human.

### 3.4 Telegram Operations & CRM Subsystem
Each merchant operates within their private Telegram Supergroup. The SaaS Master Bot is added as Administrator.
- **Isolation by `telegram_chat_id`:** PocketBase verifies that incoming updates from Telegram match `tenants.telegram_chat_id`. Updates from unauthorized groups are rejected.
- **Topic 1: Customer Inquiries & Handoff (`#Dudas-Clientes`):**
  - When the bot escalates or the customer requests a human, a message is posted to `topic_support_id`.
  - The merchant replies natively using Telegram's **Reply-to-Message** feature.
  - The bridge intercepts the reply, extracts the original `ticket_id` from the quote context, sends the merchant's answer directly to the customer's WhatsApp chat, and records the message in `messages`.
- **Topic 2: Catalog & Store Management (`#Mi-Catálogo`):**
  - Merchant submits voice notes or text (e.g., "Raise chocolate cake to $380 and mark strawberry as out of stock").
  - The backend transcribes audio, parses intent via Tool Calling (`update_catalog_item`), enforces the **GET-before-PATCH** stock safeguard, updates PayClip, and posts confirmation.
- **Topic 3: Sales & Cashier CRM (`#Ventas-y-Caja`):**
  - When a customer confirms an order, an interactive order card is published with Inline Keyboard buttons (`[Aprobar Pedido]`, `[Registrar Pago Efectivo]`, `[Cancelar]`).
  - State changes mutate the existing message using `editMessageText` and `editMessageReplyMarkup`, providing a clean, noise-free operational dashboard.

---

## 4. Layered Prompt Compiler (`buildTenantSystemPrompt`)

To prevent prompt drift and eliminate manual prompt writing for merchants, the system dynamically compiles prompts at runtime using 5 deterministic layers:

```
┌────────────────────────────────────────────────────────────────────────┐
│ LAYER 1: System Guardrails & ReAct Protocol (Hardcoded)                │
│ - Identity restrictions, tone boundaries, anti-jailbreak defenses.     │
│ - Strict tool invocation schemas and output format requirements.       │
├────────────────────────────────────────────────────────────────────────┤
│ LAYER 2: Business Vertical Preset (From Presets Catalog)               │
│ - Preconfigured schemas for FOOD_RETAIL, APPOINTMENTS, SERVICES.       │
│ - Standard operational flows and domain-specific best practices.       │
├────────────────────────────────────────────────────────────────────────┤
│ LAYER 3: Tenant Identity & Hours (From `tenants` Collection)           │
│ - Name, physical address, business hours, selected tone enum.          │
├────────────────────────────────────────────────────────────────────────┤
│ LAYER 4: Merchant Policies & Rules (From `tenant_rules` Collection)    │
│ - Discrete relational rows: SHIPPING, PAYMENT, REFUND, BOOKING, FAQ.   │
├────────────────────────────────────────────────────────────────────────┤
│ LAYER 5: Customer Long-Term Memory (From `customer_traits` EAV)        │
│ - Injected facts: allergies, favorite flavors, previous sizes, names.  │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 5. State Machines & Core Workflows

### 5.1 Ticket & Support Handoff FSM
```
[CUSTOMER INCOMING MESSAGE]
           │
           ▼
     [BOT_ACTIVE] ─────────────── Customer requests human or bot unconfident
           │                                    │
           │ Normal interaction                 ▼
           │                             [PAUSED_FOR_HUMAN]
           │                                    │
           │                                    │ Dispatch Telegram Ticket to #Dudas
           │                                    │ Merchant replies via reply_to_message
           │                                    ▼
           │                             [HUMAN_INTERVENTION]
           │                                    │
           │ Merchant sends "/resume"           │
           │ or inactivity timeout (2 hours)    │
           └────────────────────────────────────┘
```

### 5.2 Order Processing Lifecycle FSM
```
[DRAFT] ─── Customer confirms items
   │
   ▼
[PENDING_PAYMENT] ─── `create_payment_link` generated via PayClip Checkout v2
   │
   ├─────── Webhook received + API Readback verified ───► [PAID] ───► [FULFILLED]
   │
   ├─────── Merchant records manual cash payment ───────► [PAID] ───► [FULFILLED]
   │
   └─────── Expiration (30 min) / Merchant Cancelled ───► [CANCELLED]
```

---

## 6. Asynchronous Customer Memory Worker (EAV Pattern)

Customer personalization must never add latency to the customer's response cycle.
- **Fire-and-Forget Architecture:** Immediately after `sock.sendMessage` dispatches the final response, `setImmediate()` triggers `extractCustomerTraits(tenantId, customerId, conversationSnippet)`.
- **Extraction Protocol:**
  - A lightweight prompt evaluates whether the customer stated explicit facts (allergies, preferences, family dates, sizes).
  - The LLM calls the internal tool `store_customer_fact` with strict enum categories: `PREFERENCE`, `HEALTH_RESTRICTION`, `IMPORTANT_DATE`, `CONTACT_PREFERENCE`, `METRIC_SIZE`.
  - The worker performs an upsert into PocketBase collection `customer_traits`.

---

## 7. Infrastructure, Security & Reliability

1. **Zero-Trust Network Access:** Backend endpoints and webhooks are exposed strictly via **Cloudflare Tunnels (`cloudflared`)**. No inbound public ports (80/443) are opened on the host machine.
2. **Fintech Credential Encryption:** Merchant PayClip keys and Telegram tokens are stored in `tenant_credentials` encrypted with **AES-256-GCM** using a master hardware/environment key (`MASTER_ENCRYPTION_KEY`). Plaintext secrets never appear in logs or Telegram chats.
3. **Database Streaming Replication:** The SQLite database is configured in **WAL (Write-Ahead Logging)** mode. **Litestream** continuously streams WAL changes to an S3/Cloudflare R2 bucket, guaranteeing point-in-time recovery (<1s RPO) without database locks.
4. **Process Supervision:** The Node.js application and PocketBase binary are managed via `systemd` or containerized Docker Compose with automated health check restarts.
