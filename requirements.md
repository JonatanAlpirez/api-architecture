# Requisitos y Preguntas Abiertas — api-architecture

> **Estado:** levantamiento de requisitos (previo a propuesta de arquitectura).
> Doc vivo: se actualiza conforme aterricen decisiones.
> **Próximos entregables en este repo:** `architecture-proposal.node-ts.md`, `architecture-proposal.python.md` y `architecture-proposal.java-spring.md` (una propuesta por stack, comparables entre sí).

---

## ✅ Requisitos confirmados por Jonatan

| ID   | Requisito                                                                                                | Notas                                                                                                       |
| ---- | -------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| R1   | API documentada con **Swagger / OpenAPI 3.x**                                                            | El FE debe poder generar servicios y modelos automáticamente a partir del spec.                             |
| R2   | Arquitectura en capas **Controller → Service → Repository**                                              | Origen Spring Boot/Java. Abierto a equivalente moderno en cualquier stack.                                  |
| R3   | Stack **moderno y estandarizado**                                                                        | Sin preferencia rígida por ecosistema; evaluamos opciones.                                                  |
| R4   | Base de datos **relacional**                                                                             | —                                                                                                           |
| R5   | Reutilizar los datos de `~/Documents/gym_training-data/`                                                | Hoy usa SQLite; la DB destino de la API puede ser otra.                                                     |
| R6   | Consumo **local**                                                                                        | Primer cliente: `main-dashboard`. Más adelante podrían sumarse otros módulos.                               |
| R7   | Antes de la propuesta: preguntar dudas / recomendaciones + documentar reqs/pendientes                     | Hecho en este doc.                                                                                          |

---

## 🗃️ Shape actual de los datos (auditado 2026-07-11)

Para no preguntar a ciegas sobre qué entidades expone la API. Fuente: `~/Documents/gym_training-data/DB/create_schema.sqlite`.

**4 tablas** + 1 catálogo JSON:

| Tabla              | Columnas clave                                                                                   | Filas aprox. |
| ------------------ | ------------------------------------------------------------------------------------------------ | ------------ |
| `exercises`        | `id`, `name` (UNIQUE), `muscle_group`, `created_at`, `updated_at`                                | 33           |
| `workouts`         | `id`, `workout_num` (UNIQUE), `date`, `name`, `duration_sec`, `created_at`, `updated_at`         | 108          |
| `workout_exercises` | `id`, `workout_id` (FK), `exercise_id` (FK), `exercise_order`, `created_at` — junction N:N      | ~variable    |
| `sets`             | `id`, `workout_exercise_id` (FK), `set_order`, `weight_lb`, `reps`, `rpe`, `created_at`         | 1,707        |

Catálogo: `DB/muscle_group_mapping.json` — agrupa ejercicios en `Chest | Back | Shoulders | Legs | Arms | Core`.

**Endpoints obvios que el FE va a pedir** (basado en `health-dashboard` y consumo típico):

- `GET /workouts` (paginado, filtros por `date_from` / `date_to` / `muscle_group`)
- `GET /workouts/:id` (con `workout_exercises` + `sets` anidados)
- `GET /exercises` (catálogo, filtros por `muscle_group`)
- `GET /exercises/:id` (detalle + historial de uso)
- `GET /stats/...` (volumen por músculo, PRs, frecuencia semanal — a confirmar)

---

## ❓ Preguntas abiertas

> **Estado por pregunta:**
> - ✅ **Respondida por Jonatan y completada** — la decisión/el pedido ya está incorporado.
> - 🟡 **Pendiente** — default propuesto a la espera de OK.
>
> Si una default te copa, contestás "default" y avanzo. Si querés cambiar, decime el reemplazo.

### Stack / runtime

- **Q1. ¿Qué stack evaluamos?** ✅ Respondida
  - **Decisión de Jonatan:** evaluamos **TRES stacks en paralelo** y comparamos.
    - **A) Node + TypeScript** — consistencia con `main-dashboard`, codegen OpenAPI maduro, tooling moderno.
    - **B) Python** — ecosistema data/biomédico fuerte, alineado con el background de Jonatan, FastAPI da OpenAPI first-class.
    - **C) Java + Spring Boot** — stack más cercano a la formación y rol actual de Jonatan (Spring Boot es su origen, R2 lo confirma), type system más fuerte, ecosistema más maduro del mercado enterprise; tradeoff: más boilerplate, JVM startup, iteración más lenta en dev.
  - Implicación: las preguntas Q2, Q3, Q4, Q5, Q15, Q20, Q21 se aterrizan **en cada propuesta**, no en este doc.

