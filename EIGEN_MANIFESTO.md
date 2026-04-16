# EIGEN · Manifiesto Técnico

**Versión:** 1.0 · 15 abril 2026
**Dueño:** Raúl Zhou (fundador)
**Estatus:** vivo · se revisa trimestralmente
**Audiencia:** ingenieros, estudiantes del programa Semestre Tec, agentes de IA que construyen sobre el repo.

---

## § 0 · Propósito

Este documento es la doctrina técnica de Eigen. Lo lee un estudiante en su primer día, un cliente técnico durante due diligence, y un agente de IA cada vez que abre el repo (vía `CLAUDE.md` que lo carga).

No es una guía de estilo. Es la respuesta a tres preguntas:

1. **Qué construimos por default** — el stack elegido, no discutible hasta que cambie el documento.
2. **Cómo lo construimos** — el loop de trabajo con humanos y agentes de IA.
3. **Cuándo rompemos las reglas** — las excepciones explícitas, con dueño.

Si una decisión técnica no está aquí, consulta al fundador. Si está aquí y no la estás siguiendo, documenta la excepción en el PR.

---

## § 1 · Doctrina de arquitectura

### 1.1 Stack por default

| Capa | Elección | Por qué |
|---|---|---|
| **Compute** | Cloudflare Workers + Hono | Edge-native, sin cold starts, TypeScript estricto. |
| **Estado consistente por entidad** | Durable Objects (+ Facets para multi-tenant) | Strong consistency, co-location con cómputo. |
| **Datos relacionales locales** | D1 (SQLite edge) | Migraciones versionadas, lecturas globales baratas. |
| **Datos relacionales serios** | Supabase (Postgres) + Hyperdrive | Volumen transaccional, joins complejos, RLS. |
| **Blob storage** | R2 | Cero egress. URL signing para acceso privado. |
| **Cache global** | Workers Cache API + Cache Reserve | Hit ratio > 90% para contenido estático. |
| **KV** | Workers KV | Config, feature flags, sesiones eventualmente consistentes. |
| **Queue / async** | Cloudflare Queues + Workflows | At-least-once, DLQ, steps persistentes. |
| **Agentes LLM** | Claude Agent SDK + Cloudflare Agents SDK (Think) | Persistencia, skills, subagents, HITL nativo. |
| **Modelos** | Claude (default) → Workers AI (inferencia barata) | Tool use serio con Claude, clasificadores con Workers AI. |
| **Memoria semántica** | Vectorize | Edge-native, sin operación propia. |
| **Auth empleados / admin** | Cloudflare Access | SSO zero-trust sin Auth0. |
| **Auth clientes / API pública** | Workers OAuth Provider | OAuth 2.1 con PKCE, dynamic client reg. |
| **Pagos** | Stripe | Webhooks verificados + idempotencia obligatoria. |
| **Correo saliente** | Postmark (o equivalente) | Deliverability + inbox signals. |
| **Correo entrante** | Email Workers + Email Routing | Procesamiento programable. |
| **WhatsApp** | Cloud API directo (no ManyChat para nuevos proyectos) | Control total, menos fricción. |
| **Voz / WebRTC** | Workers Calls + Containers (Whisper) + ElevenLabs TTS | Latencia edge, costo controlado. |
| **Browser automation** | Browser Run (CDP + HITL) | Auth + handover humano cuando el agente se atora. |
| **Observabilidad** | Tail Workers + Analytics Engine + Sentry + PostHog | Diferenciado: errores humanos vs métricas vs UX. |
| **Analítica histórica** | R2 Data Catalog (Iceberg) + DuckDB | Consulta sin mover datos. |
| **Ingesta de eventos** | Cloudflare Pipelines | Reemplaza Kafka/Firehose. |
| **Networking privado** | Workers VPC + Mesh + Tunnel | Acceso a infra privada sin VPN. |
| **Seguridad perímetro** | WAF + Bot Management + Rate Limiting + Turnstile | Configurado, no "por defecto". |
| **Sandboxing de código IA** | Cloudflare Sandboxes + Outbound Workers | Egress zero-trust con credential injection. |
| **CI/CD** | GitHub Actions + CLI `cf` | Preview URLs por PR, gates con tests + type-check. |
| **Config y secretos** | `wrangler.toml` / `cf.toml` por repo | Secrets vía `cf secret`, nunca en `.env` commiteado. |

### 1.2 Principios arquitectónicos no-negociables

