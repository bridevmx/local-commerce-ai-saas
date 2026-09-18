# Engineering Roadmap & Implementation Milestones
## Local Commerce AI Assistant SaaS

**Document Version:** 1.0.0  
**Target Audience:** Project Managers, Engineering Leads, Product Owners  
**Delivery Model:** 5 Two-Week Sprints (10 Weeks Total to Production v1.0)  

---

## 1. Roadmap Overview & Timeline

```
SPRINT 1 (W1-W2)  │ [PHASE 1] Core Foundation & Communication Pipeline
SPRINT 2 (W3-W4)  │ [PHASE 2] AI Reasoning, OmniRoute & Handoff Pipeline
SPRINT 3 (W5-W6)  │ [PHASE 3] Fintech Core (PayClip) & Sales CRM
SPRINT 4 (W7-W8)  │ [PHASE 4] Self-Service Merchant Onboarding & Presets
SPRINT 5 (W9-W10) │ [PHASE 5] Security Hardening, Observability & Scaling
```

---

## 2. Phase-by-Phase Deliverables & Acceptance Criteria

### Phase 1: Core Foundation & Communication Pipeline (Sprint 1: Weeks 1–2)
**Goal:** Establish the persistent multi-tenant data layer and bidirectional messaging between WhatsApp Web and Telegram Supergroups.

#### Key Milestones:
* **M1.1: PocketBase Zero-JSON Schema Bootstrap:** Deploy PocketBase v0.23+ and run schema migrations for collections: `tenants`, `tenant_credentials`, `tenant_rules`, `tenant_tools`, `customers`, `customer_traits`, `sessions`, `tickets`, `messages`, `orders`, and `order_items`.
* **M1.2: Native Baileys WhatsApp Gateway:** Implement `WhatsAppSessionManager` capable of dynamically spinning up isolated `@whiskeysockets/baileys` sockets per tenant (`./storage/sessions/{tenant_id}`).
* **M1.3: In-Memory 30-Second Debounce Buffer:** Build `MessageAggregator` with sliding 30-second timers to concatenate rapid text messages and collect attachments.
* **M1.4: Telegram CRM Gateway:** Initialize Grammy framework with supergroup validation and topic routing support.

#### Acceptance Criteria (Definition of Done):
1. PocketBase contains all collections with indexed foreign keys and strict field constraints; zero JSON columns exist.
2. Sending 3 consecutive messages ("Hola", "Tienen pasteles?", "De chocolate") within 25 seconds results in **exactly one** consolidated event emitted after 30 seconds of silence.
3. WhatsApp connection simulates organic behavior (random 800-1500ms read delay and dynamic typing simulation).
4. Sockets automatically reconnect upon disconnection unless logged out.

---

### Phase 2: AI Reasoning, OmniRoute & Handoff Pipeline (Sprint 2: Weeks 3–4)
**Goal:** Integrate the self-hosted OmniRoute Gateway, implement the ReAct agent loop, and deploy the Telegram `#Dudas-Clientes` support handoff.

#### Key Milestones:
* **M2.1: OmniRoute Gateway & Audio Transcription:** Connect backend to OmniRoute (`http://localhost:20128/v1`), routing audio to Groq Whisper (`whisper-large-v3`, `language: 'es'`).
* **M2.2: Layered Prompt Compiler Engine:** Implement `buildTenantSystemPrompt(tenantId, customerId)` combining hardcoded guardrails, industry presets, business identity, active policies, and EAV traits.
* **M2.3: ReAct Engine with Guardrails:** Construct the ReAct reasoning loop with `MAX_ITERATIONS = 5` and stop condition handling (`FINAL_ANSWER`, `ESCALATE`).
* **M2.4: Human Handoff Bridge (`#Dudas-Clientes`):** Wire `escalate_to_human` to pause session, create a `tickets` record, post an alert in `#Dudas-Clientes`, and route merchant replies via Telegram `reply_to_message` directly back to the customer on WhatsApp.
* **M2.5: Asynchronous EAV Trait Extractor:** Implement background worker to detect and record customer traits (`PREFERENCE`, `HEALTH_RESTRICTION`, etc.) without blocking chat replies.

#### Acceptance Criteria (Definition of Done):
1. Voice notes sent via WhatsApp are transcribed into text with Spanish language accuracy > 95%.
2. Customer questions requiring human intervention pause the bot (`PAUSED_FOR_HUMAN`) and alert Telegram.
3. Replying to the Telegram alert quotes the ticket and dispatches the answer to the customer's WhatsApp in < 2 seconds.
4. Extracted traits persist to `customer_traits` and appear in the system prompt on the subsequent customer turn.
5. In case of Groq rate limits (HTTP 429), OmniRoute automatically falls back to Gemini 2.0 Flash without throwing unhandled exceptions.

---

### Phase 3: Fintech Core (PayClip) & Sales CRM (Sprint 3: Weeks 5–6)
**Goal:** Deliver end-to-end commerce transactions: catalog management, payment link generation, Zero-Trust webhook verification, and Telegram POS CRM.

