# Engineering Implementation Plan
## Modular Repository Structure, Step-by-Step Build Order, and Testing Strategy

**Document Version:** 1.0.0  
**Target Audience:** Lead Software Engineers, Full-Stack Developers, QA Automation Engineers  

---

## 1. Repository Directory Structure

The project follows a modular, domain-driven structure optimized for Node.js (ESM) and PocketBase:

```
local-commerce-ai-saas/
├── .env.example
├── .gitignore
├── package.json
├── ARCHITECTURE.md
├── DATABASE_SCHEMA.md
├── INTEGRATIONS.md
├── TECH_STACK.md
├── TOOL_REGISTRY_AND_PRESETS.md
├── ROADMAP.md
├── IMPLEMENTATION_PLAN.md
│
├── pb_data/                       # PocketBase local database (gitignored)
├── pb_hooks/                      # Native PocketBase JSVM database hooks
│   ├── 01_system_bootstrap.pb.js
│   ├── 02_order_rules.pb.js
│   └── 03_security_guards.pb.js
├── pb_migrations/                 # Declarative JavaScript migrations
│   ├── 1700000001_initial_schema.js
│   └── 1700000002_indexes_and_rules.js
│
├── storage/
│   └── sessions/                  # Baileys encrypted local auth folders (gitignored)
│
├── src/
│   ├── index.js                   # Main application entry point & process supervisor
│   ├── config/
│   │   ├── env.js                 # Zod validated environment variables
│   │   └── constants.js           # Timeouts, vertical enums, presets
│   │
│   ├── db/
│   │   ├── pocketbase.js          # PocketBase SDK client instance
│   │   └── repositories/
│   │       ├── tenantRepo.js      # Tenant and rules queries
│   │       ├── customerRepo.js    # Customer profiles and EAV traits
│   │       ├── sessionRepo.js     # Session state and tickets
│   │       └── orderRepo.js       # Orders and line items
│   │
│   ├── whatsapp/
│   │   ├── sessionManager.js      # Baileys socket pool & auth lifecycle
│   │   ├── presenceEmulator.js    # Organic typing & read status simulator
│   │   └── messageHandler.js      # Incoming socket event router
│   │
│   ├── buffer/
│   │   └── messageAggregator.js   # In-memory 30-second sliding debounce buffer
│   │
│   ├── agent/
│   │   ├── reactEngine.js         # ReAct loop (MAX_ITERATIONS = 5)
│   │   ├── promptCompiler.js      # 5-layer runtime system prompt builder
│   │   ├── tools/
│   │   │   ├── index.js           # Tool registry and Zod converter
│   │   │   ├── catalogTools.js    # search_catalog, update_catalog_item
│   │   │   ├── paymentTools.js    # create_payment_link, check_order_status
│   │   │   └── supportTools.js    # escalate_to_human, get_business_info
│   │   └── presets/
│   │       └── businessPresets.js # FOOD_RETAIL, SERVICE_APPOINTMENTS, etc.
│   │
│   ├── ai/
│   │   ├── omnirouteClient.js     # OpenAI-compatible client for OmniRoute
│   │   ├── audioTranscriber.js    # Groq Whisper speech-to-text
│   │   └── visionService.js       # Gemini 2.0 Flash multimodal image processor
│   │
│   ├── telegram/
│   │   ├── bot.js                 # Grammy bot initialization & runner
│   │   ├── topics/
│   │   │   ├── supportTopic.js    # #Dudas-Clientes & reply_to_message bridge
│   │   │   ├── catalogTopic.js    # #Mi-Catálogo voice/text admin
│   │   │   └── salesTopic.js      # #Ventas-y-Caja order cards & mutations
│   │   └── onboardingWizard.js    # Merchant setup and topic detection
│   │
│   ├── clip/
│   │   ├── clipClient.js          # Unified PayClip API client (3 domains)
│   │   ├── catalogService.js      # GET-before-PATCH stock preservation logic
│   │   ├── checkoutService.js     # Checkout v2 payment links
│   │   └── webhookHandler.js      # Zero-Trust Webhook Readback verification
│   │
│   ├── worker/
│   │   └── traitWorker.js         # Fire-and-forget EAV fact extraction worker
│   │
│   └── security/
│       ├── crypto.js              # AES-256-GCM encryption/decryption
│       └── ephemeralLink.js       # Signed 15-minute onboarding JWT links
│
└── tests/
    ├── mocks/
    │   ├── mockClipServer.js      # PayClip API mock (Checkout, Catalog, Cash)
    │   └── mockOmniroute.js       # OmniRoute LLM mock
    ├── unit/
    │   ├── debounceBuffer.test.js # Test 30s sliding window
    │   ├── promptCompiler.test.js # Test 5-layer prompt assembly
    │   └── stockSafeguard.test.js # Test GET-before-PATCH rule
    └── integration/
        ├── reactLoop.test.js      # Test ReAct stop conditions
        └── webhookReadback.test.js# Test Zero-Trust payment verification
```

