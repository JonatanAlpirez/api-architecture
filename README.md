# api-architecture

Workspace de docs para las decisiones de arquitectura de nuestras **APIs de backend** de consumo local. Pensado para definir el contrato y la estructura antes de empezar a codear.

**Estado:** requisitos (R1-R8) y decisiones técnicas (Q1-Q21, con Q11/Q12/Q14 fuera de scope) cerrados; próximo: escribir 3 propuestas de arquitectura paralelas.

## Cómo se organiza

| Archivo | Propósito |
|---|---|
| `architecture-decisions.md` | Decisiones técnicas de arquitectura (stack, ORM, OpenAPI, auth, runtime, tests, lint, etc.) + requisitos R1-R8. |
| `architecture-proposal.node-ts.md` | (próximo) Propuesta de stack **Node + TypeScript** — capas, OpenAPI flow, DB e integración con el data warehouse. |
| `architecture-proposal.python.md` | (próximo) Propuesta de stack **Python** — mismo nivel de profundidad que la anterior, para comparar. |
| `architecture-proposal.java-spring.md` | (próximo) Propuesta de stack **Java + Spring Boot** — mismo nivel de profundidad que las anteriores, para comparar. |

Se comparan entre sí y se elige una (o se hibridan partes de cada una) antes de empezar a codear.

Cada decisión relevante que aterrice puede documentarse como un ADR corto (p. ej. `adr-001-orm-drizzle.md`) o como sección dentro de `architecture-proposal.md` — lo que tenga más sentido cuando llegue el momento.

## Por qué existe

- Hoy el frontend consume datos **directo** desde la DB del data warehouse (vía `better-sqlite3`). Esa lectura directa es un atajo.
- La versión "correcta" a futuro: una **API** que exponga esos datos vía HTTP, con spec OpenAPI/Swagger para que el FE genere servicios y modelos automáticamente.
- Este repo es el lugar donde pensamos/documentamos esa API **antes** de empezar a codearla.

---

*Última actualización: 2026-09-29 — `requirements.md` renombrado a `architecture-decisions.md` (el contenido son decisiones de arquitectura, no solo requisitos); versión inicial limpia.*