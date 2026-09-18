# Especificación de Esquema de Base de Datos
## Modelo Relacional Estricto Zero-JSON en PocketBase v0.23+

**Versión:** 1.0.0  
**Fecha:** Marzo 2025  

---

## 1. Principio Fundamental: Modelo Relacional Estricto Zero-JSON

Bajo la arquitectura de este sistema, **está estrictamente prohibido utilizar columnas de tipo `json` en PocketBase** para almacenar lógica de negocio, carritos de compra, rasgos de clientes o metadatos dinámicos. 

### Justificación Técnica:
1. **Integridad Referencial y Claves Foráneas:** SQLite y PocketBase garantizan integridad referencial nativa mediante relaciones tipadas. Los campos JSON impiden que la base de datos valide claves foráneas, facilitando registros huérfanos y corrupción lógica.
2. **Indexación y Rendimiento:** Los campos atómicos e indexados por columnas individuales permiten búsquedas exactas con costo $O(\log N)$, mientras que la búsqueda de atributos dentro de strings JSON requiere escaneo secuencial de tablas ($O(N)$) o funciones JSON virtuales no portables.
3. **Control Estricto de Concurrencia:** La mutación de un solo ítem dentro de un documento JSON requiere bloquear y reescribir todo el registro, provocando condiciones de carrera. La normalización en tablas atómicas permite mutaciones concurrentes a nivel de fila.

---

## 2. Diagrama Entidad-Relación (ERD)

```
+-------------------+       1:N       +----------------------+
|      tenants      |----------------<|  tenant_credentials  |
+-------------------+                 +----------------------+
  | 1:N        | 1:N
  |            +--------------------< +----------------------+
  |                                   |     tenant_rules     |
  | 1:N                               +----------------------+
  |
  |            +--------------------+ 1:N +------------------+
  +-----------<|     customers      |----<| customer_traits  |
  |            +--------------------+     +------------------+
  |              | 1:N         | 1:N
  |              v             v
  |            +----------+  +----------+
  +-----------<| sessions |  | tickets  |
  |            +----------+  +----------+
  |              | 1:N
  |              v
  |            +----------+
  +-----------<| messages |
  |            +----------+
  |
  | 1:N        +----------+ 1:N       +----------------------+
  +-----------<|  orders  |----------<|     order_items      |
               +----------+           +----------------------+
```

---

## 3. Especificación Detallada de las 11 Colecciones Normalizadas

### 3.1. `tenants` (Comercios Afiliados)
Almacena la identidad y configuración base de cada comercio local.

| Campo | Tipo | Restricciones | Descripción |
| :--- | :--- | :--- | :--- |
| `id` | `TEXT(15)` | Primary Key | ID único generado por PocketBase. |
| `name` | `TEXT` | Required | Nombre comercial del negocio. |
| `slug` | `TEXT` | Required, Unique, Indexed | Identificador alfanumérico amigable para URLs. |
| `vertical` | `SELECT` | Required | `'FOOD_RETAIL'`, `'SERVICE_APPOINTMENTS'`, `'SUPPORT_LEAD'`, `'GENERIC_RETAIL'`. |
| `whatsapp_phone`| `TEXT` | Required, Unique | Número telefónico internacional asignado a la instancia de WhatsApp. |
| `telegram_chat_id`| `TEXT` | Required, Unique, Indexed | ID numérico del Supergrupo de Telegram del comercio. |
| `is_active` | `BOOL` | Required, Default: true | Bandera de activación operativa del comercio. |
| `created` | `DATETIME` | System | Fecha de registro en el sistema. |
| `updated` | `DATETIME` | System | Fecha de última modificación. |

### 3.2. `tenant_credentials` (Almacén de Claves Cifradas AES-256-GCM)
Almacena llaves de API y credenciales de integración de manera segura y segregada.

