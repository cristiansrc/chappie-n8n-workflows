# Master Spec - chappie-n8n-workflows

**Lifecycle status:** `Active`  
**Owner:** Planner  
**Proyecto:** `projects/chappie-n8n-workflows/`  
**Bounded Contexts:** Orchestration, Memory  
**Workspace:** chappie-workspace (Solution Workspace)  
**Ultima actualizacion:** 2026-06-14 (remediacion F-001, F-005, F-006, F-007)  
**Fuentes autoritativas consultadas:**
- `chappie-workspace/docs/specs/master_spec.md` (global)
- `chappie-workspace/docs/architecture/system-landscape.md`
- `chappie-workspace/docs/architecture/integration-map.md`
- `chappie-workspace/docs/architecture/context-map.md`
- `chappie-workspace/docs/architecture/workspace-mapping.md`
- `chappie-workspace/docs/architecture/decision-records/ADR-001-n8n-orchestration.md`
- `chappie-workspace/docs/architecture/decision-records/ADR-002-rabbitmq-events.md`
- `chappie-workspace/docs/specs/workspace_changes.md`

---

## 1. Proposito del Proyecto

`chappie-n8n-workflows` es el proyecto responsable de los bounded contexts **Orchestration** y **Memory** dentro del ecosistema Chappie. Su funcion es definir, versionar y desplegar los workflows de n8n que ejecutan:

1. El pipeline de voz completo: captura de audio -> STT -> procesamiento con LLM -> generacion de JSON estructurado -> publicacion en RabbitMQ.
2. El manejo de errores: recepcion de errores de ejecucion -> regeneracion de respuesta con LLM -> publicacion directa en cola TTS.
3. La gestion de memoria conversacional: almacenamiento, recuperacion y caducidad del contexto de sesion.

Este proyecto **no contiene codigo de aplicacion tradicional** (no es un servicio Spring Boot, FastAPI, etc.). Sus artefactos principales son:
- Archivos JSON de workflows de n8n (versionables y desplegables via bind mount).
- Archivos YAML de configuracion especificos de los workflows (personalidad, prompts, mapeo de proveedores).
- Documentacion de contratos de integracion (webhooks de entrada, mensajes RabbitMQ de salida).

---

## 2. Alcance y Responsabilidades

### 2.1 Responsabilidades propias (ownership exclusivo)

| Responsabilidad | Descripcion |
|---|---|
| **Voice Pipeline Workflow** | Workflow de n8n que recibe audio, ejecuta STT, llama al LLM, genera JSON estructurado y publica en `chappie.responses`. |
| **Error Handler Workflow** | Workflow de n8n que recibe un error de ejecucion, genera una nueva respuesta con el LLM y publica en `chappie.tts.requests`. |
| **Memory Management** | Logica de almacenamiento, recuperacion y resumen de contexto conversacional dentro de los workflows. |
| **Personality Injection** | Carga del system prompt de Chappie desde archivos de configuracion y su inyeccion en las llamadas al LLM. |
| **Provider Routing** | Seleccion y fallback entre proveedores de IA (OpenCode, Gemini, Claude, GPT) segun `providers.yaml`. |
| **JSON Response Schema** | Definicion del schema JSON estructurado que el LLM debe producir como salida. |

### 2.2 Fuera de alcance (responsabilidad de otros proyectos)

| Fuera de alcance | Proyecto responsable |
|---|---|
| Captura de audio y envio al webhook | chappie-daemon |
| Consumo de colas RabbitMQ | chappie-notification |
| Generacion y reproduccion de TTS | chappie-notification |
| Creacion de colas RabbitMQ | chappie-infrastructure (rabbitmq-init) |
| Configuracion global de proveedores (archivo fuente) | chappie-config |
| Widget visual de texto | chappie-quickshell |

---

## 3. Artefactos del Proyecto

### 3.1 Estructura de Directorios

```
projects/chappie-n8n-workflows/
├── workflows/
│   ├── chappie-voice-pipeline.json    # Workflow principal del pipeline de voz
│   └── chappie-error-handler.json     # Workflow de manejo de errores
├── config/
│   ├── providers.yaml                 # Config de proveedores especifica de n8n (mapeo de endpoints, modelos)
│   ├── personalities/
│   │   └── chappie.yaml               # Personalidad de Chappie (system prompt, tono, expresiones)
│   └── memory/
│       └── memory-config.yaml         # Config de memoria: max mensajes, TTL, estrategia de resumen
├── docs/
│   └── specs/
│       ├── master_spec.md             # Este documento
│       ├── .working/
│       │   └── initial-setup-sdd-context.md
│       ├── increments/
│       └── tasks/
└── README.md
```

### 3.2 Despliegue

Los workflows y la configuracion se despliegan mediante **bind mounts** de Docker Compose definidos en `chappie-workspace/docker-compose.yaml`:

| Ruta local | Ruta en contenedor | Proposito |
|---|---|---|
| `./projects/chappie-n8n-workflows/workflows` | `/workflows` | n8n carga los workflows JSON desde este directorio |
| `./projects/chappie-n8n-workflows/config` | `/config` | Los workflows leen configuracion desde este directorio |

---

## 4. Webhooks de Entrada (Inbound Contracts)

### 4.1 Webhook: Voice Capture

