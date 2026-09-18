# Plan de Implementación de Ingeniería
## Desglose de Construcción Modular y Blueprints de Código

**Versión:** 1.0.0  
**Fecha:** Marzo 2025  

---

## 1. Árbol de Directorios del Código Fuente

```
local-commerce-ai-saas/
├── .env.example
├── .gitignore
├── package.json
├── pnpm-lock.yaml
├── README.md
├── ARCHITECTURE.md
├── TECH_STACK.md
├── DATABASE_SCHEMA.md
├── INTEGRATIONS.md
├── TOOL_REGISTRY_AND_PRESETS.md
├── ROADMAP.md
├── IMPLEMENTATION_PLAN.md
├── docs/
│   └── es/
│       ├── README.md
│       ├── ARCHITECTURE.md
│       ├── TECH_STACK.md
│       ├── DATABASE_SCHEMA.md
│       ├── INTEGRATIONS.md
│       ├── TOOL_REGISTRY_AND_PRESETS.md
│       ├── ROADMAP.md
│       └── IMPLEMENTATION_PLAN.md
├── pb_migrations/
│   ├── 1710000001_init_tenants_and_credentials.js
│   ├── 1710000002_init_customers_and_traits.js
│   ├── 1710000003_init_sessions_and_messages.js
│   └── 1710000004_init_orders_and_rules.js
├── storage/
│   └── sessions/                  # Credenciales de Baileys por tenant (ignorado en git)
├── src/
│   ├── index.js                   # Punto de entrada de la aplicación
│   ├── config/
│   │   ├── env.js                 # Validación de variables de entorno con Zod
│   │   └── constants.js           # Constantes de tiempos y estados
│   ├── db/
│   │   ├── pocketbase.js          # Instancia y autenticación de PocketBase
│   │   └── crypto.js              # Cifrado/Descifrado AES-256-GCM
│   ├── channels/
│   │   ├── whatsapp/
│   │   │   ├── baileys-client.js  # Conexión socket, eventos y QR
│   │   │   ├── presence.js        # Retardos orgánicos y estado 'composing'
│   │   │   └── session-store.js   # Gestión multi-tenant de auth en disco
│   │   ├── telegram/
│   │   │   ├── bot.js             # Instancia de GramMY y middlewares
│   │   │   ├── topics.js          # Enrutamiento de topics (#Dudas, #Catálogo, #Ventas)
│   │   │   └── interactive-cards.js # Tarjetas con botones mutables (editMessageText)
│   │   └── buffer/
│   │       └── burst-aggregator.js# Buffer debounce de 30s en RAM por cliente
│   ├── agent/
│   │   ├── react-loop.js          # Ciclo ReAct (máximo 5 iteraciones)
│   │   ├── prompt-compiler.js     # Compilador dinámico de 5 capas
│   │   ├── memory-extractor.js    # Extracción asíncrona de rasgos EAV
│   │   └── tools/
│   │       ├── registry.js        # Registro de tools y despachador
│   │       ├── clip-catalog.js    # Tool: clip_check_catalog
│   │       ├── clip-payment.js    # Tool: clip_create_payment
│   │       ├── escalation.js      # Tool: escalate_to_human
│   │       └── order-lead.js      # Tool: register_order_lead
│   ├── integrations/
│   │   ├── omniroute/
│   │   │   └── client.js          # Cliente OpenAI conectado a http://localhost:20128/v1
│   │   ├── groq/
│   │   │   └── whisper.js         # Transcripción de audios en español
│   │   └── payclip/
│   │       ├── catalog-api.js     # Regla obligatoria GET-before-PATCH
│   │       ├── checkout-io.js     # Creación de Payment Links
│   │       └── webhook-verifier.js# Verificación Zero-Trust (Fast-ACK + API Readback)
│   └── webhooks/
│       └── server.js              # Servidor HTTP Express/Node para webhooks públicos
└── tests/
    ├── unit/
    │   ├── buffer.test.js
    │   ├── crypto.test.js
    │   └── prompt-compiler.test.js
    └── integration/
        ├── clip-get-before-patch.test.js
        └── webhook-zero-trust.test.js
```

---

## 2. Secuencia Atómica de 10 Tareas de Construcción

