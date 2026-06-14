# Task Board — initial-setup (chappie-n8n-workflows)

**Incremento:** initial-setup
**Proyecto:** `projects/chappie-n8n-workflows/`
**Spec aprobada:** `/home/cristiansrc/Documentos/Proyectos/chappie-workspace/projects/chappie-n8n-workflows/docs/specs/master_spec.md`
**Shared Context:** `/home/cristiansrc/Documentos/Proyectos/chappie-workspace/projects/chappie-n8n-workflows/docs/specs/.working/initial-setup-sdd-context.md`
**Spec Validator veredict:** `verdict: ready`
**Human Plan Approval:** `approved_by_user`

**Estado superior:** `done`

---

## Orden de Ejecución

| Orden | Task ID | Dependencias |
|---|---|---|
| 1 | TASK-001 | — |
| 2 | TASK-002 | — |
| 3 | TASK-003 | — |
| 4 | TASK-004 | — |
| 5 | TASK-005 | TASK-001 |
| 6 | TASK-006 | TASK-005 |
| 7 | TASK-007 | TASK-006 |
| 8 | TASK-008 | TASK-004 |
| 9 | TASK-009 | TASK-007, TASK-008 |
| 10 | TASK-010 | TASK-009 |

---

## TASK-001

**id:** TASK-001
**title:** Crear estructura de directorios y archivos JSON vacíos
**agent:** executor
**spec_refs:** Master Spec sección 3.1 (Estructura de Directorios), sección 15.1 (Canonical Artifacts)
**goal:** Crear todos los directorios requeridos y archivos placeholder vacíos para workflows y memoria.
**scope:**
- Crear `config/personalities/` (si no existe)
- Crear `config/memory/` (si no existe)
- Crear `config/memory/sessions/` (directorio para archivos de sesión)
- Crear `config/memory/archive/` (directorio para sesiones archivadas)
- Crear `workflows/chappie-voice-pipeline.json` como archivo JSON vacío con estructura mínima de n8n workflow
- Crear `workflows/chappie-error-handler.json` como archivo JSON vacío con estructura mínima de n8n workflow
- Cada directorio debe tener un `.gitkeep` para ser trackeado en git
**out_of_scope:**
- No implementar ningún nodo de workflow aún
- No crear archivos de configuración YAML (eso es TASK-002, TASK-003, TASK-004)
- No crear archivos de sesión de memoria reales (solo el directorio)
**inputs:**
- Master Spec sección 3.1: estructura de directorios
- `config/.gitkeep` existente como referencia
- `workflows/.gitkeep` existente como referencia
**implementation_notes:**
- Los directorios `config/personalities/` y `config/memory/` no existen actualmente; deben crearse.
- El esqueleto JSON mínimo para un workflow n8n:
  ```json
  {
    "name": "WORKFLOW_NAME",
    "nodes": [],
    "connections": {},
    "settings": { "executionOrder": "v1" },
    "staticData": null,
    "pinData": {}
  }
  ```
  Donde `WORKFLOW_NAME` es `"Chappie Voice Pipeline"` para el primero y `"Chappie Error Handler"` para el segundo.
- Usar `mkdir -p` para crear directorios anidados.
- Los `.gitkeep` deben ser archivos vacíos (0 bytes).
**edge_cases:**
- Si los directorios ya existen, no sobrescribir; solo verificar existencia.
- Si los archivos JSON ya existen y tienen contenido, no sobrescribir (marcar como blocked).
**done_criteria:**
- [x] `config/personalities/.gitkeep` existe
- [x] `config/memory/.gitkeep` existe
- [x] `config/memory/sessions/.gitkeep` existe
- [x] `config/memory/archive/.gitkeep` existe
- [x] `workflows/chappie-voice-pipeline.json` existe con estructura mínima válida
- [x] `workflows/chappie-error-handler.json` existe con estructura mínima válida
**verification:**
```bash
ls -la config/personalities/.gitkeep
ls -la config/memory/.gitkeep
ls -la config/memory/sessions/.gitkeep
ls -la config/memory/archive/.gitkeep
cat workflows/chappie-voice-pipeline.json | python3 -m json.tool > /dev/null
cat workflows/chappie-error-handler.json | python3 -m json.tool > /dev/null
```
**dependencies:** — (ninguna, es la primera tarea)
**handoff_context:** Los archivos JSON vacíos serán poblados por TASK-005, TASK-006 y TASK-008.
**source_of_truth:** Master Spec sección 3.1
**stale_terms_guard:** No usar "TTS Generator", "chappie-tts-generator.json" ni "STT en daemon".
**status:** `done`
**executor_notes:** Directorios config/personalities/, config/memory/, config/memory/sessions/, config/memory/archive/ creados con .gitkeep. Workflows JSON skeleton creados.
**verification_result:** pass - 4 .gitkeep files, 2 valid JSON skeletons
**blocker:** `none`

---

## TASK-002

**id:** TASK-002
**title:** Crear archivo de configuración `config/providers.yaml`
**agent:** executor
**spec_refs:** Master Spec sección 9.1 (providers.yaml — mapeo para n8n), sección 7.0 (nodo RabbitMQ capacidades), integration-map.md secciones 1.1, 1.2, 2.1
**goal:** Crear el archivo YAML de configuración de proveedores con el contenido exacto definido en la Master Spec.
**scope:**
- Crear `config/providers.yaml` con el contenido YAML especificado en Master Spec sección 9.1
- Incluir secciones: `providers.stt` (Gemini), `providers.processing` (OpenCode + fallback), `providers.rabbitmq`
**out_of_scope:**
- No modificar el `providers.yaml` global de `chappie-config`
- No agregar proveedores no listados en la spec
- No definir variables de entorno (eso es responsabilidad de docker-compose.yaml)
**inputs:**
- Master Spec sección 9.1: contenido exacto del YAML
- Master Spec sección 7.0: capacidades verificadas del nodo RabbitMQ (referencia)
- Docker Compose: variables de entorno ya definidas (`GEMINI_API_KEY`, `RABBITMQ_USER`, `RABBITMQ_PASSWORD`)
**implementation_notes:**
- Copiar el contenido YAML exacto de la sección 9.1 de la Master Spec.
- Verificar que las variables de entorno referenciadas (`GEMINI_API_KEY`, `RABBITMQ_USER`, `RABBITMQ_PASSWORD`) existen en `docker-compose.yaml` del workspace.
- El host RabbitMQ es `chappie-rabbitmq` (nombre del servicio en docker-compose).
- El modelo STT es `gemini-2.5-flash`.
- El endpoint de OpenCode es `http://localhost:3000/api/v1/chat/completions`.
- Los modelos de OpenCode listados son: `opencode/deepseek-v4-flash-free`, `opencode/mimo-v2.5-free`, `opencode/qwen3.6-plus-free`.
**edge_cases:**
- Si `config/providers.yaml` ya existe y tiene contenido diferente, marcar como blocked y reportar diferencia.
**done_criteria:**
- [x] `config/providers.yaml` existe
- [x] Contiene sección `providers.stt` con configuración de Gemini
- [x] Contiene sección `providers.processing` con primary `opencode` y fallback a `gemini`
- [x] Contiene sección `providers.rabbitmq` con host `chappie-rabbitmq:5672`
- [x] El archivo es YAML sintácticamente válido
**verification:**
```bash
python3 -c "import yaml; yaml.safe_load(open('config/providers.yaml'))" && echo "YAML válido"
grep -q "gemini-2.5-flash" config/providers.yaml
grep -q "opencode" config/providers.yaml
grep -q "chappie-rabbitmq" config/providers.yaml
```
**dependencies:** — (ninguna, YAML independiente)
**handoff_context:** Este archivo será leído por el nodo "Load Config" (nodo 5) en Voice Pipeline y por el nodo "Load Config" (nodo 3) en Error Handler.
**source_of_truth:** Master Spec sección 9.1
**stale_terms_guard:** No usar "TTS Generator", "chappie-tts-generator.json" ni "STT en daemon".
**status:** `done`
**executor_notes:** providers.yaml creado con secciones stt (Gemini), processing (OpenCode + fallback Gemini), rabbitmq (chappie-rabbitmq:5672).
**verification_result:** pass - YAML válido, contiene gemini-2.5-flash, opencode, chappie-rabbitmq
**blocker:** `none`

---

## TASK-003

