# Technology Stack Specification
## Local Commerce AI Assistant SaaS

**Document Version:** 1.0.0  
**Target Audience:** DevOps Engineers, Lead Developers, Systems Administrators  

---

## 1. Core Runtime & Execution Environment

| Component | Selected Technology | Minimum Version | Rationale |
| :--- | :--- | :--- | :--- |
| **Server Runtime** | Node.js (LTS) | `v20.18.0+` (Iron) | Native ESM support, high stability with WebSocket socket pools, mature crypto primitives. |
| **Package Manager** | `pnpm` | `9.x+` | Fast, space-efficient hard links, strict dependency isolation. |
| **Language Standards** | Modern JavaScript (ESM) | ES2023+ | Clean asynchronous workflows (`async/await`), native private identifiers, top-level await. |
| **Operating System** | Linux (Ubuntu / Debian) | Ubuntu 22.04 LTS | Standard kernel support for TCP keep-alive, epoll socket scaling, and systemd units. |

---

## 2. Dependencies & Production Libraries

### 2.1 WhatsApp Connectivity Engine
* **`@whiskeysockets/baileys` (`^6.7.12`):** Direct WebSocket protocol client for WhatsApp Web. Chosen over heavyweight wrappers (like BuilderBot) to reduce RAM footprint (~35MB vs ~80MB per session) and provide fine-grained control over presence updates and socket lifecycle.
* **`pino` (`^9.0.0`):** Ultra-fast structured JSON logger required by Baileys. Configured to level `warn` or `error` in production to eliminate console bottlenecking.
* **`qrcode-terminal` (`^0.12.0`):** CLI QR renderer for initial development and fallback provisioning.

### 2.2 Telegram Bot Framework
* **`grammy` (`^1.30.0`):** Modern, high-performance Telegram Bot framework built with TypeScript/ESM. Superior error handling and middleware composition compared to legacy `telegraf`.
* **`@grammyjs/runner` (`^2.0.0`):** High-throughput concurrent update listener for production workloads.
* **`@grammyjs/hydrate` (`^1.4.0`):** Extends Context with fluent in-place editing capabilities (`ctx.editMessageText`, `ctx.editMessageReplyMarkup`).

### 2.3 Database & State Persistence
* **PocketBase (`v0.23.0+`):** Single Go binary bundling SQLite in WAL (Write-Ahead Logging) mode, real-time event subscriptions, built-in admin UI, and embedded JSVM for native hooks.
* **`pocketbase` (`^0.21.5`):** Official JavaScript client SDK for Node.js, providing reactive subscriptions and typed query builders.

### 2.4 AI Gateway & LLM Orchestration
* **OmniRoute Gateway (`v3.8.50+`):** Self-hosted AI Gateway running locally at `http://localhost:20128/v1`.
  - Exposes standard OpenAI-compatible API.
  - Native quota-aware fallback and Combo routing.
  - Providers configured in OmniRoute:
    - Primary: Groq Cloud (`llama-3.3-70b-versatile` & `whisper-large-v3`).
    - Fallback & Vision: Google AI Studio (`gemini-2.0-flash-001`, `gemini-1.5-flash`).
* **`openai` (`^4.67.0`):** Official OpenAI SDK pointed to OmniRoute base URL (`baseURL: 'http://localhost:20128/v1'`) for standardized Tool Calling and chat completions.
* **`zod` (`^3.23.8`):** Runtime schema validation for ReAct tool arguments, webhook payloads, and environment variables.
* **`zod-to-json-schema` (`^3.23.5`):** Dynamically transforms Zod schemas into JSON Schema for OpenAI function calling definitions.

### 2.5 Security, Cryptography & Network Utilities
* **Node.js Native `crypto`:** AES-256-GCM encryption/decryption of merchant fintech credentials.
* **`jose` (`^5.9.0`):** Lightweight, zero-dependency JWT signing and verification for ephemeral onboarding links.
* **`axios` (`^1.7.7`):** HTTP client for PayClip API calls requiring custom headers and basic authentication.

---

## 3. Production Infrastructure & Operations

### 3.1 Network Ingress & Security
* **Cloudflare Tunnels (`cloudflared`):**
  - Connects the internal backend directly to the Cloudflare edge network.
  - Zero open inbound ports (80/443 closed on host firewall).
  - Handles SSL termination, DDoS protection, and public webhook routing for PayClip and ephemeral onboarding forms.

