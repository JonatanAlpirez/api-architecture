# Architecture Proposal — Python (FastAPI)

> **Estado:** propuesta cerrada — cubre los 7 puntos del plan inicial (stack, estructura, OpenAPI, DB, sync, endpoints, setup) + tradeoffs vs Node + TS y Java Spring Boot. Próximo: comparar con las otras 2 propuestas ([`architecture-proposal.node-ts.md`](../node-ts/architecture-proposal.node-ts.md), [`architecture-proposal.java-spring.md`](../java-spring/architecture-proposal.java-spring.md)).
>
> Todas las decisiones referenciadas viven en [`architecture-decisions.md`](../architecture-decisions.md). Esta propuesta **asume** que esas decisiones están cerradas y solo aterriza nombres concretos, paths y código.

---

## 1. Stack justificado

| Capa | Decisión | Versión target | Por qué |
| --- | --- | --- | --- |
| Lenguaje | **Python** | 3.12+ | Type hints nativos (`dict`, `list`, `X \| None`), async nativo, performance mejorado vs 3.11 |
| Framework HTTP | **FastAPI** | 0.110+ | Async nativo, OpenAPI automático, Pydantic integration, OpenAPI 3.x first-class |
| ORM | **SQLAlchemy** | 2.0+ (async) | Mature, async session, type-safe queries via `Mapped[]`, ecosystem enorme |
| Validación | **Pydantic** | v2 | Single source: schema + tipos + OpenAPI; Rust core (10x+ vs v1) |
| OpenAPI integration | Built-in FastAPI | (latest) | OpenAPI se genera desde endpoints + Pydantic schemas automáticamente |
| Tests | **pytest** + **pytest-asyncio** | (latest) | Standard en Python; `asyncio` mode para endpoints async |
| HTTP testing | **httpx AsyncClient** | (latest) | Async test client compatible con FastAPI |
| Logging | **Loguru** | 0.7+ | Plug-and-play, mejor DX que stdlib `logging`; structured JSON via `serialize=True` |
| HTTP logger | Custom middleware o `fastapi-logger` | — | Loggea cada request/response con duración |
| Lint/format | **Ruff** | 0.1+ | Reemplaza flake8 + black + isort + más; una tool, ultra-rápido (Rust core) |
| Build | (no build step) | — | Python corre directo; para producción: `pyinstaller` o solo `uvicorn` |
| Migraciones | **Alembic** | 1.13+ | Estándar de facto del ecosistema SQLAlchemy |
| CORS | `fastapi.middleware.cors.CORSMiddleware` | (built-in) | Built-in FastAPI; config por env var `FRONTEND_ORIGIN` |
| Package manager | **uv** | (latest) | Rápido, Rust-based; reemplaza pip/poetry/virtualenv |

**Por qué este stack sobre las alternativas evaluadas** (ver Q2/Q4/Q5/Q17/Q21 en [`architecture-decisions.md`](../architecture-decisions.md)):

- **FastAPI sobre Flask/DRF/Starlette**: OpenAPI first-class, async nativo, Pydantic integration. Flask es sync-only; DRF acoplado a Django; Starlette low-level (es lo que está debajo de FastAPI).
- **SQLAlchemy 2.0 async sobre SQLModel/Tortoise**: ecosistema maduro, async session official, Alembic integration battle-tested. SQLModel es Pydantic + SQLAlchemy pero menos maduro; Tortoise es active-record (no data-mapper).
- **Pydantic v2 sobre Marshmallow/dataclasses**: Rust core (perf), integración nativa con FastAPI, OpenAPI automático. v1 era Python puro; v2 es Rust-based.
- **Ruff sobre flake8 + black + isort**: una tool, una config, ultra-rápido. Moderno replacement del stack clásico de lint/format Python.
- **uv sobre pip/poetry**: instalación y resolución de deps 10-100x más rápido.

---

## 2. Estructura de carpetas

Monolito modular — un solo deployable, routers independientes entre sí (bajo acoplamiento, alta cohesión).

