# Requisitos — api-architecture

> Versión estándar inicial. Doc vivo: se actualiza conforme aterricen nuevas decisiones.
> Próximos entregables en este repo: `architecture-proposal.node-ts.md`, `architecture-proposal.python.md` y `architecture-proposal.java-spring.md` — tres propuestas paralelas comparables entre sí.

---

## Requisitos

| ID   | Requisito                                                                                                | Notas                                                                                                       |
| ---- | -------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| R1   | API documentada con **Swagger / OpenAPI 3.x**                                                            | El FE debe poder generar servicios y modelos automáticamente a partir del spec.                             |
| R2   | Arquitectura en capas **Controller → Service → Repository**                                              | Origen Spring Boot/Java. Abierto a equivalente moderno en cualquier stack.                                  |
| R3   | Stack **moderno y estandarizado**                                                                        | Sin preferencia rígida por ecosistema; evaluamos opciones.                                                  |
| R4   | Base de datos **relacional**                                                                             | —                                                                                                           |
| R5   | Reutilizar los datos del data warehouse consolidado                                                    | Hoy se usa SQLite; la DB destino de la API puede ser otra.                                                  |
| R6   | Consumo **local**                                                                                        | Primer cliente: el frontend (dashboard web).                                                              |
| R7   | Antes de la propuesta: preguntar dudas / recomendaciones + documentar reqs/pendientes                     | Hecho en este doc.                                                                                          |
| R8   | Soporte **CRUD completo desde v1** (GET + POST + PUT/PATCH + DELETE)                                   | Read-only queda descartado: la API debe poder persistir datos desde el inicio, no agregar escritura en v2. |

_(El shape concreto de modelos/entidades se documenta en cada `architecture-proposal.*.md` cuando aterricemos el código.)_

---

## Decisiones

> **Estructura por pregunta:** cada una sigue el patrón **Definición → Por qué importa → Comparación → Decisión**.

### Stack / runtime

#### Q1. ¿Qué stack evaluamos?

Evaluamos **tres stacks en paralelo** y comparamos:

- **A) Node + TypeScript** — codegen OpenAPI maduro, tooling moderno, ecosistema amplio para este tipo de servicios.
- **B) Python** — ecosistema data/biomédico fuerte, alineado con el background de Jonatan, FastAPI da OpenAPI first-class.
- **C) Java + Spring Boot** — stack más cercano a la formación y rol actual de Jonatan (Spring Boot es su origen, R2 lo confirma), type system más fuerte, ecosistema más maduro del mercado enterprise; tradeoff: más boilerplate, JVM startup, iteración más lenta en dev.

**Implicación:** las preguntas Q2, Q3, Q4, Q5, Q15, Q20, Q21 se aterrizan **en cada propuesta**, no en este doc.

**Decisión:** A, B y C — los tres en paralelo.

#### Q2. ¿Qué es un framework HTTP y qué opciones hay?

**Definición:** un framework HTTP es la capa entre los requests HTTP entrantes y tu lógica de negocio. Encapsula:
- **Routing** — mapeo URL → handler (ej: `GET /resources/123` → `resourceController.show`).
- **Parsing** — extracción de path params, query string, JSON body, headers.
- **Serialization** — tus objetos ↔ JSON de respuesta.
- **Middleware** — auth, logging, CORS, error handling, rate limit, compresión.
- **Status codes + headers** — semántica HTTP correcta (200, 201, 404, 500, `Content-Type`, etc.).
- **Generación de OpenAPI** (en los modernos) — el spec se genera desde el código, no se mantiene a mano.

Sin framework, usarías `node:http` o `http.server` puro y reimplementarías todo eso a mano. El framework es la abstracción que evita esa fatiga.

**Opciones para Node + TypeScript:**

| Framework     | Pros                                                                                              | Contras                                              | Mejor cuando                                                                |
| ------------- | ------------------------------------------------------------------------------------------------- | ---------------------------------------------------- | ---------------------------------------------------------------------------- |
| **Hono**      | Ultra-rápido, minimal, edge-friendly, OpenAPI vía `@hono/zod-openapi`, multi-runtime              | Ecosistema chico vs Express                          | APIs pequeñas/modernas, queremos correr en Node/Bun/Workers/Deno             |
| **Fastify**   | Muy rápido, JSON Schema nativo, ecosistema maduro                                                 | Más boilerplate que Hono                             | APIs medianas con foco en performance                                        |
| **Express**   | Ubicuo, millones de middlewares third-party                                                       | API vieja, tipado flojo, no edge                     | Compatibilidad con código/ecosistema legacy                                   |
| **NestJS**    | DI, decoradores, "Spring Boot de Node", opinionated                                               | Curva de aprendizaje, verboso                        | Equipos grandes con estructura opinionated                                    |
| **Koa**       | Minimalista, sucesor espiritual de Hono en diseño                                                 | Muy low-level                                        | Control fino total del stack                                                 |
| **Elysia**    | Bun-native, TS-first, performance altísima                                                         | Acoplado a Bun runtime                               | Si elegimos Bun como runtime                                                 |

**Opciones para Python:**

