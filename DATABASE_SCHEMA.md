# Database Schema Specification
## PocketBase Relational Model (Zero-JSON Architecture)

**Document Version:** 1.0.0  
**Target Engine:** PocketBase v0.23+ (SQLite in Write-Ahead Logging mode)  
**Architectural Rule:** Strict Zero-JSON Policy. All attributes, state, configuration, and entity relations are normalized into relational columns with explicit types, foreign keys, and indexes.

---

## 1. Relational Entity Relationship Overview

```
                      ┌───────────────┐
                      │    tenants    │
                      └───────┬───────┘
          ┌───────────────────┼───────────────────┬───────────────────┐
          │ 1:1               │ 1:N               │ 1:N               │ 1:N
          ▼                   ▼                   ▼                   ▼
┌──────────────────┐ ┌──────────────────┐ ┌───────────────┐ ┌──────────────────┐
│tenant_credentials│ │   tenant_rules   │ │ tenant_tools  │ │    customers     │
└──────────────────┘ └──────────────────┘ └───────────────┘ └────────┬─────────┘
                                                                     │
                                  ┌──────────────────────────────────┤
                                  │ 1:N                              │ 1:N
                                  ▼                                  ▼
                        ┌──────────────────┐               ┌──────────────────┐
                        │ customer_traits  │               │     sessions     │
                        │  (EAV Fact Store)│               └────────┬─────────┘
                        └──────────────────┘                        │
                                      ┌─────────────────────────────┤
                                      │ 1:N                         │ 1:N
                                      ▼                             ▼
                            ┌──────────────────┐          ┌──────────────────┐
                            │     tickets      │          │     messages     │
                            └──────────────────┘          └──────────────────┘
                                      │
                                      ▼ 1:N
                            ┌──────────────────┐
                            │      orders      │
                            └────────┬─────────┘
                                     │ 1:N
                                     ▼
                            ┌──────────────────┐
                            │   order_items    │
                            └──────────────────┘
```

---

## 2. Collection Definitions

### 2.1 Collection: `tenants`
The root multi-tenant merchant identity.

| Field Name | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `TEXT` (PB standard) | PRIMARY KEY, 15 chars | Unique Tenant ID. |
| `slug` | `TEXT` | UNIQUE, NOT NULL, INDEXED | URL-friendly merchant identifier. |
| `name` | `TEXT` | NOT NULL | Commercial business name (e.g., "Pastelería Dulce Sabor"). |
| `vertical` | `SELECT` | NOT NULL, INDEXED | `FOOD_RETAIL`, `SERVICE_APPOINTMENTS`, `RETAIL_CATALOG`, `SUPPORT_LEAD`. |
| `tone` | `SELECT` | NOT NULL | `FRIENDLY_CASUAL`, `PROFESSIONAL_FORMAL`, `CONCISE_DIRECT`. |
| `phone_number` | `TEXT` | NOT NULL, INDEXED | Registered WhatsApp phone number in E.164 format. |
| `address` | `TEXT` | NULLABLE | Physical shop address for pickup and customer reference. |
| `business_hours`| `TEXT` | NOT NULL | Natural language hours (e.g., "Mon-Sat 09:00-20:00, Sun 10:00-14:00"). |
| `timezone` | `TEXT` | NOT NULL, DEFAULT: 'America/Mexico_City' | IANA timezone identifier. |
| `telegram_chat_id` | `TEXT` | UNIQUE, NULLABLE, INDEXED | Supergroup ID (e.g., `-1002345678901`). |
| `topic_support_id` | `NUMBER`| NULLABLE | Forum Topic ID for `#Dudas-Clientes`. |
| `topic_catalog_id` | `NUMBER`| NULLABLE | Forum Topic ID for `#Mi-Catálogo`. |
| `topic_sales_id` | `NUMBER`| NULLABLE | Forum Topic ID for `#Ventas-y-Caja`. |
| `status` | `SELECT` | NOT NULL, INDEXED | `ONBOARDING`, `ACTIVE`, `SUSPENDED`. |
| `custom_instructions`| `TEXT`| NULLABLE | Optional general instructions appended to Layer 3 of prompt. |
| `created` | `DATE` | AUTO | Record creation timestamp. |
| `updated` | `DATE` | AUTO | Record modification timestamp. |

---

### 2.2 Collection: `tenant_credentials`
High-security isolation for fintech and infrastructure credentials. All secrets are stored encrypted with AES-256-GCM.

