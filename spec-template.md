# Spec Template — API Project Bootstrap

> **Propósito:** template para escribir la spec de arranque de una API nueva. Si llenás este template y lo implementás, el resultado cumple con el [playbook de `api-architecture`](./playbook.md) (S1-S16 + S18) **por construcción**, no por copy-paste de un starter.

## Cómo usar este template

1. **Copiá este archivo** a `specs/000-bootstrap/spec.md` dentro de tu proyecto nuevo.
2. **Para cada sección**, completá el prompt o tildá la opción que aplique.
3. **Para las elecciones stack-específicas** (qué ORM, qué framework HTTP, etc.), consultá la `architecture-proposal.<stack>.md` del stack elegido — esos archivos son la **versión stack-específica de este template**.
4. **Una vez completa la spec**, usala como input para implementar el proyecto (vos, otro dev, o un agente con coding tools).
5. **Después de implementar**, validá contra el checklist al final (cada item = un estándar del playbook).

## Output esperado

Cuando la spec se implementa, el resultado es un proyecto de API con:

- `specs/000-bootstrap/spec.md` — este archivo, completado con tu dominio
- `src/` (o equivalente) con entities, controllers, services, DTOs
- `tests/` con integration tests
- DB con migraciones forward-only aplicadas
- OpenAPI spec servido en `/docs`
- README con instrucciones de setup

> **NO** se clona código de los `api-<stack>-reference/`. Esos repos son **ejemplos de output ya implementado**, no starters. Se consultan para ver "qué se siente" un proyecto que cumple el playbook, no como base.

---

## Secciones

### 1. Project identity

- **Nombre del proyecto:**
- **Propósito** (1-2 oraciones): qué problema resuelve, para quién
- **Stack elegido:** Node+TS / Python / Java+Spring
- **Justificación del stack** (link a la `architecture-proposal.<stack>.md`)

### 2. Dominio

- **Entities** (con atributos y relaciones):
  - `<Entity>`: campos, primary key, relaciones (OneToMany, ManyToOne, etc.)
  - `<Entity>`: ...
- **Business rules** (las reglas del dominio, "si X entonces Y"):
  - ...
- **Glosario** (términos específicos del dominio):
  - ...

### 3. Endpoints

Para cada recurso, listá los verbos CRUD que apliquen (S3):

- `GET /<resource>` — list, paginated, filterable
- `GET /<resource>/{id}` — get one
- `POST /<resource>` — create
- `PATCH /<resource>/{id}` — partial update
- `DELETE /<resource>/{id}` — delete
- (Skipeá los que no apliquen — el playbook no obliga a los 5)

**Endpoints especiales (siempre):**
- `GET /health` — health check, sin auth
- `GET /docs` — OpenAPI UI, sin auth

**Auth (S4):** por default, todos los endpoints excepto `/health` y `/docs` requieren `X-API-Key`. Si el proyecto justifica auth real (OAuth/JWT), documentalo acá.

### 4. Validación

Para cada input (body, query, params):
- Schema: Zod (Node/Python) / Pydantic (Python) / jakarta.validation (Java) — ver stack proposal
- Por campo: type, required, min/max, regex, enum
- Estrategia 422 (S5): "validation failed" con detalle de campos que fallaron

### 5. Error envelope

- Shape (S6): `{ error: { code: string, message: string, details?: unknown } }`
- Mapeo de status codes a códigos internos:

| Status | Code                  |
| ------ | --------------------- |
| 400    | `BAD_REQUEST`         |
| 401    | `UNAUTHORIZED`        |
| 403    | `FORBIDDEN`           |
| 404    | `NOT_FOUND`           |
| 409    | `CONFLICT`            |
| 422    | `VALIDATION_ERROR`    |
| 429    | `RATE_LIMIT_EXCEEDED` |
| 5xx    | `INTERNAL_ERROR`      |

### 6. Logging

- Formato (S7): JSON a stdout
- Redacción: qué headers/campos redactar (X-API-Key, Authorization, etc.)
- Correlation IDs: cómo se generan (req.id del HTTP logger, middleware, etc.)

### 7. CORS

- Origins permitidos (S8)
- Credentials: yes / no
- Métodos: por default GET, POST, PATCH, DELETE
- Headers: por default Content-Type, Authorization, X-API-Key