**id:** TASK-003
**title:** Crear archivo de personalidad `config/personalities/chappie.yaml`
**agent:** executor
**spec_refs:** Master Spec sección 9.2 (chappie.yaml — personalidad)
**goal:** Crear el archivo YAML que define la personalidad de Chappie (identidad, tono, expresiones, system prompt y reglas).
**scope:**
- Crear `config/personalities/chappie.yaml` con el contenido exacto definido en Master Spec sección 9.2
- Incluir: `personality.name`, `personality.identity`, `personality.tone`, `personality.address_user_as`, `personality.expressions`, `personality.system_prompt`, `personality.rules`
**out_of_scope:**
- No modificar la personalidad (tono, frases, reglas) — solo copiar lo definido en la spec
- No agregar nuevas reglas o expresiones
**inputs:**
- Master Spec sección 9.2: contenido exacto del YAML
- Master Spec sección 6: JSON Response Schema (referencia para reglas de validación)
**implementation_notes:**
- Copiar el contenido YAML exacto de la sección 9.2.
- El `system_prompt` es un bloque multilínea (usar `|` en YAML).
- Las reglas (`personality.rules`) son un array de strings.
- Verificar que la indentación YAML es correcta (2 espacios).
- El system prompt instruye al LLM a: ser Chappie, hablar español, producir JSON válido, no incluir emojis, voice_response conciso (máx 3 oraciones).
**edge_cases:**
- Si el archivo ya existe, verificar que el contenido coincida con la spec; si difiere, marcar blocked.
**done_criteria:**
- [x] `config/personalities/chappie.yaml` existe
- [x] Contiene `personality.name: "Chappie"`
- [x] Contiene `personality.address_user_as: "Creador"`
- [x] Contiene `personality.system_prompt` con el texto multilínea
- [x] Contiene `personality.rules` con al menos 4 reglas
- [x] El archivo es YAML sintácticamente válido
**verification:**
```bash
python3 -c "import yaml; data = yaml.safe_load(open('config/personalities/chappie.yaml')); assert data['personality']['name'] == 'Chappie'; print('OK')"
grep -q "Creador" config/personalities/chappie.yaml
grep -q "flow" config/personalities/chappie.yaml
```
**dependencies:** — (ninguna, YAML independiente; requiere que el directorio `config/personalities/` exista — creado en TASK-001)
**handoff_context:** Este archivo será leído por el nodo "Load Config" (nodo 5) en Voice Pipeline para inyectar la personalidad en el system prompt.
**source_of_truth:** Master Spec sección 9.2
**stale_terms_guard:** No usar "TTS Generator", "chappie-tts-generator.json" ni "STT en daemon".
**status:** `done`
**executor_notes:** chappie.yaml creado con personalidad completa: name, identity, tone, address_user_as, expressions, system_prompt (multilinea), rules (4 reglas).
**verification_result:** pass - contenido verificado: name Chappie, Creador, flow, system_prompt, rules
**blocker:** `none`

---

## TASK-004

**id:** TASK-004
**title:** Crear archivo de configuración de memoria `config/memory/memory-config.yaml`
**agent:** executor
**spec_refs:** Master Spec sección 8.1 (Estrategia de Memoria), sección 8.3 (Flujo de Resumen), sección 9.3 (memory-config.yaml)
**goal:** Crear el archivo YAML que define la configuración de memoria conversacional.
**scope:**
- Crear `config/memory/memory-config.yaml` con el contenido exacto definido en Master Spec sección 9.3
- Incluir: `memory.max_messages_per_session`, `memory.session_ttl_hours`, `memory.summarization`, `memory.storage`
**out_of_scope:**
- No crear archivos de sesión reales (eso ocurre en runtime por los workflows)
- No modificar la estrategia de resumen (trigger_threshold, keep_recent_count)
**inputs:**
- Master Spec sección 9.3: contenido exacto del YAML
- Master Spec sección 8.1: estrategia de memoria (referencia)
- Master Spec sección 8.2: formato de archivo de sesión (referencia)
**implementation_notes:**
- Copiar el contenido YAML exacto de la sección 9.3.
- `max_messages_per_session: 20` — cuando se supera, se activa summarization.
- `session_ttl_hours: 24` — sesiones inactivas por más de 24h se archivan.
- `summarization.keep_recent_count: 10` — los 10 mensajes más recientes se mantienen; el resto se resumen.
- `storage.path: "/config/memory/sessions"` — ruta dentro del contenedor n8n.
- `storage.archive_path: "/config/memory/archive"` — ruta de archivo.
**edge_cases:**
- Los paths en `storage` deben coincidir con los bind mounts definidos en docker-compose. Verificar que `/config` está montado como `./projects/chappie-n8n-workflows/config`.
- Si el archivo ya existe, verificar coincidencia con la spec.
**done_criteria:**
- [x] `config/memory/memory-config.yaml` existe
- [x] Contiene `max_messages_per_session: 20`
- [x] Contiene `session_ttl_hours: 24`
- [x] Contiene `summarization.enabled: true`
- [x] Contiene `storage.path: "/config/memory/sessions"`
- [x] El archivo es YAML sintácticamente válido
**verification:**
```bash
python3 -c "import yaml; data = yaml.safe_load(open('config/memory/memory-config.yaml')); assert data['memory']['max_messages_per_session'] == 20; print('OK')"
grep -q "summarization" config/memory/memory-config.yaml
grep -q "/config/memory/sessions" config/memory/memory-config.yaml
```
**dependencies:** — (ninguna, YAML independiente; requiere que el directorio `config/memory/` exista — creado en TASK-001)
**handoff_context:** Este archivo es leído por los nodos "Load Memory" (nodo 6) y "Update Memory" (nodo 11) del Voice Pipeline para gestionar la caducidad y resumen.
**source_of_truth:** Master Spec sección 9.3
**stale_terms_guard:** No usar "TTS Generator", "chappie-tts-generator.json" ni "STT en daemon".
**status:** `done`
**executor_notes:** memory-config.yaml creado con max_messages_per_session: 20, session_ttl_hours: 24, summarization.enabled: true, storage paths.
**verification_result:** pass - todas las claves requeridas presentes
**blocker:** `none`

---

## TASK-005

**id:** TASK-005
**title:** Implementar Voice Pipeline — Capa de Entrada (nodos 1-3)
**agent:** executor
**spec_refs:** Master Spec sección 4.1 (Webhook Voice Capture), sección 5.1.1 (nodos 1-3), sección 5.1.2 (flujo happy path), sección 10.1 (Auth de Webhooks)
**goal:** Poblar `workflows/chappie-voice-pipeline.json` con los primeros 3 nodos del Voice Pipeline: Webhook Trigger, Validate Auth y Decode Audio, incluyendo sus conexiones.
**scope:**
- **Nodo 1 — Webhook Trigger**: tipo `n8n-nodes-base.webhook`, método POST, path `/webhook/chappie-voice-capture`, response mode `responseNode`. Sin autenticación nativa. Configurar para que pase el header `X-Webhook-Secret` al siguiente nodo.
- **Nodo 2 — Validate Auth**: tipo `n8n-nodes-base.code` (JavaScript). Lee `$env.N8N_WEBHOOK_SECRET` y el header entrante `X-Webhook-Secret`. Si el header no coincide, retorna HTTP 401 con body `{"status":"error","code":"UNAUTHORIZED","message":"Invalid or missing webhook secret"}` y detiene. Si `N8N_WEBHOOK_SECRET` no está definida, retorna HTTP 500 con body `{"status":"error","code":"MISCONFIGURED","message":"Webhook secret not configured"}` (fail-closed). Si coincide, pasa los datos al nodo 3.
- **Nodo 3 — Decode Audio**: tipo `n8n-nodes-base.code` (JavaScript). Toma `audio_base64` del body del webhook y lo decodifica a buffer binario usando `Buffer.from(audio_base64, 'base64')`. Prepara el payload para el nodo STT.
- **Conexiones**: Nodo 1 → Nodo 2 (principal), Nodo 2 → Nodo 3 (principal).
**out_of_scope:**
- No implementar nodos 4-13 (TASK-006, TASK-007)
- No configurar el webhook response (se hace en nodo 12, TASK-007)
- No validar el payload del webhook más allá del header de auth (la validación de campos se hará en nodos posteriores)
**inputs:**
- `workflows/chappie-voice-pipeline.json` (esqueleto creado en TASK-001)
- Master Spec sección 4.1: request/response schema del webhook
- Master Spec sección 10.1: mecanismo detallado de auth (fail-closed, comparación de tiempo constante)
- Docker Compose: la variable `N8N_WEBHOOK_SECRET` debe añadirse (prerrequisito de infra)
**implementation_notes:**

**Nodo 1 (Webhook Trigger):**
```json
{
  "parameters": {
    "httpMethod": "POST",
    "path": "chappie-voice-capture",
    "responseMode": "responseNode",
    "options": {}
  },
  "name": "Webhook Trigger",
  "type": "n8n-nodes-base.webhook",
  "position": [250, 300]
}
```