| Campo | Tipo | Restricciones | Descripción |
| :--- | :--- | :--- | :--- |
| `id` | `TEXT(15)` | Primary Key | ID único generado por PocketBase. |
| `tenant` | `RELATION` | Required, Cascade Delete, Ref: `tenants` | Comercio propietario de la credencial. |
| `service_name` | `SELECT` | Required | `'CLIP_API'`, `'CLIP_IO'`, `'CLIP_API_GW'`, `'TELEGRAM'`. |
| `encrypted_key` | `TEXT` | Required | Cadena cifrada en Base64/Hex mediante AES-256-GCM. |
| `iv` | `TEXT` | Required | Vector de Inicialización (12 bytes en Hex). |
| `auth_tag` | `TEXT` | Required | Etiqueta de Autenticación GCM (16 bytes en Hex). |
| `expires_at` | `DATETIME` | Optional | Fecha de caducidad para credenciales rotativas. |

*Índice Compuesto Único:* `(tenant, service_name)`.

### 3.3. `tenant_rules` (Reglas Atómicas de Negocio para el Prompt)
Reglas operativas y políticas dictadas por el dueño del comercio para el agente ReAct.

| Campo | Tipo | Restricciones | Descripción |
| :--- | :--- | :--- | :--- |
| `id` | `TEXT(15)` | Primary Key | ID único generado por PocketBase. |
| `tenant` | `RELATION` | Required, Cascade Delete, Ref: `tenants` | Comercio al que pertenece la regla. |
| `category` | `SELECT` | Required | `'SCHEDULE'`, `'POLICY'`, `'PAYMENT'`, `'SHIPPING'`, `'CUSTOM'`. |
| `rule_key` | `TEXT` | Required | Identificador único de la regla dentro del comercio (p. ej., `'horario_domingo'`). |
| `instruction` | `TEXT` | Required | Directiva precisa en lenguaje natural inyectada en el System Prompt. |
| `priority` | `NUMBER` | Required, Default: 10 | Orden de inyección en el prompt compilado (números menores van primero). |
| `is_active` | `BOOL` | Required, Default: true | Permite deshabilitar reglas temporalmente sin eliminarlas. |

*Índice Compuesto Único:* `(tenant, rule_key)`.

### 3.4. `tenant_tools` (Catálogo de Herramientas Habilitadas por Comercio)
Controla qué herramientas del agente ReAct tiene activadas cada comercio local.

| Campo | Tipo | Restricciones | Descripción |
| :--- | :--- | :--- | :--- |
| `id` | `TEXT(15)` | Primary Key | ID único generado por PocketBase. |
| `tenant` | `RELATION` | Required, Cascade Delete, Ref: `tenants` | Comercio que habilita la herramienta. |
| `tool_name` | `SELECT` | Required | `'clip_check_catalog'`, `'clip_create_payment'`, `'escalate_to_human'`, etc. |
| `is_enabled` | `BOOL` | Required, Default: true | Bandera de activación de la herramienta. |
| `custom_prompt` | `TEXT` | Optional | Instrucciones de uso específicas de la tool para este comercio. |

*Índice Compuesto Único:* `(tenant, tool_name)`.

### 3.5. `customers` (Directorio de Clientes)
Registro maestro de clientes que interactúan con cualquiera de los comercios.

| Campo | Tipo | Restricciones | Descripción |
| :--- | :--- | :--- | :--- |
| `id` | `TEXT(15)` | Primary Key | ID único generado por PocketBase. |
| `tenant` | `RELATION` | Required, Cascade Delete, Ref: `tenants` | Comercio donde interactúa el cliente. |
| `phone_number` | `TEXT` | Required | Número telefónico internacional en formato E.164. |
| `full_name` | `TEXT` | Optional | Nombre del cliente obtenido de WhatsApp o deducido por el agente. |
| `first_contact` | `DATETIME` | Required | Fecha y hora del primer mensaje recibido. |
| `last_contact` | `DATETIME` | Required | Fecha y hora de la última interacción registrada. |

*Índice Compuesto Único:* `(tenant, phone_number)`.

### 3.6. `customer_traits` (Memoria a Largo Plazo EAV de Clientes)
Patrón Entidad-Atributo-Valor para almacenar preferencias y hábitos de clientes sin JSON.

