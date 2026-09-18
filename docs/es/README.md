# Asistente de IA para Comercio Local SaaS
## Suite de Documentación Técnica y Especificaciones de Ingeniería

**Versión del Sistema:** 1.0.0  
**Estado:** Listo para Desarrollo / Fase de Construcción  
**Arquitectura Objetivo:** Multi-Tenant Node.js + PocketBase (Zero-JSON) + OmniRoute Gateway + Baileys + CRM en Supergrupos de Telegram  

---

## 1. Resumen Ejecutivo del Proyecto

El **SaaS de Asistente de IA para Comercio Local** es una plataforma de comercio conversacional autónomo diseñada específicamente para comercios físicos locales (pastelerías, barberías, cafeterías, clínicas boutique). Elimina por completo la fricción operativa al suprimir los paneles web complejos y conectar dos canales de comunicación de uso diario:

* **Canal de Clientes (WhatsApp):** Los clientes interactúan enviando ráfagas de texto, audios de voz y fotos de referencia. Los mensajes entrantes se consolidan a través de un buffer en memoria con ventana deslizante de 30 segundos antes de ejecutar el ciclo del agente ReAct.
* **Consola de Operaciones para el Comerciante (Supergrupo de Telegram con Temas/Foros):** El dueño y sus colaboradores gestionan el negocio sin instalar software propietario adicional:
  - **`#Dudas-Clientes`:** Handoff de soporte humano; el personal responde utilizando citas nativas de Telegram (`reply_to_message`).
  - **`#Mi-Catálogo`:** Comandos de voz o texto para modificar precios y existencias en el catálogo de Clip.
  - **`#Ventas-y-Caja`:** Tarjetas de pedidos interactivas en tiempo real con botones en línea que mutan en el mismo mensaje (`editMessageText`).

---

## 2. Índice Maestro de Documentación

Esta suite contiene la documentación técnica completa requerida por una empresa de desarrollo de software para cotizar, planificar e implementar el sistema:

| Documento | Ruta del Archivo | Enfoque y Contenido Principal |
| :--- | :--- | :--- |
| **Arquitectura de Software** | [`ARCHITECTURE.md`](./ARCHITECTURE.md) | Topología general, diagramas de componentes, máquinas de estado finito (FSM), aislamiento multi-tenant y extracción asíncrona de memoria EAV. |
| **Stack Tecnológico** | [`TECH_STACK.md`](./TECH_STACK.md) | Runtime (Node.js 20+ LTS), motor Baileys puro, PocketBase v0.23+, OmniRoute AI Gateway, GramMY, Litestream y Cloudflare Tunnels. |
| **Esquema de Base de Datos** | [`DATABASE_SCHEMA.md`](./DATABASE_SCHEMA.md) | Modelo relacional estricto Zero-JSON en PocketBase, 11 colecciones normalizadas, índices compuestos, reglas de seguridad y hooks nativos. |
| **Integraciones Externas** | [`INTEGRATIONS.md`](./INTEGRATIONS.md) | Contratos de 3 dominios de PayClip, regla GET-before-PATCH para proteger el stock, verificación Zero-Trust de Webhooks, telemetría de 14 campos y enrutamiento en Telegram. |
| **Registro de Tools y Presets** | [`TOOL_REGISTRY_AND_PRESETS.md`](./TOOL_REGISTRY_AND_PRESETS.md) | Ciclo ReAct (guardarraíl de 5 iteraciones), esquemas Zod, plantillas verticales (`FOOD_RETAIL`, `SERVICE_APPOINTMENTS`, etc.) y compilador modular de prompts en 5 capas. |
| **Hoja de Ruta (Roadmap)** | [`ROADMAP.md`](./ROADMAP.md) | 5 sprints de dos semanas (10 semanas a producción v1.0), hitos, matriz de riesgos y Criterios de Aceptación formales (Definition of Done). |
| **Plan de Implementación** | [`IMPLEMENTATION_PLAN.md`](./IMPLEMENTATION_PLAN.md) | Árbol modular de directorios, desglose atómico de 10 tareas de construcción, blueprints de código crítico y estrategia de testing con mocks. |

---

## 3. Siete Decisiones de Arquitectura No Negociables

1. **Baileys Nativo Directo (Sin el Bloat de BuilderBot):** Uso directo de `@whiskeysockets/baileys` sobre WebSockets (~35MB RAM por sesión). Emulación de presencia orgánica (delay de lectura de 800-1500ms, estado `composing` dinámico según la longitud del texto) y tráfico 100% entrante (inbound) para prevenir baneos de WhatsApp.
2. **Buffer con Debounce de 30 Segundos en Memoria:** Cada mensaje entrante por hilo `tenant:customer` reinicia un temporizador deslizante de 30,000 ms. Al vencerse el tiempo, se transcriben en paralelo los audios con Groq Whisper (`whisper-large-v3`, `language: 'es'`) y se unifica el texto en un único turno de usuario.
3. **Política Estricta Zero-JSON en Base de Datos:** PocketBase (SQLite en modo WAL) almacena todos los datos en tablas relacionales tipadas con claves foráneas e índices compuestos, eliminando columnas de tipo JSON.
4. **Patrón EAV para Memoria a Largo Plazo:** Los rasgos y preferencias del cliente (alergias, sabores favoritos, tallas) se almacenan en `customer_traits` con categorías validadas por enums del LLM y se actualizan en segundo plano (*fire-and-forget* con `setImmediate`) después de contestarle al cliente.
5. **Gateway de IA Autohospedado (OmniRoute):** Endpoint unificado compatible con OpenAI (`http://localhost:20128/v1`) con balanceo consciente de cuotas: ejecución primaria en Groq (`llama-3.3-70b-versatile`) con conmutación automática ante errores 429 a Google Gemini (`gemini-2.0-flash`) para Tool Calling y análisis multimodal de fotos enviadas por clientes.
6. **Protección de Stock en Catálogo de Clip (`GET-before-PATCH`):** Al actualizar precio o título mediante `PATCH /f2f/catalog/products/:id`, el sistema debe consultar previamente el producto mediante `GET` e inyectar el parámetro `stock` vigente para evitar que Clip lo restablezca a `null`.
7. **Verificación Zero-Trust de Webhooks:** Los webhooks entrantes de PayClip Checkout reciben un Fast-ACK inmediato con HTTP 200 (<1s). La orden solo se marca como `PAID` tras una consulta directa en segundo plano a `GET https://api.payclip.com/v2/checkout/{id}` con Basic Auth para verificar estatus y montos exactos.

---

## 4. Guía Rápida para Desarrolladores

### Prerrequisitos
* Node.js v20.18.0+ LTS y `pnpm`
* Binario de PocketBase v0.23+
* Instancia local de OmniRoute ejecutándose en el puerto `20128`

### Instrucciones de Instalación
```bash
# 1. Instalar dependencias
cd local-commerce-ai-saas
pnpm install

# 2. Configurar variables de entorno
cp .env.example .env
# Configura MASTER_ENCRYPTION_KEY, TELEGRAM_BOT_TOKEN y credenciales de PocketBase

# 3. Iniciar PocketBase
./pocketbase serve --http=127.0.0.1:8090

# 4. Ejecutar migraciones de base de datos
pnpm run db:migrate

# 5. Iniciar OmniRoute Gateway
omniroute start

# 6. Iniciar el motor del backend
pnpm run dev
```
