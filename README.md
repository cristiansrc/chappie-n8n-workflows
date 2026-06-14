# chappie-n8n-workflows

**Workflows de n8n para orquestación del asistente Chappie**

---

## Responsabilidad

- Definición de workflows de n8n para el pipeline de voz
- STT (Speech-to-Text) con Gemini
- Procesamiento con modelos de IA (multi-proveedor)
- Generación de JSON estructurado con personalidad de Chappie
- Manejo de errores
- Gestión de memoria conversacional

## Estado

**Pendiente** - Será implementado en Fase 1

## Estructura Esperada

```
chappie-n8n-workflows/
├── workflows/
│   ├── chappie-voice-pipeline.json    # Workflow principal
│   ├── chappie-error-handler.json     # Manejo de errores
│   └── chappie-tts-generator.json     # Generación de TTS
├── config/
│   ├── providers.yaml                 # APIs de modelos
│   └── personalities/
│       └── chappie.yaml               # Personalidad de Chappie
└── README.md
```

## Workflows

### chappie-voice-pipeline
1. Recibe audio via webhook
2. STT con Gemini 2.5 Flash
3. Lee configuración (providers, personality, memory)
4. Llama al modelo de procesamiento
5. Publica resultado en RabbitMQ (chappie.responses)
6. Actualiza memoria

### chappie-error-handler
1. Recibe error de execution_consumer
2. Llama al modelo con el contexto del error
3. Genera nueva respuesta con pregunta
4. Publica en chappie.tts.requests

## Integraciones

- **chappie-daemon:** Recibe audio via webhook
- **RabbitMQ:** Publica en colas de eventos
- **Gemini API:** STT y procesamiento
- **OpenCode API:** Modelos free
- **OpenCode CLI:** Ejecución de agentes

---

*Proyecto parte del workspace chappie-workspace*