| Field Name | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `TEXT` | PRIMARY KEY | Unique ID. |
| `tenant` | `RELATION` | 1:1, CASCADE DELETE, UNIQUE | Foreign key to `tenants`. |
| `clip_api_key_enc` | `TEXT` | NULLABLE | Encrypted PayClip API Key (Basic Auth). |
| `clip_api_secret_enc`| `TEXT` | NULLABLE | Encrypted PayClip Secret (Basic Auth). |
| `clip_bearer_jwt_enc`| `TEXT` | NULLABLE | Encrypted PayClip Bearer JWT (`CLIP_TOKEN` for catalog). |
| `encryption_iv` | `TEXT` | NOT NULL | 12-byte initialization vector (Hex encoded). |
| `auth_tag` | `TEXT` | NOT NULL | 16-byte GCM authentication tag (Hex encoded). |
| `baileys_auth_blob` | `FILE` | NULLABLE | Encrypted zip backup of Baileys `creds.json`. |
| `last_rotated_at` | `DATE` | NOT NULL | Timestamp of last credential rotation. |

---

### 2.3 Collection: `tenant_rules`
Atomic business rules and policies (Layer 4 of Prompt Compiler). Eliminates freeform JSON blobs.

| Field Name | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `TEXT` | PRIMARY KEY | Unique ID. |
| `tenant` | `RELATION` | CASCADE DELETE, INDEXED | Foreign key to `tenants`. |
| `category` | `SELECT` | NOT NULL, INDEXED | `SHIPPING`, `PAYMENT`, `REFUND`, `BOOKING`, `FAQ`, `CUSTOM`. |
| `rule_key` | `TEXT` | NOT NULL, INDEXED | Programmatic identifier (e.g., `min_order_delivery`). |
| `content` | `TEXT` | NOT NULL | Natural language rule content injected into the prompt. |
| `is_active` | `BOOL` | NOT NULL, DEFAULT: true | Enable/disable flag. |
| `created` | `DATE` | AUTO | Timestamp. |
| `updated` | `DATE` | AUTO | Timestamp. |

*Composite Index:* `[tenant, category, rule_key]` (UNIQUE).

---

### 2.4 Collection: `tenant_tools`
Active capability registry switches per merchant.

| Field Name | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `TEXT` | PRIMARY KEY | Unique ID. |
| `tenant` | `RELATION` | CASCADE DELETE, INDEXED | Foreign key to `tenants`. |
| `tool_name` | `TEXT` | NOT NULL, INDEXED | Identifier matching `TOOL_REGISTRY` (`search_catalog`, etc.). |
| `is_enabled` | `BOOL` | NOT NULL, DEFAULT: true | Tool activation toggle. |
| `priority_order`| `NUMBER` | NOT NULL, DEFAULT: 100 | Execution or ordering priority. |

*Composite Index:* `[tenant, tool_name]` (UNIQUE).

---

### 2.5 Collection: `customers`
End-customer profiles identified by WhatsApp phone number.

| Field Name | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `TEXT` | PRIMARY KEY | Unique ID. |
| `tenant` | `RELATION` | CASCADE DELETE, INDEXED | Foreign key to `tenants`. |
| `phone_number` | `TEXT` | NOT NULL, INDEXED | WhatsApp JID / E.164 phone number. |
| `first_name` | `TEXT` | NULLABLE | Inferred or stated customer name. |
| `first_seen_at`| `DATE` | NOT NULL | Timestamp of initial inbound message. |
| `last_seen_at` | `DATE` | NOT NULL, INDEXED | Timestamp of latest interaction. |
| `is_blocked` | `BOOL` | NOT NULL, DEFAULT: false| Blacklist flag. |

*Composite Index:* `[tenant, phone_number]` (UNIQUE).

---

### 2.6 Collection: `customer_traits` (EAV Fact Store)
Long-term memory store. Allows granular upserts, queries, and automated decay without parsing unstructured text.

| Field Name | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `TEXT` | PRIMARY KEY | Unique ID. |
| `customer` | `RELATION` | CASCADE DELETE, INDEXED | Foreign key to `customers`. |
| `tenant` | `RELATION` | CASCADE DELETE, INDEXED | Foreign key to `tenants`. |
| `category` | `TEXT` | NOT NULL, INDEXED | Controlled by LLM Enum: `PREFERENCE`, `HEALTH_RESTRICTION`, `IMPORTANT_DATE`, `CONTACT_PREFERENCE`, `METRIC_SIZE`. |
| `trait_key` | `TEXT` | NOT NULL, INDEXED | Normalized key (e.g., `allergy_nuts`, `flavor_tres_leches`). |
| `trait_value` | `TEXT` | NOT NULL | Concise value (e.g., `true`, `extra_strawberries`, `1990-05-12`). |
| `confidence` | `NUMBER` | NOT NULL, DEFAULT: 1.0 | Extraction confidence score (0.0 to 1.0). |
| `extracted_from`| `TEXT` | NULLABLE | Brief source snippet or message ID. |
| `created` | `DATE` | AUTO | Timestamp. |
| `updated` | `DATE` | AUTO | Timestamp. |

