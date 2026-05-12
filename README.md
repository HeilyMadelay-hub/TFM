# SubtextAI — Sistema de Análisis Pragmático de Comunicación Ambigua

> **Trabajo de Fin de Máster (TFM)**
> Sistema conversacional con inteligencia artificial que interpreta el subtexto de mensajes ambiguos en contextos de pareja, trabajo, entorno social y negociación, construido sobre una arquitectura cloud profesional en Azure con gobernanza de agentes, trazabilidad completa y evaluación continua automatizada.

---

## Descripción General

SubtextAI es una aplicación full stack desplegada en Azure que aplica principios de **gobernanza de agentes de IA** para analizar el significado pragmático de mensajes cotidianos. A partir de un mensaje y un contexto seleccionado por el usuario, el sistema interpreta el contenido implícito fundamentándose exclusivamente en fuentes documentales reales —indexadas y configurables— a través de un pipeline de Generación Aumentada por Recuperación (RAG) gobernado.

El proyecto demuestra que es posible construir un sistema de IA conversacional que no solo sea útil, sino también **auditable, controlado y explicable**: cada respuesta generada puede ser rastreada hasta los fragmentos documentales que la fundamentaron, el prompt exacto empleado y la versión del modelo invocado.

El corpus documental del sistema es **intercambiable y configurable**. La arquitectura de recuperación no está acoplada a ninguna fuente específica: cualquier colección de documentos en formato compatible puede ser indexada y utilizada sin modificar la lógica de gobernanza ni el pipeline de generación.

### Pilares de diseño

El sistema se construye sobre cuatro principios de gobernanza que forman parte de su arquitectura operativa, no de su configuración:

**Políticas explícitas codificadas** — reglas duras implementadas directamente en el pipeline que impiden que el modelo se ejecute si se violan condiciones críticas (longitud mínima del mensaje, idioma soportado, detección de crisis emocional, detección de prompt injection, límite de tasa por usuario). Estas reglas no residen en el prompt del sistema: residen en el código.

**Grounding obligatorio** — ninguna respuesta se entrega sin evidencia documental verificable. El nivel de confianza se calcula objetivamente a partir del score de relevancia del motor de búsqueda, y si ese score cae por debajo del umbral definido, la generación se bloquea antes de invocar al modelo principal.

**Trazabilidad completa** — cada interacción genera un `trace_id` único que permite reconstruir todo el flujo de ejecución: qué documentos influyeron en la respuesta y con qué score de relevancia, qué prompt exacto se usó y en qué versión, qué modelo fue invocado, qué políticas se evaluaron y cuál fue la latencia real del sistema.

**Evaluación continua automatizada** — un job periódico ejecuta preguntas de prueba predefinidas sobre el sistema en producción y publica las métricas resultantes en Azure Monitor, permitiendo detectar desviaciones de comportamiento sin intervención manual.

---

## Arquitectura del Sistema

El sistema se organiza en una arquitectura cloud moderna sobre Azure, con separación clara entre frontend, backend y servicios de IA.