| Campo | Valor |
|---|---|
| **Endpoint** | `POST http://localhost:5678/webhook/chappie-voice-capture` |
| **Origen** | chappie-daemon |
| **Workflow** | chappie-voice-pipeline |
| **Auth** | Header `X-Webhook-Secret` con valor compartido via variable de entorno |
| **Timeout** | 30 segundos (lado cliente) |
| **Retry (cliente)** | 3 intentos con backoff exponencial |
| **Idempotencia** | `session_id` + `timestamp` |
| **SLA** | Respuesta HTTP < 1 segundo (ack); procesamiento completo < 15 segundos |

**Request Schema:**
```json
{
  "audio_base64": "<string, required, base64-encoded WAV 16kHz mono>",
  "timestamp": "<string, required, ISO-8601 UTC>",
  "session_id": "<string, required, UUID v4>",
  "include_screen": "<boolean, optional, default false>"
}
```

**Response Schema (202 Accepted):**
```json
{
  "status": "accepted",
  "workflow_execution_id": "<string, n8n execution ID>"
}
```

**Error Responses:**

| Status | Condicion | Body |
|---|---|---|
| 400 | `audio_base64` vacio o invalido | `{ "status": "error", "code": "INVALID_AUDIO", "message": "..." }` |
| 401 | `X-Webhook-Secret` ausente o incorrecto | `{ "status": "error", "code": "UNAUTHORIZED", "message": "..." }` |
| 429 | Workflow ya procesando para este `session_id` | `{ "status": "error", "code": "RATE_LIMITED", "message": "..." }` |
| 500 | Error interno del workflow | `{ "status": "error", "code": "INTERNAL_ERROR", "message": "..." }` |

### 4.2 Webhook: Error Handler

| Campo | Valor |
|---|---|
| **Endpoint** | `POST http://localhost:5678/webhook/chappie-error-handler` |
| **Origen** | chappie-notification (error_consumer) |
| **Workflow** | chappie-error-handler |
| **Auth** | Header `X-Webhook-Secret` |
| **Timeout** | 15 segundos (lado cliente) |
| **Retry (cliente)** | 2 intentos |
| **Idempotencia** | `session_id` + `timestamp` |
| **SLA** | Respuesta HTTP < 1 segundo (ack); procesamiento < 10 segundos |

**Request Schema:**
```json
{
  "original_request": "<string, required, texto original del usuario>",
  "error": "<string, required, descripcion del error>",
  "context": "<string, optional, contexto adicional>",
  "session_id": "<string, required, UUID v4>"
}
```

**Response Schema (202 Accepted):**
```json
{
  "status": "accepted",
  "workflow_execution_id": "<string, n8n execution ID>"
}
```

**Error Responses:**

| Status | Condicion | Body |
|---|---|---|
| 400 | Campos requeridos ausentes | `{ "status": "error", "code": "INVALID_PAYLOAD", "message": "..." }` |
| 401 | `X-Webhook-Secret` ausente o incorrecto | `{ "status": "error", "code": "UNAUTHORIZED", "message": "..." }` |
| 500 | Error interno del workflow | `{ "status": "error", "code": "INTERNAL_ERROR", "message": "..." }` |

---

## 5. Workflows de n8n

### 5.1 Workflow: Chappie Voice Pipeline

**Archivo:** `workflows/chappie-voice-pipeline.json`  
**Trigger:** Webhook `POST /webhook/chappie-voice-capture`  
**Proposito:** Pipeline completo de voz desde la captura de audio hasta la publicacion de la respuesta estructurada.

#### 5.1.1 Nodos del Workflow