- **Q2. ¿Qué es un framework HTTP y qué opciones hay?** ✅ Respondida
  - **Pedido de Jonatan:** explicar qué es un framework HTTP y listar opciones.

  **Definición:** un framework HTTP es la capa entre los requests HTTP entrantes y tu lógica de negocio. Encapsula:
  - **Routing** — mapeo URL → handler (ej: `GET /workouts/123` → `workoutController.show`).
  - **Parsing** — extracción de path params, query string, JSON body, headers.
  - **Serialization** — tus objetos ↔ JSON de respuesta.
  - **Middleware** — auth, logging, CORS, error handling, rate limit, compresión.
  - **Status codes + headers** — semántica HTTP correcta (200, 201, 404, 500, `Content-Type`, etc.).
  - **Generación de OpenAPI** (en los modernos) — el spec se genera desde el código, no se mantiene a mano.

  Sin framework, usarías `node:http` o `http.server` puro y reimplementarías todo eso a mano. El framework es la abstracción que evita esa fatiga.

  **Opciones para Node + TypeScript:**

  | Framework     | Pros                                                                                              | Contras                                              | Mejor cuando                                                                |
  | ------------- | ------------------------------------------------------------------------------------------------- | ---------------------------------------------------- | ---------------------------------------------------------------------------- |
  | **Hono**      | Ultra-rápido, minimal, edge-friendly, OpenAPI vía `@hono/zod-openapi`, multi-runtime              | Ecosistema chico vs Express                          | APIs pequeñas/modernas, queremos correr en Node/Bun/Workers/Deno             |
  | **Fastify**   | Muy rápido, JSON Schema nativo, ecosistema maduro                                                 | Más boilerplate que Hono                             | APIs medianas con foco en performance                                        |
  | **Express**   | Ubicuo, millones de middlewares third-party                                                       | API vieja, tipado flojo, no edge                     | Compatibilidad con código/ecosistema legacy                                   |
  | **NestJS**    | DI, decoradores, "Spring Boot de Node", opinionated                                               | Curva de aprendizaje, verboso                        | Equipos grandes con estructura opinionated                                    |
  | **Koa**       | Minimalista, sucesor espiritual de Hono en diseño                                                 | Muy low-level                                        | Control fino total del stack                                                 |
  | **Elysia**    | Bun-native, TS-first, performance altísima                                                         | Acoplado a Bun runtime                               | Si elegimos Bun como runtime                                                 |

  **Opciones para Python:**

  | Framework             | Pros                                                                  | Contras                                                | Mejor cuando                                                    |
  | --------------------- | --------------------------------------------------------------------- | ------------------------------------------------------ | ---------------------------------------------------------------- |
  | **FastAPI**           | OpenAPI automático, async nativo, validación con Pydantic, muy popular | Relativamente nuevo (2018)                              | APIs modernas, queremos OpenAPI first-class                      |
  | **Flask**             | Simple, ubicuo, maduro                                                | No async nativo, OpenAPI vía flask-smorest/flask-pydantic | APIs chicas, equipo Python clásico                              |
  | **Django REST Framework** | Batteries-included (ORM + admin + auth + router)                  | Acoplado a Django, pesado                              | Si ya usamos Django, o queremos todo integrado                  |
  | **Starlette**         | ASGI puro, base sobre la que está construido FastAPI                  | Low-level, sin OpenAPI out-of-the-box                   | Máximo control sobre el async stack                             |
  | **Litestar**          | Más tipado que FastAPI, OpenAPI built-in                              | Comunidad chica                                        | Equipos que valoran type-safety fuerte                           |
  | **Tornado**           | Async desde 2010, maduro                                              | Sintaxis menos pythonic moderna                        | Sistemas real-time / websockets                                  |

  **Opciones para Java:**

  | Framework             | Pros                                                                                              | Contras                                                  | Mejor cuando                                                                              |
  | --------------------- | ------------------------------------------------------------------------------------------------- | -------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
  | **Spring Boot 3**     | De facto del mercado Java, batteries-included (DI, security, data, web), ecosistema enorme, springdoc-openapi para spec | Verboso, iteración más lenta en dev, JVM startup time     | APIs enterprise, equipos con background Spring (caso de Jonatan), sistemas críticos       |
  | **Quarkus**           | Kubernetes-native, GraalVM native compilation (startup <100ms), API familiar a Spring devs         | Comunidad más chica que Spring, menos Stack Overflow      | Microservicios cloud-native, funciones serverless, containers efímeros                  |
  | **Micronaut**         | Compile-time DI (más rápido que reflection), GraalVM-friendly, similar a Spring                  | Comunidad mediana, menos plugins que Spring               | Microservicios rápidos, equipos que valoran startup time                                   |
  | **Helidon**           | Oracle-maintained, Helidon SE (microframework) o MP (Jakarta EE)                                | Comunidad chica, menos conocido                          | Entornos Oracle/Java EE existentes, polyglot (Java + Kotlin)                              |

  - 🟡 **Default concreto por stack** (lo aterriza cada propuesta): A) Hono; B) FastAPI; C) Spring Boot 3 (alineado con R2 y el background de Jonatan).

