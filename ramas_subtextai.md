# SubtextAI — Estructura de Ramas del Repositorio

---

## 1. Ramas principales

### `main`
Código en producción. Solo recibe merges desde `develop` tras validación completa del pipeline de CI/CD y revisión manual. Nunca se trabaja directamente sobre esta rama.

### `develop`
Rama de integración continua. Todas las features se integran aquí antes de subir a `main`. Es la rama base desde la que se crean todas las ramas de trabajo y contra la que se abren los pull requests.

---

## 2. Features

### `feature/frontend-ui`
Interfaz React desplegada en Azure Static Web Apps. Contiene el formulario de análisis, la visualización de resultados, la fuente citada, el nivel de confianza y el `trace_id` visible al usuario. Se justifica como rama independiente porque el frontend tiene su propio ciclo de despliegue, sus propias dependencias y puede desarrollarse en paralelo al backend sin bloquearlo.

### `feature/backend-api`
Endpoints REST en .NET: `/analizar`, `/evaluar`, `/audit/{trace_id}`, `/metricas`, `/prompts` y `/health`. Es el punto de entrada de todas las peticiones al sistema y la capa que orquesta el pipeline completo. Se mantiene separada del pipeline de gobernanza porque la API puede evolucionar su contrato sin tocar la lógica interna del agente.

### `feature/governance-pipeline`
Pipeline completo de gobernanza con cortocircuito: validaciones de política, clasificador de crisis, RAG, LLM principal y trazabilidad. Incluye `PolicyEngine`, `RagService`, `LlmService` y `TraceStore`. Es la rama más crítica del sistema porque implementa el núcleo de la arquitectura de agente gobernado. Tiene rama propia porque cualquier cambio aquí afecta al comportamiento observable del sistema y debe ser revisado con más cuidado que una feature de UI.

### `feature/crisis-classifier`
Clasificador de crisis emocional mediante llamada previa al LLM con prompt especializado. Incluye la lógica de decisión, el prompt versionado `CrisisClassifier v1.0` y el registro de activaciones con `trace_id`. Se separa del pipeline general porque es un componente con prompt propio, métricas propias y una responsabilidad ética específica que merece revisión independiente.

### `feature/prompt-versioning`
Sistema de versionado de prompts, historial de cambios y comparativa de métricas entre versiones. Cada prompt del sistema se identifica con versión incremental (`v1.0`, `v1.1`, `v1.2`, `v2.0`) y se registra junto con métricas de comportamiento. Se justifica como rama propia porque el versionado de prompts es un mecanismo de gobernanza transversal que afecta a todos los componentes que usan el LLM, y sus cambios deben ser rastreables de forma independiente.

### `feature/cache`
Sistema de caché para optimizar latencia y reducir llamadas al modelo. El campo `from_cache` en el response de `/analizar` confirma que esta funcionalidad está en el contrato público de la API. Se mantiene en rama separada porque la estrategia de caché puede cambiar sin tocar la lógica de gobernanza ni el contrato de API.

### `feature/sentiment-analysis`
Análisis de sentimiento del mensaje de entrada. El campo `sentimiento` con `label` y `score` aparece explícitamente en el response de `/analizar` dentro de `metadata`. Se separa porque es un componente con lógica propia que no bloquea el pipeline principal ni forma parte de las políticas de gobernanza. Puede desarrollarse e integrarse de forma independiente sin riesgo para el flujo crítico.

### `feature/security-hardening`
Sanitización de inputs, validación avanzada de payloads, protección contra prompt injection, headers de seguridad HTTP, configuración de CORS y manejo seguro de secrets. Se justifica porque el sistema ya documenta la política `prompt_injection` con respuesta HTTP 400, pero esa lógica técnica no tiene rama propia. Sin esta rama, la seguridad existe en el papel pero no en el código de forma rastreable ni auditable.

### `feature/structured-logging`
Logging estructurado con correlación por `trace_id`, niveles `INFO/WARN/ERROR`, logs auditables y exportación a Azure Monitor y Application Insights. El sistema entero gira alrededor de la trazabilidad y cada interacción genera un `trace_id` único. Sin una rama dedicada al logging estructurado, esa trazabilidad queda dispersa entre componentes sin una estrategia coherente. Es una incoherencia arquitectónica tener trazabilidad como pilar del sistema y no tener una rama que la implemente de forma centralizada.

### `feature/config-management`
Manejo centralizado de configuración: feature flags, parámetros de políticas, thresholds del RAG (0.40 y 0.75), settings por entorno y Azure Key Vault para secrets. Se justifica porque el documento describe umbrales concretos que ahora mismo están hardcodeados en la documentación. En un sistema real esos valores deben ser configurables por entorno sin recompilar. Esta rama da coherencia entre lo que se documenta y cómo se gestiona la configuración en el código.