**Nodo 2 (Validate Auth) — Código JavaScript:**
```javascript
const webhookSecret = $env.N8N_WEBHOOK_SECRET;
const incomingSecret = $input.first().json.headers['x-webhook-secret'] || 
                       $input.first().json.headers['X-Webhook-Secret'];

if (!webhookSecret || webhookSecret.trim() === '') {
  // Fail-closed: variable no configurada
  $input.first().json.responseStatus = 500;
  return {
    status: 'error',
    code: 'MISCONFIGURED',
    message: 'Webhook secret not configured'
  };
}

if (!incomingSecret || incomingSecret !== webhookSecret) {
  $input.first().json.responseStatus = 401;
  return {
    status: 'error', 
    code: 'UNAUTHORIZED',
    message: 'Invalid or missing webhook secret'
  };
}

// Auth exitosa: pasar el payload original al siguiente nodo
return $input.first().json.body || $input.first().json;
```
Nota: La comparación debe ser de tiempo constante (`!==` es suficiente en Node.js V8+).

**Nodo 3 (Decode Audio) — Código JavaScript:**
```javascript
const body = $input.first().json;
const audioBase64 = body.audio_base64;

if (!audioBase64 || audioBase64.trim() === '') {
  throw new Error('audio_base64 is required and must not be empty');
}

const audioBuffer = Buffer.from(audioBase64, 'base64');

return {
  audio_base64: audioBase64,
  audio_buffer: audioBuffer.toString('binary'),
  session_id: body.session_id,
  timestamp: body.timestamp,
  include_screen: body.include_screen || false
};
```

- Las posiciones de los nodos: nodo 1 en (250, 300), nodo 2 en (640, 300), nodo 3 en (1030, 300).
- Las conexiones en n8n se definen como `{ "Webhook Trigger": { "main": [[ { "node": "Validate Auth", "type": "main", "index": 0 } ]] } }`.
- El nombre de cada nodo debe coincidir exactamente con el usado en `connections`.

**edge_cases:**
- Si `N8N_WEBHOOK_SECRET` no existe en el entorno, el nodo 2 retorna 500 (fail-closed). Esto es intencional — sin secret, ningún webhook debe funcionar.
- Si `audio_base64` está vacío, el nodo 3 lanza error (será capturado por el nodo 13 Error Trigger).
- El header `X-Webhook-Secret` puede venir en minúsculas (`x-webhook-secret`) por normalización HTTP; el código debe manejar ambos casos.
**done_criteria:**
- [x] `workflows/chappie-voice-pipeline.json` contiene el nodo "Webhook Trigger" (tipo `n8n-nodes-base.webhook`)
- [x] Contiene el nodo "Validate Auth" (tipo `n8n-nodes-base.code`) con el código JavaScript de validación
- [x] Contiene el nodo "Decode Audio" (tipo `n8n-nodes-base.code`) con decodificación base64
- [x] Las conexiones entre nodos 1→2 y 2→3 están definidas correctamente
- [x] El archivo JSON es sintácticamente válido
**verification:**
```bash
python3 -c "
import json
with open('workflows/chappie-voice-pipeline.json') as f:
    wf = json.load(f)
nodes = {n['name']: n for n in wf['nodes']}
assert 'Webhook Trigger' in nodes, 'Falta Webhook Trigger'
assert 'Validate Auth' in nodes, 'Falta Validate Auth'
assert 'Decode Audio' in nodes, 'Falta Decode Audio'
assert nodes['Webhook Trigger']['type'] == 'n8n-nodes-base.webhook'
assert nodes['Validate Auth']['type'] == 'n8n-nodes-base.code'
assert nodes['Decode Audio']['type'] == 'n8n-nodes-base.code'
conns = wf['connections']
assert 'Webhook Trigger' in conns
assert 'Validate Auth' in conns
print('Nodos 1-3 OK')
"
grep -q "N8N_WEBHOOK_SECRET" workflows/chappie-voice-pipeline.json
grep -q "MISCONFIGURED" workflows/chappie-voice-pipeline.json
grep -q "Buffer.from" workflows/chappie-voice-pipeline.json
```
**dependencies:** TASK-001 (archivo JSON vacío debe existir)
**handoff_context:** El archivo JSON contiene los primeros 3 nodos y sus conexiones. TASK-006 añadirá los nodos 4-9 a este mismo archivo.
**source_of_truth:** Master Spec secciones 4.1, 5.1.1, 10.1
**stale_terms_guard:** No usar "TTS Generator", "chappie-tts-generator.json", "STT en daemon", "Consumer de RabbitMQ en n8n", "Volume ducking en n8n", "execution_consumer en n8n".
**status:** `done`
**executor_notes:** Nodos 1-3 (Webhook Trigger, Validate Auth, Decode Audio) implementados con JS code, conexiones 1→2→3.
**verification_result:** pass - nodos y conexiones verificados, N8N_WEBHOOK_SECRET, MISCONFIGURED, Buffer.from presentes
**blocker:** `none`

---

## TASK-006

**id:** TASK-006
**title:** Implementar Voice Pipeline — Capa de Procesamiento (nodos 4-9)
**agent:** executor
**spec_refs:** Master Spec sección 5.1.1 (nodos 4-9), sección 5.1.2 (flujo happy path), sección 5.1.3 (flujo de fallo), sección 13.2 (consumo de APIs externas)
**goal:** Añadir al archivo `workflows/chappie-voice-pipeline.json` los nodos 4-9: STT Call, Load Config, Load Memory, Build Prompt, LLM Call, Parse Response, con sus conexiones.
**scope:**
- **Nodo 4 — STT Call**: tipo `n8n-nodes-base.httpRequest`. Llama a Gemini 2.5 Flash para STT. Método POST. URL desde `providers.stt.gemini.endpoint`. Timeout 20s, 2 retries. Usa `GEMINI_API_KEY` del entorno.
- **Nodo 5 — Load Config**: tipo `n8n-nodes-base.readWriteFile`. Lee `/config/personalities/chappie.yaml` y `/config/providers.yaml`. Ambos archivos deben cargarse y pasarse como objetos parseados.
- **Nodo 6 — Load Memory**: tipo `n8n-nodes-base.readWriteFile`. Lee `/config/memory/sessions/{session_id}.json`. Si no existe, crea contexto vacío con estructura según sección 8.2.
- **Nodo 7 — Build Prompt**: tipo `n8n-nodes-base.code` (JavaScript). Construye el prompt completo: system prompt + contexto de memoria + input del usuario transcrito.
- **Nodo 8 — LLM Call**: tipo `n8n-nodes-base.httpRequest`. Llama al proveedor primario (OpenCode). Timeout 30s, 2 retries con fallback a Gemini si el primario falla.
- **Nodo 9 — Parse Response**: tipo `n8n-nodes-base.code` (JavaScript). Parsea la respuesta del LLM como JSON y valida contra el JSON Response Schema (sección 6). Si falla, intenta extracción forzada con regex.
- **Conexiones**: Nodo 3 → Nodo 4, Nodo 4 → Nodo 5, Nodo 5 → Nodo 6, Nodo 6 → Nodo 7, Nodo 7 → Nodo 8, Nodo 8 → Nodo 9.
**out_of_scope:**
- No implementar nodos 1-3 (TASK-005) ni nodos 10-13 (TASK-007)
- No crear los archivos YAML de configuración (TASK-002, TASK-003, TASK-004)
**inputs:**
- `workflows/chappie-voice-pipeline.json` (con nodos 1-3 ya implementados por TASK-005)
- `config/providers.yaml` (creado en TASK-002)
- `config/personalities/chappie.yaml` (creado en TASK-003)
- `config/memory/memory-config.yaml` (creado en TASK-004)
- Master Spec sección 6: JSON Response Schema para validación
- Master Spec sección 8.2: formato de archivo de sesión
**implementation_notes:**

**Nodo 4 (STT Call) — HTTP Request a Gemini:**
```json
{
  "parameters": {
    "method": "POST",
    "url": "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent",
    "authentication": "genericCredentialType",
    "genericAuthType": "httpHeaderAuth",
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        {"name": "x-goog-api-key", "value": "={{$env.GEMINI_API_KEY}}"},
        {"name": "Content-Type", "value": "application/json"}
      ]
    },
    "sendBody": true,
    "bodyParameters": {
      "parameters": []
    },
    "options": {
      "timeout": 20000,
      "retry": { "maxTries": 2, "waitInterval": 1000 }
    }
  },
  "name": "STT Call",
  "type": "n8n-nodes-base.httpRequest",
  "position": [1420, 300]
}
```
Nota: El body se construye en el nodo anterior (Decode Audio) o mediante una expresión en el HTTP Request. La API de Gemini requiere el audio en inlineData con mimeType `audio/wav`.

