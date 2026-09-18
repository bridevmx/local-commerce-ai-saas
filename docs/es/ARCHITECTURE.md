# Especificación de Arquitectura de Software
## SaaS de Asistente de IA para Comercio Local

**Versión:** 1.0.0  
**Fecha:** Marzo 2025  

---

## 1. Topología del Sistema

El sistema opera como una arquitectura modular orientada a eventos en Node.js 20+ LTS, utilizando WebSockets directos hacia WhatsApp (Baileys nativo), webhooks asíncronos y long-polling hacia la API de Bots de Telegram, coordinado mediante PocketBase (SQLite WAL) y un gateway local OmniRoute.

```
                  +-------------------------------------------------------------+
                  |                      CLIENTES FINALES                       |
                  +-------------------------------------------------------------+
                                                 |
                                     [WhatsApp Multi-Device]
                                                 |
                                                 v
                  +-------------------------------------------------------------+
                  |               GATEWAY BAILEYS (Nativo Node.js)              |
                  |  - Conexión WebSocket directa por Tenant                    |
                  |  - Presencia orgánica (delay 800-1500ms + composing)        |
                  |  - Sesiones en ./storage/sessions/{tenant_id}               |
                  +-------------------------------------------------------------+
                                                 |
                                     [Evento de Mensaje Crudo]
                                                 |
                                                 v
                  +-------------------------------------------------------------+
                  |           BUFFER DE CONCURRENCIA (Debounce en RAM)          |
                  |  - Temporizador de inactividad de 30s por cliente           |
                  |  - Acumula texto, imágenes y audios de voz                  |
                  |  - Transcripción paralela con Groq Whisper                  |
                  +-------------------------------------------------------------+
                                                 |
                                     [Turno de Mensaje Unificado]
                                                 |
                                                 v
+---------------------------------------------------------------------------------------------+
|                                    MOTOR DEL AGENTE REACT                                   |
|                                                                                             |
|   +--------------------------+  +--------------------------+  +-------------------------+   |
|   |   Compilador de Prompt   |  |   OmniRoute AI Gateway   |  |   Registro de Tools     |   |
|   |  - 5 Capas Dinámicas     |  |  - Groq Llama 3.3 70B    |  |  - clip_check_catalog   |   |
|   |  - Reglas por Tenant     |  |  - Fallback: Gemini 2.0  |  |  - clip_create_payment  |   |
|   |  - Memoria EAV Clientes  |  |  - Máx. 5 Iteraciones    |  |  - escalate_to_human    |   |
|   +--------------------------+  +--------------------------+  +-------------------------+   |
+---------------------------------------------------------------------------------------------+
               |                                            |                        |
     [Lectura / Escritura]                        [Handoff / Alertas]      [Transacciones / Stock]
               v                                            v                        v
+-----------------------------+               +-----------------------+  +--------------------+
| POCKETBASE (SQLite WAL)     |               | TELEGRAM BOT (GramMY) |  |   CORE PAYCLIP     |
| - 11 Colecciones Zero-JSON  |               | - Supergrupo Tenant   |  | - 3 Dominios API   |
| - Litestream Réplica S3/R2  |               | - Tópicos dedicados   |  | - GET-before-PATCH |
| - Credenciales AES-256-GCM  |               | - Tarjetas interactiv.|  | - Zero-Trust Hook  |
+-----------------------------+               +-----------------------+  +--------------------+
```

---

## 2. Diagrama de Flujo de Datos

```
Cliente (WA)       Baileys          Buffer RAM         OmniRoute         PocketBase        Telegram
    |                 |                 |                  |                 |                 |
    |-- "Hola!" ----->|                 |                  |                 |                 |
    |                 |-- Encolar ----->|                  |                 |                 |
    |                 |                 | (Inicia 30s)     |                 |                 |
    |-- [Audio 8s] -->|                 |                  |                 |                 |
    |                 |-- Encolar ----->|                  |                 |                 |
    |                 |                 | (Reinicia 30s)   |                 |                 |
    |                 |                 |                  |                 |                 |
    |                 |                 | [Vencen 30s]     |                 |                 |
    |                 |                 |-- Transcribe --->|                 |                 |
    |                 |                 |<-- Texto unif. --|                 |                 |
    |                 |                 |                  |                 |                 |
    |                 |                 |-- Invocar ReAct ------------------>|                 |
    |                 |                 |                  |-- Traer Reglas->|                 |
    |                 |                 |                  |<-- Reglas/Mem --|                 |
    |                 |                 |                  |                 |                 |
    |                 |                 |                  |-- Tool Call --->|                 |
    |                 |                 |                  |<-- Resultado ---|                 |
    |                 |                 |                  |                 |                 |
    |                 |                 |                  |-- ¿Escalar? ----|---------------> |
    |                 |                 |                  |                 | (Tópico Dudas)  |
    |                 |                 |                  |                 |                 |
    |                 |<-- Respuesta final ----------------|                 |                 |
    |<-- "Listo!" ----|                 |                  |                 |                 |
    |                 |                 |                  |                 |                 |
    |                 |                 |                  |-- Extracción EAV (Async) -------> |
    |                 |                 |                  |                 | (Guarda Rasgos) |
```

---

## 3. Aislamiento Multi-Tenant y Seguridad

