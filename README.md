# SubtextAI â€” Sistema de AnÃ¡lisis PragmÃ¡tico de ComunicaciÃ³n Ambigua

> **Trabajo de Fin de MÃ¡ster (TFM)**
> Sistema conversacional con inteligencia artificial que interpreta el subtexto de mensajes ambiguos en contextos de pareja, trabajo, entorno social y negociaciÃ³n, construido sobre una arquitectura cloud profesional en Azure con gobernanza de agentes, trazabilidad completa y evaluaciÃ³n continua automatizada.

---

## DescripciÃ³n General

SubtextAI es una aplicaciÃ³n full stack desplegada en Azure que aplica principios de **gobernanza de agentes de IA** para analizar el significado pragmÃ¡tico de mensajes cotidianos. A partir de un mensaje y un contexto seleccionado por el usuario, el sistema interpreta el contenido implÃ­cito fundamentÃ¡ndose exclusivamente en fuentes documentales reales â€”indexadas y configurablesâ€” a travÃ©s de un pipeline de GeneraciÃ³n Aumentada por RecuperaciÃ³n (RAG) gobernado.

El proyecto demuestra que es posible construir un sistema de IA conversacional que no solo sea Ãºtil, sino tambiÃ©n **auditable, controlado y explicable**: cada respuesta generada puede ser rastreada hasta los fragmentos documentales que la fundamentaron, el prompt exacto empleado y la versiÃ³n del modelo invocado.

El corpus documental del sistema es **intercambiable y configurable**. La arquitectura de recuperaciÃ³n no estÃ¡ acoplada a ninguna fuente especÃ­fica: cualquier colecciÃ³n de documentos en formato compatible puede ser indexada y utilizada sin modificar la lÃ³gica de gobernanza ni el pipeline de generaciÃ³n.

### Pilares de diseÃ±o

El sistema se construye sobre cuatro principios de gobernanza que forman parte de su arquitectura operativa, no de su configuraciÃ³n:

**PolÃ­ticas explÃ­citas codificadas** â€” reglas duras implementadas directamente en el pipeline que impiden que el modelo se ejecute si se violan condiciones crÃ­ticas (longitud mÃ­nima del mensaje, idioma soportado, detecciÃ³n de crisis emocional, detecciÃ³n de prompt injection, lÃ­mite de tasa por usuario). Estas reglas no residen en el prompt del sistema: residen en el cÃ³digo.

**Grounding obligatorio** â€” ninguna respuesta se entrega sin evidencia documental verificable. El nivel de confianza se calcula objetivamente a partir del score de relevancia del motor de bÃºsqueda, y si ese score cae por debajo del umbral definido, la generaciÃ³n se bloquea antes de invocar al modelo principal.

**Trazabilidad completa** â€” cada interacciÃ³n genera un `trace_id` Ãºnico que permite reconstruir todo el flujo de ejecuciÃ³n: quÃ© documentos influyeron en la respuesta y con quÃ© score de relevancia, quÃ© prompt exacto se usÃ³ y en quÃ© versiÃ³n, quÃ© modelo fue invocado, quÃ© polÃ­ticas se evaluaron y cuÃ¡l fue la latencia real del sistema.

**EvaluaciÃ³n continua automatizada** â€” un job periÃ³dico ejecuta preguntas de prueba predefinidas sobre el sistema en producciÃ³n y publica las mÃ©tricas resultantes en Azure Monitor, permitiendo detectar desviaciones de comportamiento sin intervenciÃ³n manual.

---

## Arquitectura del Sistema

El sistema se organiza en una arquitectura cloud moderna sobre Azure, con separaciÃ³n clara entre frontend, backend y servicios de IA.

