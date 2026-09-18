# Local Commerce AI Assistant SaaS
## Production Engineering Documentation & Technical Specifications Suite

**System Version:** 1.0.0  
**Status:** Engineering Ready / Development Phase  
**Target Architecture:** Multi-Tenant Node.js + PocketBase (Zero-JSON) + OmniRoute Gateway + Baileys + Telegram Supergroup CRM  

---

## 1. Executive Project Overview

The **Local Commerce AI Assistant SaaS** is an autonomous conversational commerce platform tailored for local brick-and-mortar merchants (bakeries, barbershops, cafes, boutique clinics). It solves operational friction by eliminating proprietary merchant dashboards and bridging two omnipresent communication channels:

* **Customer Channel (WhatsApp):** Customers send text bursts, voice notes, and reference photos. Incoming messages are debounced through an in-memory 30-second sliding window buffer before triggering an autonomous ReAct agent loop.
* **Merchant Command Center (Telegram Supergroup with Forum Topics):** The business owner and staff manage their entire operation without installing new apps:
  - **`#Dudas-Clientes`:** Human customer service handoff; staff replies natively using Telegram quotes (`reply_to_message`).
  - **`#Mi-Catálogo`:** Voice or text commands to adjust prices and stock in the PayClip POS catalog.
  - **`#Ventas-y-Caja`:** Real-time interactive order cards with inline keyboards that mutate in place via `editMessageText`.

---

## 2. Master Documentation Index

This repository contains the complete specification suite required by a custom software engineering agency to implement, test, and deploy the system:

| Document | File Path | Focus & Core Content |
| :--- | :--- | :--- |
| **System Architecture** | [`ARCHITECTURE.md`](./ARCHITECTURE.md) | High-level topology, component diagrams, FSMs for tickets and orders, multi-tenant isolation, and EAV async memory extraction. |
| **Technology Stack** | [`TECH_STACK.md`](./TECH_STACK.md) | Runtime (Node.js 20+ LTS), native Baileys engine, PocketBase v0.23+, OmniRoute AI Gateway, GramMY, Litestream, and Cloudflare Tunnels. |
| **Database Schema** | [`DATABASE_SCHEMA.md`](./DATABASE_SCHEMA.md) | Strict Zero-JSON PocketBase relational model, 11 normalized collections, composite indexes, API security rules, and database hooks. |
| **External Integrations** | [`INTEGRATIONS.md`](./INTEGRATIONS.md) | PayClip 3-domain contracts, GET-before-PATCH catalog stock safeguard, Zero-Trust Webhook Readback, 14-field TPV telemetry, and Telegram bot routing. |
| **Tool Registry & Presets** | [`TOOL_REGISTRY_AND_PRESETS.md`](./TOOL_REGISTRY_AND_PRESETS.md) | ReAct loop rules (MAX_ITERATIONS = 5), Zod tool schemas, vertical presets (`FOOD_RETAIL`, `SERVICE_APPOINTMENTS`, etc.), and 5-layer prompt compiler. |
| **Project Roadmap** | [`ROADMAP.md`](./ROADMAP.md) | 5 two-week sprints (10 weeks total to v1.0), milestones, deliverables, risk matrix, and formal Acceptance Criteria (Definition of Done). |
| **Implementation Plan** | [`IMPLEMENTATION_PLAN.md`](./IMPLEMENTATION_PLAN.md) | Repository directory blueprint, 10-step atomic development order, code blueprints for critical modules, and QA testing strategy with mocks. |

---

## 3. Seven Non-Negotiable Architectural Decisions

1. **Direct Native Baileys (No BuilderBot Bloat):** Use `@whiskeysockets/baileys` directly over WebSockets (~35MB RAM/session). Enforce organic human simulation (800-1500ms read delay, dynamic `composing` typing duration) and 100% inbound traffic to prevent socket bans.
2. **30-Second In-Memory Sliding Debounce Window:** Inbound messages per `tenant:customer` thread reset a 30,000ms rolling timer. On flush, all audio notes are transcribed in parallel with Groq Whisper (`whisper-large-v3`, `language: 'es'`) and text is concatenated into a single consolidated turn.
3. **Strict Zero-JSON Database Policy:** PocketBase (SQLite in WAL mode) stores all data across normalized relational tables with typed columns, foreign keys, and composite indexes.
4. **EAV Pattern for Customer Long-Term Memory:** Personalization traits (allergies, preferences, sizes) are stored in `customer_traits` with LLM-validated enum categories and updated asynchronously (*fire-and-forget* via `setImmediate`) after replying to the customer.
5. **Self-Hosted OmniRoute AI Gateway (`http://localhost:20128/v1`):** Unified OpenAI-compatible endpoint with quota-aware automatic routing: primary execution on Groq (`llama-3.3-70b-versatile`) with seamless fallback to Google Gemini (`gemini-2.0-flash`) for rate-limit protection, tool calling, and multimodal customer photo analysis.
6. **PayClip Catalog Stock Safeguard (`GET-before-PATCH`):** When modifying product pricing or titles via `PATCH /f2f/catalog/products/:id`, the system must query the product first via `GET` and explicitly re-inject the current `stock` parameter to prevent PayClip from resetting inventory to `null`.
7. **Zero-Trust Webhook Readback:** Incoming PayClip Checkout webhooks receive an immediate HTTP 200 Fast-ACK (<1s). The order is only marked `PAID` after an asynchronous backend call executes `GET https://api.payclip.com/v2/checkout/{id}` with Basic Auth and confirms status and exact amount matching.

---

## 4. Quickstart for Developers

### Prerequisites
* Node.js v20.18.0+ LTS & `pnpm`
* PocketBase v0.23+ executable
* Self-hosted OmniRoute Gateway running on port `20128`

### Setup Instructions
```bash
# 1. Clone repository and install dependencies
cd local-commerce-ai-saas
pnpm install

# 2. Configure environment variables
cp .env.example .env
# Edit .env with your MASTER_ENCRYPTION_KEY, TELEGRAM_BOT_TOKEN, and POCKETBASE credentials

# 3. Start PocketBase server
./pocketbase serve --http=127.0.0.1:8090

# 4. Run automated database migrations
pnpm run db:migrate

# 5. Start OmniRoute AI Gateway daemon
omniroute start

# 6. Start the SaaS Backend Engine
pnpm run dev
```