1. **TypeScript estricto end-to-end.** `strict: true`. Ningún `any` en límites públicos.
2. **Zod en todas las fronteras.** Request/response, webhooks, tool inputs, DB boundaries.
3. **Idempotencia por default en consumers.** At-least-once es un hecho de la vida.
4. **Correlation IDs propagados.** Desde el Worker de entrada hasta el último log.
5. **Tests antes de prod.** No merge a `main` sin tests pasando + type-check limpio.
6. **Observabilidad al día 1.** Un servicio sin métricas + alertas no está "done".
7. **Costo medible por cliente.** Analytics Engine con `customerId` en cada data point.

### 1.3 Excepciones al stack (cuándo salir de Cloudflare)

Escape hatches explícitos. Cada uno requiere PR al manifesto con justificación y aprobación del fundador:

| Caso | Destino correcto | Razón |
|---|---|---|
| Modelos custom / fine-tuning | Modal, Replicate, Hugging Face | CF no corre entrenamiento. |
| Video editing / compositing pesado | fly.io (GPU) | CF Stream es delivery, no edición. |
| Postgres > 1 TB, alta carga transaccional | Supabase dedicated o RDS | D1 no aplica. |
| Cliente exige AWS/Azure/GCP con contrato | caso por caso | Negocio decide. |
| Compliance que exige residencia específica no soportada | caso por caso | Legal manda. |

**Regla meta:** "Si tengo que salir de Cloudflare, tiene que haber una razón escrita, no una corazonada."

---

## § 2 · Doctrina de construcción

### 2.1 Cómo escribimos código

- **PRs chicos.** < 300 líneas de delta. Si algo es más grande, se parte en varios PRs secuenciales.
- **Un archivo, una responsabilidad.** > 500 líneas = señal de refactor. > 1000 líneas = bug de diseño.
- **Tests adyacentes al código.** `foo.ts` + `foo.test.ts` hermanos.
- **Commits convencionales.** `feat:`, `fix:`, `refactor:`, `docs:`, `chore:`, `test:`.
- **No comentamos lo obvio.** Los comentarios explican *por qué*, no *qué*. El código explica el *qué*.
- **No hay código muerto.** Si está comentado o `// TODO` sin dueño + deadline, se borra.

### 2.2 Cómo trabajamos CON IA

Eigen es un shop que usa Claude Code / Cursor / Copilot agresivamente. El manifesto no lo pelea — lo canaliza.

**El loop obligatorio (spec-first):**

1. **Brainstorming** — entender intent, requirements, trade-offs. Cualquier feature no trivial pasa por aquí, no se empieza a codear.
2. **Plan escrito** — un `.md` que describe el cambio: qué, dónde, cómo, tests, rollback. Revisado por humano antes de ejecutar.
3. **Implementación** — asistida por IA, pero con el plan en mano.
4. **Verificación** — tests, manual check, correlation IDs en logs, métricas en dashboard.
5. **Review humano** — otro par de ojos, siempre. Especialmente si la IA escribió el 80%.

**Qué aprueba siempre un humano (nunca la IA sola):**
- Arquitectura y elecciones de stack.
- Invariantes de negocio no escritas.
- Cambios de prompt productivo (requieren evals).
- Irreversibles (DB migrations destructivas, destrucción de datos, cambios de auth).
- Anything que toque dinero de clientes.

**Qué puede proponer la IA (y normalmente acepta el humano con review rápido):**
- Código de implementación que sigue un plan existente.
- Tests unitarios.
- Documentación técnica (ADRs, runbooks, postmortems).
- Refactors locales con tests que demuestran equivalencia.
- Debugging con context engineering (logs, stack traces, repo).

**Red flags al aceptar código de IA:**

| Síntoma | Interpretación |
|---|---|
| Tests pasan pero no validan invariante real | Silent failure — rechazar, pedir mejor test. |
| Try/catch que traga errores | Tapa un bug — pedir handling explícito o propagación. |
| Abstracciones preventivas para "flexibilidad futura" | Over-engineering — simplificar. |
| Cambios fuera del scope del plan | Scope creep — separar en otro PR o rechazar. |
| Imports o deps nuevas no justificadas | Dependencia fantasma — cuestionar. |
| Renombramientos de variables públicas "por claridad" | Puede romper callers — rechazar salvo que sea el objetivo. |

### 2.3 Prompt engineering en producción

Prompts productivos son **código**. Se versionan, se revisan, se testean, se cachean.