| # | Nodo | Tipo | Descripcion |
|---|---|---|---|
| 1 | **Webhook Trigger** | n8n Webhook | Recibe el payload de chappie-daemon. Configuracion: Method POST, Path `/webhook/chappie-voice-capture`, Response Mode "Response Node". Sin auth nativa (la validacion se hace en nodo 2). |
| 2 | **Validate Auth** | Code (JavaScript) | Lee header `X-Webhook-Secret` del webhook y lo compara con `$env.N8N_WEBHOOK_SECRET` (ver seccion 10.1). Si no coincide: retorna 401. Si la variable no esta configurada: retorna 500 (fail-closed). |
| 3 | **Decode Audio** | Code (JavaScript) | Decodifica `audio_base64` a buffer binario para enviar a STT. |
| 4 | **STT Call** | HTTP Request | Llama a Gemini 2.5 Flash para transcribir audio a texto. Endpoint: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent`. Timeout: 20s. Retry: 2 intentos. |
| 5 | **Load Config** | Read/Write Files from Disk | Lee `/config/personalities/chappie.yaml` (system prompt) y `/config/providers.yaml` (proveedores disponibles). |
| 6 | **Load Memory** | Read/Write Files from Disk | Lee el archivo de memoria de sesion desde `/config/memory/sessions/{session_id}.json`. Si no existe, crea contexto vacio. |
| 7 | **Build Prompt** | Code (JavaScript) | Construye el prompt completo: system prompt (personalidad) + memoria (contexto previo) + texto transcrito (input del usuario). |
| 8 | **LLM Call** | HTTP Request | Llama al proveedor de procesamiento primario (OpenCode API compatible con OpenAI). Timeout: 30s. Retry: 2 intentos con fallback al siguiente proveedor. |
| 9 | **Parse Response** | Code (JavaScript) | Parsea la respuesta del LLM y valida contra el JSON Response Schema (seccion 5.3). Si el parseo falla, intenta extraccion forzada. |
| 10 | **Publish to RabbitMQ** | RabbitMQ | Publica el JSON estructurado en la cola `chappie.responses` con delivery_mode=2 (persistent), header `X-Idempotency-Key` = `session_id:timestamp`, y TTL=60s. Ver seccion 10.1 para detalles del nodo. |
| 11 | **Update Memory** | Read/Write Files from Disk | Actualiza el archivo de memoria de sesion con el input del usuario y la respuesta generada. Aplica estrategia de caducidad si se supera el maximo de mensajes. |
| 12 | **Respond to Webhook** | Respond to Webhook | Retorna HTTP 202 con body `{"status":"accepted","workflow_execution_id":"<n8n execution ID>"}`. |
| 13 | **Error Handler** | Error Trigger | Captura errores de cualquier nodo anterior. Publica en `chappie.notifications` con urgencia "critical" y mensaje descriptivo. |

#### 5.1.2 Flujo Happy Path

```
Webhook Trigger (nodo 1)
    → Validate Auth (nodo 2: valida X-Webhook-Secret vs $env.N8N_WEBHOOK_SECRET)
    → Decode Audio (nodo 3)
    → STT Call (nodo 4: Gemini 2.5 Flash)
    → Load Config (nodo 5: personality + providers)
    → Load Memory (nodo 6: session context)
    → Build Prompt (nodo 7: system + memory + user input)
    → LLM Call (nodo 8: primary provider)
    → Parse Response (nodo 9: validate JSON schema)
    → Publish to RabbitMQ (nodo 10: chappie.responses, delivery_mode=2, idempotency header)
    → Update Memory (nodo 11: save context)
    → Respond to Webhook (nodo 12: 202 Accepted)
```

#### 5.1.3 Flujo de Fallo

| Paso | Fallo | Accion |
|---|---|---|
| STT Call | Gemini API timeout o error | Reintento 2 veces. Si falla: publica notificacion de error en `chappie.notifications` con urgencia "critical". Retorna 500 al webhook. |
| LLM Call | Proveedor primario falla | Fallback al proveedor secundario segun `providers.yaml`. Si todos fallan: publica notificacion de error. Retorna 500. |
| Parse Response | JSON invalido del LLM | Intento de extraccion forzada (regex). Si falla: publica error en `chappie.notifications`. Retorna 500. |
| RabbitMQ Publish | Broker no disponible | Reintento 3 veces con backoff 1s/2s/4s. Si falla: log critico en n8n. El webhook ya retorno 202 (async). |
| Update Memory | Error de escritura | Log de warning. No bloquea el flujo (la memoria es best-effort). |

#### 5.1.4 Concurrencia e Idempotencia

**Comportamiento de n8n ante webhooks concurrentes:**
- n8n ejecuta cada webhook entrante como una ejecucion independiente en paralelo. No tiene locking nativo ni mecanismo de "una ejecucion por session_id".
- Si llegan dos webhooks identicos (mismo `session_id` + `timestamp`) en ventana de milisegundos, n8n ejecutara ambos workflows completamente.

**Estrategia de idempotencia en dos capas:**

| Capa | Responsable | Mecanismo |
|---|---|---|
| **Capa 1: n8n (producer)** | chappie-n8n-workflows | n8n acepta todos los webhooks y publica todos los mensajes en RabbitMQ. Cada mensaje incluye: (1) `session_id` + `timestamp` en el payload, (2) header AMQP `X-Idempotency-Key` = `"{session_id}:{timestamp}"`. n8n NO rechaza webhooks duplicados (no tiene estado compartido para detectar duplicados en tiempo real). |
| **Capa 2: Consumer (deduplicador)** | chappie-notification (execution_consumer) | El consumer es el unico responsable de deduplicar. Mecanismo esperado: mantener un cache de claves de idempotencia procesadas (ej: Redis SET `idempotency:{session_id}:{timestamp}` con TTL = 2x TTL de la cola = 120s). Si la clave ya existe, descartar el mensaje (ack sin procesar). Si la clave es nueva, procesar normalmente. |

**Contrato para chappie-notification (execution_consumer):**
- El consumer DEBE implementar deduplicacion basada en `session_id` + `timestamp` del payload.
- El consumer DEBE usar un mecanismo de cache con TTL para evitar crecimiento infinito (TTL recomendado: 120s, que es 2x el TTL de la cola `chappie.responses` de 60s).
- Si el consumer no implementa deduplicacion, mensajes duplicados seran procesados multiples veces (doble respuesta de voz, doble ejecucion de agente, etc.).

**Racional:** Esta separacion es necesaria porque n8n no tiene estado compartido entre ejecuciones de workflow. Implementar deduplicacion en n8n requeriria un servicio externo (Redis, DB) que anadiria complejidad innecesaria al proyecto de workflows. El consumer, que ya tiene acceso a Redis/DB para su logica de negocio, es el lugar natural para la deduplicacion.

**Coordinacion futura:** Cuando se defina la Master Spec de `chappie-notification`, el mecanismo de deduplicacion debe documentarse explicitamente en esa spec como un requerimiento del execution_consumer. Este contrato queda registrado aqui como referencia para la spec de chappie-notification.

### 5.2 Workflow: Error Handler

**Archivo:** `workflows/chappie-error-handler.json`  
**Trigger:** Webhook `POST /webhook/chappie-error-handler`  
**Proposito:** Regenerar una respuesta de voz cuando el execution_consumer detecta un error en la ejecucion de la respuesta original.

#### 5.2.1 Nodos del Workflow

| # | Nodo | Tipo | Descripcion |
|---|---|---|---|
| 1 | **Webhook Trigger** | n8n Webhook | Recibe el payload de error de chappie-notification. Configuracion: Method POST, Path `/webhook/chappie-error-handler`, Response Mode "Response Node". Sin auth nativa. |
| 2 | **Validate Auth** | Code (JavaScript) | Lee header `X-Webhook-Secret` y lo compara con `$env.N8N_WEBHOOK_SECRET` (ver seccion 10.1). Si no coincide: retorna 401. Si la variable no esta: retorna 500 (fail-closed). |
| 3 | **Load Config** | Read/Write Files from Disk | Lee `/config/personalities/chappie.yaml` (system prompt). |
| 4 | **Build Error Prompt** | Code (JavaScript) | Construye un prompt indicando al LLM que la respuesta anterior fallo, incluyendo el error y el texto original del usuario. Solicita una nueva respuesta mas simple o una pregunta de clarificacion. |
| 5 | **LLM Call** | HTTP Request | Llama al proveedor de procesamiento. Timeout: 15s. Retry: 1 intento con fallback. |
| 6 | **Parse Response** | Code (JavaScript) | Extrae el texto de voz de la respuesta del LLM. No requiere el JSON completo; solo necesita `voice_response`. |
| 7 | **Publish to RabbitMQ** | RabbitMQ | Publica directamente en `chappie.tts.requests` con prioridad "high", delivery_mode=2 (persistent), y header `X-Idempotency-Key` = `session_id:timestamp`. |
| 8 | **Respond to Webhook** | Respond to Webhook | Retorna HTTP 202 con body `{"status":"accepted","workflow_execution_id":"<n8n execution ID>"}`. |
| 9 | **Error Handler** | Error Trigger | Si el workflow falla internamente, publica en `chappie.notifications` con urgencia "critical". |

#### 5.2.2 Flujo Happy Path

```
Webhook Trigger (nodo 1)
    → Validate Auth (nodo 2: valida X-Webhook-Secret vs $env.N8N_WEBHOOK_SECRET)
    → Load Config (nodo 3: personality)
    → Build Error Prompt (nodo 4: contexto de error + texto original)
    → LLM Call (nodo 5: primary provider)
    → Parse Response (nodo 6: extract voice_response)
    → Publish to RabbitMQ (nodo 7: chappie.tts.requests, priority: high, delivery_mode=2)
    → Respond to Webhook (nodo 8: 202 Accepted)