- **Q3.** ¿Bun/Deno como runtime, o Node LTS? 🟡
  - Default: **Node LTS** (universalmente compatible). Bun se evalúa dentro de la propuesta Node como alternativa.

### Base de datos y ORM

- **Q4. ¿Qué DB destino?** ✅ Respondida
  - **Pedido de Jonatan:** tabla para explorar características de las opciones.

  En nuestro contexto (API read-only local + sync desde `gym_tracker.db`) las opciones razonables son tres:

  | Característica                | SQLite (libsql)                                                       | PostgreSQL                                                          | MySQL/MariaDB                                                |
  | ----------------------------- | --------------------------------------------------------------------- | ------------------------------------------------------------------- | ------------------------------------------------------------ |
  | Forma                         | Embedded (archivo `.db`)                                              | Server (local: Docker o `brew services start postgresql`)           | Server                                                       |
  | Setup local                   | Cero — un archivo                                                     | Bajo — un container o servicio                                      | Bajo                                                         |
  | Concurrencia                  | Writer lock global (un writer a la vez); excelente para read-heavy    | MVCC — muchos writers en paralelo                                   | MVCC — buena concurrencia                                    |
  | Tipos de datos                | Core limitado, JSON nativo (JSON1)                                    | Rico: JSONB, Arrays, Ranges, GIS, full-text                         | Menos rico que PG, JSON nativo                               |
  | ORM Node (Drizzle)            | Excelente — driver oficial `libsql`                                   | Excelente (`postgres-js`, `pg`, `node-postgres`)                    | Bueno (`mysql2`)                                             |
  | ORM Python (SQLAlchemy)       | Bueno                                                                 | **Excelente** (camino más popular)                                  | Bueno                                                        |
  | Migraciones                   | Drizzle Kit, Alembic, scripts raw                                     | Todas las herramientas del ecosistema                               | Igual                                                        |
  | Footprint para nuestro caso   | **Perfecto** — mismo motor que `gym_tracker.db`, zero-config           | Overkill para local single-machine                                  | Overkill                                                     |
  | Path de escalado              | Vertical + replicas de lectura (libsql / Turso remoto)                | Vertical + horizontal: replicas, partitioning, lógica multi-nodo    | Vertical + horizontal                                        |

  - **Recomendación actual:** **SQLite (libsql)** se mantiene como default. Razones en este proyecto:
    - Mismo motor que `gym_tracker.db` → el sync es trivial (conectar a ambos SQLite desde el mismo proceso).
    - Cero servicios adicionales corriendo en local.
    - Si la API crece y necesita JSONB o múltiples writers, migrar a Postgres es un cambio de una línea (driver) en ambas propuestas.
    - La alternativa "leer `gym_tracker.db` directo" se descarta: acopla la API al filesystem del data warehouse.

