# Shared Context - initial-setup (chappie-n8n-workflows)

**Incremento:** initial-setup  
**Proyecto:** `projects/chappie-n8n-workflows/`  
**Lifecycle status:** `validated-not-executed`  
**Creado:** 2026-06-14  
**Ultima actualizacion:** 2026-06-14 (post-remediacion F-001, F-005, F-006, F-007)  

---

## Current status

**Estado del incremento:** `validated-not-executed`

La Master Spec del proyecto `chappie-n8n-workflows` fue revisada por Spec Validator (segunda ronda). **Todos los 7 findings fueron resueltos correctamente:**

- **Findings globales (F-002, F-003, F-004):** Corregidos por Enterprise Architect en `integration-map.md` y `system-landscape.md`.
- **Findings locales (F-001, F-005, F-006, F-007):** Corregidos por Planner en `master_spec.md` local.

**Spec Validator Approval:** `verdict: ready` otorgado. Master Spec completa, consistente y lista para descomposición.

**Transición requerida:** Aprobación humana del plan antes de avanzar a Task Decomposer.

---

## Canonical artifacts

| Artifact | Ruta absoluta | Estado |
|---|---|---|
| Master Spec (local) | `/home/cristiansrc/Documentos/Proyectos/chappie-workspace/projects/chappie-n8n-workflows/docs/specs/master_spec.md` | `draft` (post-remediacion) |
| Shared Context | `/home/cristiansrc/Documentos/Proyectos/chappie-workspace/projects/chappie-n8n-workflows/docs/specs/.working/initial-setup-sdd-context.md` | `validator-review` |
| Master Spec (global workspace) | `/home/cristiansrc/Documentos/Proyectos/chappie-workspace/docs/specs/master_spec.md` | Active |
| System Landscape | `/home/cristiansrc/Documentos/Proyectos/chappie-workspace/docs/architecture/system-landscape.md` | Active |
| Integration Map | `/home/cristiansrc/Documentos/Proyectos/chappie-workspace/docs/architecture/integration-map.md` | Active |
| Context Map | `/home/cristiansrc/Documentos/Proyectos/chappie-workspace/docs/architecture/context-map.md` | Active |
| Workspace Changes | `/home/cristiansrc/Documentos/Proyectos/chappie-workspace/docs/specs/workspace_changes.md` | Active |
| Docker Compose | `/home/cristiansrc/Documentos/Proyectos/chappie-workspace/docker-compose.yaml` | Active |
| ADR-001 (n8n orchestration) | `/home/cristiansrc/Documentos/Proyectos/chappie-workspace/docs/architecture/decision-records/ADR-001-n8n-orchestration.md` | Accepted |
| ADR-002 (RabbitMQ events) | `/home/cristiansrc/Documentos/Proyectos/chappie-workspace/docs/architecture/decision-records/ADR-002-rabbitmq-events.md` | Accepted |

---

## Artifact evidence

| Artifact | Verificado | Resultado |
|---|---|---|
| Master Spec (local) | Si (remediada en esta sesion) | `pass` - 4 findings corregidos (F-001, F-005, F-006, F-007). Lifecycle actualizado a `draft`. |
| Shared Context | Si (creado en esta sesion) | `pass` - Archivo creado en ruta esperada con estado `planning`. |
| Master Spec (global) | Si (leido en esta sesion) | `pass` - 220 lineas. Define bounded contexts Orchestration y Memory para chappie-n8n-workflows. |
| System Landscape | Si (leido en esta sesion) | `pass` - 289 lineas. Define flujos de voz, error y notificacion interactiva. |
| Integration Map | Si (leido en esta sesion) | `pass` - 537 lineas. Define webhooks, colas RabbitMQ y schemas JSON. |
| Context Map | Si (leido en esta sesion) | `pass` - 197 lineas. Define relaciones Customer-Supplier entre contextos. |
| Workspace Changes | Si (leido en esta sesion) | `pass` - 72 lineas. Sin cambios globales pendientes que afecten este proyecto. |
| Docker Compose | Si (leido en esta sesion) | `pass` - 102 lineas. Bind mounts para workflows y config ya definidos. |
| ADR-001 | Si (leido en esta sesion) | `pass` - Decision aceptada: n8n como orquestador central. |
| ADR-002 | Si (leido en esta sesion) | `pass` - Decision aceptada: RabbitMQ como bus de eventos. |
| Workflows dir | Si (listado en esta sesion) | `pass` - Directorio existe con `.gitkeep`. Listo para archivos JSON. |
| Config dir | Si (listado en esta sesion) | `pass` - Directorio existe con `.gitkeep`. Listo para archivos YAML. |