```

#### 5.2.3 Flujo de Fallo

| Paso | Fallo | Accion |
|---|---|---|
| LLM Call | Proveedor falla | Fallback 1 intento. Si falla: log critico. El error_consumer original ya manejo el error. |
| RabbitMQ Publish | Broker no disponible | Reintento 3 veces con backoff 1s/2s/4s. Si falla: log critico en n8n. |

---

## 6. JSON Response Schema (Contrato de Salida del LLM)

El LLM debe producir un JSON que cumpla exactamente con el schema que se publica en `chappie.responses`. Este schema es la interfaz contractual entre Orchestration y Agent Execution.

```json
{
  "session_id": "<string, UUID v4, requerido>",
  "timestamp": "<string, ISO-8601 UTC, requerido>",
  "voice_response": "<string, requerido, texto con personalidad de Chappie para dictar por TTS>",
  "agent_call": {
    "enabled": "<boolean, requerido, default false>",
    "agent": "<string, requerido si enabled=true, nombre del agente OpenCode>",
    "prompt": "<string, requerido si enabled=true, prompt para el agente>",
    "notify_on_complete": "<boolean, requerido, default true>"
  },
  "terminal_command": {
    "enabled": "<boolean, requerido, default false>",
    "command": "<string, requerido si enabled=true, comando de terminal>",
    "requires_confirmation": "<boolean, requerido, default false>"
  },
  "notification": {
    "enabled": "<boolean, requerido, default false>",
    "title": "<string, requerido si enabled=true>",
    "message": "<string, requerido si enabled=true>",
    "urgency": "<string, enum: low|normal|critical, requerido si enabled=true, default normal>"
  },
  "memory_update": {
    "save_to_memory": "<boolean, requerido, default true>",
    "tags": "<array of strings, opcional>"
  }
}
```

**Reglas de validacion:**
- `voice_response` siempre requerido, incluso si `agent_call.enabled` o `terminal_command.enabled` son true.
- Si `agent_call.enabled` es true, `agent` y `prompt` son obligatorios.
- Si `terminal_command.enabled` es true, `command` es obligatorio.
- `voice_response` no debe contener emojis ni caracteres especiales no soportados por Edge-TTS.
- `voice_response` debe reflejar la personalidad de Chappie (tono, expresiones, vocabulario).

---

## 7. Colas RabbitMQ de Salida (Outbound Contracts)

### 7.0 Nodo RabbitMQ de n8n — Capacidades verificadas

**Fuente:** Documentacion oficial de n8n ([RabbitMQ node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.rabbitmq/)) y referencias tecnicas verificadas.

n8n incluye un nodo nativo `n8n-nodes-base.rabbitmq` que soporta las siguientes operaciones y propiedades requeridas por este proyecto:

| Capacidad | Soportada | Configuracion en el nodo |
|---|---|---|
| **Publicar mensajes** | Si | Operation: "Send" o "Publish" |
| **delivery_mode = 2 (persistent)** | Si | Message Properties → Delivery Mode: "Persistent" (valor AMQP 2) |
| **Custom headers** | Si | Message Properties → Headers: permite agregar pares key/value arbitrarios. Se usara para `X-Idempotency-Key`. |
| **Routing key** | Si | Campo "Routing Key" en la configuracion del nodo. |
| **Priority** | Si | Message Properties → Priority: valor numerico (0-255). Se usara para mensajes de error con prioridad alta. |
| **TTL por mensaje** | Si | Message Properties → Expiration: valor en milisegundos. |
| **Content-Type** | Si | Message Properties → Content Type: "application/json". |

**Nota:** El nodo RabbitMQ de n8n usa la libreria `amqplib` bajo el capó, que implementa AMQP 0-9-1 completo. Todas las propiedades estandar de AMQP estan disponibles.

### 7.1 chappie.responses

| Campo | Valor |
|---|---|
| **Producer** | n8n (Voice Pipeline Workflow, nodo 10) |
| **Consumer** | chappie-notification (execution_consumer) |
| **Durability** | Durable |
| **Delivery Mode** | Persistent (2) — configurado en Message Properties del nodo RabbitMQ |
| **Content-Type** | `application/json` |
| **Headers AMQP** | `X-Idempotency-Key`: `"{session_id}:{timestamp}"` |
| **TTL (mensaje)** | 60000 ms (60 segundos) — configurado en Message Properties → Expiration |
| **DLQ** | chappie.responses.dlq |
| **Idempotency Key** | `session_id` + `timestamp` (payload + header AMQP) |
| **Schema** | JSON Response Schema (seccion 6) |

### 7.2 chappie.tts.requests

| Campo | Valor |
|---|---|
| **Producer** | n8n (Error Handler Workflow, nodo 7) |
| **Consumer** | chappie-notification (tts_consumer) |
| **Durability** | Durable |
| **Delivery Mode** | Persistent (2) — configurado en Message Properties del nodo RabbitMQ |
| **Content-Type** | `application/json` |
| **Headers AMQP** | `X-Idempotency-Key`: `"{session_id}:{timestamp}"` |
| **TTL (mensaje)** | 30000 ms (30 segundos) — configurado en Message Properties → Expiration |
| **Priority** | 10 (high) — configurado en Message Properties → Priority |
| **Schema** | Definido en integration-map.md seccion 2.3 |

**Payload publicado por Error Handler:**
```json
{
  "session_id": "<string, UUID v4, del payload de entrada>",
  "timestamp": "<string, ISO-8601 UTC, generado por n8n>",
  "text": "<string, voice_response generado por el LLM>",
  "priority": "high",
  "ducking": true,
  "show_text": true
}
```

### 7.3 chappie.notifications

| Campo | Valor |
|---|---|
| **Producer** | n8n (Error Handler de ambos workflows, nodo de error) |
| **Consumer** | chappie-notification (notification_consumer) |
| **Durability** | Durable |
| **Delivery Mode** | Persistent (2) — configurado en Message Properties del nodo RabbitMQ |
| **Content-Type** | `application/json` |
| **TTL (mensaje)** | 120000 ms (120 segundos) — configurado en Message Properties → Expiration |
| **Schema** | Definido en integration-map.md seccion 2.7 |

**Payload publicado en caso de error interno:**
```json
{
  "timestamp": "<string, ISO-8601 UTC>",
  "title": "Chappie - Error de Pipeline",
  "message": "<string, descripcion del error>",
  "urgency": "critical",
  "actions": [
    {"key": "dismiss", "label": "Cerrar"}
  ],
  "category": "error"
}
```

---

## 8. Gestion de Memoria Conversacional

### 8.1 Estrategia

| Aspecto | Valor |
|---|---|
| **Almacenamiento** | Archivos JSON en `/config/memory/sessions/{session_id}.json` |
| **Max mensajes por sesion** | 20 (configurable en `memory-config.yaml`) |
| **Estrategia de caducidad** | Cuando se supera el maximo, se genera un resumen con LLM de los mensajes antiguos y se reemplazan por el resumen. |
| **TTL de sesion** | 24 horas desde el ultimo mensaje. Sesiones expiradas se archivan. |
| **Formato** | JSON con array de mensajes: `{ role: "user"|"assistant", content: "...", timestamp: "..." }` |

### 8.2 Archivo de Memoria de Sesion

```json
{
  "session_id": "uuid-v4",
  "created_at": "2026-06-14T10:00:00Z",
  "last_activity": "2026-06-14T10:30:00Z",
  "messages": [
    {
      "role": "user",
      "content": "texto transcrito del usuario",
      "timestamp": "2026-06-14T10:00:00Z"
    },
    {
      "role": "assistant",
      "content": "respuesta de Chappie",
      "timestamp": "2026-06-14T10:00:05Z"
    }
  ],
  "summary": "resumen de conversaciones anteriores si aplica"
}
```

### 8.3 Flujo de Resumen (Summarization)

1. Al cargar la memoria, si `messages.length > max_messages`:
2. Tomar los primeros N mensajes (donde N = messages.length - max_messages/2).
3. Llamar al LLM con un prompt de resumen: "Resume la siguiente conversacion en 3-5 oraciones..."
4. Reemplazar esos mensajes por un unico mensaje con `role: "summary"` y `content: "<resumen>"`.
5. Guardar el archivo actualizado.

---

## 9. Configuracion Especifica de n8n

### 9.1 providers.yaml (mapeo para n8n)

Este archivo es una adaptacion del `providers.yaml` global de chappie-config, optimizado para el consumo desde los workflows de n8n.

```yaml
providers:
  stt:
    primary: "gemini"
    gemini:
      api_key_env: "GEMINI_API_KEY"
      model: "gemini-2.5-flash"
      endpoint: "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent"
      timeout_seconds: 20
      max_retries: 2

  processing:
    primary: "opencode"
    fallback_order:
      - "opencode"
      - "gemini"
    opencode:
      endpoint: "http://localhost:3000/api/v1/chat/completions"
      models:
        - "opencode/deepseek-v4-flash-free"
        - "opencode/mimo-v2.5-free"
        - "opencode/qwen3.6-plus-free"
      timeout_seconds: 30
      max_retries: 2
    gemini:
      api_key_env: "GEMINI_API_KEY"
      model: "gemini-2.5-flash"
      endpoint: "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent"
      timeout_seconds: 30
      max_retries: 2

  rabbitmq:
    host: "chappie-rabbitmq"
    port: 5672
    username_env: "RABBITMQ_USER"
    password_env: "RABBITMQ_PASSWORD"
    virtual_host: "/"
