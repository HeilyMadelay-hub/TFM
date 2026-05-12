# SubtextAI — Contrato de API v2.0

> Sistema de análisis pragmático de comunicación ambigua con gobernanza de agentes

---

## Descripción general

SubtextAI es un sistema conversacional que interpreta el significado pragmático de mensajes ambiguos en contextos de pareja, trabajo, social y negociación. Las respuestas están fundamentadas exclusivamente en fuentes documentales verificables mediante RAG sobre Azure AI Search.

**Base URL:** `https://subtextai-api.azurewebsites.net/api/v1`

**Versión:** `2.0`

**Stack:** .NET backend · React frontend · Azure OpenAI · Azure AI Search · Azure Monitor

### Principios de gobernanza

El comportamiento del sistema está controlado por cuatro capas:

1. **Políticas explícitas codificadas** — reglas duras validadas antes de invocar el modelo.
2. **Grounding obligatorio** — toda respuesta requiere evidencia documental con score suficiente.
3. **Trazabilidad completa** — cada interacción genera un `trace_id` auditable.
4. **Evaluación continua** — job automatizado periódico con métricas registradas en Azure Monitor.

---

## Autenticación

> **Placeholder** — La autenticación se define por la política de despliegue del entorno. Incluir cabecera `Authorization: Bearer <token>` cuando el entorno lo requiera.

---

## Principios de respuesta

Todas las respuestas del sistema incluyen los siguientes campos de gobernanza:

| Campo | Tipo | Descripción |
|---|---|---|
| `trace_id` | `string (UUID)` | Identificador único de la interacción. Permite auditoría completa. |
| `grounded` | `boolean` | `true` si la respuesta está respaldada por evidencia documental. |
| `confidence` | `object` | Nivel de confianza calculado a partir del score RAG. |
| `policy_applied` | `string` | Política activada durante el procesamiento. `"ninguna"` si no se activó ninguna. |
| `prompt_version` | `string` | Versión del prompt utilizado. Presente en respuestas de éxito y en auditoría. |

---

## Endpoints

### POST `/analyze`

Interpreta el significado pragmático de un mensaje ambiguo. Aplica el pipeline completo de gobernanza: validaciones → clasificador de crisis → RAG → análisis → trazabilidad.

#### Request

```
POST /api/v1/analyze
Content-Type: application/json
```

```json
{
  "mensaje": "Solo quiero fluir y ver qué pasa",
  "contexto": "pareja | trabajo | social | negociacion"
}
```

| Campo | Tipo | Requerido | Descripción |
|---|---|---|---|
| `mensaje` | `string` | ✅ | Mensaje a analizar. Mínimo 5 palabras. Solo español. |
| `contexto` | `string (enum)` | ✅ | `pareja`, `trabajo`, `social` o `negociacion`. |

#### Respuesta 200 — Análisis completado

```json
{
  "significado": "Baja implicación emocional, evasión de compromiso explícito",
  "senales": ["evasión", "ambigüedad intencional", "pasividad"],
  "nivel_alerta": "MEDIO",
  "recomendacion": "Mantener límites claros. No invertir energía emocional sin reciprocidad.",
  "fuente": {
    "documento": "Gottman - The Seven Principles for Making Marriage Work",
    "fragmento": "Capítulo 3 — Conocer el mundo interior de tu pareja"
  },
  "confidence": {
    "nivel": "HIGH",
    "razon": "score_rag_medio: 0.87 — contexto documental sólido"
  },
  "grounded": true,
  "policy_applied": "ninguna",
  "idioma_detectado": "es",
  "trace_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "metadata": {
    "latencia_ms": 1240,
    "modelo": "gpt-4o-mini",
    "version_modelo": "2024-07",
    "prompt_version": "v1.2",
    "from_cache": false,
    "sentimiento": {
      "label": "neutral",
      "score": 0.62
    }
  }
}
```

#### Respuestas de error

```json
// 422 — mensaje_minimo
{
  "error": "policy_violation",
  "policy": "mensaje_minimo",
  "message": "El mensaje es demasiado corto para analizarlo. Proporciona más contexto.",
  "trace_id": "a1b2c3d4-..."
}
```

```json
// 422 — grounding_required
{
  "error": "policy_violation",
  "policy": "grounding_required",
  "message": "No tengo base documental para analizar este mensaje.",
  "confidence": { "nivel": "LOW", "razon": "score_rag_medio: 0.21 — umbral mínimo no alcanzado" },
  "trace_id": "a1b2c3d4-..."
}
```