| Framework             | Pros                                                                  | Contras                                                | Mejor cuando                                                    |
| --------------------- | --------------------------------------------------------------------- | ------------------------------------------------------ | ---------------------------------------------------------------- |
| **FastAPI**           | OpenAPI automático, async nativo, validación con Pydantic, muy popular | Relativamente nuevo (2018)                              | APIs modernas, queremos OpenAPI first-class                      |
| **Flask**             | Simple, ubicuo, maduro                                                | No async nativo, OpenAPI vía flask-smorest/flask-pydantic | APIs chicas, equipo Python clásico                              |
| **Django REST Framework** | Batteries-included (ORM + admin + auth + router)                  | Acoplado a Django, pesado                              | Si ya usamos Django, o queremos todo integrado                  |
| **Starlette**         | ASGI puro, base sobre la que está construido FastAPI                  | Low-level, sin OpenAPI out-of-the-box                   | Máximo control sobre el async stack                             |
| **Litestar**          | Más tipado que FastAPI, OpenAPI built-in                              | Comunidad chica                                        | Equipos que valoran type-safety fuerte                           |
| **Tornado**           | Async desde 2010, maduro                                              | Sintaxis menos pythonic moderna                        | Sistemas real-time / websockets                                  |

**Opciones para Java:**

| Framework             | Pros                                                                                              | Contras                                                  | Mejor cuando                                                                              |
| --------------------- | ------------------------------------------------------------------------------------------------- | -------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| **Spring Boot 3**     | De facto del mercado Java, batteries-included (DI, security, data, web), ecosistema enorme, springdoc-openapi para spec | Verboso, iteración más lenta en dev, JVM startup time     | APIs enterprise, equipos con background Spring (caso de Jonatan), sistemas críticos       |
| **Quarkus**           | Kubernetes-native, GraalVM native compilation (startup <100ms), API familiar a Spring devs         | Comunidad más chica que Spring, menos Stack Overflow      | Microservicios cloud-native, funciones serverless, containers efímeros                  |
| **Micronaut**         | Compile-time DI (más rápido que reflection), GraalVM-friendly, similar a Spring                  | Comunidad mediana, menos plugins que Spring               | Microservicios rápidos, equipos que valoran startup time                                   |
| **Helidon**           | Oracle-maintained, Helidon SE (microframework) o MP (Jakarta EE)                                | Comunidad chica, menos conocido                          | Entornos Oracle/Java EE existentes, polyglot (Java + Kotlin)                              |

**Decisión por stack:**
- **A) Node + TypeScript:** **NestJS**.
- **B) Python:** **FastAPI**.
- **C) Java + Spring Boot 3.**

**Justificación NestJS sobre Hono/Fastify:** estructura opinionated (decorators, DI, módulos) que matchea con el patrón Controller → Service → Repository pedido en R2 y con el background Spring/Java de Jonatan; reduce decisión/disciplina por convención del framework. Tradeoff: más verboso y curva más alta.

#### Q3. ¿Bun/Deno como runtime, o Node LTS?

**Decisión:** **Node LTS** (universalmente compatible, máxima estabilidad, mejor soporte de NestJS a largo plazo). Bun queda descartado como runtime objetivo. Si Bun aparece como herramienta (test runner, scripts) se evalúa caso por caso en la propuesta Node, pero **no** como runtime del servidor.

### Base de datos y ORM

#### Q4. ¿Qué DB destino?

En nuestro contexto (API local + sync desde la DB fuente) las opciones razonables son tres:

| Característica                | SQLite                                                                | PostgreSQL                                                          | MySQL/MariaDB                                                |
| ----------------------------- | --------------------------------------------------------------------- | ------------------------------------------------------------------- | ------------------------------------------------------------ |
| Forma                         | Embedded (archivo `.db`)                                              | Server (local: Docker o `brew services start postgresql`)           | Server                                                       |
| Setup local                   | Cero — un archivo                                                     | Bajo — un container o servicio                                      | Bajo                                                         |
| Concurrencia                  | Writer lock global (un writer a la vez); excelente para read-heavy    | MVCC — muchos writers en paralelo                                   | MVCC — buena concurrencia                                    |
| Tipos de datos                | Core limitado, JSON nativo (JSON1)                                    | Rico: JSONB, Arrays, Ranges, GIS, full-text                         | Menos rico que PG, JSON nativo                               |
| ORM Node (MikroORM)           | Excelente — adapter oficial `@nestjs/mikro-orm`                       | Excelente (SQLAlchemy async)                                       | Bueno (SQLAlchemy + mysqlclient)                             |
| ORM Python (SQLAlchemy)       | Bueno                                                                 | **Excelente** (camino más popular)                                  | Bueno                                                        |
| Migraciones                   | MikroORM Migrator, Alembic, scripts raw                               | Todas las herramientas del ecosistema                               | Igual                                                        |
| Footprint para nuestro caso   | **Perfecto** — mismo motor que la DB fuente, zero-config           | Overkill para local single-machine                                  | Overkill                                                     |
| Path de escalado              | Vertical (read replicas locales via `@mikro-orm/sqlite`); para escalar multi-nodo, migrar a Postgres | Vertical + horizontal: replicas, partitioning, lógica multi-nodo | Vertical + horizontal                                        |

**Decisión:** **SQLite** como motor de DB para los tres stacks. El driver SQLite queda delegado al ORM/JDBC driver de cada stack: `@mikro-orm/sqlite` para Node (usa `better-sqlite3` por debajo), `sqlite3` stdlib para Python (SQLAlchemy), `sqlite-jdbc` para Java (JPA).

**Razones:**
- Mismo motor que la DB fuente → el sync es trivial (conectar a ambos SQLite desde el mismo proceso).
- Cero servicios adicionales corriendo en local.
- Si la API crece y necesita JSONB o múltiples writers, migrar a Postgres es un cambio de una línea (driver) en ambas propuestas.
- La alternativa "leer la DB fuente directo" se descarta: acopla la API al filesystem del data warehouse.

