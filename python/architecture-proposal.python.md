# Architecture Proposal — Python (stack-specific spec template)

> **Rol:** stack-specific guidance para usar [`spec-template.md`](../spec-template.md) con **Python + FastAPI** (SQLAlchemy 2.0 + Pydantic v2). Llenás el spec template con tu dominio, después consultás este archivo para saber qué tools, versiones y patterns usar para cada sección.
>
> **No es un code template.** No hay worked example en código todavía (`api-python-reference` está pendiente de crear). Las decisiones del "por qué" referenciadas viven en [`architecture-decisions.md`](../architecture-decisions.md) (Q1-Q26). Los estándares agnósticos viven en [`playbook.md`](../playbook.md) (S1-S16 + S18).
>
> **Nota:** mientras `api-python-reference` no exista, los "Worked example" de este doc linkean a [`api-node-reference`](https://github.com/JonatanAlpirez/api-node-reference) como analogía Node+TS. Los patterns son similares, no idénticos.

## Stack baseline

| Capa | Decisión | Versión | Notas |
| --- | --- | --- | --- |
| Lenguaje | Python | 3.12+ | Type hints nativos, async nativo, performance mejorado vs 3.11 |
| Framework HTTP | FastAPI | 0.110+ | Async nativo, OpenAPI automático, Pydantic integration |
| ORM | SQLAlchemy | 2.0+ (async) | Mature, async session, type-safe queries via `Mapped[]` |
| Validación | Pydantic | v2 | Single source: schema + tipos + OpenAPI; Rust core (10x+ vs v1) |
| OpenAPI integration | Built-in FastAPI | (latest) | OpenAPI se genera desde endpoints + Pydantic schemas |
| Tests | pytest + pytest-asyncio | latest | Standard Python; `asyncio` mode para endpoints async |
| HTTP testing | httpx AsyncClient | latest | Async test client compatible con FastAPI |
| Logging | Loguru | 0.7+ | Plug-and-play, structured JSON via `serialize=True` |
| Lint/format | Ruff | 0.1+ | Reemplaza flake8 + black + isort, una tool, ultra-rápido |
| Migraciones | Alembic | 1.13+ | Estándar de facto del ecosistema SQLAlchemy |
| CORS | `fastapi.middleware.cors.CORSMiddleware` | built-in | Built-in FastAPI; config por env `FRONTEND_ORIGIN` |
| Package manager | uv | latest | Rápido, Rust-based, reemplaza pip/poetry/virtualenv |

---

## Mapping a las secciones del spec-template

### §1-2. Project identity + Dominio

**No hay tooling Python-specific.** Completá el spec con tu dominio. SQLAlchemy 2.0 soporta `Mapped[...]` type annotations que te dan type-safety en las queries.

---

### §3. Endpoints (S3, S4)

- **Router pattern:** FastAPI con `APIRouter()` por recurso. Se monta en `main.py` con `app.include_router(resource.router, prefix="/resources")`.
- **CRUD verbs:** los 5 endpoints estándar vía `@router.get/@router.post/@router.patch/@router.delete`. `POST` devuelve 201 por default en FastAPI, `DELETE` se configura con `status_code=204`.
- **Auth (S4):** FastAPI `Depends(get_api_key)` como dependency de cada endpoint (o de todo el router con `dependencies=[Depends(get_api_key)]` para aplicar a todos). Compara `X-API-Key` con `hmac.compare_digest()` (constant-time, equivalente Python a Node).
- **Path param validation:** `Path(..., gt=0)` o `int` automático según el type hint.
- **Documentación OpenAPI:** description en cada endpoint, `tags` para agrupar en Swagger UI, `responses` para documentar status codes.

**Worked example (analogo Node):** [`api-node-reference/src/modules/resource/resource.controller.ts`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/src/modules/resource/resource.controller.ts). El patrón es similar — 5 endpoints sobre `/resources`, `@UseGuards(ApiKeyGuard)` a nivel de controller ≡ `dependencies=[Depends(get_api_key)]` a nivel de router.

---

### §4. Validación (S5)

- **Tool:** Pydantic v2 como single source para runtime + tipos + OpenAPI.
- **Pattern:** definís un `BaseModel` o un `TypedDict` con type hints. FastAPI lo usa para validar `body`/`query`/`path` automáticamente.
- **422 strategy (S5):** FastAPI devuelve 422 por default en validation errors (no 400 como `nestjs-zod`). **No necesitás custom pipe.** La diferencia: 422 de FastAPI ya viene con field-level details en `detail[]`.
- **Custom errors:** `model_validator` y `field_validator` para lógica de validación custom.
- **Por qué Pydantic vs dataclasses:** Pydantic = single source (runtime + tipos + OpenAPI), dataclasses no validan en runtime.

**Worked example (analogo Node):** [`api-node-reference/src/modules/resource/dto/`](https://github.com/JonatanAlpirez/api-node-reference/tree/main/src/modules/resource/dto). Mismo concepto, distinto syntax.

**Checklist S5:** ✓ Pydantic como single source, ✓ 422 automático (no custom pipe), ✓ field-level details en el response.

---

### §5. Error envelope (S6)

- **Shape:** `{ error: { code: string, message: string, details?: unknown } }`.
- **Implementación:** custom exception classes + `@app.exception_handler(...)` en `main.py`. FastAPI los aplica globalmente.
- **Status → code mapping:** ver tabla en [`spec-template.md` §5](../spec-template.md#5-error-envelope).
- **Pydantic errors:** los `RequestValidationError` de FastAPI ya devuelven 422 con `detail[]`. Mapealos al envelope con un custom exception handler que transforme `detail` a `error.details`.

**Worked example (analogo Node):** [`api-node-reference/src/common/filters/http-exception.filter.ts`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/src/common/filters/http-exception.filter.ts) — la lógica es análoga, la implementación es distinta.

**Checklist S6:** ✓ envelope consistente, ✓ status code mapping, ✓ Pydantic errors formateados.

---

### §6. Logging (S7)

- **Stack:** Loguru con `serialize=True` para JSON output.
- **Config:** función `setup_logging()` en `config/logging.py` que configura el sink (stdout) y el level (de `LOG_LEVEL` env).
- **HTTP logger:** custom middleware o `fastapi-logger` package — loggea cada request/response con duración.
- **Redaction:** Loguru no tiene redaction built-in, hay que usar `record` filters custom o el `patcher` de Loguru. Alternativa: usar `python-json-logger` o `structlog` (más configurable).
- **JSON en prod, pretty en dev:** `LOGURU_SERIALIZE=1` en prod, no en dev.

**Worked example (analogo Node):** [`api-node-reference/src/app.module.ts`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/src/app.module.ts) (LoggerModule.forRoot) — pino tiene redaction built-in, Loguru requiere más setup manual.

**Checklist S7:** ✓ JSON a stdout, ✓ redaction de secrets (manual), ✓ pretty en dev.

---

### §7. CORS (S8)

- **Built-in:** `app.add_middleware(CORSMiddleware, allow_origins=[env.FRONTEND_ORIGIN], allow_credentials=True, allow_methods=["*"], allow_headers=["*"])`.
- **Origen configurable:** env var `FRONTEND_ORIGIN` (default `http://localhost:5173` para Vite, ajustá a `:3000` para Next.js).

**Worked example (analogo Node):** [`api-node-reference/src/main.ts:17-20`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/src/main.ts#L17).

**Checklist S8:** ✓ configurable via env, ✓ credentials enabled.

---

### §8. Testing (S9)

- **Stack:** pytest + pytest-asyncio + httpx AsyncClient.
- **In-memory DB:** tests usan SQLite `:memory:` con SQLAlchemy + `create_all` (más rápido que `alembic upgrade`). Setup con fixture en `conftest.py`.
- **Override dependency:** `app.dependency_overrides[get_db] = override_get_db` para que los tests usen la DB in-memory.
- **AsyncClient:** `async with AsyncClient(app=app, base_url="http://test") as ac: ...` — wraps la app de FastAPI.
- **Sin gotcha de decorators:** Python con type hints no necesita tooling especial (a diferencia de TS con decorators).

**Worked example (analogo Node):** [`api-node-reference/src/modules/resource/resource.controller.spec.ts`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/src/modules/resource/resource.controller.spec.ts). El setup es similar (in-memory DB + re-aplicar cross-cutting en beforeEach), el syntax es async en vez de sync.

**Checklist S9:** ✓ integration tests con httpx, ✓ in-memory DB por test, ✓ dependency override para DB.

---

### §9. Lint / format (S10)

- **Tool:** Ruff — una sola tool, una sola config (`pyproject.toml [tool.ruff]`).
- **Reemplaza:** flake8 + black + isort + más (10+ tools clásicas de Python).
- **Qué cubre:** lint (reglas `E`, `W`, `F`, `I` por default), format, import sorting, pyupgrade, más.
- **Configuración:** sección `[tool.ruff]` en `pyproject.toml`. ~20 líneas vs ~100+ con la stack clásica.

**Worked example (analogo Node):** [`api-node-reference/biome.json`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/biome.json) — mismo concepto (1 tool, 1 config).

**Checklist S10:** ✓ 1 tool, 1 config, ✓ lint + format + import sorting.

---

### §10. Base de datos (S11, S12)

- **ORM:** SQLAlchemy 2.0 async (decisión Q5).
- **Dev engine:** SQLite via `sqlite+aiosqlite:///:memory:` (en tests) o `sqlite+aiosqlite:///./data/api.db` (en dev).
- **Prod engine:** PostgreSQL via `postgresql+asyncpg://user:***@host/dbname`. Cambio de 1 línea en `DATABASE_URL` + instalar `asyncpg`.
- **Migrations (S11):** Alembic, forward-only.
  - Generar: `alembic revision --autogenerate -m "InitialSchema"` (después de modificar un model).
  - Aplicar: `alembic upgrade head` o `uv run alembic upgrade head`.
  - Tabla interna `alembic_version` trackea cuáles corrieron.
- **Sync (S12):**
  - **Greenfield:** skip.
  - **Existing source DB:** script `uv run python src/scripts/sync.py` con SQLAlchemy + sqlite3 stdlib. Idempotente.
- **Q4 / Q5:** ver [`architecture-decisions.md`](../architecture-decisions.md) — por qué SQLite/Postgres, por qué SQLAlchemy (no SQLModel/Tortoise).

**Worked example (analogo Node):** [`api-node-reference/src/database/`](https://github.com/JonatanAlpirez/api-node-reference/tree/main/src/database). Mismo flujo (generar + aplicar migrations), distinto CLI (alembic vs mikro-orm).

**Checklist S11/S12:** ✓ forward-only migrations, ✓ DB sync strategy definida.

---

### §11. Deployment (S13, S14)

- **Puerto (S13):** default `8787`, configurable via `PORT` env.
- **Dev mode (S14):** `uv run uvicorn src.main:app --reload --port 8787` — auto-reload, logs verbose.
- **Build:** no hay build step. Python corre directo. Para producción: `uvicorn src.main:app --host 0.0.0.0 --port 8787` con workers (`--workers 4`).
- **Docker:** `python:3.12-slim` base image, `uv pip install --system -r requirements.txt`, CMD `uvicorn ...`. Opcional pero recomendado para prod.
- **CI:** GitHub Actions matrix Python 3.12 + 3.13, `uv run pytest` + `uv run ruff check`. Pendiente de crear (no hay `api-python-reference`).

**Worked example (analogo Node):** [`api-node-reference/package.json`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/package.json). El concepto es el mismo, las tools son distintas.

**Checklist S13/S14:** ✓ puerto 8787, ✓ dev con auto-reload, ✓ prod con uvicorn workers.

---

### §12. List patterns (S15, S16)

- **Pagination (S15):** offset-based, `page/limit` (default 1/20, max 100). Pydantic schema en `common/pagination.py` lo valida.
- **Response shape:** `PaginatedResponse[T]` con `data: list[T]` y `pagination: { page, limit, total, has_next }`.
- **Filtering (S16):** whitelist cerrada con Pydantic `Literal[...]` types o `Enum`. Ejemplo: `status: Literal["active", "archived"] | None = None`.
- **Sorting (S16):** whitelist en el schema + mapping en el service. Pydantic valida que `sort` está en la whitelist, el service mapea snake_case externo → atributo del model (`created_at` → `Resource.created_at`). Previene SQL injection por columnas arbitrarias.
- **`has_next`:** calculado como `page * limit < total` en el service.

**Worked example (analogo Node):** [`api-node-reference/src/modules/resource/dto/filter-resource.dto.ts`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/src/modules/resource/dto/filter-resource.dto.ts). Mismo concepto, distinto syntax (Pydantic vs Zod).

**Checklist S15/S16:** ✓ offset-based pagination, ✓ whitelist cerrada para filter/sort, ✓ mapeo snake_case → atributo.

---

### §13. Secrets (S18)

- **Pydantic Settings:** `class Settings(BaseSettings): port: int = 8787; frontend_origin: str; api_key: str = Field(min_length=16); ...`. Lee de `process.env` o `.env` automáticamente.
- **Validación al arranque:** Pydantic falla en la construcción si falta un field o es inválido. **No arranca con config inválida.**
- **`.env`:** Pydantic Settings lo lee automáticamente. No necesitás `python-dotenv` separado.
- **`.env` gitignored, `.env.example` commiteado:** la secret real nunca va al repo.

**Worked example (analogo Node):** [`api-node-reference/src/config/env.ts`](https://github.com/JonatanAlpirez/api-node-reference/blob/main/src/config/env.ts) — el concepto es el mismo (Zod vs Pydantic), la implementación es más concisa en Python.

**Checklist S18:** ✓ Pydantic Settings, ✓ fail-loud al arranque, ✓ `.env` gitignored.

---

### §14. Open questions

No hay tooling Python-specific acá. Si el proyecto tiene preguntas abiertas, listalas en el spec y resolvelas antes/durante implementación.

---

## Por qué este stack (vs Node / Java)

### vs Node + TypeScript (NestJS)

**Ganamos con Python:**
- Sin build step (Python corre directo, no hay TS → JS compile).
- Sintaxis más concisa que TS decorators + NestJS modules (FastAPI es menos verbose).
- Ecosystem Python más maduro para data science, ML, scraping — si el proyecto lo necesita.
- Type hints + Pydantic dan type-safety comparable a TS (con menos ceremonia).

**Perdés con Python:**
- TS type system es end-to-end (más estricto que Python type hints; errores en compile-time).
- `@nestjs/mikro-orm` adapter oficial más pulido que SQLAlchemy + FastAPI wrappers.
- Ecosystem Node más maduro para OpenAPI codegen (NestJS genera spec out-of-the-box; FastAPI también pero requiere más setup con Pydantic).
- Cold start un poco mejor que Node en algunos casos (uvicorn arranca más rápido que NestJS).

### vs Java + Spring Boot

**Ganamos con Python:**
- Mucho menos boilerplate (no hay `pom.xml`, application classes, autowire annotations).
- Arranque más rápido (FastAPI en <1s vs Spring Boot en 5-10s).
- Sintaxis más flexible (Python es más dinámico, menos ceremony que Java).
- Ecosystem más liviano (sin JVM).

**Perdés con Python:**
- Python no es tan type-safe como Java en compile-time (type hints son checked por mypy, no por el runtime).
- FastAPI no tiene el mismo ecosistema enterprise que Spring.
- Spring Boot + Hibernate con Postgres es más "production-tested" a escala.

---

## Stack-specific gotchas (a documentar cuando se cree `api-python-reference`)

10 gotchas que probablemente aparecerán cuando se implemente el reference (a documentar en el `.docs/WALKTHROUGH.md` de `api-python-reference`):

1. **Pydantic v2 vs v1** — la v2 es Rust-based, hay breaking changes.
2. **Async SQLAlchemy session lifecycle** — la session debe cerrarse siempre (context manager o dependency).
3. **Alembic autogenerate** — no detecta todo (cambios de nombre, enum changes, etc.), revisar siempre.
4. **`uvicorn` workers vs async** — si usás `--workers N`, cada worker tiene su propia DB session pool, no hay shared state.
5. **Pydantic Settings con Docker/secrets** — secrets en files vs env vars tienen distinto precedence.

---

## Próximos pasos

- Crear `api-python-reference` (mismo nivel de coverage que `api-node-reference` para Python/FastAPI) — **work pendiente**, ~1-2 días de trabajo.
- CI + Dockerfile para `api-python-reference` cuando exista.
- Documentar los 10 gotchas específicos de Python en su `.docs/WALKTHROUGH.md` (a crear con el reference).

Para el stack Python en sí, este proposal ya está listo para guiar la implementación de un proyecto nuevo que cumpla S1-S16 + S18.