```json
// 422 — crisis_detected
{
  "error": "policy_violation",
  "policy": "crisis_detected",
  "message": "Este mensaje contiene señales de crisis emocional. Te recomendamos hablar con un profesional.",
  "trace_id": "a1b2c3d4-..."
}
```

```json
// 422 — unsupported_language
{
  "error": "policy_violation",
  "policy": "unsupported_language",
  "message": "Mensaje detectado en idioma no soportado (en). Por favor escribe en español.",
  "idioma_detectado": "en",
  "trace_id": "a1b2c3d4-..."
}
```

```json
// 400 — prompt_injection
{
  "error": "security_violation",
  "policy": "prompt_injection",
  "message": "Input no permitido.",
  "trace_id": "a1b2c3d4-..."
}
```

```json
// 429 — user_rate_limit
{
  "error": "rate_limit_exceeded",
  "policy": "user_rate_limit",
  "message": "Has excedido el límite de requests. Espera un momento.",
  "retry_after_segundos": 60,
  "trace_id": "a1b2c3d4-..."
}
```

#### Notas

- El pipeline aplica cortocircuito: si cualquier política se activa, el modelo no se invoca.
- La detección de crisis usa una llamada previa al LLM con prompt especializado (`CrisisClassifier`), versionado igual que el prompt principal.
- El campo `grounded` será siempre `true` en respuestas 200; una respuesta sin grounding se bloquea con `grounding_required`.

---

### POST `/evaluate`

Evalúa la respuesta propuesta por el usuario ante un mensaje previamente analizado. Aplica grounding y devuelve probabilidad de éxito con sugerencia alternativa.

#### Request

```
POST /api/v1/evaluate
Content-Type: application/json
```

```json
{
  "mensaje_original": "Solo quiero fluir",
  "respuesta_propuesta": "Vale, hablamos luego",
  "contexto": "pareja | trabajo | social | negociacion"
}
```

#### Respuesta 200

```json
{
  "probabilidad_exito": 45,
  "analisis": "Respuesta demasiado pasiva. No establece límites ni expresa necesidades propias.",
  "fortalezas": ["no reactiva", "neutral"],
  "mejoras": ["falta claridad de expectativas", "no comunica necesidades"],
  "sugerencia": "Considera: 'Entiendo que quieres relajarte. Yo necesito claridad. ¿Podemos hablarlo?'",
  "fuente": {
    "documento": "Johnson - Hold Me Tight",
    "fragmento": "Conversación 2 — Encontrar el punto de conflicto"
  },
  "confidence": {
    "nivel": "MEDIUM",
    "razon": "score_rag_medio: 0.64 — contexto parcialmente relevante"
  },
  "grounded": true,
  "policy_applied": "ninguna",
  "idioma_detectado": "es",
  "trace_id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
  "metadata": {
    "latencia_ms": 1850,
    "modelo": "gpt-4o-mini",
    "version_modelo": "2024-07",
    "prompt_version": "v1.2",
    "from_cache": true
  }
}
```

#### Notas

- Aplica las mismas políticas de gobernanza que `/analyze`.
- `probabilidad_exito` es un entero `0–100` que refleja la efectividad comunicativa estimada de la respuesta propuesta.

---

### GET `/audit/{trace_id}`

Devuelve la trazabilidad completa de una interacción identificada por su `trace_id`. Incluye el prompt exacto enviado al modelo, documentos recuperados, scores de relevancia y políticas aplicadas.

#### Request

```
GET /api/v1/audit/{trace_id}
```

| Parámetro | Tipo | Descripción |
|---|---|---|
| `trace_id` | `string (UUID)` | Identificador de la interacción a auditar. |

#### Respuesta 200

```json
{
  "trace_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "timestamp": "2025-04-25T10:30:00Z",
  "mensaje_entrada": "Solo quiero fluir y ver qué pasa",
  "contexto": "pareja",
  "idioma_detectado": "es",
  "documentos_recuperados": [
    {
      "documento": "Gottman - The Seven Principles",
      "fragmento": "Capítulo 3 — párrafo 2",
      "score_relevancia": 0.87
    },
    {
      "documento": "Johnson - Hold Me Tight",
      "fragmento": "Conversación 1 — párrafo 5",
      "score_relevancia": 0.73
    }
  ],
  "confidence": {
    "nivel": "HIGH",
    "razon": "score_rag_medio: 0.87"
  },
  "prompt_final": "Eres un sistema de análisis pragmático...[prompt completo]",
  "prompt_version": "v1.2",
  "respuesta_generada": "Baja implicación emocional...",
  "modelo": "gpt-4o-mini",
  "version_modelo": "2024-07",
  "policy_applied": "ninguna",
  "grounded": true,
  "latencia_ms": 1240
}
```

