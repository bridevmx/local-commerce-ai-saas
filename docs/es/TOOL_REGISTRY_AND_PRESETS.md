# Registro de Herramientas, Presets y Compilador de Prompts
## Motor ReAct y Personalización Modular en 5 Capas

**Versión:** 1.0.0  
**Fecha:** Marzo 2025  

---

## 1. Arquitectura del Bucle ReAct y Guardarraíl de Iteraciones

El motor del agente ejecuta un patrón de razonamiento y acción (ReAct) con un guardarraíl estricto de **máximo 5 iteraciones**:

```javascript
// Pseudocódigo del Bucle ReAct en src/agent/react-loop.js
const MAX_ITERATIONS = 5;
let iterations = 0;

while (iterations < MAX_ITERATIONS) {
  iterations++;
  const response = await aiGateway.chat.completions.create({
    model: selectModelForContext(context),
    messages: conversationHistory,
    tools: tenantActiveTools,
    tool_choice: "auto"
  });

  const choice = response.choices[0];
  if (!choice.message.tool_calls || choice.message.tool_calls.length === 0) {
    // Respuesta final alcanzada
    return choice.message.content;
  }

  // Ejecución de llamadas a herramientas
  for (const toolCall of choice.message.tool_calls) {
    const result = await executeTool(tenantId, toolCall);
    conversationHistory.push({
      role: "tool",
      tool_call_id: toolCall.id,
      content: JSON.stringify(result)
    });

    if (toolCall.function.name === "escalate_to_human") {
      return "He transferido tu duda con nuestro equipo en tienda. En breve un colaborador te responderá directamente por aquí.";
    }
  }
}

// Failsafe tras agotar iteraciones
await triggerHumanEscalation(tenantId, customerId, "MAX_ITERATIONS_EXCEEDED");
return "Disculpa la demora, estoy consultando un detalle con nuestro equipo en tienda para brindarte la información exacta. En unos instantes te atendemos.";
```

---

## 2. Registro de Herramientas del Agente con Esquemas Zod

Cada herramienta disponible para el LLM está tipada y validada mediante esquemas de Zod antes de su ejecución:

### 2.1. `clip_check_catalog`
Consulta el catálogo y las existencias de productos en tiempo real.
```javascript
import { z } from 'zod';

export const ClipCheckCatalogSchema = z.object({
  query: z.string().describe("Nombre o palabra clave del producto a buscar (ej: 'pastel de chocolate', 'corte clasico')"),
  category: z.string().optional().describe("Filtro opcional por categoría del catálogo")
});
```

### 2.2. `clip_create_payment`
Genera un enlace de pago oficial de Clip Checkout.
```javascript
export const ClipCreatePaymentSchema = z.object({
  order_id: z.string().describe("ID de la orden registrada en la colección orders de PocketBase"),
  amount: z.number().positive().describe("Monto exacto a cobrar en pesos mexicanos (MXN)"),
  description: z.string().describe("Concepto breve del pedido para el recibo de compra")
});
```

### 2.3. `escalate_to_human`
Transfiere la conversación al Supergrupo de Telegram para atención por personal del comercio.
```javascript
export const EscalateToHumanSchema = z.object({
  reason: z.string().describe("Causa concisa de la escalación (ej: 'Solicitud de diseño de pastel personalizado no listado')"),
  urgency: z.enum(["LOW", "MEDIUM", "HIGH"]).default("MEDIUM").describe("Nivel de prioridad del caso")
});
```

### 2.4. `register_order_lead`
Crea una orden preliminar en estado `DRAFT` con sus partidas desglosadas.
```javascript
export const RegisterOrderLeadSchema = z.object({
  items: z.array(z.object({
    product_id: z.string().describe("Identificador de producto"),
    product_name: z.string().describe("Nombre legible del artículo"),
    unit_price: z.number().positive().describe("Precio unitario"),
    quantity: z.number().int().positive().describe("Cantidad de unidades")
  })).min(1).describe("Listado de artículos del pedido"),
  notes: z.string().optional().describe("Indicaciones especiales de preparación o entrega")
});
```

### 2.5. `check_availability`
Verifica horarios libres para citas y reservaciones de servicios.
```javascript
export const CheckAvailabilitySchema = z.object({
  service_type: z.string().describe("Nombre del servicio (ej: 'Corte y Barba', 'Consulta de Valoración')"),
  preferred_date: z.string().regex(/^\d{4}-\d{2}-\d{2}$/).describe("Fecha en formato YYYY-MM-DD"),
  preferred_time_range: z.enum(["MORNING", "AFTERNOON", "EVENING"]).optional().describe("Preferencia de horario")
});
```

---

