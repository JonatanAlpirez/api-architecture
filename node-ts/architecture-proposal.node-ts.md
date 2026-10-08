# Architecture Proposal — Node + TypeScript (stack-specific spec template)

> **Rol:** stack-specific guidance para usar [`spec-template.md`](../spec-template.md) con **Node + TypeScript** (NestJS + MikroORM + Zod). Llenás el spec template con tu dominio, después consultás este archivo para saber qué tools, versiones y patterns usar para cada sección.
>
> **No es un code template.** El código de referencia vive en [`api-node-reference`](https://github.com/JonatanAlpirez/api-node-reference) _(repo hermano)_. Acá hay links a los archivos relevantes para que veas el patrón aplicado end-to-end.
>
> Las decisiones del "por qué" referenciadas viven en [`architecture-decisions.md`](../architecture-decisions.md) (Q1-Q26). Los estándares agnósticos viven en [`playbook.md`](../playbook.md) (S1-S16 + S18).

## Stack baseline

| Capa | Decisión | Versión | Notas |
| --- | --- | --- | --- |
| Lenguaje | TypeScript | 5.4+ | `target: ES2022`, `module: ESNext`, strict mode |
| Runtime | Node.js | 22.x LTS | LTS estable, soporte maduro de NestJS y MikroORM |
| Framework HTTP | NestJS | 10.x | Opinado (matchea S1 Controller → Service → Repository), DI nativo |
| ORM | MikroORM | 6.x | Data-mapper, identity map, UnitOfWork |
| Driver DB | `@mikro-orm/sqlite` | 6.x | Usa `better-sqlite3` por debajo |
| Validación | Zod + `nestjs-zod` | 3.x | Single source: schema = tipos TS + runtime + OpenAPI |
| OpenAPI | `@nestjs/swagger` | 7.x | Code-first, spec 3.0 |
| Tests | Vitest + supertest | 1.x / latest | `unplugin-swc` para decorator metadata (gotcha #4) |
| Logging | `nestjs-pino` + pino | 8.x | JSON a stdout, redaction built-in |
| Lint/format | Biome | 1.x | Reemplaza ESLint+Prettier, una config |
| Build | `tsup` | latest | Build rápido a ESM |
| Migraciones | MikroORM Migrator | (via CLI) | Forward-only (S11) |
| Package manager | pnpm | latest | Rápido, monorepo-friendly |

---

## Mapping a las secciones del spec-template

### §1-2. Project identity + Dominio

**No hay tooling Node-specific.** Completá el spec con tu dominio (nombre, propósito, entities, business rules). Las decisiones de modelado son tuyas; este stack no impone nada acá.

---

### §3. Endpoints (S3, S4)

- **Controller pattern:** NestJS con `@Controller('resources')` (un controller por recurso).
- **CRUD verbs:** los 5 endpoints estándar. `POST` con `@HttpCode(201)`, `DELETE` con `@HttpCode(204)`, `GET` y `PATCH` con default 200.
- **Auth (S4):** `@UseGuards(ApiKeyGuard)` a nivel de controller (no de cada método). El guard valida `X-API-Key` con `timingSafeEqual`.
- **ID parameter:** `@Param('id', ParseIntPipe) id: number` — el entity usa autoincrement `number`, el pipe convierte el string del path. IDs no-numéricos devuelven 400 limpio.
- **Documentación OpenAPI:** `@ApiOperation({ summary: '...' })` en cada método, `@ApiSecurity('X-API-Key')` a nivel de controller.

**Worked example:** [`api-node-reference/src/modules/resource/resource.controller.ts`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/src/modules/resource/resource.controller.ts) — 5 endpoints sobre `/resources` siguiendo este patrón.

---

### §4. Validación (S5)

- **Tool:** Zod 3.x como single source para runtime + tipos TS + OpenAPI metadata.
- **Pattern:** definís un schema Zod, lo wrapeás con `createZodDto(schema)` de `nestjs-zod` para tener una clase con metadata.
- **422 strategy (S5, gotcha #2):** el default de `nestjs-zod` (`createZodValidationPipe()`) devuelve **400**. El playbook dice 422. Solución: custom `ZodValidationPipe` que wrappea el default con un `exceptionFactory` que lanza `UnprocessableEntityException`.
- **Por qué Zod vs class-validator:** Zod = single source (runtime + tipos + OpenAPI), class-validator requiere definir tipos TS aparte y duplica.
- **Q17:** ver [`architecture-decisions.md`](../architecture-decisions.md) — se evaluó Hono/Zod vs class-validator, se eligió nestjs-zod.

**Worked example:** [`api-node-reference/src/common/pipes/zod-validation.pipe.ts`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/src/common/pipes/zod-validation.pipe.ts) (el custom pipe) + [`src/modules/resource/dto/`](https://github.com/JonatanAlpirez/api-node-reference/tree/main/src/modules/resource/dto) (los 3 DTOs con schemas Zod).

**Checklist S5:** ✓ custom pipe para 422, ✓ field-level details en el envelope, ✓ Zod como single source.

---

### §5. Error envelope (S6)

- **Shape:** `{ error: { code: string, message: string, details?: unknown } }`.
- **Implementación:** `HttpExceptionFilter` global con `@Catch()` (sin args = captura todo). Mapea status codes a códigos internos.
- **Status → code mapping:** ver tabla en [`spec-template.md` §5](../spec-template.md#5-error-envelope).
- **Zod errors:** detectados por `instanceof ZodError`, formateados con `zod-validation-error` para `details`.

**Worked example:** [`api-node-reference/src/common/filters/http-exception.filter.ts`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/src/common/filters/http-exception.filter.ts) — 95 líneas, captura todo y normaliza al envelope.

**Checklist S6:** ✓ envelope consistente, ✓ status code mapping, ✓ Zod errors formateados.

---

### §6. Logging (S7)

- **Stack:** `nestjs-pino` (integración) + `pino` (logger) + `pino-pretty` (formateo en dev).
- **Config:** `LoggerModule.forRoot(...)` en `app.module.ts` con `pinoHttp.level` (de `LOG_LEVEL` env) y `transport.pino-pretty` solo en dev.
- **Buffer logs:** `NestFactory.create(AppModule, { bufferLogs: true })` + `app.useLogger(app.get(Logger))` — sin esto, los logs de NestJS no salen por pino.
- **Redaction built-in:** `redact: { paths: ['req.headers["x-api-key"]', 'req.headers.authorization'], censor: '***REDACTED***' }`.
- **JSON en prod, pretty en dev:** `pino-pretty` solo cuando `NODE_ENV=development`.

**Worked example:** [`api-node-reference/src/app.module.ts`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/src/app.module.ts) (LoggerModule.forRoot).

**Checklist S7:** ✓ JSON a stdout, ✓ redaction de secrets, ✓ pretty en dev.

---

### §7. CORS (S8)

- **Built-in:** `app.enableCors({ origin: env.FRONTEND_ORIGIN, credentials: true })` en `main.ts`.
- **Origen configurable:** env var `FRONTEND_ORIGIN` (default `http://localhost:5173` para Vite, ajustá a `:3000` para Next.js).

**Worked example:** [`api-node-reference/src/main.ts:17-20`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/src/main.ts#L17).

**Checklist S8:** ✓ configurable via env, ✓ credentials enabled (si el FE necesita cookies).

---

### §8. Testing (S9)

- **Stack:** Vitest 1.x + supertest.
- **In-memory DB:** tests usan `dbName: ':memory:'` + `generator.createSchema()` (más rápido que `migration:up`).
- **`allowGlobalContext: true`:** permite a MikroORM usar el contexto global sin inyectar `EntityManager` explícito.
- **Re-aplicar cross-cutting:** `Test.createTestingModule(...).compile()` no copia los global pipes/filters del `AppModule` — hay que re-aplicarlos manualmente en `beforeEach` (`useGlobalPipes(new ZodValidationPipe())`, `useGlobalFilters(new HttpExceptionFilter())`).
- **Gotcha #4 (esbuild + decorators):** esbuild (default de Vitest) no emite `decoratorMetadata` que NestJS necesita para DI. Solución: `unplugin-swc` en `vitest.config.ts`.
- **API key de tests:** 39+ chars, set en `vitest.setup.ts` antes de cualquier import.

**Worked example:** [`api-node-reference/src/modules/resource/resource.controller.spec.ts`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/src/modules/resource/resource.controller.spec.ts) (4 integration tests).

**Checklist S9:** ✓ integration tests con supertest, ✓ in-memory DB por test, ✓ unplugin-swc para decorator metadata.

---

### §9. Lint / format (S10)

- **Tool:** Biome 1.x — una sola tool, una sola config (`biome.json`).
- **Reemplaza:** ESLint + Prettier (2 tools, 2 configs, 2 lockfiles).
- **Qué cubre:** lint (reglas `recommended` + selected), format, organizeImports.
- **Configuración:** 30 líneas en `biome.json`. Indent 2 spaces, lineWidth 100, single quotes, semicolons asNeeded, trailing commas all.

**Worked example:** [`api-node-reference/biome.json`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/biome.json).

**Checklist S10:** ✓ 1 tool, 1 config, 1 lockfile, ✓ lint + format + organizeImports.

---

### §10. Base de datos (S11, S12)

- **ORM:** MikroORM 6.x (decisión Q5/Q6).
- **Dev engine:** SQLite via `@mikro-orm/sqlite` + `better-sqlite3`. DB file en `data/api.db` (gitignored).
- **Prod engine:** PostgreSQL via `@mikro-orm/postgresql`. Cambio de 3 líneas en `mikro-orm.config.ts` (driver + dbName).
- **Migrations (S11):** MikroORM Migrator, forward-only.
  - Generar: `npx mikro-orm migration:create` (después de modificar un entity).
  - Aplicar: `npx mikro-orm migration:up` o `npm run db:migrate` (script discoverable).
  - Tabla interna `mikro_orm_migrations` trackea cuáles corrieron.
- **Sync (S12):**
  - **Greenfield:** skip (la DB se inicializa con migraciones vacías y los datos nacen vía `POST`).
  - **Existing source DB:** sync one-way vía script `tsx src/scripts/sync.ts` (idempotente, re-ejecutable). Out of scope para v1 si es greenfield.
- **Q4 / Q5 / Q6:** ver [`architecture-decisions.md`](../architecture-decisions.md) — por qué SQLite/Postgres, por qué MikroORM (no Drizzle/Prisma/TypeORM).

**Worked example:** [`api-node-reference/mikro-orm.config.ts`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/mikro-orm.config.ts) + [`src/database/`](https://github.com/JonatanAlpirez/api-node-reference/tree/main/src/database) (módulo + migrations).

**Checklist S11/S12:** ✓ forward-only migrations, ✓ DB sync strategy definida, ✓ DB inicializada con `db:migrate`.

---

### §11. Deployment (S13, S14)

- **Puerto (S13):** default `8787`, configurable via `PORT` env. (El "8" es por consistencia con el resto de nuestros servicios.)
- **Dev mode (S14):** `npm run dev` corre `nest start --watch` — auto-reload, logs verbose.
- **Dev alternativo (gotcha #8):** `npm run dev:tsx` corre `tsx watch --env-file=.env src/main.ts` — útil si `nest start --watch` falla por decorator metadata.
- **Build:** `tsup src/main.ts --format esm --target node22 --clean` → `dist/main.js` (ESM, ~chico).
- **Prod start:** `node --env-file=.env dist/main.js`.
- **CI + Dockerfile:** pendientes (ver "Próximos pasos" abajo).

**Worked example:** [`api-node-reference/package.json`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/package.json) (scripts) + `tsup` config en el mismo.

**Checklist S13/S14:** ✓ puerto 8787, ✓ dev con auto-reload, ✓ prod con bundle compilado.

---

### §12. List patterns (S15, S16)

- **Pagination (S15):** offset-based, `page/limit` (default 1/20, max 100). Coerce con `z.coerce.number()`.
- **Response shape:** `{ data: T[], pagination: { page, limit, total, has_next } }`.
- **Filtering (S16):** whitelist cerrada con `z.enum([...])`. Ejemplo: `status: z.enum(['active', 'archived']).optional()`.
- **Sorting (S16):** whitelist en el DTO + mapping en el service.
  - DTO: `sort: z.enum(['created_at', '-created_at', 'name', '-name'])` (snake_case externo).
  - Service: `parseSort()` mapea snake_case → camelCase property del entity (`created_at` → `createdAt`), previene SQL injection por columnas arbitrarias.
- **`has_next`:** calculado como `page * limit < total` en el service.

**Worked example:** [`api-node-reference/src/modules/resource/dto/filter-resource.dto.ts`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/src/modules/resource/dto/filter-resource.dto.ts) (DTO) + [`src/modules/resource/resource.service.ts:parseSort()`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/src/modules/resource/resource.service.ts).

**Checklist S15/S16:** ✓ offset-based pagination, ✓ whitelist cerrada para filter/sort, ✓ mapeo snake_case → camelCase.

---

### §13. Secrets (S18)

- **Zod schema:** `envSchema = z.object({ PORT, FRONTEND_ORIGIN, API_KEY, DATABASE_URL, NODE_ENV, LOG_LEVEL })` con `z.coerce.number()` y `z.enum(...)` para tipos.
- **Lazy via Proxy:** `env` se exporta como un `Proxy` que evalúa `envSchema.safeParse(process.env)` solo en el primer acceso. Esto permite a los tests setear `process.env` antes de que el módulo se importe.
- **Fail-loud:** si falta un env var requerido o es inválido, `process.exit(1)` con `console.error(...)` de los field errors. **No arranca con config inválida.**
- **`dotenv/config` import en `main.ts`:** (gotcha #1) `nest start --watch` no carga `.env` automáticamente, hay que importarlo manualmente al boot.
- **`.env` gitignored, `.env.example` commiteado:** la secret real nunca va al repo.

**Worked example:** [`api-node-reference/src/config/env.ts`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/src/config/env.ts) (54 líneas, Proxy + lazy).

**Checklist S18:** ✓ Zod schema, ✓ fail-loud al arranque, ✓ dotenv import, ✓ .env gitignored.

---

### §14. Open questions

No hay tooling Node-specific acá. Si el proyecto tiene preguntas abiertas, listalas en el spec y resolvelas antes/durante implementación.

---

## Por qué este stack (vs Python / Java)

### vs Python (FastAPI)

**Ganamos con Node:**
- TS type system end-to-end (más estricto que Python type hints; errores en compile-time).
- `@nestjs/mikro-orm` adapter oficial — integración más pulida que SQLAlchemy + FastAPI wrappers.
- Ecosystem Node más maduro para OpenAPI codegen (NestJS genera spec out-of-the-box; FastAPI también pero requiere más setup con Pydantic).
- Async nativo con Promises/async-await (mismo modelo mental que JS del FE).

**Perdés con Node:**
- NestJS es más verboso que FastAPI (decorators en todas partes, módulos explícitos).
- Compilación step (TS → JS) suma fricción vs Python que se ejecuta directo.
- Cold start un poco peor que FastAPI (Node también arranca rápido, pero con más overhead de módulos que uvicorn).

### vs Java + Spring Boot

**Ganamos con Node:**
- Arranque significativamente más rápido (NestJS en ~1s vs Spring Boot en ~5-10s).
- Mucho menos boilerplate (no hay `pom.xml`, application classes, autowire annotations).
- TS es más flexible que Java (structural typing, decorators opcionales).
- Ecosystem más liviano (sin JVM, sin classpath hell).

**Perdés con Node:**
- TS no es tan type-safe como Java en compile-time (más reliance en runtime checks).
- NestJS no tiene el mismo ecosistema enterprise que Spring (security distribuida, transactions distribuidas, etc.) — v1 no lo necesita.
- Spring Boot + Hibernate con Postgres es el camino más "production-tested" del mercado; Node + MikroORM es más nuevo y con menos battle-testing a escala.

---

## Stack-specific gotchas (cross-reference)

10 gotchas documentados, todos relevantes para este stack. Ver el detalle en [`api-node-reference/.docs/WALKTHROUGH.md` §5](https://github.com/JonatanAlpirez/api-node-reference/blob/main/.docs/WALKTHROUGH.md).

Top 5 (los que rompen tests o runtime si no los conocés):

1. **`import 'dotenv/config'` en `main.ts`** — `nest start --watch` no carga `.env` automáticamente.
2. **Custom `ZodValidationPipe` para 422** — default de `nestjs-zod` es 400.
3. **Proxy lazy en `config/env.ts`** — sin esto, los tests fallan al import por env vars no seteadas.
4. **`unplugin-swc` en `vitest.config.ts`** — esbuild (default Vitest) no emite `decoratorMetadata` que NestJS necesita.
5. **`ParseIntPipe` en `:id`** — el entity es `number`, sin pipe MikroORM hace type coercion raro.

---

## Próximos pasos

Para cerrar el reframe del playbook completo, falta:

- Crear `api-python-reference` (mismo nivel de coverage que `api-node-reference` para Python/FastAPI).
- Crear `api-java-reference` (idem para Java+Spring Boot 3).
- CI + Dockerfile en `api-node-reference` (los 2 pendientes del README).
- Documentar Q27 (Postgres hosting options) — la decisión de cuál Postgres para prod.

Para el stack Node en sí, este proposal + `api-node-reference` ya cubren 17/17 estándares S1-S16 + S18.