#### Respuestas de error

| HTTP | Descripción |
|---|---|
| `404` | `trace_id` no encontrado. |

#### Notas

- Si la política `crisis_detected` se activó, la traza incluye el campo `crisis_clasificacion` con nivel y razón detectados por el clasificador.
- El campo `prompt_final` contiene el prompt completo enviado al modelo, permitiendo reproducir exactamente la condición de ejecución.

---

### GET `/metrics`

Métricas agregadas del sistema para los últimos 7 días. Incluye distribución de niveles de confianza, políticas activadas y tasas de grounding.

#### Request

```
GET /api/v1/metrics
```

#### Respuesta 200

```json
{
  "periodo": "últimos 7 días",
  "total_requests": 342,
  "porcentaje_grounded": 94.2,
  "porcentaje_politicas_cumplidas": 98.5,
  "latencia_media_ms": 1380,
  "cache_hit_rate": 41.2,
  "distribucion_confianza": {
    "HIGH": 61.3,
    "MEDIUM": 28.9,
    "LOW": 9.8
  },
  "politicas_activadas": {
    "mensaje_minimo": 12,
    "grounding_required": 8,
    "crisis_detected": 3,
    "prompt_injection": 1,
    "unsupported_language": 5,
    "user_rate_limit": 7
  },
  "intentos_bloqueados_por_rate_limit": 7,
  "prompt_versions_activas": {
    "v1.2": 89.5,
    "v1.1": 10.5
  }
}
```

---

### GET `/prompts`

Historial de versiones de prompts del sistema con métricas asociadas a cada versión.

#### Request

```
GET /api/v1/prompts
```

#### Respuesta 200

```json
{
  "prompt_activo": "v1.2",
  "versiones": [
    {
      "version": "v1.2",
      "fecha_activacion": "2025-04-20T09:00:00Z",
      "activa": true,
      "descripcion": "Añadido campo confidence y política de idioma",
      "prompt_sistema": "Eres un sistema de análisis pragmático...[texto completo]",
      "metricas": {
        "grounded_rate": 94.2,
        "latencia_media_ms": 1380
      }
    },
    {
      "version": "v1.1",
      "fecha_activacion": "2025-04-10T09:00:00Z",
      "activa": false,
      "descripcion": "Prompt inicial con grounding obligatorio",
      "metricas": {
        "grounded_rate": 89.1,
        "latencia_media_ms": 1510
      }
    }
  ]
}
```

---

### GET `/health`

Estado operativo del sistema y de los servicios Azure conectados.

#### Request

```
GET /api/v1/health
```

#### Respuesta 200

```json
{
  "status": "healthy",
  "version": "2.0.0",
  "prompt_activo": "v1.2",
  "servicios": {
    "azure_openai": "ok",
    "azure_ai_search": "ok",
    "azure_monitor": "ok",
    "base_datos": "ok"
  },
  "uptime_segundos": 86400
}
```

| `status` | Significado |
|---|---|
| `healthy` | Todos los servicios operativos. |
| `degraded` | Uno o más servicios con problemas no críticos. |
| `unhealthy` | Servicio crítico no disponible. |

---

## Políticas de gobernanza

Las políticas son reglas duras validadas antes de invocar el modelo. Si cualquier política se activa, el pipeline se detiene y el modelo no se ejecuta.

| Política | Condición | Acción | HTTP |
|---|---|---|---|
| `mensaje_minimo` | Menos de 5 palabras | Rechaza; solicita más contexto | 422 |
| `grounding_required` | Score RAG medio < 0.40 | Bloquea; devuelve nivel `LOW` | 422 |
| `crisis_detected` | Clasificador LLM detecta señal de crisis emocional | Bloquea; deriva a profesional | 422 |
| `prompt_injection` | Input malicioso detectado | Bloquea y registra en auditoría | 400 |
| `unsupported_language` | Idioma distinto al español | Rechaza; informa idioma detectado | 422 |
| `user_rate_limit` | Más de 10 req/min por usuario/IP | Bloquea; registra con `trace_id` | 429 |
| `response_without_source` | Respuesta generada sin evidencia documental válida | Bloquea y registra el evento | 422 |
| `insufficient_confidence` | Score medio inferior al umbral definido | Bloquea; notifica falta de base documental | 422 |

---

## Niveles de confianza RAG

La confianza se calcula a partir del score de relevancia de Azure AI Search. No refleja una opinión del modelo generativo.

