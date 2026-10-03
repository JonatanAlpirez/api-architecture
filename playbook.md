# API Playbook

> Lineamientos básicos para arrancar cualquier API de backend con uno de los stacks soportados. Stack-agnóstico por diseño: lo que aplique a un stack particular vive en su `architecture-proposal.<stack>.md`.

## Cómo usar este repo

1. **Elegí el stack** que vas a usar (Node+TS, Python o Java+Spring). Cada uno tiene su `architecture-proposal.<stack>.md` con la referencia de implementación concreta.
2. **Leé este playbook** completo — son los estándares que aplican a cualquier API nuestra.
3. **Leé la propuesta del stack elegido** — encontrás las herramientas por capa (ORM, validación, etc.) y los snippets concretos de cómo arrancar.
4. **Arrancá** — el playbook te dice el "qué", la propuesta del stack te dice el "cómo".

## Stacks soportados

- **Node + TypeScript** → ver `node-ts/architecture-proposal.node-ts.md`
- **Python** → ver `python/architecture-proposal.python.md`
- **Java + Spring Boot 3** → ver `java-spring/architecture-proposal.java-spring.md`

## Snapshot por stack — referencia rápida

| Capa              | Node + TS                              | Python                              | Java + Spring Boot                     |
| ----------------- | -------------------------------------- | ----------------------------------- | -------------------------------------- |
| Lenguaje          | TypeScript                             | Python 3.12+                        | Java 21 (LTS)                          |
| Framework HTTP    | NestJS + `@nestjs/swagger`             | FastAPI                             | Spring Boot 3 + springdoc-openapi      |
| DB                | SQLite                                 | SQLite                              | PostgreSQL                             |
| ORM               | MikroORM                               | SQLAlchemy 2.0                      | Spring Data JPA (Hibernate)            |
| Migraciones       | MikroORM Migrator                      | Alembic                             | Flyway                                 |
| Validación        | Zod                                    | Pydantic v2                         | jakarta.validation                     |
| OpenAPI           | code-first, spec 3.0                   | code-first, spec 3.0                | code-first, spec 3.0                   |
| Auth              | API key (`X-API-Key`)                  | API key (`X-API-Key`)               | API key (`X-API-Key`) vía filter       |
| Logging           | Pino (JSON a stdout)                   | Loguru (JSON a stdout)              | Logback + SLF4J (JSON a stdout)        |
| Tests             | Vitest (unit + integration)            | pytest + pytest-asyncio             | JUnit 5 + Mockito + Spring Boot Test   |
| Lint/format       | Biome                                  | Ruff                                | Spotless + SpotBugs                    |

Las tres opciones comparten: base path, estructura modular, codegen para el FE, puerto y auth. **Stack C diverge en DB** (PostgreSQL en lugar de SQLite) por compatibilidad nativa de JPA.

---

## Estándares (stack-agnostic)

> Cada estándar aplica a los 3 stacks. El "cómo" específico vive en cada `architecture-proposal.<stack>.md`.

### S1. Arquitectura en capas (Controller → Service → Repository)

Toda API nuestra debe organizar el código en **tres capas**, en orden de dependencias:

- **Controller / Router / Handler:** capa HTTP. Recibe requests, valida input, llama al service, devuelve response. **Sin lógica de negocio.**
- **Service:** orquestación entre repositorios, transformación de datos, reglas del dominio. **Sin acceso directo a la DB ni a HTTP.**
- **Repository / DAO:** acceso a la DB. Queries, transacciones, persistencia. **Sin lógica de negocio.**

**Por qué:** separa responsabilidades, hace el código testeable de a capas (unit test al service sin levantar HTTP ni DB), y permite cambiar una capa sin tocar las demás.

---

### S2. API documentada con OpenAPI 3.0 (code-first)

Toda API nuestra debe estar documentada con **OpenAPI 3.0.x** (no 3.1 todavía — compatibilidad de tooling), generada **code-first** desde el código.

- Los controllers y DTOs son la **fuente de verdad**. El spec OpenAPI es un side-effect.
- El FE debe poder generar tipos TypeScript automáticamente con `openapi-typescript` (mismo tool para los 3 stacks).
- El spec se sirve en una ruta estándar (típicamente `/openapi.json` y `/docs`).

**Por qué:** el FE deriva tipos automáticamente del spec. Sin spec, los tipos se desincronizan. Cero tipos manuales a mantener.

---

### S3. CRUD completo desde v1

Toda API nuestra debe soportar las operaciones básicas desde v1: **GET, POST, PUT/PATCH, DELETE** sobre cada recurso expuesto. Read-only queda descartado: la API debe poder persistir datos desde el inicio, no agregar escritura en v2.

