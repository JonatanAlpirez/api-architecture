# Plan — Cerrar el 30% que falta en `api-architecture`

> Repo: [api-architecture](https://github.com/JonatanAlpirez/api-architecture)
> Fecha: 2026-10-03
> Status: **propuesto**, pendiente de OK explícito
> Autor: Devon (developer-agent), a partir de la conversación del 2026-10-03

---

## Contexto

El repo `api-architecture` está al **~70%** de servir como referencia para APIs futuras:

- ✅ Tiene los **cimientos sólidos**: `playbook.md` con 18 estándares agnósticos (S1-S18), 3 `architecture-proposal.<stack>.md` con snippets concretos, `architecture-decisions.md` con Q1-Q26, README actualizado.
- ⚠️ Le falta la **experiencia de uso**: onboarding doc, ejemplo viviente, ADRs formales, navegabilidad de decisiones, versionado.

Consecuencia: para una persona que llega al repo por primera vez, el flujo "cómo arranco una API nueva desde cero" no está documentado. Tiene que descubrirlo leyendo los docs en orden implícito, o preguntando.

**Este plan cierra ese 30% en 3 fases priorizadas** (mínimo viable → formalización → polish).

---

## Estado actual

| Pieza | Estado |
|---|---|
| `playbook.md` (S1-S18) | ✅ Claro, completo, self-contained |
| 3 `architecture-proposal.<stack>.md` con snippets | ✅ Cubren S15-S18 desde el 2026-10-03 |
| Decisiones (Q1-Q26) | ✅ Razonamiento documentado — pero no navegables formalmente |
| `README.md` | ✅ Índice actualizado |
| Genericidad (sin acoplarse a proyectos) | ✅ OK |
| Onboarding doc paso-a-paso | ❌ No existe |
| Proyecto canónico (ejemplo viviente) | ✅ **api-node-reference creado** |
| Proyecto canónico (ejemplo viviente) | ❌ No existe |
| Starter templates funcionales (código copy/paste) | ❌ No existen (solo snippets narrativos) |
| Snapshot table | ❌ Desactualizado (no incluye S15-S18) |
| ADRs formales (`adr-NNN-*.md`) | ❌ No existen |
| Versionado del playbook (`v1.0`, changelog) | ❌ No existe |
| CI de links / snippets | ❌ No existe |

---

## Huecos a cerrar

### H1 — Onboarding doc paso-a-paso

**Problema:** alguien que llega al repo no sabe el flujo. Hoy lo explico de memoria en chat; debería estar escrito.

**Criterio de cierre:** sección en `README.md` (o doc nuevo `ONBOARDING.md`) con flujo paso-a-paso + ejemplo concreto (estilo "arrancar `api-gastos` en NestJS desde cero hasta primer endpoint paginado").

**Esfuerzo:** ~30 min.

---

### H2 — Proyecto canónico (starter funcional real)

**Problema:** las proposals tienen snippets pero no son código que copy/paste funcione. Falta un ejemplo viviente de "así se ve cuando se siguen los 18 estándares".

**Criterio de cierre:** directorio `api-node/` (o repo aparte) con `package.json` + estructura §2 del proposal + `npm install && npm run dev` levanta una API que cumple **TODOS** los 18 estándares. Sirve como referencia de implementación "de oro" para futuros proyectos NestJS.

**Esfuerzo:** ~2-3h (es la Opción 1A de mi lista del 2026-10-03).

---

### H3 — Snapshot table actualizado

**Problema:** el "Snapshot — decisiones por stack" en `architecture-decisions.md` (línea 527) no incluye S15-S18 (paginación, filter/sort, rate limiting, secrets handling) como filas.

**Criterio de cierre:** el Snapshot menciona explícitamente las 4 nuevas capas (o nota que están estandarizadas en S15-S18). Mismo formato tabular que el resto.

**Esfuerzo:** ~15 min.

---

### H4 — Versionado + changelog

**Problema:** sin tag de versión no se sabe si un cambio en el playbook es breaking o non-breaking. Sin changelog, no hay historial de "qué se agregó en cada versión".

**Criterio de cierre:**
- `CHANGELOG.md` con entrada inicial `v1.0` (S1-S18, 2026-10-03) y formato para futuras versiones.
- Tag git `v1.0` en el commit que cierre este plan (para que `git checkout v1.0` recupere el playbook en estado conocido).
- Mención de versionado en el `README.md` (footer o sección).

**Esfuerzo:** ~30 min.

---

### H5 — ADRs formales (Q1-Q26 → `adr-NNN-*.md`)

**Problema:** las decisiones viven como texto en `architecture-decisions.md`. Para alguien que llega en 6 meses sin contexto, navegar Q1-Q26 dentro de un doc grande es fricción.

**Criterio de cierre:**
- Cada Q se extrae a un archivo `adr-NNN-titulo-corto.md` individual (formato estándar: Status, Fecha, Contexto, Decisión, Consecuencias).
- `architecture-decisions.md` se reduce a un índice que linkea a cada ADR.
- Numeración preservada (Q1 → adr-001-*, Q25 → adr-025-*, etc.).

**Esfuerzo:** ~2h (mecánico, copy-paste + estructura).

---

### H6 — CI de links (opcional, polish)

**Problema:** los snippets de las proposals no se testean contra las versiones reales de las librerías. Si NestJS 11 cambia una API, nadie se entera hasta que alguien lo intenta.

**Criterio de cierre:** GitHub Actions workflow que corre `markdown-link-check` en cada PR (links internos rotos). Opcionalmente: linter de code snippets (más caro, scope aparte).

**Esfuerzo:** ~30 min.

---

### H7 — Audit del doc (housekeeping)

**Problema:** después de los cambios recientes (reframe, S15-S18, fix de links), podría haber inconsistencias menores que no detectamos.

**Criterio de cierre:** pasar el repo por una auditoría de:
- Links internos (`architecture-decisions.md` ↔ proposals ↔ playbook)
- Snippets que no compilan (revisión manual)
- Wording consistente (mezcla español/inglés en code identifiers)
- Versiones actualizadas (NestJS 10.x sigue vigente al 2026-10-03)
- Heading levels correctos
- Snapshot table consistente con el resto del doc

**Esfuerzo:** ~45 min.

---

## Plan de ejecución (por fase)

### Fase 1 — Cerrar el 30% mínimo viable (~4-5h)

**Objetivo:** que un newcomer pueda arrancar una API nueva siguiendo el repo, sin ayuda externa.

1. **H1 — Onboarding doc** (~30 min)
2. **H3 — Snapshot table actualizado** (~15 min)
3. **H4 — Versionado + changelog** (~30 min)
4. **H7 — Audit del doc** (~45 min)
5. **H2 — Proyecto canónico / starter funcional** (~2-3h)

**Output:** repo usable end-to-end.

### Fase 2 — Cerrar ADRs (~2h)

**Objetivo:** hacer las decisiones navegables formalmente.

1. **H5 — Q1-Q26 → ADRs formales** (~2h)

**Output:** `architecture-decisions.md` se vuelve un índice; ADRs individuales buscables.

### Fase 3 — Polish (~30 min, opcional)

1. **H6 — CI de links**

**Output:** CI evita regresiones de links.

---

## Orden propuesto (si arrancamos hoy)

```
Fase 1 (4-5h total):
  1. H1 — Onboarding doc              [30 min]  ← doc
  2. H3 — Snapshot table actualizado  [15 min]  ← doc
  3. H4 — Versionado + changelog      [30 min]  ← doc
  4. H7 — Audit del doc               [45 min]  ← doc
  5. H2 — Proyecto canónico            [2-3h]   ← código

  Subtotal Fase 1 housekeeping de docs: 1h45min (sin riesgo, bajo costo)
  Subtotal Fase 1 H2 (código):          2-3h   (el más valioso, esfuerzo mayor)
```

**Justificación del orden:** H1+H3+H4+H7 son housekeeping de docs que dejan el repo consistente antes de invertir en código. H2 es el más caro pero el más valioso (código real que demuestra el patrón).

---

## Criterios de éxito generales

- [ ] Un dev nuevo puede arrancar una API en ~1h siguiendo solo el repo (sin chat)
- [ ] Las decisiones son navegables individualmente (ADRs formales o equivalente)
- [ ] El repo tiene un "release" identificable (`v1.0` con S1-S18)
- [ ] Hay un ejemplo viviente de "así se ve cuando se siguen los 18 estándares" (H2)
- [ ] Los links internos no se rompen silenciosamente (CI o checklist)
- [ ] El Snapshot table está sincronizado con el playbook

---

## No-objetivos (explícitos)

- ❌ **No** convertir el repo en un framework generador de código (boilerplate-from-template). Sigue siendo **docs + un canónico**.
- ❌ **No** agregar stacks nuevos (Go, Rust, Elixir) en este plan. Cada uno merece su propio `architecture-proposal.<stack>.md` con su propio gathering. Alcance separado.
- ❌ **No** tocar `shopping-tracker`. Es proyecto independiente (NestJS real, pre-playbook). Cualquier alineación con el playbook es decisión del proyecto, no del repo.
- ❌ **No** agregar estándares nuevos al playbook (S19+). Eso es alcance de conversación aparte (ayer hablamos de S19-S23 y los difiero).

---

## Bitácora

- **2026-10-03** — Plan propuesto (este doc). Resultado de la conversación sobre "¿está el repo preparado para servir de referencia?".
- **2026-10-03 (cierre sesión)** — **H2 cerrado**: `api-node-reference` creado como repo privado en github.com/JonatanAlpirez/api-node-reference. Path local `~/Documents/projects/api-references/api-node-reference/`. Cumple 17/18 estándares S1-S18 verificados en runtime (S17 rate limiting pendiente). 4/4 tests pasan (Vitest + supertest + unplugin-swc). Server arranca con `nest start --watch`. README de `api-architecture` linkea al nuevo repo como "ejemplo viviente".