### `feature/auth-rate-limit`
Identificación de usuarios o API keys, rate limiting por usuario e IP y control básico de acceso. El contrato de API ya define explícitamente la respuesta HTTP 429 con `retry_after_segundos` y `trace_id`. La política `rate_limit_usuario` existe en la tabla de políticas del sistema. Sin esta rama, esa lógica no tiene un lugar claro en el repositorio y su implementación queda sin contexto propio.

### `feature/llm-evaluation`
Evaluación automática de calidad de respuestas: grounding score, consistencia semántica y comparación de comportamiento entre versiones de prompt y modelos. Se diferencia de `test/evaluation-dataset` en que el dataset contiene los casos de prueba, mientras que esta rama contiene la lógica que los ejecuta, mide y compara. El sistema ya describe un job en Azure Functions que evalúa periódicamente. Esta rama es la implementación de ese evaluador, no los datos que usa.

### `feature/cost-optimization`
Optimización de tokens, estrategias de caché avanzadas, batching de requests y control de coste operacional del pipeline. Se diferencia de `feature/cache` en que la caché es una estrategia concreta de reducción de latencia, mientras que esta rama abarca decisiones más amplias: qué modelo usar según el tipo de request, cómo reducir el tamaño de prompts sin perder calidad, cómo medir y controlar el gasto en Azure OpenAI. Es relevante para la defensa porque demuestra que el sistema no solo funciona sino que es operable de forma sostenible.

---

## 3. Infraestructura y cloud

### `infra/azure-setup`
Configuración de recursos Azure: App Service, Static Web Apps, Azure OpenAI, Azure AI Search y base de datos. Es la rama que provisiona la infraestructura base sobre la que corre todo el sistema. Se mantiene separada para que los cambios de infraestructura no se mezclen con cambios de código de aplicación.

### `infra/indexing-pipeline`
Script de chunking, vectorización e indexación de los libros de Gottman y Johnson en Azure AI Search. Se justifica como rama independiente porque el proceso de indexación tiene su propio ciclo de vida: puede necesitar re-ejecutarse cuando cambian los documentos fuente, cuando cambia la estrategia de chunking o cuando se añaden nuevas fuentes, sin que eso implique ningún cambio en el backend.

### `infra/monitoring`
Configuración de Azure Monitor: métricas del sistema, alertas y job de evaluación automatizada en Azure Functions. Se separa de `infra/azure-setup` porque el monitoring tiene configuración propia que evoluciona a medida que se identifican nuevas métricas o umbrales de alerta, y no debería mezclarse con el aprovisionamiento inicial de recursos.

### `infra/database-schema`
Diseño de tablas, migraciones de EF Core, índices y esquema de persistencia para trazas de auditoría, métricas y prompts versionados. El documento describe con detalle qué se guarda en base de datos por cada interacción, pero no existe ninguna rama que implemente ese esquema. Es un hueco real en la estructura actual: sin esta rama, la persistencia no tiene un lugar propio ni un historial de cambios rastreable.

### `infra/cicd`
Pipeline de integración y despliegue continuo con GitHub Actions o Azure DevOps: build automático, ejecución de tests y despliegue a App Service y Static Web Apps. El README solo describe comandos manuales de CLI. Un repositorio profesional necesita que cualquier push a `develop` o `main` desencadene el ciclo completo de validación y despliegue de forma automática y reproducible.

---

## 4. Testing y calidad

### `test/backend-unit`
Tests unitarios del pipeline de gobernanza y lógica de políticas. Cubre los componentes críticos: `PolicyEngine`, `RagService`, `CrisisClassifier` y `TraceStore`. Se mantiene en rama separada para que los tests puedan evolucionar sin bloquear el desarrollo de features.

### `test/api-integration`
Tests de integración sobre los endpoints REST con validación de respuestas, códigos de error y comportamiento de políticas. Cubre todos los casos de respuesta definidos en el contrato de API v2.0.

### `test/evaluation-dataset`
Conjunto de preguntas de prueba para la evaluación automatizada periódica del sistema. Contiene los casos de prueba que ejecuta el job de Azure Functions. Se diferencia de `feature/llm-evaluation` en que aquí están los datos, no la lógica que los procesa.

---

## 5. Documentación

### `docs/tfm`
Memoria técnica del TFM. Evoluciona en paralelo al desarrollo y refleja las decisiones de arquitectura, las ADR y los resultados de evaluación.

### `docs/api-contract`
Contrato de API v2.0 con ejemplos de request y response, tabla de políticas y niveles de confianza del RAG. Es la referencia pública del comportamiento del sistema.