| Campo | Tipo | Restricciones | Descripción |
| :--- | :--- | :--- | :--- |
| `id` | `TEXT(15)` | Primary Key | ID único generado por PocketBase. |
| `customer` | `RELATION` | Required, Cascade Delete, Ref: `customers` | Cliente al que se asocia la memoria. |
| `category` | `SELECT` | Required | `'dietary_preference'`, `'favorite_item'`, `'delivery_address'`, `'pet_breed'`, `'general_note'`. |
| `trait_key` | `TEXT` | Required | Clave del atributo (p. ej., `'alergia_frutos_secos'`, `'direccion_casa'`). |
| `trait_value` | `TEXT` | Required | Valor en texto plano (p. ej., `'Alérgico a nueces'`, `'Calle Hidalgo 45'`). |
| `confidence` | `NUMBER` | Required, Default: 1.0 | Puntuación de certeza inferida por el modelo (0.1 a 1.0). |

*Índice Compuesto Único:* `(customer, category, trait_key)`.

### 3.7. `sessions` (Sesiones de Conversación)
Agrupa intercambios de mensajes en ventanas de conversación de 24 horas.

| Campo | Tipo | Restricciones | Descripción |
| :--- | :--- | :--- | :--- |
| `id` | `TEXT(15)` | Primary Key | ID único generado por PocketBase. |
| `tenant` | `RELATION` | Required, Ref: `tenants` | Comercio de la conversación. |
| `customer` | `RELATION` | Required, Cascade Delete, Ref: `customers` | Cliente titular de la sesión. |
| `is_active` | `BOOL` | Required, Default: true | Bandera de estado activo de la conversación. |
| `started_at` | `DATETIME` | Required | Marca de tiempo de apertura de sesión. |
| `ended_at` | `DATETIME` | Optional | Marca de tiempo de cierre de sesión. |

### 3.8. `tickets` (Casos de Soporte y Escalación Humana)
Gestiona el handoff entre la inteligencia artificial y el personal del comercio.

| Campo | Tipo | Restricciones | Descripción |
| :--- | :--- | :--- | :--- |
| `id` | `TEXT(15)` | Primary Key | ID único generado por PocketBase. |
| `tenant` | `RELATION` | Required, Ref: `tenants` | Comercio donde se generó el ticket. |
| `customer` | `RELATION` | Required, Ref: `customers` | Cliente asociado al ticket. |
| `session` | `RELATION` | Optional, Ref: `sessions` | Sesión conversacional en la que se generó la duda. |
| `status` | `SELECT` | Required | `'OPEN'`, `'IN_PROGRESS'`, `'ESCALATED'`, `'RESOLVED'`, `'CLOSED'`. |
| `reason` | `TEXT` | Required | Causa de la escalación resumida por el agente ReAct. |
| `telegram_thread_id`| `NUMBER` | Optional | ID del mensaje raíz en el tópico de Telegram para soporte. |

### 3.9. `messages` (Historial Cronológico de Mensajes)
Registro de auditoría de cada mensaje intercambiado por WhatsApp o Telegram.

| Campo | Tipo | Restricciones | Descripción |
| :--- | :--- | :--- | :--- |
| `id` | `TEXT(15)` | Primary Key | ID único generado por PocketBase. |
| `session` | `RELATION` | Required, Cascade Delete, Ref: `sessions` | Sesión a la que pertenece el mensaje. |
| `sender_type` | `SELECT` | Required | `'CUSTOMER'`, `'AI_AGENT'`, `'HUMAN_OPERATOR'`. |
| `content_type` | `SELECT` | Required | `'TEXT'`, `'AUDIO_TRANSCRIPTION'`, `'IMAGE_DESCRIPTION'`. |
| `body` | `TEXT` | Required | Contenido legible en texto del mensaje. |
| `media_url` | `TEXT` | Optional | Ruta o referencia del archivo multimedia en almacenamiento local. |
| `created` | `DATETIME` | System | Marca de tiempo exacta del envío o recepción. |