## 3. Plantillas de Prompts y Presets por Vertical de Negocio

El sistema incluye presets optimizados para sectores clave:

### 3.1. Preset `FOOD_RETAIL` (Pastelerías, Cafeterías, Restaurantes)
* **Tono:** Cálido, servicial y enfocado en el apetito visual.
* **Directivas de Venta Sugestiva:** Siempre indagar sobre ocasiones especiales, alergias a ingredientes y ofrecer complementos lógicos (p. ej., velitas de cumpleaños, bebidas frías).
* **Manejo de Stock:** No prometer pedidos con entrega el mismo día si el producto requiere horneado especial sin validar existencias.

### 3.2. Preset `SERVICE_APPOINTMENTS` (Barberías, Spas, Clínicas Boutique)
* **Tono:** Profesional, ágil y organizado.
* **Directivas:** Identificar duración de turnos, sugerir opciones de horario concretas (máximo 2 alternativas a la vez) y confirmar nombre y teléfono del titular antes de bloquear la agenda.

### 3.3. Preset `SUPPORT_LEAD` (Ventas Técnicas y Comercio General)
* **Tono:** Directo, resolutivo y transparente con precios y métodos de entrega.
* **Directivas:** Responder dudas frecuentes de envíos y calificar el interés de compra de forma no invasiva.

---

## 4. El Compilador Modular de Prompts de 5 Capas

El System Prompt nunca es un texto estático; se ensambla dinámicamente en tiempo de ejecución para cada turno conversacional integrando 5 capas jerárquicas:

```
+-------------------------------------------------------------------------------+
| CAPA 1: Guardarraíles Universales de Seguridad y Anti-Alucinación             |
| - Prohibición de inventar precios, descuentos no autorizados o existencias    |
| - Regla de una sola pregunta a la vez para mantener fluidez conversacional    |
| - Detección de intentos de Prompt Injection o manipulación de roles           |
+-------------------------------------------------------------------------------+
                                       |
                                       v
+-------------------------------------------------------------------------------+
| CAPA 2: Preset de Industria y Vertical de Negocio                             |
| - Reglas de dominio ('FOOD_RETAIL', 'SERVICE_APPOINTMENTS', etc.)             |
+-------------------------------------------------------------------------------+
                                       |
                                       v
+-------------------------------------------------------------------------------+
| CAPA 3: Identidad y Datos Operativos del Comercio (Tenants)                   |
| - Nombre del negocio, dirección física, horarios de apertura y zonas de envío|
+-------------------------------------------------------------------------------+
                                       |
                                       v
+-------------------------------------------------------------------------------+
| CAPA 4: Reglas Atómicas de Negocio del Comercio (`tenant_rules`)              |
| - Políticas dictadas por el dueño en Telegram (prioridad 1 a N)               |
| - Ejemplo: "Los domingos solo atendemos pedidos de pastelería sobre encargo"  |
+-------------------------------------------------------------------------------+
                                       |
                                       v
+-------------------------------------------------------------------------------+
| CAPA 5: Memoria a Largo Plazo del Cliente (`customer_traits`)                 |
| - Rasgos EAV recuperados de interacciones previas                             |
| - Ejemplo: "Cliente alérgico a las nueces; prefiere pagar con tarjeta Clip"   |
+-------------------------------------------------------------------------------+
```

### Implementación del Compilador
```javascript
// src/agent/prompt-compiler.js
export async function compileSystemPrompt(tenantId, customerId) {
  const [tenant, rules, traits] = await Promise.all([
    pb.collection('tenants').getOne(tenantId),
    pb.collection('tenant_rules').getFullList({
      filter: `tenant = "${tenantId}" && is_active = true`,
      sort: 'priority'
    }),
    customerId ? pb.collection('customer_traits').getFullList({
      filter: `customer = "${customerId}"`
    }) : []
  ]);

  const layer1 = getUniversalGuardrails();
  const layer2 = getVerticalPreset(tenant.vertical);
  const layer3 = `Eres el asistente oficial de WhatsApp de "${tenant.name}".`;
  
  const layer4 = rules.length > 0
    ? "POLÍTICAS Y REGLAS ESPECÍFICAS DE ESTA TIENDA:\n" + rules.map(r => `- [${r.category}] ${r.instruction}`).join('\n')
    : "";

  const layer5 = traits.length > 0
    ? "INFORMACIÓN PERSONALIZADA DEL CLIENTE CON EL QUE HABLAS:\n" + traits.map(t => `- ${t.trait_key}: ${t.trait_value}`).join('\n')
    : "";

  return [layer1, layer2, layer3, layer4, layer5].filter(Boolean).join("\n\n");
}
```