- **Q5. ¿Qué ORM?** ✅ Respondida
  - **Pedido de Jonatan:** tabla comparativa de ORMs.

  **Node + TypeScript:**

  | ORM           | Tipo                                | TS types                              | Migraciones                  | Query style                                  | Curva        | Footprint                       |
  | ------------- | ----------------------------------- | ------------------------------------- | ---------------------------- | -------------------------------------------- | ------------ | ------------------------------- |
  | **Drizzle**   | Query builder + schema typed         | First-class (definís en TS)           | Drizzle Kit (schema → SQL)   | SQL-like, explícito                          | Bajo-medio   | Liviano                         |
  | **Prisma**    | Schema-first (DSL propio `.prisma`) | Genera TS desde schema                | Prisma Migrate               | High-level (`db.users.findMany()`)            | Bajo         | Pesado (engine binario Rust)     |
  | **Kysely**    | Query builder puro                  | Types solo en el builder              | Knex migrations (separado)   | SQL puro tipado                              | Medio        | Liviano                         |
  | **MikroORM**  | Data-mapper estilo JPA/Spring       | Bueno                                 | MikroORM CLI                 | Similar a Hibernate                          | Medio-alto   | Medio                           |
  | **TypeORM**   | Decoradores, data-mapper            | TS con decoradores                    | TypeORM CLI                  | Data-mapper / active-record                  | Alto         | Medio                           |

  **Python:**

  | ORM                  | Estilo                                      | Types                            | Migraciones                  | Curva        | Footprint                         |
  | -------------------- | ------------------------------------------- | -------------------------------- | ---------------------------- | ------------ | --------------------------------- |
  | **SQLAlchemy 2.0**   | Core + ORM (dos modos, se complementan)     | `Mapped[]` hints + mypy          | Alembic                      | Medio-alto   | Medio                             |
  | **SQLModel**         | Inspirado en SQLAlchemy + Pydantic          | Pydantic = models                | Alembic                      | Bajo-medio   | Liviano                           |
  | **Django ORM**       | Active-record                               | Limitado (`django-stubs`)         | `makemigrations` + `migrate` | Bajo         | Pesado si sumás todo Django       |
  | **Tortoise ORM**     | Async, active-record                        | Tipos vía Pydantic                | Aerich                       | Bajo-medio   | Liviano                           |
  | **Peewee**           | Minimalista                                 | Tipos básicos                    | peewee-migrate o manual      | Bajo         | Liviano                           |

  **Java:**

  | ORM                          | Estilo                                                  | Types                          | Migraciones                     | Curva        | Footprint              |
  | ---------------------------- | ------------------------------------------------------- | ------------------------------ | ------------------------------- | ------------ | ---------------------- |
  | **Spring Data JPA (Hibernate)** | De facto del mundo Java, JPA estándar, repositories generados | Inferencia por nombre de método | Flyway / Liquibase             | Bajo (con starters de Spring) | Medio-pesado |
  | **jOOQ**                     | SQL-first, code-gen del schema → tipos Java             | First-class                     | Flyway / Liquibase              | Medio        | Liviano                |
  | **Spring Data JDBC**         | Más simple que JPA, sin lazy-loading magic              | Inferencia básica               | Flyway / Liquibase              | Bajo         | Liviano                |
  | **MyBatis**                  | SQL mapper, vos escribís el SQL                         | Manual                         | Flyway / Liquibase              | Bajo         | Liviano                |
  | **Jdbi**                     | SQL-first fluent API (similar a jOOQ)                   | Manual                         | Flyway / Liquibase              | Bajo-medio   | Liviano                |

  - **Recomendación por stack:** A) Drizzle (liviano, TS-first, encaja con Hono/Fastify); B) SQLAlchemy 2.0 (maduras, mypy-friendly, Alembic es battle-tested); C) **Spring Data JPA / Hibernate** (estándar en Spring Boot, encaja con R2 "origen Spring Boot/Java", repositorios derivados sin escribir SQL).

