# Architecture Proposal — Node + TypeScript

> **Estado:** propuesta cerrada — cubre los 7 puntos del plan inicial (stack, estructura, OpenAPI, DB, sync, endpoints, setup) + tradeoffs vs Python y Java Spring Boot. Próximo paso: validar el esqueleto en código y, si OK, usar como template para las propuestas Python y Java.
>
> Todas las decisiones referenciadas viven en [`architecture-decisions.md`](./architecture-decisions.md). Esta propuesta **asume** que esas decisiones están cerradas y solo aterriza nombres concretos, paths y código.

---

## 1. Stack justificado

| Capa | Decisión | Versión target | Por qué |
| --- | --- | --- | --- |
| Lenguaje | **TypeScript** | 5.4+, target ES2022 | Strict mode; mejor autocompletado y type-safety que JS puro |
| Runtime | **Node.js** | 22.x LTS | LTS estable, soporte maduro de NestJS y MikroORM |
| Framework HTTP | **NestJS** | 10.x | Opinado (matchea R2 Controller → Service → Repository); DI nativo; ecosystem NestJS |
| ORM | **MikroORM** | 6.x | Data-mapper, identity map, UnitOfWork — paridad filosófica con SQLAlchemy |
| Driver DB | `@mikro-orm/sqlite` | (via MikroORM) | Usa `better-sqlite3` por debajo |
| Validación | **Zod** | 3.x | Single source: schema = tipos TS + runtime + OpenAPI |
| OpenAPI integration | `nestjs-zod` + `@nestjs/swagger` | (latest) | Genera spec desde controllers + Zod schemas automáticamente |
| Tests | **Vitest** | 1.x | Mejor TS support que Jest; mocks built-in |
| HTTP testing | `supertest` | (latest) | Standard para testing de endpoints NestJS |
| Logging | **Pino** | 8.x | Ultra-rápido, JSON nativo, child loggers, redaction built-in |
| HTTP logger | `pino-http` | (latest) | Middleware Pino para request/response logging |
| Lint/format | **Biome** | 1.x | Reemplaza ESLint+Prettier; una config, ultra-rápido |
| Build | `tsup` | (latest) | Build rápido a ESM/CJS; alternativa: `tsc` + `tsc-alias` |
| Migraciones | **MikroORM Migrator** | (via MikroORM CLI) | CLI `npx mikro-orm migrator:generate` |
| CORS | Built-in `app.enableCors()` | (NestJS) | Config por env var `FRONTEND_ORIGIN` |
| Package manager | `pnpm` | (latest) | Rápido, monorepo-friendly si el FE lo necesita a futuro |

**Por qué este stack sobre las alternativas evaluadas** (ver Q2/Q4/Q5/Q17/Q21 en `architecture-decisions.md`):

- **NestJS sobre Hono/Fastify/Express**: estructura opinionated (DI, módulos, decorators) que matchea R2 y el background Spring/Java de Jonatan.
- **MikroORM sobre Drizzle/Prisma/TypeORM**: paridad filosófica con SQLAlchemy + `@nestjs/mikro-orm` oficial.
- **Zod sobre class-validator/Joi**: single source para runtime + tipos TS + OpenAPI.
- **Vitest sobre Jest**: mejor TS support out-of-the-box, sin config extra.
- **Biome sobre ESLint+Prettier**: una tool, una config, ultra-rápido.

---

## 2. Estructura de carpetas

Monolito modular — un solo deployable, módulos independientes entre sí (bajo acoplamiento, alta cohesión).

