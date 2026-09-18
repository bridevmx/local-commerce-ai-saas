# Especificación de Integraciones Externas
## PayClip, Telegram Bot API y OmniRoute Gateway

**Versión:** 1.0.0  
**Fecha:** Marzo 2025  

---

## 1. Matriz de Dominios y Autenticación de PayClip

La API de Clip se distribuye en tres dominios con esquemas de autenticación diferenciados:

| Dominio | Esquema de URL | Cabecera de Autenticación | Funcionalidades Soportadas |
| :--- | :--- | :--- | :--- |
| **Clip API (F2F)** | `https://api.clip.mx` | `Authorization: Bearer <token>` | Catálogo F2F, consulta de transacciones en terminales físicas. |
| **Clip IO (Checkout)**| `https://api.payclip.com` | `Authorization: Basic <base64(key:secret)>` | Creación de Enlaces de Pago (Payment Links) y Checkout API. |
| **Clip API-GW (Telemetría)**| `https://api-gw.payclip.com`| `Authorization: Bearer <token>` | Eventos de telemetría, pagos en efectivo y comisiones de hardware. |

---

## 2. Regla Obligatoria `GET-before-PATCH` para Protección de Catálogo

### El Peligro del Sobrescrito Inadvertido
En el endpoint `PATCH /f2f/catalog/products/:id` de Clip, cualquier propiedad no enviada en el cuerpo de la petición puede ser reseteada a su valor por defecto por el backend de Clip. Específicamente, omitir la propiedad `stock` al modificar el precio o el nombre de un artículo provoca que el inventario se reinicie a `null`, deshabilitando el control de existencias.

### Contrato de Implementación
Toda mutación de producto ejecutada por el agente o el comerciante debe cumplir de forma estricta con el flujo:
1. Realizar una consulta previa: `GET https://api.clip.mx/f2f/catalog/products/:id`.
2. Conservar el valor exacto de `stock`, `sku`, `categories` y propiedades no mutadas.
3. Enviar el `PATCH` inyectando tanto el nuevo valor deseado como el estado vigente de `stock`.