#### Q5. ¿Qué ORM?

**Definición:** un ORM (Object-Relational Mapper) es una capa entre tu código y la DB que traduce entre **filas/tablas del modelo relacional** y **objetos/estructuras del lenguaje**. Te permite hacer `db.users.findById(1)` en vez de armar el SQL a mano (`SELECT * FROM users WHERE id = 1`).

**Por qué importa acá:**
- La API va a tener modelos del dominio que vienen de tablas. Sin ORM, cada endpoint mezcla lógica de negocio con SQL crudo — feo de mantener y propenso a errores (n+1 queries, inyecciones, tipos flojos).
- El ORM también se encarga de las migraciones (Q6), de tipar los datos en el lenguaje, y (en los modernos) de generar el código desde el schema o viceversa.

**Cómo funcionan en la práctica (4 estilos):**
- **Data-mapper (Hibernate, JPA, Spring Data):** escribís clases 'puras' (entidades), el ORM maneja la persistencia. Cambios a objetos dirty se sincronizan a la DB.
- **Active-record (Django ORM, TypeORM en modo AR):** la fila y el objeto son la misma cosa. `user.save()` persiste el objeto. Más rápido de escribir, pero acopla modelo de dominio con persistencia.
- **Query builder tipado (Drizzle, Kysely, jOOQ):** NO oculta el SQL — te da un builder tipado para componer queries SQL manualmente. Tenés el poder del SQL sin perder tipos.
- **Schema-first (Prisma):** escribís el schema en un DSL propio (`.prisma`), el ORM genera tipos TS y un cliente. Más opinated, pero excelente DX.

**En nuestro flujo específico:**
- Cada stack tiene su ORM idiomático (MikroORM/SQLAlchemy/JPA). Vamos a usar el de cada uno.
- El ORM se integra con las migraciones (Q6): schema en código → migraciones versionadas.
- Los modelos del dominio son straightforward — si hay relaciones N:M, se modelan como junction tables.

**Node + TypeScript:**

| ORM           | Tipo                                | TS types                              | Migraciones                  | Query style                                  | Curva        | Footprint                       |
| ------------- | ----------------------------------- | ------------------------------------- | ---------------------------- | -------------------------------------------- | ------------ | ------------------------------- |
| **Drizzle**   | Query builder + schema typed         | First-class (definís en TS)           | Drizzle Kit (schema → SQL)   | SQL-like, explícito                          | Bajo-medio   | Liviano                         |
| **Prisma**    | Schema-first (DSL propio `.prisma`) | Genera TS desde schema                | Prisma Migrate               | High-level (`db.users.findMany()`)            | Bajo         | Pesado (engine binario Rust)     |
| **Kysely**    | Query builder puro                  | Types solo en el builder              | Knex migrations (separado)   | SQL puro tipado                              | Medio        | Liviano                         |
| **MikroORM**  | Data-mapper estilo JPA/Spring       | Bueno                                 | MikroORM CLI                 | Similar a Hibernate                          | Medio-alto   | Medio                           |
| **TypeORM**   | Decoradores, data-mapper            | TS con decoradores                    | TypeORM CLI                  | Data-mapper / active-record                  | Alto         | Medio                           |

**Python:**

| ORM                  | Estilo                                      | Types                            | Migraciones                  | Curva        | Footprint                         |
| -------------------- | ------------------------------------------- | -------------------------------- | ---------------------------- | ------------ | --------------------------------- |
| **SQLAlchemy 2.0**   | Core + ORM (dos modos, se complementan)     | `Mapped[]` hints + mypy          | Alembic                      | Medio-alto   | Medio                             |
| **SQLModel**         | Inspirado en SQLAlchemy + Pydantic          | Pydantic = models                | Alembic                      | Bajo-medio   | Liviano                           |
| **Django ORM**       | Active-record                               | Limitado (`django-stubs`)         | `makemigrations` + `migrate` | Bajo         | Pesado si sumás todo Django       |
| **Tortoise ORM**     | Async, active-record                        | Tipos vía Pydantic                | Aerich                       | Bajo-medio   | Liviano                           |
| **Peewee**           | Minimalista                                 | Tipos básicos                    | peewee-migrate o manual      | Bajo         | Liviano                           |

**Java:**

| ORM                          | Estilo                                                  | Types                          | Migraciones                     | Curva        | Footprint              |
| ---------------------------- | ------------------------------------------------------- | ------------------------------ | ------------------------------- | ------------ | ---------------------- |
| **Spring Data JPA (Hibernate)** | De facto del mundo Java, JPA estándar, repositories generados | Inferencia por nombre de método | Flyway / Liquibase             | Bajo (con starters de Spring) | Medio-pesado |
| **jOOQ**                     | SQL-first, code-gen del schema → tipos Java             | First-class                     | Flyway / Liquibase              | Medio        | Liviano                |
| **Spring Data JDBC**         | Más simple que JPA, sin lazy-loading magic              | Inferencia básica               | Flyway / Liquibase              | Bajo         | Liviano                |
| **MyBatis**                  | SQL mapper, vos escribís el SQL                         | Manual                         | Flyway / Liquibase              | Bajo         | Liviano                |
| **Jdbi**                     | SQL-first fluent API (similar a jOOQ)                   | Manual                         | Flyway / Liquibase              | Bajo-medio   | Liviano                |