```
┌──────────────────────────────────────────────────────────────┐
│                    USUARIO (navegador)                        │
└────────────────────────┬─────────────────────────────────────┘
                         │ HTTPS
┌────────────────────────▼─────────────────────────────────────┐
│           Azure Static Web Apps (Frontend React)             │
└────────────────────────┬─────────────────────────────────────┘
                         │ REST API
┌────────────────────────▼─────────────────────────────────────┐
│            Azure App Service (.NET Backend)                  │
│                                                              │
│  Pipeline de gobernanza (cortocircuito en cascada):          │
│                                                              │
│  1. Validaciones de política (sin LLM)                       │
│     → longitud mínima, idioma, rate limit, prompt injection  │
│                                                              │
│  2. Clasificador de crisis emocional (LLM call #1)           │
│     → GPT-4o-mini con prompt especializado (~150–200 tokens) │
│                                                              │
│  3. RAG — Recuperación documental                            │
│     → Azure AI Search (búsqueda híbrida: vectorial+semántica)│
│     → Reordenamiento de resultados antes de la generación    │
│                                                              │
│  4. LLM principal — Análisis pragmático (LLM call #2)        │
│     → GPT-4o-mini con grounding obligatorio                  │
│                                                              │
│  5. Trazabilidad                                             │
│     → Registro completo en base de datos con trace_id único  │
└──┬───────────┬──────────────────┬────────────────────────────┘
   │           │                  │
   ▼           ▼                  ▼
Azure OpenAI  Azure AI Search   Azure SQL / Cosmos DB
(GPT-4o-mini) (Índice documental (Trazas, métricas,
 clasificador  configurable con   versiones de prompts)
 + análisis)   búsqueda híbrida)        │
                                        ▼
                                 Azure Monitor
                                 (Métricas de calidad
                                  + alertas)
                                        │
                                        ▼
                                 Azure Functions
                                 (Job de evaluación
                                  automatizada periódica)
```

### Pipeline de gobernanza (cortocircuito en cascada)

El pipeline implementa un patrón de **cortocircuito**: si cualquier política se activa en un paso, el flujo se detiene inmediatamente y el modelo principal nunca se invoca. Esto garantiza que el sistema no produzca respuestas fuera del control establecido.

```
mensaje_usuario
      │
      ▼
┌─────────────────┐
│ 1. Políticas    │  → longitud <5 palabras, idioma, rate limit, injection
└────────┬────────┘
         │ si no viola
         ▼
┌─────────────────┐
│ 2. Crisis       │  → LLM call #1: clasificador especializado (~150 tokens)
│   emocional     │
└────────┬────────┘
    crisis=false              crisis=true
         │                        │
         ▼                        ▼
┌─────────────────┐    ┌────────────────────┐
│   3. RAG        │    │ Política activada   │ → HTTP 422 + deriva a profesional
└────────┬────────┘    └────────────────────┘
         │ si score_rag >= 0.40
         ▼
┌─────────────────┐
│ 4. LLM          │  → LLM call #2: análisis pragmático con grounding
│   principal     │
└────────┬────────┘
         ▼
┌─────────────────┐
│ 5. Trazabilidad │  → Registro completo en BBDD con trace_id único
└─────────────────┘
```

---

## Sistema de Recuperación de Conocimiento (RAG)

El componente RAG es el núcleo del grounding documental del sistema. Su función es recuperar los fragmentos más relevantes del corpus indexado antes de que el modelo principal genere cualquier respuesta.

### Diseño desacoplado

El sistema RAG está construido de forma **genérica e independiente del corpus**. La selección de fuentes documentales es una decisión de configuración, no de arquitectura. Cualquier colección de documentos en formato compatible (PDF, texto plano, o estructurado) puede ser procesada por el pipeline de indexación y utilizada sin modificar el código del backend ni la lógica de gobernanza.

### Pipeline de indexación

Los documentos se procesan en tres fases antes de ser consultables:

**Chunking** — los documentos se dividen en fragmentos de tamaño controlado para equilibrar precisión de recuperación y coherencia contextual.

**Vectorización** — cada fragmento se transforma en un vector de embeddings mediante el servicio de embeddings de Azure OpenAI, capturando su representación semántica.

**Indexación híbrida** — los vectores y el texto se almacenan en Azure AI Search, que combina búsqueda vectorial (similitud semántica) y búsqueda por palabras clave (BM25) en una búsqueda híbrida para maximizar la cobertura de recuperación.

### Reordenamiento y selección

Tras la recuperación inicial, el sistema aplica un paso de **reordenamiento** (*reranking*) que reordena los resultados candidatos por relevancia contextual antes de pasarlos al modelo principal. Este paso mejora la selección de fragmentos cuando la consulta del usuario es semántica pero ambigua.