*Composite Index:* `[customer, trait_key]` (UNIQUE).

---

### 2.7 Collection: `sessions`
Tracks conversational state and human handoff lockouts.

| Field Name | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `TEXT` | PRIMARY KEY | Unique ID. |
| `tenant` | `RELATION` | CASCADE DELETE, INDEXED | Foreign key to `tenants`. |
| `customer` | `RELATION` | CASCADE DELETE, INDEXED | Foreign key to `customers`. |
| `state` | `SELECT` | NOT NULL, INDEXED | `BOT_ACTIVE`, `PAUSED_FOR_HUMAN`, `RESOLVED`. |
| `paused_until` | `DATE` | NULLABLE | Automatic unpause timestamp (defaults to 2 hours if human fails to reply). |
| `active_ticket` | `RELATION` | NULLABLE | Foreign key to active `tickets` record if paused. |
| `last_message_at`| `DATE` | NOT NULL, INDEXED | Timestamp of latest message in session. |

*Composite Index:* `[tenant, customer]` (UNIQUE).

---

### 2.8 Collection: `tickets`
Human escalation tickets dispatched to Telegram `#Dudas-Clientes`.

| Field Name | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `TEXT` | PRIMARY KEY | Unique ID. |
| `tenant` | `RELATION` | CASCADE DELETE, INDEXED | Foreign key to `tenants`. |
| `customer` | `RELATION` | CASCADE DELETE, INDEXED | Foreign key to `customers`. |
| `session` | `RELATION` | CASCADE DELETE | Foreign key to `sessions`. |
| `status` | `SELECT` | NOT NULL, INDEXED | `OPEN`, `IN_PROGRESS`, `RESOLVED`. |
| `reason` | `TEXT` | NOT NULL | Escalation reason extracted by `escalate_to_human`. |
| `telegram_message_id`| `NUMBER`| NOT NULL, INDEXED | Telegram message ID in `#Dudas` for `reply_to_message` routing. |
| `resolved_by` | `TEXT` | NULLABLE | Telegram username or user ID of the merchant staff. |
| `resolved_at` | `DATE` | NULLABLE | Timestamp of ticket resolution. |

---

### 2.9 Collection: `messages`
Immutable conversational audit trail.

| Field Name | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `TEXT` | PRIMARY KEY | Unique ID. |
| `session` | `RELATION` | CASCADE DELETE, INDEXED | Foreign key to `sessions`. |
| `tenant` | `RELATION` | CASCADE DELETE, INDEXED | Foreign key to `tenants`. |
| `customer` | `RELATION` | CASCADE DELETE, INDEXED | Foreign key to `customers`. |
| `direction` | `SELECT` | NOT NULL, INDEXED | `INBOUND`, `OUTBOUND`. |
| `sender_type`| `SELECT` | NOT NULL | `CUSTOMER`, `BOT_AI`, `HUMAN_MERCHANT`. |
| `content_type`| `SELECT` | NOT NULL | `TEXT`, `AUDIO`, `IMAGE`, `DOCUMENT`. |
| `text_content`| `TEXT` | NOT NULL | Full message body or transcription text. |
| `media_url` | `TEXT` | NULLABLE | Local path or Cloudflare R2 URL to audio/image file. |
| `telegram_msg_id`| `NUMBER`| NULLABLE | Linked Telegram message ID if mirrored. |
| `created` | `DATE` | AUTO, INDEXED | Message timestamp. |

---

### 2.10 Collection: `orders`
Financial order tracking synchronized with PayClip and Telegram `#Ventas-y-Caja`.