```
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚                    USUARIO (navegador)                        â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
                         â”‚ HTTPS
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â–¼â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚           Azure Static Web Apps (Frontend React)             â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
                         â”‚ REST API
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â–¼â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚            Azure App Service (.NET Backend)                  â”‚
â”‚                                                              â”‚
â”‚  Pipeline de gobernanza (cortocircuito en cascada):          â”‚
â”‚                                                              â”‚
â”‚  1. Validaciones de polÃ­tica (sin LLM)                       â”‚
â”‚     â†’ longitud mÃ­nima, idioma, rate limit, prompt injection  â”‚
â”‚                                                              â”‚
â”‚  2. Clasificador de crisis emocional (LLM call #1)           â”‚
â”‚     â†’ GPT-4o-mini con prompt especializado (~150â€“200 tokens) â”‚
â”‚                                                              â”‚
â”‚  3. RAG â€” RecuperaciÃ³n documental                            â”‚
â”‚     â†’ Azure AI Search (bÃºsqueda hÃ­brida: vectorial+semÃ¡ntica)â”‚
â”‚     â†’ Reordenamiento de resultados antes de la generaciÃ³n    â”‚
â”‚                                                              â”‚
â”‚  4. LLM principal â€” AnÃ¡lisis pragmÃ¡tico (LLM call #2)        â”‚
â”‚     â†’ GPT-4o-mini con grounding obligatorio                  â”‚
â”‚                                                              â”‚
â”‚  5. Trazabilidad                                             â”‚
â”‚     â†’ Registro completo en base de datos con trace_id Ãºnico  â”‚
â””â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
   â”‚           â”‚                  â”‚
   â–¼           â–¼                  â–¼
Azure OpenAI  Azure AI Search   Azure SQL / Cosmos DB
(GPT-4o-mini) (Ãndice documental (Trazas, mÃ©tricas,
 clasificador  configurable con   versiones de prompts)
 + anÃ¡lisis)   bÃºsqueda hÃ­brida)        â”‚
                                        â–¼
                                 Azure Monitor
                                 (MÃ©tricas de calidad
                                  + alertas)
                                        â”‚
                                        â–¼
                                 Azure Functions
                                 (Job de evaluaciÃ³n
                                  automatizada periÃ³dica)
```

### Pipeline de gobernanza (cortocircuito en cascada)

El pipeline implementa un patrÃ³n de **cortocircuito**: si cualquier polÃ­tica se activa en un paso, el flujo se detiene inmediatamente y el modelo principal nunca se invoca. Esto garantiza que el sistema no produzca respuestas fuera del control establecido.

```
mensaje_usuario
      â”‚
      â–¼
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚ 1. PolÃ­ticas    â”‚  â†’ longitud <5 palabras, idioma, rate limit, injection
â””â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”˜
         â”‚ si no viola
         â–¼
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚ 2. Crisis       â”‚  â†’ LLM call #1: clasificador especializado (~150 tokens)
â”‚   emocional     â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”˜
    crisis=false              crisis=true
         â”‚                        â”‚
         â–¼                        â–¼
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”    â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚   3. RAG        â”‚    â”‚ PolÃ­tica activada   â”‚ â†’ HTTP 422 + deriva a profesional
â””â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”˜    â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
         â”‚ si score_rag >= 0.40
         â–¼
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚ 4. LLM          â”‚  â†’ LLM call #2: anÃ¡lisis pragmÃ¡tico con grounding
â”‚   principal     â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”˜
         â–¼
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚ 5. Trazabilidad â”‚  â†’ Registro completo en BBDD con trace_id Ãºnico
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
```

---

## Sistema de RecuperaciÃ³n de Conocimiento (RAG)

El componente RAG es el nÃºcleo del grounding documental del sistema. Su funciÃ³n es recuperar los fragmentos mÃ¡s relevantes del corpus indexado antes de que el modelo principal genere cualquier respuesta.

### DiseÃ±o desacoplado

El sistema RAG estÃ¡ construido de forma **genÃ©rica e independiente del corpus**. La selecciÃ³n de fuentes documentales es una decisiÃ³n de configuraciÃ³n, no de arquitectura. Cualquier colecciÃ³n de documentos en formato compatible (PDF, texto plano, o estructurado) puede ser procesada por el pipeline de indexaciÃ³n y utilizada sin modificar el cÃ³digo del backend ni la lÃ³gica de gobernanza.

### Pipeline de indexaciÃ³n

Los documentos se procesan en tres fases antes de ser consultables:

**Chunking** â€” los documentos se dividen en fragmentos de tamaÃ±o controlado para equilibrar precisiÃ³n de recuperaciÃ³n y coherencia contextual.

**VectorizaciÃ³n** â€” cada fragmento se transforma en un vector de embeddings mediante el servicio de embeddings de Azure OpenAI, capturando su representaciÃ³n semÃ¡ntica.

