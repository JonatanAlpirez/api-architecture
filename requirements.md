# Requisitos y Preguntas Abiertas — api-architecture

> **Estado:** levantamiento de requisitos (previo a propuesta de arquitectura).
> Doc vivo: se actualiza conforme aterricen decisiones.
> **Próximo entregable en este repo:** `architecture-proposal.md` (un solo documento, con una opción concreta de stack + capas + flujo OpenAPI + DB + integración con `gym_training-data/`).

---

## ✅ Requisitos confirmados por Jonatan

| ID   | Requisito                                                                                                | Notas                                                                                                       |
| ---- | -------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| R1   | API documentada con **Swagger / OpenAPI 3.x**                                                            | El FE debe poder generar servicios y modelos automáticamente a partir del spec.                             |
| R2   | Arquitectura en capas **Controller → Service → Repository**                                              | Origen Spring Boot/Java. Abierto a equivalente moderno en cualquier stack.                                  |
| R3   | Stack **moderno y estandarizado**                                                                        | Sin preferencia rígida por ecosistema; evaluamos opciones.                                                  |
| R4   | Base de datos **relacional**                                                                             | —                                                                                                           |
| R5   | Reutilizar los datos de `~/Documents/gym_training-data/`                                                | Hoy usa SQLite; la DB destino de la API puede ser otra (PostgreSQL, etc.).                                  |
| R6   | Consumo **local**                                                                                        | Primer cliente: `main-dashboard`. Más adelante podrían sumarse otros módulos.                               |
| R7   | Antes de la propuesta: preguntar dudas / recomendaciones + documentar reqs/pendientes                     | Hecho en este doc.                                                                                          |

---

## 📦 Contexto heredado (decisiones previas)

> Estas se tomaron el **2026-07-08** para el (ya cancelado) módulo backend de `main-dashboard`. Sirven como **punto de partida** — conviene confirmar si siguen vigentes para el alcance nuevo (que ahora vive en `api-architecture/`).

- **ORM:** **Drizzle** (por-módulo), driver **`libsql`**.
- **Layering:** **Controller → Service → Repository, siempre** (no opt-in).
- Justificación original: `main-dashboard/PLAN-01.md §11.7.1` y §11.7.2 — ahora superseded.

→ Pregunta explícita más abajo (Q5) para confirmar o reemplazar.

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

> **Formato:** marco cada pregunta con default sugerido entre paréntesis. Si te copa el default, contestá "default" y avanzo. Si querés cambiar, decime y reemplazo.

### Stack / runtime

- **Q1.** **Lenguaje / runtime preferido?**
  - Default: **Node + TypeScript** (consistente con `main-dashboard`, ecosistema de OpenAPI maduro, mejor para alinear tipado FE↔BE).
  - Alternativas: Go, Python (FastAPI — fue el plan original cancelado), Java/Spring Boot, Bun, Deno.
- **Q2.** **Framework HTTP?**
  - Default: **Hono** (moderno, ultra-liviano, edge-friendly, OpenAPI vía `@hono/zod-openapi` o `hono-openapi`).
  - Alternativas: Fastify, NestJS (más opinionado, "Spring Boot-like"), Elysia, FastAPI, Spring Boot, Gin, Fiber.
- **Q3.** ¿Exploramos Bun/Deno como runtime o nos quedamos en Node LTS?

### Base de datos y ORM

- **Q4.** **DB destino?**
  - Default: **SQLite via `libsql`** (consistente con `gym_training-data/`, cero fricción de migración, sirve para local; usar archivo separado `api_health.db` para no tocar el original).
  - Alternativas: PostgreSQL (más serio, mejor para futuro multi-módulo), MySQL/MariaDB.
- **Q5.** **ORM — confirmar Drizzle o reevaluar?**
  - Default: **Drizzle** (heredado de la decisión 2026-07-08, typed-first, migraciones con Drizzle Kit, liviano).
  - Alternativas: Prisma (más fácil pero más opinated), Kysely (query builder puro), MikroORM.
- **Q6.** **Migraciones?**
  - Default: **Drizzle Kit** (asume Q5=Drizzle). Genera SQL a partir de schema TS, idempotente.
  - Alternativas: golang-migrate, Flyway, Alembic, raw SQL versionado.

### Contrato de API (Swagger / OpenAPI)

- **Q7.** **Spec-first o code-first?**
  - Default: **Code-first** (anoto controllers/types, genero OpenAPI desde el código con `@hono/zod-openapi` o equivalente del stack final). Más rápido de iterar, menos archivos para mantener sincronizados.
  - Alternativa: spec-first (escribo `openapi.yaml` primero, genero tipos/validators desde el spec).
- **Q8.** **Versión del spec?** 3.0 (compatible con `openapi-typescript` que ya usa `main-dashboard`) o 3.1 (más moderno, JSON Schema nativo).
  - Default: **3.0** (reuso directo del codegen del FE sin reconfigurar nada).
- **Q9.** **Codegen para el frontend?**
  - Default: **mantener `openapi-typescript`** (ya en uso en `main-dashboard`).
  - Alternativas: `orval` (genera hooks de React Query además de tipos), `openapi-generator` (más pesado, multi-lenguaje).

### Integración con `gym_training-data/`

- **Q10.** **¿Cómo accede la API a los datos?**
  - Default: la API **abre su propio SQLite** (`api_health.db`) y los datos se **sincronizan** desde `gym_training-data/DB/gym_tracker.db` mediante un comando/script (`pnpm sync:health` o similar). La API no toca el archivo fuente.
  - Alternativas: (a) la API lee directo el SQLite de `gym_training-data/` (sin sync, pero acopla la API al filesystem del data warehouse); (b) se hace una **migración one-shot** y `gym_training-data/` queda solo como fuente histórica para re-imports.