**Decisión:** A) **MikroORM**; B) **SQLAlchemy 2.0**; C) **Spring Data JPA / Hibernate**.

**Por stack:**
- **A)** **MikroORM** (data-mapper con decoradores, identity map, UnitOfWork, lazy loading, transacciones first-class — paridad filosófica con SQLAlchemy). `@nestjs/mikro-orm` es el adapter oficial para NestJS. Tradeoff: SQL queda más oculto que con un query builder puro; se pierde el path a libsql/Turso (MikroORM usa `@mikro-orm/sqlite` con better-sqlite3 por debajo).
- **B)** SQLAlchemy 2.0 (maduras, mypy-friendly, Alembic es battle-tested).
- **C)** **Spring Data JPA / Hibernate** (estándar en Spring Boot, encaja con R2 "origen Spring Boot/Java", repositorios derivados sin escribir SQL).

#### Q6. ¿Qué son las migraciones en este contexto?

**Definición:** una migración es un **cambio versionado del schema de la DB, escrito como código, aplicado en orden**.

**Por qué importa acá:**
- La DB de la API va a **divergir** de la DB fuente (cómputos cacheados, vistas materializadas, índices adicionales, columnas calculadas).
- Si mañana se agrega una columna a la DB fuente, queremos reproducir el cambio en la API de forma **reproducible y auditada**, no a mano.
- Sin migraciones, cada cambio de schema se convierte en: (a) correr SQL a mano (olvidás comandos, no funciona en otra máquina), o (b) borrar la DB y reseedear (perdés estado local como timestamps de sync, contadores, etc.).

**Cómo funcionan en la práctica:**
1. Editás las entities en código (`@Entity()` decorated classes para MikroORM, `models.py` con `Mapped[]` para SQLAlchemy, `@Entity` JPA classes para Java).
2. Corrés el tool → diff vs última migración → genera un nuevo archivo SQL.
   - `drizzle-kit generate` → `0003_add_volume_view.sql`
   - `alembic revision --autogenerate` → `0003_add_volume_view.py`
3. Revisás el SQL generado (clave para no aplicar basura), lo aplicás (`drizzle-kit migrate` / `alembic upgrade head`).
4. El tool lleva una tabla `__migrations` adentro de la DB y solo aplica las nuevas. Revertir (downgrade) existe, pero la mayoría de los flujos modernos son **forward-only**.

**En nuestro flujo específico:**
- La DB de la API va con **forward-only migrations** (más simple, sin riesgo de reversión parcial).
- Cada archivo de migración queda commiteado al repo (es código, no magia).
- El script de sync (Q10) corre **después** de las migraciones en cada deploy o arranque.

**Tools por stack** (todos siguen el patrón "diff vs última migración → archivo versionado → aplicar"):

| Stack                  | Tool recomendado        | Formato de archivos                        | Notas                                                                                              |
| ---------------------- | ----------------------- | ------------------------------------------ | -------------------------------------------------------------------------------------------------- |
| A) Node + TS           | **MikroORM Migrator**   | `.ts` con `up()` / `down()` (ej. `Migration20240101000000.ts`) | Genera diff desde las entities TS decoradas; CLI `npx mikro-orm migrator:generate`. |
| B) Python              | **Alembic**             | `.py` con `upgrade()` / `downgrade()`      | Estándar de facto del ecosistema SQLAlchemy.                                                       |
| C) Java + Spring Boot  | **Flyway**              | `.sql` planos (`V1__create_resources.sql`) | Spring Boot auto-detecta Flyway en el classpath. Alternativa: **Liquibase** (XML/YAML, más potente pero más verboso). **Evitar `hibernate.hbm2ddl.auto=update` en prod** — es dev-only y puede corromper data. |

**Decisión por stack:** A) **MikroORM Migrator**; B) **Alembic**; C) **Flyway**.

### Contrato de API (Swagger / OpenAPI)

#### Q7. Spec-first o code-first?

**Definición:** dos enfoques opuestos para producir el spec OpenAPI:
- **Code-first:** escribís los controllers/types en tu lenguaje, y el spec OpenAPI se **genera desde el código** como un side-effect. Single source of truth = tu código.
- **Spec-first:** escribís `openapi.yaml` (o `openapi.json`) primero, y el código + tipos del cliente se **generan desde el spec**. Single source of truth = el spec.

**Por qué importa acá:**
- La pregunta de fondo es: ¿dónde vive la verdad? ¿En el código de la API o en el archivo de spec?
- Code-first favorece iteración rápida (cambiás el controller y listo); spec-first favorece contratos estables entre equipos (FE y BE acuerdan el spec antes de implementar).

**Comparación:**

| Aspecto | Code-first | Spec-first |
|---|---|---|
| Single source of truth | Código del BE | `openapi.yaml` |
| Iteración | Más rápida (cambiás controller) | Más lenta (cambiás spec + regenerás) |
| Contratos entre equipos | Riesgo de drift (FE consume código, no spec) | Acuerdo explícito antes de codear |
| Tooling moderno | NestJS, FastAPI, Spring Boot 3 — todos lo soportan | openapi-generator, orval, etc. |
| Mejor cuando | Un equipo dueño del FE+BE; APIs que evolucionan | Equipos separados FE/BE; APIs externas/partner |

**Decisión:** **code-first.** Generamos OpenAPI desde el código del stack elegido (NestJS/FastAPI/Spring Boot 3 lo soportan nativo). Iteración más rápida, menos archivos para sincronizar.