---

## Spec Validator Approval

verdict: ready
reviewed_at: 2026-06-14T12:00:00Z
validator_agent: spec-validator
artifact_set_reviewed:
  - /home/cristiansrc/Documentos/Proyectos/chappie-workspace/projects/chappie-n8n-workflows/docs/specs/master_spec.md
  - /home/cristiansrc/Documentos/Proyectos/chappie-workspace/projects/chappie-n8n-workflows/docs/specs/.working/initial-setup-sdd-context.md
summary: Todos los 7 findings resueltos correctamente. F-001 (blocker): mecanismo de auth detallado con nodo Code y fail-closed. F-002, F-003, F-004 (globales): corregidos por Enterprise Architect en integration-map.md y system-landscape.md. F-005: healthcheck /healthz verificado como endpoint nativo de n8n. F-006: contrato de idempotencia en dos capas definido (producer headers + consumer dedup con TTL). F-007: capacidades del nodo RabbitMQ verificadas (delivery_mode, headers, priority, TTL). Master Spec completa, consistente y lista para descomposición tras aprobación humana.
invalidated_by_changes_since: none

---

## Decisions locked

| ID | Decision | Fuente | Razon |
|---|---|---|---|
| D-001 | n8n como orquestador central (no Python daemon, no Go custom) | ADR-001 | Decision arquitectonica aceptada. |
| D-002 | RabbitMQ como bus de eventos (no Redis Pub/Sub, no Kafka) | ADR-002 | Decision arquitectonica aceptada. |
| D-003 | 2 workflows: Voice Pipeline + Error Handler | Master Spec global seccion 5.1, 5.2 | Flujos definidos en system-landscape.md. |
| D-004 | STT ejecutado por n8n via Gemini API (no por chappie-daemon) | Master Spec global seccion 6 | chappie-daemon "No ejecuta STT ni orquestacion (delega a n8n)". |
| D-005 | n8n solo publica en RabbitMQ, no consume colas | Master Spec global seccion 4.2 | Los consumers son todos de chappie-notification. |
| D-006 | Bind mounts para despliegue de workflows y config | Docker Compose | Ya configurado en docker-compose.yaml del workspace. |
| D-007 | Auth de webhooks via header `X-Webhook-Secret` | Integration Map seccion 1.1, 1.2 | Contrato global ya definido. |
| D-008 | JSON Response Schema con campos: voice_response, agent_call, terminal_command, notification, memory_update | Integration Map seccion 2.1 | Contrato global ya definido. |
| D-009 | Memoria conversacional en archivos JSON por session_id | Master Spec local | Sin base de datos; n8n usa filesystem via bind mount. |
| D-010 | Error Handler publica directamente en chappie.tts.requests (no en chappie.responses) | Master Spec global seccion 5.2 | Flujo de error definido: evita re-procesamiento por execution_consumer. |
| D-011 | Auth de webhooks via nodo Code (no auth nativa de n8n) | F-001 remediation | n8n no tiene auth nativa de webhooks por header. Se implementa nodo Code que compara `X-Webhook-Secret` vs `$env.N8N_WEBHOOK_SECRET`. Fail-closed si variable no existe. |
| D-012 | Idempotencia en dos capas: producer (header AMQP) + consumer (dedup con cache TTL) | F-006 remediation | n8n no tiene estado compartido. Consumer (execution_consumer) es responsable de deduplicar. Cache TTL=120s recomendado. |
| D-013 | Healthcheck `/healthz` es endpoint nativo de n8n (verificado) | F-005 remediation | Documentacion oficial confirma `/healthz` retorna 200 si instancia es alcanzable. No requiere config adicional. |
| D-014 | Nodo RabbitMQ de n8n soporta delivery_mode=2, custom headers, priority (verificado) | F-007 remediation | Documentacion oficial confirma todas las capacidades AMQP requeridas. |