**IndexaciÃ³n hÃ­brida** â€” los vectores y el texto se almacenan en Azure AI Search, que combina bÃºsqueda vectorial (similitud semÃ¡ntica) y bÃºsqueda por palabras clave (BM25) en una bÃºsqueda hÃ­brida para maximizar la cobertura de recuperaciÃ³n.

### Reordenamiento y selecciÃ³n

Tras la recuperaciÃ³n inicial, el sistema aplica un paso de **reordenamiento** (*reranking*) que reordena los resultados candidatos por relevancia contextual antes de pasarlos al modelo principal. Este paso mejora la selecciÃ³n de fragmentos cuando la consulta del usuario es semÃ¡ntica pero ambigua.

### Umbral de confianza

El sistema evalÃºa el score de relevancia medio de los fragmentos recuperados y lo traduce en un nivel de confianza explÃ­cito que acompaÃ±a a cada respuesta:

| Nivel | Score RAG medio | Comportamiento |
|-------|-----------------|----------------|
| **ALTA** | â‰¥ 0.75 | Respuesta completa con cita de fragmento y secciÃ³n. `grounded=true`. |
| **MEDIA** | 0.40 â€“ 0.74 | Respuesta con advertencia de contexto parcial. Se cita fuente disponible. |
| **BAJA** | < 0.40 | Se activa `grounding_obligatorio`. El sistema no genera respuesta ni inventa. |

Si el score medio cae por debajo del umbral definido (0.40 por defecto), la polÃ­tica `grounding_obligatorio` se activa y la generaciÃ³n se bloquea. El sistema responde con un mensaje explÃ­cito indicando que no dispone de base documental suficiente.

---

## Gobernanza del Agente

La gobernanza en SubtextAI se implementa en cuatro capas complementarias que operan de forma independiente al modelo generativo.

### PolÃ­ticas explÃ­citas

Las polÃ­ticas son reglas duras codificadas en el pipeline, no en el prompt. El modelo nunca se invoca si alguna polÃ­tica se ha violado. Esta separaciÃ³n garantiza que el comportamiento de seguridad no dependa del modelo ni de su configuraciÃ³n.

| PolÃ­tica | CondiciÃ³n de activaciÃ³n | AcciÃ³n | HTTP |
|----------|------------------------|--------|------|
| `mensaje_minimo` | Menos de 5 palabras | Rechaza y solicita mÃ¡s contexto | 422 |
| `grounding_obligatorio` | Score RAG medio < 0.40 | Bloquea generaciÃ³n sin base documental | 422 |
| `crisis_detected` | SeÃ±ales de crisis emocional detectadas | Bloquea y deriva a profesional | 422 |
| `prompt_injection` | Input malicioso detectado | Bloquea y registra en auditorÃ­a | 400 |
| `idioma_no_soportado` | Idioma distinto al configurado | Rechaza informando el idioma detectado | 422 |
| `rate_limit_usuario` | > 10 req/min por usuario/IP | Bloquea temporalmente con registro en BBDD | 429 |
| `respuesta_sin_fuente` | Respuesta generada sin evidencia documental vÃ¡lida | Bloquea la respuesta antes de entregarla | 422 |
| `confianza_insuficiente` | Score medio inferior al umbral definido | Rechaza generaciÃ³n | 422 |
| `politica_no_cumplida` | Conflicto interno o inconsistencia entre polÃ­ticas | Bloquea flujo | 500 |

### Clasificador de crisis emocional