```
api-node/
├── src/
│   ├── main.ts                           # bootstrap: app = await NestFactory.create(); app.enableCors(); app.listen(8787)
│   ├── app.module.ts                     # módulo raíz: importa ConfigModule, DatabaseModule, ResourceModule, HealthModule
│   │
│   ├── config/
│   │   └── env.ts                        # Zod schema para process.env (PORT, FRONTEND_ORIGIN, API_KEY, DATABASE_URL)
│   │
│   ├── common/
│   │   ├── filters/
│   │   │   └── http-exception.filter.ts  # ExceptionFilter global → envelope { error: { code, message, details? } }
│   │   ├── guards/
│   │   │   └── api-key.guard.ts          # CanActivate: valida X-API-Key contra env.API_KEY
│   │   ├── interceptors/
│   │   │   └── logging.interceptor.ts    # Loggea request → response con duración (vía pino)
│   │   └── middleware/
│   │       └── pino-http.middleware.ts   # HTTP request logger (vía pino-http)
│   │
│   ├── database/
│   │   ├── mikro-orm.config.ts           # config de MikroORM (DB driver, entities discovery, migrations path)
│   │   ├── database.module.ts            # MikroOrmModule.forRoot(config) — global module
│   │   └── migrations/                   # archivos Migration<TIMESTAMP>.ts generados por migrator:generate
│   │
│   ├── modules/
│   │   └── resource/                     # ejemplo: módulo "resource" (otros features siguen este patrón)
│   │       ├── resource.entity.ts        # @Entity() class Resource
│   │       ├── dto/
│   │       │   ├── create-resource.dto.ts # Zod schema + ZodDto wrapper
│   │       │   ├── update-resource.dto.ts
│   │       │   └── query-resource.dto.ts  # filtros, paginación
│   │       ├── resource.service.ts       # @Injectable() — business logic
│   │       ├── resource.controller.ts    # @Controller('resources') — HTTP layer
│   │       ├── resource.module.ts        # importa entity + service + controller; MikroOrmModule.forFeature([Resource])
│   │       └── resource.controller.spec.ts # Vitest integration test
│   │
│   ├── health/
│   │   ├── health.controller.ts          # GET /health → { status: 'ok' } (sin auth)
│   │   └── health.module.ts
│   │
│   └── scripts/
│       └── sync.ts                       # CLI script: lee DB fuente → escribe DB API
│
├── data/
│   └── api.db                            # SQLite DB de la API (gitignored)
│
├── .env.example                          # PORT, FRONTEND_ORIGIN, API_KEY, DATABASE_URL, SOURCE_DB_URL, NODE_ENV, LOG_LEVEL
├── package.json
├── tsconfig.json                         # strict: true, target: ES2022, module: ESNext, moduleResolution: Bundler
├── biome.json                            # lint + format config
├── vitest.config.ts
└── mikro-orm.config.ts                   # alternativa a src/database/mikro-orm.config.ts
```

### Convenciones de NestJS (antes del patrón)

Tres cosas que confunden al que viene de Spring / FastAPI / Express:

**1. `modules/<feature>/` = organización por feature, no por capa.** NestJS (heredado de Angular) agrupa **todo** lo relativo a un concepto de negocio en una carpeta: entity, DTOs, service, controller, tests. No hay `controllers/`, `services/`, `entities/` globales — eso sería por capa técnica. Cada feature es independiente (bajo acoplamiento).

**2. `*.module.ts` ≠ módulo runtime.** No es un package ni se ejecuta por request. Es una **clase de metadata** (`@Module()`) que el contenedor de DI lee una vez al boot para registrar providers/controllers/imports:

```typescript
@Module({
  imports: [MikroOrmModule.forFeature([Resource])],  // hace EntityRepository<Resource> inyectable
  controllers: [ResourceController],                  // registra las rutas HTTP
  providers: [ResourceService],                       // registra services inyectables
  exports: [ResourceService],                         // (opcional) expone a otros módulos
})
export class ResourceModule {}
```

Equivalente granular al `@Configuration` de Spring, pero **uno por feature** en vez de uno global.

**3. Patrón por feature.** Cada `<feature>/` tiene la misma estructura interna (convención, no regla dura):