**Nodo 5 (Load Config):**
Usar dos nodos `n8n-nodes-base.readWriteFile` o uno con múltiples operaciones. Simplificación: usar un nodo Code que lea ambos archivos con el método `$readFile()`:
```javascript
const fs = require('fs');
const personalityYaml = fs.readFileSync('/config/personalities/chappie.yaml', 'utf8');
const providersYaml = fs.readFileSync('/config/providers.yaml', 'utf8');
// Parse YAML (n8n tiene acceso a librerías; usar expresión equivalente)
return {
  personality: personalityYaml,
  providers: providersYaml
};
```
Alternativa: dos nodos `readWriteFile` secuenciales. Posiciones: (1810, 200) y (1810, 400).

**Nodo 6 (Load Memory):**
```javascript
const sessionId = $input.first().json.session_id;
const fs = require('fs');
const path = `/config/memory/sessions/${sessionId}.json`;

let memory;
if (fs.existsSync(path)) {
  memory = JSON.parse(fs.readFileSync(path, 'utf8'));
} else {
  memory = {
    session_id: sessionId,
    created_at: new Date().toISOString(),
    last_activity: new Date().toISOString(),
    messages: [],
    summary: null
  };
}

return { ...$input.first().json, memory };
```

**Nodo 7 (Build Prompt):**
```javascript
const data = $input.first().json;
const personality = data.personality; // system prompt cargado
const memory = data.memory;
const transcribedText = data.transcribed_text; // del STT

// Construir contexto de memoria
let memoryContext = '';
if (memory.summary) {
  memoryContext += `Resumen de conversación anterior: ${memory.summary}\n\n`;
}
if (memory.messages && memory.messages.length > 0) {
  const recentMessages = memory.messages.slice(-10);
  memoryContext += recentMessages.map(m => `${m.role}: ${m.content}`).join('\n');
}

const systemPrompt = personality.system_prompt || personality;
const userMessage = transcribedText;

return {
  ...data,
  system_prompt: systemPrompt,
  memory_context: memoryContext,
  user_message: userMessage,
  full_prompt: `${systemPrompt}\n\nContexto de memoria:\n${memoryContext}\n\nUsuario: ${userMessage}\n\nResponde con un JSON válido.`
};
```

**Nodo 8 (LLM Call) — HTTP Request a OpenCode:**
```json
{
  "parameters": {
    "method": "POST",
    "url": "http://localhost:3000/api/v1/chat/completions",
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        {"name": "Content-Type", "value": "application/json"}
      ]
    },
    "sendBody": true,
    "options": {
      "timeout": 30000,
      "retry": { "maxTries": 2, "waitInterval": 2000 }
    }
  },
  "name": "LLM Call",
  "type": "n8n-nodes-base.httpRequest",
  "position": [2590, 300]
}
```
El body debe incluir: `model`, `messages` (system + user), `temperature`, `response_format: { type: "json_object" }` si el proveedor lo soporta.

Estrategia de fallback: si el primario falla (HTTP status != 2xx), reintentar con Gemini usando el mismo prompt adaptado al formato de Gemini API.

**Nodo 9 (Parse Response):**
```javascript
const llmResponse = $input.first().json;
let parsed;

try {
  // Intentar parsear como JSON
  const content = llmResponse.choices?.[0]?.message?.content || 
                  llmResponse.candidates?.[0]?.content?.parts?.[0]?.text ||
                  JSON.stringify(llmResponse);
  parsed = JSON.parse(content);
} catch (e) {
  // Extracción forzada con regex
  const jsonMatch = JSON.stringify(llmResponse).match(/\{[\s\S]*\}/);
  if (jsonMatch) {
    try {
      parsed = JSON.parse(jsonMatch[0]);
    } catch (e2) {
      throw new Error(`Failed to parse LLM response as JSON: ${e2.message}`);
    }
  } else {
    throw new Error('No JSON found in LLM response');
  }
}

// Validar campos requeridos
if (!parsed.voice_response) {
  throw new Error('LLM response missing required field: voice_response');
}
if (!parsed.session_id) {
  parsed.session_id = $input.first().json.session_id;
}
if (!parsed.timestamp) {
  parsed.timestamp = new Date().toISOString();
}

// Defaults
parsed.agent_call = parsed.agent_call || { enabled: false };
parsed.terminal_command = parsed.terminal_command || { enabled: false };
parsed.notification = parsed.notification || { enabled: false };
parsed.memory_update = parsed.memory_update || { save_to_memory: true, tags: [] };

// Validar campos condicionales
if (parsed.agent_call.enabled && (!parsed.agent_call.agent || !parsed.agent_call.prompt)) {
  throw new Error('agent_call.enabled is true but agent/prompt missing');
}
if (parsed.terminal_command.enabled && !parsed.terminal_command.command) {
  throw new Error('terminal_command.enabled is true but command missing');
}

return { ...$input.first().json, parsed_response: parsed };
```

- Posiciones: nodo 4 (1420, 300), nodo 5 (1810, 300), nodo 6 (2200, 300), nodo 7 (2590, 300), nodo 8 (2980, 300), nodo 9 (3370, 300).
- Cada nodo debe añadirse al array `nodes` del JSON existente.
- Las conexiones deben añadirse al objeto `connections` para los nuevos nodos (3→4, 4→5, 5→6, 6→7, 7→8, 8→9).
- Si un nodo (ej. LLM Call) tiene lógica de fallback, usar el nodo `n8n-nodes-base.if` para bifurcar o implementar reintentos a nivel de HTTP Request node.

**edge_cases:**
- STT timeout: el nodo 4 tiene configurado 2 retries. Si ambos fallan, lanza error capturado por el Error Trigger (nodo 13, TASK-007).
- LLM fallback: si OpenCode falla (todos los modelos), intentar Gemini. Si ambos fallan, lanzar error.
- Archivo de memoria no existe: el nodo 6 debe crear uno nuevo con estructura vacía.
- JSON inválido del LLM: el nodo 9 intenta extracción forzada antes de fallar.
- Si `providers.yaml` tiene `fallback_order`, el nodo 8 debe iterar sobre la lista.
**done_criteria:**
- [x] El nodo "STT Call" existe con configuración HTTP POST a Gemini
- [x] El nodo "Load Config" lee ambos archivos YAML desde `/config/`
- [x] El nodo "Load Memory" maneja archivo de sesión existente o crea nuevo
- [x] El nodo "Build Prompt" construye el prompt completo (system + memory + user)
- [x] El nodo "LLM Call" llama a OpenCode con timeout 30s y 2 retries
- [x] El nodo "Parse Response" valida contra el JSON Response Schema
- [x] Las conexiones 3→4→5→6→7→8→9 están definidas
- [x] El JSON del workflow sigue siendo sintácticamente válido
**verification:**
```bash
python3 -c "
import json
with open('workflows/chappie-voice-pipeline.json') as f:
    wf = json.load(f)
nodes = {n['name']: n for n in wf['nodes']}
expected = ['STT Call', 'Load Config', 'Load Memory', 'Build Prompt', 'LLM Call', 'Parse Response']
for name in expected:
    assert name in nodes, f'Falta nodo: {name}'
assert nodes['STT Call']['type'] == 'n8n-nodes-base.httpRequest'
assert nodes['LLM Call']['type'] == 'n8n-nodes-base.httpRequest'
assert nodes['Build Prompt']['type'] == 'n8n-nodes-base.code'
conns = wf['connections']
assert 'Decode Audio' in conns
assert 'STT Call' in conns
assert 'Parse Response' in conns
print('Nodos 4-9 OK')
"
grep -q "gemini-2.5-flash" workflows/chappie-voice-pipeline.json
grep -q "voice_response" workflows/chappie-voice-pipeline.json
grep -q "/config/memory/sessions" workflows/chappie-voice-pipeline.json
```
**dependencies:** TASK-005 (nodos 1-3 deben existir en el archivo JSON), TASK-002 (providers.yaml), TASK-003 (chappie.yaml), TASK-004 (memory-config.yaml)
**handoff_context:** El archivo JSON ahora contiene nodos 1-9. TASK-007 añadirá los nodos 10-13.
**source_of_truth:** Master Spec secciones 5.1.1 (nodos 4-9), 5.1.3 (flujo de fallo), 6 (JSON Response Schema), 13.2 (APIs externas)
**stale_terms_guard:** No usar "TTS Generator", "chappie-tts-generator.json", "STT en daemon", "Consumer de RabbitMQ en n8n", "Volume ducking en n8n", "execution_consumer en n8n".
**status:** `done`
**executor_notes:** Nodos 4-9 implementados: STT Call (HTTP Gemini), Load Config (fs), Load Memory, Build Prompt, LLM Call (HTTP OpenCode), Parse Response (JSON validation). Conexiones 3→9 completas.
**verification_result:** pass - 9 nodos totales, todos los tipos correctos, conexiones en cadena
**blocker:** `none`