```
api-python/
├── src/
│   ├── main.py                              # FastAPI app factory; CORS, routers, exception handlers
│   ├── app.py                               # instancia app para uvicorn (módulo separado para no ejecutar side effects en import)
│   │
│   ├── config/
│   │   ├── __init__.py
│   │   ├── settings.py                      # Pydantic BaseSettings para env vars (PORT, FRONTEND_ORIGIN, API_KEY, DATABASE_URL)
│   │   └── logging.py                       # Setup de Loguru (JSON output, sinks)
│   │
│   ├── common/
│   │   ├── __init__.py
│   │   ├── errors.py                        # Excepciones custom (ResourceNotFoundError, ValidationError) + handlers
│   │   ├── middleware.py                    # Auth middleware (X-API-Key), logging middleware
│   │   ├── deps.py                          # FastAPI Depends comunes (get_db, get_current_api_key)
│   │   └── pagination.py                    # Pydantic schemas para paginación (page, page_size)
│   │
│   ├── database/
│   │   ├── __init__.py
│   │   ├── session.py                       # SQLAlchemy async session factory + dependency injection
│   │   ├── base.py                          # Declarative base para ORM models
│   │   └── migrations/                      # archivos de Alembic (generados con alembic revision --autogenerate)
│   │
│   ├── modules/
│   │   └── resource/                       # ejemplo: módulo "resource" (otros features siguen este patrón)
│   │       ├── __init__.py
│   │       ├── models.py                   # SQLAlchemy 2.0 declarative model
│   │       ├── schemas.py                  # Pydantic schemas (Create, Update, Response, Query)
│   │       ├── service.py                  # Business logic (orquesta models, sin HTTP)
│   │       ├── router.py                   # FastAPI APIRouter (rutas HTTP, validación, delega a service)
│   │       └── test_router.py              # pytest + httpx AsyncClient integration test
│   │
│   ├── health/
│   │   ├── __init__.py
│   │   └── router.py                       # GET /health → { status: 'ok' } (sin auth)
│   │
│   └── scripts/
│       └── sync.py                         # CLI script: lee DB fuente (sqlite3) → escribe DB API (SQLAlchemy async)
│
├── data/
│   └── api.db                              # SQLite DB de la API (gitignored)
│
├── .env.example                            # PORT, FRONTEND_ORIGIN, API_KEY, DATABASE_URL, SOURCE_DB_URL, LOG_LEVEL
├── pyproject.toml                          # dependencias + ruff config + pytest config
├── uv.lock                                 # uv lockfile (reproducible installs)
├── alembic.ini                             # Alembic config
└── README.md
```

### Convenciones de FastAPI (antes del patrón)

Cuatro cosas que confunden al que viene de NestJS / Express / Django:

**1. `modules/<feature>/` = organización por feature, no por capa.** FastAPI (al igual que NestJS/Angular) agrupa **todo** lo relativo a un concepto de negocio en una carpeta: model, schemas, service, router, tests. No hay `controllers/`, `services/`, `models/` globales — eso sería por capa técnica. Cada feature es independiente.

**2. `router.py` ≠ "controller", pero juega el mismo rol.** Es una instancia de `APIRouter()` que agrupa endpoints de un feature:

```python
from fastapi import APIRouter, Depends, status
from .service import ResourceService
from .schemas import ResourceResponse, CreateResourceRequest

router = APIRouter(prefix="/resources", tags=["resources"])

@router.get("/", response_model=list[ResourceResponse])
async def list_resources(
    service: ResourceService = Depends(),
) -> list[ResourceResponse]:
    return await service.find_all()
```

No es una "clase" como en NestJS — es un módulo Python con funciones decoradas. La DI se hace vía `Depends()` (FastAPI resuelve el grafo de dependencias automáticamente).

**3. Pydantic models son single-source para validación + tipos + OpenAPI.** A diferencia de NestJS (donde Zod + `nestjs-zod` generan el OpenAPI), en FastAPI los Pydantic models son la fuente directa:

```python
from pydantic import BaseModel, Field

class CreateResourceRequest(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    description: str | None = Field(default=None, max_length=500)
```

El mismo schema valida el body request, genera el JSON Schema para OpenAPI, y sirve como tipo para la response (via `response_model=`).

**4. SQLAlchemy 2.0 declarative models ≠ Pydantic schemas.** Es importante no confundirlos:

- **ORM model** (`models.py`): representa la tabla en DB. Tiene columnas, relaciones, índices. Vive en la sesión SQLAlchemy.
- **Pydantic schema** (`schemas.py`): representa el contrato HTTP. Tiene campos con validaciones. Vive en el request/response.

Se mapean entre sí explícitamente en el service (no automático como en algunos ORMs).

---

## 3. Flujo OpenAPI

```
FastAPI endpoints + Pydantic schemas
        │
        │ (runtime: FastAPI genera OpenAPI automático)
        ▼
OpenAPI spec 3.0 (auto-generado)
        │
        │ servido en:
        ├── /docs       → Swagger UI (default FastAPI)
        ├── /redoc      → ReDoc UI (default FastAPI)
        └── /openapi.json → spec.json (para FE codegen)
                  │
                  │ FE corre: openapi-typescript http://localhost:8787/openapi.json -o src/api/types.ts
                  ▼
            src/api/types.ts (tipos TS)
```

**Setup en `main.py`**:

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from contextlib import asynccontextmanager
from .config.settings import settings
from .modules.resource.router import router as resource_router
from .health.router import router as health_router

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: nada por ahora (DB se conecta por sesión)
    yield
    # Shutdown: cleanup si es necesario

app = FastAPI(
    title="API de [recurso]",
    description="Backend local para servir datos del data warehouse",
    version="1.0",
    lifespan=lifespan,
)