- **Q6. ¿Qué son las migraciones en este contexto?** ✅ Respondida
  - **Pedido de Jonatan:** explicar a qué se refiere con "migraciones" en este contexto.

  **Definición:** una migración es un **cambio versionado del schema de la DB, escrito como código, aplicado en orden**.

  **Por qué importan acá:**
  - `api_health.db` va a **divergir** de `gym_tracker.db` (cómputos cacheados, vistas materializadas, índices adicionales, columnas calculadas).
  - Si mañana Jonatan agrega una columna a `gym_tracker.db` (su CSV de Strong evoluciona), queremos reproducir el cambio en la API de forma **reproducible y auditada**, no a mano.
  - Sin migraciones, cada cambio de schema se convierte en: (a) correr SQL a mano (olvidás comandos, no funciona en otra máquina), o (b) borrar la DB y reseedear (perdés estado local como timestamps de sync, contadores, etc.).

  **Cómo funcionan en la práctica:**
  1. Editás el schema en código (`schema.ts` para Drizzle, `models.py` para SQLAlchemy).
  2. Corrés el tool → diff vs última migración → genera un nuevo archivo SQL.
     - `drizzle-kit generate` → `0003_add_volume_view.sql`
     - `alembic revision --autogenerate` → `0003_add_volume_view.py`
  3. Revisás el SQL generado (clave para no aplicar basura), lo aplicás (`drizzle-kit migrate` / `alembic upgrade head`).
  4. El tool lleva una tabla `__migrations` adentro de la DB y solo aplica las nuevas. Revertir (downgrade) existe, pero la mayoría de los flujos modernos son **forward-only**.

  **En nuestro flujo específico:**
  - `api_health.db` va con **forward-only migrations** (más simple, sin riesgo de reversión parcial).
  - Cada archivo de migración queda commiteado al repo (es código, no magia).
  - El script de sync (Q10) corre **después** de las migraciones en cada deploy o arranque.

  **Tools por stack** (todos siguen el patrón "diff vs última migración → archivo versionado → aplicar"):

  | Stack                  | Tool recomendado        | Formato de archivos                        | Notas                                                                                              |
  | ---------------------- | ----------------------- | ------------------------------------------ | -------------------------------------------------------------------------------------------------- |
  | A) Node + TS           | **Drizzle Kit**         | `.sql` planos (`0001_create_exercises.sql`) | Genera SQL desde el schema TS; lightweight.                                                        |
  | B) Python              | **Alembic**             | `.py` con `upgrade()` / `downgrade()`      | Estándar de facto del ecosistema SQLAlchemy.                                                       |
  | C) Java + Spring Boot  | **Flyway**              | `.sql` planos (`V1__create_exercises.sql`) | Spring Boot auto-detecta Flyway en el classpath. Alternativa: **Liquibase** (XML/YAML, más potente pero más verboso). **Evitar `hibernate.hbm2ddl.auto=update` en prod** — es dev-only y puede corromper data. |

  - **Default por stack:** A) **Drizzle Kit**; B) **Alembic**; C) **Flyway**.

### Contrato de API (Swagger / OpenAPI)

- **Q7. Spec-first o code-first?** 🟡
  - Default: **code-first** (anoto controllers/types, genero OpenAPI desde el código con `@hono/zod-openapi` o equivalente del stack final). Más rápido de iterar, menos archivos para mantener sincronizados.
  - Alternativa: spec-first (escribo `openapi.yaml` primero, genero tipos/validators desde el spec).
- **Q8. Versión del spec?** 3.0 vs 3.1? 🟡
  - Default: **3.0** (reuso directo del codegen del FE sin reconfigurar nada).
- **Q9. Codegen para el frontend?** 🟡
  - Default: mantener **`openapi-typescript`** (ya en uso en `main-dashboard`).
  - Alternativas: `orval` (genera hooks de React Query además de tipos), `openapi-generator`.

### Integración con `gym_training-data/`

- **Q10. ¿Cómo accede la API a los datos?** 🟡
  - Default: la API **abre su propio SQLite** (`api_health.db`) y los datos se **sincronizan** desde `gym_training-data/DB/gym_tracker.db` mediante un comando/script (`pnpm sync:health` o `python -m api.sync`). La API no toca el archivo fuente.
  - Alternativas: (a) la API lee directo el SQLite de `gym_training-data/` (sin sync, pero acopla la API al filesystem del data warehouse); (b) se hace una **migración one-shot** y `gym_training-data/` queda solo como fuente histórica para re-imports.
