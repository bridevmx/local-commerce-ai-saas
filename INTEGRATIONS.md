# External Systems & Integrations Specification
## PayClip, Telegram Bot API, and OmniRoute Gateway

**Document Version:** 1.0.0  
**Target Audience:** Integration Engineers, Backend Developers, Security Auditors  

---

## 1. PayClip Fintech Core Integration

PayClip operations are divided across three separate domain environments, each requiring distinct authentication mechanisms, headers, and operational safeguards.

### 1.1 Domain Architecture & Credentials Matrix

| Domain Endpoint | Authentication Scheme | Target Capabilities |
| :--- | :--- | :--- |
| `https://api.payclip.com` | `Basic base64(api_key:api_secret)` | Checkout v2 generation, status verification, refunds. |
| `https://api.payclip.io` | `Bearer <CLIP_TOKEN>` (JWT) | Merchant identity, Point-of-Sale (F2F) catalog items and stock. |
| `https://api-gw.payclip.com` | `Bearer <CLIP_TOKEN>` + Vendor Header | Counter cash transactions (TPV emulation). |

---

### 1.2 Checkout v2 Protocol (`https://api.payclip.com`)

#### A. Generate Hosted Payment Link (`POST /v2/checkout`)
* **Headers:**
  - `Authorization: Basic <base64(api_key:api_secret)>`
  - `Content-Type: application/json`
* **Request Payload:**
  ```json
  {
    "amount": 350.00,
    "currency": "MXN",
    "purchase_description": "Pastel Tres Leches - Orden #ord_8921",
    "redirection_url": {
      "success": "https://api.yourcommercebot.com/checkout/success",
      "error": "https://api.yourcommercebot.com/checkout/error",
      "default": "https://api.yourcommercebot.com/checkout/status"
    },
    "metadata": {
      "tenant_id": "ten_9876543210abcde",
      "order_id": "ord_8921",
      "customer_phone": "+5215512345678"
    }
  }
  ```
* **Response Handling:**
  - Save `id` to `orders.clip_checkout_id`.
  - Save `payment_request_url` to `orders.clip_payment_url` and deliver it to the customer via WhatsApp.

---

### 1.3 Critical Rule: Catalog Stock Preservation (`GET-before-PATCH`)

#### The PayClip Catalog Stock Bug
When calling `PATCH https://api.payclip.io/f2f/catalog/products/:id`, if the `stock` property is omitted from the JSON body, PayClip resets the product's inventory stock to `null`, completely corrupting the merchant's POS state.

#### Mandatory Architectural Safeguard:
Every catalog update **must** execute a two-step transaction:

```javascript
// Step 1: Fetch current product state
const currentProduct = await axios.get(
  `https://api.payclip.io/f2f/catalog/products/${productId}`,
  { headers: { Authorization: `Bearer ${clipBearerToken}` } }
);

const currentStock = currentProduct.data.stock ?? null;

// Step 2: Inject current stock into the PATCH payload
await axios.patch(
  `https://api.payclip.io/f2f/catalog/products/${productId}`,
  {
    name: updatedName ?? currentProduct.data.name,
    price: updatedPrice ?? currentProduct.data.price,
    description: updatedDescription ?? currentProduct.data.description,
    stock: currentStock // CRITICAL: Preserves stock from being cleared to null
  },
  { headers: { Authorization: `Bearer ${clipBearerToken}` } }
);
```

---

### 1.4 Critical Rule: Zero-Trust Webhook Readback Architecture

PayClip Checkout v2 webhooks do not provide HMAC cryptographic signatures. Accepting webhook payloads at face value introduces severe fraud vulnerabilities (e.g., spoofed `PAID` updates).

#### Production Webhook Verification Sequence:
```
[PAYCLIP EDGE] ─────────── POST /webhook/clip/checkout ──────────► [BACKEND WEBHOOK]
                                                                        │
                                                                        │ Fast-ACK (<1s)
                                                                        ▼
                                                                  [HTTP 200 OK]
                                                                        │
                                                                        ▼
                                                        [ASYNC BACKGROUND VERIFICATION]
                                                                        │
                                                GET https://api.payclip.com/v2/checkout/{id}
                                                Header: Basic base64(api_key:api_secret)
                                                                        │
                                                                        ▼
                                                        [VERIFY PAYLOAD INTEGRITY]
                                                        - status === "CHECKOUT_COMPLETED"
                                                        - amount === orders.total_amount
                                                        - currency === "MXN"
                                                                        │
                                       ┌────────────────────────────────┴───────────────────────────────┐
                                       ▼ (MATCH)                                                        ▼ (MISMATCH / UNCONFIRMED)
                             [MARK ORDER: PAID]                                               [FLAG FRAUD ALERT]
                             Update PocketBase order.status                                   Dispatch critical alert to
                             Mutate card in Telegram #Ventas                                  Telegram #Ventas topic
                             Send receipt on WhatsApp