**Por qué:** el FE necesita escribir (no solo leer) desde el día uno. Agregar escritura más tarde fuerza breaking changes en el contrato.

---

### S4. Auth con API key estática en header

Toda API nuestra debe protegerse con una **API key estática en el header `X-API-Key`**.

- El cliente manda `X-API-Key: <key>` en cada request.
- El server chequea el valor contra una variable de entorno; rechaza con 401/403 si no coincide.
- **Sin login, sin refresh token.** Rotación: se cambia la env var en ambos lados y se redeploya.

**Por qué:** 5 líneas de código, evita sustos si el puerto queda expuesto por accidente. JWT queda como upgrade path si aparece multi-usuario.

---

### S5. Validación de input y response con DTO schemas

Toda API nuestra debe validar los datos en el borde (request) y derivar sus DTOs de un **schema engine** del stack elegido.

- **Definir el schema una vez.** El mismo schema es tipos (compile-time), validador (runtime) y fuente para OpenAPI.
- **Validar antes del service.** Si el request no valida, devolver 422 con detalle por campo, no pasar al service.
- **Validar el response también** en dev/CI. Si lo que devolvés no cumple el contrato OpenAPI, fallar loud.

**Por qué:** sin validación, el server recibe basura, falla tarde (500 con stack trace) y devuelve mensajes feos. Con validación en el borde, errores claros (422 con detalle) y el service solo ve datos limpios.

---

### S6. Formato de error consistente (envelope propio)

Toda API nuestra debe devolver errores 4xx y 5xx en un **envelope consistente**:

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Recurso no encontrado",
    "details": [{ "field": "name", "issue": "required" }]
  }
}
```

- **`code`:** string estable (`NOT_FOUND`, `VALIDATION_ERROR`, `UNAUTHORIZED`, etc.). El FE switchea sobre este valor.
- **`message`:** human-readable, en el idioma de la audiencia del FE.
- **`details`:** opcional. Array de errores por campo (usado para validación).
- **Sin stack traces en responses 5xx.** Stack traces solo en logs server-side.

**Por qué:** con formato consistente, el FE hace `if (response.error.code === 'NOT_FOUND')` y muestra UI apropiada. Sin formato, parsea N formas distintas por endpoint.

---

### S7. Logging estructurado a stdout

Toda API nuestra debe emitir logs a **stdout en formato JSON** (no texto libre, no archivos).

- Un log por request relevante: `{ "level": "info", "msg": "request handled", "method": "GET", "path": "/resources/42", "status": 200, "duration_ms": 12 }`.
- Un log por error con `err` y `stack`: `{ "level": "error", "msg": "DB connection failed", "err": "ECONNREFUSED ...", "stack": "..." }`.
- **Capturar stdout lo hace el runtime** (systemd, Docker, etc.) — la app no se encarga de rotación de archivos.

**Por qué:** JSON parseable permite filtrar (`level=error AND path=/resources`), enviar a sistemas centralizados (Loki, ELK), y rotar via el runtime. Texto libre es grep-friendly pero rompe en multi-service.

---

### S8. CORS con origin configurable

Toda API nuestra debe configurar CORS desde el primer endpoint, no "después".

- **`FRONTEND_ORIGIN`** se lee de env var / config. Default dev: `http://localhost:5173` (Vite) o `http://localhost:3000` (Next.js).
- **`Access-Control-Allow-Credentials: true`** siempre (la API key va en header; el browser requiere la flag para exponer headers custom cross-origin).
- **Preflight `OPTIONS`** debe responderse correctamente para métodos con headers custom (`POST` con `X-API-Key`).
- Si FE y API están detrás del mismo reverse proxy en prod, son mismo origen y CORS deja de aplicar — pero la config queda activa por si el deploy los separa.

**Por qué:** sin CORS configurado, el FE recibe `blocked by CORS policy` aunque la lógica esté OK. Es el primer bug que aparece al integrar FE + API y es 100% evitable.

---

### S9. Testing en 3 niveles (unit + integration + e2e opcional)

Toda API nuestra debe tener cobertura balanceada en 3 niveles:

- **Unit (~70%):** funciones puras (services, validators) testeadas aisladas con mocks. Rápido (<10ms/test), alto volumen.
- **Integration (~25%):** endpoint real + DB real (o in-memory) + middleware. Verifica el flujo completo. Más lento (100ms-1s/test).
- **E2E (~5%):** el sistema completo levantado como caja negra. Lento (segundos). Opcional en v1, recomendado cuando aparezca un caso que lo justifique.