---

## Validator findings

### F-001 [BLOCKER] → RESUELTO

**Clasificacion:** `contract-drift`  
**Accion:** Spec corregida + prerequisito de infra documentado.  
**Cambio aplicado:**
- Seccion 10 reescrita: mecanismo de auth detallado (nodo Code en cada workflow que compara header vs `$env.N8N_WEBHOOK_SECRET`).
- Fail-closed definido: si la variable no existe, retorna 500.
- Prerequisito documentado: `N8N_WEBHOOK_SECRET` debe anadirse a `docker-compose.yaml` (responsabilidad de chappie-infrastructure o workspace root).
- Nodos de workflows actualizados: nuevo nodo 2 "Validate Auth" en ambos workflows.

### F-002 [HIGH] → RESUELTO (por Enterprise Architect)

Corregido en `integration-map.md` global. Fuera del alcance de este archivo.

### F-003 [HIGH] → RESUELTO (por Enterprise Architect)

Corregido en `integration-map.md` global. Fuera del alcance de este archivo.

### F-004 [MEDIUM] → RESUELTO (por Enterprise Architect)

Corregido en `system-landscape.md` global. Fuera del alcance de este archivo.

### F-005 [MEDIUM] → RESUELTO

**Clasificacion:** `validator/process-bug` (el healthcheck si es valido)  
**Accion:** Verificacion tecnica completada.  
**Evidencia:** Documentacion oficial de n8n ([docs.n8n.io/hosting/logging-monitoring/monitoring](https://docs.n8n.io/hosting/logging-monitoring/monitoring/)) confirma que `/healthz` es un endpoint nativo que retorna HTTP 200 si la instancia es alcanzable. No requiere variable de entorno adicional. OQ-003 resuelta.  
**Cambio aplicado:** Seccion 11 actualizada con referencia a fuente y nota sobre `/healthz/readiness`.

### F-006 [MEDIUM] → RESUELTO

**Clasificacion:** `design-decision`  
**Accion:** Contrato de idempotencia definido explicitamente.  
**Cambio aplicado:**
- Seccion 5.1.4 reescrita: estrategia de dos capas (n8n publica con header `X-Idempotency-Key`, consumer deduplica con cache TTL).
- Contrato para chappie-notification documentado: consumer DEBE implementar deduplicacion basada en `session_id` + `timestamp` con cache TTL=120s.
- Seccion 13.1 actualizada con detalles de idempotencia en ambas capas.
- Racional documentado: n8n no tiene estado compartido; el consumer es el lugar natural para deduplicacion.

### F-007 [LOW] → RESUELTO

**Clasificacion:** `validator/process-bug` (las capacidades si estan soportadas)  
**Accion:** Verificacion tecnica completada.  
**Evidencia:** Documentacion oficial de n8n ([RabbitMQ node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.rabbitmq/)) confirma que el nodo nativo `n8n-nodes-base.rabbitmq` soporta: delivery_mode (persistent/transient), custom headers, routing key, priority, TTL por mensaje, content-type. OQ-002 resuelta.  
**Cambio aplicado:**
- Nueva seccion 7.0: capacidades verificadas del nodo RabbitMQ de n8n.
- Secciones 7.1, 7.2, 7.3 actualizadas con detalles de configuracion AMQP (delivery_mode, headers, TTL, priority, content-type).

---

## Resolved findings

| ID | Severidad | Resumen | Resuelto por | Fecha |
|---|---|---|---|---|
| F-001 | blocker | `N8N_WEBHOOK_SECRET` no existe en docker-compose.yaml | Planner (spec + prerequisito) | 2026-06-14 |
| F-002 | high | Integration Map 1.4 asigna ejecucion de agentes a n8n | Enterprise Architect (integration-map.md) | 2026-06-14 |
| F-003 | high | Integration Map 2.3 omite a n8n como producer | Enterprise Architect (integration-map.md) | 2026-06-14 |
| F-004 | medium | System Landscape C4 lista "TTS Generator" inexistente | Enterprise Architect (system-landscape.md) | 2026-06-14 |
| F-005 | medium | Healthcheck `/healthz` no verificado | Planner (verificacion tecnica: endpoint nativo confirmado) | 2026-06-14 |
| F-006 | medium | Idempotencia sin contrato coordinado | Planner (contrato de dos capas definido) | 2026-06-14 |
| F-007 | low | Nodo RabbitMQ capacidades no verificadas | Planner (verificacion tecnica: capacidades confirmadas) | 2026-06-14 |

---

## Open questions

| ID | Pregunta | Impacto | Estado |
|---|---|---|---|
| OQ-001 | El archivo `providers.yaml` de n8n debe ser una copia del `providers.yaml` de chappie-config o un archivo independiente? Si es independiente, como se sincronizan? | Configuracion de proveedores | Abierta - Requiere decision de arquitectura con chappie-config. |
| OQ-002 | ~~n8n tiene un nodo nativo de RabbitMQ? Se confirma que el nodo "RabbitMQ" de n8n soporta publicacion con delivery_mode=2 (persistent) y custom headers?~~ | Publicacion en colas | **RESUELTA** (F-007): Si, nodo nativo `n8n-nodes-base.rabbitmq` soporta delivery_mode, custom headers, routing key, priority, TTL. Fuente: docs oficiales n8n. |
| OQ-003 | ~~El healthcheck de n8n usa `/healthz` pero n8n por defecto no expone ese endpoint. Se requiere configuracion adicional o un proxy?~~ | Observabilidad | **RESUELTA** (F-005): `/healthz` es endpoint nativo de n8n. Retorna 200 si instancia es alcanzable. No requiere config adicional. Fuente: docs oficiales n8n. |
| OQ-004 | Como maneja n8n la concurrencia de ejecuciones para el mismo session_id? Se requiere configurar "Execution Order" en el workflow? | Idempotencia | **RESUELTA** (F-006): n8n ejecuta webhooks en paralelo sin locking. Deduplicacion delegada al consumer (execution_consumer) con cache TTL. Ver seccion 5.1.4 de Master Spec. |

---

## Stale terms guard

Los siguientes terminos estan PROHIBIDOS en este proyecto porque no corresponden a la responsabilidad de chappie-n8n-workflows:

| Termino prohibido | Razon | Termino correcto |
|---|---|---|
| "TTS Generator Workflow" | n8n no genera TTS; lo hace chappie-notification | "Voice Pipeline Workflow" o "Error Handler Workflow" |
| "STT en daemon" | chappie-daemon no ejecuta STT | "STT en n8n via Gemini API" |
| "Consumer de RabbitMQ en n8n" | n8n solo publica, no consume | "Publisher en n8n" / "Consumer en chappie-notification" |
| "Volume ducking en n8n" | El ducking lo hace chappie-daemon | "Volume ducking en chappie-daemon" |
| "chappie-tts-generator.json" | Este workflow NO existe en n8n | No usar; el TTS es responsabilidad de chappie-notification |
| "execution_consumer en n8n" | execution_consumer es de chappie-notification | "Voice Pipeline Workflow publica en chappie.responses" |

---

## Next action

**Acción requerida:** Executor comienza implementación

**Estado:** Task Board creado por Task Decomposer. 10 tareas atómicas definidas en `docs/specs/tasks/initial-setup-task-board.md`.

**Task Board:** `/home/cristiansrc/Documentos/Proyectos/chappie-workspace/projects/chappie-n8n-workflows/docs/specs/tasks/initial-setup-task-board.md`

**Transición permitida:** Executor puede comenzar con TASK-001 (Crear estructura de directorios y archivos JSON vacíos).

---

## Human Plan Approval: approved_by_user