| Archivo | Decorator | Capa |
| --- | --- | --- |
| `<feature>.entity.ts` | `@Entity()` | DB (MikroORM) |
| `<feature>.service.ts` | `@Injectable()` | Lógica (sin HTTP) |
| `<feature>.controller.ts` | `@Controller()` | HTTP (rutas, validación) |
| `<feature>.module.ts` | `@Module()` | Wiring de DI |
| `dto/<feature>-*.dto.ts` | `ZodDto` wrapper | Validación (Zod + OpenAPI) |
| `<feature>.controller.spec.ts` | (Vitest) | Tests de integración |

**4. ¿Por qué `health/` está fuera de `modules/`?** No es feature de negocio — es infra sin entity ni service. Regla mental: si tiene entity propia → `modules/<feature>/`; si no → al lado (`health/`, `common/`, `config/`, `auth/` si crece).

---

**Patrón módulo** (ejemplo con `resource`):

```typescript
// modules/resource/resource.entity.ts
import { Entity, PrimaryKey, Property } from '@mikro-orm/core';

@Entity()
export class Resource {
  @PrimaryKey()
  id!: number;

  @Property()
  name!: string;

  @Property({ nullable: true })
  description?: string;

  @Property()
  createdAt: Date = new Date();
}
```

```typescript
// modules/resource/dto/create-resource.dto.ts
import { z } from 'zod';
import { createZodDto } from 'nestjs-zod';

export const CreateResourceSchema = z.object({
  name: z.string().min(1).max(100),
  description: z.string().max(500).optional(),
});

export class CreateResourceDto extends createZodDto(CreateResourceSchema) {}
```

```typescript
// modules/resource/resource.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@mikro-orm/nestjs';
import { EntityRepository } from '@mikro-orm/sqlite';
import { Resource } from './resource.entity';
import { CreateResourceDto, UpdateResourceDto, QueryResourceDto } from './dto';

@Injectable()
export class ResourceService {
  constructor(
    @InjectRepository(Resource)
    private readonly repo: EntityRepository<Resource>,
  ) {}

  findAll(q: QueryResourceDto) {
    return this.repo.findAll({
      limit: q.pageSize,
      offset: (q.page - 1) * q.pageSize,
      orderBy: { createdAt: 'DESC' },
    });
  }

  findOne(id: number) {
    return this.repo.findOneOrFail({ id });  // tira NotFoundError si no existe
  }

  create(dto: CreateResourceDto) {
    const resource = this.repo.create(dto);
    return this.repo.getEntityManager().persistAndFlush(resource);
  }

  async update(id: number, dto: UpdateResourceDto) {
    const resource = await this.findOne(id);
    this.repo.assign(resource, dto);
    await this.repo.getEntityManager().flush();
    return resource;
  }

  async delete(id: number) {
    const resource = await this.findOne(id);
    await this.repo.getEntityManager().removeAndFlush(resource);
  }
}
```

```typescript
// modules/resource/resource.controller.ts
import { Body, Controller, Delete, Get, HttpCode, Param, ParseIntPipe, Patch, Post, Put, Query, UseGuards } from '@nestjs/common';
import { ApiKeyGuard } from '../../common/guards/api-key.guard';
import { ResourceService } from './resource.service';
import { CreateResourceDto, UpdateResourceDto, QueryResourceDto } from './dto';
import { ApiTags, ApiSecurity } from '@nestjs/swagger';

@ApiTags('resources')
@ApiSecurity('X-API-Key')
@UseGuards(ApiKeyGuard)
@Controller('resources')
export class ResourceController {
  constructor(private readonly service: ResourceService) {}

  @Get()
  list(@Query() q: QueryResourceDto) {
    return this.service.findAll(q);
  }

  @Get(':id')
  detail(@Param('id', ParseIntPipe) id: number) {
    return this.service.findOne(id);
  }

  @Post()
  @HttpCode(201)
  create(@Body() dto: CreateResourceDto) {
    return this.service.create(dto);
  }

  @Put(':id')
  update(@Param('id', ParseIntPipe) id: number, @Body() dto: UpdateResourceDto) {
    return this.service.update(id, dto);
  }

  @Patch(':id')
  patch(@Param('id', ParseIntPipe) id: number, @Body() dto: UpdateResourceDto) {
    return this.service.update(id, dto);  // partial = same handler, Zod schema hace el trabajo
  }

  @Delete(':id')
  @HttpCode(204)
  async delete(@Param('id', ParseIntPipe) id: number) {
    await this.service.delete(id);
  }
}
```