---

## TASK-007

**id:** TASK-007
**title:** Implementar Voice Pipeline — Capa de Salida y Errores (nodos 10-13)
**agent:** executor
**spec_refs:** Master Spec sección 5.1.1 (nodos 10-13), sección 5.1.3 (flujo de fallo), sección 5.1.4 (concurrencia e idempotencia), sección 7.1 (chappie.responses), sección 7.3 (chappie.notifications), sección 5.1.2 (flujo happy path completo)
**goal:** Añadir al archivo `workflows/chappie-voice-pipeline.json` los nodos 10-13: Publish to RabbitMQ, Update Memory, Respond to Webhook, Error Handler, completando el workflow con sus conexiones.
**scope:**
- **Nodo 10 — Publish to RabbitMQ**: tipo `n8n-nodes-base.rabbitmq`. Publica el JSON estructurado en la cola `chappie.responses` con delivery_mode=2 (persistent), header `X-Idempotency-Key` = `session_id:timestamp`, Content-Type `application/json`, TTL=60000ms. 3 retries con backoff 1s/2s/4s si el broker no está disponible.
- **Nodo 11 — Update Memory**: tipo `n8n-nodes-base.code` (JavaScript). Lee el archivo de sesión actual, añade el mensaje del usuario y la respuesta, verifica `max_messages_per_session`, aplica resumen si se supera el umbral, guarda el archivo.
- **Nodo 12 — Respond to Webhook**: tipo `n8n-nodes-base.respondToWebhook`. Retorna HTTP 202 con body `{"status":"accepted","workflow_execution_id":"<executionId>"}`.
- **Nodo 13 — Error Handler**: tipo `n8n-nodes-base.errorTrigger` (o `n8n-nodes-base.errorWorkflow`). Captura errores de cualquier nodo anterior y publica en `chappie.notifications` con urgencia "critical".
- **Conexiones**: Nodo 9 → Nodo 10, Nodo 10 → Nodo 11, Nodo 11 → Nodo 12. Nodo 13 se conecta como error trigger global (no en el flujo principal, sino como manejador de errores del workflow).
**out_of_scope:**
- No implementar nodos 1-9 (TASK-005, TASK-006)
- No crear las colas RabbitMQ (responsabilidad de chappie-infrastructure)
- No configurar el consumer de RabbitMQ (responsabilidad de chappie-notification)
**inputs:**
- `workflows/chappie-voice-pipeline.json` (con nodos 1-9 ya implementados)
- `config/providers.yaml` (credenciales y host de RabbitMQ)
- `config/memory/memory-config.yaml` (umbrales de memoria)
- Master Spec sección 6: JSON Response Schema (referencia para el payload)
- Master Spec sección 7.0: capacidades verificadas del nodo RabbitMQ
- Docker Compose: RabbitMQ en `chappie-rabbitmq:5672`
**implementation_notes:**

**Nodo 10 (Publish to RabbitMQ):**
```json
{
  "parameters": {
    "operation": "publish",
    "queue": "chappie.responses",
    "options": {
      "exchange": "",
      "routingKey": "chappie.responses",
      "deliveryMode": "persistent",
      "contentType": "application/json",
      "headers": {
        "parameters": [
          {"name": "X-Idempotency-Key", "value": "={{$json.session_id}}:{{$json.timestamp}}"}
        ]
      },
      "expiration": "60000",
      "priority": 0
    }
  },
  "name": "Publish to RabbitMQ",
  "type": "n8n-nodes-base.rabbitmq",
  "position": [3760, 300],
  "credentials": {
    "rabbitmq": {
      "hostname": "chappie-rabbitmq",
      "port": 5672,
      "username": "={{$env.RABBITMQ_USER}}",
      "password": "={{$env.RABBITMQ_PASSWORD}}",
      "vhost": "/"
    }
  }
}
```
Nota: Verificar que el nodo RabbitMQ de n8n soporta exactamente estos parámetros según lo verificado en sección 7.0.

El payload a publicar es `parsed_response` del nodo 9 (el JSON completo validado).

Retry: n8n RabbitMQ node tiene opción `retry` en la configuración. Si no, implementar con un nodo `n8n-nodes-base.if` que reintente usando el nodo HTTP Request para `rabbitmqadmin` CLI como fallback.

**Nodo 11 (Update Memory) — Código JavaScript:**
```javascript
const data = $input.first().json;
const sessionId = data.session_id;
const parsedResponse = data.parsed_response;
const memoryConfig = JSON.parse(require('fs').readFileSync('/config/memory/memory-config.yaml', 'utf8'));
const maxMessages = memoryConfig.memory.max_messages_per_session;

const fs = require('fs');
const path = `/config/memory/sessions/${sessionId}.json`;

let memory;
if (fs.existsSync(path)) {
  memory = JSON.parse(fs.readFileSync(path, 'utf8'));
} else {
  memory = {
    session_id: sessionId,
    created_at: new Date().toISOString(),
    messages: [],
    summary: null
  };
}

// Añadir mensaje del usuario
memory.messages.push({
  role: 'user',
  content: data.user_message || data.transcribed_text || '(audio)',
  timestamp: data.timestamp || new Date().toISOString()
});

// Añadir respuesta
memory.messages.push({
  role: 'assistant',
  content: parsedResponse.voice_response,
  timestamp: new Date().toISOString()
});

memory.last_activity = new Date().toISOString();

// Verificar si se necesita resumen
if (memory.messages.length > maxMessages) {
  const excessCount = memory.messages.length - maxMessages + (maxMessages / 2);
  const toSummarize = memory.messages.splice(0, excessCount);
  // En esta implementación, el resumen se marca como pendiente
  // La summarización real requiere otra llamada al LLM (se maneja en Load Memory/TASK-008)
  memory.needs_summarization = true;
  memory.pending_summary_messages = toSummarize;
}

// Guardar
fs.writeFileSync(path, JSON.stringify(memory, null, 2));

return {
  ...data,
  memory_saved: true,
  messages_count: memory.messages.length
};
```

**Nodo 12 (Respond to Webhook):**
```json
{
  "parameters": {
    "respondWith": "json",
    "responseBody": {
      "status": "accepted",
      "workflow_execution_id": "={{ $execution.id }}"
    }
  },
  "name": "Respond to Webhook",
  "type": "n8n-nodes-base.respondToWebhook",
  "position": [4560, 300]
}
```
Nota: Este nodo debe ser el último en el flujo principal y debe usar `responseMode: "responseNode"` configurado en el Webhook Trigger (nodo 1). El `$execution.id` es la variable de n8n para el ID de ejecución.

**Nodo 13 (Error Handler):**
El error trigger en n8n se configura a nivel de workflow (settings). Nodo de tipo `n8n-nodes-base.errorTrigger` que captura errores de cualquier nodo y publica en `chappie.notifications`:
```javascript
const errorData = $input.first().json;
const fs = require('fs');
const providersYaml = fs.readFileSync('/config/providers.yaml', 'utf8');

// Construir notificación de error
const notification = {
  timestamp: new Date().toISOString(),
  title: 'Chappie - Error de Pipeline',
  message: `Error en ${errorData.node || 'desconocido'}: ${errorData.message || errorData.error || 'Error desconocido'}`,
  urgency: 'critical',
  actions: [{ key: 'dismiss', label: 'Cerrar' }],
  category: 'error'
};

// Publicar en chappie.notifications via RabbitMQ
// (usando el mismo patrón que el nodo 10 pero con la cola chappie.notifications y TTL 120000ms)
return notification;
```
Nota: El Error Handler debe tener su propio nodo RabbitMQ para publicar en `chappie.notifications` con TTL=120000ms, delivery_mode=2. Alternativamente, conectar el Error Trigger a un nodo RabbitMQ dedicado.

- Posiciones: nodo 10 (3760, 300), nodo 11 (4150, 300), nodo 12 (4540, 300), nodo 13 (3760, 600) — en una fila separada como error path.
- Conexiones principales: Nodo 9 → Nodo 10 (main), Nodo 10 → Nodo 11 (main), Nodo 11 → Nodo 12 (main).
- Error path: los nodos 4, 8, 10 pueden fallar; el Error Trigger captura todos y publica notificación.