| Field Name | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `TEXT` | PRIMARY KEY | Unique ID. |
| `tenant` | `RELATION` | CASCADE DELETE, INDEXED | Foreign key to `tenants`. |
| `customer` | `RELATION` | CASCADE DELETE, INDEXED | Foreign key to `customers`. |
| `status` | `SELECT` | NOT NULL, INDEXED | `DRAFT`, `PENDING_PAYMENT`, `PAID`, `FULFILLED`, `CANCELLED`. |
| `total_amount`| `NUMBER`| NOT NULL | Total amount in MXN (e.g., `450.00`). |
| `currency` | `TEXT` | NOT NULL, DEFAULT: 'MXN' | 3-letter currency code. |
| `payment_method`| `SELECT`| NOT NULL | `CLIP_CHECKOUT`, `CASH_COUNTER`. |
| `clip_checkout_id`| `TEXT` | NULLABLE, INDEXED | PayClip Checkout ID from `POST /v2/checkout`. |
| `clip_payment_url`| `TEXT` | NULLABLE | Hosted checkout URL sent to customer. |
| `telegram_card_msg_id`| `NUMBER`| NULLABLE | Message ID of the live card in `#Ventas-y-Caja`. |
| `paid_at` | `DATE` | NULLABLE | Verification timestamp from Webhook Readback. |
| `created` | `DATE` | AUTO | Timestamp. |

---

### 2.11 Collection: `order_items`
Discrete relational line items belonging to an order.

| Field Name | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `TEXT` | PRIMARY KEY | Unique ID. |
| `order` | `RELATION` | CASCADE DELETE, INDEXED | Foreign key to `orders`. |
| `tenant` | `RELATION` | CASCADE DELETE, INDEXED | Foreign key to `tenants`. |
| `product_id` | `TEXT` | NOT NULL | PayClip product ID or catalog SKU. |
| `product_name`| `TEXT` | NOT NULL | Snapshot of product title at time of purchase. |
| `unit_price` | `NUMBER` | NOT NULL | Price per unit in MXN. |
| `quantity` | `NUMBER` | NOT NULL | Number of units purchased. |
| `subtotal` | `NUMBER` | NOT NULL | `unit_price * quantity`. |
| `notes` | `TEXT` | NULLABLE | Customization notes (e.g., "Add Happy Birthday name"). |

---

## 3. PocketBase API Rules (Security & Isolation)

By default, PocketBase collections are completely locked down from public access. The application backend authenticates as a **Superuser** via the PocketBase SDK.

| Collection | List / Search Rule | View Rule | Create Rule | Update Rule | Delete Rule |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `tenants` | `@request.auth.id != ""` | `@request.auth.id != ""` | `null` (Admin only) | `null` (Admin only) | `null` (Admin only) |
| `tenant_credentials` | `null` (Superuser only) | `null` | `null` | `null` | `null` |
| `tenant_rules` | `@request.auth.id != ""` | `@request.auth.id != ""` | `@request.auth.id != ""` | `@request.auth.id != ""` | `@request.auth.id != ""` |
| `tenant_tools` | `@request.auth.id != ""` | `@request.auth.id != ""` | `null` | `null` | `null` |
| `customers` | `@request.auth.id != ""` | `@request.auth.id != ""` | `@request.auth.id != ""` | `@request.auth.id != ""` | `null` |
| `customer_traits` | `@request.auth.id != ""` | `@request.auth.id != ""` | `@request.auth.id != ""` | `@request.auth.id != ""` | `@request.auth.id != ""` |
| `sessions` | `@request.auth.id != ""` | `@request.auth.id != ""` | `@request.auth.id != ""` | `@request.auth.id != ""` | `null` |
| `tickets` | `@request.auth.id != ""` | `@request.auth.id != ""` | `@request.auth.id != ""` | `@request.auth.id != ""` | `null` |
| `messages` | `@request.auth.id != ""` | `@request.auth.id != ""` | `@request.auth.id != ""` | `null` (Immutable) | `null` |
| `orders` | `@request.auth.id != ""` | `@request.auth.id != ""` | `@request.auth.id != ""` | `@request.auth.id != ""` | `null` |
| `order_items` | `@request.auth.id != ""` | `@request.auth.id != ""` | `@request.auth.id != ""` | `@request.auth.id != ""` | `null` |

---

## 4. Native PocketBase Hooks Blueprint (`pb_hooks`)

PocketBase JSVM hooks are placed in `pb_hooks/` to enforce business invariants natively at the database level:

```javascript
// pb_hooks/order_protection.pb.js
// Prevent modification of orders once marked PAID
onRecordBeforeUpdateRequest((e) => {
  const original = $app.findRecordById("orders", e.record.id);
  if (original.getString("status") === "PAID" && e.record.getString("status") === "CANCELLED") {
    throw new BadRequestError("Cannot cancel an order that has already been verified as PAID.");
  }
  e.next();
}, "orders");

// Auto-calculate subtotal for order items
onRecordBeforeCreateRequest((e) => {
  const price = e.record.getFloat("unit_price");
  const qty = e.record.getFloat("quantity");
  e.record.set("subtotal", price * qty);
  e.next();
}, "order_items");
```