### Umbral de confianza

El sistema evalúa el score de relevancia medio de los fragmentos recuperados y lo traduce en un nivel de confianza explícito que acompaña a cada respuesta:

| Nivel | Score RAG medio | Comportamiento |
|-------|-----------------|----------------|
| **ALTA** | ≥ 0.75 | Respuesta completa con cita de fragmento y sección. `grounded=true`. |
| **MEDIA** | 0.40 – 0.74 | Respuesta con advertencia de contexto parcial. Se cita fuente disponible. |
| **BAJA** | < 0.40 | Se activa `grounding_obligatorio`. El sistema no genera respuesta ni inventa. |

Si el score medio cae por debajo del umbral definido (0.40 por defecto), la política `grounding_obligatorio` se activa y la generación se bloquea. El sistema responde con un mensaje explícito indicando que no dispone de base documental suficiente.

---

## Gobernanza del Agente

La gobernanza en SubtextAI se implementa en cuatro capas complementarias que operan de forma independiente al modelo generativo.

### Políticas explícitas

Las políticas son reglas duras codificadas en el pipeline, no en el prompt. El modelo nunca se invoca si alguna política se ha violado. Esta separación garantiza que el comportamiento de seguridad no dependa del modelo ni de su configuración.

| Política | Condición de activación | Acción | HTTP |
|----------|------------------------|--------|------|
| `mensaje_minimo` | Menos de 5 palabras | Rechaza y solicita más contexto | 422 |
| `grounding_obligatorio` | Score RAG medio < 0.40 | Bloquea generación sin base documental | 422 |
| `crisis_detected` | Señales de crisis emocional detectadas | Bloquea y deriva a profesional | 422 |
| `prompt_injection` | Input malicioso detectado | Bloquea y registra en auditoría | 400 |
| `idioma_no_soportado` | Idioma distinto al configurado | Rechaza informando el idioma detectado | 422 |
| `rate_limit_usuario` | > 10 req/min por usuario/IP | Bloquea temporalmente con registro en BBDD | 429 |
| `respuesta_sin_fuente` | Respuesta generada sin evidencia documental válida | Bloquea la respuesta antes de entregarla | 422 |
| `confianza_insuficiente` | Score medio inferior al umbral definido | Rechaza generación | 422 |
| `politica_no_cumplida` | Conflicto interno o inconsistencia entre políticas | Bloquea flujo | 500 |

### Clasificador de crisis emocional

