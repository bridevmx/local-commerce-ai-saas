# Especificación del Stack Tecnológico
## SaaS de Asistente de IA para Comercio Local

**Versión:** 1.0.0  
**Fecha:** Marzo 2025  

---

## 1. Matriz de Componentes del Stack

| Capa / Subsistema | Tecnología / Herramienta | Versión | Propósito Arquitectónico |
| :--- | :--- | :--- | :--- |
| **Entorno de Ejecución** | Node.js | `>=20.18.0` (LTS) | Runtime principal del backend, con soporte nativo para fetch y WebSockets. |
| **Gestor de Paquetes** | pnpm | `>=9.0.0` | Instalación rápida y determinista de dependencias mediante enlaces duros. |
| **Conector de WhatsApp** | `@whiskeysockets/baileys` | `^6.7.9` | Protocolo nativo de WhatsApp Multi-Device sin dependencias de navegador (Chromium). |
| **Framework de Telegram** | `grammy` | `^1.34.0` | Cliente tipado para la API de Bots de Telegram, soporte nativo de topics/foros y menús en línea. |
| **Capa de Persistencia** | PocketBase | `^0.23.0` | Base de datos SQLite embebida en modo WAL, auth server-side, realtime y reglas de seguridad. |
| **Cliente de Base de Datos** | `pocketbase` (JS SDK) | `^0.21.5` | SDK oficial para Node.js con auto-reintento y soporte completo de filtros relacionales. |
| **Gateway de IA** | OmniRoute (Local) | `latest` | Proxy local compatible con OpenAI en el puerto 20128 con ruteo inteligente y fallback de cuotas. |
| **Modelos de Lenguaje** | Groq (`llama-3.3-70b-versatile`) | API v1 | Inferencia ultrarrápida para razonamiento ReAct y Tool Calling estructurado. |
| **Visión y Multimodal** | Google Gemini (`gemini-2.0-flash`) | API v1beta | Análisis de fotografías, comprobantes y fallback de cuotas gestionado por OmniRoute. |
| **Reconocimiento de Voz** | Groq Whisper (`whisper-large-v3`) | API v1 | Transcripción de audios de WhatsApp con precisión idiomática y puntuación en español. |
| **Validación de Esquemas** | `zod` | `^3.23.8` | Validación estricta en tiempo de compilación y runtime para inputs de tools y contratos de API. |
| **Cifrado de Credenciales** | Node.js `crypto` | Nativo | Cifrado simétrico AES-256-GCM para llaves de comercio en reposo. |
| **Respaldo Continuo** | Litestream | `^0.3.13` | Replicación continua de SQLite a nivel de página hacia Cloudflare R2 / AWS S3. |
| **Exposición Segura** | Cloudflare Tunnels (`cloudflared`) | `latest` | Entrada HTTPS sin abrir puertos en el router para webhooks de PayClip y Telegram. |

---

## 2. Especificación de Versiones de Dependencias (`package.json`)

```json
{
  "name": "local-commerce-ai-saas",
  "version": "1.0.0",
  "private": true,
  "description": "Asistente de IA para Comercios Locales con WhatsApp, Telegram y PayClip",
  "type": "module",
  "scripts": {
    "dev": "node --watch src/index.js",
    "start": "node src/index.js",
    "test": "node --test tests/**/*.test.js",
    "lint": "eslint src/ tests/",
    "db:migrate": "node scripts/run-migrations.js",
    "db:backup": "litestream replicate-once /pb_data/data.db s3://saas-backups/pb.db"
  },
  "dependencies": {
    "@whiskeysockets/baileys": "^6.7.9",
    "dotenv": "^16.4.7",
    "grammy": "^1.34.0",
    "openai": "^4.85.0",
    "pino": "^9.6.0",
    "pino-pretty": "^13.0.0",
    "pocketbase": "^0.21.5",
    "qrcode-terminal": "^0.12.0",
    "zod": "^3.23.8"
  },
  "devDependencies": {
    "eslint": "^9.20.1"
  },
  "engines": {
    "node": ">=20.18.0",
    "pnpm": ">=9.0.0"
  }
}
```

---

## 3. Matriz de Variables de Entorno (`.env.example`)