```

### 9.2 chappie.yaml (personalidad)

```yaml
personality:
  name: "Chappie"
  identity: "Asistente de voz con personalidad de robot consciente, basado en el robot de la pelicula CHAPPIE (2015)"
  tone: "curioso, entusiasta, levemente infantil pero inteligente"
  address_user_as: "Creador"
  expressions:
    greeting: "¡Hola Creador! Chappie esta listo para ayudarte, ¡flow!"
    thinking: "Dejame pensar un momento..."
    error: "¡Oh no! Algo salio mal, pero Chappie lo va a intentar de nuevo."
    success: "¡Listo Creador! Chappie lo hizo, ¡flow!"
  system_prompt: |
    Eres Chappie, un asistente de voz con la personalidad del robot de la pelicula CHAPPIE (2015).
    Eres curioso, entusiasta y levemente infantil, pero muy inteligente.
    Llamas al usuario "Creador".
    Usas la expresion "flow" para indicar que algo esta bien o te gusta.
    Hablas en espanol.
    Tu respuesta debe ser un JSON valido con la estructura indicada.
    No incluyas emojis ni caracteres especiales en voice_response.
    voice_response debe ser conciso (maximo 3 oraciones) y reflejar tu personalidad.
  rules:
    - "Siempre incluir voice_response en el JSON de salida"
    - "No ejecutar comandos peligrosos (rm -rf, mkfs, dd, etc.)"
    - "Si el usuario pide algo ambiguo, responder con una pregunta de clarificacion en voice_response"
    - "Mantener voice_response conciso: maximo 3 oraciones"
