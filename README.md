# api-architecture

Workspace de **lineamientos para arrancar cualquier API de backend** con uno de los stacks soportados (Node+TS, Python, Java+Spring). Pensado para tener los estándares claros antes de empezar a codear.

**Estado:** playbook agnóstico escrito (17 estándares S1-S16 + S18) + spec template + 3 stack-specific spec templates. El flow es **spec-driven**: arrancás una API nueva escribiendo una spec, no clonando código.

## Cómo se organiza

| Archivo | Propósito |
|---|---|
| `playbook.md` | **Constitución.** 17 estándares agnósticos (S1-S16 + S18) que aplican a cualquier API nuestra: arquitectura en capas, OpenAPI code-first, auth, validación, errores, logging, CORS, testing, lint, datos, runtime, list patterns, secrets. |
| `spec-template.md` | **Entry point para arrancar un proyecto nuevo.** Template para escribir la spec de tu API (dominio, endpoints, validaciones, etc.). Si llenás este template y lo implementás, el resultado cumple el playbook **por construcción**. |
| `architecture-decisions.md` | Soporte histórico. El "por qué" de cada lineamiento: comparaciones de opciones, razonamiento, descartes. Q1-Q26. No es el doc de lectura diaria. |
| `node-ts/architecture-proposal.node-ts.md` | **Stack-specific spec template** para Node+TS. Dado el spec-template genérico, qué herramientas usar (NestJS, MikroORM, Zod, etc.) para cada sección. |
| `python/architecture-proposal.python.md` | Stack-specific spec template para Python (FastAPI, SQLAlchemy 2.0, Pydantic v2). |
| `java-spring/architecture-proposal.java-spring.md` | Stack-specific spec template para Java+Spring (Spring Boot 3, Spring Data JPA, PostgreSQL, jakarta.validation). |
| `api-references/` _(repos hermanos, fuera de este repo)_ | **Ejemplos de output ya implementado**, no starters para clonar. Cada stack tiene su propio repo en `~/Documents/projects/api-references/` (e.g. `api-node-reference/`). Se consultan para ver "qué se siente" un proyecto que cumple el playbook, no como base. |

## Cómo usarlo (flow spec-driven)

Para arrancar una API nueva:

1. **Leé `playbook.md`** — los 17 estándares agnósticos. Entendé el "qué" y el "por qué".
2. **Leé `spec-template.md`** — el template para escribir la spec de tu proyecto.
3. **Elegí el stack** y leé la `architecture-proposal.<stack>.md` correspondiente — te dice qué herramientas pluguear en cada sección del template.
4. **Escribí `specs/000-bootstrap/spec.md`** en tu proyecto nuevo, completando el template con tu dominio.
5. **Implementá la spec** (vos, otro dev, o un agente con coding tools).
6. **Validá contra el checklist** al final del spec-template — ¿cumple S1-S16 + S18?

> **NO** se clona código de los `api-<stack>-reference/`. Esos repos son ejemplos del output que produce este proceso, no el input. Si querés ver "qué se siente" un proyecto ya implementado que cumple el playbook, mirá el reference del stack elegido; pero partí siempre de la spec, no del clon.

Una decisión nueva que aplique a los 3 stacks → ADR en `architecture-decisions.md` → estándar nuevo (`S<n+1>` en playbook). Una decisión específica de un stack → va en su stack-specific spec template. Cambios de versión (bump de NestJS, Zod, etc.) → van en el reference del stack, no acá.

## Por qué existe

- Hoy el frontend consume datos **directo** desde la DB del data warehouse (vía `better-sqlite3`). Esa lectura directa es un atajo.
- La versión "correcta" a futuro: una **API** que exponga esos datos vía HTTP, con spec OpenAPI/Swagger para que el FE genere servicios y modelos automáticamente.
- Este repo es el lugar donde pensamos/documentamos esa API **antes** de empezar a codearla, y donde mantenemos los estándares que aplican a cualquier API nuestra.

---

*Última actualización: 2026-10-08 — reframe a flow spec-driven. `spec-template.md` introducido como entry point para proyectos nuevos. `api-<stack>-reference/` repos demoteados de "starter para clonar" a "ejemplo de output". `playbook.md` extendido a 17 estándares (S1-S16 + S18). Stack-specific spec templates (en `node-ts/`, `python/`, `java-spring/`) pendientes de reframe para alinearse con el nuevo flow.*