### 8. Testing

- Tipos de test (S9):
  - Unit (service-level, con mocks)
  - Integration (controller + service + DB)
  - E2E (HTTP end-to-end, proceso separado)
- Coverage target (e.g., 70% statements / 25% branches / 5% funcs)
- Qué NO testear (third-party libs, glue code trivial)

### 9. Lint / format

- Tool (S10): Biome / Ruff / Spotless — ver stack proposal
- Format on save: yes / no
- Pre-commit hook: yes / no

### 10. Base de datos

- Engine: SQLite (dev/greenfield) / PostgreSQL (prod) — ver Q4
- ORM: ver stack proposal (S11)
- Migraciones: forward-only, ver stack proposal
- Sync (S12): greenfield (skip) / producción (one-way)

### 11. Deployment

- Puerto (S13): default 8787
- Dev mode (S14): auto-reload, logs verbose
- Prod mode: bundle compilado, logs info-level
- Build target: depende del stack
- Containerización: Dockerfile (recomendado para prod)

### 12. List patterns

- Paginación (S15): offset-based, `page/limit`, default 1/20, max 100
- Response shape: `{ data: [...], pagination: { page, limit, total, has_next } }`
- Filtering (S16): qué campos son filtrables (whitelist cerrada)
- Sorting (S16): qué campos son sortables, asc/desc, default

### 13. Secrets

- Env vars (S18): listar cada una con descripción y default
- Validación al arranque: schema con Zod (o equivalente), fail-loud
- `.env` management: gitignored, `.env.example` commiteado
- Dónde vienen los secrets en dev (`.env` local) vs prod (env vars del hosting)

### 14. Open questions

- Decisiones pendientes para antes/durante la implementación
- Cosas para revisar después del MVP

---

## Validation checklist (post-implementación)

Después de implementar, validá que cada estándar esté cumplido:

- [ ] **S1** — Arquitectura en 3 capas: Controller → Service → Repository
- [ ] **S2** — OpenAPI 3.0 code-first, spec servido en `/docs`
- [ ] **S3** — Endpoints CRUD sobre cada recurso, todos desde v1
- [ ] **S4** — Auth con API key (`X-API-Key`), constant-time compare
- [ ] **S5** — Validación devuelve 422 (no 400) con detalle por campo
- [ ] **S6** — Error envelope: `{ error: { code, message, details? } }`
- [ ] **S7** — Logs JSON a stdout, con redacción de secrets
- [ ] **S8** — CORS configurable via env
- [ ] **S9** — Tests con coverage target, integration tests para endpoints
- [ ] **S10** — Un solo tool de lint+format, una sola config
- [ ] **S11** — DB con migraciones forward-only, ORM con metadata
- [ ] **S12** — DB sync strategy definida (skip para greenfield, one-way para prod)
- [ ] **S13** — Server escucha en puerto 8787
- [ ] **S14** — Dev mode con auto-reload, prod mode con bundle compilado
- [ ] **S15** — Paginación con `page/limit`, response shape `{ data, pagination }`
- [ ] **S16** — Filter/sort whitelist, sin nombres de columna libres
- [ ] **S18** — Secrets validados al arranque, `.env` gitignored, `.env.example` commiteado

---

## See also

- [`playbook.md`](./playbook.md) — los 17 estándares agnósticos (S1-S16 + S18)
- [`node-ts/architecture-proposal.node-ts.md`](./node-ts/architecture-proposal.node-ts.md) — stack-specific spec template para Node+TS
- [`python/architecture-proposal.python.md`](./python/architecture-proposal.python.md) — idem Python
- [`java-spring/architecture-proposal.java-spring.md`](./java-spring/architecture-proposal.java-spring.md) — idem Java+Spring
- [`architecture-decisions.md`](./architecture-decisions.md) — Q1-Q26, el "por qué" de cada estándar
- **`api-node-reference/`** _(repo hermano)_ — ejemplo de output ya implementado para Node+TS
- **`api-python-reference/`** _(a crear)_ — ejemplo de output para Python
- **`api-java-reference/`** _(a crear)_ — ejemplo de output para Java+Spring