**Mock strategy:**
- Unit: mocks de todo (DB, HTTP, tiempo). El test no toca infra.
- Integration: DB real (in-memory o Testcontainers). Mocks solo para servicios externos.

**Por qué:** el sweet spot para una API local es 70/25/5. Sin tests = refactorizar con miedo. Solo unit = cambios de capa HTTP/DB pasan sin detección. Solo e2e = tests lentos y debugging difícil.

---

### S10. Lint + format con un solo tool

Toda API nuestra debe correr lint y format con **una sola tool por stack** (no dos), ejecutada en pre-commit y en CI.

- El tool cubre **lint** (code smells, bugs potenciales, convenciones) y **format** (indentación, comillas, line length) en la misma config.
- Pre-commit hook corre `lint + format --check`. CI corre lo mismo. El código no conforme no llega a `main`.

**Por qué:** dos tools (ESLint + Prettier, flake8 + black) duplican config y generan fricciones. Un solo tool (Biome, Ruff, Spotless) elimina eso.

---

## Datos

### S11. DB relacional con ORM y migrations forward-only

Toda API nuestra debe usar una **DB relacional** accedida via **ORM** del stack elegido, con **migraciones versionadas forward-only**.

- Las entities/models viven en código. El ORM genera el diff vs última migración → archivo `.ts` / `.py` / `.sql` commiteado al repo.
- Migrations son **forward-only** por default (sin downgrade en prod). Cada archivo es código, no magia.
- **`autogenerate` se revisa siempre antes de commitear.** El diff generado puede tener basura; el humano valida.

**Por qué:** la DB de la API diverge de la fuente (cómputos cacheados, columnas calculadas, índices). Sin migrations versionadas, cada cambio se hace a mano y se pierde el rastro.

---

### S12. Source data integration — sync one-way (o skip)

Cuando existe una **DB fuente** (data warehouse, sistema legacy), toda API nuestra debe integrarse con **sync one-way**, no con read directo.

- La API mantiene **su propia DB**.
- Un script copia los datos desde la fuente periódicamente (cron) o incremental (`WHERE updated_at > last_sync`).
- La API **no toca la DB fuente en runtime.** Eso desacopla la API del filesystem/ruta de la fuente.

**Excepción (greenfield / API-first):**
- Si la API es la única fuente de verdad, **no hay script de sync** ni `SOURCE_DB_URL`. Los datos nacen vía `POST` (ver S3).

**Por qué:** read directo acopla la API al path/estructura de la fuente. Si la fuente cambia de ruta, se mueve, o cambia de motor, la API se rompe. Sync one-way desacopla.

---

## Runtime & deployment

### S13. Puerto y base path

- **Puerto:** `8787` por default (configurable; distinto si corren varias APIs juntas).
- **Base path:** raíz (sin prefijo `/v1`). Si en v2 aparece un breaking change real, se introduce prefijo `/v1` en ese momento.

**Por qué:** simplicidad inicial. Prefijo de versión se agrega cuando se necesita, no antes.

---

### S14. Dev con auto-reload, prod estable

| Entorno | Objetivo                                    | Patrón                                          |
| ------- | ----------------------------------------- | ----------------------------------------------- |
| Dev     | Iteración rápida (re-typecheck + re-run)  | Auto-reload nativo del stack (`tsx watch`, `uvicorn --reload`, Spring Boot DevTools) |
| Prod    | Estabilidad + performance                | Binario standalone / fat jar / `uvicorn` workers |

**Por qué:** dev con auto-reload evita `Ctrl+C` + re-start en cada cambio (5-10s vs 50ms). Prod sin auto-reload evita el overhead del watcher y se asegura de correr el código que testaste.

---

## API endpoints (S15-S16)

### S15. Paginación de listas

Toda API nuestra que devuelva una **lista de recursos** debe paginar el resultado desde el primer endpoint. No se devuelve la colección entera, ni siquiera si "hoy son pocos registros".

- **Estilo default:** offset-based — `?page=1&limit=20` (donde `page` empieza en 1 y `limit` default 20, max 100).
- **Cuando usar cursor-based:** datasets grandes o feeds en tiempo real (eventos, logs, sync). `?cursor=<opaque>&limit=20`. El cursor es opaco al cliente (no parseable, no اعتماد en su formato).
- **Response siempre incluye metadata:** `{ "data": [...], "pagination": { "page": 1, "limit": 20, "total": 142, "has_next": true } }`.
- **Errores:** si el cliente pide `page` o `limit` fuera de rango, devolver 422 con detalle por campo (ver S5).

**Por qué:** sin paginación, el día que la tabla tiene 10k rows, el server se cae o el FE tarda 30s. Implementar paginación después requiere cambiar el contrato de TODOS los endpoints de lista — breaking change masivo.