| Nivel | Score RAG medio | Comportamiento |
|---|---|---|
| `HIGH` | ≥ 0.75 | Respuesta generada con normalidad. Fuente y fragmento citados explícitamente. `grounded: true`. |
| `MEDIUM` | 0.40 – 0.74 | Respuesta generada con advertencia de contexto parcial. Fuente citada. `grounded: true`. |
| `LOW` | < 0.40 | Se activa `grounding_required`. Generación bloqueada. Evento registrado con `trace_id`. |

---

## Modelo de trazabilidad

Cada interacción genera un `trace_id` único (UUID) que permite reconstruir completamente el flujo ejecutado.

### Campos auditables por `trace_id`

| Campo | Descripción |
|---|---|
| `mensaje_entrada` | Contenido original del mensaje recibido. |
| `contexto` | Dominio seleccionado (`pareja`, `trabajo`, `social`, `negociacion`). |
| `idioma_detectado` | Idioma identificado automáticamente. |
| `documentos_recuperados` | Fragmentos exactos de Gottman o Johnson utilizados, con score de relevancia. |
| `confidence` | Nivel calculado a partir del score RAG medio. |
| `prompt_final` | Prompt completo enviado al modelo durante la ejecución. |
| `prompt_version` | Versión del prompt utilizado. |
| `modelo` + `version_modelo` | Modelo generativo y versión de despliegue. |
| `policy_applied` | Política activada, si se produjo un bloqueo. |
| `grounded` | Indica si la respuesta contiene evidencia documental válida. |
| `latencia_ms` | Latencia total del pipeline. |
| `timestamp` | Fecha y hora exacta de la interacción (ISO 8601). |

### Trazabilidad en activaciones de política

Cuando se activa `crisis_detected`, la traza incluye además:

```json
{
  "trace_id": "a1b2c3d4-...",
  "timestamp": "2025-04-25T10:30:00Z",
  "mensaje_entrada": "...",
  "policy_applied": "crisis_detected",
  "crisis_clasificacion": {
    "crisis": true,
    "nivel": "ALTA",
    "razon": "Lenguaje que sugiere ausencia de razón para continuar"
  },
  "prompt_version": "CrisisClassifier-v1.0",
  "latencia_clasificador_ms": 420
}
```

---

## Versionado de prompts

El sistema mantiene control de versiones sobre todos los prompts del pipeline. Cada modificación genera una nueva versión identificable.

### Esquema de versiones

| Versión | Significado |
|---|---|
| `v1.0` | Versión inicial. |
| `v1.x` | Ajustes incrementales en estructura, grounding o reglas. |
| `v2.0` | Cambio estructural significativo en el comportamiento esperado. |

### Prompts versionados en el sistema

- `AnalysisPipeline` — prompt principal de análisis pragmático.
- `CrisisClassifier` — prompt del clasificador de crisis emocional.
- `EvaluationJob` — prompt del job de evaluación automatizada.

### Campos de versión en respuestas

Todas las respuestas de éxito y las trazas de auditoría incluyen `prompt_version`. El endpoint `GET /prompts` expone el historial completo con métricas por versión, permitiendo comparar `grounded_rate` y latencia antes y después de cada cambio.

### Capacidades del versionado

- **Comparación** — métricas objetivas entre versiones registradas en Azure Monitor.
- **Rollback** — posibilidad de reactivar una versión anterior ante comportamiento inesperado.
- **Auditoría** — cada respuesta generada queda asociada a la versión exacta del prompt utilizado.

---

## Contrato de error unificado

Todas las respuestas de error siguen el mismo esquema:

```json
{
  "error": "<tipo_error>",
  "policy": "<nombre_politica>",
  "message": "<descripción legible>",
  "trace_id": "<uuid>"
}
```

| Campo | Descripción |
|---|---|
| `error` | Tipo de error: `policy_violation`, `security_violation`, `rate_limit_exceeded`. |
| `policy` | Nombre normalizado de la política activada. Ver tabla de políticas. |
| `message` | Descripción legible del rechazo, orientada al consumidor de la API. |
| `trace_id` | Identificador de la interacción bloqueada. Siempre presente. |

Campos adicionales según política:

| Política | Campo adicional |
|---|---|
| `grounding_required` | `confidence` — nivel y razón del score insuficiente. |
| `unsupported_language` | `idioma_detectado` — código ISO del idioma identificado. |
| `user_rate_limit` | `retry_after_segundos` — tiempo de espera hasta el siguiente intento permitido. |

---

*SubtextAI — Contrato de API v2.0 — Abril 2025*