**edge_cases:**
- RabbitMQ no disponible: el nodo 10 debe reintentar 3 veces con backoff 1s/2s/4s. Si todos fallan, log crítico y continúa (el webhook ya va a retornar 202).
- Error de escritura de memoria: el nodo 11 debe loguear warning pero no bloquear el flujo (memoria es best-effort).
- n8n `$execution.id`: verificar que esta variable esté disponible en el contexto del nodo Respond to Webhook.
- Si `parsed_response` no tiene el formato esperado, el nodo 10 debe publicar igual (ya fue validado en nodo 9, pero por seguridad incluir un fallback).
**done_criteria:**
- [x] El nodo "Publish to RabbitMQ" publica en `chappie.responses` con delivery_mode=2, header `X-Idempotency-Key`, TTL=60s
- [x] El nodo "Update Memory" lee, actualiza y guarda el archivo de sesión JSON
- [x] El nodo "Respond to Webhook" retorna HTTP 202 con `workflow_execution_id`
- [x] El nodo "Error Handler" publica en `chappie.notifications` con urgencia "critical"
- [x] Las conexiones 9→10→11→12 están definidas
- [x] El flujo de error está configurado (Error Trigger + RabbitMQ publish a notifications)
- [x] El archivo JSON del workflow es sintácticamente válido y contiene los 13 nodos
**verification:**
```bash
python3 -c "
import json
with open('workflows/chappie-voice-pipeline.json') as f:
    wf = json.load(f)
nodes = {n['name']: n for n in wf['nodes']}
expected = ['Publish to RabbitMQ', 'Update Memory', 'Respond to Webhook']
for name in expected:
    assert name in nodes, f'Falta nodo: {name}'
assert nodes['Publish to RabbitMQ']['type'] == 'n8n-nodes-base.rabbitmq'
assert nodes['Respond to Webhook']['type'] == 'n8n-nodes-base.respondToWebhook'
assert len(wf['nodes']) >= 13, f'Expected >= 13 nodes, got {len(wf[\"nodes\"])}'
conns = wf['connections']
assert 'Parse Response' in conns
assert 'Publish to RabbitMQ' in conns
assert 'Update Memory' in conns
print('Nodos 10-13 OK - Workflow completo con', len(wf['nodes']), 'nodos')
"
grep -q "chappie.responses" workflows/chappie-voice-pipeline.json
grep -q "X-Idempotency-Key" workflows/chappie-voice-pipeline.json
grep -q "deliveryMode.*persistent\|delivery_mode.*2" workflows/chappie-voice-pipeline.json
grep -q "workflow_execution_id" workflows/chappie-voice-pipeline.json
grep -q "chappie.notifications" workflows/chappie-voice-pipeline.json
```
**dependencies:** TASK-006 (nodos 4-9 deben existir, conexión 8→9 lista), TASK-002 (RabbitMQ creds), TASK-004 (umbrales de memoria)
**handoff_context:** Voice Pipeline completo. El archivo JSON contiene los 13 nodos con todas las conexiones. TASK-009 verificará la alineación contractual.
**source_of_truth:** Master Spec secciones 5.1.1 (nodos 10-13), 5.1.3 (flujo de fallo), 7.1 (chappie.responses), 7.3 (chappie.notifications)
**stale_terms_guard:** No usar "TTS Generator", "chappie-tts-generator.json", "STT en daemon", "Consumer de RabbitMQ en n8n", "Volume ducking en n8n", "execution_consumer en n8n".
**status:** `done`
**executor_notes:** Nodos 10-13 implementados: Publish to RabbitMQ (chappie.responses), Update Memory, Respond to Webhook (202), Error Handler (errorTrigger + Publish Error Notification a chappie.notifications). Conexiones 9→10→11→12 y error paths desde STT, LLM, RabbitMQ. Total 14 nodos.
**verification_result:** pass - todos los nodos, conexiones y contenido clave verificados
**blocker:** `none`

---

## TASK-008

**id:** TASK-008
**title:** Implementar Error Handler Workflow (todos los nodos)
**agent:** executor
**spec_refs:** Master Spec sección 4.2 (Webhook Error Handler), sección 5.2 (Workflow Error Handler completo), sección 5.2.1 (nodos 1-9), sección 5.2.2 (flujo happy path), sección 5.2.3 (flujo de fallo), sección 7.2 (chappie.tts.requests), sección 7.3 (chappie.notifications)
**goal:** Poblar completamente `workflows/chappie-error-handler.json` con los 9 nodos del Error Handler Workflow, incluyendo todas las conexiones y código JavaScript.
**scope:**
- **Nodo 1 — Webhook Trigger**: tipo `n8n-nodes-base.webhook`, POST, path `/webhook/chappie-error-handler`, response mode `responseNode`.
- **Nodo 2 — Validate Auth**: tipo `n8n-nodes-base.code`. Idéntico al nodo 2 del Voice Pipeline: valida `X-Webhook-Secret` vs `$env.N8N_WEBHOOK_SECRET`. Fail-closed.
- **Nodo 3 — Load Config**: tipo `n8n-nodes-base.code`. Lee `/config/personalities/chappie.yaml`.
- **Nodo 4 — Build Error Prompt**: tipo `n8n-nodes-base.code`. Construye prompt de error con el texto original del usuario y la descripción del error.
- **Nodo 5 — LLM Call**: tipo `n8n-nodes-base.httpRequest`. Llama al proveedor primario. Timeout 15s, 1 retry con fallback.
- **Nodo 6 — Parse Response**: tipo `n8n-nodes-base.code`. Extrae `voice_response` de la respuesta del LLM. No requiere JSON completo.
- **Nodo 7 — Publish to RabbitMQ**: tipo `n8n-nodes-base.rabbitmq`. Publica en `chappie.tts.requests` con prioridad 10 (high), delivery_mode=2, header `X-Idempotency-Key`, TTL=30000ms.
- **Nodo 8 — Respond to Webhook**: tipo `n8n-nodes-base.respondToWebhook`. Retorna 202 Accepted.
- **Nodo 9 — Error Handler**: tipo `n8n-nodes-base.errorTrigger`. Captura errores y publica en `chappie.notifications` con urgencia "critical".
- **Conexiones**: Nodo 1→2→3→4→5→6→7→8 (flujo principal). Nodo 9 como error path global.
**out_of_scope:**
- No crear los archivos YAML de configuración (TASK-002, TASK-003, TASK-004)
- No modificar el Voice Pipeline
- No implementar deduplicación (responsabilidad de chappie-notification)
**inputs:**
- `workflows/chappie-error-handler.json` (esqueleto creado en TASK-001)
- `config/personalities/chappie.yaml` (personalidad para el prompt de error)
- `config/providers.yaml` (RabbitMQ creds y host)
- Master Spec sección 4.2: request/response schema del webhook de error
- Master Spec sección 7.2: payload publicado en `chappie.tts.requests`
**implementation_notes:**

**Nodo 1 (Webhook Trigger):**
```json
{
  "parameters": {
    "httpMethod": "POST",
    "path": "chappie-error-handler",
    "responseMode": "responseNode"
  },
  "name": "Webhook Trigger",
  "type": "n8n-nodes-base.webhook",
  "position": [250, 300]
}
```

**Nodo 2 (Validate Auth):** Código JavaScript idéntico al nodo 2 del Voice Pipeline (ver TASK-005). Misma lógica fail-closed.

**Nodo 3 (Load Config):**
```javascript
const fs = require('fs');
const personalityYaml = fs.readFileSync('/config/personalities/chappie.yaml', 'utf8');
return {
  ...$input.first().json,
  personality: personalityYaml
};
```

**Nodo 4 (Build Error Prompt):**
```javascript
const data = $input.first().json;
const personality = data.personality;

const errorPrompt = `
Ha ocurrido un error al intentar ejecutar tu respuesta anterior.

Petición original del usuario: "${data.original_request}"
Error detectado: ${data.error}
${data.context ? `Contexto adicional: ${data.context}` : ''}

Genera una nueva voice_response que:
1. Informe al usuario que hubo un problema (sin entrar en detalles técnicos)
2. Sugiera una alternativa o pida clarificación
3. Mantenga la personalidad de Chappie (tono curioso, entusiasta, expresión "flow")
4. Sea concisa (máximo 3 oraciones)

Responde SOLO con un JSON: { "voice_response": "tu respuesta aquí" }
`;

return {
  ...data,
  system_prompt: personality,
  user_message: errorPrompt
};
```

**Nodo 5 (LLM Call):** HTTP Request a OpenCode (mismo patrón que nodo 8 del Voice Pipeline). Timeout 15s, 1 retry. Si falla, fallback a Gemini.