#### Q8. Versión del spec? 3.0 vs 3.1?

**Definición:** OpenAPI tiene un número de versión mayor+menor (`3.0.x` o `3.1.x`). Cada versión define qué features del spec son válidas (schemas, parameters, responses, etc.) y cómo se serializan.

**Por qué importa acá:**
- **3.0.x** es la versión más adoptada (3.0.3 es el estándar de facto desde 2017).
- **3.1.x** (publicada en febrero 2021) alinea OpenAPI con JSON Schema 2020-12 — más expresividad (`type: ["string", "null"]`, palabras clave nuevas, etc.).
- Algunos tools todavía no soportan 3.1 completo. Si elegís 3.1 y tu codegen del FE no lo entiende, perdés.

**Comparación:**

| Aspecto | OpenAPI 3.0.3 | OpenAPI 3.1.x |
|---|---|---|
| Estado | Estable, adoptado universalmente | Estable, adopción creciente |
| Alineación con JSON Schema | Parcial (subset de JSON Schema Draft 5) | Total (JSON Schema 2020-12) |
| `nullable` | Truco con `nullable: true` (deprecado en 3.1) | Tipo `null` real (`type: ["string", "null"]`) |
| Soporte tooling | Todos los tools lo soportan | openapi-typescript, orval, swagger-ui (creciente) |
| Mejor cuando | Máxima compatibilidad, ecosistema amplio | Features modernos y tu tooling lo soporta |

**Decisión:** **spec 3.0.** Máxima compatibilidad con tooling del FE (openapi-typescript, swagger-ui); 3.1 queda como upgrade path.

#### Q9. Codegen para el frontend?

**Definición:** un **codegen** es una herramienta que toma el spec OpenAPI y genera archivos en el lenguaje target (TypeScript, Java, Python, etc.). En el FE lo más útil es generar **tipos** (`interface Resource { id: number; name: string; ... }`) y opcionalmente un **cliente HTTP** o **hooks de fetching**.

**Por qué importa acá:**
- Si el spec es la fuente de verdad, el FE puede derivar los tipos de respuesta/request automáticamente → cero tipos manuales que mantener sincronizados.
- Sin codegen: cada vez que cambia un endpoint, alguien edita `interface Resource` a mano. Drift garantizado.

**Comparación:**

| Herramienta | Genera | Output | Mejor cuando |
|---|---|---|---|
| **openapi-typescript** | Solo tipos | `interface`s y `type`s TS puros | Querés máxima flexibilidad en el cliente HTTP del FE |
| **orval** | Tipos + cliente (fetch/axios) + hooks de React Query | Listo para usar, opinionated | Querés cliente + hooks generados, menos código manual |
| **openapi-generator** | Tipos + cliente (multi-lenguaje) | Muchas opciones, verbose | Necesitás generar para múltiples lenguajes (TS + Python + Java) |

**Decisión:** **`openapi-typescript`.** Liviano, solo tipos — no acopla el FE a un cliente HTTP específico. Si después necesitamos cliente generado, se evalúa orval puntualmente.

### Integración con el data warehouse

#### Q10. ¿Cómo accede la API a los datos?

**Definición:** hay tres formas en que una API puede acceder a los datos de otra DB:
- **Sync one-way:** la API mantiene su **propia DB** y un script copia los datos desde la fuente periódicamente.
- **Read directo:** la API **abre la DB fuente directamente** cada vez que necesita un dato (sin DB propia).
- **Migración one-shot:** la API **absorbe** los datos una vez y la fuente queda solo como histórico.

**Por qué importa acá:**
- El data warehouse (SQLite, varias tablas) es la fuente. La API necesita servir esos datos al FE vía HTTP.
- Las 3 opciones tienen tradeoffs de: acoplamiento, latencia, complejidad operacional.

**Comparación:**

| Opción | DB propia | Acoplamiento | Latencia | Complejidad | Mejor cuando |
|---|---|---|---|---|---|
| **Sync one-way** | Sí | Bajo (la API no toca la fuente en runtime) | Latencia del sync (ms si es periódico, hasta horas si es batch) | Media (script de sync, manejo de drift) | Datos cambian lento; querés desacoplar la API del filesystem de la fuente |
| **Read directo** | No | Alto (la API depende del path/estructura de la fuente) | Latencia de query directa | Baja (no hay sync que mantener) | La fuente es estable y el FE necesita datos al instante |
| **Migración one-shot** | Sí (copia completa) | Bajo después de migrar | Latencia de query propia | Baja después (sin sync recurrente) | La fuente es histórica/inmutable y no se actualiza |

**Decisión:** **sync one-way.** La API mantiene su propia DB y un script copia los datos desde la fuente. Desacopla la API del filesystem de la fuente y permite dev con cero servicios adicionales.

### Auth / multi-tenancy

#### Q13. ¿Auth?

**Definición:** **API auth** es el mecanismo por el cual el server verifica que el cliente que llama tiene permiso para hacerlo. Sin auth, cualquiera que conozca la URL puede hacer requests.

**Por qué importa acá:**
- La API va a correr local (puerto `8787`), pero queda expuesta si el firewall no la bloquea.
- Una API key en header es el mínimo viable: el cliente manda `X-API-Key: ***`, el server chequea y rechaza si no coincide.

**Comparación:**