1. **Aislamiento en Base de Datos:** Cada tabla transaccional (`customers`, `sessions`, `tickets`, `messages`, `orders`, `tenant_rules`, `tenant_tools`) implementa un campo indexado `tenant` que actúa como clave foránea hacia la colección `tenants`.
2. **Aislamiento de Sesiones de WhatsApp:** Las credenciales de conexión de Baileys se almacenan en carpetas aisladas por comercio: `./storage/sessions/{tenant_id}/`. Los procesos de socket nunca comparten estado de memoria entre comercios.
3. **Cifrado en Reposo de Claves de API:** Las credenciales de Clip y tokens de terceros se almacenan en la colección `tenant_credentials` cifradas mediante **AES-256-GCM**, empleando un vector de inicialización único (IV) y etiqueta de autenticación (auth tag). La clave maestra de descifrado proviene de `process.env.MASTER_ENCRYPTION_KEY`.
4. **Enrutamiento Multi-Tenant en Telegram:** Cada comercio opera con su propio Supergrupo de Telegram identificado por `tenant.telegram_chat_id`. El bot valida rigurosamente el `ctx.chat.id` antes de procesar mutaciones de catálogo o resolver tickets de soporte.

---

## 4. Patrón de Agregación de Ráfagas (Buffer Debounce de 30 Segundos)

Los clientes de comercios locales envían consultas fragmentadas en ráfagas (mensajes de texto cortos seguidos de notas de voz e imágenes). Enviar cada mensaje inmediatamente al LLM genera respuestas fragmentadas, colisiones de estado y un gasto innecesario de tokens.

### Mecánica del Buffer en Memoria
* Clave única en memoria: `buffer:${tenant_id}:${customer_phone}`.
* Ventana de inactividad fija: **30 segundos (30,000 milisegundos)**.
* Cada nuevo mensaje entrante dentro de la ventana cancela el `NodeJS.Timeout` activo, añade el fragmento al arreglo de la cola y crea un nuevo temporizador de 30s.
* Al expirar el temporizador:
  1. El buffer extrae de forma atómica los elementos acumulados y libera la clave en memoria.
  2. Los mensajes de voz (`audio/ogg; codecs=opus`) se descargan desde el socket de Baileys y se envían en paralelo a Groq Whisper (`whisper-large-v3`, `language: 'es'`).
  3. Las imágenes se procesan como blobs base64 para inspección visual con Gemini 2.0 Flash.
  4. Los textos transcritos y los mensajes escritos se concatenan cronológicamente en un único bloque de entrada para el usuario.
  5. Se emite la presencia `composing` en WhatsApp durante la inferencia y se activa el ciclo ReAct.

---

## 5. Máquinas de Estados Finitos (FSM)

### A. Ciclo de Vida del Ticket de Soporte (`tickets.status`)
```
     +--------+   Requiere humano o ReAct escala   +-----------+
---> |  OPEN  | ---------------------------------> | ESCALATED |
     +--------+                                    +-----------+
         |                                               |
         | Consulta normal satisfecha                    | Dueño responde
         v                                               v
     +------------+                                +-----------+
     |  RESOLVED  | <----------------------------- | IN_PROGRESS|
     +------------+         Handoff concluido      +-----------+
```

### B. Ciclo de Vida de la Orden de Compra (`orders.status`)
```
     +-----------+  Link Clip generado   +----------------+
---> | DRAFT     | --------------------> | PENDING_PAYMENT|
     +-----------+                       +----------------+
           |                                     |
           | Timeout 15 min                      | Webhook Zero-Trust verificado
           v                                     v
     +-----------+                       +----------------+
     | CANCELLED |                       | PAID           |
     +-----------+                       +----------------+
                                                 |
                                                 | Dueño confirma en Telegram
                                                 v
                                         +----------------+
                                         | FULFILLED      |
                                         +----------------+
```

---

## 6. Integración del Gateway OmniRoute

OmniRoute se despliega como servicio local en `http://localhost:20128/v1` ofreciendo interfaz estándar compatible con OpenAI:

* **Inferencia Primaria:** `groq/llama-3.3-70b-versatile` para baja latencia (150-300ms time-to-first-token) y ejecución precisa de llamadas a herramientas (Tool Calling).
* **Conmutación Automática (Failover):** Ante errores de cuota (`429 Too Many Requests`) o indisponibilidad en Groq, OmniRoute conmuta de manera transparente a `google/gemini-2.0-flash`.
* **Procesamiento de Fotos:** Todas las consultas con imágenes adjuntas se dirigen automáticamente a `google/gemini-2.0-flash` para descripción de productos y validación de comprobantes.
* **Límite de Seguridad:** El bucle ReAct del backend impone un máximo estricto de **5 iteraciones**. Si el modelo no concluye en 5 pasos, fuerza una salida de contingencia y escala el caso a Telegram con contexto completo.

---

## 7. Extracción Asíncrona de Rasgos del Cliente (Memoria EAV)

Para personalizar futuras conversaciones sin ralentizar la respuesta actual, la memoria del cliente se procesa en segundo plano:

1. El agente responde al cliente en WhatsApp inmediatamente tras generar la respuesta final.
2. Mediante `setImmediate`, se invoca una llamada secundaria al LLM utilizando JSON estructurado para analizar la conversación completa.
3. El analizador extrae hechos atómicos del cliente clasificados en categorías cerradas (`dietary_preference`, `favorite_item`, `delivery_address`, `pet_breed`, `general_note`).
4. Los atributos se insertan o actualizan en la tabla `customer_traits` mediante clave única compuesta `(customer, category, trait_key)`.
5. En la próxima interacción del cliente, el compilador de prompts inyecta estos rasgos estructurados en la Capa 5 del contexto.