# CORS (Q22)
app.add_middleware(
    CORSMiddleware,
    allow_origins=[settings.FRONTEND_ORIGIN],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Routers
app.include_router(health_router)
app.include_router(resource_router, dependencies=[Depends(verify_api_key)])
```

**FE workflow** (referencia, no parte de este repo):

```bash
# Una vez (o en CI cuando cambia el spec):
pnpm dlx openapi-typescript http://localhost:8787/openapi.json -o src/api/types.ts
```

El FE importa los tipos generados sin acoplamiento a un cliente HTTP específico (decisión Q9).

---

## 4. Esquema DB

SQLite, mismo motor que la DB fuente. Migraciones forward-only via Alembic.

> **Escenarios posibles** (la elección se difiere a implementación, ver Q10 en [`architecture-decisions.md`](../architecture-decisions.md)):
> - **A) Greenfield / API-first:** la DB de la API es la **única** fuente de verdad. Datos nacen vía `POST /resources` (R8 CRUD desde v1).
> - **B) Alongside existing DB (caso actual):** DB fuente pre-existente; sync poblará la DB de la API (ver §5).
> - **C) Source sigue activa:** sync periódico o incremental.

**ORM model example** (recursos del dominio siguen este patrón):

```python
# modules/resource/models.py
from sqlalchemy import String, Integer, DateTime, func
from sqlalchemy.orm import Mapped, mapped_column
from datetime import datetime
from src.database.base import Base

class Resource(Base):
    __tablename__ = "resources"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    name: Mapped[str] = mapped_column(String(100), nullable=False)
    description: Mapped[str | None] = mapped_column(String(500), nullable=True)
    created_at: Mapped[datetime] = mapped_column(DateTime, server_default=func.now())
```

**Migraciones**:

- Inicializar: `alembic init src/database/migrations`.
- Generar: `alembic revision --autogenerate -m "add resource table"`.
- Aplicar: `alembic upgrade head`.
- Archivos commiteados al repo.
- Forward-only (ver Q6 en [`architecture-decisions.md`](../architecture-decisions.md)).

**Tabla `alembic_version`** (auto-manejada por Alembic):
- Lleva registro de qué migraciones se aplicaron.
- Solo aplica las nuevas al `alembic upgrade head`.

**Path local**: `data/api.db` (gitignored).

**Driver**: `sqlite3` stdlib (built-in Python) vía `create_async_engine("sqlite+aiosqlite:///./data/api.db")`. Sin servicio externo corriendo (Q4 — SQLite embedido).

---

## 5. Plan de sync (solo si escenario B o C)

> **Si el escenario es A (greenfield / API-first):** esta sección **no aplica**. La DB de la API se crea vacía desde migraciones y los datos nacen vía `POST /resources` (R8). En ese caso, eliminar `SOURCE_DB_URL`, el script `src/scripts/sync.py`, y el comando `uv run sync`. Mantener §4 (esquema DB) y §6 (endpoints) tal cual.

Script CLI que copia datos desde la DB fuente (SQLite del data warehouse) hacia la DB de la API. **Idempotente** — re-ejecutable sin duplicar.

```python
# scripts/sync.py (esqueleto)
import sqlite3
import asyncio
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker
from src.config.settings import settings
from src.modules.resource.models import Resource

async def main():
    # 1. Conectar a la DB fuente (read-only, sync sqlite3 — sin SQLAlchemy)
    source = sqlite3.connect(settings.SOURCE_DB_URL)
    source.row_factory = sqlite3.Row  # access by column name

    # 2. Conectar a la DB de la API via SQLAlchemy async
    engine = create_async_engine(settings.DATABASE_URL)
    async_session = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

    # 3. Sync por entidad (transacción por batch)
    async with async_session() as session:
        async with session.begin():
            source_rows = source.execute("SELECT * FROM source_table").fetchall()
            for row in source_rows:
                # Mapear source row → ORM model shape (puede haber diferencias de schema)
                await session.merge(Resource(
                    id=row["id"],
                    name=row["name"],
                    description=row.get("description"),
                ))

    # 4. Cleanup
    source.close()
    await engine.dispose()

if __name__ == "__main__":
    asyncio.run(main())
```

**Comando**: `uv run sync` (alias de `python src/scripts/sync.py`).

**Cuándo corre**:
- Dev: manual cuando el FE necesita data fresca.
- Prod: después de las migraciones en cada deploy (cron o webhook — fuera de scope v1).

**Decisiones de sync** (ver Q10 en [`architecture-decisions.md`](../architecture-decisions.md)):
- Sync one-way (fuente → API).
- API mantiene su propia DB; la fuente no se toca en runtime.
- Si la DB crece, sync incremental con `WHERE updated_at > last_sync` — pero v1 hace full sync, se optimiza si la performance lo demanda.

---

## 6. Endpoints iniciales

CRUD completo desde v1 (R8). Ejemplo con `resource` (los demás recursos siguen el patrón).

| Método | Path | Auth | Body | Response | Status |
| --- | --- | --- | --- | --- | --- |
| `GET` | `/resources` | `X-API-Key` | — | `list[ResourceResponse]` | 200 |
| `GET` | `/resources/{id}` | `X-API-Key` | — | `ResourceResponse` | 200 / 404 |
| `POST` | `/resources` | `X-API-Key` | `CreateResourceRequest` | `ResourceResponse` | 201 / 422 |
| `PUT` | `/resources/{id}` | `X-API-Key` | `UpdateResourceRequest` | `ResourceResponse` | 200 / 404 / 422 |
| `PATCH` | `/resources/{id}` | `X-API-Key` | `UpdateResourceRequest` (parcial) | `ResourceResponse` | 200 / 404 / 422 |
| `DELETE` | `/resources/{id}` | `X-API-Key` | — | — | 204 / 404 |
| `GET` | `/health` | — | — | `{ status: 'ok' }` | 200 |

**Headers siempre presentes**:
- Request: `X-API-Key: ***` (excepto `/health`).
- Request: `Content-Type: application/json` (en POST/PUT/PATCH).
- Response: `Content-Type: application/json` + CORS headers (`Access-Control-Allow-Origin`, etc.).

**Envelope de error** (ver Q19 en [`architecture-decisions.md`](../architecture-decisions.md)):

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

**Validación Pydantic** (Q17): si el body no cumple el schema, devuelve 422 con `details` listando los campos inválidos (FastAPI nativo).

**Códigos de error comunes**:
- `VALIDATION_ERROR` → 422 (Pydantic native)
- `UNAUTHORIZED` → 401 (falta `X-API-Key` o inválido)
- `NOT_FOUND` → 404
- `INTERNAL_ERROR` → 500

---

## 7. Setup commands

```bash
# Setup inicial
uv sync                       # instala deps desde uv.lock

# Migraciones
uv run alembic upgrade head   # aplica migraciones pendientes

# Sync inicial desde la DB fuente
uv run sync                   # python src/scripts/sync.py

# Dev (auto-reload con uvicorn)
uv run dev                    # uvicorn src.app:app --reload --port 8787

# Tests
uv run pytest                 # pytest -v (corre una vez)
uv run pytest --watch         # pytest-watch (requiere pytest-watch)
uv run pytest --cov           # con coverage

# Lint / format
uv run lint                   # ruff check
uv run format                 # ruff format
```

**`pyproject.toml` scripts**:

```toml
[project]
name = "api-python"
requires-python = ">=3.12"
# ...

[tool.uv]
dev-dependencies = ["pytest", "pytest-asyncio", "pytest-cov", "pytest-watch"]

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "W", "I", "N", "UP", "B", "C4", "SIM"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["src"]

[project.scripts]
dev = "uvicorn src.app:app --reload --port 8787"
sync = "python src/scripts/sync.py"
```

**Variables de entorno** (`.env.example`):

```bash
PORT=8787
FRONTEND_ORIGIN=http://localhost:5173
API_KEY=***
DATABASE_URL=sqlite+aiosqlite:///./data/api.db
# SOURCE_DB_URL solo si escenario B/C (ver §4-§5). En escenario A (greenfield), eliminar.
SOURCE_DB_URL=sqlite:///./path/to/data-warehouse.db
LOG_LEVEL=DEBUG
```

---

## Tradeoffs vs las otras 2 propuestas

### vs Node + TypeScript (NestJS)

**Ganamos**:
- **Type hints nativos en Python 3.12+** que se sienten casi como TS (`def find_one(id: int) -> Resource | None:`). El IDE autocompleta y refactorea con la misma calidad.
- **Pydantic v2 (Rust core)** es el equivalente de Zod pero validado en runtime por una librería nativa — menos overhead que TS compilation step.
- **uv** como package manager es comparable a `pnpm` en velocidad.
- Python es el lenguaje "lingua franca" del data science — si el FE o el data warehouse crecen a data science / ML, Python es el camino.

**Perdés**:
- **No hay compile-time type checking** como TS. Pydantic valida en runtime (no en build). Esto es un trade-off importante — errores de tipo aparecen en producción, no en CI.
- **Runtime performance** es menor que Node (Python ~5-10x más lento que Node en CPU-bound tasks, comparable en I/O-bound).
- **Async story es más reciente** en Python. Async/await funciona pero el ecosystem todavía tiene paquetes sync-only. FastAPI lo maneja bien, pero integraciones pueden ser tricky.

### vs Java + Spring Boot

**Ganamos**:
- **Concisión significativa**: ~3-5x menos líneas que Java para el mismo feature (Pythonic syntax + Pydantic reduce boilerplate).
- **Arranque rápido** (~1-2s vs ~5-10s de Spring Boot).
- **Iteración más rápida**: edit code → reload (uvicorn --reload) es instantáneo vs Spring Boot DevTools que tarda varios segundos.
- **Type hints + Pydantic** dan DX cercana a Java con tipos (menos ceremony que TS, menos boilerplate que Java).

**Perdés**:
- **Type-safety runtime, no compile-time** (ya mencionado arriba). Java es el rey del compile-time type-safety.
- **Ecosystem enterprise menos maduro** que Java/Spring. Para security distribuida, transactions distribuidas, etc., Spring gana.
- **GIL (Global Interpreter Lock)** limita paralelismo CPU-bound (no afecta I/O-bound como API HTTP).

---

## Próximos pasos si esta propuesta se aprueba

1. **Implementar el esqueleto**: scaffolding con `uv init`, FastAPI app, un módulo `resource` mínimo (model + schemas + service + router + test).
2. **Validar manualmente**:
   - Swagger UI en `http://localhost:8787/docs`
   - Un POST → GET → PATCH → DELETE con `curl` o Postman
   - El spec en `/openapi.json` genera tipos TS correctos via `openapi-typescript`
3. **Implementar auth + CORS** reales y testear con un FE mínimo (curl + browser).
4. **Implementar sync** desde la DB fuente para una entidad de ejemplo (si escenario B/C aplica).

---

*Propuesta cerrada 2026-10-01 — list para review.*