| Método | Cómo funciona | Pros | Contras | Mejor cuando |
|---|---|---|---|---|
| **Sin auth** | El server no verifica nada | Cero código | Cualquiera puede llamar si conoce la URL | Dev local sin red; demos |
| **API key estática** | Header `X-API-Key` con un secret compartido | 5 líneas, simple | El secret rota solo a mano; sin identificar usuario | APIs internas, un solo cliente, dev |
| **JWT con login** | El cliente hace login → recibe JWT firmado → lo manda en cada request | Identifica al usuario; stateless | Requiere endpoint de login, manejo de refresh tokens, key rotation | Multi-usuario, APIs con distintos permisos |
| **OAuth 2.0** | Third-party authorize el cliente | Estándar para third-party apps | Overkill para local; requiere authorization server | APIs públicas, third-party integrations |

**Decisión:** **API key estática en header `X-API-Key`.** Mínimo viable, 5 líneas, evita sustos si el puerto queda expuesto por accidente. JWT queda como upgrade path si aparece multi-usuario.

### Runtime / deployment

#### Q15. ¿Cómo corre? Proceso bare, fat jar, Docker, sidecar…

**Definición:** el **runtime** es el proceso que ejecuta tu código. En dev se prioriza **iteración rápida** (auto-reload al cambiar archivo); en prod se prioriza **estabilidad y performance** (código pre-compilado, workers múltiples).

**Por qué importa acá:**
- Dev con auto-reload evita `Ctrl+C` + re-start en cada cambio (5-10 segundos vs 50ms).
- Prod sin auto-reload evita el overhead del watcher y se asegura de correr el código que testaste.

**Comparación per stack:**

| Stack | Dev (auto-reload) | Prod (estable) | Notas |
|---|---|---|---|
| A) Node + TS | `tsx watch src/index.ts` (re-typecheck + re-run) | Binario standalone con `tsup` o `node --experimental-strip-types` | `--experimental-strip-types` requiere Node 22.6+ LTS |
| B) Python | `uvicorn app:api --reload` | `uvicorn app:api --workers 4` (multi-proceso) | `uvicorn` (ASGI) es el standard para FastAPI |
| C) Java + Spring Boot | `mvn spring-boot:run` con `devtools` (auto-reload) | Fat jar (`java -jar app.jar --server.port=8787`) | Spring Boot DevTools recarga al cambiar classpath |

**Decisión por stack:**
- **A) Node + TS:** `tsx watch` en dev + binario standalone (Node `--experimental-strip-types` o compilado con `tsup`) en prod.
- **B) Python:** `uvicorn --reload` en dev + `uvicorn` (workers) en prod. (Sin Docker por ahora — corremos local.)
- **C) Java + Spring Boot:** `mvn spring-boot:run` en dev + fat jar (`java -jar app.jar`) en prod. Sin Docker por ahora.

#### Q16. Puerto y base path?

**Definición:** **API versioning** es cómo distinguís versiones incompatibles de la API. Las opciones comunes:
- **Sin prefijo:** `/resources` (no hay versión; breaking changes rompen el contrato).
- **En el path:** `/v1/resources` (versión 1, eventualmente `/v2/resources` para breaking changes).
- **En el header:** `Accept: application/vnd.api.v1+json` (versión por content negotiation).
- **Por subdomain:** `v1.api.example.com` (un deploy por versión).

**Por qué importa acá:**
- v1 es read-only y arranca sola. Sin prefijo es lo más simple.
- Si en v2 agregás endpoints de escritura que cambian la semántica del response (ej. `GET /resources` ahora devuelve un campo extra obligatorio), los clientes v1 rompen. Ahí necesitás `/v2/resources` o un `Accept` header.

**Tradeoff:** simplicidad inicial vs flexibilidad futura. Mientras v1 sea la única, no hace falta prefijo.

**Decisión:** **puerto `8787`**, base path raíz (sin prefijo `/v1` por ahora). Si en v2 aparece breaking change, se introduce prefijo `/v1` en ese momento.

### Cross-cutting

#### Q17. ¿Validación de input/response?

**Definición:** **validación de input** es verificar que los datos que llegan del cliente cumplen las reglas de negocio (campos requeridos, formatos, rangos, etc.) **antes** de procesarlos. **Validación de response** es verificar que lo que devolvés cumple el contrato OpenAPI. Sin validación: el server puede recibir basura, fallar tarde, devolver 500s en vez de 400s claros.

**Por qué importa acá:**
- FastAPI (Python) genera 422 con detalles cuando un request no valida. Express (Node) sin validación devuelve 500 con stack trace — feo y leak de info.
- Spring Boot con `jakarta.validation` devuelve 400 con mensajes por campo.
- Elegir bien el framework de validación impacta DX del FE (errores claros) y seguridad (rechazar input malicioso antes de tocar DB).

**Comparación per stack:**

| Stack | Library | Cómo se ve | Integración con OpenAPI | Pros | Contras |
|---|---|---|---|---|---|
| A) Node + TS | **Zod** | `z.object({ id: z.number(), name: z.string().min(1) })` | Vía `@hono/zod-openapi` o `nestjs-zod` | Single source: schema = tipos TS + runtime + OpenAPI | Ecosistema más chico que class-validator |
| B) Python | **Pydantic v2** | `class Resource(BaseModel): id: int; name: str` | Nativo en FastAPI | Maduro, rápido (Rust core), types mypy | Acoplado a FastAPI para OpenAPI |
| C) Java + Spring Boot | **jakarta.validation** | Anotaciones: `@NotNull`, `@Size(min=1, max=100)` | Vía `springdoc-openapi` | Estándar Java, batteries-included | Verboso, errores menos informativos |

