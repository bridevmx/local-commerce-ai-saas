# Hoja de Ruta de Ingeniería (Roadmap)
## Cronograma de 5 Fases (10 Semanas) y Matriz de Riesgos

**Versión:** 1.0.0  
**Fecha:** Marzo 2025  

---

## 1. Cronograma de 5 Fases (10 Semanas) hacia Producción

```
+-----------------------------------------------------------------------------------------+
| SPRINT 1 (Semanas 1-2): Cimientos del Sistema, Base de Datos y Core de Conectividad     |
| - Setup de PocketBase v0.23+, migraciones JS y 11 colecciones Zero-JSON.                |
| - Módulo de cifrado AES-256-GCM para llaves de comercio.                                |
| - Integración de Baileys nativo con presencia orgánica y almacenamiento de sesiones.   |
+-----------------------------------------------------------------------------------------+
                                             |
                                             v
+-----------------------------------------------------------------------------------------+
| SPRINT 2 (Semanas 3-4): Buffer de Concurrencia y AI Gateway OmniRoute                   |
| - Implementación del Buffer de Debounce en memoria (30s de inactividad por cliente).    |
| - Integración paralela con Groq Whisper para notas de voz en español.                   |
| - Configuración del Gateway OmniRoute (Llama 3.3 70B primario + fallback Gemini 2.0).   |
+-----------------------------------------------------------------------------------------+
                                             |
                                             v
+-----------------------------------------------------------------------------------------+
| SPRINT 3 (Semanas 5-6): Motor ReAct, Tools de PayClip y Compilador de Prompts           |
| - Bucle ReAct con guardarraíl de 5 iteraciones y esquemas Zod.                          |
| - Integración de PayClip: regla GET-before-PATCH y generación de Checkouts.             |
| - Compilador de Prompts en 5 capas con inyección dinámica de `tenant_rules`.             |
+-----------------------------------------------------------------------------------------+
                                             |
                                             v
+-----------------------------------------------------------------------------------------+
| SPRINT 4 (Semanas 7-8): Consola Operativa en Telegram y Extracción Asíncrona EAV        |
| - Bot en GramMY con aislamiento multi-tenant por `telegram_chat_id`.                    |
| - Tópico #Dudas-Clientes con handoff humano bidireccional vía citas nativas.           |
| - Tópico #Mi-Catálogo con control de stock y #Ventas-y-Caja con tarjetas mutables.      |
| - Extracción de rasgos en segundo plano (`setImmediate`) hacia `customer_traits`.       |
+-----------------------------------------------------------------------------------------+
                                             |
                                             v
+-----------------------------------------------------------------------------------------+
| SPRINT 5 (Semanas 9-10): Webhooks Zero-Trust, Pruebas E2E y Despliegue en Producción    |
| - Receptor de Webhook de Clip con Fast-ACK y validación API Readback en segundo plano.  |
| - Telemetría de 14 campos para transacciones en efectivo.                               |
| - Replicación de base de datos con Litestream hacia Cloudflare R2 / S3.                 |
| - Pruebas de estrés y despliegue del piloto en 3 comercios reales.                      |
+-----------------------------------------------------------------------------------------+
```

---

## 2. Criterios de Aceptación Formales (Definition of Done) por Fase

### Fase 1: Cimientos y Conectores
* Las 11 colecciones de PocketBase se crean automáticamente vía script de migración.
* Las llaves de API de comercio se cifran y descifran de forma determinista mediante AES-256-GCM.
* Baileys se conecta mediante código QR en terminal y almacena credenciales en `./storage/sessions/{tenant_id}/`.
* Se emiten retardos orgánicos (`composing` y lectura diferida entre 800 y 1500ms).

### Fase 2: Buffer e Inferencia
* Múltiples mensajes de texto y notas de voz recibidos en un intervalo menor a 30 segundos se unifican en un solo turno conversacional.
* Groq Whisper transcribe notas de voz en español en menos de 1,200 ms.
* Si el modelo primario en Groq devuelve error 429, OmniRoute redirige la consulta a Gemini 2.0 Flash sin interrumpir la experiencia del usuario.

### Fase 3: ReAct y PayClip
* El bucle ReAct no excede en ningún escenario las 5 iteraciones.
* Al actualizar un producto de catálogo, el sistema valida que el stock no sea sobreescrito a `null`.
* El prompt del sistema compila en menos de 50 ms inyectando correctamente las reglas de `tenant_rules`.

### Fase 4: Telegram y Memoria EAV
* Las respuestas dadas por el comerciante en `#Dudas-Clientes` mediante citas de Telegram son entregadas con éxito al WhatsApp del cliente correspondiente.
* Los rasgos de clientes extraídos asíncronamente aparecen en `customer_traits` sin retrasar la respuesta al cliente.
* Las tarjetas de pedidos en `#Ventas-y-Caja` actualizan su estado utilizando `editMessageText`.

### Fase 5: Seguridad y Producción
* Los webhooks de Clip responden con HTTP 200 en menos de 500 ms y verifican el estado del pago mediante API Readback.
* Litestream replica los bloques WAL hacia R2 con un desfase menor a 1 segundo.
* Suite de pruebas unitarias y de integración con cobertura superior al 85% de la lógica de negocio.

---

## 3. Matriz de Riesgos Técnicos y Mitigaciones

| Riesgo Técnico | Probabilidad | Impacto | Estrategia de Mitigación Arquitectónica |
| :--- | :--- | :--- | :--- |
| **Baneo de número de WhatsApp por Meta** | Media | Catastrófico | Operación 100% inbound (solo responder a clientes que inicien contacto), presencia orgánica (delay de lectura de 800-1500ms, emulación de escritura `composing`) y aislamiento estricto de sesiones. |
| **Sobrescritura accidental de existencias en Clip** | Alta | Crítico | Regla obligatoria `GET-before-PATCH`: consultar siempre el producto antes de enviar el parche y reinyectar el `stock` vigente. |
| **Agotamiento de cuota de inferencia (Error 429)** | Alta | Alto | Gateway OmniRoute local con balanceo automático hacia Gemini 2.0 Flash y Groq Whisper. |
| **Corrupción de base de datos en caso de apagón** | Baja | Alto | SQLite configurado en modo WAL con `PRAGMA synchronous = NORMAL`, más replicación continua en streaming hacia S3/R2 mediante Litestream. |
| **Alucinación de precios o inventario por el LLM** | Media | Alto | Guardarraíles de Capa 1 en el compilador de prompts prohibiendo explícitamente deducir precios sin invocar la herramienta `clip_check_catalog`. |
| **Suplantación de pagos mediante webhooks falsos** | Media | Crítico | Verificación Zero-Trust: descarte de datos no verificados del webhook y confirmación obligatoria mediante API Readback (`GET /v2/checkout/{id}`). |

---

## 4. Hitos de Entrega Clave

* **Hito M0 (Fin de Semana 2):** Conexión multi-tenant de WhatsApp activa y esquema PocketBase validado.
* **Hito M1 (Fin de Semana 4):** Pipeline de buffer de 30 segundos, transcripción de voz y pasarela OmniRoute en marcha.
* **Hito M2 (Fin de Semana 6):** Agente ReAct autónomo con capacidad de consulta y cobro en PayClip.
* **Hito M3 (Fin de Semana 8):** Consola operativa en Telegram enlazada bidireccionalmente con WhatsApp y memoria EAV activa.
* **Hito M4 (Fin de Semana 10):** Webhooks seguros verificados, replicación Litestream activa y despliegue del piloto productivo.
