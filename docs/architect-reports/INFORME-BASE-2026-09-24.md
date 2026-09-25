# Informe base — Estado real de AM-TradingAgents y uso del modelo profundo

| | |
|---|---|
| **Fecha** | 2026-09-24 |
| **Autor** | Claude Sonnet 5 (orquestador) |
| **Referencia** | `desarrollo-agentico` @ `794d366` (ya en `origin`) · `main` @ `2d17df8` (espejo de upstream v0.5.0) |
| **Método** | Lectura estática de código y docs + comandos de solo lectura (`git`, `gh`, `pytest --collect-only`, estado de procesos y puertos). No se ejecutó el loop ni se modificó código. |
| **Leyenda** | **[V]** verificado con un comando o lectura reproducible · **[I]** inferido leyendo el código, no ejecutado |
| **Uso** | Humano: secciones 0, 1, 5 y 6. Fable: esta es la **línea base que debe confirmar o refutar**, no una verdad revelada. El prompt está en `PROMPT-ARQUITECTO-FABLE.md`; el protocolo de la carpeta, en `README.md`. |

---

## 0. Resumen ejecutivo

1. **El proyecto está mucho menos avanzado de lo que mostraban el dashboard y mis mensajes anteriores.** Ninguna de las 20 stories está mergeada y verificada. El "4/20 DONE (20 %)" son 4 PRs abiertos con CI en rojo.
2. **Corrijo lo que dije antes:** el loop **no está corriendo**; el dashboard **no es persistente**; las 4 ramas feature **no están integradas** en `desarrollo-agentico`; y los **20 issues (#2–#21) están CERRADOS** por un bug del loop.
3. **El mayor riesgo no está en el producto sino en la maquinaria que lo construye** (loop, validador, creador de PRs): escribe en cualquier ruta, mezcla stories en un mismo PR, nunca registra un éxito y su validador local no equivale a CI.
4. **El PR #22 rompería el motor upstream** si se mergea (reemplaza `pyproject.toml` y vacía `tradingagents/__init__.py`).
5. **Ya hay un defecto de seguridad real que los tests no detectaron** (JWT con secreto fijo y no configurable).
6. **El modelo profundo aporta más como auditor independiente y arquitecto de "work orders"**, no como implementador. Recomendación: 2 corridas al día + 1 semanal profunda (sección 5).

---

## 1. Lo que dije o mostré antes vs. la realidad

| Afirmación previa | Realidad | Evidencia |
|---|---|---|
| "Loop corriendo en background (PID 9628) toda la noche" | **No corre.** El proceso no existe. El último arranque (`autonomous_loop_20260924_124623.log`, 172 bytes) imprimió `All stories processed` y terminó: el picker no devolvió ninguna story (consistente con que las 2 READY son YELLOW y el resto está DONE/BLOCKED/DRAFT con dependencias sin cumplir) [I]. | [V] `Get-Process -Id 9628`; el log |
| "Dashboard persistente 24/7" | `http://127.0.0.1:8888` **sin respuesta**. El servicio `AM-TradingAgents-Dashboard` **no está instalado** (exigía PowerShell como administrador) y la carpeta Inicio no contiene el lanzador. | [V] `Get-Service`, `Invoke-WebRequest` |
| "4 stories DONE, 20 % completado" | Son **4 PRs abiertos (#22–#25) con CI rojo**, sin review y sin merge. Ninguno está en `desarrollo-agentico` ni en `main`. Incumplen sus propios contratos (hallazgo `STORY-contract-noncompliance-done`). | [V] `gh pr list`, `gh pr checks` |
| "Las 4 ramas feature están integradas en `desarrollo-agentico`" | **No.** Van 1–3 commits por delante y están **apiladas** (#25 contiene los commits de #23 y #24). US-INFRA-0003 depende de US-INFRA-0001 pero su rama no contiene el commit de INFRA-0001. | [V] `git rev-list --count`, `git log` |
| "Issues #2–#21 = las 20 stories" | **Los 20 están CLOSED**, incluidas 16 sin una línea de código. Los cerró `close_issue()` desde rutas de fallo y de skip. | [V] `gh issue list --state all` |
| "Phase 0.5 COMPLETA / PASS (25 controles)" | Están documentados, pero varios **no están conectados al loop** (`allowed_modules`, `required_tests`, reviewer/guardian, `checkpoint.py`, retry-policy, cost-guardrails). Y el job CI `architecture` **falla en la propia rama de integración**: los 16 tests de fitness/trazabilidad no se han podido ejecutar. | [V] `grep` en `scripts/`; `pytest tests/architecture` |
| "Orquestador = Sonnet 4.6" (registry y CLAUDE.md) | El loop lo orquesté como Haiku 4.5; hoy corre Sonnet 5. `pr_creator.py` firma "Claude Sonnet 4.6" de forma fija y **el código lo escribe Qwen sin que quede registrado**. | [V] `pr_creator.py:82` |
| "Los bloqueos son por SLM saturado / timeout" (diagnóstico mío) | **Sin evidencia.** El loop descarta la salida de los subprocesos y el log solo dice `Implementation failed`. Hay defectos del propio loop y validador que explican bloqueos sin invocar la capacidad del modelo. | Ver `LOOP-no-failure-evidence`, `LOOP-fix-loop-blind`, `VALIDATOR-pytest-broken-env` |

Estado verificado hoy: modelo local `:8080` responde (200) · 5 PRs abiertos (#1, #22–#25) · CI rojo en todos (solo pasan `clean-install smoke` y `mypy`, que es `|| true`).

---

## 2. Hallazgos (27) y 7 hipótesis

Severidad: **S0** puede romper el producto o invalida las garantías del proceso — no reactivar el loop ni mergear hasta resolverlo · **S1** alto · **S2** medio · **S3** bajo.
Los IDs son estables; el estado vivo está en `ledger.yaml`.

### S0

- **`PR-infra0001-breaks-upstream`** — El PR #22 reemplaza `pyproject.toml` (+24/−67): renombra el paquete a `am-tradingagents 0.1.0`, sube a `requires-python>=3.12`, reduce las dependencias a pydantic/sqlalchemy/fastapi (elimina langchain*, langgraph, pandas, yfinance, stockstats, typer, rich…), borra `[project.scripts] tradingagents` y `packages.find`. Además vacía `tradingagents/__init__.py` (+1/−37: pierde la carga de `.env` y los filtros de warnings). Mergearlo rompe la instalación, el CLI y el motor agéntico. **[V]** `git diff desarrollo-agentico...feature/US-INFRA-0001-repository-structure-and-ci-skeleton -- pyproject.toml tradingagents/__init__.py`. *Corrige:* no mergear; rehacer la story de forma **aditiva** (registrar los marcadores `architecture`/`traceability`, no reemplazar el archivo). Owner: Sonnet + decisión humana.
- **`LOOP-unrestricted-file-writes`** — `local_model.py:169-199` escribe **cualquier** ruta que el modelo emita, reemplazando el archivo completo. No hay lista blanca (`allowed_modules`), ni rutas protegidas, ni defensa contra `..` o rutas absolutas; ningún script lee `allowed_modules`, `protected_modules` ni `required_tests` de `stories.yaml`. Efecto observado: reescribió `tests/conftest.py` (+12/−60 de 67 líneas), `docker-compose.yml` (+14/−31 de 36), `scripts/local_model.py` y `.github/workflows/ci.yml`. **[V]** `grep` + `git diff --numstat`. *Corrige:* validar cada ruta contra `allowed_modules`; denegar `scripts/`, `.github/`, `docs/control-plane/`, `CLAUDE.md`, `pyproject.toml`, `tests/conftest.py`; rechazar `..`/absolutas; para archivos existentes exigir `patch`, no reemplazo. Owner: Sonnet.
- **`LOOP-branch-contamination`** — `pr_creator.py` crea la rama con `git checkout -b` desde el HEAD actual (`:64`), nunca vuelve a la base, y hace `git add` de **todo** lo que salga en `git status --porcelain` (`:41-47`). Resultado: PRs apilados y archivos ajenos dentro de PRs de story (`apps/web/dev-dashboard.html`, `apps/web/server.py`); #25 toca 55 archivos (+2,780/−93). Todos los PRs apuntan a `main` (`:136`), que es el espejo de upstream, no la rama de integración. Viola las invariantes 19–20. **[V]** *Corrige:* worktree limpio por story desde `desarrollo-agentico`; stage solo de rutas permitidas; base del PR = rama de integración. Owner: Sonnet.
- **`LOOP-no-success-writeback`** — Al crear un PR con éxito el loop no actualiza `stories.yaml` (`autonomous_loop.py:171-182`; no existe `mark_story_done` ni un estado `IN_REVIEW`). El picker vuelve a elegir la misma story, la reimplementa y `git checkout -b` falla por rama existente → `create_pr` falla → se marca `BLOCKED_AUTOMATION`: **una story exitosa termina bloqueada**. Por eso los DONE se marcaron a mano. **[I]** alta confianza. *Corrige:* estados `IN_REVIEW` (PR abierto) y `DONE` (solo tras merge + CI verde), con escritura atómica y lock. Owner: Sonnet.

### S1

- **`STORY-done-with-red-ci`** — #22 falla todo (tests 3.10–3.13, ruff, mypy, smoke). #23–#25 fallan `architecture`, `ruff`, `secret scan` y `tests` (pasan solo smoke y mypy, que nunca falla). **[V]** `gh pr checks 22..25`.
- **`CI-architecture-job-uncollectable`** — En `desarrollo-agentico` (el PR #1 también está rojo) `pytest tests/architecture -m "architecture or traceability"` falla en la colección: `'architecture' not found in markers configuration option`. `pyproject.toml` usa `--strict-markers` y solo registra `unit/integration/smoke`. Los 16 tests de fitness y trazabilidad nunca se han ejecutado con éxito; probable origen de la reescritura de `pyproject.toml` por Qwen en #22. **[V]** ejecutado.
- **`VALIDATOR-pytest-broken-env`** — `validate_story.py:68-87` invoca `pytest --timeout=60`, pero `pytest-timeout` no está instalado ni figura en los extras `dev`: pytest sale con código 4 (`unrecognized arguments`). Además la selección de tests nunca funciona (`tests/unit/test_infra_0001` sin `.py`; el fallback `tests/` siempre aplica, y la rama "no hay tests → skip" es inalcanzable), corre toda la suite upstream con `-x` y **no verifica `required_tests`** del contrato (invariante 15). **[V]** ejecutado + lectura.
- **`ENV-dev-environment-incomplete`** — `pytest --collect-only`: 174 tests, **62 errores de colección** (`ModuleNotFoundError: pandas`…). No hay venv con `pip install -e ".[dev]"`; no existe línea base verde para distinguir una regresión de un problema de entorno. **[V]**
- **`DEPS-undeclared-runtime-deps`** — El código de #23–#25 importa `fastapi` (10 imports), `sqlalchemy` (18), `pydantic` (7) y `jwt`, pero ni `pyproject.toml` ni `requirements.txt` los declaran (tampoco `psycopg` ni `alembic`). No se puede instalar desde la declaración del proyecto. **[V]**
- **`LOOP-fix-loop-blind`** — `autonomous_loop.py:71-89` pasa `result.stderr` como prompt de fix, pero `validate_story.py:145-158` imprime el detalle de los fallos por **stdout**. Con prompt vacío, `local_model.py:228` (`args.prompt or sys.stdin.read()`) se queda esperando stdin hasta el timeout o envía texto vacío al modelo. Los "3 intentos con auto-fix" son ciegos. **[I]** alta confianza.
- **`LOOP-no-failure-evidence`** — Los subprocesos usan `capture_output=True` y su salida se descarta al fallar; el log solo dice `Implementation failed`. `.orchestrator/` está vacío: `checkpoint.py` no se usa (invariante 32). El diagnóstico "SLM saturado" nunca pudo probarse. **[V]**
- **`LOOP-slm-context-starvation`** — `local_model.py:87-99` entrega a Qwen solo los primeros 1,500 caracteres de `target-architecture.md`, `domain-model.md` y `docs/12-adr/README.md` (que no contiene ADRs); el contrato que recibe es la prosa de `initial-20-stories.md`, no el contrato ejecutable de `stories.yaml`. Resultado: deriva de layout (creó `app/` en vez de `apps/api/`). **[V]**
- **`STORY-contract-noncompliance-done`** — US-INFRA-0003 permite `apps/api/`, `tradingagents/api/`, `tests/unit/api/` y exige `tests/unit/api/test_health.py`; la rama creó `app/main.py`, `app/api/v1/router.py` y `tests/unit/test_health_endpoint.py`. US-INFRA-0001 escribió en `tradingagents/`, `scripts/`, `apps/` y `migrations/` (fuera de `allowed_modules`) y no creó `tests/unit/test_infra_0001_ci_smoke.py`. **[V]**
- **`LOOP-no-independent-review`** — El flujo documentado Dev → Reviewer → Guardian → Merge (`docs/control-plane/review-model.md`; roles `reviewer_agent` y `architecture_guardian` en `model-registry.yaml`) no existe en `autonomous_loop.py`: implementar → validar → PR. Qwen escribe los tests de su propio código y el modo `fix` puede reescribirlos; nada detecta aserciones debilitadas (invariantes 11, 12, 14). **[V]**
- **`LOOP-issues-closed-falsely`** — `close_issue()` se llama en rutas de fallo y de skip (`autonomous_loop.py:150,160,168,180`), no solo al terminar. Los 20 issues están cerrados. **[V]** *Corrige:* reabrir; cerrar solo al mergear el PR.
- **`SEC-jwt-secret-not-configurable`** — En #23–#25, `app/core/config.py` define `class Settings(BaseModel)` (pydantic, **no** `pydantic-settings`): no lee variables de entorno aunque su docstring diga lo contrario. `jwt_secret` es siempre `"change-me-in-production"` (HS256): cualquiera puede firmar tokens válidos. `security.py` además devuelve `str(exc)` de PyJWT al cliente. Los tests (`test_unauthorized_returns_401.py`) solo cubren "sin token" y "token basura": no prueban token válido, expirado, `alg=none`, secreto distinto ni configuración por entorno. Mismo patrón en `app/db/config.py:14` (password por defecto). **[V]** lectura del código y de los tests. *Es un ejemplo real de "defecto que se escapó a los tests".*

### S2

- **`VALIDATOR-scope-mismatch`** — `validate_story.py:41-65`: ruff y mypy apuntan a `tradingagents/` y `apps/`, pero Qwen escribió en `app/` (nunca se valida). Además `ruff --fix` corre sobre los árboles completos y modifica archivos ajenos a la story (viola `allowed_modules`, ensucia el diff). **[V]**
- **`VALIDATOR-arch-check-weak`** — `validate_story.py:90-108` usa coincidencia de subcadenas (`f"import {banned}" in content`) y solo 2 reglas: no detecta imports relativos, alias ni `importlib`, y da falsos positivos en comentarios/strings. Duplica, en peor versión, a `tests/architecture/test_fitness.py` (basado en AST). **[V]**
- **`TEST-secret-scan-blind-spot`** — `test_no_hardcoded_secrets_patterns` (`tests/architecture/test_fitness.py:222-244`) solo recorre `tradingagents/**/*.py`: no ve `app/`, `apps/`, `scripts/`, YAML ni `docker-compose.yml`; solo 5 patrones. **[V]**
- **`DATA-stories-yaml-mojibake`** — 10 líneas de `stories.yaml` con doble codificación (`AgÃ©ntico`, `â€”`, `â†’`); `story_picker.py:34` abre el YAML sin `encoding='utf-8'` (en Windows usa cp1252); `mark_story_blocked()` reserializa todo el archivo con `yaml.dump` (pierde comentarios y formato) sin lock. El dashboard muestra texto corrupto. **[V]** contenido / **[I]** causa.
- **`DOCS-no-adrs`** — `docs/12-adr/` solo tiene un README; la invariante 6 exige ADR para cambios de arquitectura y ya se tomaron decisiones de facto (layout `app/`, Postgres + Alembic, JWT HS256, relación con upstream). **[V]**
- **`ARCH-layout-and-frontend-duplication`** — 3 layouts de backend: `tradingagents/` (upstream), `apps/api/` (declarado en `stories.yaml`) y `app/` (lo que creó Qwen); 3 UIs: `src/` (Next.js, 9 archivos), `frontend/` (25) y `apps/web` (dashboard estático). No hay canon ni ADR. **[V]** `git ls-files`.
- **`PROCESS-branching-and-fork-strategy`** — `main` es espejo de upstream v0.5.0 y los PRs de story apuntan a él; el PR #1 (abierto el 2026-09-23 como "Phase 0") ya acumula 94 archivos / +11,412 y está en rojo; no hay política para integrar upstream (el remote `upstream` existe) ni una rama de integración definida. **[V]**

### S3

- **`LOOP-blind-retries`** — `autonomous_loop.py:20-42`: `run()` reintenta una vez cualquier comando fallido (incluido `git push`, `gh issue close`, `implement`), contra la invariante 31. **[V]**
- **`LOOP-pr-metadata-wrong`** — `pr_creator.py:82` fija `Co-Authored-By: Claude Sonnet 4.6`; el título que pasa el loop sale del ID (`us infra 0001` → `Us Infra 0001`, `autonomous_loop.py:93`) en vez del título de la story. **[V]**
- **`CI-mypy-non-blocking`** — `ci.yml:76`: `mypy … || true`; el "pass" de mypy en los PRs no significa nada. **[V]**
- **`OPS-dashboard-not-persistent`** — Servicio no instalado, sin lanzador en Inicio, puerto 8888 caído. Además el dashboard lee `stories.yaml`, cuyo estado no es fiable (ver arriba). **[V]**

### Hipótesis de dominio (aún sin código; Fable debe confirmar o refutar)

- **`H-data-yfinance-licensing`** — US-DATA-0001 y el motor upstream dependen de yfinance (extractor no oficial de Yahoo). Para un SaaS comercial hay riesgo de términos de uso, límites de tasa y disponibilidad. Proponer abstracción de fuente, proveedor con licencia y política de degradación. *(No es asesoría legal: requiere revisión humana.)*
- **`H-agent-prompt-injection-via-news`** — Los agentes ingieren noticias, redes y fundamentales (texto no confiable): la inyección indirecta puede sesgar un `TradeIntent`. Verificar aislamiento del texto no confiable, salida estructurada validada, el Risk Engine determinístico como último control y pruebas adversariales.
- **`H-risk-oms-toctou-idempotency`** — Carrera entre la evaluación del Risk Engine y el envío del OMS (el estado cambia entre ambos), órdenes duplicadas por reintentos, kill switch que falle abierto. Diseñar claves de idempotencia, estado transaccional y comportamiento fail-closed.
- **`H-tenant-isolation-design`** — Modelo multi-tenant (US-SEC-0001): ¿RLS en Postgres o filtrado en la aplicación? Riesgo de fuga entre tenants por consultas sin filtro; contenido de los claims JWT.
- **`H-money-time-calendar`** — `float` vs `Decimal` en precios y P&L; zonas horarias, calendario de mercado, ajustes por splits y dividendos.
- **`H-reproducibility-nondeterminism`** — Reproducibilidad de decisiones LLM (versiones de prompt/modelo/semilla según `reproducibility-contract.yaml`) y `DecisionTrace` sin chain-of-thought (invariante 30).
- **`H-upstream-drift`** — Política para integrar futuras versiones de TradingAgents sin romper `tradingagents/`.

---

## 3. Dónde el modelo profundo aporta más valor

Ordenado por retorno esperado para **este** proyecto.

| # | Área | Qué haría | Por qué aquí | Frecuencia |
|---|---|---|---|---|
| 1 | **Reality check del orquestador** | Contrastar lo que afirman `stories.yaml`, dashboard, commits y mis mensajes contra la verdad (git, PRs, CI, issues, procesos). | Este informe existe porque el orquestador dio por hechas cosas falsas. Un auditor independiente evita que se repita. | Cada corrida |
| 2 | **Revisión independiente de cada diff antes del merge** | Hacer de Reviewer + Architecture Guardian (los roles documentados pero no implementados). | Qwen escribe el código *y* sus tests; nadie más lo revisa. | Cada corrida (AM) |
| 3 | **Auditoría de la maquinaria** (loop, validador, CI, registry) | Buscar defectos en `scripts/`, `.github/`, `docs/control-plane/`. | Hoy es donde está el riesgo S0. Es el código que gobierna todo lo demás. | Cada corrida; a fondo semanal |
| 4 | **Stories a "tamaño SLM" + work orders** | Re-especificar cada story: 1 módulo, ≤5 archivos, ≤~200 LOC, interfaces exactas, `allowed_paths`, tests que deben fallar antes del fix. | Es la palanca que más sube la tasa de éxito de un modelo de 12 GB de VRAM. | PM |
| 5 | **Ejemplos dorados y esqueletos** | Escribir implementaciones y tests de referencia por tipo de story (endpoint, modelo ORM, servicio, agente LangGraph) para usarlos como few-shot de Qwen. | Reduce la deriva de layout y de estilo. | Semanal / bajo demanda |
| 6 | **Arquitectura y ADRs** | Redactar los ADR que faltan (layout, un solo frontend, auth, relación con upstream, rama de integración) y desafiar `target-architecture.md`. | `DOCS-no-adrs`, `ARCH-layout…`, `PROCESS-branching…`. | PM / semanal |
| 7 | **Tests que no fallarían aunque el código esté mal** | Análisis tipo mutation: "¿este test fallaría si rompo el invariante?", con el cambio exacto que lo demostraría. Revisar la fuerza de los fitness tests. | Ejemplo real: `SEC-jwt-secret-not-configurable` pasó sus tests. | AM / semanal |
| 8 | **Seguridad y modelado de amenazas** (STRIDE) | Auth, tenant, secretos, `BrokerGateway`, gating LIVE, cadena de suministro, licencias del fork Apache-2.0. | Es un SaaS con dinero de por medio. | Semanal |
| 9 | **Seguridad específica de trading agéntico** | Inyección indirecta vía noticias, validación de `TradeIntent`, idempotencia OMS, TOCTOU Risk↔OMS, kill switch fail-closed, `Decimal`/zona horaria/look-ahead. | Hipótesis `H-*`; ningún test estándar las cubre. | PM / semanal |
| 10 | **Calibración de ruteo de modelos y prompts** | Analizar fallos reales de Qwen (una vez que el loop guarde evidencia), reescribir los prompts `implement`/`fix` y decidir qué stories van a Qwen, a Sonnet o a humano. | Los niveles de riesgo y el ruteo hoy son intuiciones, no datos. | Semanal |
| 11 | **Trazabilidad y planificación** | Matriz FR/NFR ↔ story ↔ test; ciclos y aristas faltantes en el grafo de dependencias; camino crítico; re-priorización del roadmap. | 20 stories cubren una fracción de `docs/01-requirements`. | PM / semanal |
| 12 | **Diseño de evaluación de agentes** | Harness de backtests/replays con métricas, criterios de aceptación del comportamiento agéntico, sesgos (look-ahead, supervivencia). | `docs/control-plane/ai-evaluation-strategy.md` existe pero no está implementado. | Semanal |
| 13 | **Coste y operación** | Modelo de coste por corrida (tokens, cómputo), cuotas de `cost-guardrails.yaml`, ergonomía de operar en Windows (servicios, logs, rotación). | El loop hoy no mide nada. | Semanal |
| 14 | **Coherencia documental** | Detectar deriva entre `CLAUDE.md`, `model-registry.yaml`, `stories.yaml` y el código. | Ya hay 3 contradicciones (orquestador, DONE, Phase 0.5). | Cada corrida |

**Fuera de alcance para Fable:** escribir código de producto (lo hace Qwen), tocar credenciales, activar LIVE, mergear PRs.

---

## 4. Riesgos de este enfoque (para decidirlo con los ojos abiertos)

- **Costo:** MAX es caro; por eso solo semanal. AM/PM se limitan a diff + lo que va a ejecutarse.
- **Falsos positivos:** un modelo profundo también se equivoca. Por eso todo hallazgo S0/S1 lo **reproduzco yo** antes de actuar y los rechazados quedan en el ledger con motivo (alimentan el siguiente prompt).
- **Cuello de botella humano:** los hallazgos ORANGE/RED y las decisiones de arquitectura llegan a ti. Fable debe formularlas como preguntas cerradas.
- **Deriva del prompt:** el prompt está versionado (`v1.0`); cualquier cambio sube versión y se registra en `model-registry.yaml`.

---

## 5. Modelo operativo propuesto

**Roles**

| Actor | Modelo | Puede escribir |
|---|---|---|
| Arquitecto / auditor | Claude Fable 5.1 (`claude-fable-5-1`) o el más reciente | **Solo** archivos nuevos en `docs/architect-reports/reports/` |
| Orquestador | Claude Sonnet 5 (Claude Code) | `ledger.yaml`, `stories.yaml`, scripts, CI, ADRs; verifica y delega |
| Implementador | Qwen3.8-27B local | Solo las rutas de un work order aprobado |
| Humano | — | Decisiones ORANGE/RED, merges, credenciales, LIVE |

Regla: **Qwen no modifica la maquinaria que lo restringe** (`scripts/`, `.github/`, `docs/control-plane/`, `CLAUDE.md`, `pyproject.toml`). Eso lo hace el orquestador con tu aprobación.

**Cadencia** (tú corres a Fable a mano)

| Corrida | Hora sugerida | Nivel | Foco |
|---|---|---|---|
| **AM** | 07:00 | STANDARD o DEEP | Post-mortem de lo ocurrido desde la última corrida (commits, PRs, ramas, logs): reality check, revisión de código y de tests de lo nuevo, verificación de fixes. |
| **PM** | 19:00 | DEEP | Estratégica: arquitectura, requisitos, re-especificación de stories, ADRs y **cola de work orders para la noche** (≤ 8). |
| **WEEKLY** | domingo | MAX | Auditoría completa de todos los ejes, amenazas, coherencia docs↔código, recalibración de riesgo y ruteo. |

**Primera corrida: WEEKLY + MAX**, para validar esta línea base. Si tu interfaz permite fijar el esfuerzo de razonamiento, úsalo acorde al nivel.

**Flujo**

```
Humano (2×/día) ──prompt──▶ Fable  (lee el repo, solo lectura)
                              │  escribe reports/*.md + *.findings.yaml → push a desarrollo-agentico
                              ▼
Sonnet ── git fetch → lee findings → REPRODUCE cada S0/S1 → ledger.yaml
   ├─ work order pequeño ──▶ story US-FIX-#### en stories.yaml ──▶ Qwen ──▶ PR ──▶ CI ──▶ (merge humano)
   ├─ arreglo de maquinaria (scripts/CI/control-plane) ──▶ Sonnet, con tu aprobación
   ├─ decisión de arquitectura / riesgo ORANGE-RED ──▶ pregunta cerrada al humano (+ ADR)
   └─ falso positivo ──▶ rejected + motivo
                              ▼
Siguiente corrida de Fable re-verifica los `fixed_pending_verify`
```

Detalles del protocolo (formato de informes, estados del ledger, pasos de ingesta): `README.md`.

---

## 6. Acciones inmediatas recomendadas (en este orden)

| # | Acción | Requiere tu OK |
|---|---|---|
| 1 | **No mergear** los PRs #22–#25. Decidir si se cierran o se dejan como evidencia hasta rehacerlos. | Sí (cerrar) |
| 2 | **Mantener el loop apagado** hasta corregir los S0 (`LOOP-*`, `PR-infra0001-…`). | — |
| 3 | Corregir la maquinaria: lista blanca de escritura, worktree por story, write-back `IN_REVIEW`/`DONE`, cierre de issues solo al merge, guardar la salida de fallos, stdout/stderr del fix-loop, `pytest-timeout`, marcadores de pytest. | Sí (código del control-plane) |
| 4 | Preparar entorno: venv + `pip install -e ".[dev]" pytest-timeout` y capturar la línea base de tests. | — |
| 5 | Reabrir los issues #2–#21 (`gh issue reopen`). | Sí (acción en GitHub) |
| 6 | Reparar el encoding de `stories.yaml`; pasar las 4 stories de `DONE` a `IN_REVIEW` (cambio de schema + dashboard). | Sí |
| 7 | Decidir por ADR: layout del backend (`apps/api` vs `app`), un solo frontend (`src` vs `frontend`), rama base de PRs y estrategia con upstream. | Sí (decisión) |
| 8 | Registrar en `model-registry.yaml` el rol `architect_auditor` (`claude-fable-5-1`) y el prompt `architect_audit_v1` (control #20). | Sí |
| 9 | **Primera corrida de Fable (WEEKLY + MAX).** | Tú la ejecutas |
| 10 | Baja prioridad: dashboard como servicio (admin) o lanzador en Inicio. | Sí (admin) |

---

## 7. Cómo sabremos si funciona

- % de hallazgos S0/S1 que yo confirmo al reproducirlos (calidad de Fable).
- Tiempo entre "hallazgo" y "fix verificado" en una corrida posterior.
- % de PRs de Qwen con CI verde **al primer intento** (subirá con work orders pequeños y ejemplos dorados).
- Defectos que aparecen después del merge y que un informe previo no anticipó (escapes).
- Stories realmente `DONE` (mergeadas + CI verde) vs. las que decía el dashboard.

---

## Anexo A — Comandos que reproducen la evidencia

```bash
git log --oneline main..desarrollo-agentico                       # 13 commits por delante de main
git rev-list --count desarrollo-agentico..feature/US-*            # 3, 2, 1, 1
gh pr list --repo orisonsoto/AM-TradingAgents --state all         # #1, #22–#25 (base main)
gh pr checks 22 --repo orisonsoto/AM-TradingAgents                # y 23, 24, 25, 1
gh issue list --repo orisonsoto/AM-TradingAgents --state all      # 20 CLOSED
git diff --numstat desarrollo-agentico...<feature> -- pyproject.toml tradingagents/__init__.py tests/conftest.py docker-compose.yml
python -m pytest tests --co -q                                    # 174 tests, 62 errores de colección
python -m pytest tests/architecture -q -m "architecture or traceability"   # error de marcadores
python -m pytest tests/architecture -q --timeout=60               # unrecognized arguments
grep -n "allowed_modules\|required_tests\|checkpoint" scripts/*.py         # sin coincidencias
```