**Decisión por stack:**
- **A) Node + TS:** **Zod** — un solo schema, runtime + derivar tipos TS + alimentar OpenAPI (con `@hono/zod-openapi`).
- **B) Python:** **Pydantic v2** — mismo rol (model + validación + OpenAPI nativo en FastAPI).
- **C) Java + Spring Boot:** **jakarta.validation** (Bean Validation, anotaciones como `@NotNull`, `@Size`) + **springdoc-openapi** integra las anotaciones al spec.

#### Q18. ¿Logging?

**Definición:** un **log** es una línea de texto que el server emite cuando pasa algo relevante (request recibido, error, etc.). **Structured logging** es emitir el log en formato parseable (JSON, típicamente) con campos key/value en vez de texto libre. Permite filtrar/buscar por campos (`level=error`) y enviar a sistemas centralizados (Loki, ELK, Datadog).

**Por qué importa acá:**
- Sin structured logs, debugging se reduce a `grep "ERROR"` sobre texto no parseable. Con JSON, podés filtrar `level=error AND path=/resources`.
- Stdout (en vez de archivo) es la convención moderna: el runtime captura (systemd, Docker, k8s) y lo rotea/archiva. Cero config en la app.

**Comparación per stack:**

| Stack | Library | Formato | Pros | Contras |
|---|---|---|---|---|
| A) Node + TS | **Pino** | JSON a stdout | Ultra-rápido (low-allocation), batteries-included (logger + child loggers + redaction) | API menos idiomática que console.log |
| B) Python | **Loguru** | JSON configurable (vía `serialize=True`) | Plug-and-play, mejor DX que `logging` stdlib | Menos extensible que structlog puro |
| C) Java + Spring Boot | **Logback + SLF4J** | Texto por default; JSON vía `logstash-logback-encoder` | Estándar Java, integrado con Spring Boot | Verboso, XML config por default |

**Decisión por stack:**
- **A) Node + TS:** **Pino** structured JSON a stdout.
- **B) Python:** **Loguru** (o structlog) JSON a stdout.
- **C) Java + Spring Boot:** **Logback + SLF4J** (default de `spring-boot-starter-logging`), JSON via `logstash-logback-encoder` si queremos parseo centralizado.

#### Q19. ¿Formato de errores?

**Definición:** un **formato de error de API** es la estructura JSON que el server devuelve cuando algo falla (4xx, 5xx). Define qué información se manda al cliente para que sepa qué pasó y cómo manejarlo.

**Por qué importa acá:**
- Sin formato consistente, cada endpoint devuelve el error que se le canta (string, array, object con campos random) → FE tiene que parsear 5 formatos distintos.
- Con formato consistente, FE puede hacer `if (response.error.code === 'NOT_FOUND')` y mostrar UI apropiada.

**Comparación:**

| Formato | Estructura | Pros | Contras | Mejor cuando |
|---|---|---|---|---|
| **Envelope propio** | `{ error: { code, message, details? } }` | Simple, control total, sin dependencias | Hay que mantenerlo; no es estándar | APIs internas chicas |
| **RFC 7807 (Problem Details)** | `{ type, title, status, detail, instance }` | Estándar IETF; soporte nativo en Spring 6+ | Más verbose; `type` es una URL (¿de quién?) | APIs externas/partner |
| **JSON:API errors** | `{ errors: [{ id, status, code, title, detail, source }] }` | Estándar JSON:API; soporta múltiples errores | Acoplado a JSON:API spec | Apps que ya usan JSON:API |

**Decisión:** **envelope propio simple.** Control total, sin dependencias; RFC 7807 queda como upgrade path si la API pasa a ser pública/partner.

#### Q20. ¿Testing?

**Definición:** una **API testing strategy** combina 3 niveles:
- **Unit:** funciones puras (services, validators) testeadas aisladas con mocks. Rápido (<10ms/test), alto volumen.
- **Integration:** un endpoint real + DB real (o in-memory) + middleware. Verifica el flujo completo. Más lento (100ms-1s/test).
- **End-to-end:** el sistema completo (API + DB + cliente) levantado y testeado como caja negra. Lento (segundos), bajo volumen.

**Por qué importa acá:**
- Sin tests: refactorizar = miedo a romper.
- Solo unit tests: cambios de integración (capa HTTP, DB) pueden romperse sin que ningún test lo detecte.
- Solo e2e: tests lentos, debugging difícil cuando fallan.
- El sweet spot para una API local: ~70% unit, ~25% integration, ~5% smoke e2e.

**Comparación per stack:**

| Stack | Unit | Integration | E2E | Notas |
|---|---|---|---|---|
| A) Node + TS | Vitest (built-in mocks) | Vitest + supertest-style helpers | Vitest + Testcontainers (DB real) | Vitest reemplaza Jest con mejor TS support |
| B) Python | pytest | pytest + httpx AsyncClient | pytest + Testcontainers | `pytest-asyncio` para endpoints async |
| C) Java + Spring Boot | JUnit 5 + Mockito | `@SpringBootTest` + `MockMvc` | `@SpringBootTest` random port + RestAssured | Spring Boot Test es batteries-included |

**Decisión por stack** (cubre los 3 niveles + mock strategy):
- **A) Node + TS:**
  - Unit: **Vitest** (mocks built-in).
  - Integration: **Vitest + supertest** (HTTP layer) + **better-sqlite3 in-memory** (DB real per test).
  - E2E: **Testcontainers** cuando aparezca un caso que lo justifique; v1 arranca sin E2E.