- **Q11.** **¿La API solo expone `health` o ya planeamos otros módulos desde el inicio?**
  - Default: arranca **solo con `health`** (workouts/exercises). Cuando definamos `finance` o `reminders` como APIs, agregamos módulos a la misma API o las separamos — decisión arquitectónica que aterriza en la propuesta.
- **Q12.** **¿Endpoints de escritura (POST/PUT/DELETE)?** Por ahora la fuente de verdad es el CSV de Strong + script de import, así que la API podría ser **read-only**.
  - Default: **read-only en v1**. Escritura queda para v2 si surge necesidad (p. ej. registrar workouts desde la app).

### Auth / multi-tenancy

- **Q13.** **Auth?** Consumo local → opciones: (a) **sin auth** (asumimos que nadie más llega al puerto); (b) **API key estática** en header (defensa en profundidad barata); (c) **JWT con login simple** (overkill probable para local).
  - Default: **(b) API key estática en header `X-API-Key`** — cuesta 5 líneas, evita sustos si el puerto queda expuesto por accidente.
- **Q14.** **¿Una sola API para todos los módulos o un servicio por módulo?**
  - Default: **monolito modular** (`/health/...`, `/finance/...`, `/reminders/...`) por ahora. Separamos en microservicios solo si la complejidad lo pide.

### Runtime / deployment

- **Q15.** **¿Cómo corre?** Proceso bare (`node`/`tsx` watch), Docker, Docker Compose, sidecar…
  - Default: **proceso bare con `tsx watch` en dev** + **binario standalone (Node `--experimental-strip-types` o compilado con `tsup`) en "producción"**. Sin Docker por ahora — corremos local.
- **Q16.** **Puerto y base path?**
  - Default: **puerto `8787`**, base path raíz (sin prefijo `/v1` por ahora — agregamos cuando haya v2).
  - Si te suena mal el puerto u querés path versionado desde el día 1, decime.

### Cross-cutting

- **Q17.** **Validación de input/response?**
  - Default: **Zod** (si vamos TS) — un solo schema, sirve para runtime + derivar tipos TS + alimentar OpenAPI (con `@hono/zod-openapi`).
- **Q18.** **Logging?**
  - Default: **Pino** structured JSON a stdout. Liviano, rápido, parseable.
- **Q19.** **Formato de errores?**
  - Default: **envelope propio simple** `{ error: { code, message, details? } }` con status HTTP semántico. (RFC 7807 es excelente pero overkill para local — lo dejo como upgrade path.)
- **Q20.** **Testing?**
  - Default: **Vitest** (asume TS). Unit + integration con `mockFetch`/supertest-style helpers. E2E opcional con `playwright` cuando haya UI tests cruzados.
- **Q21.** **Lint/format?**
  - Default: **Biome** (reemplaza ESLint+Prettier, una config, ultra-rápido). Si preferís mantener ESLint+Prettier, también funciona.

---

## 🎯 Resumen de defaults propuestos (snapshot)

> Esto es lo que **propongo usar como punto de partida** para la `architecture-proposal.md`. Si nada te genera ruido, confirmá y arranco la propuesta con este stack. Si algo te chirría, lo cambiamos antes.

| Capa               | Default                                                  |
| ------------------ | -------------------------------------------------------- |
| Lenguaje           | TypeScript                                               |
| Runtime            | Node LTS (con Bun como experimento futuro)               |
| Framework HTTP     | Hono + `@hono/zod-openapi`                               |
| ORM                | Drizzle (driver `libsql`)                                |
| DB                 | SQLite (`api_health.db`, separada de `gym_tracker.db`)  |
| Migraciones        | Drizzle Kit                                              |
| Validación         | Zod                                                      |
| OpenAPI            | code-first, spec 3.0                                     |
| Frontend codegen   | `openapi-typescript` (sin cambios en `main-dashboard`)   |
| Auth               | API key estática (`X-API-Key`)                           |
| Layout             | Monolito modular (`/health/...`, etc.)                   |
| Datos gym          | Sync one-way desde `gym_training-data/DB/gym_tracker.db` |
| Read/Write         | Read-only en v1                                          |
| Puerto             | `8787`                                                   |
| Logging            | Pino (JSON a stdout)                                     |
| Errors             | Envelope propio (`{ error: { code, message, details } }`) |
| Tests              | Vitest                                                   |
| Lint/format        | Biome                                                    |

---

## ➡️ Próximo paso

Tan pronto respondas las preguntas (o digas "defaults están bien, arranca"), escribo **`architecture-proposal.md`** en esta carpeta con:

1. **Stack concreto** justificado (por qué cada elección vs alternativas).
2. **Estructura de carpetas** del proyecto backend (`src/` con controllers, services, repositories, schemas, db).
3. **Flujo OpenAPI** de punta a punta (cómo se genera el spec, cómo se sirve Swagger UI, cómo el FE hace codegen).
4. **Esquema de DB** mapeado 1:1 desde `gym_training-data/` (entidades + relaciones).
5. **Plan de sync** entre `gym_tracker.db` (fuente) y `api_health.db` (destino de la API).
6. **Lista de endpoints** inicial (read-only) con request/response shape.
7. **Setup commands** (instalación, dev, sync, build, start).

---

*Última actualización: 2026-07-11 — auditoría de `gym_training-data/` y consolidación de preguntas abiertas.*