```javascript
// src/integrations/clip/catalog.js
export async function updateCatalogProductSafe(tenantId, productId, updateFields) {
  const creds = await getTenantCredentials(tenantId, 'CLIP_API');
  
  // Paso 1: GET del estado actual obligatorio
  const current = await fetch(`https://api.clip.mx/f2f/catalog/products/${productId}`, {
    headers: { 'Authorization': `Bearer ${creds.token}` }
  }).then(r => r.json());

  // Paso 2: Fusión defensiva inyectando siempre el stock actual
  const payload = {
    ...current,
    ...updateFields,
    stock: updateFields.stock !== undefined ? updateFields.stock : current.stock
  };

  // Paso 3: Enviar PATCH seguro
  return await fetch(`https://api.clip.mx/f2f/catalog/products/${productId}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${creds.token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(payload)
  }).then(r => r.json());
}
```

---

## 3. Verificación Zero-Trust de Webhooks de PayClip

Para garantizar que un cliente malicioso no suplante webhooks ni falsifique pagos:

1. **Fast-ACK Inmediato (< 1s):** El endpoint público responde con HTTP 200 de manera instantánea tras recibir el payload para evitar reintentos agresivos del servidor de Clip.
2. **Descarte del Payload No Verificado:** Los datos recibidos en el cuerpo del webhook se consideran no confiables. No se actualiza el estado de la orden basándose únicamente en el contenido de la notificación.
3. **Consulta Directa en Segundo Plano (API Readback):** El backend extrae el `id` o `payment_request_id` del checkout y realiza inmediatamente una llamada autenticada a:
   ```http
   GET https://api.payclip.com/v2/checkout/{id}
   Authorization: Basic <base64(key:secret)>
   ```
4. **Validación Criptográfica y de Montos:** Solo cuando la respuesta oficial de la API de Clip confirma `status: "PAID"` y el monto coincide al centavo con `orders.total_amount`, se transiciona la orden a `PAID`.
5. **Notificación en Telegram:** Se muta la tarjeta del pedido en el tópico `#Ventas-y-Caja` actualizando los botones de acción a "Confirmar Preparación".

---

## 4. Telemetría de 14 Campos para Pagos F2F en Efectivo

Para comercios que combinan cobros digitales con pagos en efectivo registrados en terminales Clip o caja física, el sistema estructura una carga de telemetría normalizada de 14 campos:

| # | Campo de Telemetría | Tipo | Descripción |
| :- | :--- | :--- | :--- |
| 1 | `transaction_id` | `UUID/String` | Identificador único de la transacción en caja. |
| 2 | `tenant_id` | `String` | ID del comercio en PocketBase. |
| 3 | `timestamp_iso` | `ISO8601` | Marca de tiempo exacta del cobro. |
| 4 | `payment_method` | `'CASH'` | Constante indicadora de efectivo presencial. |
| 5 | `gross_amount` | `Number` | Monto bruto cobrado al cliente. |
| 6 | `currency` | `'MXN'` | Divisa local mexicana. |
| 7 | `cash_received` | `Number` | Monto de efectivo entregado por el cliente. |
| 8 | `change_given` | `Number` | Cambio o vuelto entregado (`cash_received - gross_amount`). |
| 9 | `cashier_user_id` | `String` | ID de Telegram o nombre del colaborador que operó la caja. |
| 10| `device_terminal_id` | `String` | Número de serie o alias de la terminal Clip utilizada (si aplica). |
| 11| `customer_phone` | `String` | Teléfono del cliente si se identificó en el sistema. |
| 12| `order_id` | `String` | Referencia a la orden en la colección `orders`. |
| 13| `items_count` | `Integer` | Suma total de unidades de productos en la transacción. |
| 14| `tax_amount` | `Number` | Monto de impuesto desglosado (IVA 16%). |

---

## 5. Consola Operativa en Telegram (Supergrupo con Foros)

Cada comercio local se gestiona a través de un Supergrupo de Telegram con topics (foros) habilitados:

### Aislamiento por Supergrupo
El bot valida el campo `tenants.telegram_chat_id`. Si un mensaje proviene de un chat no registrado, es ignorado de inmediato.

### Estructura de Tópicos (Message Threads)
* **Tópico `#Dudas-Clientes` (Handoff de Soporte):**
  - Cuando el cliente solicita un humano o el LLM agota sus 5 iteraciones ReAct, el bot emite una alerta en este tópico incluyendo un resumen de la duda, el historial reciente del cliente y su número de WhatsApp.
  - El personal del comercio responde directamente usando la función **Citar / Responder (`reply_to_message`)** nativa de Telegram.
  - El bot captura el mensaje citado, extrae el `session_id` o `customer_id` y despacha el texto como un mensaje saliente a través del WebSocket de Baileys en WhatsApp.
* **Tópico `#Mi-Catálogo` (Mutación de Inventario y Precios):**
  - El comerciante puede enviar audios o textos como: *"Sube el pastel de chocolate a 350 pesos y pon 5 unidades en stock"*.
  - El bot procesa el audio con Whisper, genera los parámetros estructurados y ejecuta la regla `GET-before-PATCH` contra Clip.
  - Devuelve un mensaje de confirmación con teclado en línea para revertir el cambio si fue un error.
* **Tópico `#Ventas-y-Caja` (Tarjetas Interactivas de Pedido):**
  - Cuando entra un pedido o se confirma un pago, se publica una tarjeta interactiva con botones en línea:
    `[ Aceptar Pedido ]` `[ Marcar Listo ]` `[ Cancelar ]`.
  - Al presionar un botón, el bot muta el mensaje existente mediante `editMessageText` sin inundar el canal con mensajes nuevos.

---

## 6. Integración del Gateway OmniRoute

El backend interactúa con OmniRoute en `http://localhost:20128/v1` mediante el SDK de OpenAI:

```javascript
import OpenAI from 'openai';

export const aiGateway = new OpenAI({
  baseURL: process.env.AI_GATEWAY_URL || 'http://localhost:20128/v1',
  apiKey: process.env.AI_GATEWAY_API_KEY || 'omniroute-local-token'
});
```

* **Prioridad:** Llama 3.3 70B en Groq para razonamiento general.
* **Fallback Transparente:** Manejado a nivel de proxy por OmniRoute ante errores `429` hacia Gemini 2.0 Flash.
* **Procesamiento de Fotos:** El backend detecta `image_url` en el arreglo de mensajes y selecciona explícitamente `google/gemini-2.0-flash` como modelo para garantizar alta precisión en visión computacional.