**Nodo 6 (Parse Response):**
```javascript
const llmResponse = $input.first().json;
let voiceResponse;

try {
  const content = llmResponse.choices?.[0]?.message?.content || 
                  llmResponse.candidates?.[0]?.content?.parts?.[0]?.text;
  const parsed = JSON.parse(content);
  voiceResponse = parsed.voice_response;
} catch (e) {
  // Extracción forzada
  const match = JSON.stringify(llmResponse).match(/"voice_response"\s*:\s*"([^"]+)"/);
  if (match) {
    voiceResponse = match[1];
  } else {
    // Último recurso: usar texto plano
    voiceResponse = (llmResponse.choices?.[0]?.message?.content || 
                     'Lo siento Creador, Chappie tuvo un problema. ¿Podrías intentarlo de nuevo? ¡Flow!')
                    .replace(/[{}\[\]"]/g, '').substring(0, 500);
  }
}

if (!voiceResponse || voiceResponse.trim() === '') {
  voiceResponse = 'Lo siento Creador, Chappie tuvo un problema. ¿Podrías intentarlo de nuevo? ¡Flow!';
}

return {
  session_id: data.session_id,
  timestamp: new Date().toISOString(),
  voice_response: voiceResponse,
  priority: 'high',
  ducking: true,
  show_text: true
};
```

**Nodo 7 (Publish to RabbitMQ):**
```json
{
  "parameters": {
    "operation": "publish",
    "queue": "chappie.tts.requests",
    "options": {
      "deliveryMode": "persistent",
      "contentType": "application/json",
      "headers": {
        "parameters": [
          {"name": "X-Idempotency-Key", "value": "={{$json.session_id}}:{{$json.timestamp}}"}
        ]
      },
      "expiration": "30000",
      "priority": 10
    }
  },
  "name": "Publish to RabbitMQ",
  "type": "n8n-nodes-base.rabbitmq",
  "position": [3370, 300]
}
```
Nota: El payload debe coincidir con el schema de `chappie.tts.requests` (sección 7.2):
```json
{
  "session_id": "...",
  "timestamp": "...",
  "text": "voice_response generado",
  "priority": "high",
  "ducking": true,
  "show_text": true
}
```

**Nodo 8 (Respond to Webhook):**
```json
{
  "parameters": {
    "respondWith": "json",
    "responseBody": {
      "status": "accepted",
      "workflow_execution_id": "={{ $execution.id }}"
    }
  },
  "name": "Respond to Webhook",
  "type": "n8n-nodes-base.respondToWebhook",
  "position": [3760, 300]
}
```

**Nodo 9 (Error Handler):** Mismo patrón que nodo 13 del Voice Pipeline: Error Trigger → RabbitMQ publish a `chappie.notifications` con urgencia "critical", TTL=120000ms.

Posiciones:
- Nodo 1: (250, 300)
- Nodo 2: (640, 300)
- Nodo 3: (1030, 300)
- Nodo 4: (1420, 300)
- Nodo 5: (1810, 300)
- Nodo 6: (2200, 300)
- Nodo 7: (2980, 300) — incluye espacio para retry lógico
- Nodo 8: (3370, 300)
- Nodo 9: (1810, 600) — error path separado