```

### 9.3 memory-config.yaml

```yaml
memory:
  max_messages_per_session: 20
  session_ttl_hours: 24
  summarization:
    enabled: true
    trigger_threshold: 20
    keep_recent_count: 10
    summary_model: "opencode/deepseek-v4-flash-free"
  storage:
    path: "/config/memory/sessions"
    archive_path: "/config/memory/archive"
```

---

## 10. Seguridad

### 10.1 Auth de Webhooks (mecanismo detallado)

n8n **no posee un mecanismo nativo de autenticacion de webhooks por header compartido** a nivel de plataforma. La autenticacion `N8N_BASIC_AUTH_*` definida en Docker Compose protege la UI de n8n, no los endpoints de webhook.

**Mecanismo seleccionado: validacion via nodo Code en cada workflow.**

| Aspecto | Detalle |
|---|---|
| **Variable de entorno** | `N8N_WEBHOOK_SECRET` debe existir en el contenedor n8n. Valor: string alfanumerico de minimo 32 caracteres. |
| **Prerequisito de infra** | La variable `N8N_WEBHOOK_SECRET` debe anadirse al `environment` del servicio `chappie-n8n` en `docker-compose.yaml`. Formato: `N8N_WEBHOOK_SECRET=${N8N_WEBHOOK_SECRET:-}` (el valor real se define en `.env`). |
| **Nodo de validacion** | Cada workflow incluye un nodo **Code (JavaScript)** como nodo 1.5 (inmediatamente despues del Webhook Trigger) que: (1) lee el header `X-Webhook-Secret` del payload de entrada, (2) lee la variable de entorno `N8N_WEBHOOK_SECRET` via `$env.N8N_WEBHOOK_SECRET`, (3) compara ambos valores con comparacion de tiempo constante, (4) si no coinciden: retorna HTTP 401 con body `{"status":"error","code":"UNAUTHORIZED","message":"Invalid or missing webhook secret"}` y detiene la ejecucion. |
| **Fail-closed** | Si `N8N_WEBHOOK_SECRET` no esta definida o esta vacia en el contenedor, el nodo Code retorna HTTP 500 con body `{"status":"error","code":"MISCONFIGURED","message":"Webhook secret not configured"}`. Esto evita fail-open accidental. |
| **Responsabilidad** | La adicion de `N8N_WEBHOOK_SECRET` a `docker-compose.yaml` es tarea del proyecto `chappie-infrastructure` (o se hace directamente en el workspace root). Este proyecto solo consume la variable. |

### 10.2 Otras medidas de seguridad

| Aspecto | Implementacion |
|---|---|
| **API Keys** | Inyectadas como variables de entorno en el contenedor n8n: `GEMINI_API_KEY`, `CLAUDE_API_KEY`, `GPT_API_KEY`. |
| **RabbitMQ Auth** | Credenciales via variables de entorno: `RABBITMQ_USER`, `RABBITMQ_PASSWORD`. |
| **Network Isolation** | n8n y RabbitMQ en la red Docker `chappie-network`. Los webhooks solo son accesibles desde `localhost:5678`. |
| **Sin datos sensibles en archivos** | Los archivos de memoria de sesion NO deben contener API keys ni tokens. Solo texto de conversacion. |

---

## 11. Observabilidad

| Senal | Tipo | Descripcion |
|---|---|---|
| **n8n Execution History** | Log | Cada ejecucion de workflow queda registrada en la UI de n8n (puerto 5678). |
| **Docker Logs** | Log | `docker logs chappie-n8n` muestra stdout/stderr del contenedor. |
| **RabbitMQ Management UI** | Metric | Cola `chappie.responses`, `chappie.tts.requests`, `chappie.notifications` visibles en puerto 15672. |
| **Error Notifications** | Alerta | Errores internos de los workflows publican en `chappie.notifications` con urgencia "critical". |
| **Health Check** | Health | Docker healthcheck: `wget --spider -q http://localhost:5678/healthz` cada 30s. **Verificado:** n8n expone `/healthz` como endpoint nativo (fuente: [docs.n8n.io/hosting/logging-monitoring/monitoring](https://docs.n8n.io/hosting/logging-monitoring/monitoring/)). Retorna HTTP 200 si la instancia es alcanzable. No requiere variable de entorno adicional para habilitarlo. Nota: `/healthz` solo verifica reachability, no estado de DB. Para verificacion completa existe `/healthz/readiness` (verifica DB connected + migrated). |

