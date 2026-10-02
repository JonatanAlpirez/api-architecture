# api-architecture

Workspace de **lineamientos para arrancar cualquier API de backend** con uno de los stacks soportados (Node+TS, Python, Java+Spring). Pensado para tener los estándares claros antes de empezar a codear.

**Estado:** playbook agnóstico escrito (14 estándares S1-S14) + 3 referencias de implementación por stack. Listo para arrancar una API nueva eligiendo stack.

## Cómo se organiza

| Archivo | Propósito |
|---|---|
| `playbook.md` | **Punto de entrada.** Lineamientos agnósticos que aplican a cualquier stack: arquitectura en capas, OpenAPI code-first, auth, validación, errores, logging, CORS, testing, lint, datos, runtime. |
| `architecture-decisions.md` | Soporte histórico. El "por qué" de cada lineamiento: comparaciones de opciones, razonamiento, descartes. No es el doc de lectura diaria. |
| `node-ts/architecture-proposal.node-ts.md` | Referencia de implementación del stack **Node + TypeScript** (NestJS, MikroORM, Zod). Cómo aterrizar cada lineamiento del playbook. |
| `python/architecture-proposal.python.md` | Referencia de implementación del stack **Python** (FastAPI, SQLAlchemy 2.0, Pydantic v2). |
| `java-spring/architecture-proposal.java-spring.md` | Referencia de implementación del stack **Java + Spring Boot 3** (Spring Data JPA, PostgreSQL, jakarta.validation). |

## Cómo usarlo

Para arrancar una API nueva:

1. **Elegí el stack** que vas a usar.
2. **Leé `playbook.md`** completo — son los estándares agnósticos que aplican a cualquier API nuestra.
3. **Leé la reference del stack elegido** — encontrás las herramientas por capa y snippets de cómo arrancar.
4. **Arrancá.**

Una decisión nueva que aplique a los 3 stacks → ADR (`adr-NNN-titulo.md`) → estándar nuevo (`S<n+1>` en playbook). Una decisión específica de un stack → va en su reference.

## Por qué existe

- Hoy el frontend consume datos **directo** desde la DB del data warehouse (vía `better-sqlite3`). Esa lectura directa es un atajo.
- La versión "correcta" a futuro: una **API** que exponga esos datos vía HTTP, con spec OpenAPI/Swagger para que el FE genere servicios y modelos automáticamente.
- Este repo es el lugar donde pensamos/documentamos esa API **antes** de empezar a codearla, y donde mantenemos los estándares que aplican a cualquier API nuestra.

---

*Última actualización: 2026-10-02 — playbook agnóstico introducido como punto de entrada del repo; `architecture-decisions.md` queda como soporte histórico; proposals se reframean como "referencias de implementación" (sin rename físico).*