### 3.2 Database Replication & Disaster Recovery
* **Litestream (`v0.3.13+`):**
  - Runs alongside PocketBase as a background daemon.
  - Monitors SQLite's Write-Ahead Log (`data.db-wal`) and streams incremental transactions in real time (<1s latency) to an S3-compatible bucket (e.g., Cloudflare R2 or AWS S3).
  - Recovery time objective (RTO): < 30 seconds.
  - Recovery point objective (RPO): < 1 second.

### 3.3 Process Management
* **`pm2` (or systemd):**
  - Clustering / Process supervisor ensuring zero-downtime auto-restarts upon uncaught exceptions.
  - Separate managed processes:
    1. `commerce-core`: Node.js WhatsApp & Telegram orchestration engine.
    2. `pocketbase`: PocketBase binary (`./pocketbase serve --http=127.0.0.1:8090`).
    3. `omniroute`: OmniRoute gateway daemon (`omniroute start`).
    4. `litestream`: Replication daemon (`litestream replicate`).

---

## 4. Hardware Sizing & Capacity Estimations

### Single VPS Profile (Scales 50 to 100 Active Merchants)
* **CPU:** 4 vCPUs (Modern x86_64 or ARM64)
* **RAM:** 8 GB DDR4 / DDR5
* **Storage:** 80 GB NVMe SSD (High IOPS for SQLite WAL)
* **Network:** 1 Gbps unmetered

### Memory Budget Breakdown (100 Active Tenants)
```
┌─────────────────────────────────────────────────────────────┐
│ Component                                     Estimated RAM │
├─────────────────────────────────────────────────────────────┤
│ OS & System Daemons (Ubuntu 22.04 LTS)               ~500 MB │
│ PocketBase (v0.23+ SQLite WAL Cache)                 ~300 MB │
│ OmniRoute AI Gateway (Next.js/Node runtime)          ~350 MB │
│ Cloudflare Tunnel & Litestream Daemons               ~150 MB │
│ Node.js Application Base Runtime                     ~200 MB │
│ WhatsApp Sockets Pool (100 sessions @ 40MB avg)    ~4,000 MB │
│ 30s Debounce Message Buffers & ReAct Active Tasks    ~500 MB │
├─────────────────────────────────────────────────────────────┤
│ TOTAL ESTIMATED MEMORY FOOTPRINT:                  ~6,000 MB │
│ HEADROOM / BUFFER:                                 ~2,000 MB │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Environment Variables Contract (`.env.example`)

```bash
# ==============================================================================
# SERVER CONFIGURATION
# ==============================================================================
NODE_ENV=production
PORT=3000
HOST=127.0.0.1
PUBLIC_URL=https://api.yourcommercebot.com

# ==============================================================================
# SECURITY & ENCRYPTION
# ==============================================================================
# 32-byte hex key for AES-256-GCM credential encryption
MASTER_ENCRYPTION_KEY=0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
# Ephemeral onboarding JWT signing secret
JWT_SIGNING_SECRET=super_secret_ephemeral_link_token_32_bytes_min

# ==============================================================================
# POCKETBASE CONFIGURATION
# ==============================================================================
POCKETBASE_URL=http://127.0.0.1:8090
POCKETBASE_ADMIN_EMAIL=admin@yourcommercebot.com
POCKETBASE_ADMIN_PASSWORD=change_this_in_production_secure_pass

# ==============================================================================
# OMNIROUTE AI GATEWAY
# ==============================================================================
OMNIROUTE_BASE_URL=http://localhost:20128/v1
OMNIROUTE_API_KEY=omniroute_local_api_key_from_dashboard
DEFAULT_REACT_MODEL=groq/llama-3.3-70b-versatile
FALLBACK_REACT_MODEL=google/gemini-2.0-flash-001
MULTIMODAL_VISION_MODEL=google/gemini-2.0-flash-001
AUDIO_TRANSCRIPTION_MODEL=groq/whisper-large-v3

# ==============================================================================
# TELEGRAM MASTER BOT
# ==============================================================================
TELEGRAM_BOT_TOKEN=123456789:ABCdefGHIjklMNOpqrsTUVwxyz123456789
TELEGRAM_BOT_USERNAME=LocalCommerceMasterBot

# ==============================================================================
# DEBOUNCE BUFFER TUNING
# ==============================================================================
DEBOUNCE_DELAY_MS=30000
MAX_AUDIO_DURATION_SEC=90
```