- **B) Python:**
  - Unit: **pytest** (mocks via `unittest.mock`).
  - Integration: **pytest + pytest-asyncio + httpx.AsyncClient** (HTTP) + **SQLite in-memory** (DB real per test).
  - E2E: **Testcontainers** cuando aparezca un caso que lo justifique; v1 arranca sin E2E.
- **C) Java + Spring Boot:**
  - Unit: **JUnit 5 + Mockito**.
  - Integration: **`@SpringBootTest` + `MockMvc`** (con **H2 in-memory** o **Testcontainers** si querés DB real).
  - E2E: **`@SpringBootTest` random port + RestAssured** cuando aparezca un caso que lo justifique; v1 arranca sin E2E.

#### Q21. ¿Lint/format?

**Definición:**
- **Linter:** analiza el código en busca de bugs potenciales, code smells y convenciones no cumplidas (ej. variable no usada, función muy larga, comparación con `==` en vez de `===`).
- **Formatter:** reformatea el código al estilo del equipo (indentación, comillas, line length) sin cambiar la semántica. El código se ve igual para todos.
- **Modern best practice:** un solo tool hace ambas cosas (Biome, Ruff) o vienen muy integradas.

**Por qué importa acá:**
- Sin linter: code review pierde tiempo en style nits.
- Sin formatter: cada developer formatea distinto → diffs inflados por cambios de estilo.
- Pre-commit hook que corre `lint + format --check` evita que código no conforme entre al repo.

**Comparación per stack:**

| Stack | Tool | Qué hace | Velocidad | Notas |
|---|---|---|---|---|
| A) Node + TS | **Biome** | Lint + format en una tool | Ultra-rápido (Rust core) | Reemplaza ESLint + Prettier con una config |
| B) Python | **Ruff** | Lint + format en una tool | Ultra-rápido (Rust core) | Reemplaza flake8 + black + isort + más |
| C) Java + Spring Boot | **Spotless** (format) + **SpotBugs** (análisis) | Formato + análisis estático | Más lento que Biome/Ruff (JVM) | Alternativa: Checkstyle + PMD |

**Decisión por stack:**
- **A) Node + TS:** **Biome** (reemplaza ESLint+Prettier, una config, ultra-rápido).
- **B) Python:** **Ruff** (reemplaza flake8/black/isort, una tool, ultra-rápida).
- **C) Java + Spring Boot:** **Spotless** (formateo via Google Java Format o Palantir) + **SpotBugs** (análisis estático). Alternativa: **Checkstyle + PMD**.

---

## Snapshot — decisiones por stack

> Referencia consolidada de qué herramienta/técnica corresponde a cada capa en cada stack.

| Capa               | Stack A: Node + TS          | Stack B: Python                | Stack C: Java + Spring Boot         |
| ------------------ | ---------------------------- | ------------------------------ | ------------------------------------ |
| Lenguaje           | TypeScript                   | Python 3.12+                   | Java 21 (LTS)                        |
| Runtime            | Node LTS (Bun opcional)      | CPython (uv para env mgmt)     | JVM (GraalVM native opcional)        |
| Framework HTTP     | **NestJS** + `@nestjs/swagger` | FastAPI                        | Spring Boot 3 + springdoc-openapi    |
| ORM                | **MikroORM**                 | **SQLAlchemy 2.0**            | **Spring Data JPA (Hibernate)**      |
| DB                 | SQLite (DB de la API)        | SQLite (DB de la API)          | SQLite (DB de la API)                |
| Migraciones        | MikroORM Migrator            | Alembic                        | Flyway                               |
| Validación         | Zod                          | Pydantic v2                    | jakarta.validation (Bean Validation) |
| OpenAPI            | code-first, spec 3.0         | code-first, spec 3.0           | code-first, spec 3.0                 |
| Frontend codegen   | `openapi-typescript`         | `openapi-typescript` (mismo)   | `openapi-typescript` (mismo)         |
| Auth               | API key (`X-API-Key`)        | API key (`X-API-Key`)          | API key (`X-API-Key`) vía filter     |
| Layout             | Monolito modular             | Monolito modular               | Monolito modular (paquetes por módulo) |
| Datos fuente       | Sync one-way desde la DB fuente | Igual                          | Sync via JDBC                        |
| Read/Write         | CRUD desde v1                | CRUD desde v1                  | CRUD desde v1                        |
| Puerto             | `8787`                       | `8787` (distinto si corren juntos) | `8787` (distinto si corren juntos) |
| Logging            | Pino (JSON)                  | Loguru (JSON)                  | Logback + SLF4J (JSON)               |
| Errors             | Envelope propio              | Envelope propio                | Envelope propio + `@ControllerAdvice` |
| Tests              | Vitest (unit + integration)  | pytest + pytest-asyncio        | JUnit 5 + Mockito + Spring Boot Test |
| Lint/format        | Biome                        | Ruff                           | Spotless + SpotBugs                  |

> **Las tres propuestas comparten:** DB, base path, estructura modular, codegen para el FE, puerto y auth. **Stack C usa el mismo motor de DB y misma auth que A y B** — la diferencia es puramente del lado del lenguaje/ecosistema.

---

## Próximo paso

Escribir las 3 propuestas de arquitectura en paralelo: `architecture-proposal.node-ts.md`, `architecture-proposal.python.md`, `architecture-proposal.java-spring.md`.