```

---

### 1.5 Cash Counter TPV Emulation (`https://api-gw.payclip.com`)

When registering in-store cash payments via Telegram `#Ventas-y-Caja`, PayClip requires 14 mandatory hardware and telemetry fields.

* **Endpoint:** `POST https://api-gw.payclip.com/cash/transaction`
* **Mandatory Headers:**
  - `Authorization: Bearer <CLIP_TOKEN>`
  - `Accept: application/vnd.com.payclip.v2+json`
  - `Content-Type: application/json`
* **Telemetry Payload:**
  ```json
  {
    "amount": 350.00,
    "currency": "MXN",
    "received_amount": 400.00,
    "change_amount": 50.00,
    "external_reference": "ord_8921",
    "device_info": {
      "device_id": "SERVER-NODE-POS-01",
      "model": "Clip-Agent-Gateway",
      "os": "Linux",
      "os_version": "Ubuntu-22.04",
      "app_version": "2.4.0",
      "battery_level": 100,
      "is_charging": true,
      "network_type": "WIFI",
      "latitude": 19.432608,
      "longitude": -99.133209,
      "accuracy": 5.0,
      "timestamp": "2026-09-18T14:30:00.000Z",
      "is_mock_location": false,
      "serial_number": "EMULATED-POS-SYS"
    }
  }
  ```

---

## 2. Telegram Bot Operations CRM (Grammy Framework)

Merchants manage business operations, resolve support tickets, and review sales directly inside their private Telegram Supergroup.

### 2.1 Supergroup Forum Topics Topology

Every merchant Supergroup must have **Forum Topics (Temas)** enabled:

| Topic Name | Telegram Thread ID | Operational Scope |
| :--- | :--- | :--- |
| **`#Dudas-Clientes`** | `tenants.topic_support_id` | Human support escalations, client questions, and quote-based replies. |
| **`#Mi-Catálogo`** | `tenants.topic_catalog_id` | Voice/text catalog administration (pricing, stock adjustments). |
| **`#Ventas-y-Caja`** | `tenants.topic_sales_id` | Live interactive order cards with inline keyboards and payment status. |

---

### 2.2 Human Handoff via Native `reply_to_message`

When a merchant replies to a support alert in `#Dudas-Clientes`, Grammy resolves the target customer through quote referencing:

```javascript
bot.on('message:text', async (ctx) => {
  // Verify message originated from a Supergroup and is within the Support topic
  const tenant = await getTenantByTelegramChatId(ctx.chat.id);
  if (!tenant || ctx.message.message_thread_id !== tenant.topic_support_id) return;

  const replyTo = ctx.message.reply_to_message;
  if (!replyTo) return; // Ignore regular conversational chatter not quoting an alert

  // Find the open ticket linked to the quoted Telegram message ID
  const ticket = await pb.collection('tickets').getFirstListItem(
    `tenant = "${tenant.id}" && telegram_message_id = ${replyTo.message_id} && status != "RESOLVED"`
  );

  if (!ticket) return;

  const customer = await pb.collection('customers').getOne(ticket.customer);

  // Dispatch human merchant reply to WhatsApp
  await waManager.sendMessage(tenant.id, customer.phone_number, ctx.message.text);

  // Record audit trail in messages collection
  await pb.collection('messages').create({
    session: ticket.session,
    tenant: tenant.id,
    customer: customer.id,
    direction: 'OUTBOUND',
    sender_type: 'HUMAN_MERCHANT',
    content_type: 'TEXT',
    text_content: ctx.message.text,
    telegram_msg_id: ctx.message.message_id
  });

  await ctx.reply("Respuesta enviada al WhatsApp del cliente.", {
    reply_to_message_id: ctx.message.message_id
  });
});
```

