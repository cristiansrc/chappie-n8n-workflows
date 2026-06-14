# chappie-n8n-workflows

**Workflows de n8n para orquestación del asistente Chappie**

---

## Responsabilidad

- Definición de workflows de n8n para el pipeline de voz
- STT (Speech-to-Text) con Gemini 2.5 Flash
- Procesamiento con modelos de IA (multi-proveedor: OpenCode + fallback Gemini)
- Generación de JSON estructurado con personalidad de Chappie
- Manejo de errores con regeneración de respuesta vía LLM
- Gestión de memoria conversacional (archivos JSON por sesión)

## Estado

**Implementado - Fase 1** (Workflows instalados, pendiente prueba de integración)

## Estructura

```
chappie-n8n-workflows/
├── workflows/
│   ├── chappie-voice-pipeline.json    # Workflow principal (13 nodos)
│   └── chappie-error-handler.json     # Manejo de errores (9 nodos)
├── config/
│   ├── providers.yaml                 # APIs de modelos (STT + LLM)
│   ├── personalities/
│   │   └── chappie.yaml               # Personalidad de Chappie
│   └── memory/
│       ├── memory-config.yaml         # Config de memoria conversacional
│       ├── sessions/                  # Archivos de sesión (runtime)
│       └── archive/                   # Sesiones archivadas (runtime)
├── docs/
│   └── specs/
│       ├── master_spec.md
│       ├── .working/
│       └── tasks/
└── README.md
```

## Workflows

### chappie-voice-pipeline (13 nodos)

Pipeline completo de voz desde la captura de audio hasta la publicación en RabbitMQ:

1. **Webhook Trigger** — Recibe audio en base64 vía `POST /webhook/chappie-voice-capture`
2. **Validate Auth** — Valida header `X-Webhook-Secret` contra `$env.N8N_WEBHOOK_SECRET` (fail-closed)
3. **Decode Audio** — Decodifica `audio_base64` a buffer binario
4. **STT Call** — Transcribe audio con Gemini 2.5 Flash (timeout 20s, 2 retries)
5. **Load Config** — Lee `/config/personalities/chappie.yaml` y `/config/providers.yaml`
6. **Load Memory** — Lee contexto de sesión desde `/config/memory/sessions/{session_id}.json`
7. **Build Prompt** — Construye prompt: system prompt + memoria + input del usuario
8. **LLM Call** — Llama a OpenCode API (timeout 30s, 2 retries con fallback a Gemini)
9. **Parse Response** — Parsea y valida JSON contra schema definido (con extracción forzada)
10. **Publish to RabbitMQ** — Publica JSON estructurado en `chappie.responses` con delivery_mode=2, header `X-Idempotency-Key`, TTL=60s
11. **Update Memory** — Guarda contexto de conversación en archivo de sesión
12. **Respond to Webhook** — Retorna HTTP 202 con `workflow_execution_id`
13. **Error Handler** — Captura errores y publica notificación en `chappie.notifications`

### chappie-error-handler (9 nodos)

Workflow de manejo de errores cuando falla la ejecución de la respuesta original:

1. **Webhook Trigger** — Recibe error vía `POST /webhook/chappie-error-handler`
2. **Validate Auth** — Valida `X-Webhook-Secret` (misma lógica fail-closed)
3. **Load Config** — Lee personalidad de Chappie
4. **Build Error Prompt** — Construye prompt con error y texto original del usuario
5. **LLM Call** — Llama a OpenCode (timeout 15s, 1 retry con fallback)
6. **Parse Response** — Extrae `voice_response` con fallback a texto plano
7. **Publish to RabbitMQ** — Publica en `chappie.tts.requests` con prioridad 10 (high), delivery_mode=2, TTL=30s
8. **Respond to Webhook** — Retorna HTTP 202 con `workflow_execution_id`
9. **Error Handler** — Captura errores internos y publica en `chappie.notifications`

## Configuración

### Archivos YAML

| Archivo | Propósito |
|---|---|
| `config/providers.yaml` | Mapeo de proveedores: STT (Gemini), procesamiento (OpenCode + fallback Gemini), RabbitMQ |
| `config/personalities/chappie.yaml` | Personalidad de Chappie: identidad, tono, system prompt, reglas |
| `config/memory/memory-config.yaml` | Configuración de memoria: máximo 20 mensajes/sesión, TTL 24h, resumen automático |

### Prerrequisitos

Las siguientes variables de entorno deben estar definidas en el contenedor n8n (`chappie-n8n` en docker-compose):

- `N8N_WEBHOOK_SECRET` — Secreto compartido para autenticación de webhooks (mín. 32 caracteres)
- `GEMINI_API_KEY` — API key de Google Gemini para STT y LLM fallback
- `RABBITMQ_USER` — Usuario de RabbitMQ
- `RABBITMQ_PASSWORD` — Contraseña de RabbitMQ

## Integraciones

- **chappie-daemon:** Envía audio vía webhook `POST /webhook/chappie-voice-capture`
- **chappie-notification:** Consume colas RabbitMQ (`chappie.responses`, `chappie.tts.requests`, `chappie.notifications`)
- **Gemini API:** STT y procesamiento (fallback)
- **OpenCode API:** Procesamiento primario de LLM
- **RabbitMQ:** Bus de eventos en `chappie-rabbitmq:5672`

---

*Proyecto parte del workspace chappie-workspace*
*Documentación técnica: `docs/specs/master_spec.md`*