---

## 3. Flujo OpenAPI

```
NestJS controllers + Zod schemas
        │
        │ (runtime: nestjs-zod + @nestjs/swagger decorators)
        ▼
OpenAPI spec 3.0 (auto-generado)
        │
        │ servido en:
        ├── /docs       → Swagger UI (solo dev)
        └── /docs-json  → spec.json (para FE codegen)
                  │
                  │ FE corre: openapi-typescript http://localhost:8787/docs-json -o src/api/types.ts
                  ▼
            src/api/types.ts (tipos TS)
```

**Setup en `main.ts`**:

```typescript
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // CORS (Q22)
  app.enableCors({
    origin: process.env.FRONTEND_ORIGIN,
    credentials: true,
  });

  // OpenAPI (Q7/Q8)
  const config = new DocumentBuilder()
    .setTitle('API de [recurso]')
    .setDescription('Backend local para servir datos del data warehouse')
    .setVersion('1.0')
    .addApiKey({ type: 'apiKey', name: 'X-API-Key', in: 'header' }, 'X-API-Key')
    .build();
  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('docs', app, document);

  await app.listen(process.env.PORT ?? 8787);
}
bootstrap();
```

**FE workflow** (referencia, no parte de este repo):

```bash
# Una vez (o en CI cuando cambia el spec):
pnpm dlx openapi-typescript http://localhost:8787/docs-json -o src/api/types.ts
```

El FE importa los tipos generados sin acoplamiento a un cliente HTTP específico (decisión Q9).

---

## 4. Esquema DB

SQLite como motor (mismo que la DB fuente cuando existe — ver §5 para escenarios). Migraciones forward-only via MikroORM Migrator.

> **Escenarios posibles** (la elección se difiere a implementación, ver Q10 en `architecture-decisions.md`):
> - **A) Greenfield / API-first:** la DB de la API es la **única** fuente de verdad. Datos nacen vía `POST /resources` (R8 CRUD desde v1).
> - **B) Alongside existing DB (caso actual):** DB fuente pre-existente; sync poblará la DB de la API (ver §5).
> - **C) Source sigue activa:** sync periódico o incremental.

**Entity example** (recursos del dominio siguen este patrón):

```typescript
// modules/resource/resource.entity.ts (con relación OneToMany)
import { Collection, Entity, OneToMany, PrimaryKey, Property } from '@mikro-orm/core';
import { Tag } from './tag.entity';

@Entity()
export class Resource {
  @PrimaryKey()
  id!: number;

  @Property()
  name!: string;

  @Property({ nullable: true })
  description?: string;

  @OneToMany(() => Tag, t => t.resource)
  tags = new Collection<Tag>(this);

  @Property()
  createdAt: Date = new Date();
}
```

**Migraciones**:

- Generadas: `npx mikro-orm migrator:generate --path src/database/migrations InitialSchema`.
- Aplicadas: `npx mikro-orm migrator:up`.
- Archivos commiteados al repo.
- Forward-only (ver Q6 en `architecture-decisions.md`).

**Tabla `mikro_orm_migrations`** (auto-manejada por MikroORM):
- Lleva registro de qué migraciones se aplicaron.
- Solo aplica las nuevas al `migrator:up`.

**Path local**: `data/api.db` (gitignored).

**Driver**: `@mikro-orm/sqlite` que envuelve `better-sqlite3`. Sin servicio externo corriendo (Q4 — SQLite embedido).

---

## 5. Plan de sync (solo si escenario B o C)

> **Si el escenario es A (greenfield / API-first):** esta sección **no aplica**. La DB de la API se crea vacía desde migraciones y los datos nacen vía `POST /resources` (R8). En ese caso, eliminar `SOURCE_DB_URL`, el script `src/scripts/sync.ts`, y el comando `pnpm run sync`. Mantener §4 (esquema DB) y §6 (endpoints) tal cual.