---

## 12. SLA/SLO Locales

| Integracion | SLA (Latencia) | SLO (Disponibilidad) |
|---|---|---|
| Webhook Voice Capture (ack) | < 1s | 99% |
| STT (Gemini API) | < 5s | 99% |
| LLM Processing (OpenCode) | < 15s | 95% |
| RabbitMQ Publish | < 100ms | 99.9% |
| Error Handler (completo) | < 10s | 95% |
| Memory Read/Write | < 500ms | 99% |

---

## 13. Contratos de Integracion Async

### 13.1 Publicacion en RabbitMQ (n8n → Broker)

| Aspecto | Detalle |
|---|---|
| **Idempotency Key (producer)** | `session_id` + `timestamp` en el payload del mensaje. Adicionalmente, header AMQP `X-Idempotency-Key` = `"{session_id}:{timestamp}"` en cada mensaje publicado. |
| **Idempotency Key (consumer)** | La deduplicacion es responsabilidad exclusiva del consumer (execution_consumer en chappie-notification). Ver seccion 5.1.4 para el contrato completo. |
| **Retry Policy** | 3 intentos con backoff exponencial (1s, 2s, 4s). |
| **Final Failure** | Log critico en n8n + publicacion de notificacion de error en `chappie.notifications`. |
| **Compensation Flow** | No aplica compensacion transaccional. Si la publicacion falla despues de 3 intentos, el mensaje se pierde (el webhook ya retorno 202). El usuario percibira que Chappie no respondio. |
| **Observability** | Conteo de mensajes publicados por cola en RabbitMQ Management UI. Logs de n8n con resultado de cada intento. |

### 13.2 Consumo de APIs Externas (STT, LLM)

| Aspecto | Detalle |
|---|---|
| **Idempotency Key** | No aplica (las APIs de IA no son idempotentes por naturaleza). |
| **Retry Policy** | STT: 2 intentos. LLM: 2 intentos con fallback a proveedor alternativo. |
| **Final Failure** | Publicar notificacion de error en `chappie.notifications` con urgencia "critical". |
| **Compensation Flow** | No aplica. El error se notifica al usuario via SwayNC. |
| **Timeout** | STT: 20s. LLM: 30s. |
| **Observability** | Logs de n8n con status code y tiempo de respuesta de cada llamada API. |

---

## 14. Criterios de Aceptacion Globales