- **Q11. ¿La API solo expone `health` o ya planeamos otros módulos desde el inicio?** 🟡
  - Default: arranca **solo con `health`** (workouts/exercises). Cuando definamos `finance` o `reminders` como APIs, agregamos módulos a la misma API o las separamos — decisión arquitectónica que aterriza en la propuesta.
- **Q12. ¿Endpoints de escritura (POST/PUT/DELETE)?** 🟡
  - Default: **read-only en v1**. Escritura queda para v2 si surge necesidad (p. ej. registrar workouts desde la app).

### Auth / multi-tenancy

- **Q13. ¿Auth?** 🟡
  - Default: **API key estática en header `X-API-Key`** — cuesta 5 líneas, evita sustos si el puerto queda expuesto por accidente.
  - Alternativas: (a) sin auth; (b) JWT con login simple (overkill probable para local).
- **Q14. ¿Una sola API para todos los módulos o un servicio por módulo?** 🟡
  - Default: **monolito modular** (`/health/...`, `/finance/...`, `/reminders/...`) por ahora. Separamos en microservicios solo si la complejidad lo pide.

### Runtime / deployment

- **Q15. ¿Cómo corre?** Proceso bare, fat jar, Docker, sidecar… 🟡
  - Default por stack:
    - **A) Node + TS:** `tsx watch` en dev + binario standalone (Node `--experimental-strip-types` o compilado con `tsup`) en prod.
    - **B) Python:** `uvicorn --reload` en dev + `uvicorn` (workers) en prod. (Sin Docker por ahora — corremos local.)
    - **C) Java + Spring Boot:** `mvn spring-boot:run` en dev + fat jar (`java -jar app.jar`) en prod. Sin Docker por ahora.
- **Q16. Puerto y base path?** 🟡
  - Default: **puerto `8787`**, base path raíz (sin prefijo `/v1` por ahora).

### Cross-cutting

- **Q17. ¿Validación de input/response?** 🟡
  - Default por stack:
    - **A) Node + TS:** **Zod** — un solo schema, runtime + derivar tipos TS + alimentar OpenAPI (con `@hono/zod-openapi`).
    - **B) Python:** **Pydantic v2** — mismo rol (model + validación + OpenAPI nativo en FastAPI).
    - **C) Java + Spring Boot:** **jakarta.validation** (Bean Validation, anotaciones como `@NotNull`, `@Size`) + **springdoc-openapi** integra las anotaciones al spec.
- **Q18. ¿Logging?** 🟡
  - Default por stack:
    - **A) Node + TS:** **Pino** structured JSON a stdout.
    - **B) Python:** **Loguru** (o structlog) JSON a stdout.
    - **C) Java + Spring Boot:** **Logback + SLF4J** (default de `spring-boot-starter-logging`), JSON via `logstash-logback-encoder` si queremos parseo centralizado.
- **Q19. ¿Formato de errores?** 🟡
  - Default: **envelope propio simple** `{ error: { code, message, details? } }` con status HTTP semántico. (RFC 7807 es excelente pero overkill para local — queda como upgrade path.)
- **Q20. ¿Testing?** 🟡
  - Default por stack:
    - **A) Node + TS:** **Vitest** — unit + integration con `mockFetch`/supertest-style helpers.
    - **B) Python:** **pytest** — unit + integration, con `pytest-asyncio` para los endpoints async.
    - **C) Java + Spring Boot:** **JUnit 5 + Mockito + Spring Boot Test** (integration con `@SpringBootTest` y `MockMvc`).
- **Q21. ¿Lint/format?** 🟡
  - Default por stack:
    - **A) Node + TS:** **Biome** (reemplaza ESLint+Prettier, una config, ultra-rápido).
    - **B) Python:** **Ruff** (reemplaza flake8/black/isort, una tool, ultra-rápido).
    - **C) Java + Spring Boot:** **Spotless** (formateo via Google Java Format o Palantir) + **SpotBugs** (análisis estático). Alternativa: **Checkstyle + PMD**.

---

## 🎯 Snapshot — defaults consolidados