#### Key Milestones:
* **M3.1: PayClip Client & Domain Isolation:** Build client modules for `api.payclip.com` (Checkout v2 / Basic Auth), `api.payclip.io` (Catalog / Bearer JWT), and `api-gw.payclip.com` (Cash / Telemetry).
* **M3.2: Catalog GET-before-PATCH Safeguard:** Implement `updateProductSafe` to preserve stock quantities during title or price edits.
* **M3.3: Voice/Text Catalog Admin (`#Mi-Catálogo`):** Enable merchants to update prices and inventory using voice notes or text in `#Mi-Catálogo`.
* **M3.4: Order Cards & Cash POS (`#Ventas-y-Caja`):** Post live interactive order cards in `#Ventas-y-Caja` with inline buttons (`[💵 Registrar Efectivo]`, `[❌ Cancelar]`) that update in place via `editMessageText`.
* **M3.5: Zero-Trust Webhook Readback:** Deploy webhook receiver with immediate HTTP 200 Fast-ACK followed by API readback (`GET /v2/checkout/{id}`) to verify payment status and exact currency/amount matching.

#### Acceptance Criteria (Definition of Done):
1. Merchant voice note ("El pastel de tres leches sube a $350") successfully updates PayClip catalog without resetting stock to `null`.
2. Completing a PayClip checkout triggers the webhook, completes the API readback, marks the order `PAID` in PocketBase, updates the Telegram card in place, and sends a payment receipt to the customer on WhatsApp.
3. Clicking `[Registrar Efectivo]` executes the 14-field hardware telemetry payload to `api-gw.payclip.com` and updates the order card.

---

### Phase 4: Self-Service Merchant Onboarding & Presets (Sprint 4: Weeks 7–8)
**Goal:** Streamline new merchant acquisition with zero-friction setup via Telegram and secure credential onboarding.

#### Key Milestones:
* **M4.1: Telegram Onboarding Wizard:** Interactive flow where a new merchant adds the bot to their Supergroup, and the bot detects `chat_id` and provisions the 3 forum topics.
* **M4.2: Ephemeral Secure Web Form:** Deliver single-use, 15-minute signed JWT web links for merchants to submit their PayClip API keys and Bearer tokens.
* **M4.3: Fintech AES-256-GCM Encryption:** Encrypt merchant API keys and credentials before storing them in `tenant_credentials`.
* **M4.4: Business Vertical Preset Provisioning:** Quick-select menu in Telegram to apply preconfigured presets (`FOOD_RETAIL`, `SERVICE_APPOINTMENTS`, `SUPPORT_LEAD`).
* **M4.5: WhatsApp QR Onboarding:** Render WhatsApp pairing QR code directly inside the merchant's Telegram private chat for immediate smartphone scanning.

#### Acceptance Criteria (Definition of Done):
1. A new merchant can complete onboarding (select vertical, scan WhatsApp QR, submit Clip keys) in under 5 minutes.
2. Merchant API keys are never stored in plaintext and never appear in Telegram message logs.
3. Forum topics `#Dudas-Clientes`, `#Mi-Catálogo`, and `#Ventas-y-Caja` are auto-mapped to `tenants` collection.

---

### Phase 5: Security Hardening, Observability & Scaling (Sprint 5: Weeks 9–10)
**Goal:** Prepare the platform for enterprise stability, disaster recovery, and multi-tenant operational scale.

#### Key Milestones:
* **M5.1: Zero-Port Cloudflare Tunnels:** Configure `cloudflared` daemon to route webhooks and onboarding forms through encrypted Cloudflare tunnels with rate-limiting and DDoS defense.
* **M5.2: Litestream Continuous WAL Replication:** Set up continuous streaming replication of PocketBase SQLite WAL to Cloudflare R2 / AWS S3 (<1s RPO).
* **M5.3: Automated Health Checks & Auto-Recovery:** Implement process supervision with `pm2` or Docker Compose with memory thresholds and auto-restart on socket stall.
* **M5.4: Comprehensive Integration Test Suite:** Develop end-to-end tests with mocked PayClip and WhatsApp sockets to validate ReAct loops, debounce timing, and webhook readbacks.

#### Acceptance Criteria (Definition of Done):
1. Host machine has zero open inbound listening ports (all traffic securely tunnels via Cloudflare).
2. Simulating a catastrophic server termination allows restoring the full database state from Cloudflare R2 with zero data loss.
3. System sustains 50 simulated concurrent WhatsApp conversations without memory leaks or dropped debounce queues.

---

## 3. Technical Risk Assessment & Mitigation Matrix

| Risk Scenario | Probability | Impact | Architectural Mitigation |
| :--- | :--- | :--- | :--- |
| **WhatsApp Socket Bans** | Low | High | Enforce 100% inbound architecture; simulate human reading delays (800-1500ms) and dynamic `composing` presence updates; use official User-Agents. |
| **Groq API Rate Limits (429)** | Medium | Medium | Local OmniRoute Gateway automatically switches to Google Gemini 2.0 Flash with zero downtime or manual code intervention. |
| **PayClip Catalog Stock Wipe** | High | Critical | Enforce mandatory `updateProductSafe` (GET-before-PATCH) in the catalog service layer; unit test with mock PayClip API. |
| **Fake Webhook Injection** | High | Critical | Zero-Trust Webhook Readback: never mark orders `PAID` from webhook payload alone; always execute signed GET verification to PayClip servers. |
| **SQLite WAL Lock Contention** | Low | High | PocketBase handles concurrency via Go channels and SQLite WAL mode; long operations (audio transcription, LLM generation) execute outside database write locks. |