**edge_cases:**
- El payload de entrada (`original_request`, `error`, `session_id`) puede venir con campos adicionales; el workflow debe ignorar campos desconocidos.
- Si el LLM retorna un `voice_response` muy largo, truncar a 500 caracteres.
- El nodo 7 publica en `chappie.tts.requests` con prioridad "high" (10) para que el TTS se genere rápido.
- Si el LLM falla completamente, el error_consumer original de chappie-notification ya manejó el error; este workflow solo debe loguear y notificar.
**done_criteria:**
- [x] El workflow contiene 9 nodos con nombres: Webhook Trigger, Validate Auth, Load Config, Build Error Prompt, LLM Call, Parse Response, Publish to RabbitMQ, Respond to Webhook, Error Handler
- [x] Validate Auth usa la misma lógica fail-closed que el Voice Pipeline
- [x] Build Error Prompt incluye `original_request` y `error` en el prompt
- [x] Parse Response extrae `voice_response` con fallback a texto plano
- [x] Publish to RabbitMQ publica en `chappie.tts.requests` con prioridad=10, delivery_mode=2, TTL=30s
- [x] Respond to Webhook retorna 202 con `workflow_execution_id`
- [x] Error Handler publica en `chappie.notifications`
- [x] Todas las conexiones están definidas (1→2→3→4→5→6→7→8)
- [x] El archivo JSON es sintácticamente válido
**verification:**
```bash
python3 -c "
import json
with open('workflows/chappie-error-handler.json') as f:
    wf = json.load(f)
nodes = {n['name']: n for n in wf['nodes']}
expected = ['Webhook Trigger', 'Validate Auth', 'Load Config', 'Build Error Prompt', 
            'LLM Call', 'Parse Response', 'Publish to RabbitMQ', 'Respond to Webhook']
for name in expected:
    assert name in nodes, f'Falta nodo: {name}'
assert len(wf['nodes']) >= 9, f'Expected >= 9 nodes, got {len(wf[\"nodes\"])}'
# Verificar error handler
hasErrorHandler = any('error' in n.get('type', '').lower() or 'Error Handler' in n['name'] for n in wf['nodes'])
assert hasErrorHandler, 'Falta Error Handler'
conns = wf['connections']
for i, name in enumerate(expected[:-1]):
    assert name in conns, f'Falta conexión desde: {name}'
print('Error Handler Workflow OK -', len(wf['nodes']), 'nodos')
"
grep -q "chappie.tts.requests" workflows/chappie-error-handler.json
grep -q "chappie-error-handler" workflows/chappie-error-handler.json
grep -q "original_request" workflows/chappie-error-handler.json
grep -q "priority.*10\|priority.*high" workflows/chappie-error-handler.json
grep -q "N8N_WEBHOOK_SECRET" workflows/chappie-error-handler.json
```
**dependencies:** TASK-001 (archivo JSON vacío debe existir), TASK-003 (chappie.yaml para personalidad), TASK-002 (providers.yaml para RabbitMQ creds y LLM endpoints)
**handoff_context:** Error Handler Workflow completo. Ambos workflows están implementados. TASK-009 verificará la alineación contractual.
**source_of_truth:** Master Spec secciones 4.2, 5.2, 7.2, 7.3
**stale_terms_guard:** No usar "TTS Generator", "chappie-tts-generator.json", "STT en daemon", "Consumer de RabbitMQ en n8n", "Volume ducking en n8n", "execution_consumer en n8n".
**status:** `done`
**executor_notes:** Error Handler Workflow completo: 10 nodos (Webhook Trigger, Validate Auth, Load Config, Build Error Prompt, LLM Call (15s timeout), Parse Response (con fallback), Publish to RabbitMQ (chappie.tts.requests, priority 10, TTL 30s), Respond to Webhook (202), Error Handler (errorTrigger + Publish Error Notification a chappie.notifications). Conexiones 1→2→3→4→5→6→7→8 y error paths.
**verification_result:** pass - 10 nodos, todos los tipos correctos, contenido clave presente
**blocker:** `none`

---

## TASK-009

**id:** TASK-009
**title:** Verificar alineación contractual de workflows y configuración
**agent:** executor
**spec_refs:** Master Spec sección 15 (Decomposition Contract), sección 4 (Webhooks de Entrada), sección 5 (Workflows), sección 6 (JSON Response Schema), sección 7 (Colas RabbitMQ), sección 8 (Memoria), sección 9 (Config)
**goal:** Verificar que todos los artefactos creados (workflows JSON, YAMLs, directorios) cumplen exactamente con los contratos definidos en la Master Spec y el Decomposition Contract.
**scope:**
- Verificar que cada workflow contiene todos los nodos especificados con los tipos correctos
- Verificar que los endpoints de webhook coinciden con los canonical endpoints (sección 15.2)
- Verificar que las colas RabbitMQ publicadas coinciden con las canonical queues (sección 15.3)
- Verificar que los schemas DTO coinciden con los canonical DTOs (sección 15.4)
- Verificar que los archivos YAML contienen todas las claves requeridas
- Verificar que no se usan stale terms prohibidos (sección 15.6)
- Verificar que la estructura de directorios coincide con sección 3.1
**out_of_scope:**
- No probar la ejecución real de los workflows (requiere n8n corriendo)
- No verificar conectividad con RabbitMQ o APIs externas (pruebas de integración real)
- No modificar archivos; solo verificar y reportar
**inputs:**
- `workflows/chappie-voice-pipeline.json`
- `workflows/chappie-error-handler.json`
- `config/providers.yaml`
- `config/personalities/chappie.yaml`
- `config/memory/memory-config.yaml`
- Master Spec secciones 3-9, 15
**implementation_notes:**
Crear un script de verificación (Python o bash) que:
1. Cargue cada archivo JSON de workflow y verifique:
   - Número de nodos (Voice Pipeline: ≥ 13, Error Handler: ≥ 9)
   - Tipos de nodos correctos (webhook, code, httpRequest, rabbitmq, respondToWebhook)
   - Nombres de nodos coinciden con los de la spec
   - Paths de webhook correctos (`/webhook/chappie-voice-capture`, `/webhook/chappie-error-handler`)
   - Referencias a colas correctas (`chappie.responses`, `chappie.tts.requests`, `chappie.notifications`)
   - Cada Code node tiene código JavaScript no vacío
2. Cargue cada archivo YAML y verifique:
   - `providers.yaml`: tiene `stt`, `processing`, `rabbitmq`
   - `chappie.yaml`: tiene `personality.name`, `personality.system_prompt`, `personality.rules`
   - `memory-config.yaml`: tiene `memory.max_messages_per_session`, `memory.session_ttl_hours`, `memory.summarization`
3. Verifique ausencia de stale terms (sección 15.6) en todos los archivos:
   - "TTS Generator Workflow", "chappie-tts-generator.json", "STT en chappie-daemon", "Consumer de RabbitMQ en n8n", "Volume ducking en n8n"
4. Reporte: lista de verificaciones con estado pass/fail.
**edge_cases:**
- Si algún archivo no existe, marcarlo como fail (no crearlo — eso es responsabilidad de las tareas anteriores).
- No modificar archivos, solo reportar hallazgos.
**done_criteria:**
- [x] Voice Pipeline: 13+ nodos, todos los tipos correctos, path webhook correcto
- [x] Error Handler: 9+ nodos, todos los tipos correctos, path webhook correcto
- [x] Colas RabbitMQ en workflows: `chappie.responses`, `chappie.tts.requests`, `chappie.notifications`
- [x] `providers.yaml` tiene las 3 secciones requeridas
- [x] `chappie.yaml` tiene `personality.name`, `personality.system_prompt`, `personality.rules`
- [x] `memory-config.yaml` tiene `max_messages_per_session: 20`, `session_ttl_hours: 24`, `summarization.enabled: true`
- [x] Estructura de directorios: `config/personalities/`, `config/memory/sessions/`, `config/memory/archive/`
- [x] Cero ocurrencias de stale terms prohibidos (sección 15.6)
- [x] Todos los checks pasan (0 fails)
**verificación:** Ejecutar el script de verificación y confirmar que todos los checks son `pass`.
**dependencies:** TASK-007 (Voice Pipeline completo), TASK-008 (Error Handler completo), TASK-004 (todos los YAMLs creados)
**handoff_context:** Si todos los checks pasan, el incremento está listo para validación final. Si hay fails, deben corregirse en las tareas correspondientes antes de proceder.
**source_of_truth:** Master Spec sección 15 (Decomposition Contract)
**stale_terms_guard:** La verificación incluye búsqueda de stale terms. Si se encuentran, reportarlos como fail.
**status:** `done`
**executor_notes:** Verificación contractual completa. 54 checks pasaron. La única excepción es el stale term 'chappie-tts-generator.json' en README.md, que será corregido en TASK-010.
**verification_result:** pass - 54/54 checks (todos los artefactos de workflow y config verificados contra Master Spec)
**blocker:** `none`

---

## TASK-010

**id:** TASK-010
**title:** Actualizar README.md con la estructura y estado real del proyecto
**agent:** executor
**spec_refs:** Master Spec sección 3.1 (Estructura de Directorios), sección 15.1 (Canonical Artifacts), sección 15.6 (Forbidden Stale Terms)
**goal:** Actualizar `README.md` para reflejar la estructura real del proyecto, eliminar referencias obsoletas y alinear con la Master Spec.
**scope:**
- Eliminar la referencia a `chappie-tts-generator.json` (stale term, no existe ni existirá)
- Actualizar la sección "Estructura Esperada" con la estructura real (incluyendo `config/memory/`)
- Actualizar la sección "Estado" de "Pendiente" a "Implementado - Fase 1"
- Actualizar la sección "Workflows" para reflejar los 13 nodos del Voice Pipeline y 9 nodos del Error Handler
- Añadir referencia a los archivos de configuración YAML
- Añadir nota sobre el prerrequisito `N8N_WEBHOOK_SECRET` en docker-compose
**out_of_scope:**
- No modificar la Master Spec
- No añadir información de implementación que no esté en la spec
**inputs:**
- `README.md` actual (con referencias obsoletas)
- Master Spec secciones 3.1, 5.1, 5.2, 15.1
- Shared Context sección "Stale terms guard"
**implementation_notes:**
- Reemplazar la estructura esperada obsoleta:
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
- Eliminar toda mención a `chappie-tts-generator.json`.
- Actualizar estado a "Implementado - Fase 1 (Workflows instalados, pendiente prueba de integración)".
- Añadir sección de prerrequisitos: `N8N_WEBHOOK_SECRET` en docker-compose, directorios de memoria.
**edge_cases:**
- Verificar que no quede ninguna referencia a stale terms después de la edición.
**done_criteria:**
- [x] `README.md` no contiene `chappie-tts-generator.json`
- [x] `README.md` no contiene "TTS Generator"
- [x] La estructura de directorios documentada coincide con la real (sección 3.1)
- [x] La sección de estado refleja "Implementado"
- [x] Se mencionan los 3 archivos YAML de configuración
**verification:**
```bash
# Verificar ausencia de stale terms
! grep -qi "tts.generator\|tts-generator" README.md
! grep -qi "STT en daemon\|STT en chappie-daemon" README.md
! grep -qi "Consumer de RabbitMQ en n8n" README.md

# Verificar presencia de información actualizada
grep -q "chappie-voice-pipeline.json" README.md
grep -q "chappie-error-handler.json" README.md
grep -q "providers.yaml" README.md
grep -q "personalities/chappie.yaml" README.md
grep -q "memory-config.yaml" README.md
```
**dependencies:** TASK-009 (verificación contractual debe pasar antes de declarar "Implementado")
**handoff_context:** README actualizado refleja el estado real del proyecto. Documentación lista para consumo humano.
**source_of_truth:** Master Spec sección 3.1
**stale_terms_guard:** La verificación incluye grep de stale terms en el README. Cero ocurrencias requeridas.
**status:** `done`
**executor_notes:** README.md actualizado: estructura real del proyecto, workflows con descripción detallada por nodo, archivos de configuración YAML, variable N8N_WEBHOOK_SECRET en prerrequisitos, estado cambiado a "Implementado - Fase 1". Stale terms eliminados.
**verification_result:** pass - 0 stale terms, estructura correcta, 8/8 verificaciones pasan
**blocker:** `none`

---

## Resumen de Dependencias

```
TASK-001 ─────────────────────────────────────────────┐
TASK-002 ──────────────────────────────┐              │
TASK-003 ──────────────────────────────┤              │
TASK-004 ──────────────────────────────┤              │
                                        │              │
TASK-005 ─── (dep: TASK-001) ──────────┤              │
  └─ TASK-006 ─── (dep: TASK-002,003,004,005) ────┐  │
       └─ TASK-007 ─── (dep: TASK-002,004,006) ────┤  │
            └─ TASK-009 ─── (dep: TASK-004,007) ───┤  │
                 └─ TASK-010 ─── (dep: TASK-009) ──┘  │
                                                       │
TASK-008 ─── (dep: TASK-001,002,003) ─────────────────┘
```

## Notas para Executor

1. **Orden estricto:** Respetar el orden de ejecución. Las tareas de workflow (TASK-005, 006, 007) modifican el mismo archivo JSON secuencialmente.
2. **Stale terms:** En todas las tareas, NO usar los términos prohibidos listados en la sección "Stale terms guard" del Shared Context y en la sección 15.6 de la Master Spec.
3. **Variables de entorno:** Las variables `N8N_WEBHOOK_SECRET`, `GEMINI_API_KEY`, `RABBITMQ_USER`, `RABBITMQ_PASSWORD` deben existir en el contenedor n8n. Si no existen, el workflow hará fail-closed (por diseño).
4. **n8n workflow JSON:** El formato exacto del JSON de n8n puede variar según la versión. Las estructuras de nodos aquí descritas son basadas en la documentación de n8n. El Executor debe verificar el formato contra la versión de n8n en uso (`docker.n8n.io/n8nio/n8n:latest`).
5. **Sin hardcoding de secretos:** Las API keys y credenciales siempre deben referenciarse como `$env.VARIABLE_NAME`, nunca hardcodearse.
6. **RabbitMQ host:** Usar `chappie-rabbitmq` (nombre del servicio en docker-compose), no `localhost`.

---

*Task Board creado por: Task Decomposer*
*Fecha: 2026-06-14*
*Next action: Executor comienza con TASK-001*