1. **Tarea 1 - Configuración de Repositorio y Validación de Entorno:** Setup de Node.js 20+, `package.json`, `.gitignore` y módulo de variables de entorno con Zod (`src/config/env.js`).
2. **Tarea 2 - Migraciones de Base de Datos Zero-JSON:** Creación de las 11 colecciones en PocketBase con índices únicos compuestos y reglas de seguridad cerradas (`null`).
3. **Tarea 3 - Módulo Criptográfico AES-256-GCM:** Implementación de cifrado y descifrado seguro para llaves de comercio en `tenant_credentials`.
4. **Tarea 4 - Conector Nativo de WhatsApp con Baileys:** Implementación de socket Multi-Device con emulación de presencia orgánica (retardo de lectura 800-1500ms y `composing` proporcional).
5. **Tarea 5 - Buffer de Agregación de Ráfagas (30 Segundos):** Administrador de colas en memoria con `NodeJS.Timeout` deslizante y transcripción en paralelo con Whisper.
6. **Tarea 6 - Integración de OmniRoute y Groq Whisper:** Cliente OpenAI configurado para inferencia primaria en Groq Llama 3.3 70B y conmutación hacia Gemini 2.0 Flash.
7. **Tarea 7 - Compilador Modular de Prompts en 5 Capas:** Ensamblado dinámico en tiempo de ejecución combinando guardarraíles, preset vertical, datos del comercio, reglas atómicas y memoria EAV.
8. **Tarea 8 - Motor ReAct y Registro de Tools Zod:** Bucle de inferencia con límite de 5 iteraciones y herramientas para consultar inventario, crear pagos y escalar a soporte humano.
9. **Tarea 9 - Consola en Supergrupo de Telegram:** Bot de GramMY con aislamiento por `telegram_chat_id`, soporte para respuestas citadas en `#Dudas-Clientes` y tarjetas mutables en `#Ventas-y-Caja`.
10. **Tarea 10 - Webhooks Zero-Trust de PayClip y Telemetría F2F:** Endpoint público con Fast-ACK inmediato y verificación autenticada en segundo plano con la API de Clip.

---

## 3. Blueprints de Código Crítico

### 3.1. Buffer de Debounce en Memoria (`src/channels/buffer/burst-aggregator.js`)
```javascript
import { transcribeAudio } from '../../integrations/groq/whisper.js';

export class BurstAggregator {
  constructor(timeoutMs = 30000, onFlushCallback) {
    this.timeoutMs = timeoutMs;
    this.onFlushCallback = onFlushCallback;
    this.buffers = new Map(); // key: `tenant:customer` -> { timer, items: [] }
  }

  push(tenantId, customerPhone, messageItem) {
    const key = `${tenantId}:${customerPhone}`;
    let entry = this.buffers.get(key);

    if (entry) {
      clearTimeout(entry.timer);
    } else {
      entry = { items: [], timer: null };
      this.buffers.set(key, entry);
    }

    entry.items.push(messageItem);

    entry.timer = setTimeout(async () => {
      this.buffers.delete(key);
      await this.processFlushedItems(tenantId, customerPhone, entry.items);
    }, this.timeoutMs);
  }

  async processFlushedItems(tenantId, customerPhone, items) {
    const textFragments = [];

    for (const item of items) {
      if (item.type === 'text') {
        textFragments.push(item.text);
      } else if (item.type === 'audio') {
        const transcription = await transcribeAudio(item.audioBuffer);
        textFragments.push(`[Nota de voz transcrita: "${transcription}"]`);
      } else if (item.type === 'image') {
        textFragments.push(`[Imagen adjunta del cliente: ${item.caption || 'sin descripción'}]`);
      }
    }

    const unifiedUserMessage = textFragments.join('\n');
    await this.onFlushCallback(tenantId, customerPhone, unifiedUserMessage, items);
  }
}
```

### 3.2. Cifrado AES-256-GCM (`src/db/crypto.js`)
```javascript
import crypto from 'node:crypto';

const ALGORITHM = 'aes-256-gcm';
const KEY = Buffer.from(process.env.MASTER_ENCRYPTION_KEY, 'hex');

export function encryptSecret(plainText) {
  const iv = crypto.randomBytes(12);
  const cipher = crypto.createCipheriv(ALGORITHM, KEY, iv);
  
  let encrypted = cipher.update(plainText, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  const authTag = cipher.getAuthTag().toString('hex');

  return {
    encrypted_key: encrypted,
    iv: iv.toString('hex'),
    auth_tag: authTag
  };
}

export function decryptSecret(encryptedHex, ivHex, authTagHex) {
  const decipher = crypto.createDecipheriv(
    ALGORITHM,
    KEY,
    Buffer.from(ivHex, 'hex')
  );
  decipher.setAuthTag(Buffer.from(authTagHex, 'hex'));

  let decrypted = decipher.update(encryptedHex, 'hex', 'utf8');
  decrypted += decipher.final('utf8');
  return decrypted;
}
```

---

## 4. Estrategia de Testing con Mocks

Para garantizar alta fiabilidad sin consumir saldo en APIs externas:

1. **Mock de PayClip (`tests/mocks/clip-api.mock.js`):** Intercepta llamadas hacia `api.clip.mx` y `api.payclip.com` simulando respuestas válidas, catálogos simulados y comprobación de la regla GET-before-PATCH.
2. **Mock de OmniRoute AI Gateway:** Servidor HTTP local con respuestas deterministas en formato JSON compatible con OpenAI para validar las transiciones del bucle ReAct.
3. **Simulador de Socket Baileys:** Emite secuencias de eventos de mensajes entrantes con marcas de tiempo arbitrarias para verificar que el agregador de ráfagas reinicie el temporizador de 30 segundos y unifique las cargas de forma confiable.