---

### 2.3 Interactive Order Cards with In-Place Message Mutation

In `#Ventas-y-Caja`, order states are represented by a single live message that mutates in place via `ctx.editMessageText` and `ctx.editMessageReplyMarkup`:

#### Card Presentation Layout:
```
NUEVA ORDEN #ord_8921
━━━━━━━━━━━━━━━━━━━━
Cliente: Mariana Gómez (+52 1 55 1234 5678)
Estado: PENDIENTE DE PAGO

Items:
• 1x Pastel Tres Leches ($350.00)
• 1x Velas Doradas ($35.00)
Total: $385.00 MXN

Link de Pago: https://payclip.com/co/chk_abc123
━━━━━━━━━━━━━━━━━━━━
[ 💵 Registrar Efectivo ]  [ ❌ Cancelar ]
```

#### In-Place Update Handler:
```javascript
bot.callbackQuery(/^pay_cash:(\w+)$/, async (ctx) => {
  const orderId = ctx.match[1];
  const order = await pb.collection('orders').getOne(orderId);

  // Execute PayClip Cash Transaction
  await clipClient.recordCashTransaction(order.tenant, order.total_amount, orderId);

  // Update PocketBase order record
  await pb.collection('orders').update(orderId, {
    status: 'PAID',
    payment_method: 'CASH_COUNTER',
    paid_at: new Date().toISOString()
  });

  // Mutate message in place to eliminate chat clutter
  await ctx.editMessageText(
    `✅ ORDEN PAGADA EN EFECTIVO #${orderId}\n━━━━━━━━━━━━━━━━━━━━\nTotal: $${order.total_amount} MXN\nRegistrado por: @${ctx.from.username}`,
    { reply_markup: null }
  );

  await ctx.answerCallbackQuery({ text: "Pago registrado exitosamente." });
});
```

---

## 3. OmniRoute AI Gateway API Specification

The backend connects to OmniRoute (`http://localhost:20128/v1`) using the standard OpenAI Node.js SDK:

### 3.1 Gateway Initialization & Combo Configuration
```javascript
import OpenAI from 'openai';

export const omniroute = new OpenAI({
  apiKey: process.env.OMNIROUTE_API_KEY,
  baseURL: process.env.OMNIROUTE_BASE_URL || 'http://localhost:20128/v1'
});
```

### 3.2 Chat Completions & Tool Calling Protocol
* **Endpoint:** `POST /v1/chat/completions`
* **Model Configuration:** `model: "auto"` or configured Combo:
  - Primary: `groq/llama-3.3-70b-versatile`
  - Fallback: `google/gemini-2.0-flash-001`
* **Request Schema:**
  ```json
  {
    "model": "auto",
    "temperature": 0.2,
    "max_tokens": 1024,
    "messages": [
      { "role": "system", "content": "..." },
      { "role": "user", "content": "..." }
    ],
    "tools": [
      {
        "type": "function",
        "function": {
          "name": "search_catalog",
          "description": "Search Clip catalog for items and stock",
          "parameters": {
            "type": "object",
            "properties": {
              "query": { "type": "string" }
            },
            "required": ["query"]
          }
        }
      }
    ],
    "tool_choice": "auto"
  }
  ```

---

### 3.3 Audio Transcriptions via OmniRoute (`Groq Whisper`)
Voice notes collected during the 30-second debounce window are dispatched to:
* **Endpoint:** `POST /v1/audio/transcriptions`
* **Parameters:**
  - `file`: Audio stream or buffer (`audio/ogg; codecs=opus`)
  - `model`: `groq/whisper-large-v3`
  - `language`: `es`
  - `response_format`: `json`

---

### 3.4 Multimodal Vision Payload (Customer WhatsApp Photos)
When customers send images (e.g., cake customization references), they are routed to Gemini 2.0 Flash through OmniRoute:
```javascript
const response = await omniroute.chat.completions.create({
  model: 'google/gemini-2.0-flash-001',
  messages: [
    {
      role: 'user',
      content: [
        { type: 'text', text: 'Analiza esta imagen y describe qué producto busca el cliente.' },
        {
          type: 'image_url',
          image_url: { url: `data:image/jpeg;base64,${base64ImageBuffer}` }
        }
      ]
    }
  ]
});
```
