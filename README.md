# api-architecture

Workspace de docs para las decisiones de arquitectura de nuestras **APIs de backend** de consumo local. Arranca pensado para alimentar `main-dashboard`, pero el scope puede crecer a otros módulos (finance, reminders, etc.).

**Estado:** levantamiento de requisitos (previo a propuesta de arquitectura).

## Cómo se organiza

| Archivo | Propósito |
|---|---|
| `requirements.md` | Requisitos confirmados + preguntas abiertas que faltan resolver. |
| `architecture-proposal.node-ts.md` | (próximo) Propuesta de stack **Node + TypeScript** — capas, OpenAPI flow, DB e integración con `gym_training-data/`. |
| `architecture-proposal.python.md` | (próximo) Propuesta de stack **Python** — mismo nivel de profundidad que la anterior, para comparar. |

Se comparan entre sí y se elige una (o se hibridan partes de cada una) antes de empezar a codear.

Cada decisión relevante que aterrice puede documentarse como un ADR corto (p. ej. `adr-001-orm-drizzle.md`) o como sección dentro de `architecture-proposal.md` — lo que tenga más sentido cuando llegue el momento.

## Por qué existe

- `main-dashboard` hoy consume datos de health **directo** desde `gym_training-data/DB/gym_tracker.db` (vía `better-sqlite3` en el proyecto hermano `health-dashboard`). Esa lectura directa es un atajo.
- La versión "correcta" a futuro: una **API** que exponga esos datos vía HTTP, con spec OpenAPI/Swagger para que el FE genere servicios y modelos automáticamente.
- Este repo es el lugar donde pensamos/documentamos esa API **antes** de empezar a codearla.

## Vecinos relevantes

- `~/Documents/gym_training-data/` — fuente de datos canónica (SQLite, 4 tablas + catálogo de ejercicios). La API debería consumir/exponer estos datos.
- `~/Documents/projects/main-dashboard/` — frontend estricto (Vite + React), consume APIs externas vía `openapi-typescript`. Primer cliente de la API que diseñemos acá.
- `~/Documents/projects/health-dashboard/` — implementación previa que lee SQLite directo. Referencia útil de **qué endpoints necesita el FE**.

---

*Última actualización: 2026-07-11 — creación del repo, requirements gathering en curso.*