El sistema incorpora una fase de clasificación previa que evalúa si el mensaje del usuario contiene señales de crisis emocional severa antes de ejecutar cualquier análisis pragmático. Esta clasificación se realiza mediante una llamada independiente al modelo (LLM call #1) con un prompt especializado y minimalista (~150–200 tokens), diseñado exclusivamente para esta tarea.

La decisión de implementar este clasificador mediante prompt LLM —en lugar de un modelo externo fine-tuned o una lista de palabras clave— fue adoptada priorizando la coherencia arquitectónica, la auditabilidad y la velocidad de desarrollo en el contexto de un MVP. Esta decisión está documentada junto con sus alternativas evaluadas y sus condiciones de revisión futura.

El prompt del clasificador está versionado igual que el resto de prompts del sistema, y cada activación de la política `crisis_detected` queda registrada con su `trace_id` y nivel de severidad detectado.

### Trazabilidad completa

Cada interacción del sistema genera un registro persistente identificado por un `trace_id` único. Este registro incluye:

- El mensaje de entrada y el contexto seleccionado
- Los fragmentos documentales recuperados y sus scores de relevancia individuales
- El prompt exacto enviado al modelo y su versión (`prompt_version`)
- La versión del modelo utilizado
- El nivel de confianza calculado objetivamente
- La política aplicada (si alguna fue violada)
- El campo `grounded` (verdadero/falso)
- La latencia total del sistema
- El timestamp de la interacción

Este registro permite reconstruir cualquier respuesta generada y auditar completamente la cadena de decisiones que la produjo.

### Evaluación continua automatizada

Un job periódico ejecutado en Azure Functions lanza un conjunto predefinido de preguntas de prueba sobre el sistema y publica los resultados en Azure Monitor. Las métricas evaluadas incluyen: porcentaje de respuestas grounded, tasa de cumplimiento de políticas, distribución de niveles de confianza, latencia media y porcentaje de cache hit. Este pipeline permite detectar desviaciones de comportamiento entre versiones de prompt y validar cambios antes de promoverlos a producción.

### Versionado de prompts

Cada cambio en el prompt del sistema genera una nueva versión versionada (v1.1, v1.2…). El campo `prompt_version` aparece en todas las respuestas del sistema y en el endpoint `/audit`, lo que permite comparar métricas de calidad entre versiones y justificar cada cambio con datos observables.

---

## Tecnologías Utilizadas

**Frontend** — React desplegado en Azure Static Web Apps. Interfaz conversacional con selección de contexto y visualización de resultados con fuente citada, nivel de confianza y `trace_id` accesible para auditoría.

**Backend** — ASP.NET Core (C#) desplegado en Azure App Service. Implementa el pipeline completo de gobernanza: validaciones, clasificación de crisis, integración RAG, análisis pragmático y registro de trazabilidad.

**Motor de IA generativa** — Azure OpenAI Service con modelo GPT-4o-mini, utilizado tanto para la clasificación de crisis emocional (LLM call #1) como para el análisis pragmático principal (LLM call #2).

**Recuperación documental (RAG)** — Azure AI Search con indexación de documentos mediante búsqueda híbrida (vectorial + semántica) y reordenamiento de resultados para optimizar la selección de fragmentos relevantes.

**Embeddings** — Azure OpenAI Embeddings para vectorización de fragmentos documentales durante la fase de indexación.

**Base de datos** — almacenamiento persistente de trazas completas por `trace_id`, incluyendo mensajes, fragmentos recuperados, scores de relevancia, prompts utilizados, versiones del modelo y políticas aplicadas. Compatible con Azure SQL Database o Azure Cosmos DB.

**Observabilidad** — Azure Monitor para registro de métricas de calidad del sistema: porcentaje grounded, cumplimiento de políticas, latencia media, distribución de confianza y cache hit rate.

**Evaluación automatizada** — Azure Functions ejecuta periódicamente el conjunto de preguntas de prueba y publica los resultados en Azure Monitor.

**Seguridad y control** — rate limiting por usuario/IP (máx. 10 req/min), detección de prompt injection y clasificación de crisis emocional como paso previo al análisis pragmático.

**ORM y migraciones** — Entity Framework Core para la gestión del esquema de base de datos y las migraciones.

**Toolchain frontend** — Vite como bundler y TypeScript para tipado estático en la capa de integración con la API.

---

## Instalación

### Requisitos previos

Antes de comenzar, asegúrate de disponer de lo siguiente:

- .NET 8 SDK o superior
- Node.js 18+ y npm
- Una suscripción de Azure activa con acceso a Azure OpenAI Service, Azure AI Search y Azure App Service
- Azure CLI instalado y configurado (`az login`)
- Acceso al modelo `gpt-4o-mini` en tu instancia de Azure OpenAI
- Una instancia de base de datos compatible (Azure SQL Database o Azure Cosmos DB)

### 1. Clonar el repositorio

```bash
git clone https://github.com/<tu-usuario>/subtextai.git
cd subtextai
```

### 2. Configurar el backend (.NET)

Navega al directorio del backend e instala las dependencias:

```bash
cd backend
dotnet restore
```

Crea el archivo de configuración local copiando la plantilla:

```bash
cp appsettings.Example.json appsettings.Development.json
```

Edita `appsettings.Development.json` con tus credenciales de Azure:

```json
{
  "AzureOpenAI": {
    "Endpoint": "https://<tu-instancia>.openai.azure.com/",
    "ApiKey": "<tu-api-key>",
    "DeploymentName": "gpt-4o-mini",
    "ModelVersion": "2024-07"
  },
  "AzureAISearch": {
    "Endpoint": "https://<tu-instancia>.search.windows.net",
    "ApiKey": "<tu-api-key>",
    "IndexName": "<nombre-del-indice>"
  },
  "ConnectionStrings": {
    "DefaultConnection": "<tu-cadena-de-conexion>"
  },
  "Policies": {
    "MinWordCount": 5,
    "RagScoreThreshold": 0.40,
    "RateLimitPerMinute": 10
  }
}
```

Aplica las migraciones de base de datos:

```bash
dotnet ef database update
```

### 3. Indexar los documentos en Azure AI Search

Coloca los documentos fuente en `scripts/docs/` en formato PDF o texto plano. Ejecuta el script de indexación, que se encarga del chunking, la vectorización y la carga en Azure AI Search:

```bash
cd scripts
dotnet run --project IndexDocuments -- \
  --source ./docs \
  --index <nombre-del-indice>
```

El corpus documental es completamente configurable. El script acepta cualquier colección de documentos compatible sin requerir cambios en el pipeline de gobernanza ni en el backend.

### 4. Configurar el frontend (React)

Navega al directorio del frontend e instala las dependencias:

```bash
cd ../frontend
npm install
```

Crea el archivo de variables de entorno local:

```bash
cp .env.example .env.local
```

Edita `.env.local` con la URL del backend local:

```env
VITE_API_BASE_URL=https://localhost:7000/api/v1
```

---

## Ejecución en Desarrollo

### Levantar el backend

Desde el directorio `backend/`:

```bash
dotnet run --environment Development
```

El backend quedará disponible en `https://localhost:7000`. El estado del sistema puede verificarse en `https://localhost:7000/api/v1/health`.

### Levantar el frontend

Desde el directorio `frontend/`, en un terminal separado:

```bash
npm run dev
```

La aplicación estará disponible en `http://localhost:5173`.

---

## Despliegue en Azure

### Backend en Azure App Service

```bash
cd backend
dotnet publish -c Release -o ./publish
az webapp deploy \
  --resource-group subtextai-rg \
  --name subtextai-api \
  --src-path ./publish
```

### Frontend en Azure Static Web Apps

```bash
cd frontend
npm run build
az staticwebapp deploy \
  --app-location "." \
  --output-location "dist" \
  --name subtextai-frontend
```

Actualiza la variable `VITE_API_BASE_URL` en la configuración de la Static Web App para apuntar a la URL de producción del backend.

---

## Estructura de Carpetas

```
subtextai/
│
├── backend/                          # API REST en ASP.NET Core
│   ├── Controllers/                  # Endpoints: Analizar, Evaluar, Audit, Metricas, Prompts, Health
│   ├── Pipeline/                     # Lógica del pipeline de gobernanza
│   │   ├── PolicyEngine.cs           # Validaciones de política sin LLM (longitud, idioma, etc.)
│   │   ├── CrisisClassifier.cs       # Clasificador de crisis emocional (LLM call #1)
│   │   ├── RagService.cs             # Recuperación documental con Azure AI Search
│   │   ├── LlmService.cs             # Análisis pragmático principal (LLM call #2)
│   │   └── TraceStore.cs             # Persistencia de trazas con trace_id
│   ├── Models/                       # DTOs de request y response
│   ├── Prompts/                      # Prompts versionados del sistema
│   │   ├── CrisisClassifier-v1.0.txt # Prompt especializado de clasificación de crisis
│   │   └── AnalisisPragmatico-v1.2.txt # Prompt principal del sistema
│   ├── Migrations/                   # Migraciones de base de datos (EF Core)
│   ├── appsettings.json              # Configuración base
│   └── appsettings.Example.json      # Plantilla de configuración (sin secretos)
│
├── frontend/                         # Aplicación React
│   ├── src/
│   │   ├── components/               # Componentes UI reutilizables
│   │   ├── pages/                    # Vistas principales (Análisis, Auditoría, Métricas)
│   │   ├── services/                 # Capa de integración con la API REST
│   │   └── types/                    # Tipos TypeScript para respuestas del backend
│   ├── .env.example                  # Plantilla de variables de entorno
│   └── vite.config.ts
│
├── scripts/                          # Utilidades de indexación y evaluación
│   ├── IndexDocuments/               # Script de carga de documentos en Azure AI Search
│   │   └── docs/                     # Directorio de documentos fuente (configurable)
│   └── EvaluationJob/                # Preguntas de prueba para evaluación automatizada
│
└── docs/                             # Documentación del proyecto
```

---

## Endpoints de la API

**Base URL:** `https://subtextai-api.azurewebsites.net/api/v1`

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| `POST` | `/analizar` | Interpreta el significado pragmático de un mensaje ambiguo |
| `POST` | `/evaluar` | Evalúa una respuesta propuesta por el usuario ante un mensaje analizado |
| `GET` | `/audit/{trace_id}` | Devuelve la trazabilidad completa y versionada de una respuesta |
| `GET` | `/metricas` | Métricas de calidad agregadas del sistema (últimos 7 días) |
| `GET` | `/prompts` | Historial de versiones de prompts con métricas comparativas |
| `GET` | `/health` | Estado del sistema y de todos los servicios Azure conectados |

### POST /analizar

Recibe un mensaje y un contexto, lo pasa por el pipeline completo de gobernanza y devuelve el análisis pragmático fundamentado en fuentes documentales.

**Request:**
```json
{
  "mensaje": "Solo quiero fluir y ver qué pasa",
  "contexto": "pareja | trabajo | social | negociacion"
}
```

**Response 200:**
```json
{
  "significado": "Baja implicación emocional, evasión de compromiso explícito",
  "senales": ["evasión", "ambigüedad intencional", "pasividad"],
  "nivel_alerta": "MEDIO",
  "recomendacion": "Mantener límites claros. No invertir energía emocional sin reciprocidad.",
  "fuente": {
    "documento": "<título del documento fuente>",
    "fragmento": "<sección o capítulo relevante>"
  },
  "confianza": {
    "nivel": "ALTA",
    "razon": "score_rag_medio: 0.87 — contexto documental sólido"
  },
  "grounded": true,
  "politica_aplicada": "ninguna",
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

**Códigos de error posibles:**

| Código | Política | Descripción |
|--------|----------|-------------|
| `422` | `mensaje_minimo` | Mensaje con menos de 5 palabras |
| `422` | `grounding_obligatorio` | Score RAG medio < 0.40; sin base documental suficiente |
| `422` | `crisis_detected` | Señales de crisis emocional detectadas; se deriva a profesional |
| `422` | `idioma_no_soportado` | Idioma distinto al español |
| `400` | `prompt_injection` | Input malicioso detectado y bloqueado |
| `429` | `rate_limit_usuario` | Más de 10 requests/minuto por usuario/IP |

### POST /evaluar

Evalúa la respuesta propuesta por el usuario ante un mensaje ya analizado. Devuelve una valoración de la respuesta con probabilidad de éxito, fortalezas, áreas de mejora y sugerencia alternativa, todo ello grounded en fuentes documentales.

### GET /audit/{trace_id}

Devuelve el registro completo de una interacción identificada por su `trace_id`. Incluye los documentos recuperados con sus scores de relevancia individuales, el prompt exacto enviado al modelo en su versión específica, el modelo utilizado, las políticas evaluadas y la latencia real.

### GET /metricas

Devuelve métricas agregadas del sistema correspondientes a los últimos 7 días: porcentaje de respuestas grounded, distribución de niveles de confianza, políticas activadas por tipo, latencia media, cache hit rate y versiones de prompt activas.

### GET /prompts

Devuelve el historial de versiones de prompts del sistema con fecha de activación, descripción de cambios y métricas de rendimiento asociadas a cada versión. Permite comparar el comportamiento del sistema antes y después de cada modificación.

### GET /health

Devuelve el estado del sistema y de cada servicio Azure conectado (Azure OpenAI, Azure AI Search, base de datos, Azure Monitor), junto con la versión del sistema y el prompt activo.

---

## Trazabilidad y Observabilidad

SubtextAI implementa observabilidad en dos niveles complementarios.

### Trazabilidad por interacción

Cada análisis genera un `trace_id` único que permite reconstruir completamente la cadena de decisiones del agente. La auditoría de cualquier respuesta es accesible en tiempo real a través del endpoint `GET /audit/{trace_id}`, que expone:

- Los fragmentos documentales que influyeron en la respuesta y su score de relevancia
- El prompt exacto enviado al modelo y su versión
- La versión del modelo invocado
- Si se activó alguna política y cuál
- El nivel de confianza calculado objetivamente a partir del score RAG
- La latencia total del sistema
- Si el rate limit fue alcanzado por ese usuario

### Observabilidad del sistema

Azure Monitor recibe las métricas agregadas del sistema generadas tanto por las interacciones en producción como por el job de evaluación automatizada. Estas métricas permiten monitorizar la evolución del comportamiento del agente a lo largo del tiempo, detectar desviaciones y comparar el rendimiento entre versiones de prompt de forma cuantitativa.

---

## Mejoras Futuras

El diseño del sistema establece un conjunto de evoluciones planificadas con condiciones de revisión explícitas documentadas durante el desarrollo.

**Clasificador de crisis emocional especializado** — cuando el volumen supere los 10.000 requests/día, se evaluará la sustitución del clasificador basado en prompt por un modelo fine-tuned (DistilBERT u equivalente) para reducir coste por token y mejorar precisión. Esta alternativa fue descartada en el MVP por el tiempo de integración adicional, pero está documentada como evolución natural del sistema.

**Soporte multilingüe** — extensión del sistema para soportar idiomas adicionales más allá del español, con prompts adaptados y pipelines de evaluación específicos por idioma.

**Expansión del corpus documental** — incorporación de nuevas fuentes especializadas para ampliar la capacidad interpretativa del sistema sin modificar su lógica de gobernanza ni su pipeline de recuperación.

**Panel de auditoría** — interfaz de administración para visualizar trazas, comparar versiones de prompts y revisar eventos de activación de políticas sin necesidad de consultar la base de datos directamente.

**Mejora de observabilidad** — ampliación de los indicadores registrados en Azure Monitor para detectar tendencias de uso, errores recurrentes y cambios de comportamiento del agente a lo largo del tiempo.

**Clasificador determinista adicional** — en escenarios que requieran auditoría regulatoria, se añadiría una segunda capa de clasificación basada en reglas deterministas sobre la clasificación LLM existente, reforzando el registro de auditoría y la auditabilidad del sistema.

**Caché semántico** — mejora del sistema de cacheo actual para incorporar similitud semántica entre consultas, reduciendo llamadas al modelo para mensajes funcionalmente equivalentes y optimizando el coste operativo.

---

## Autoras

**Nombre:** Elizabeth Sáenz Camacho y Heily Madelay Tandazo  
**Máster:** Desarrollo Full Stack & Arquitecturas Cloud  
**Institución:** Tajamar  
**Año:** 2026

---

## Licencia

Este proyecto ha sido desarrollado como Trabajo de Fin de Máster con fines académicos.
_[Especificar licencia si aplica]_