---

## 2. Step-by-Step Build Sequence (Engineering Tasks)

```
TASK 1: Environment & PocketBase Initialization
  ├── Install Node.js v20+, pnpm, and download PocketBase binary.
  ├── Create pb_migrations/ for all Zero-JSON collections and indexes.
  └── Setup AES-256-GCM crypto utility in src/security/crypto.js.

TASK 2: WhatsApp Baileys Engine & Organic Emulation
  ├── Implement src/whatsapp/sessionManager.js with dynamic socket map.
  ├── Build src/whatsapp/presenceEmulator.js (read delay + typing duration).
  └── Store credentials in ./storage/sessions/{tenant_id}/.

TASK 3: 30-Second In-Memory Debounce Buffer
  ├── Create src/buffer/messageAggregator.js.
  ├── Support sliding window: reset timer to 30s on each incoming event.
  └── On flush: parallel audio transcription + text concatenation.

TASK 4: OmniRoute Client & Layered Prompt Compiler
  ├── Setup src/ai/omnirouteClient.js pointing to http://localhost:20128/v1.
  ├── Implement src/ai/audioTranscriber.js for Groq Whisper.
  └── Code src/agent/promptCompiler.js combining Layers 1 through 5.

TASK 5: ReAct Agent Core & Tool Registry
  ├── Implement src/agent/tools/ with Zod schemas and handlers.
  ├── Build src/agent/reactEngine.js with MAX_ITERATIONS = 5.
  └── Handle stop conditions: FINAL_ANSWER, ESCALATE, MAX_ITERATIONS.

TASK 6: Telegram CRM & Topics Bridge
  ├── Initialize Grammy bot in src/telegram/bot.js with @grammyjs/hydrate.
  ├── Implement #Dudas-Clientes quote routing (reply_to_message -> WhatsApp).
  └── Implement #Ventas-y-Caja interactive order cards with in-place edit.

TASK 7: PayClip Core & Fintech Hardening
  ├── Implement src/clip/clipClient.js with 3-domain authentication.
  ├── Implement GET-before-PATCH stock preservation in catalogService.js.
  ├── Build Zero-Trust Webhook Readback (Fast-ACK + GET /v2/checkout/{id}).
  └── Implement cash counter POS emulation with 14-field telemetry.

TASK 8: Background Async Fact Extraction (EAV Memory)
  ├── Build src/worker/traitWorker.js using setImmediate().
  └── Call OmniRoute to extract traits with strict enum categories.

TASK 9: Merchant Onboarding & Ephemeral Web Forms
  ├── Build Telegram bot onboarding wizard to detect forum topics.
  └── Deploy ephemeral 15-minute web form for secure Clip key submission.

TASK 10: Production Hardening, Tunnels & Litestream
  ├── Configure Cloudflare Tunnels (cloudflared) for zero-port ingress.
  └── Configure Litestream daemon for continuous SQLite WAL streaming.
```

---

## 3. Concrete Code Blueprints for Critical Modules

### 3.1 30-Second In-Memory Debounce Buffer (`src/buffer/messageAggregator.js`)
```javascript
export class MessageAggregator {
  constructor(debounceMs = 30000, onFlushCallback) {
    this.debounceMs = debounceMs;
    this.onFlush = onFlushCallback;
    this.buffers = new Map(); // key: `${tenantId}:${customerPhone}`
  }

  enqueue(tenantId, customerPhone, messageItem) {
    const key = `${tenantId}:${customerPhone}`;

    if (!this.buffers.has(key)) {
      this.buffers.set(key, {
        tenantId,
        customerPhone,
        items: [],
        timer: null
      });
    }

    const entry = this.buffers.get(key);

    // Cancel prior rolling timer
    if (entry.timer) {
      clearTimeout(entry.timer);
    }

    // Push item (text, audio buffer, or image)
    entry.items.push(messageItem);

    // Arm new 30-second timer
    entry.timer = setTimeout(async () => {
      this.buffers.delete(key);
      try {
        await this.onFlush(tenantId, customerPhone, entry.items);
      } catch (err) {
        console.error(`Error processing batch for ${key}:`, err);
      }
    }, this.debounceMs);
  }
}
```

---

