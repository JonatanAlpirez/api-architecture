# api-architecture

Workspace de docs para las decisiones de arquitectura de nuestras **APIs de backend** de consumo local. Pensado para definir el contrato y la estructura antes de empezar a codear.

**Estado:** requisitos (R1-R8) y decisiones técnicas (Q1-Q22, con Q11/Q12/Q14 fuera de scope) cerrados, **3 propuestas de arquitectura escritas** (Node + TS, Python + FastAPI, Java + Spring Boot) — listas para comparar y elegir stack.

## Cómo se organiza

| Archivo | Propósito |
|---|---|
| `architecture-decisions.md` | Decisiones técnicas de arquitectura (stack, ORM, OpenAPI, auth, runtime, tests, lint, CORS, etc.) + requisitos R1-R8. |
| `architecture-proposal.node-ts.md` | Propuesta de stack **Node + TypeScript** (NestJS 10 + MikroORM + Zod). |
| `architecture-proposal.python.md` | Propuesta de stack **Python** (FastAPI + SQLAlchemy 2.0 + Pydantic v2). |
| `architecture-proposal.java-spring.md` | Propuesta de stack **Java + Spring Boot 3** (Spring Data JPA + PostgreSQL). |

Se comparan entre sí y se elige una (o se hibridan partes de cada una) antes de empezar a codear.

Cada decisión relevante que aterrice puede documentarse como un ADR corto (p. ej. `adr-001-orm.md`) o como sección dentro de `architecture-proposal.md` — lo que tenga más sentido cuando llegue el momento.

## Por qué existe

- Hoy el frontend consume datos **directo** desde la DB del data warehouse (vía `better-sqlite3`). Esa lectura directa es un atajo.
- La versión "correcta" a futuro: una **API** que exponga esos datos vía HTTP, con spec OpenAPI/Swagger para que el FE genere servicios y modelos automáticamente.
- Este repo es el lugar donde pensamos/documentamos esa API **antes** de empezar a codearla.

---

*Última actualización: 2026-10-01 — README sincronizado con la finalización de las 3 proposals (Node + TS, Python, Java Spring Boot); paths estandarizados a OpenAPI 3.x (`/resources`, `{id}`).*