### 3.10. `orders` (Órdenes de Compra)
Encabezado relacional de pedidos generados en el comercio.

| Campo | Tipo | Restricciones | Descripción |
| :--- | :--- | :--- | :--- |
| `id` | `TEXT(15)` | Primary Key | ID único generado por PocketBase. |
| `tenant` | `RELATION` | Required, Ref: `tenants` | Comercio receptor de la orden. |
| `customer` | `RELATION` | Required, Ref: `customers` | Cliente comprador. |
| `status` | `SELECT` | Required | `'DRAFT'`, `'PENDING_PAYMENT'`, `'PAID'`, `'CANCELLED'`, `'FULFILLED'`. |
| `total_amount` | `NUMBER` | Required | Monto total de la venta en centavos o decimales de precisión fija. |
| `currency` | `TEXT` | Required, Default: `'MXN'` | Código de divisa ISO 4217. |
| `clip_payment_id`| `TEXT` | Optional, Indexed | ID de transacción o checkout devuelto por la API de Clip. |
| `clip_payment_url`| `TEXT` | Optional | Enlace público de pago generado para el cliente. |
| `telegram_message_id`| `NUMBER` | Optional | ID del mensaje de la tarjeta interactiva en Telegram para mutación (`editMessageText`). |

### 3.11. `order_items` (Partidas Atómicas de Órdenes)
Líneas individuales de productos asociadas a una orden, eliminando listas JSON.

| Campo | Tipo | Restricciones | Descripción |
| :--- | :--- | :--- | :--- |
| `id` | `TEXT(15)` | Primary Key | ID único generado por PocketBase. |
| `order` | `RELATION` | Required, Cascade Delete, Ref: `orders` | Orden principal a la que pertenece la partida. |
| `product_id` | `TEXT` | Required | Identificador de producto en el catálogo de Clip o catálogo local. |
| `product_name` | `TEXT` | Required | Nombre comercial del producto al momento de la venta. |
| `unit_price` | `NUMBER` | Required | Precio unitario aplicado en la transacción. |
| `quantity` | `NUMBER` | Required | Número entero de unidades solicitadas. |
| `subtotal` | `NUMBER` | Required | Cálculo exacto: `unit_price * quantity`. |

---

## 4. Reglas de API y Permisos de Seguridad

Para garantizar un modelo de seguridad cerrado:

1. **Reglas de Acceso Público:** Todas las reglas de API (`List`, `View`, `Create`, `Update`, `Delete`) de las 11 colecciones se configuran como **`null`** (acceso público totalmente bloqueado).
2. **Autenticación Exclusiva del Backend:** Solo el proceso backend en Node.js accede a PocketBase utilizando autenticación administrativa con privilegios de Superusuario (`pb.admins.authWithPassword`).
3. **Imposibilidad de Manipulación Externa:** Ningún cliente web, bot o tercero puede consultar ni mutar registros sin pasar por la validación y mediación del backend del SaaS.

---

## 5. Estrategia de Migraciones en JavaScript de PocketBase

Las migraciones se ejecutan mediante scripts declarativos en `pb_migrations/` para mantener el control de versiones en Git:

```javascript
// pb_migrations/1710000001_init_schema.js
migrate((app) => {
  const tenants = new Collection({
    name: "tenants",
    type: "base",
    schema: [
      { name: "name", type: "text", required: true },
      { name: "slug", type: "text", required: true, unique: true },
      { name: "vertical", type: "select", options: { values: ["FOOD_RETAIL", "SERVICE_APPOINTMENTS", "SUPPORT_LEAD", "GENERIC_RETAIL"] }, required: true },
      { name: "whatsapp_phone", type: "text", required: true, unique: true },
      { name: "telegram_chat_id", type: "text", required: true, unique: true },
      { name: "is_active", type: "bool", required: true }
    ],
    listRule: null,
    viewRule: null,
    createRule: null,
    updateRule: null,
    deleteRule: null
  });

  return app.save(tenants);
}, (app) => {
  const collection = app.findCollectionByNameOrId("tenants");
  return app.delete(collection);
});
```
