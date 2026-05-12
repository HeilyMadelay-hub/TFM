# Orden de creación de ramas — SubtextAI

---

## Ramas base (siempre primero)

Antes de cualquier otra cosa, crea las dos ramas principales del repositorio:

- `main` — producción, no se toca directamente
- `develop` — base de integración desde la que nacen todas las demás

---

## Fase 1 — Cimientos

Las tres se pueden crear y trabajar en paralelo. Sin ellas no hay dónde desplegar ni dónde persistir datos.

1. `infra/azure-setup` — provisiona los recursos Azure (App Service, Static Web Apps, Azure OpenAI, Azure AI Search, base de datos)
2. `infra/database-schema` — diseña las tablas, migraciones EF Core e índices para trazas, métricas y prompts
3. `infra/cicd` — configura el pipeline de GitHub Actions o Azure DevOps para build, tests y despliegue automático

---

## Fase 2 — Núcleo del backend

Solo cuando la infraestructura y el esquema de base de datos estén definidos.

4. `feature/backend-api` — endpoints REST: `/analizar`, `/evaluar`, `/audit/{trace_id}`, `/metricas`, `/prompts`, `/health`
5. `feature/governance-pipeline` — pipeline completo de gobernanza con PolicyEngine, RagService, LlmService y TraceStore (depende de que backend-api tenga su estructura básica)

---

## Fase 3 — Componentes IA

Solo cuando el pipeline de gobernanza esté en pie. Primero el indexado, luego los tres en paralelo.

6. `infra/indexing-pipeline` — chunking, vectorización e indexación de los libros de Gottman y Johnson en Azure AI Search
7. `feature/crisis-classifier` — clasificador de crisis emocional con prompt especializado y registro de activaciones (paralelo con 8 y 9)
8. `feature/prompt-versioning` — sistema de versionado de prompts con historial y comparativa de métricas (paralelo con 7 y 9)
9. `feature/sentiment-analysis` — análisis de sentimiento del mensaje de entrada, campo `sentimiento` en el response (paralelo con 7 y 8)

---

## Fase 4 — Seguridad y configuración

Con el sistema funcionando y los componentes IA integrados. Las cuatro en paralelo.

10. `feature/security-hardening` — sanitización, validación de payloads, protección contra prompt injection, headers HTTP y CORS
11. `feature/config-management` — feature flags, thresholds del RAG, settings por entorno y Azure Key Vault
12. `feature/auth-rate-limit` — identificación de usuarios/API keys, rate limiting por usuario e IP, respuesta HTTP 429
13. `feature/structured-logging` — logging estructurado con correlación por `trace_id`, niveles INFO/WARN/ERROR y exportación a Azure Monitor

---

## Fase 5 — Frontend

Una vez que la API es estable, el contrato está definido y la seguridad está en su sitio.

14. `feature/frontend-ui` — interfaz React en Azure Static Web Apps con formulario de análisis, visualización de resultados, fuente citada, nivel de confianza y `trace_id`

---

## Fase 6 — Optimización

Solo cuando el sistema completo funciona y sabes qué es lento y qué cuesta tokens. Las dos en paralelo.

15. `feature/cache` — caché para reducir latencia y llamadas al modelo, campo `from_cache` en el response
16. `feature/cost-optimization` — estrategias de reducción de tokens, selección de modelo por tipo de request y control de gasto en Azure OpenAI

---

## Fase 7 — Calidad y monitoring

Sistema optimizado y estable. El dataset de evaluación y el evaluador van juntos; el resto en paralelo.

17. `test/evaluation-dataset` — conjunto de casos de prueba para la evaluación automatizada periódica (crear junto con 18)
18. `feature/llm-evaluation` — lógica del evaluador automático: grounding score, consistencia semántica, comparación entre versiones (crear junto con 17)
19. `test/backend-unit` — tests unitarios de PolicyEngine, RagService, CrisisClassifier y TraceStore (paralelo con 20 y 21)
20. `test/api-integration` — tests de integración sobre todos los endpoints REST (paralelo con 19 y 21)
21. `infra/monitoring` — configuración de Azure Monitor, alertas y job de evaluación en Azure Functions (paralelo con 19 y 20)

---

## Fase 8 — Documentación final

Cuando el sistema esté validado. Se documenta el estado real, no el planificado.

22. `docs/api-contract` — contrato de API v2.0 con ejemplos, tabla de políticas y niveles de confianza del RAG
23. `docs/tfm` — memoria técnica del TFM con decisiones de arquitectura, ADR y métricas reales

---

## Resumen de dependencias clave

- `feature/governance-pipeline` necesita que `feature/backend-api` tenga estructura básica
- `infra/indexing-pipeline` debe ir antes que `feature/crisis-classifier`, `feature/prompt-versioning` y `feature/sentiment-analysis`
- `feature/frontend-ui` necesita que `feature/backend-api` y `feature/security-hardening` estén integrados
- `feature/llm-evaluation` y `test/evaluation-dataset` se crean y trabajan juntas
- `feature/cache` y `feature/cost-optimization` solo tienen sentido sobre un sistema que ya funciona
- `docs/tfm` y `docs/api-contract` se escriben siempre al final, sobre el estado real del sistema