Script CLI que copia datos desde la DB fuente (SQLite del data warehouse) hacia la DB de la API. **Idempotente** — re-ejecutable sin duplicar.

```typescript
// scripts/sync.ts (esqueleto)
import Database from 'better-sqlite3';
import { MikroORM } from '@mikro-orm/core';

async function main() {
  // 1. Conectar a la DB fuente (read-only, sin MikroORM — query SQL directo)
  const source = new Database(process.env.SOURCE_DB_URL!, { readonly: true });

  // 2. Conectar a la DB de la API via MikroORM
  const orm = await MikroORM.init();
  const em = orm.em.fork();

  // 3. Sync por entidad (transacción por batch)
  await em.transactional(async em => {
    const sourceRows = source.prepare('SELECT * FROM source_table').all();
    for (const row of sourceRows) {
      // Mapear source row → entity shape (puede haber diferencias de schema)
      em.upsert(Resource, mapSourceToResource(row));
    }
  });

  // 4. Cleanup
  source.close();
  await orm.close();
}

main().catch(err => {
  console.error(err);
  process.exit(1);
});
```

**Comando**: `npm run sync` (alias de `tsx src/scripts/sync.ts`).

**Cuándo corre**:
- Dev: manual cuando el FE necesita data fresca.
- Prod: después de las migraciones en cada deploy (cron o webhook — fuera de scope v1).

**Decisiones de sync** (ver Q10 en `architecture-decisions.md`):
- Sync one-way (fuente → API).
- API mantiene su propia DB; la fuente no se toca en runtime.
- Si la DB crece, sync incremental con `WHERE updated_at > last_sync` — pero v1 hace full sync, se optimiza si la performance lo demanda.

---

## 6. Endpoints iniciales

CRUD completo desde v1 (R8). Ejemplo con `resource` (los demás recursos siguen el patrón).

| Método | Path | Auth | Body | Response | Status |
| --- | --- | --- | --- | --- | --- |
| `GET` | `/resources` | `X-API-Key` | — | `{ data: Resource[], meta: { total, page, pageSize } }` | 200 |
| `GET` | `/resources/:id` | `X-API-Key` | — | `Resource` | 200 / 404 |
| `POST` | `/resources` | `X-API-Key` | `CreateResourceDto` | `Resource` | 201 / 422 |
| `PUT` | `/resources/:id` | `X-API-Key` | `UpdateResourceDto` | `Resource` | 200 / 404 / 422 |
| `PATCH` | `/resources/:id` | `X-API-Key` | `UpdateResourceDto` (parcial) | `Resource` | 200 / 404 / 422 |
| `DELETE` | `/resources/:id` | `X-API-Key` | — | — | 204 / 404 |
| `GET` | `/health` | — | — | `{ status: 'ok' }` | 200 |

**Headers siempre presentes**:
- Request: `X-API-Key: <secret>` (excepto `/health`).
- Request: `Content-Type: application/json` (en POST/PUT/PATCH).
- Response: `Content-Type: application/json` + CORS headers (`Access-Control-Allow-Origin`, etc.).

**Envelope de error** (ver Q19 en `architecture-decisions.md`):

```json
// 4xx / 5xx
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "Resource with id 123 does not exist",
    "details": { "resourceId": 123 }
  }
}
```

**Validación Zod** (Q17): si el body no cumple el schema, devuelve 422 con `details` listando los campos inválidos.

**Códigos de error comunes**:
- `VALIDATION_ERROR` → 422
- `UNAUTHORIZED` → 401 (falta `X-API-Key` o inválido)
- `NOT_FOUND` → 404
- `INTERNAL_ERROR` → 500

---

## 7. Setup commands