> Estado **provisional al cierre del requirements-gathering**. Las dos propuestas traerán sus propios snapshots justificados; este queda como referencia de qué era firme.

| Capa               | Stack A: Node + TS          | Stack B: Python                | Stack C: Java + Spring Boot         |
| ------------------ | ---------------------------- | ------------------------------ | ------------------------------------ |
| Lenguaje           | TypeScript                   | Python 3.12+                   | Java 21 (LTS)                        |
| Runtime            | Node LTS (Bun opcional)      | CPython (uv para env mgmt)     | JVM (GraalVM native opcional)        |
| Framework HTTP     | Hono + `@hono/zod-openapi`   | FastAPI                        | Spring Boot 3 + springdoc-openapi    |
| ORM                | Drizzle (driver `libsql`)    | SQLAlchemy 2.0                 | Spring Data JPA (Hibernate)          |
| DB                 | SQLite (`api_health.db`)     | SQLite (`api_health.db`)       | SQLite (`api_health.db`)             |
| Migraciones        | Drizzle Kit                  | Alembic                        | Flyway                               |
| Validación         | Zod                          | Pydantic v2                    | jakarta.validation (Bean Validation) |
| OpenAPI            | code-first, spec 3.0         | code-first, spec 3.0           | code-first, spec 3.0                 |
| Frontend codegen   | `openapi-typescript`         | `openapi-typescript` (mismo)   | `openapi-typescript` (mismo)         |
| Auth               | API key (`X-API-Key`)        | API key (`X-API-Key`)          | API key (`X-API-Key`) vía filter     |
| Layout             | Monolito modular             | Monolito modular               | Monolito modular (paquetes por módulo) |
| Datos gym          | Sync one-way desde `gym_tracker.db` | Igual                    | Sync via JDBC                        |
| Read/Write         | Read-only v1                 | Read-only v1                   | Read-only v1                         |
| Puerto             | `8787`                       | `8787` (distinto si corren juntos) | `8787` (distinto si corren juntos) |
| Logging            | Pino (JSON)                  | Loguru (JSON)                  | Logback + SLF4J (JSON)               |
| Errors             | Envelope propio              | Envelope propio                | Envelope propio + `@ControllerAdvice` |
| Tests              | Vitest                       | pytest                         | JUnit 5 + Mockito + Spring Boot Test |
| Lint/format        | Biome                        | Ruff                           | Spotless + SpotBugs                  |

> **Las tres propuestas comparten:** DB, read-only v1, base path, estructura modular, codegen para el FE, puerto y auth. **Stack C usa el mismo motor de DB y misma auth que A y B** — la diferencia es puramente del lado del lenguaje/ecosistema.

---

## ➡️ Próximo paso

Voy a escribir **TRES archivos** en este repo, al mismo nivel de profundidad:

- `architecture-proposal.node-ts.md` — propuesta stack A.
- `architecture-proposal.python.md` — propuesta stack B.
- `architecture-proposal.java-spring.md` — propuesta stack C.

Los tres cubren los 7 puntos del plan inicial (stack justificado, estructura de carpetas, flujo OpenAPI, esquema DB, plan de sync, endpoints iniciales, setup commands).

Decime cómo querés avanzar:
1. **Arrancar ya con los tres** — orden propuesto: Node+TS (más maduro en este proyecto) → Python → Java/Spring.
2. **Esperar a que respondas las Qs pendientes** (Q3, Q7, Q8, Q9, Q10, Q11, Q12, Q13, Q14, Q15–Q21) y con esas respuestas escritas, las tres propuestas salen más ajustadas.
3. **Otra forma** que prefieras.

---

*Última actualización: 2026-07-12 — segundo pase de expansión: agregado tercer stack (Java + Spring Boot) en Q1; tablas Java para Q2 (frameworks: Spring Boot, Quarkus, Micronaut, Helidon) y Q5 (ORMs: Spring Data JPA, jOOQ, Spring Data JDBC, MyBatis, Jdbi); Q6 extendido con tabla comparativa de herramientas de migración por stack (Drizzle Kit / Alembic / Flyway); defaults de Q15/Q17/Q18/Q20/Q21 con variante Java; snapshot con columna Stack C; README actualizado con la tercera fila.*