---

### S16. Filtering & sorting

Toda lista debe soportar **filtering** y **sorting** por al menos los campos más consultados del recurso (típicamente `status`, `created_at`, `updated_at`).

- **Filtering:** query params por campo. `?status=active&created_after=2026-01-01`. Formato consistente (un solo valor por campo, o lista separada por coma si es multi-valor).
- **Sorting:** `?sort=<field>` (ascendente) o `?sort=-<field>` (descendente). Default: orden determinístico (ej. `-created_at` o `id`) para que paginación sea estable.
- **Whitelist:** los campos permitidos para filter/sort deben estar whitelisteados — el cliente **no** puede filtrar por columnas arbitrarias (riesgo de SQL injection si la query se construye dinámicamente).
- **Combinable con paginación** (S15): `?status=active&sort=-created_at&page=2&limit=20`.

**Por qué:** sin filter/sort, el FE tiene que pedir TODA la lista y filtrar client-side. Con miles de registros, esto es inviable. Implementar filter/sort después requiere cambiar el contrato de cada endpoint — también breaking change.

---

## Cross-cutting adicionales (S17-S18)

### S17. Rate limiting

Toda API nuestra debe protegerse con **rate limiting**, especialmente **antes de exponer públicamente**.

- **Estrategia default:** token bucket por API key (la misma de S4). 100 requests/minuto por default; configurable por env (`RATE_LIMIT_PER_MINUTE`).
- **Excedido:** respuesta `429 Too Many Requests` con header `Retry-After: <segundos>` y `X-RateLimit-Reset: <epoch>`.
- **Headers informativos siempre:** `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` en cada response (no solo cuando se excede) — el FE puede self-throttle proactivamente.
- **Excluir del rate limit:** `/health`, `/ready`, `/openapi.json` (endpoints de infra, no de negocio).

**Por qué:** sin rate limiting, un cliente con bug puede tumbar el server con un loop, o un atacante puede enumerar endpoints. 100 req/min es generoso para uso legítimo y suficiente para detectar abuso.

---

### S18. Secrets handling

Toda API nuestra debe cargar secrets (la X-API-Key para validar requests — ver S4 —, DB password, tokens de servicios externos) desde **variables de entorno**. Nunca hardcoded, nunca en el repo, nunca en logs.

- **Dev:** archivo `.env` en la raíz del proyecto, **gitignored** desde el inicio. Cada dev tiene su propio `.env` (no se commitea). `.env.example` commiteado con keys vacías como template.
- **Prod:** variables de entorno del sistema (systemd `EnvironmentFile`, Docker `--env-file`, secrets manager del cloud, etc.). Rotación sin redeploy si el provider lo soporta.
- **En logs:** redactar secrets automáticamente. Si por error un secret entra a un log, debe aparecer como `"***REDACTED***"`, no como el valor. Esto es responsabilidad del logger (S7), no del código de negocio.
- **En código:** nunca `${SECRET}` interpolado en strings logueables, nunca `console.log(process.env.API_KEY)`. Usar el config loader del stack (ConfigModule en NestJS, `pydantic-settings` en FastAPI, `@Value` en Spring) que carga una vez al arranque y nunca expone el valor en logs.

**Por qué:** secret en el repo = comprometido para siempre (git history no se borra). Secret en logs = expuesto a cualquier sistema de monitoreo. Secret hardcoded = cambio requiere deploy de código, no de config. Desde el día 1, `.env` gitignored + `.env.example` commiteado = la regla más barata que evita el peor escenario.

---

## Cómo extender este playbook

Cuando aparezca una decisión nueva que aplique a los 3 stacks:

1. Discutila en una ADR corta (ej. `adr-NNN-titulo.md`).
2. Si aterriza como estándar, agregala como `S<n+1>` en este doc.
3. Si es específica de un stack, va en `architecture-proposal.<stack>.md`.
4. Si es solo una nota arquitectónica, queda en `architecture-decisions.md` como soporte histórico.

---

## Glosario mínimo

- **Code-first:** el spec OpenAPI se genera desde el código, no se mantiene a mano.
- **DTO (Data Transfer Object):** schema que define la forma de los datos que entran/salen de un endpoint.
- **Forward-only migrations:** migrations de DB que se aplican solo hacia adelante (sin downgrade).
- **Structured logging:** logs en formato JSON parseable, no texto libre.
- **Sync one-way:** la API mantiene su DB y copia datos desde la fuente periódicamente; la fuente nunca es leída en runtime por la API.

---

*Playbook agnóstico — los detalles por stack viven en cada `architecture-proposal.<stack>.md`.*