```bash
# Setup inicial
pnpm install                  # instala deps

# Migraciones
pnpm run db:migrate           # aplica migraciones pendientes (tsx src/database/migrate.ts)

# Sync inicial desde la DB fuente
pnpm run sync                 # tsx src/scripts/sync.ts

# Dev (auto-reload con tsx watch)
pnpm run dev                  # tsx watch src/main.ts

# Build
pnpm run build                # tsup src/main.ts → dist/

# Prod
pnpm start                    # node dist/main.js

# Tests
pnpm test                     # Vitest — unit + integration (corre una vez)
pnpm run test:watch           # Vitest watch mode
pnpm run test:cov             # Vitest con coverage

# Lint / format
pnpm run lint                 # Biome check
pnpm run format               # Biome format --write
```

**`package.json` scripts**:

```json
{
  "scripts": {
    "dev": "tsx watch src/main.ts",
    "build": "tsup src/main.ts --format esm --target node22",
    "start": "node dist/main.js",
    "db:migrate": "tsx src/database/migrate.ts",
    "sync": "tsx src/scripts/sync.ts",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:cov": "vitest run --coverage",
    "lint": "biome check",
    "format": "biome format --write"
  }
}
```

**Variables de entorno** (`.env.example`):

```bash
PORT=8787
FRONTEND_ORIGIN=http://localhost:5173
API_KEY=dev-secret-change-in-prod
DATABASE_URL=file:./data/api.db
# SOURCE_DB_URL solo si escenario B/C (ver §4-§5). En escenario A (greenfield), eliminar.
SOURCE_DB_URL=file:./path/to/data-warehouse.db
NODE_ENV=development
LOG_LEVEL=debug
```

---

## Tradeoffs vs las otras 2 propuestas

### vs Python (FastAPI)

**Ganamos**:
- TS type system end-to-end (más estricto que Python type hints; errores en compile-time).
- `@nestjs/mikro-orm` adapter oficial — integración más pulida que SQLAlchemy + FastAPI wrappers.
- Ecosystem Node más maduro para OpenAPI codegen (NestJS genera spec out-of-the-box; FastAPI también pero requiere más setup con Pydantic).
- Async nativo con Promises/async-await (mismo modelo mental que JavaScript del FE).

**Perdés**:
- NestJS es más verboso que FastAPI (decorators en todas partes, módulos explícitos).
- Compilación step (TS → JS) suma fricción vs Python que se ejecuta directo.
- Cold start un poco peor que FastAPI (Node también arranca rápido, pero con más overhead de módulos que uvicorn).

### vs Java + Spring Boot

**Ganamos**:
- Arranque significativamente más rápido (NestJS en ~1s vs Spring Boot en ~5-10s).
- Mucho menos boilerplate (no hay `pom.xml`, application classes, autowire annotations).
- TS es más flexible que Java (e.g., structural typing, decorators opcionales).
- Ecosystem más liviano (sin JVM, sin classpath hell).

**Perdés**:
- TypeScript no es tan type-safe como Java en compile-time (más reliance en runtime checks).
- NestJS no tiene el mismo ecosistema enterprise que Spring (security distribuida, transactions distribuidas, etc.) — pero v1 no lo necesita.
- Spring Boot + Hibernate con Postgres es el camino más "production-tested" del mercado; Node + MikroORM es más nuevo y con menos battle-testing a escala.

---

## Próximos pasos si esta propuesta se aprueba

1. **Validar el esqueleto**: implementar un módulo `resource` mínimo (entity + DTOs + service + controller) end-to-end. Confirmar que el flow OpenAPI + Zod + MikroORM funciona como se describe.
2. **Implementar auth + CORS** reales y testear con un FE mínimo (curl + browser).
3. **Implementar sync** desde la DB fuente para una entidad de ejemplo.
4. **Una vez validado**: usar este esqueleto como template para `architecture-proposal.python.md` (FastAPI equivalente) y `architecture-proposal.java-spring.md` (Spring Boot equivalente) — vía subagentes para que salgan consistentes.

---

*Propuesta cerrada 2026-09-30 — list para review antes del fan-out a Python y Java.*