El sistema incorpora una fase de clasificaciÃ³n previa que evalÃºa si el mensaje del usuario contiene seÃ±ales de crisis emocional severa antes de ejecutar cualquier anÃ¡lisis pragmÃ¡tico. Esta clasificaciÃ³n se realiza mediante una llamada independiente al modelo (LLM call #1) con un prompt especializado y minimalista (~150â€“200 tokens), diseÃ±ado exclusivamente para esta tarea.

La decisiÃ³n de implementar este clasificador mediante prompt LLM â€”en lugar de un modelo externo fine-tuned o una lista de palabras claveâ€” fue adoptada priorizando la coherencia arquitectÃ³nica, la auditabilidad y la velocidad de desarrollo en el contexto de un MVP. Esta decisiÃ³n estÃ¡ documentada junto con sus alternativas evaluadas y sus condiciones de revisiÃ³n futura.

El prompt del clasificador estÃ¡ versionado igual que el resto de prompts del sistema, y cada activaciÃ³n de la polÃ­tica `crisis_detected` queda registrada con su `trace_id` y nivel de severidad detectado.

### Trazabilidad completa

Cada interacciÃ³n del sistema genera un registro persistente identificado por un `trace_id` Ãºnico. Este registro incluye:

- El mensaje de entrada y el contexto seleccionado
- Los fragmentos documentales recuperados y sus scores de relevancia individuales
- El prompt exacto enviado al modelo y su versiÃ³n (`prompt_version`)
- La versiÃ³n del modelo utilizado
- El nivel de confianza calculado objetivamente
- La polÃ­tica aplicada (si alguna fue violada)
- El campo `grounded` (verdadero/falso)
- La latencia total del sistema
- El timestamp de la interacciÃ³n

Este registro permite reconstruir cualquier respuesta generada y auditar completamente la cadena de decisiones que la produjo.

### EvaluaciÃ³n continua automatizada

Un job periÃ³dico ejecutado en Azure Functions lanza un conjunto predefinido de preguntas de prueba sobre el sistema y publica los resultados en Azure Monitor. Las mÃ©tricas evaluadas incluyen: porcentaje de respuestas grounded, tasa de cumplimiento de polÃ­ticas, distribuciÃ³n de niveles de confianza, latencia media y porcentaje de cache hit. Este pipeline permite detectar desviaciones de comportamiento entre versiones de prompt y validar cambios antes de promoverlos a producciÃ³n.

### Versionado de prompts

Cada cambio en el prompt del sistema genera una nueva versiÃ³n versionada (v1.1, v1.2â€¦). El campo `prompt_version` aparece en todas las respuestas del sistema y en el endpoint `/audit`, lo que permite comparar mÃ©tricas de calidad entre versiones y justificar cada cambio con datos observables.

---

## TecnologÃ­as Utilizadas

**Frontend** â€” React desplegado en Azure Static Web Apps. Interfaz conversacional con selecciÃ³n de contexto y visualizaciÃ³n de resultados con fuente citada, nivel de confianza y `trace_id` accesible para auditorÃ­a.

**Backend** â€” ASP.NET Core (C#) desplegado en Azure App Service. Implementa el pipeline completo de gobernanza: validaciones, clasificaciÃ³n de crisis, integraciÃ³n RAG, anÃ¡lisis pragmÃ¡tico y registro de trazabilidad.

**Motor de IA generativa** â€” Azure OpenAI Service con modelo GPT-4o-mini, utilizado tanto para la clasificaciÃ³n de crisis emocional (LLM call #1) como para el anÃ¡lisis pragmÃ¡tico principal (LLM call #2).

**RecuperaciÃ³n documental (RAG)** â€” Azure AI Search con indexaciÃ³n de documentos mediante bÃºsqueda hÃ­brida (vectorial + semÃ¡ntica) y reordenamiento de resultados para optimizar la selecciÃ³n de fragmentos relevantes.

**Embeddings** â€” Azure OpenAI Embeddings para vectorizaciÃ³n de fragmentos documentales durante la fase de indexaciÃ³n.

**Base de datos** â€” almacenamiento persistente de trazas completas por `trace_id`, incluyendo mensajes, fragmentos recuperados, scores de relevancia, prompts utilizados, versiones del modelo y polÃ­ticas aplicadas. Compatible con Azure SQL Database o Azure Cosmos DB.

**Observabilidad** â€” Azure Monitor para registro de mÃ©tricas de calidad del sistema: porcentaje grounded, cumplimiento de polÃ­ticas, latencia media, distribuciÃ³n de confianza y cache hit rate.

**EvaluaciÃ³n automatizada** â€” Azure Functions ejecuta periÃ³dicamente el conjunto de preguntas de prueba y publica los resultados en Azure Monitor.

**Seguridad y control** â€” rate limiting por usuario/IP (mÃ¡x. 10 req/min), detecciÃ³n de prompt injection y clasificaciÃ³n de crisis emocional como paso previo al anÃ¡lisis pragmÃ¡tico.

**ORM y migraciones** â€” Entity Framework Core para la gestiÃ³n del esquema de base de datos y las migraciones.

**Toolchain frontend** â€” Vite como bundler y TypeScript para tipado estÃ¡tico en la capa de integraciÃ³n con la API.

---

## InstalaciÃ³n

### Requisitos previos

Antes de comenzar, asegÃºrate de disponer de lo siguiente:

- .NET 8 SDK o superior
- Node.js 18+ y npm
- Una suscripciÃ³n de Azure activa con acceso a Azure OpenAI Service, Azure AI Search y Azure App Service
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

Crea el archivo de configuraciÃ³n local copiando la plantilla:

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

Coloca los documentos fuente en `scripts/docs/` en formato PDF o texto plano. Ejecuta el script de indexaciÃ³n, que se encarga del chunking, la vectorizaciÃ³n y la carga en Azure AI Search:

```bash
cd scripts
dotnet run --project IndexDocuments -- \
  --source ./docs \
  --index <nombre-del-indice>
```

El corpus documental es completamente configurable. El script acepta cualquier colecciÃ³n de documentos compatible sin requerir cambios en el pipeline de gobernanza ni en el backend.

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

## EjecuciÃ³n en Desarrollo

### Levantar el backend

Desde el directorio `backend/`:

```bash
dotnet run --environment Development
```

El backend quedarÃ¡ disponible en `https://localhost:7000`. El estado del sistema puede verificarse en `https://localhost:7000/api/v1/health`.

### Levantar el frontend

Desde el directorio `frontend/`, en un terminal separado:

```bash
npm run dev
```

La aplicaciÃ³n estarÃ¡ disponible en `http://localhost:5173`.

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

Actualiza la variable `VITE_API_BASE_URL` en la configuraciÃ³n de la Static Web App para apuntar a la URL de producciÃ³n del backend.

---

## Estructura de Carpetas

```
subtextai/
â”‚
â”œâ”€â”€ backend/                          # API REST en ASP.NET Core
â”‚   â”œâ”€â”€ Controllers/                  # Endpoints: Analizar, Evaluar, Audit, Metricas, Prompts, Health
â”‚   â”œâ”€â”€ Pipeline/                     # LÃ³gica del pipeline de gobernanza
â”‚   â”‚   â”œâ”€â”€ PolicyEngine.cs           # Validaciones de polÃ­tica sin LLM (longitud, idioma, etc.)
â”‚   â”‚   â”œâ”€â”€ CrisisClassifier.cs       # Clasificador de crisis emocional (LLM call #1)
â”‚   â”‚   â”œâ”€â”€ RagService.cs             # RecuperaciÃ³n documental con Azure AI Search
â”‚   â”‚   â”œâ”€â”€ LlmService.cs             # AnÃ¡lisis pragmÃ¡tico principal (LLM call #2)
â”‚   â”‚   â””â”€â”€ TraceStore.cs             # Persistencia de trazas con trace_id
â”‚   â”œâ”€â”€ Models/                       # DTOs de request y response
â”‚   â”œâ”€â”€ Prompts/                      # Prompts versionados del sistema
â”‚   â”‚   â”œâ”€â”€ CrisisClassifier-v1.0.txt # Prompt especializado de clasificaciÃ³n de crisis
â”‚   â”‚   â””â”€â”€ AnalisisPragmatico-v1.2.txt # Prompt principal del sistema
â”‚   â”œâ”€â”€ Migrations/                   # Migraciones de base de datos (EF Core)
â”‚   â”œâ”€â”€ appsettings.json              # ConfiguraciÃ³n base
â”‚   â””â”€â”€ appsettings.Example.json      # Plantilla de configuraciÃ³n (sin secretos)
â”‚
â”œâ”€â”€ frontend/                         # AplicaciÃ³n React
â”‚   â”œâ”€â”€ src/
â”‚   â”‚   â”œâ”€â”€ components/               # Componentes UI reutilizables
â”‚   â”‚   â”œâ”€â”€ pages/                    # Vistas principales (AnÃ¡lisis, AuditorÃ­a, MÃ©tricas)
â”‚   â”‚   â”œâ”€â”€ services/                 # Capa de integraciÃ³n con la API REST
â”‚   â”‚   â””â”€â”€ types/                    # Tipos TypeScript para respuestas del backend
â”‚   â”œâ”€â”€ .env.example                  # Plantilla de variables de entorno
â”‚   â””â”€â”€ vite.config.ts
â”‚
â”œâ”€â”€ scripts/                          # Utilidades de indexaciÃ³n y evaluaciÃ³n
â”‚   â”œâ”€â”€ IndexDocuments/               # Script de carga de documentos en Azure AI Search
â”‚   â”‚   â””â”€â”€ docs/                     # Directorio de documentos fuente (configurable)
â”‚   â””â”€â”€ EvaluationJob/                # Preguntas de prueba para evaluaciÃ³n automatizada
â”‚
â””â”€â”€ docs/                             # DocumentaciÃ³n del proyecto
```

---

## Endpoints de la API

**Base URL:** `https://subtextai-api.azurewebsites.net/api/v1`

| MÃ©todo | Endpoint | DescripciÃ³n |
|--------|----------|-------------|
| `POST` | `/analizar` | Interpreta el significado pragmÃ¡tico de un mensaje ambiguo |
| `POST` | `/evaluar` | EvalÃºa una respuesta propuesta por el usuario ante un mensaje analizado |
| `GET` | `/audit/{trace_id}` | Devuelve la trazabilidad completa y versionada de una respuesta |
| `GET` | `/metricas` | MÃ©tricas de calidad agregadas del sistema (Ãºltimos 7 dÃ­as) |
| `GET` | `/prompts` | Historial de versiones de prompts con mÃ©tricas comparativas |
| `GET` | `/health` | Estado del sistema y de todos los servicios Azure conectados |

### POST /analizar

Recibe un mensaje y un contexto, lo pasa por el pipeline completo de gobernanza y devuelve el anÃ¡lisis pragmÃ¡tico fundamentado en fuentes documentales.

**Request:**
```json
{
  "mensaje": "Solo quiero fluir y ver quÃ© pasa",
  "contexto": "pareja | trabajo | social | negociacion"
}
```

**Response 200:**
```json
{
  "significado": "Baja implicaciÃ³n emocional, evasiÃ³n de compromiso explÃ­cito",
  "senales": ["evasiÃ³n", "ambigÃ¼edad intencional", "pasividad"],
  "nivel_alerta": "MEDIO",
  "recomendacion": "Mantener lÃ­mites claros. No invertir energÃ­a emocional sin reciprocidad.",
  "fuente": {
    "documento": "<tÃ­tulo del documento fuente>",
    "fragmento": "<secciÃ³n o capÃ­tulo relevante>"
  },
  "confianza": {
    "nivel": "ALTA",
    "razon": "score_rag_medio: 0.87 â€” contexto documental sÃ³lido"
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

**CÃ³digos de error posibles:**

| CÃ³digo | PolÃ­tica | DescripciÃ³n |
|--------|----------|-------------|
| `422` | `mensaje_minimo` | Mensaje con menos de 5 palabras |
| `422` | `grounding_obligatorio` | Score RAG medio < 0.40; sin base documental suficiente |
| `422` | `crisis_detected` | SeÃ±ales de crisis emocional detectadas; se deriva a profesional |
| `422` | `idioma_no_soportado` | Idioma distinto al espaÃ±ol |
| `400` | `prompt_injection` | Input malicioso detectado y bloqueado |
| `429` | `rate_limit_usuario` | MÃ¡s de 10 requests/minuto por usuario/IP |

### POST /evaluar

EvalÃºa la respuesta propuesta por el usuario ante un mensaje ya analizado. Devuelve una valoraciÃ³n de la respuesta con probabilidad de Ã©xito, fortalezas, Ã¡reas de mejora y sugerencia alternativa, todo ello grounded en fuentes documentales.

### GET /audit/{trace_id}

Devuelve el registro completo de una interacciÃ³n identificada por su `trace_id`. Incluye los documentos recuperados con sus scores de relevancia individuales, el prompt exacto enviado al modelo en su versiÃ³n especÃ­fica, el modelo utilizado, las polÃ­ticas evaluadas y la latencia real.

### GET /metricas

Devuelve mÃ©tricas agregadas del sistema correspondientes a los Ãºltimos 7 dÃ­as: porcentaje de respuestas grounded, distribuciÃ³n de niveles de confianza, polÃ­ticas activadas por tipo, latencia media, cache hit rate y versiones de prompt activas.

### GET /prompts

Devuelve el historial de versiones de prompts del sistema con fecha de activaciÃ³n, descripciÃ³n de cambios y mÃ©tricas de rendimiento asociadas a cada versiÃ³n. Permite comparar el comportamiento del sistema antes y despuÃ©s de cada modificaciÃ³n.

### GET /health

Devuelve el estado del sistema y de cada servicio Azure conectado (Azure OpenAI, Azure AI Search, base de datos, Azure Monitor), junto con la versiÃ³n del sistema y el prompt activo.

---

## Trazabilidad y Observabilidad

SubtextAI implementa observabilidad en dos niveles complementarios.

### Trazabilidad por interacciÃ³n

Cada anÃ¡lisis genera un `trace_id` Ãºnico que permite reconstruir completamente la cadena de decisiones del agente. La auditorÃ­a de cualquier respuesta es accesible en tiempo real a travÃ©s del endpoint `GET /audit/{trace_id}`, que expone:

- Los fragmentos documentales que influyeron en la respuesta y su score de relevancia
- El prompt exacto enviado al modelo y su versiÃ³n
- La versiÃ³n del modelo invocado
- Si se activÃ³ alguna polÃ­tica y cuÃ¡l
- El nivel de confianza calculado objetivamente a partir del score RAG
- La latencia total del sistema
- Si el rate limit fue alcanzado por ese usuario

### Observabilidad del sistema

Azure Monitor recibe las mÃ©tricas agregadas del sistema generadas tanto por las interacciones en producciÃ³n como por el job de evaluaciÃ³n automatizada. Estas mÃ©tricas permiten monitorizar la evoluciÃ³n del comportamiento del agente a lo largo del tiempo, detectar desviaciones y comparar el rendimiento entre versiones de prompt de forma cuantitativa.

---

## Mejoras Futuras

El diseÃ±o del sistema establece un conjunto de evoluciones planificadas con condiciones de revisiÃ³n explÃ­citas documentadas durante el desarrollo.

**Clasificador de crisis emocional especializado** â€” cuando el volumen supere los 10.000 requests/dÃ­a, se evaluarÃ¡ la sustituciÃ³n del clasificador basado en prompt por un modelo fine-tuned (DistilBERT u equivalente) para reducir coste por token y mejorar precisiÃ³n. Esta alternativa fue descartada en el MVP por el tiempo de integraciÃ³n adicional, pero estÃ¡ documentada como evoluciÃ³n natural del sistema.

**Soporte multilingÃ¼e** â€” extensiÃ³n del sistema para soportar idiomas adicionales mÃ¡s allÃ¡ del espaÃ±ol, con prompts adaptados y pipelines de evaluaciÃ³n especÃ­ficos por idioma.

**ExpansiÃ³n del corpus documental** â€” incorporaciÃ³n de nuevas fuentes especializadas para ampliar la capacidad interpretativa del sistema sin modificar su lÃ³gica de gobernanza ni su pipeline de recuperaciÃ³n.

**Panel de auditorÃ­a** â€” interfaz de administraciÃ³n para visualizar trazas, comparar versiones de prompts y revisar eventos de activaciÃ³n de polÃ­ticas sin necesidad de consultar la base de datos directamente.

**Mejora de observabilidad** â€” ampliaciÃ³n de los indicadores registrados en Azure Monitor para detectar tendencias de uso, errores recurrentes y cambios de comportamiento del agente a lo largo del tiempo.

**Clasificador determinista adicional** â€” en escenarios que requieran auditorÃ­a regulatoria, se aÃ±adirÃ­a una segunda capa de clasificaciÃ³n basada en reglas deterministas sobre la clasificaciÃ³n LLM existente, reforzando el registro de auditorÃ­a y la auditabilidad del sistema.

**CachÃ© semÃ¡ntico** â€” mejora del sistema de cacheo actual para incorporar similitud semÃ¡ntica entre consultas, reduciendo llamadas al modelo para mensajes funcionalmente equivalentes y optimizando el coste operativo.

---

## Autoras

**Nombre:** Elizabeth SÃ¡enz Camacho y Heily Madelay Tandazo  
**MÃ¡ster:** Desarrollo Full Stack & Arquitecturas Cloud  
**InstituciÃ³n:** Tajamar  
**AÃ±o:** 2026

---

## Licencia

Este proyecto ha sido desarrollado como Trabajo de Fin de MÃ¡ster con fines acadÃ©micos.
Distribuido bajo la licencia MIT. Consulta el archivo [LICENSE](LICENSE) para más información.