- **Viven en el repo.** `/prompts/*.ts` con template + Zod schema del input.
- **Cached.** Prompt caching de Anthropic activado por default (5 min para prompts frecuentes, 1 h para system prompts grandes).
- **Evaluados.** Cada prompt crítico tiene eval suite. Cambio sin evals actualizadas = rechazado en review.
- **Versionados.** Si cambias un prompt productivo, se registra la versión en Analytics Engine junto con el request (para correlar cambios con métricas).
- **Sanitizados.** Nunca concatenar input de usuario directamente. Siempre template + validation.
- **Pequeños y compuestos.** Prompts monolíticos de 3000 tokens son un anti-patrón. Mejor: prompt base + skills cargados on-demand + tool use explícito.

### 2.4 Context engineering para el repo

El repo es un entorno. Los agentes que trabajan en él deben tener todo el contexto que necesitan sin preguntar.

**Cada repo de Eigen contiene:**

- `README.md` — qué hace, cómo correr, cómo testear. 5 minutos de lectura.
- `CLAUDE.md` — contexto para agentes: stack, convenciones del repo, skills a cargar, cuándo preguntar. Carga por referencia al manifesto global.
- `/docs/adr/` — Architecture Decision Records. Formato Michael Nygard.
- `/docs/postmortems/` — incidentes pasados, blameless. Cada uno con timeline, root cause, action items.
- `/docs/runbooks/` — cómo responder los 5 escenarios más comunes de producción.
- `/.claude/skills/` — skills específicas del repo cuando aplique.

**Qué NO poner en CLAUDE.md:**

- Lo que ya está en el manifesto maestro (se carga por referencia).
- Convenciones genéricas de TypeScript/testing (ya están en el manifesto).
- Secretos, URLs privadas, datos de cliente.
- Decisiones ephemeras ("in-progress work"). Para eso existen tareas/plan docs.

**Qué SÍ debe tener un CLAUDE.md de repo:**

- Una línea de "este repo hace X para el cliente Y".
- Desviaciones del stack default (si las hay, con razón).
- Comandos críticos: `bun test`, `bun run deploy:staging`, etc.
- Skills a cargar: `["cloudflare-workers-patterns", "eigen-testing"]`.
- Reglas específicas del dominio: "nunca tocar `/api/billing` sin dos approvers".

### 2.5 Memoria persistente vs contexto efímero

- **Memoria** = lo que necesitamos en futuras sesiones. Vive en `/docs/adr/`, `/docs/postmortems/`, `skills/`. Commiteada.
- **Contexto efímero** = estado de la conversación actual. Vive en planes temporales, TodoList, scratchpads. NO se commitea como si fuera memoria.
- **Anti-patrón:** commitear notas de "in progress work" como si fueran documentación. Pudre el repo. Los agentes las leen como verdad y alucinan basados en ellas.

---

## § 3 · Doctrina de delivery

### 3.1 Environments

- **local** — `cf dev` con Miniflare. Todo el stack simulado (KV, D1, DO, R2).
- **dev** — preview URLs por PR automáticas vía GitHub Actions.
- **staging** — branch `staging`. Se promueve desde `main` cuando pasa QA.
- **production** — branch `main` tagged. Deploy solo con aprobación + checks verdes.

### 3.2 Promoción

- `feature-branch` → PR → tests + type-check + preview URL + review humano.
- Merge a `main` → deploy a staging automático.
- Tag de versión → deploy a production con approval manual en GitHub.
- Nunca push directo a production. Nunca `git push --force` a `main`.

### 3.3 Rollouts

- **Cambios triviales** (typo, copy): deploy directo.
- **Cambios de lógica**: gradual rollout 10% → 50% → 100% con 24 h entre etapas.
- **Cambios de prompt productivo**: shadow run (10% compara nueva vs vieja offline) → 25% → 100%.
- **Cambios de schema DB**: migración backward-compatible primero, deploy código, luego cleanup.
- **Rollback**: instantáneo vía Versioned Deploys. Si no puedes rollback en < 2 minutos, no estás listo para deploy.

### 3.4 Observabilidad mínima por servicio

Un servicio no puede ir a producción sin:

- **Correlation ID** propagado en todos los logs y downstream calls.
- **Métricas**: latency p50/p95/p99, error rate, throughput, cost per request.
- **Alertas de 3 niveles**: info (Analytics Engine), warn (Slack canal general), critical (Slack canal on-call + page).
- **Runbook**: enlace desde la alerta a un runbook que describe qué hacer en los 3 escenarios más probables.
- **Dashboard**: una URL que cualquiera del equipo puede abrir y ver salud del servicio.

### 3.5 Compliance baseline

Todos los servicios de Eigen deben cumplir, mínimo:

- **PII redactada** en logs salientes (Logpush con PII redaction rules).
- **Encryption at rest** en R2 (server-side).
- **Audit trails** en R2 con Object Lock cuando se toca dinero o datos médicos.
- **Secrets rotados** al menos trimestralmente, registro de rotación en ADR.
- **NDA firmada** con clientes antes de tocar datos reales.
- **BAA** si aplica (clientes médicos US → HIPAA).

---

## § 4 · Doctrina de operación

### 4.1 On-call e incidentes

- **On-call rotation** cuando haya >= 3 ingenieros. Hasta entonces, el fundador.
- **Cada alerta crítica** tiene runbook enlazado. Sin runbook = bug del servicio.
- **Postmortem blameless** obligatorio para P0 y P1. Formato: timeline, root cause, impact, action items, lessons. Publicado en `/docs/postmortems/`.
- **Agent Lee configurado** para triage de primera línea (investigación, correlación, sugerencia de mitigación). Humano sigue decidiendo acción destructiva.

### 4.2 Costo

- **Budget cap mensual** por servicio, alerta al 80%.
- **Cost attribution** por cliente en Analytics Engine (label `customerId` en cada data point de costo).
- **Review mensual** de bill de Cloudflare + Anthropic + Supabase + Stripe fees. Línea por servicio + línea por cliente. Si algo creció > 50% sin razón, investigar antes del siguiente mes.
- **Prompt caching hit rate >= 70%** para agentes productivos. Métrica medida en AI Gateway.

### 4.3 Seguridad operacional

- **Secrets rotation calendar** mantenido. Stripe keys, Anthropic API key, Google OAuth, Supabase service role key — todos con fecha de próxima rotación.
- **Access reviews** trimestrales: quién tiene acceso a qué, revocar lo inactivo.
- **Dependency audits** mensuales: `bun outdated` + CVE scan. Parchar lo crítico en < 72 h.
- **Security review** obligatorio antes de exponer cualquier endpoint público.

---

## § 5 · Doctrina de gente

### 5.1 Onboarding

- **Día 1**: cuenta de GitHub, acceso al monorepo, leer manifesto completo, abrir un PR trivial (fix de typo, mejora de doc). Objetivo: cerrar el loop PR → review → merge → deploy a staging.
- **Semana 1**: un issue "good first issue" cerrado completamente, con tests y review aprobado.
- **Semana 2-4**: shadow de operación real — llamadas con cliente, review de un postmortem, on-call pasivo con mentor.
- **Mes 2+**: ownership de un servicio o un vertical de cliente.

### 5.2 Cómo escalamos decisiones

- **Default**: tu criterio. Toma la decisión, documenta en el PR.
- **Cuando escalar**: irreversible, > $500 MXN de costo adicional, afecta a cliente en producción, cruza un límite del manifesto, o simplemente no estás seguro. Mejor escalar.
- **Cómo escalar**: async, por escrito, en el canal correcto (Slack `#decisiones` o PR). Con opciones ("veo A, B, C — recomiendo B por X, Y") no abierto ("qué hago?").

### 5.3 Feedback

- **Semanal 1:1** con mentor (estudiantes) o peer (ingenieros).
- **Retro mensual** del equipo: qué funcionó, qué no, qué cambiamos.
- **Review bi-anual** con fundador: trayectoria, siguiente paso.

### 5.4 Respeto por el craft

- No se permite "ship and pray". Si no estás seguro, no despliegues.
- No se permite "el test pasa, so what". Los tests son medio, no fin.
- No se permite culpar a la IA. Si la IA escribió el bug y lo mergeaste, es tu bug.
- No se permite heroísmo. Las noches épicas de deploy no son un badge; son un failure de proceso.

---

## § 6 · Meta · cómo se mantiene este documento

- **Cualquier ingeniero puede proponer un cambio** vía PR al repo `eigen/manifesto`.
- **Aprobación**: fundador (por ahora; eventualmente comité técnico).
- **Review cadence**: trimestralmente, o cuando un cambio significativo lo justifique (nuevo servicio, incidente mayor, cambio de stack).
- **Changelog**: al final de este doc, cada cambio con fecha, autor, razón.
- **Si encuentras algo aquí que contradice la realidad**, el doc está mal o la realidad está mal. Cualquiera es bug. Abre PR o issue.

---

## § 7 · Changelog

| Versión | Fecha | Autor | Cambios |
|---|---|---|---|
| 1.0 | 2026-04-15 | Raúl Zhou | Documento inicial. Stack Cloudflare-first post Agents Week. |

---

*"La doctrina existe para que las decisiones del día-a-día no se tomen de cero. Cuando la doctrina está mal, se cambia la doctrina — no se ignora. Un manifesto que no se aplica es basura decorativa."*

— Eigen · 2026