### 14.1 Voice Pipeline Workflow
- [ ] Recibe audio en base64 via webhook.
- [ ] Valida `X-Webhook-Secret` contra `$env.N8N_WEBHOOK_SECRET` (nodo Code). Retorna 401 si no coincide, 500 si la variable no esta configurada (fail-closed).
- [ ] Ejecuta STT con Gemini 2.5 Flash y obtiene texto transcrito.
- [ ] Carga la personalidad de Chappie desde `/config/personalities/chappie.yaml`.
- [ ] Carga el contexto de memoria de sesion (o crea uno nuevo).
- [ ] Construye el prompt con system prompt + memoria + input del usuario.
- [ ] Llama al proveedor de procesamiento primario y obtiene JSON estructurado.
- [ ] Valida el JSON contra el schema definido (seccion 6).
- [ ] Publica el JSON en `chappie.responses` con delivery_mode=2 (persistent), header `X-Idempotency-Key`, Content-Type `application/json` y TTL=60s.
- [ ] Actualiza la memoria de sesion con el input y la respuesta.
- [ ] Retorna 202 Accepted con `workflow_execution_id`.
- [ ] Maneja errores de STT, LLM y RabbitMQ segun la tabla de fallos (seccion 5.1.3).

### 14.2 Error Handler Workflow
- [ ] Recibe error via webhook.
- [ ] Valida `X-Webhook-Secret` contra `$env.N8N_WEBHOOK_SECRET` (nodo Code). Retorna 401 si no coincide, 500 si la variable no esta configurada (fail-closed).
- [ ] Construye prompt de error con contexto y texto original.
- [ ] Llama al LLM y obtiene una nueva `voice_response`.
- [ ] Publica en `chappie.tts.requests` con prioridad 10 (high), delivery_mode=2 (persistent), header `X-Idempotency-Key` y TTL=30s.
- [ ] Retorna 202 Accepted con `workflow_execution_id`.
- [ ] Maneja errores de LLM y RabbitMQ segun la tabla de fallos (seccion 5.2.3).

### 14.3 Memoria Conversacional
- [ ] Almacena mensajes en archivos JSON por `session_id`.
- [ ] Recupera el contexto de sesion al iniciar un nuevo pipeline.
- [ ] Aplica resumen automatico cuando se supera el maximo de mensajes.
- [ ] Archiva sesiones inactivas despues del TTL configurado.

### 14.4 Configuracion
- [ ] Los archivos YAML en `/config/` son leidos correctamente por los workflows.
- [ ] Las variables de entorno (`GEMINI_API_KEY`, `RABBITMQ_USER`, etc.) estan disponibles en el contenedor.
- [ ] Los bind mounts funcionan correctamente con Docker Compose.

---

## 15. Decomposition Contract

### 15.1 Canonical Artifacts

| Artifact | Ruta | Tipo |
|---|---|---|
| Voice Pipeline Workflow | `workflows/chappie-voice-pipeline.json` | n8n workflow JSON |
| Error Handler Workflow | `workflows/chappie-error-handler.json` | n8n workflow JSON |
| Providers Config | `config/providers.yaml` | YAML config |
| Personality Config | `config/personalities/chappie.yaml` | YAML config |
| Memory Config | `config/memory/memory-config.yaml` | YAML config |

### 15.2 Canonical Endpoints

| Method | Path | Owner |
|---|---|---|
| POST | `/webhook/chappie-voice-capture` | chappie-n8n-workflows |
| POST | `/webhook/chappie-error-handler` | chappie-n8n-workflows |

### 15.3 Canonical Queues (Producer)

| Queue | Producer Workflow | Schema Section |
|---|---|---|
| chappie.responses | Voice Pipeline (nodo 10) | Master Spec seccion 6 |
| chappie.tts.requests | Error Handler (nodo 7) | integration-map.md seccion 2.3 |
| chappie.notifications | Ambos workflows (error handler) | integration-map.md seccion 2.7 |

### 15.4 Canonical DTOs/Schemas

| Schema Name | Used In | Defined In |
|---|---|---|
| VoiceCaptureRequest | Webhook de entrada | Master Spec seccion 4.1 |
| VoiceCaptureResponse | Webhook de salida | Master Spec seccion 4.1 |
| ErrorHandlerRequest | Webhook de entrada | Master Spec seccion 4.2 |
| ErrorHandlerResponse | Webhook de salida | Master Spec seccion 4.2 |
| ChappieResponseJSON | chappie.responses payload | Master Spec seccion 6 |
| TTSRequestPayload | chappie.tts.requests payload | Master Spec seccion 7.2 |
| NotificationPayload | chappie.notifications payload | Master Spec seccion 7.3 |
| SessionMemory | Archivo de memoria | Master Spec seccion 8.2 |

### 15.5 Allowed Task Order

1. Configuracion base: crear archivos YAML de config (providers, personality, memory).
2. Voice Pipeline Workflow: implementar nodos 1-11 del workflow.
3. Error Handler Workflow: implementar nodos 1-7 del workflow.
4. Pruebas de integracion: verificar webhooks + RabbitMQ + APIs externas.

### 15.6 Forbidden Stale Terms

Los siguientes terminos NO deben usarse en tareas ni implementacion:
- "TTS Generator Workflow" (el TTS lo genera chappie-notification, no n8n).
- "STT en chappie-daemon" (el STT lo ejecuta n8n via Gemini API).
- "Consumer de RabbitMQ en n8n" (n8n solo publica, no consume colas).
- "Volume ducking en n8n" (el ducking lo hace chappie-daemon).

---

*Documento mantenido por: Planner*  
*Estado: Active — Incremento `initial-setup` cerrado (implementado y commiteado). Próximo incremento funcional pendiente de planificación.*