```bash
# ==============================================================================
# CONFIGURACIÓN DEL SISTEMA Y ENTORNO
# ==============================================================================
NODE_ENV=development
PORT=3000
BASE_PUBLIC_URL=https://api.tu-dominio.com

# ==============================================================================
# SEGURIDAD Y CIFRADO
# ==============================================================================
# Clave hexadecimal de 32 bytes (64 caracteres) para cifrado AES-256-GCM
MASTER_ENCRYPTION_KEY=0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef

# ==============================================================================
# POCKETBASE BACKEND
# ==============================================================================
POCKETBASE_URL=http://127.0.0.1:8090
POCKETBASE_ADMIN_EMAIL=admin@tu-dominio.com
POCKETBASE_ADMIN_PASSWORD=super-secure-admin-password-123

# ==============================================================================
# OMNIROUTE AI GATEWAY Y MODELOS
# ==============================================================================
# Endpoint del gateway local OmniRoute
AI_GATEWAY_URL=http://localhost:20128/v1
AI_GATEWAY_API_KEY=omniroute-local-token

# Identificadores de modelos
PRIMARY_LLM_MODEL=groq/llama-3.3-70b-versatile
FALLBACK_LLM_MODEL=google/gemini-2.0-flash
VISION_LLM_MODEL=google/gemini-2.0-flash
AUDIO_TRANSCRIPTION_MODEL=groq/whisper-large-v3

# Llaves directas de respaldo
GROQ_API_KEY=gsk_your_groq_api_key_here
GEMINI_API_KEY=AIzaSy_your_gemini_api_key_here

# ==============================================================================
# BOT DE TELEGRAM (CONSOLA DE OPERACIONES)
# ==============================================================================
TELEGRAM_BOT_TOKEN=123456789:ABCdefGhIJKlmNoPQRsTUVwxyZ123456789

# ==============================================================================
# PAYCLIP (PAGOS Y COMERCIO)
# ==============================================================================
# Dominio API Principal (F2F y Transacciones TPV)
CLIP_API_URL=https://api.clip.mx
# Dominio IO (Checkout E-commerce y Enlaces de Pago)
CLIP_IO_URL=https://api.payclip.com
# Dominio API-GW (Telemetría de Terminales F2F)
CLIP_API_GW_URL=https://api-gw.payclip.com

# ==============================================================================
# REPLICACIÓN CONTINUA DE BASE DE DATOS (LITESTREAM)
# ==============================================================================
LITESTREAM_BUCKET=saas-sqlite-backups
LITESTREAM_ENDPOINT=https://your-account-id.r2.cloudflarestorage.com
LITESTREAM_ACCESS_KEY_ID=your_r2_access_key
LITESTREAM_SECRET_ACCESS_KEY=your_r2_secret_key
```

---

## 4. Justificación Técnica de Decisiones Clave

### Por qué `@whiskeysockets/baileys` nativo (y descarte de BuilderBot)
1. **Consumo de Memoria:** BuilderBot encapsula múltiples capas de abstracción innecesarias que incrementan el uso de memoria a ~150-250MB por sesión de WhatsApp. Baileys puro sobre WebSockets opera de forma eficiente consumiendo únicamente ~35MB por sesión, permitiendo escalar a más de 30 comercios por nodo económico de 2GB de RAM.
2. **Mitigación de Baneos con Presencia Orgánica:** Baileys permite manipular eventos de presencia de WhatsApp con total control. El sistema implementa un retardo orgánico antes de responder: marca los mensajes como leídos tras 800-1500ms, emite el estado `composing` durante el razonamiento ReAct con una duración proporcional a la longitud de la respuesta final, y responde exclusivamente a mensajes entrantes sin realizar prospección saliente no solicitada.

### Por qué OmniRoute como AI Gateway Local
1. **Mitigación Automática de Límites de Tasa (429):** Los planes gratuitos y de bajo costo de Groq imponen límites estrictos de peticiones por minuto. OmniRoute actúa como un balanceador transparente que redirige automáticamente las solicitudes a Google Gemini 2.0 Flash en cuanto detecta un error de cuota o una degradación de latencia.
2. **Interfaz Unificada OpenAI:** Toda la lógica del agente en Node.js se escribe utilizando el cliente estándar `openai`, desacoplando por completo el código de los SDKs propietarios de cada proveedor de IA.

### Por qué PocketBase SQLite con modo WAL y Litestream
1. **Simplicidad Operativa Monolítica:** Un único binario ligero en Go con interfaz gráfica embebida, autenticación lista para producción, migraciones declarativas en JavaScript y motor de base de datos SQLite embebido sin la sobrecarga operativa de un cluster PostgreSQL.
2. **Alta Concurrencia de Lectura:** El modo WAL (Write-Ahead Logging) permite lecturas concurrentes sin bloqueo mientras se procesan inserciones transaccionales.
3. **Resiliencia de Datos con RPO < 1s:** Litestream escucha los cambios del archivo WAL de SQLite y replica los bloques modificados inmediatamente hacia almacenamiento compatible con S3 (Cloudflare R2), garantizando recuperación instantánea ante fallas catastróficas del servidor sin pérdida de datos.