### 3.2 ReAct Engine Core (`src/agent/reactEngine.js`)
```javascript
export async function executeReActLoop(tenantId, customerId, sessionId, userTurn) {
  const MAX_ITERATIONS = 5;
  let iterations = 0;

  // Compile 5-layer prompt
  const systemPrompt = await buildTenantSystemPrompt(tenantId, customerId);
  const activeTools = await getActiveTenantTools(tenantId);

  const conversationHistory = [
    { role: 'system', content: systemPrompt },
    { role: 'user', content: userTurn }
  ];

  while (iterations < MAX_ITERATIONS) {
    iterations++;

    const response = await omniroute.chat.completions.create({
      model: process.env.DEFAULT_REACT_MODEL || 'groq/llama-3.3-70b-versatile',
      messages: conversationHistory,
      tools: activeTools,
      tool_choice: 'auto',
      temperature: 0.2
    });

    const choice = response.choices[0].message;
    conversationHistory.push(choice);

    // STOP CONDITION 1: FINAL_ANSWER (No tools requested)
    if (!choice.tool_calls || choice.tool_calls.length === 0) {
      return {
        type: 'FINAL_ANSWER',
        replyText: choice.content
      };
    }

    // Process Tool Calls
    for (const toolCall of choice.tool_calls) {
      const toolName = toolCall.function.name;
      const toolArgs = JSON.parse(toolCall.function.arguments);

      // STOP CONDITION 2: ESCALATE TO HUMAN
      if (toolName === 'escalate_to_human') {
        await handleEscalation(tenantId, customerId, sessionId, toolArgs);
        return {
          type: 'ESCALATE',
          reason: toolArgs.reason
        };
      }

      // Execute tool
      const observation = await executeTool(toolName, toolArgs, { tenantId, customerId, sessionId });

      conversationHistory.push({
        role: 'tool',
        tool_call_id: toolCall.id,
        content: JSON.stringify(observation)
      });
    }
  }

  // STOP CONDITION 3: MAX ITERATIONS EXCEEDED (FAILSAFE)
  await handleFailsafeEscalation(tenantId, customerId, sessionId, "Max ReAct iterations reached without conclusion.");
  return {
    type: 'MAX_ITERATIONS',
    replyText: "Estoy transfiriendo tu mensaje con uno de nuestros asesores para atenderte personalmente."
  };
}
```

---

### 3.3 PayClip Catalog Safe Update (`src/clip/catalogService.js`)
```javascript
export async function updateProductSafe(tenantId, { productId, name, price, description }) {
  const credentials = await getTenantCredentials(tenantId);
  const token = credentials.clipBearerJwt;

  // STEP 1: GET current product state to protect stock
  const getRes = await axios.get(
    `https://api.payclip.io/f2f/catalog/products/${productId}`,
    { headers: { Authorization: `Bearer ${token}` } }
  );

  const existing = getRes.data;
  const preservedStock = existing.stock ?? null;

  // STEP 2: PATCH with explicit stock preserved
  const patchPayload = {
    name: name ?? existing.name,
    price: price ?? existing.price,
    description: description ?? existing.description,
    stock: preservedStock // MANDATORY: Prevents stock from being cleared to null
  };

  const patchRes = await axios.patch(
    `https://api.payclip.io/f2f/catalog/products/${productId}`,
    patchPayload,
    { headers: { Authorization: `Bearer ${token}` } }
  );

  return patchRes.data;
}
```

---

## 4. Testing Strategy & Quality Assurance

### 4.1 Unit Testing Suite (Vitest / Node Test Runner)
* **`tests/unit/debounceBuffer.test.js`:**
  - Verify that rapid sequential additions within 30 seconds delay the flush event.
  - Verify that the buffer automatically unreferences completed timers without memory leaks.
* **`tests/unit/stockSafeguard.test.js`:**
  - Mock `api.payclip.io/f2f/catalog/products/:id` and assert that `stock` is always present in the outgoing `PATCH` request.
* **`tests/unit/promptCompiler.test.js`:**
  - Validate that prompt output correctly incorporates Layer 1 (Guardrails), Layer 3 (Tenant), Layer 4 (Rules), and Layer 5 (Customer traits).

### 4.2 Integration & Simulation Tests
* **`tests/integration/webhookReadback.test.js`:**
  - Simulate incoming fake webhook with status `PAID` but modify order amount.
  - Verify that the API readback detects the discrepancy and flags a fraud alert instead of marking the order as paid.
* **`tests/integration/reactLoop.test.js`:**
  - Inject tool calls that fail to terminate and verify that the engine strictly aborts on iteration 5 and escalates to human.

---

## 5. Operations & Runbook Commands

```bash
# 1. Start PocketBase in background
./pocketbase serve --http=127.0.0.1:8090

# 2. Run schema migrations
npm run db:migrate

# 3. Start OmniRoute AI Gateway
omniroute start

# 4. Start Node.js Application Core
npm run start:prod

# 5. Run test suite
npm test
```
